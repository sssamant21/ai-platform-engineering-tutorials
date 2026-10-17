# Chapter 059 — GPU DCGM, Node and Kubernetes Observability

**Part 08 — LLM Observability, Metrics and Tracing | Chapter 059/100**

> **Status:** Written tutorial published. Live execution, independent technical/security review and operational acceptance pending. DCGM exporter metric families, NVIDIA GPU Operator resources, MIG identifiers and cloud node telemetry depend on versions and configuration. Treat all production changes as approved change-controlled work.

## Learning objectives

Instrument GPU hardware and Kubernetes infrastructure for inference reliability; interpret DCGM metrics, VRAM, SM utilization, memory bandwidth, ECC/Xid, thermal and power telemetry; correlate node/Pod placement, GPU allocation and model-serving SLOs; build low-cardinality Prometheus dashboards, alerts, troubleshooting runbooks and cost attribution.

## 1. Reference architecture

```text
Inference gateway → vLLM / Triton / Ray Serve
                            │
                       Pod / node
                            │
            NVIDIA driver + device plugin
                            │
          DCGM exporter / GPU Operator
                            │
  kube-state-metrics + kubelet + node-exporter
                            │
             Prometheus / OTel Collector
                            │
      recording rules + topology normalization
                            │
   TTFT/TPOT SLO ↔ GPU ↔ node ↔ cost dashboard
                            │
                  alerts / runbooks
```

No single layer explains inference health. GPU utilization without request latency, queue, KV-cache and terminal-valid goodput is incomplete evidence.

## 2. Telemetry layers and ownership

| Layer | Core signals | Owner |
|---|---|---|
| GPU | utilization, VRAM, power, temperature, ECC, Xid | GPU platform |
| device plugin | advertised extended resources, health | platform |
| node | CPU, RAM, network, filesystem, kubelet | infrastructure |
| Kubernetes | Pod phase, readiness, restarts, scheduling, limits | SRE |
| inference | TTFT, TPOT, queue, KV, tokens, completions | model serving |
| FinOps | GPU-hours, idle warm pool, valid tokens/$ | platform finance |

## 3. DCGM exporter fundamentals

NVIDIA DCGM Exporter can expose GPU metrics to Prometheus. Typical metric families include `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_FB_USED`, `DCGM_FI_DEV_FB_FREE`, `DCGM_FI_DEV_POWER_USAGE`, `DCGM_FI_DEV_GPU_TEMP`, `DCGM_FI_DEV_SM_CLOCK` and selected ECC/Xid counters or fields. Actual availability depends on GPU SKU, DCGM version, driver, exporter counters file, MIG configuration and permissions. Inspect `# HELP`/`# TYPE` before writing queries.

## 4. Utilization is not goodput

High SM utilization can coexist with poor TTFT due to queueing; low GPU utilization can coexist with severe latency because of CPU preprocessing, routing, model loading or memory stalls. Correlate DCGM with waiting requests, active sequences, KV-cache pressure, output token rate and terminal-valid completions.

## 5. VRAM and memory bandwidth

Measure framebuffer used/free, allocation headroom and out-of-memory events. VRAM pressure includes model weights, KV cache, activations, CUDA graphs and fragmentation. Memory bandwidth pressure may limit decode throughput even when GPU utilization appears moderate. Verify vendor-specific counter support.

## 6. ECC, Xid, thermal and power

ECC and Xid indicators can signal hardware or driver instability; interpretation depends on exact error code and device family. Track temperature, throttling and power usage with appropriate thresholds, and corroborate with node events and NVIDIA diagnostics. Do not reboot or reset GPUs as a first diagnostic step.

## 7. MIG and time-slicing

MIG partitions and time-sliced replicas complicate labels, GPU counts and utilization interpretation. A MIG slice is not a whole GPU, and time-sliced resource advertisements do not guarantee memory isolation. Correlate GPU UUID/MIG instance, node and Pod using supported exporter/Kubernetes mappings; never assume labels are stable across reconfiguration.

## 8. Device plugin and allocatable GPUs

```bash
kubectl get nodes -o wide
kubectl describe node APPROVED_GPU_NODE
kubectl get daemonsets -A
```

Compare GPU Capacity/Allocatable, Pod requests and actual device-plugin readiness. A device visible in `nvidia-smi` but missing from node Allocatable suggests plugin registration, driver or kubelet issues.

## 9. Node telemetry

Track CPU saturation, memory pressure, disk throughput/latency, network errors and packet loss, container runtime issues and kubelet readiness. Model artifact downloads and GPU driver initialization can be limited by CPU, disk or network before GPU execution begins.

## 10. Kubernetes workload signals

Track Pod readiness, restarts, `OOMKilled`, `CrashLoopBackOff`, Pending reasons, HPA/KEDA targets, PDBs, topology spread, node pressure, evictions and scheduling failures. Distinguish CPU memory OOM from CUDA VRAM OOM.

## 11. Cardinality and privacy

Avoid raw GPU UUID, Pod UID, request ID or tenant ID in every high-volume aggregate series unless operationally necessary and bounded. Retain controlled drill-down labels for hardware diagnostics. Never export raw prompts, model responses, secrets or PHI.

## 12. PromQL examples [VERIFY ACTUAL METRICS]

```promql
avg by (instance) (DCGM_FI_DEV_GPU_UTIL)
```

```promql
sum by (instance) (DCGM_FI_DEV_FB_USED)
```

```promql
sum by (node) (kube_pod_container_resource_requests{resource="nvidia_com_gpu"})
```

These queries are illustrative: exporter labels, resource encoding, MIG granularity and kube-state-metrics versions vary. Verify `HELP`, `TYPE`, unit, labels and resource names before using.

## 13. GPU-to-model correlation

Join only verified, unique topology keys between model Pod, node, GPU device and region. Beware many-to-many joins, Pod churn, MIG UUID changes and duplicate scrapes. Record deployment/model version and zone so TTFT regressions can be tied to hardware or placement.

## 14. Dashboard design

Create a three-level dashboard: fleet (GPU inventory, zone, availability, cost), node/device (utilization, VRAM, ECC/Xid, temperature, power) and inference (TTFT/TPOT, queue, KV, terminal-valid goodput). Link symptoms to the relevant Pod, node, device and incident runbook.

## 15. Alerting

Page on GPU device failures with customer impact, persistent CUDA errors, unavailable GPU capacity and SLO violations. Warn on thermal/power trends, VRAM headroom and high idle GPU spend. Avoid paging solely on high utilization or isolated transient temperature spikes.

## 16. FinOps and capacity

Compute GPU-hours from actual node/device allocation duration, distinguish reserved/idle/active pools and normalize by terminal-valid completions or useful tokens. Account for MIG sharing, multi-zone spare capacity, warmup and metric ingestion overhead. Track model/SKU efficiency, not raw utilization alone.

## 17. Safe-read discovery

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get servicemonitors -A 2>/dev/null || true
kubectl describe node APPROVED_GPU_NODE
kubectl -n APPROVED_NS describe pod APPROVED_GPU_POD
```

Authorized internal metrics inspection:

```bash
curl -fsS --max-time 10 http://127.0.0.1:9400/metrics | grep -E '^(# HELP|# TYPE|DCGM_)' | head -100
```

The DCGM endpoint and port are examples; discover the approved endpoint before use.

## 18. Troubleshooting decision tree

```text
GPU observability incident?
 ├─ DCGM target DOWN → service/network/auth
 ├─ DCGM series absent → counters file/version/permissions
 ├─ GPU Allocatable zero → driver/plugin/kubelet
 ├─ GPU high, TTFT high → queue/KV/saturation
 ├─ GPU low, TTFT high → CPU/network/prefill/routing
 ├─ VRAM high/OOM → KV/model/fragmentation
 ├─ Xid/ECC rises → hardware/driver investigation
 ├─ MIG labels drift → reconfiguration/topology
 ├─ Pod Pending → scheduler/taints/quota
 └─ metrics disagree → units/joins/duplicate scrapes
```

## 19. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | DCGM exporter absent | DaemonSet |
| F02 | scrape target DOWN | Prometheus |
| F03 | counter disabled in collector | HELP/TYPE |
| F04 | driver/DCGM version incompatible | logs |
| F05 | GPU Allocatable zero | node |
| F06 | device plugin crash loop | DaemonSet |
| F07 | VRAM OOM misread as host OOM | CUDA logs |
| F08 | GPU utilization treated as SLO | dashboard |
| F09 | memory bandwidth saturation hidden | GPU |
| F10 | ECC/Xid errors ignored | diagnostics |
| F11 | thermal throttling hidden | telemetry |
| F12 | power limit misconfigured | GPU |
| F13 | MIG labels mismatched | topology |
| F14 | time-sliced replicas counted as physical | inventory |
| F15 | Pod-to-GPU mapping stale | join |
| F16 | duplicate scrapes inflate series | targets |
| F17 | node CPU bottleneck mistaken for GPU | node |
| F18 | model download saturates network | traffic |
| F19 | Pod restarts omitted | kube-state |
| F20 | GPU node zone label absent | nodes |
| F21 | high-cardinality GPU/Pod labels | TSDB |
| F22 | sensitive labels exposed | privacy |
| F23 | alerts page on harmless utilization | alerts |
| F24 | idle GPU-hours unaccounted | FinOps |

## 20. Sixteen operational runbooks

1. Inventory GPU exporter and scrape coverage.
2. Recover DCGM exporter deployment.
3. Restore DCGM counter/HELP/TYPE visibility.
4. Resolve driver/DCGM compatibility.
5. Restore device-plugin GPU registration.
6. Diagnose VRAM versus host memory OOM.
7. Correlate GPU utilization with TTFT/TPOT.
8. Diagnose memory bandwidth and KV saturation.
9. Investigate ECC/Xid hardware incidents.
10. Investigate thermal/power throttling.
11. Reconcile MIG/time-slicing device topology.
12. Repair Pod-to-GPU label joins.
13. Restore node CPU/network/storage telemetry.
14. Repair Kubernetes readiness/restart signals.
15. Contain cardinality/privacy and alert noise.
16. Reconcile GPU-hours, rollback and RCA.

## 21. Eight tabletop game days

1. DCGM exporter disappears from one GPU node.
2. Device plugin stops advertising GPU resources.
3. CUDA VRAM OOM occurs while node RAM is healthy.
4. MIG reconfiguration changes device labels.
5. ECC/Xid event coincides with inference errors.
6. CPU preprocessing bottleneck masquerades as GPU shortage.
7. Duplicate scrapes inflate GPU inventory dashboards.
8. Idle warm pool drives GPU cost above budget.

Tabletop by default; live injection requires approved isolated staging.

## 22. Twenty acceptance gates

G01 environment/versions; G02 DCGM exporter; G03 scrape freshness; G04 GPU inventory; G05 allocatable/requests; G06 GPU utilization; G07 VRAM; G08 ECC/Xid; G09 thermal/power; G10 MIG/sharing; G11 Pod-to-GPU mapping; G12 node CPU/memory; G13 network/storage; G14 Pod readiness/restarts; G15 TTFT/TPOT correlation; G16 labels/privacy; G17 alerting; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/059-gpu-dcgm-node-kubernetes-observability-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 060 — OpenTelemetry Traces Across Inference Paths.**
