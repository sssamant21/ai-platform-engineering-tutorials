# Chapter 015 — GPU Diagnostics, DCGM and Failure Lab

**Part 03 — GPU Architecture, CUDA and VRAM | Chapter 015 of 100 | Part 03 capstone**

> **Status:** Written tutorial published. No live lab, failure injection, security review or operational acceptance has been completed by publication. All numerical exercises are synthetic.

## Learning objectives

Operate a layered GPU observability and incident workflow; distinguish NVIDIA driver/device failures from CUDA, memory, scheduler and model-serving failures; interpret DCGM and nvidia-smi signals; diagnose XID/ECC, thermal, power, OOM, NCCL and topology incidents; verify metrics freshness; and produce a production-grade incident report, game-day plan and reliability acceptance rubric.

## 1. GPU SRE diagnostic architecture

```text
Synthetic customer-path probe
  → gateway/auth/stream completion → model runtime TTFT/TPOT/queue
  → Kubernetes workload/node scheduling
  → NVIDIA device plugin / container toolkit
  → CUDA driver/runtime, NCCL and GPU kernels
  → GPU SM/HBM/PCIe/NVLink/power/thermal/ECC
                  │
                  ▼
    DCGM exporter / approved GPU telemetry
                  │
                  ▼
    Prometheus / dashboards / alerts / incident evidence
```

No single signal proves GPU health. High GPU utilization can coexist with poor token throughput, and a healthy `nvidia-smi` result does not prove that a container can initialize CUDA or that customer SSE requests finish successfully.

## 2. DCGM, exporter and device-level observability

NVIDIA Data Center GPU Manager (DCGM) exposes GPU health and performance telemetry on supported devices. DCGM Exporter can publish selected metrics for Prometheus. **Actual metric names and availability depend on GPU, driver, DCGM/exporter version, permissions and exporter configuration.** Do not write alerts against assumed series without first inspecting the deployed metrics.

| Signal category | Typical question | Important caveat |
|---|---|---|
| GPU utilization / SM activity | is compute active? | not useful work efficiency |
| HBM memory used / bandwidth | is memory capacity/bandwidth pressured? | not allocatable KV blocks |
| Power / clocks / temperature | is throttling occurring? | workload and GPU limits vary |
| ECC counters | corrected or uncorrected memory errors? | support depends on device |
| XID / driver errors | device/driver failure? | requires host logs and NVIDIA guidance |
| PCIe/NVLink throughput | is communication bottlenecked? | topology-specific |
| GPU process memory | which workloads allocate VRAM? | device-wide vs tenant attribution |
| Exporter freshness | are metrics current? | missing series is not zero |
| Runtime KV/preemption | does serving pressure explain symptoms? | runtime/version-specific |
| Complete SSE synthetic | is customer journey healthy? | separate from GPU telemetry |

Prometheus counter resets, scrape gaps, GPU resets and device replacement can create false trends. Label cardinality must be bounded; do not label GPU metrics with raw prompts or user identifiers.

## 3. Read-only GPU diagnostics

**[SAFE-READ]** only on approved contexts, nodes or authorized GPU troubleshooting containers:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,daemonsets -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
kubectl describe nodes
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi -q
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,memory.used,utilization.gpu,temperature.gpu,power.draw,clocks.sm --format=csv
```

If DCGM exporter is installed, locate it through approved cluster inventory and inspect its configured scrape target and sample timestamps. Do not assume a namespace or DaemonSet name. If `nvidia-smi` is unavailable in a container, investigate device/runtime visibility before concluding the host driver is absent.

For a verified serving deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Redact prompts, secrets, tokens and model artifacts from evidence.

## 4. GPU hardware versus software failure domains

| Domain | Failure example | Primary evidence |
|---|---|---|
| Kubernetes | GPU Pod Pending | events, allocatable, taints |
| Device plugin | missing GPU resource | DaemonSet, node status |
| Container toolkit | GPU invisible in Pod | runtime/device mounts |
| Host driver | module failure | host logs, nvidia-smi |
| CUDA/framework | incompatible runtime | application error, release manifest |
| VRAM/KV | allocation OOM | per-device memory and cache blocks |
| Scheduler | queue/preemption storm | runtime counters |
| Hardware | XID/ECC/thermal | device/driver telemetry |
| Topology | NCCL timeout | rank logs, network/fabric |
| Monitoring | stale DCGM metrics | scrape target, timestamp |
| Customer path | incomplete HTTP 200 SSE | synthetic parser, terminal event |

Do not equate every CUDA error with defective hardware. XID codes are diagnostic indicators requiring exact NVIDIA documentation, driver context and hardware escalation; avoid universal automatic remediation based on an arbitrary code.

## 5. ECC, XID and device reset response

1. Verify incident scope, customer impact and affected GPU UUID/node.
2. Preserve sanitized driver logs, timestamps, device identity and recent changes.
3. Check for correlated ECC, XID, power, thermal or PCIe events.
4. Distinguish recoverable application failure from persistent device instability.
5. Escalate to GPU platform/cloud provider with documented evidence.
6. If approved, migrate workloads and drain/recreate affected capacity through the authoritative controller.
7. Never issue a GPU reset, change persistence/MIG mode or restart the driver on production without explicit change authorization.
8. Validate device plugin health, model load, complete SSE and sustained SLO recovery.

## 6. Thermal and power throttling

Low clocks under high demand may reflect thermal, power or application-specific behavior. Collect power draw, enforced limits, temperature, clock history, GPU utilization and throughput. A high temperature reading alone is not sufficient to declare throttling; compare against documented device thresholds and throttle reason indicators where supported.

Investigate airflow, rack/power limits, competing workloads, node cooling and cloud instance constraints through infrastructure owners. Do not alter power limits in production as an ad hoc fix.

## 7. CUDA OOM, KV pressure and fragmentation

```text
GPU memory envelope =
 weights + KV cache + workspace/activations + runtime buffers
 + allocator fragmentation + safety headroom
```

OOM at model load suggests weights/initialization/workspace; OOM during prefill may reflect large prompts and temporary buffers; decode OOM may reflect resident KV tokens and concurrency. Device-wide free VRAM does not always represent allocatable contiguous runtime blocks.

Correlate model/tokenizer/runtime revisions, prompt/output length cohorts, active sequences, KV blocks, preemptions, GPU memory, tensor-parallel rank allocation and CUDA logs. Avoid blind restarts or raising concurrency without a capacity model.

## 8. Multi-GPU and NCCL diagnostics

Collect per-rank GPU UUID, memory, process health, topology, NCCL version, fabric and collective timeout evidence. One rank crash or OOM may manifest as a timeout on healthy peers. Compare intra-node and inter-node performance and check for mixed drivers/images or NUMA/NIC locality changes.

Verbose NCCL logging, packet capture and invasive profiling require approval because of performance and information exposure.

## 9. SLI/SLO and alerting contract

**Customer-facing SLIs:** valid complete-stream rate, TTFT, TPOT/ITL, E2E, 5xx/429 classification and eligible requests.

**Platform signals:** GPU memory, runtime cache occupancy, OOM, preemption, XID/ECC, clock/power/temperature, device-plugin health, pending GPU Pods, rank failure and DCGM freshness.

Use multi-window burn-rate alerts for customer impact plus actionable device alerts. Avoid treating high GPU utilization alone as an incident. Alert routing should identify service owner, GPU platform owner, severity and rollback authority.

### Synthetic error-budget example

For a 99.9% success SLO and 50,000 eligible requests, the permitted failures are 50. If 250 requests fail, observed success is 99.5%, the failure rate is 0.5%, and burn is 5× the allowed 0.1% rate. This is illustrative, not an observed incident.

## 10. Incident response and change control

**Triage:** customer-path synthetic → gateway/runtime → scheduler/KV → CUDA/driver → hardware/topology → telemetry freshness.

**Containment options (approval-dependent):** shed overload, limit long-context admission, reduce concurrent sequences, reroute to healthy replicas, revert a serving image, or replace a degraded GPU node. Confirm owner, cost, spare capacity and rollback before action.

**Recovery:** verify full SSE completion, TTFT/TPOT p95/p99, cache stability, no recurring XID/ECC, restored telemetry and sustained customer-path health.

**RCA/COA:** timeline, blast radius, triggering change, contributing factors, evidence, corrective/preventive actions, assigned owners, deadlines and independent review.

## 11. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | DCGM exporter missing | target inventory |
| F02 | GPU metrics stale | scrape timestamps |
| F03 | wrong GPU label mapping | UUID/node correlation |
| F04 | counter reset misread | exporter/device restart |
| F05 | device plugin crash | DaemonSet/events |
| F06 | GPU allocatable disappears | node status |
| F07 | CUDA initialization failure | driver/runtime |
| F08 | driver module error | host logs |
| F09 | XID event | driver event and GPU UUID |
| F10 | uncorrected ECC event | device health |
| F11 | corrected ECC trend | error counters |
| F12 | thermal throttling | clocks/temperature |
| F13 | power throttling | limits/clocks |
| F14 | model load OOM | weight allocation |
| F15 | prefill workspace OOM | input-token cohort |
| F16 | decode KV-cache OOM | resident tokens |
| F17 | allocator fragmentation | runtime allocator |
| F18 | preemption storm | scheduler counters |
| F19 | high GPU utilization, low throughput | token rate/batching |
| F20 | low GPU utilization, high queue | admission/CPU |
| F21 | NCCL timeout | rank and network |
| F22 | per-rank memory imbalance | per-device VRAM |
| F23 | GPU reset disrupts streams | change history |
| F24 | HTTP 200 with incomplete SSE | synthetic validator |

## 12. Sixteen operational runbooks

1. Verify cluster context and incident scope.
2. Check customer-path complete-stream SLI.
3. Validate DCGM/exporter freshness.
4. Map GPU UUID to node, Pod and model.
5. Triage missing device-plugin allocatable resources.
6. Diagnose host driver and CUDA initialization.
7. Investigate XID and ECC incidents.
8. Investigate thermal and power throttling.
9. Investigate load-time OOM.
10. Investigate prefill/decode KV-cache OOM.
11. Investigate fragmentation and preemption.
12. Diagnose GPU utilization versus token throughput.
13. Investigate NCCL/rank and topology failures.
14. Review GPU capacity, idle cost and headroom.
15. Execute approved mitigation and rollback.
16. Validate recovery, RCA/COA and signoff.

Each runbook needs named owner, authorized evidence, impact, escalation, rollback and recovery verification.

## 13. Ten game days

1. DCGM exporter stops scraping while requests degrade.
2. GPU disappears from Kubernetes allocatable.
3. XID error appears during a model rollout.
4. ECC alert correlates with incomplete streams.
5. Thermal throttling doubles TPOT.
6. KV-cache pressure causes CUDA OOM.
7. Allocator fragmentation produces intermittent failures.
8. NCCL rank loss stalls a multi-GPU group.
9. High GPU utilization masks low valid tokens/s.
10. Recovery appears green on HTTP 200 but SSE completion remains broken.

Default **tabletop**. Live fault injection or stress tests require separately approved staging and bounded GPU cost.

## 14. 100-point Part 03 acceptance rubric

| Category | Points |
|---|---:|
| Safety, access and data protection | 10 |
| GPU architecture and memory hierarchy | 10 |
| CUDA driver/container compatibility | 15 |
| VRAM, precision and quantization | 15 |
| Multi-GPU topology and NCCL | 15 |
| DCGM/telemetry and incident diagnosis | 15 |
| Reliability, rollback and recovery | 10 |
| FinOps, evidence and independent review | 10 |
| **Total** | **100** |

**Proposed threshold:** ≥85/100 with no critical safety failure, correct CUDA/driver diagnosis, sound VRAM math, validated customer-path completion, approved rollback and independent reviewer signoff. Publication alone does not satisfy the rubric.

## 15. Twenty acceptance gates

G01 scope and authorization; G02 correct context; G03 privacy-safe evidence; G04 GPU architecture; G05 memory hierarchy; G06 CUDA/driver compatibility; G07 container/device plugin; G08 VRAM budget; G09 quantization quality; G10 TP/PP/DP/EP; G11 topology/NCCL; G12 DCGM freshness; G13 XID/ECC triage; G14 thermal/power; G15 CUDA OOM; G16 TTFT/TPOT/SSE; G17 SLO burn; G18 FinOps; G19 rollback/RCA; G20 independent signoff.

All gates **pending** until executed and reviewed. See [companion capstone lab](../labs/015-gpu-diagnostics-dcgm-failure-lab-check.md).

## 16. Part 03 publication closeout

Chapters 011–015 and companion labs constitute the **written** GPU/CUDA/VRAM curriculum. Operational completion requires real evidence, review, lab execution where authorized and accepted critical findings.

**Next: Chapter 016 — vLLM Architecture and Production Deployment**, beginning Part 04 — vLLM Administration and Operations.
