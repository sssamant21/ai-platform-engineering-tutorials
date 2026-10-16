# Chapter 021 — vLLM Tensor/Pipeline Parallelism and Distributed Serving

**Part 04 — vLLM Administration and Operations | Chapter 021 of 100**

> **Status:** Written tutorial published; technical/security review, live lab execution and operational acceptance pending. vLLM distributed backends, flags and network behavior change by version; validate the pinned release.

## Learning objectives

Design and operate tensor parallel (TP) and pipeline parallel (PP) inference in vLLM; distinguish intra-node and multi-node serving; map GPU ranks, NCCL collectives and KV-cache shards; validate distributed launch, Kubernetes placement, networking, security and recovery; benchmark scaling efficiency, complete SSE, TTFT/TPOT and GPU cost.

## 1. Distributed vLLM architecture

```text
TLS/Auth/Quota Gateway
          │
       API Service
          │
    vLLM API / Engine
          │
   Distributed coordinator
          │
   ┌──────┴─────────┐
   │                │
Node A           Node B
GPU0 GPU1        GPU2 GPU3
  TP group         TP group
  │                  │
  └── PP stage link ─┘
          │
  NCCL / network fabric
          │
  KV/cache and rank health
          ▼
  Complete SSE / SLO
```

This diagram is conceptual. Actual placement, process boundaries, backends and supported TP/PP combinations depend on the model and vLLM release. Never assume a distributed engine can survive loss of a rank without disrupting in-flight requests.

## 2. Tensor versus pipeline parallelism

| Strategy | How it partitions | Advantage | Tradeoff |
|---|---|---|---|
| Tensor parallel (TP) | layer tensor operations across GPUs | fit wide layers, parallel compute | frequent collective communication |
| Pipeline parallel (PP) | layers across sequential stages | fit deep models across stages/nodes | stage imbalance and pipeline bubbles |
| Data parallel serving (DP) | independent replicas/groups | higher aggregate throughput and fault isolation | duplicated weights and routing |
| Hybrid TP+PP | groups within stages and stage chain | scale beyond one node | scheduling and network complexity |

TP is often attractive for fast GPU-to-GPU interconnects within a node. PP can span nodes but may suffer stage latency and idle time. These are architectural tendencies, not universal performance guarantees.

## 3. Rank and placement mathematics

For a supported TP × PP layout:

```text
participating GPUs ≈ TP_size × PP_size
```

Synthetic examples:
- TP4, PP1: four GPUs sharing tensor-parallel work in one stage.
- TP2, PP2: four GPUs across two pipeline stages, each stage with two-way tensor parallelism.
- TP4, PP2: eight GPUs across two stages.

The runtime's mapping of ranks to devices/nodes must be checked. Do not assume that a scheduler will automatically select the fastest NVLink peers or allocate contiguous GPUs.

A rough per-rank weight estimate is `total_weight_bytes / participating_GPUs` **plus** replicated parameters, embeddings, buffers and uneven layer partitioning. KV/cache sharding is architecture- and runtime-specific. A single rank OOM can fail the entire group.

## 4. GPU and network topology

Use actual device topology, not node labels alone:

- **PCIe:** root complexes, switches, NUMA locality and transfer contention.
- **NVLink/NVSwitch:** supported GPU-to-GPU high-bandwidth paths.
- **Ethernet/InfiniBand/RDMA:** cross-node transport; NIC locality, MTU, congestion and firewall rules matter.
- **NCCL:** collective initialization and operations; hangs can be caused by a failed peer, bad mapping or network trouble.
- **Availability zones:** inter-zone latency and failure domains can dominate distributed inference.

Never assume GPUDirect RDMA or NVLink is available solely from a GPU SKU. Confirm provider, host and driver support.

## 5. Version-aware vLLM configuration

A conceptual **isolated staging** launch pattern:

```bash
vllm serve "APPROVED_MODEL_ID_OR_LOCAL_PATH" \
  --served-model-name "APPROVED_MODEL_ALIAS" \
  --tensor-parallel-size 2 \
  --pipeline-parallel-size 1
```

This is **not** a production deployment recipe. Verify `vllm serve --help` in the pinned image and confirm support for the chosen model, GPU count and backend. Multi-node execution requires a supported distributed executor and secure cluster orchestration; do not improvise a distributed cluster by exposing unrestricted coordination ports.

## 6. Multi-node orchestration

Distributed vLLM serving may use an executor/backend such as Ray in supported releases and deployments. Confirm version-specific orchestration guidance. Multi-node inference requires:
1. Immutable image, model/tokenizer revision and compatible driver/CUDA/NCCL on every node.
2. Approved workload identity and model artifact access.
3. Explicit rank-to-node/GPU map, network interface selection and reachability.
4. Bounded coordination traffic with firewall/network policy and trusted membership.
5. GPU scheduling and gang/co-scheduling behavior appropriate to the deployment platform.
6. Node health, startup ordering, failure detection and restart/rollback ownership.
7. Model warmup and full-stream customer-path validation.

Avoid exposing Ray dashboards, cluster control ports, NCCL rendezvous or debugging interfaces publicly. Treat distributed workers as privileged model-execution infrastructure and isolate them.

## 7. Kubernetes GPU placement

**[SAFE-READ]**:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Review extended resource requests, node affinity, tolerations, topology spread, device plugin, quotas and GPU SKU. A Kubernetes Pod requiring four GPUs generally needs those four GPU resources available on one node; multi-node groups require a distributed orchestration design rather than a single Pod spanning nodes.

## 8. Distributed memory and KV-cache pressure

Per rank, account for:
- local weight shards and replicated layers;
- KV-cache allocations and resident tokens;
- collective communication buffers;
- CUDA graphs and runtime workspace;
- fragmentation and headroom.

Use **per-GPU peak VRAM**, not only cluster aggregate. Distributed model startup may succeed while long-context decode fails one rank due to uneven KV or temporary allocations.

## 9. Communication and latency model

```text
request latency ≈ queue + prefill + decode + communication
                + stage synchronization + stream transport
```

This is a conceptual decomposition; some terms overlap. TP speedup is constrained by collective latency, batch size and topology. PP throughput depends on microbatching and stage balance.

### Synthetic TP example

Single-GPU step 80 ms; TP4 ideal compute 20 ms; collectives and coordination 18 ms:

```text
TP4 step ≈ 38 ms
speedup ≈ 80 / 38 = 2.11×
parallel efficiency ≈ 2.11 / 4 = 52.6%
```

Illustrative only. When the model cannot fit on one GPU, a single-GPU benchmark may not exist.

## 10. Distributed serving SLOs

Track complete SSE success, p95/p99 TTFT, TPOT, E2E, queue wait, active requests, preemption, per-rank GPU memory, rank restarts, NCCL timeout, network congestion and GPU-hours per valid request.

For multi-node serving, include per-stage time, collective timing and fabric counters where supported. Validate metrics freshness and avoid high-cardinality rank/request labels.

## 11. Read-only incident triage

For a verified serving deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Check all participating ranks and their GPU UUIDs. NCCL timeouts can be secondary symptoms of a rank OOM or process crash. Do not enable verbose NCCL tracing or packet capture on production without authorization; redact model data and network secrets.

## 12. Failure decision tree

```text
Distributed serving degraded?
 ├─ Pod Pending → GPU quota, affinity, gang scheduling
 ├─ rank missing → GPU visibility, launch, placement
 ├─ model load OOM → per-rank weights/workspace
 ├─ decode OOM → per-rank KV and concurrency
 ├─ NCCL init fails → transport, versions, rank map
 ├─ collective timeout → peer crash, network, skew
 ├─ only cross-node slow → fabric/NIC/MTU/congestion
 ├─ TP scales poorly → collective overhead/topology
 ├─ PP slow → stage imbalance/pipeline bubbles
 ├─ streaming incomplete → rank loss, proxy timeout
 ├─ canary unstable → mixed images/model revisions
 ├─ security exposure → coordinator ports/policies
 └─ GPU cost rises → poor efficiency/idle stages
```

## 13. Change control and rollback

1. Freeze model, tokenizer, runtime image, driver/CUDA/NCCL and rank topology.
2. Obtain authorization for GPU cost, network access, coordination ports and scheduling.
3. Validate a single-node topology before multi-node deployment where model fit permits.
4. Verify placement, rank membership, model load and per-rank VRAM.
5. Run bounded synthetic full-stream tests across short/long prompt and output cohorts.
6. Measure TTFT/TPOT, collective time, stage utilization, complete SSE and cost.
7. Canary only with a pre-approved failure-domain and routing strategy.
8. Stop on rank instability, NCCL timeout, OOM, incomplete SSE, SLO burn or cost ceiling.
9. Roll back via the authoritative GitOps/orchestration controller, ensuring no orphan GPU workers.
10. Verify sustained recovery and document RCA/COA.

## 14. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | insufficient GPU count | scheduler |
| F02 | incorrect TP size | config/rank |
| F03 | incorrect PP size | stage map |
| F04 | rank-to-device mismatch | worker logs |
| F05 | mixed GPU SKU | inventory |
| F06 | driver/CUDA mismatch | initialization |
| F07 | NCCL version mismatch | library inventory |
| F08 | NCCL initialization hang | rank logs |
| F09 | collective timeout | peer health |
| F10 | rank crash mid-stream | worker status |
| F11 | one rank OOM | per-rank VRAM |
| F12 | KV imbalance | cache/rank metrics |
| F13 | PCIe bottleneck | topology |
| F14 | NVLink unavailable | topo matrix |
| F15 | cross-node congestion | fabric counters |
| F16 | wrong network interface | transport |
| F17 | coordination port blocked | connectivity |
| F18 | insecure coordinator exposure | network policy |
| F19 | pipeline stage imbalance | stage latency |
| F20 | PP bubble under low batch | idle time |
| F21 | TP collective overhead dominates | collective timing |
| F22 | rollout mixes model revisions | manifest |
| F23 | HTTP 200 partial SSE | completion validator |
| F24 | GPU cost per valid request rises | FinOps |

## 15. Sixteen operational runbooks

1. Verify cluster context, release and rank topology.
2. Diagnose GPU scheduling and quotas.
3. Validate GPU UUID/rank mapping.
4. Investigate model load and per-rank OOM.
5. Investigate KV cache/rank imbalance.
6. Validate driver/CUDA/NCCL compatibility.
7. Diagnose NCCL initialization.
8. Diagnose collective timeout and rank failure.
9. Diagnose PCIe/NVLink locality.
10. Diagnose cross-node fabric and interface.
11. Audit coordinator/network security.
12. Diagnose TP scaling efficiency.
13. Diagnose PP stage imbalance.
14. Validate full-stream SLO and GPU FinOps.
15. Execute approved rollback and worker cleanup.
16. Verify sustained recovery and RCA/COA.

## 16. Eight tabletop game days

1. One GPU rank crashes during decode.
2. NCCL initialization fails after rollout.
3. Cross-node network latency increases TPOT.
4. Wrong TP/PP sizes leave ranks unassigned.
5. Pipeline stage imbalance causes idle GPUs.
6. One rank OOMs while others remain healthy.
7. Coordinator port becomes unintentionally exposed.
8. TP4 costs more per completed request than an alternative.

Default **tabletop**. Fault injection requires separately approved staging, bounded network/GPU costs and rollback.

## 17. Twenty acceptance gates

G01 pinned runtime/model; G02 GPU inventory; G03 TP/PP architecture; G04 rank mapping; G05 GPU scheduling; G06 topology/NIC; G07 CUDA/NCCL compatibility; G08 coordinator security; G09 per-rank VRAM; G10 KV allocation; G11 NCCL diagnostics; G12 PP stage balance; G13 TP scaling; G14 complete SSE; G15 TTFT/TPOT; G16 failure-domain design; G17 safe staging; G18 rollback; G19 FinOps; G20 independent review.

See [companion lab](../labs/021-vllm-tensor-pipeline-parallelism-distributed-serving-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 022 — vLLM Security, Authentication and Network Isolation.**
