# 4.6 — GPU Validation, Monitoring and Troubleshooting

**Part:** 04 — GPU Infrastructure  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, ML inference operators  
**Operational safety:** Production commands are **[SAFE-READ]** unless explicitly stated. Changes to drivers, exporters, alerts, nodes or model serving require **[CHANGE-CONTROL]**.

## 1. Objectives

Build an end-to-end GPU health model spanning node hardware, driver, device plugin, Kubernetes scheduling, container GPU access, framework initialization and inference quality. Instrument NVIDIA DCGM exporter, Prometheus and Grafana; distinguish utilization from allocation; identify GPU hardware errors; and execute evidence-led incident triage.

## 2. The GPU health model

```text
GPU hardware and ECC / Xid health
      ↓
NVIDIA driver / nvidia-smi
      ↓
Device plugin / GPU resource advertisement
      ↓
Kubernetes node and Pod scheduling
      ↓
Container GPU visibility / CUDA initialization
      ↓
PyTorch / vLLM model readiness
      ↓
Inference throughput, latency, errors and SLOs
```

A healthy exporter does not prove successful inference; a Running Pod does not prove CUDA initialized; low GPU utilization does not prove schedulable GPU capacity. Collect independent signals for each transition.

## 3. Observability signals and responsibilities

| Layer | Primary signals | Common source | Owner |
|---|---|---|---|
| Hardware | GPU temperature, power, ECC errors, Xid events | DCGM, driver logs, cloud hardware telemetry | GPU infrastructure |
| Driver | Driver version, host GPU enumeration, reset errors | `nvidia-smi`, node logs | Node image / GPU stack |
| Registration | `nvidia.com/gpu` capacity and allocatable | Kubernetes Node API | Device plugin / kubelet |
| Scheduling | Pending duration, FailedScheduling events, allocated GPU requests | Scheduler events, kube-state-metrics | Kubernetes platform |
| Runtime | Device allocation, container creation, GPU visibility | Pod events, container logs | Runtime / toolkit |
| Framework | CUDA availability, initialization failures, OOM | PyTorch / vLLM logs | Model platform |
| Inference | P50/P95/P99 latency, throughput, request errors, queue time | Application metrics / tracing | Serving team |
| Economics | GPU node-hours, idle allocation, GPU-hours per request | Billing + telemetry | Platform / FinOps |

**Correlate by cluster, node, GPU identity, namespace, workload, deployment and time.** GPU UUID and Kubernetes Pod identity may require additional mapping; do not assume the exporter exposes every desired label by default.

## 4. DCGM exporter fundamentals

NVIDIA DCGM exporter can expose GPU telemetry in Prometheus format. It may be deployed standalone or managed by GPU Operator. Exported metric names and availability depend on DCGM/exporter version, hardware and configured collector CSV. Confirm the live `/metrics` endpoint before writing dashboards or alert rules.

Common DCGM metric families include:
- `DCGM_FI_DEV_GPU_UTIL` — GPU utilization.
- `DCGM_FI_DEV_FB_USED` and `DCGM_FI_DEV_FB_FREE` — framebuffer memory usage and free memory.
- `DCGM_FI_DEV_GPU_TEMP` — GPU temperature.
- `DCGM_FI_DEV_POWER_USAGE` — GPU power.
- `DCGM_FI_DEV_XID_ERRORS` — Xid error reporting, when exposed.
- `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL` — double-bit ECC error count, when supported.

**Metric caveats:** some series are gauges, others counters or event-like signals; Xid/ECC semantics and label sets vary. Do not blindly apply `rate()` to an event-like gauge or assume all GPUs support ECC. Confirm HELP/TYPE output and collector configuration first.

## 5. Production-safe discovery [SAFE-READ]

### 5.1 Identify cluster and GPU nodes

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}'
```

`allocatable` is not remaining free GPUs. Check allocated resource requests on the node.

### 5.2 Discover GPU exporter and plugin

```bash
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl get services -A
kubectl get servicemonitors -A
```

The ServiceMonitor command requires the Prometheus Operator CRD; if unavailable, record that and inspect the cluster's actual scrape configuration. GPU Operator and exporter namespaces/names vary by deployment.

```bash
kubectl -n <gpu-namespace> get pods -o wide
kubectl -n <gpu-namespace> describe pod <dcgm-exporter-pod>
kubectl -n <gpu-namespace> logs <dcgm-exporter-pod> --tail=150
```

### 5.3 Inspect exporter metric schema

Where authorized, use a local port-forward to an existing exporter service or Pod (no production configuration changes):

```bash
kubectl -n <gpu-namespace> port-forward pod/<dcgm-exporter-pod> 9400:9400
```

In a separate terminal:

```bash
curl -fsS http://127.0.0.1:9400/metrics | grep -E '^(# (HELP|TYPE) DCGM_FI_DEV_|DCGM_FI_DEV_(GPU_UTIL|FB_USED|GPU_TEMP|XID_ERRORS))' | head -80
```

Port 9400 is a common default, not guaranteed. Use the actual exporter listen port and avoid binding externally. A successful scrape is evidence of exporter reachability, not proof that Prometheus is scraping it.

### 5.4 Inspect workload and application

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> logs <gpu-pod> --tail=200
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Capture TrueFoundry deployment identity, immutable image digest, GPU request/limit, node assignment, framework CUDA result, model readiness and inference errors. Avoid dumping secrets.

## 6. Prometheus and Grafana dashboard design

Build panels only from confirmed live metrics and labels. Recommended panels:

| Panel | Purpose | Caution |
|---|---|---|
| GPU utilization by node/GPU | Identify saturation and idle devices | Does not equal schedulable capacity |
| Framebuffer used/free | Detect VRAM pressure | May require device/Pod attribution mapping |
| GPU temperature and power | Identify thermal/power constraints | Vendor thresholds are SKU-specific |
| Xid/ECC event signals | Detect possible GPU/driver faults | Confirm semantics and severity |
| GPU node allocatable vs requested | Explain Pending Pods and fragmentation | Requires Kubernetes allocation metrics |
| Pending GPU Pod duration | Detect scheduling/provisioning issues | Filter by GPU-requesting workloads |
| CUDA/vLLM initialization errors | Catch application GPU failures | Logs or application metrics required |
| Inference P95/P99, throughput, error rate | Measure user-visible impact | Tie to SLO and workload |
| GPU cost and allocation efficiency | FinOps | Billing rates and workload attribution required |

### Example PromQL patterns

These examples require the named metrics and labels to exist in the live environment:

```promql
# GPU utilization, one series per exporter-provided GPU identity
DCGM_FI_DEV_GPU_UTIL

# Framebuffer memory used; confirm unit from live metric HELP text
DCGM_FI_DEV_FB_USED

# Average GPU utilization by node (only if the exporter supplies a node label)
avg by (node) (DCGM_FI_DEV_GPU_UTIL)

# Number of exported GPU utilization series (not necessarily physical GPU count)
count(DCGM_FI_DEV_GPU_UTIL)
```

For allocation, prefer kube-state-metrics resource metrics or a verified custom recording rule, checking the installed kube-state-metrics schema and `resource="nvidia_com_gpu"` label convention where applicable. Do not assume a universal PromQL query across installations.

## 7. Alert design and noise control

Use multi-signal alerts tied to actionable ownership:

| Alert concept | Trigger design | Action |
|---|---|---|
| GPU capacity disappeared | Previously GPU-advertising Ready node loses expected `nvidia.com/gpu` | Inspect plugin/driver/node lifecycle |
| GPU exporter missing | Expected scrape target down for sustained window | Inspect exporter/service/scrape path |
| GPU hardware fault | Supported ECC/Xid evidence plus node/workload symptoms | Escalate GPU node health |
| VRAM pressure | Sustained high memory use plus OOM/inference errors | Review model concurrency and memory |
| GPU workload stuck Pending | Sustained Pending + FailedScheduling and demand | Inspect allocations/autoscaler |
| CUDA initialization regression | Error increase after image/node change | Compare known-good versions |
| Inference SLO breach | Latency/error burn rate beyond policy | Treat as user-impacting incident |

Do not hardcode GPU temperature or Xid severity thresholds without hardware-specific validation. Suppress planned-maintenance noise using approved silences and retain actionable routing.

## 8. GPU failure troubleshooting matrix

| Symptom | First evidence | Probable boundary |
|---|---|---|
| GPU node NotReady | Node conditions, cloud lifecycle, kubelet logs | Node / provisioning |
| `nvidia.com/gpu` absent | Device-plugin Pods, driver state | Registration |
| GPU Pod Pending | Scheduler events, selectors, taints, allocations | Scheduling |
| Pod Running, CUDA unavailable | Container device visibility, PyTorch build | Runtime / framework |
| `No CUDA runtime is found` in vLLM | Pod GPU allocation, CUDA checks, vLLM logs | Multi-layer; do not guess |
| GPU utilization low but high latency | Queueing, CPU bottleneck, batching, model time | Serving / scheduling |
| GPU utilization high and latency high | VRAM, throughput, model saturation | Capacity / serving |
| Xid errors and device resets | Driver logs, DCGM, node health | GPU hardware / driver |
| Exporter down but inference healthy | Scrape target, exporter Pod, service | Observability |
| OOM despite free node RAM | GPU framebuffer and model allocation | VRAM / application |

**Hardware errors:** an Xid code does not by itself establish a universal action. Capture the exact code, frequency, affected GPU, workload symptoms and vendor guidance before deciding whether to quarantine, replace or reset hardware.

## 9. TrueFoundry incident response runbook

1. Record incident time, workspace, deployment, cluster, namespace, GPU node and user impact.
2. Check inference latency, error rate and model readiness before interpreting hardware graphs.
3. Verify Kubernetes node Ready state, GPU capacity and scheduling.
4. Check exporter health and metric freshness; distinguish monitoring outage from workload outage.
5. Correlate GPU utilization, VRAM, temperature, Xid/ECC and application logs over the same window.
6. Compare recent image, node image, driver, plugin, operator, pool or autoscaler changes.
7. Record **Lowest Proven Healthy Layer**, **First Failed Transition** and actionable owner.
8. Preserve evidence before remediation; follow change control for drains, restarts or node replacement.
9. Validate recovery end-to-end: advertised GPU → scheduled Pod → CUDA → model Ready → successful inference → SLO and monitoring normal.

### Evidence template

```text
Incident ID / timestamp / severity:
TrueFoundry workspace / deployment:
Cluster / namespace / node / GPU identity:
User impact and inference SLO:
GPU pool / model / driver:
Node GPU capacity and allocated requests:
Device plugin / exporter status:
GPU utilization / VRAM / temperature:
Xid / ECC observations and semantics:
Pod scheduling / container state:
CUDA / vLLM logs:
Recent changes:
Lowest Proven Healthy Layer:
First Failed Transition:
Actionable owner:
Remediation approval and rollback:
Post-recovery inference and monitoring evidence:
```

## 10. Maintenance, recovery and rollback [CHANGE-CONTROL]

Do not reset a GPU, restart device plugins, delete GPU Pods, drain nodes or change driver/operator versions just to gather evidence. For confirmed hardware/driver faults, coordinate with the cloud/provider or hardware owner. Use a canary node/pool and compatible driver/runtime/image versions for upgrades.

**Rollback:** revert authoritative IaC/Helm/image configuration to known-good revisions; restore expected GPU registration and workloads. Node replacement or driver rollback may require outage and GPU capacity headroom.

**Acceptance:** exporter fresh, GPU capacity correct, no unexpected hardware errors, CUDA smoke test passes in approved environment, vLLM model Ready, successful inference and SLO baseline restored.

## 11. Hands-on acceptance

- [ ] Distinguish GPU utilization, VRAM use and schedulable GPU allocation.
- [ ] Discover the actual DCGM exporter and inspect its metric schema.
- [ ] Design dashboards from live metrics and validated labels.
- [ ] Classify exporter failure separately from GPU/inference failure.
- [ ] Interpret Xid/ECC evidence without inventing universal thresholds.
- [ ] Produce an incident evidence record and safe recovery plan.

**Lab:** [4.6 — GPU Monitoring and Troubleshooting Check](../labs/06-gpu-validation-monitoring-troubleshooting-check.md)  
**Next:** Chapter 4.7 — Production GPU Infrastructure Recovery Lab.

## References

- [NVIDIA DCGM](https://developer.nvidia.com/dcgm)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
