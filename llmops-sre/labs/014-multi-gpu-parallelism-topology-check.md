# Chapter 014 — Hands-On Lab: Multi-GPU Parallelism and Topology

**Part 03 | Chapter 014 | Published lab plan; execution pending**

## Goal

Map GPU topology, compare tensor/pipeline/data/expert parallelism, estimate collective overhead, design per-rank memory and reliability checks, and prepare an NCCL incident response worksheet.

## Safety

**[SAFE-READ]** approved GPU inventory, Kubernetes metadata and redacted dashboards. **[LAB-MUTATE]** separately authorized staging synthetic multi-GPU inference. **[CHANGE-CONTROL]** production rank mapping, GPU node pools, networking, driver, NCCL, routing or model changes. Do not run distributed stress tests on production.

## Lab 0 — Preflight

Record UTC, context/namespace, node/GPU SKUs, driver/CUDA/NCCL/runtime/image digests, GPU topology, network fabric, rank mapping, model revision, owner, budget and reviewer.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — GPU topology [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Record GPU-to-GPU links, host/NUMA locality and NIC relationships where shown. If unavailable, mark blocked; do not add privileges.

## Lab 2 — Parallelism architecture [DESIGN]

Draw:
- TP: matrix shards and collectives;
- PP: layers and stage queues;
- DP: full replicas and routing;
- EP: experts and all-to-all.

Identify model fit, throughput, TTFT/TPOT, GPU-hours and failure-domain tradeoffs.

## Lab 3 — Synthetic scaling arithmetic [DESIGN]

```text
single GPU step = 80 ms
ideal TP4 compute = 80/4 = 20 ms
communication/coordination = 18 ms
TP4 step = 38 ms
speedup = 80/38 ≈ 2.11×
parallel efficiency = 2.11/4 ≈ 52.6%
```

State why the arithmetic is illustrative and not a measured benchmark.

## Lab 4 — Per-rank VRAM [DESIGN]

For each hypothetical rank, allocate weight shard, KV-cache shard, workspace, NCCL buffers, allocator reserve and headroom. Record the highest rank rather than using only total VRAM. Explain why one rank OOM fails the group.

## Lab 5 — NCCL and scheduling evidence [SAFE-READ]

```bash
kubectl describe nodes
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Review authorized rank logs and dashboards for collective timeout, transport errors, stage imbalance, replica queue, GPU memory and stream completion. Redact sensitive data. Do not enable verbose NCCL logging without approval.

## Lab 6 — Staging benchmark plan [DESIGN]

Define one supported configuration comparison (for example TP2 vs TP4 or DP1 vs DP2), with same model/tokenizer/workload/GPU SKU, bounded synthetic token cohorts, warmup, TTFT/TPOT, complete-stream success, p99, rank VRAM, collectives, GPU-hours, abort thresholds and rollback. Execution is optional and separately approved.

## Lab 7 — Eight tabletop game days

01 rank loss; 02 PCIe topology regression; 03 NCCL init; 04 fabric congestion; 05 PP imbalance; 06 DP skew; 07 expert hotspot; 08 GPU cost regression.

## Twenty-scenario checklist

F01 GPU shortage; F02 rank mapping; F03 mixed SKU; F04 PCIe; F05 NVLink; F06 fabric; F07 NCCL init; F08 collective timeout; F09 rank crash; F10 TP efficiency; F11 PP imbalance; F12 bubbles; F13 DP skew; F14 EP hotspot; F15 all-to-all; F16 rank OOM; F17 NUMA; F18 mixed versions; F19 node failure; F20 cost.

## Eighteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | TP/PP/DP/EP distinctions | Pending |
| G02 | GPU topology inventory | Pending |
| G03 | PCIe/NVLink/fabric | Pending |
| G04 | Rank mapping | Pending |
| G05 | TP collective analysis | Pending |
| G06 | PP bubble analysis | Pending |
| G07 | DP routing | Pending |
| G08 | EP routing | Pending |
| G09 | Per-rank VRAM | Pending |
| G10 | NCCL diagnostics | Pending |
| G11 | Kubernetes placement | Pending |
| G12 | Failure-domain analysis | Pending |
| G13 | TTFT/TPOT/SSE | Pending |
| G14 | Scaling math | Pending |
| G15 | GPU FinOps | Pending |
| G16 | Safe staging plan | Pending |
| G17 | Rollback/recovery | Pending |
| G18 | Reviewer signoff | Pending |

## Evidence template

```text
UTC/context/approval:
GPU SKU/topology/NUMA/NIC:
Driver/CUDA/NCCL/runtime:
TP/PP/DP/EP and rank map:
Collective timing assumptions:
Per-rank weights/KV/workspace:
Queue/TTFT/TPOT/SSE and GPU-hours:
Game-day mode and findings:
Rollback, owner, reviewer, acceptance:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop any separately approved staging load, remove only owned resources and reconcile GPU cost. All gates remain pending until independently reviewed.

**Next: Chapter 015 — GPU Diagnostics, DCGM and Failure Lab.**
