# Chapter 084 — GPU Rightsizing and Utilization

**Part 11 — Capacity, FinOps and Reliability | AI Infrastructure Engineering & SRE**

**Publication:** Written tutorial published. Offline lab execution, GPU staging benchmarks, security review and independent technical acceptance remain **pending**.

## 1. Mission

Improve the **useful work per GPU-dollar** of Kubernetes AI platforms while preserving inference p99, model accuracy, training deadlines, checkpoint safety, failure-domain resilience and tenant isolation. Diagnose GPU underutilization, oversized memory reservations, poor batching, wrong accelerator SKUs, MIG fragmentation and idle allocated capacity. Select **reversible, workload-specific** changes instead of indiscriminate consolidation.

**Prerequisites:** Chapter 083, NVIDIA GPU/MIG basics, Kubernetes extended resources, DCGM/Prometheus, distributed training, serving engines, FinOps and change management.

### Learning objectives

- Distinguish allocation, utilization, memory occupancy, throughput and productive efficiency.
- Identify bottlenecks across CPU, PCIe, networking, storage, CUDA and model serving.
- Rightsize training/inference by measured **performance and SLO constraints**.
- Evaluate MIG, time-slicing and full-GPU isolation tradeoffs.
- Calculate realized savings net of retries, migration, reliability headroom and commitments.
- Deploy controlled canaries with clear rollback and independent acceptance evidence.

## 2. GPU efficiency is multidimensional

\`\`\`text
GPU inventory / SKU / price / topology
       ↓
Allocated GPU-hours vs observed busy time
       ↓
GPU SM utilization + HBM occupancy + power
       ↓
Useful work: tokens/sec, steps/sec, SLO-compliant requests
       ↓
Reliability: p99, errors, OOM, retries, deadlines
       ↓
Cost per successful unit of work
\`\`\`

A GPU with high memory use and low SM utilization may be constrained by KV cache, data loading, CPU preprocessing or small batch sizes. A GPU showing 90% utilization can still be inefficient if the model runs with poor throughput or high retry rates. **Neither high utilization nor high memory occupancy is itself an optimization objective.**

## 3. Workload inventory and baselines

| Dimension | Record | Operational interpretation |
|---|---|---|
| GPU SKU/UUID | model, HBM, interconnect, MIG support | compatible placement |
| Allocation | requested GPUs, allocated time, idle windows | scheduling waste |
| GPU activity | SM utilization, memory bandwidth, power | bottleneck clues |
| Memory | used/peak HBM, allocator reserved, KV cache | fit and headroom |
| Training | steps/sec, tokens/sec, global batch, retries | productive compute |
| Serving | requests/sec, tokens/sec, p50/p99, TTFT/TPOT | customer experience |
| Infrastructure | CPU throttling, disk I/O, NIC/RDMA, NUMA | upstream stalls |
| Reliability | OOM, preemption, node failure, checkpoints | risk-adjusted efficiency |
| Cost | actual billed rate, commitments, storage/egress | true cost per work |
| Constraints | accuracy, deadline, SLO, residency, tenancy | feasibility |

Use a representative baseline across peak and off-peak periods. Include workload versions, context length, request distributions and GPU clock/power policy. DCGM metric availability varies by GPU, exporter and configuration; discover actual names.

## 4. Key formulas and pitfalls

### 4.1 Allocated GPU-hours

\[
H_{\mathrm{allocated}}=\sum_i G_i\,T_i
\]

For a job with 8 allocated GPUs for 6 hours: **48 GPU-hours**. This does not measure compute efficiency.

### 4.2 Weighted utilization

\[
U_{\mathrm{weighted}}=
\frac{\sum_i H_i U_i}{\sum_i H_i}
\]

Here \`U_i\` is a measured utilization fraction and \`H_i\` is the associated allocated GPU-hour weight. This is a monitoring statistic, not a substitute for useful work or SLO attainment.

### 4.3 Productive cost

\[
C_{\mathrm{per\,unit}}=
\frac{C_{\mathrm{GPU}}+C_{\mathrm{CPU}}+C_{\mathrm{storage}}+C_{\mathrm{network}}+C_{\mathrm{retry}}}
{\mathrm{successful\ tokens,\ steps,\ or\ SLO\!-\!compliant\ requests}}
\]

Avoid mixing units across training and inference. Use actual contractual billing and amortization rules; do not invent current cloud prices.

### 4.4 Realized savings

\[
S_{\mathrm{net}}=C_{\mathrm{baseline}}-C_{\mathrm{new}}-C_{\mathrm{migration}}-C_{\mathrm{added\,risk}}
\]

Risk cost must be justified with an explicit method and not treated as an arbitrary discount. Reserved-capacity charges may remain unchanged even after a workload is downsized; distinguish **resource freed**, **avoidable spend** and **actual invoice reduction**.

## 5. Synthetic worked example

A batch-inference service uses **4 GPUs × 24 hours = 96 GPU-hours/day**. Measured sustainable throughput at its target p99 is **300 requests/sec across the 4 GPUs**. A validated change uses **3 GPUs** while still meeting the same traffic, p99, availability and headroom gates.

- Baseline: 96 GPU-hours/day.
- Candidate: 72 GPU-hours/day.
- Resource reduction: 24 GPU-hours/day = **25%**.
- If the GPUs are on non-cancelable commitments, near-term invoice savings may be **zero**; the freed capacity has opportunity value only if reused or future purchases avoided.
- The candidate must be tested under realistic peak traffic and node/AZ loss. A 3-GPU configuration may be rejected even if normal-load p99 passes, due to insufficient redundancy.

This is an illustrative calculation, not a real benchmark result.

## 6. Diagnose underutilization before rightsizing

| Symptom | Hypothesis | Verify |
|---|---|---|
| Low SM util, high HBM occupancy | model/KV cache bound, insufficient concurrency | serving queue, TTFT/TPOT, memory |
| Low SM util, low HBM occupancy | CPU/input bottleneck, idle allocation | data loader, CPU throttle, I/O |
| High GPU util, low steps/sec | inefficient kernels, comm overhead | profiler, NCCL, batch size |
| Large GPU memory peaks | transient batch/context spikes | p99 HBM, allocator traces |
| Many Pending GPU pods | fragmentation, quota, topology | scheduler events, MIG profiles |
| Long checkpoint pauses | storage bottleneck | checkpoint p99 and retries |
| High retry GPU-hours | spot interruptions, OOM, failures | failed attempts and RPO |
| Inference p99 degrades at peak | underprovisioning or batching | realistic load test |

Never remove GPUs simply because average utilization is low. An inference replica may be deliberately idle as **failure headroom** or to satisfy cold-start and latency requirements.

## 7. GPU SKU rightsizing

Evaluate workload compatibility against:
- HBM capacity including model weights, activations, optimizer state and KV cache.
- Tensor/pipeline parallelism, interconnect bandwidth and GPU generation.
- Kernel/CUDA compatibility, precision support and numerical correctness.
- Per-GPU throughput at SLO, power and energy efficiency.
- GPU node shape, CPU/RAM, NIC and storage throughput.
- Regional supply, residency and failure-domain placement.

A cheaper GPU SKU can cost **more per successful token/step** if it causes extra replicas, slower training, increased retries or network bottlenecks.

## 8. MIG, time-slicing and isolation

### 8.1 MIG

NVIDIA Multi-Instance GPU partitions supported physical GPUs into hardware-isolated instances with defined compute/memory resources. Availability, profiles and reconfiguration behavior depend on GPU model, driver and GPU Operator/device-plugin configuration.

**Use cases:** small independent inference models and workloads with predictable HBM footprints.

**Risks:** stranded capacity from incompatible profiles, reduced flexibility, profile changes requiring disruption, noninterchangeability with full GPUs and scheduling fragmentation. Do not assume MIG is supported on all GPUs.

### 8.2 Time-slicing

GPU time-slicing can allow multiple workloads to share a device, but it is **not equivalent to MIG hardware isolation** and may lack memory isolation or predictable latency. Use only when tenancy, security, performance and supported software permit it.

### 8.3 Full GPU

Prefer dedicated GPUs for latency-sensitive serving, large-memory models, high-bandwidth training, strict isolation and jobs that require exclusive collectives. Evaluate topology and gang scheduling before pooling.

## 9. Kubernetes resource optimization

Review:
- \`nvidia.com/gpu\` or MIG extended-resource requests and allocatable.
- CPU and memory requests/limits that can bottleneck GPUs.
- Pod/node affinity, taints, topology spread, priority and preemption.
- Node autoscaling and provisioning delays.
- Kueue/Volcano admission, cohort borrowing and full-node/gang scheduling.
- PodDisruptionBudgets, node maintenance, training checkpoint readiness.
- DCGM exporter coverage and node GPU health quarantine.

Kubernetes GPU extended resources are typically integer units; GPU sharing semantics come from vendor plugins and policies, not arbitrary fractional \`nvidia.com/gpu\` requests.

## 10. Training rightsizing

Consider:
1. Batch size and gradient accumulation versus accuracy/convergence.
2. Mixed precision with numerical correctness validation.
3. Activation checkpointing, optimizer sharding and FSDP/DeepSpeed overhead.
4. Data-loader parallelism, CPU and storage performance.
5. NCCL communication cost versus more/fewer GPUs.
6. Checkpoint cadence, spot retry loss and deadline constraints.

A smaller GPU allocation may reduce cost but extend wall-clock time beyond a training deadline. Compare **time to target quality** and **cost to target quality**, not merely GPU utilization.

## 11. Inference rightsizing

Evaluate:
- realistic input/output token distributions and context lengths;
- continuous batching, max concurrency and admission controls;
- model quantization and accuracy/regression tests;
- KV-cache budget and memory fragmentation;
- autoscaling cold starts and model load time;
- p95/p99 latency, TTFT, TPOT, error rate and failure-zone redundancy.

Use canary traffic with fixed success and abort thresholds. A throughput gain that breaches p99 or reduces answer quality is not an accepted optimization.

## 12. Reliability-aware FinOps policy

Every candidate must satisfy:

\`\`\`text
Candidate change
  ├─ GPU SKU + memory compatibility PASS?
  ├─ Numerical/quality acceptance PASS?
  ├─ p99, error and throughput SLO PASS?
  ├─ Node/AZ loss and headroom PASS?
  ├─ Checkpoint/rollback readiness PASS?
  ├─ Tenant security and isolation PASS?
  └─ Actual avoidable spend or capacity benefit proven?
         → approved canary → staged rollout → effectiveness review
\`\`\`

Track capacity released, workload cost per unit, commitment utilization, incident rate and rollback count. Avoid “savings” claims that only move costs between teams or conceal underprovisioning.

## 13. Five production runbooks

**A — Low GPU utilization:** correlate GPU activity, data-loader/CPU wait, NCCL and storage; find true bottleneck before modifying GPU count.

**B — Oversized serving fleet:** measure peak traffic and p99, stage replica/batching reduction, validate failure headroom, canary and rollback.

**C — MIG fragmentation:** map physical GPU profiles, pending resource requests and tenant isolation; plan reconfiguration in maintenance only with owner approval.

**D — Training GPU efficiency:** compare time-to-quality, throughput, memory, collective overhead and retry cost; test fewer GPUs/sharding changes in staging.

**E — FinOps savings verification:** reconcile billed commitments, released capacity, actual avoided purchases, quality and reliability after change.

## 14. Sixteen hands-on exercises

1. Inventory GPU SKU, node, MIG and failure domains.
2. Reconcile allocated GPU-hours with observed GPU activity.
3. Build utilization-weighted baseline with coverage checks.
4. Analyze synthetic low-utilization training job.
5. Analyze synthetic inference memory/KV-cache pressure.
6. Compare GPU SKU compatibility and HBM fit.
7. Calculate baseline cost per useful training step.
8. Calculate inference cost per SLO-compliant request.
9. Model MIG profiles and fragmentation.
10. Compare MIG, time-slicing and full-GPU isolation.
11. Inspect Kubernetes GPU requests, quotas and pending pods.
12. Evaluate batch/precision/sharding changes for training.
13. Evaluate serving batching and replica reduction under p99.
14. Simulate node/AZ failure headroom and rollback.
15. Compute realized vs nominal savings with commitments.
16. Produce rightsizing proposal, risk register and acceptance score.

## 15. Twelve troubleshooting scenarios

| Symptom | Required investigation |
|---|---|
| GPU util 15%, high latency | CPU/storage/queue and memory |
| GPU util 90%, poor throughput | kernels, NCCL, retry overhead |
| Low allocated utilization | idle reservation vs HA headroom |
| High HBM occupancy, low SM | KV cache/model fit |
| MIG resources stranded | profile fragmentation |
| Time-sliced tenants interfere | isolation/latency |
| Smaller GPU OOMs | peak HBM and context |
| Training cheaper per hour but slower | cost to target quality |
| Inference p99 fails after reduction | canary rollback |
| Node autoscaler adds wrong SKU | node pool constraints |
| Spot savings erased by retries | failed GPU-hours |
| Commitments unchanged after resize | invoice vs capacity accounting |

## 16. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Inventory, telemetry and baseline | 20 |
| Bottleneck and workload classification | 20 |
| SKU/MIG/scheduler feasibility | 20 |
| Training/inference SLO and quality tests | 20 |
| Net savings and resilience analysis | 10 |
| Runbooks, rollback and peer review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical SLO, accuracy, tenant isolation, checkpoint integrity, unapproved disruption or fabricated savings failure. Publication does not imply live execution.

## 17. References

- [NVIDIA Multi-Instance GPU User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kueue Documentation](https://kueue.sigs.k8s.io/)
- [FinOps Foundation Framework](https://www.finops.org/framework/)

**Next:** Chapter 085 — Capacity Reservation and Quota Governance.
