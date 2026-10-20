# Lab 078 — NCCL Hangs and Distributed Deadlocks Acceptance Gate

**Chapter:** [078 — NCCL Hangs and Distributed Deadlocks](../part-10-incident-response-and-troubleshooting/078-nccl-hangs-distributed-deadlocks.md)

**Status:** NOT EXECUTED. Staging, offline exercises and independent review are separate pending milestones.

## 1. Safety

Run collectives only on approved dedicated staging GPUs. Never intentionally deadlock, kill ranks, block RDMA traffic, modify MTU/PFC or force distributed-job restarts in production. Obtain job owner approval, checkpoint assurance, timeout budget, and rollback plan. Use synthetic log fixtures for unsafe failure modes.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | Version inventory | PyTorch/CUDA/NCCL/driver/launcher | Pending |
| E02 | Rank topology | rank/pod/node/GPU/world-size matrix | Pending |
| E03 | UTC incident timeline | earliest failure by rank | Pending |
| E04 | Local NCCL smoke | 2-GPU all-reduce logs | Pending |
| E05 | Multi-node smoke | staged transport evidence or N/A | Pending |
| E06 | GPU/NIC topology | placement and affinity map | Pending |
| E07 | Fabric counters | RDMA/TCP errors and drops | Pending |
| E08 | Synthetic collective mismatch | offline trace analysis | Pending |
| E09 | Synthetic missing rank | rendezvous diagnosis | Pending |
| E10 | Data-loader straggler | per-rank latency | Pending |
| E11 | Xid/CUDA correlation | earliest-error timeline | Pending |
| E12 | Checkpoint pause | storage and barrier evidence | Pending |
| E13 | Diagnostic logging | version, log volume, redaction | Pending |
| E14 | Rank-loss tabletop | verified checkpoint restart plan | Pending |
| E15 | Training stall alerts | synthetic receiver delivery | Pending |
| E16 | Acceptance | RCA, reviewer, score | Pending |

## 3. Read-only incident collection

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> get pods -o wide -l <job-label-selector>
kubectl -n <namespace> logs <rank-pod> --timestamps=true --since=2h
kubectl -n <namespace> describe pod <rank-pod>
\`\`\`

Approved GPU/network host only:

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version --format=csv
ip -s link
ss -tan
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|mlx|rdma|AER'
\`\`\`

## 4. Staging NCCL smoke test

Create \`nccl_smoke.py\` in a version-pinned staging PyTorch image:

\`\`\`python
import os
from datetime import timedelta
import torch
import torch.distributed as dist

rank = int(os.environ["RANK"])
local_rank = int(os.environ["LOCAL_RANK"])
torch.cuda.set_device(local_rank)
dist.init_process_group("nccl", timeout=timedelta(seconds=120))
try:
    x = torch.tensor([float(rank + 1)], device=f"cuda:{local_rank}")
    dist.all_reduce(x, op=dist.ReduceOp.SUM)
    expected = dist.get_world_size() * (dist.get_world_size() + 1) / 2
    torch.cuda.synchronize()
    assert x.item() == expected
    print(f"rank={rank}, sum={x.item()}", flush=True)
finally:
    dist.destroy_process_group()
\`\`\`

Run only on a staging node with two allocated GPUs:

\`\`\`bash
torchrun --standalone --nnodes=1 --nproc-per-node=2 nccl_smoke.py
\`\`\`

Expected \`sum=3.0\` from each rank. Record version, GPU UUIDs, runtime duration, output and exit status. This **does not** test cross-node fabric.

## 5. Synthetic fault-analysis fixtures

**A — Collective mismatch:** rank 0 executes \`all_reduce\`; rank 1 skips it due to a conditional. Identify code-path divergence and propose consistent collective participation.

**B — Rendezvous failure:** rank 2 never joins because DNS/port policy is wrong. Determine first network/configuration failure rather than increasing the collective timeout.

**C — Xid precedes NCCL timeout:** one GPU logs a serious NVRM Xid, then peers time out. Establish timeline and checkpoint-safe recovery.

**D — Checkpoint stall:** one rank waits on object storage while others enter a barrier. Identify storage/coordination cause.

**E — Fabric congestion:** NCCL all-reduce latency increases across nodes with NIC drops; compare GPU/NIC affinity and network counters before any tuning.

## 6. Evidence record

\`\`\`yaml
chapter: "078"
environment: "STAGING_OR_OFFLINE"
execution_date_utc: null
reviewer: null
pytorch_version: null
cuda_version: null
nccl_version: null
driver_version: null
world_size: null
rank_to_node_gpu_mapping: {}
local_smoke: "NOT_RUN"
multi_node_smoke: "NOT_RUN"
collective_mismatch_analysis: "NOT_RUN"
fabric_health_analysis: "NOT_RUN"
checkpoint_integrity_review: "PENDING"
staging_alert_delivery: "NOT_RUN"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 7. Acceptance

Score Chapter 078 gates **20 + 20 + 20 + 20 + 10 + 10 = 100**. Pass at **≥85** with no critical data loss, unsafe rank termination or security failure. Mark offline tabletop separately from staging execution; do not claim multi-node validation based on the local two-GPU smoke test.

**Next:** Chapter 079 — Network and Storage Bottleneck Incidents.
