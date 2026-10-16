# Chapter 021 — Hands-On Lab: vLLM Tensor/Pipeline Parallelism and Distributed Serving

**Part 04 | Chapter 021 | Published lab plan; execution pending**

## Goal

Design TP/PP rank topology, map GPU and network placement, validate per-rank memory and NCCL failure domains, plan safe distributed serving tests and produce SLO/FinOps acceptance evidence.

## Safety

**[SAFE-READ]** approved Kubernetes/GPU metadata and redacted logs. **[LAB-MUTATE]** separately approved isolated staging distributed serving with bounded GPU-hours and network access. **[CHANGE-CONTROL]** production rank mapping, distributed executor, firewall, GPU nodes and serving routes. No PHI, customer prompts, secrets, privileged debug or public coordination endpoints.

## Lab 0 — Preflight

Record UTC, cluster/context, GPU SKU/count/UUID, node topology, driver/CUDA/NCCL, vLLM image/version, model/tokenizer, executor, TP/PP sizes, network fabric, owner, cost ceiling and rollback.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — TP/PP mapping [DESIGN]

Draw and compare:
- TP4/PP1: four GPUs, one stage.
- TP2/PP2: four GPUs, two stages.
- TP4/PP2: eight GPUs, two stages.

For each, record node count, GPU mapping, collectives, pipeline boundaries, memory and failure domain. Verify model/runtime support; these are synthetic layouts.

## Lab 2 — GPU topology [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Capture PCIe/NVLink topology and NIC locality where exposed. Do not infer direct links from GPU SKU.

## Lab 3 — Distributed security [DESIGN]

Document trusted coordination membership, identity, restricted ports, network policies, approved model artifact access, immutable image digests and redacted logs. Identify the owner of each exposed control-plane or worker interface. No open Ray/NCCL coordination interfaces to the public internet.

## Lab 4 — Per-rank memory [DESIGN]

Estimate weights, KV, workspace, communication buffers and headroom for each rank. Identify the **maximum** rank, not only cluster aggregate. Describe why a single-rank OOM can stall or fail a distributed group.

## Lab 5 — Synthetic scaling [DESIGN]

```text
single-GPU step = 80 ms
TP4 ideal compute = 20 ms
collectives/coordination = 18 ms
TP4 step ≈ 38 ms
speedup ≈ 2.11×
efficiency ≈ 52.6%
```

This is illustrative, not measured. Explain why model fit, PP bubbles and network locality can make direct comparison invalid.

## Lab 6 — Safe staging benchmark [DESIGN; optional approval]

Define short/long prompt and output cohorts, bounded concurrency, TTFT/TPOT, complete SSE, per-rank VRAM, collective time, stage utilization, NCCL errors, GPU-hours and abort thresholds. Use synthetic requests only. Do not execute a multi-node stress test without written approval.

## Lab 7 — Eight tabletop game days

01 rank crash; 02 NCCL init; 03 fabric latency; 04 TP/PP mismatch; 05 PP imbalance; 06 rank OOM; 07 coordinator exposure; 08 cost regression.

## Twenty-four failure checklist

F01 GPU count; F02 TP size; F03 PP size; F04 rank mapping; F05 mixed SKU; F06 driver; F07 NCCL version; F08 init; F09 collective; F10 rank crash; F11 rank OOM; F12 KV; F13 PCIe; F14 NVLink; F15 fabric; F16 interface; F17 port; F18 security; F19 stage; F20 bubble; F21 TP overhead; F22 revision; F23 SSE; F24 cost.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned runtime/model | Pending |
| G02 | GPU inventory | Pending |
| G03 | TP/PP design | Pending |
| G04 | Rank mapping | Pending |
| G05 | GPU scheduling | Pending |
| G06 | Topology/NIC | Pending |
| G07 | CUDA/NCCL compatibility | Pending |
| G08 | Coordinator security | Pending |
| G09 | Per-rank VRAM | Pending |
| G10 | KV allocation | Pending |
| G11 | NCCL diagnostics | Pending |
| G12 | PP stage balance | Pending |
| G13 | TP scaling | Pending |
| G14 | Complete SSE | Pending |
| G15 | TTFT/TPOT | Pending |
| G16 | Failure domains | Pending |
| G17 | Safe staging plan | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/authorization:
vLLM/model/image/driver/CUDA/NCCL:
TP/PP and rank-to-GPU map:
Node/GPU/NIC topology:
Coordinator security:
Per-rank weights/KV/workspace:
TTFT/TPOT/SSE and collective timing:
Failure domain and GPU-hours:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. For approved staging, stop load, terminate only owned workers via the authoritative controller, verify no orphan GPU allocations and reconcile GPU-hours. Acceptance gates remain pending until reviewed.

**Next: Chapter 022 — vLLM Security, Authentication and Network Isolation.**
