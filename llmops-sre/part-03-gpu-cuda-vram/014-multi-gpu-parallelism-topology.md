# Chapter 014 — Multi-GPU Parallelism and Topology

**Part 03 — GPU Architecture, CUDA and VRAM | Chapter 014 of 100**

> **Status:** Written tutorial published. Technical/security review, real lab execution and operational acceptance remain pending. All numerical examples are illustrative, not measurements.

## Learning objectives

Distinguish tensor, pipeline, data and expert parallelism; explain GPU communication topology (PCIe, NVLink/NVSwitch and inter-node fabric); estimate memory and communication overhead; diagnose NCCL failures and rank imbalance; evaluate GPU placement, failure domains, throughput, tail latency and cost; and design a bounded staging acceptance plan.

## 1. Why multi-GPU inference?

A model may exceed one GPU's memory, or a single GPU may not meet throughput/latency objectives. Multi-GPU execution can partition work, replicate independent model instances or both. **Adding GPUs does not guarantee linear speedup.** Communication, scheduling, memory imbalance and idle pipeline stages can reduce efficiency.

```text
Gateway / Router
     |
     +---- Replica A (data parallel serving) ----+
     |                                           |
     |    GPU 0 ←→ GPU 1 ←→ GPU 2 ←→ GPU 3       |
     |       tensor parallel group               |
     |       (NCCL collectives, KV/cache shards) |
     |                                           |
     +---- Replica B (separate model instance) ---+
     |
     +---- queue / admission / SLO / tracing

Within a node: PCIe / NVLink / NVSwitch where available
Across nodes: network fabric, RDMA where supported and configured
```

## 2. Parallelism models

| Strategy | Partitioning | Common benefit | Main tradeoff |
|---|---|---|---|
| Tensor parallel (TP) | layer matrices/tensors across GPUs | fit large layers, share compute | frequent collectives and topology sensitivity |
| Pipeline parallel (PP) | contiguous layer groups/stages | model fits across stages/nodes | pipeline bubbles, stage imbalance |
| Data parallel (DP) | full model replica per GPU/group | higher independent request throughput | duplicate weights, routing/fairness |
| Expert parallel (EP) | MoE experts across GPUs | distribute sparse expert capacity | all-to-all traffic, token imbalance |
| Hybrid TP+PP+DP | combination | fit and scale | operational complexity, failure domain |

The supported combinations, rank mapping and cache partitioning depend on model architecture, runtime and release. Not all frameworks support every strategy.

## 3. Topology: physical and logical

- **PCIe:** common host/device and device/device transport; generation, lane width, switches and NUMA placement matter.
- **NVLink/NVSwitch:** higher-bandwidth GPU interconnects on supported systems; not present on all GPUs.
- **Inter-node fabric:** Ethernet or InfiniBand/RDMA deployments vary; NIC locality, congestion and MTU affect collectives.
- **NUMA:** CPU socket and GPU/NIC locality influence staging, pinned memory and network transfers.
- **MIG:** partitioning is not interchangeable with full-GPU topology; inspect supported combinations and memory boundaries.

Do not infer direct GPU connectivity from node count or instance family name alone. Use actual topology inventory and vendor documentation.

## 4. Tensor-parallel communication

TP commonly requires all-reduce, all-gather or reduce-scatter at particular layer boundaries. Collective volume and frequency depend on architecture and implementation. A simplistic compute-plus-communication model:

```text
step_time ≈ local_compute_time + collective_time + synchronization_overhead
```

The terms may overlap; this is a reasoning model, not a scheduler timing guarantee.

### Synthetic example

A single-GPU step takes 80 ms. Four GPUs reduce the ideal compute portion to 20 ms, but communication and coordination add 18 ms:

```text
TP4 step ≈ 20 + 18 = 38 ms
speedup ≈ 80/38 = 2.11×
parallel efficiency ≈ 2.11/4 = 52.6%
```

This is illustrative. Real memory capacity and workload may make single-GPU execution impossible, so a direct speedup comparison may not be available.

## 5. Pipeline parallelism and bubbles

PP divides model layers into stages. Throughput depends on microbatch count, stage balance and pipeline scheduling. A slow stage can dominate, while low batch counts leave GPUs idle. Decode introduces repeated small steps and synchronization challenges; the actual implementation matters.

Capture per-stage compute time, stage queue, memory, token throughput and idle time. Avoid using overall GPU utilization alone as a balance metric.

## 6. Data parallel serving and routing

DP uses separate model replicas and can increase aggregate throughput if routing and GPU resources scale. Replicas need admission, sticky routing where appropriate, per-tenant quotas, autoscaling, warmup and consistent model revisions. A router imbalance can create one overloaded replica while others are idle.

Replicas may run on distinct nodes or zones for failure isolation. A distributed TP group that spans nodes introduces more network and shared failure exposure than a single-node group.

## 7. Expert parallelism for MoE

Mixture-of-Experts models activate subsets of experts per token. EP distributes expert weights and may require all-to-all token dispatch and gather. Hot experts, skewed token routing and network congestion can create tail latency even when aggregate GPU utilization appears healthy.

Track expert load distribution, communication, routing imbalance, per-rank memory and collective latency where supported. Do not assume dense-model capacity equations directly apply to MoE.

## 8. VRAM and rank balance

Each rank may hold a weight shard, KV-cache shard, temporary workspace, communication buffers and runtime overhead. Replication, embeddings and output layers can cause uneven allocation.

```text
per_rank_VRAM ≈ weight_shard + KV_shard + workspace
              + NCCL/runtime buffers + fragmentation + headroom
```

A single rank OOM can fail the entire model group. Record peak memory **per device**, not only summed cluster VRAM. KV sharding and tensor layout are runtime/model-specific.

## 9. Scheduling and Kubernetes

Kubernetes allocates GPUs as extended resources (commonly `nvidia.com/gpu`), but does not automatically guarantee a topology-optimal placement. Runtime rank assignment, node affinity, topology constraints, NUMA and device plugin behavior affect performance.

Before modifying placement, inspect:
- node SKU, GPU count, topology and allocatable resources;
- Pod GPU requests/limits, affinity, tolerations and spread;
- network interface placement, bandwidth and congestion;
- runtime TP/PP/DP/EP configuration and rank-to-device mapping;
- availability zones, disruption budgets and replacement capacity.

Changes to GPU node groups, rank topology or collective environment are **[CHANGE-CONTROL]**.

## 10. Read-only diagnostics

**[SAFE-READ]** in an approved context:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

On an authorized GPU node:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

For a verified serving deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

NCCL debug verbosity and environment settings can expose sensitive topology or create log volume. Do not enable verbose tracing on production without authorization. Redact logs and never capture customer prompts.

## 11. NCCL and collective failure triage

```text
Distributed inference unhealthy?
 ├─ Pod Pending → GPU allocatable/affinity/quotas
 ├─ rank missing → process launch, GPU visibility, placement
 ├─ NCCL init failure → versions, device mapping, transport
 ├─ collective timeout → network, rank crash, skew, deadlock
 ├─ TP slow → topology, collective latency, small batch
 ├─ PP slow → stage imbalance, bubbles
 ├─ DP skew → router/tenant distribution
 ├─ EP skew → hot experts/all-to-all
 ├─ rank OOM → per-rank weights/KV/workspace
 ├─ only cross-node slow → NIC/fabric/RDMA/MTU
 ├─ intermittent after rollout → mixed images or drivers
 └─ metrics stale → exporter/synthetic freshness
```

Differentiate network partition, GPU hardware failure, process crash and a legitimate slow collective before restarting. Repeated blind restarts can amplify impact.

## 12. Reliability and FinOps

Compute cost per **successful completed request**, not merely GPU utilization. Include:
- GPU-hours across all ranks and replicas;
- model-loading and warmup time;
- network/fabric overhead and idle PP stages;
- retry waste after rank failure;
- spare capacity and zone resilience;
- throughput, p95/p99 TTFT/TPOT and complete-stream rate.

Use a failure-domain worksheet: one GPU failure may remove an entire TP/PP group, while DP replicas can provide isolation if routing and spare capacity exist.

## 13. Twenty production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | insufficient GPUs for TP | allocatable/scheduler |
| F02 | incorrect rank-to-device mapping | runtime rank logs |
| F03 | mixed GPU SKUs | inventory |
| F04 | PCIe bottleneck | topology/transfer |
| F05 | NVLink unavailable | topology matrix |
| F06 | cross-node fabric congestion | network counters |
| F07 | NCCL init failure | library/transport |
| F08 | NCCL collective timeout | rank health |
| F09 | rank crash during decode | process logs |
| F10 | TP communication dominates | collective timing |
| F11 | PP stage imbalance | per-stage latency |
| F12 | pipeline bubble at low load | stage utilization |
| F13 | DP router imbalance | replica queue |
| F14 | expert hot spot | MoE routing |
| F15 | all-to-all congestion | EP communication |
| F16 | per-rank CUDA OOM | per-device VRAM |
| F17 | NUMA/NIC locality regression | placement/topology |
| F18 | mixed driver/framework revision | release manifest |
| F19 | single-node failure drops TP group | failure-domain map |
| F20 | more GPUs increase cost/request | FinOps + SLO |

## 14. Fourteen operational runbooks

1. Verify GPU topology and cluster context.
2. Validate TP/PP/DP/EP configuration.
3. Investigate rank/device mapping.
4. Diagnose pending GPU Pods and placement.
5. Investigate PCIe/NVLink bottlenecks.
6. Investigate cross-node fabric/RDMA congestion.
7. Triage NCCL initialization.
8. Triage collective timeout and rank failure.
9. Diagnose TP scaling efficiency.
10. Diagnose PP stage imbalance.
11. Diagnose DP router fairness.
12. Diagnose EP hot experts and all-to-all.
13. Investigate per-rank OOM and GPU cost.
14. Execute approved rollback and verify complete-stream recovery.

Each requires named owner, evidence, severity, approved mitigation, rollback and customer-path verification.

## 15. Eight game days

1. One rank disappears during decode.
2. TP group crosses a slower PCIe topology.
3. NCCL initialization fails after image rollout.
4. Cross-node network congestion increases TPOT.
5. Pipeline stage imbalance causes GPU idle time.
6. DP router overloads one replica.
7. MoE expert skew increases tail latency.
8. Four-GPU configuration costs more per valid request.

Default **tabletop**; fault injection requires separately approved staging and bounded GPU cost.

## 16. Eighteen acceptance gates

G01 parallelism distinctions; G02 topology inventory; G03 PCIe/NVLink/fabric; G04 rank mapping; G05 TP communication; G06 PP bubbles; G07 DP routing; G08 EP routing; G09 per-rank VRAM; G10 NCCL diagnostics; G11 Kubernetes placement; G12 failure domains; G13 TTFT/TPOT/SSE; G14 scaling arithmetic; G15 FinOps; G16 safe experiment; G17 rollback/recovery; G18 independent review.

See [companion lab](../labs/014-multi-gpu-parallelism-topology-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 015 — GPU Diagnostics, DCGM and Failure Lab.**
