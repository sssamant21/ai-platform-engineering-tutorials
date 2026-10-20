# Chapter 078 — NCCL Hangs and Distributed Deadlocks

**Part 10 — Incident Response and Troubleshooting | AI Infrastructure Engineering & SRE**

**Publication:** Documentation published; offline execution, live GPU staging validation, security review and independent production acceptance remain pending.

## 1. Mission

Diagnose distributed PyTorch/NCCL training jobs that stall, hang, time out, diverge across ranks or abort after a peer fails. Distinguish a **collective mismatch**, **slow or failed rank**, **GPU/driver error**, **network fabric failure**, **CPU/data-loader starvation**, **checkpoint/storage stall** and **process-group lifecycle bug**. Recover without losing verified checkpoints or disrupting unrelated GPU tenants.

**Prerequisites:** Chapters 075–077, distributed training fundamentals, NCCL and PyTorch \`torch.distributed\`, Kubernetes GPU operations. All disruptive tests run in isolated staging only.

## 2. Collective semantics and why jobs hang

Distributed data parallelism requires all participating ranks in a process group to enter compatible collective operations in consistent order, with matching tensor shapes/dtypes where required. A single rank skipping a collective can leave peers waiting until a watchdog or process-group timeout. A timeout is a **symptom**; the first failing rank or earliest divergence may be elsewhere.

\`\`\`text
Kubernetes scheduler / launcher (torchrun, Job, operator)
           ↓
    rank/world-size/rendezvous
           ↓
   CUDA kernels and per-rank data loaders
           ↓
   torch.distributed collective order
           ↓
   NCCL transport: NVLink/NVSwitch/PCIe, TCP/IB/RoCE
           ↓
   network fabric, NIC/driver, topology, GPU health
\`\`\`

A deadlock-like stall may be logical (different code paths across ranks), environmental (a rank lost network access) or performance-related (a straggler never reaches the collective). Do not diagnose “NCCL bug” solely from a timeout log.

## 3. Failure taxonomy

| Symptom | Likely cause | First evidence |
|---|---|---|
| \`Watchdog caught collective operation timeout\` | rank divergence, peer crash, fabric delay, GPU stall | earliest rank logs and operation sequence |
| \`NCCL WARN NET/IB\` / transport errors | RDMA device, GID, MTU, routing, congestion | NIC/RDMA counters and network events |
| \`unhandled cuda error\` in NCCL | earlier CUDA fault, OOM, device failure | first CUDA exception and Xid logs |
| \`Connection refused\` / rendezvous timeout | DNS, service, port, firewall, launcher config | endpoint resolution, events, process group |
| One rank stalls in data loading | storage/network/data-loader exhaustion | per-rank data-loader and I/O timing |
| Hang after checkpoint | slow storage, rank-specific filesystem error, barriers | checkpoint and filesystem events |
| \`invalid usage\` / collective mismatch | inconsistent operation order, tensor contract | code path and collective sequence |
| Job restarts repeatedly | launcher retry policy, failed rank, resource pressure | controller and pod restart history |
| Slow all-reduce but no timeout | fabric congestion, topology, message size | per-rank timing and NIC utilization |

NCCL diagnostic strings and behavior vary by NCCL, CUDA and PyTorch release. Always record exact versions and error codes.

## 4. First 15 minutes: preserve evidence

### 4.1 Kubernetes blast radius

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> get pods -o wide -l <job-label-selector>
kubectl -n <namespace> describe pod <rank-pod>
kubectl -n <namespace> logs <rank-pod> --all-containers=true --timestamps=true --since=2h
kubectl -n <namespace> logs <rank-pod> --previous --timestamps=true
\`\`\`

Capture job/controller UID, pod UIDs, node placement, GPU/MIG identity, image digests, world size, global/local rank, rendezvous endpoint, checkpoint location and timestamp. Collect logs from **all ranks** before any restart. \`--previous\` may be unavailable.

### 4.2 GPU, host and fabric evidence

Approved host access only:

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,memory.used --format=csv
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|nvidia|AER|mlx|rdma'
ip -s link
ss -tan
\`\`\`

For RDMA/RoCE or InfiniBand, use approved read-only tools (\`ibstat\`, \`rdma link\`, \`ethtool -S\`) if installed. Counters and meanings are device-specific. Avoid active stress tests or changing NIC/MTU/PFC settings during a live incident without change approval.

### 4.3 Correlation timeline

Normalize all timestamps to UTC; account for clock skew and buffered asynchronous logging. Build a table per rank: last successful training step, last completed collective, first CUDA/NCCL error, node/GPU event, network counters, checkpoint status and exit/restart. A later watchdog timeout is not necessarily the root cause.

## 5. Controlled NCCL diagnostic configuration

Use diagnostic environment variables only on **approved staging** jobs or bounded incident reproductions:

\`\`\`bash
# Illustrative settings; verify against installed NCCL/PyTorch versions.
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,COLL,NET
export TORCH_DISTRIBUTED_DEBUG=DETAIL
\`\`\`

Increased logging can be expensive, reveal topology or identifiers and alter timing. Remove it after diagnosis. Some PyTorch builds expose flight-recorder/trace-buffer features for NCCL process groups; exact \`TORCH_NCCL_*\` variables and supported semantics are version-specific. Confirm official documentation and deployment version before enabling.

**Do not** leave \`NCCL_DEBUG=TRACE\` or verbose collective logging fleet-wide. Never use \`NCCL_IB_DISABLE=1\`, \`NCCL_P2P_DISABLE=1\` or network interface overrides as a production “fix” without proving the affected transport and measuring performance; these can mask the root cause and significantly degrade training.

## 6. Collective-order correctness

### 6.1 Typical mismatch

\`\`\`python
# INCORRECT: only rank 0 enters all_reduce
if rank == 0:
    dist.all_reduce(tensor)
\`\`\`

All participating ranks must enter the same collective in compatible order:

\`\`\`python
# CORRECT: every rank in the process group participates
dist.all_reduce(tensor)
if rank == 0:
    print("reduced result:", tensor.item())
\`\`\`

Do not use unconditional \`dist.barrier()\` as a universal repair: it can create a new hang if a rank failed or skipped it. Asynchronous CUDA operations may surface errors at a later synchronization point.

### 6.2 Safe minimal staging smoke test

\`\`\`python
# file: nccl_smoke.py — run ONLY in a staging job with 2 GPUs
import os
from datetime import timedelta
import torch
import torch.distributed as dist

rank = int(os.environ["RANK"])
local_rank = int(os.environ["LOCAL_RANK"])
torch.cuda.set_device(local_rank)
dist.init_process_group(backend="nccl", timeout=timedelta(seconds=120))
try:
    x = torch.tensor([float(rank + 1)], device=f"cuda:{local_rank}")
    dist.all_reduce(x, op=dist.ReduceOp.SUM)
    expected = dist.get_world_size() * (dist.get_world_size() + 1) / 2
    torch.cuda.synchronize()
    assert x.item() == expected, (x.item(), expected)
    print(f"rank={rank} world_size={dist.get_world_size()} sum={x.item()}", flush=True)
finally:
    dist.destroy_process_group()
\`\`\`

Example staging command:

\`\`\`bash
# Two GPUs on one approved staging node; verify device availability first.
torchrun --standalone --nnodes=1 --nproc-per-node=2 nccl_smoke.py
\`\`\`

This verifies basic local NCCL collective operation; it does **not** validate cross-node fabric, topology, sustained throughput or production-scale reliability.

## 7. Multi-node networking and topology

For multi-node collectives, verify:
- GPU-to-NIC affinity, PCIe/NVLink/NVSwitch topology, NUMA placement and host network path.
- RDMA device visibility and configuration, CNI/host-network requirements, GID selection, MTU, routing and ACLs.
- TCP rendezvous endpoint DNS, firewall/NetworkPolicy and consistent \`MASTER_ADDR\`, \`MASTER_PORT\`, \`RANK\`, \`WORLD_SIZE\`.
- NIC drops, retransmits, link flaps, congestion and packet loss in the relevant time window.
- Node time synchronization, rank-specific throughput and checkpoint/storage load.

\`nccl-tests\` (e.g. \`all_reduce_perf\`) can help characterize transport **only in dedicated staging**, using version-matched builds and an approved workload limit. A healthy bandwidth benchmark does not prove application collective-order correctness.

## 8. Rank failure and checkpoint integrity

A failed rank may cause all other ranks to hang or abort. **Do not kill individual ranks in production to “unstick” the job** without verifying the distributed training framework's failure semantics. Preserve the last **complete, verified** checkpoint (including optimizer state, scheduler state, RNG/shard metadata as required). Validate checkpoint consistency and object-store commit markers before resume.

For elastic training, confirm whether world-size changes are supported by the model, sharding strategy and checkpoint format; do not assume all distributed jobs can resize safely.

## 9. Triage decision tree

\`\`\`text
All ranks present and rendezvous complete?
 ├─ No → scheduler, pod crash, DNS, port, auth, launcher
 └─ Yes → did ranks reach same collective sequence?
          ├─ No → code path, data loader, exception, checkpoint stall
          └─ Yes → any CUDA/Xid/OOM on a rank?
                   ├─ Yes → device/driver/application failure
                   └─ No → transport/fabric/topology/congestion,
                            backend and timeout configuration
\`\`\`

Increasing timeouts is not a root-cause fix. It may be appropriate only after measuring legitimate long-running collectives/checkpoint pauses and verifying no stalled ranks.

## 10. Sixteen hands-on exercises

1. Inventory PyTorch, CUDA, NCCL, driver and launcher versions.
2. Map ranks, world size, pod UIDs, nodes, GPUs and rendezvous.
3. Capture a normalized UTC timeline from all rank logs.
4. Run the two-GPU staging all-reduce smoke test.
5. Validate a staging multi-node smoke test where available.
6. Compare topology and GPU/NIC affinity.
7. Review RDMA/TCP fabric health and relevant counters.
8. Reproduce a **synthetic** mismatched collective order using offline trace fixtures.
9. Reproduce a **synthetic** missing-rank rendezvous scenario using logs/tabletop.
10. Analyze a data-loader straggler that delays a collective.
11. Analyze an Xid/CUDA error preceding NCCL timeout.
12. Analyze a checkpoint/storage pause and recovery.
13. Validate bounded staging diagnostic logging and safe redaction.
14. Tabletop rank loss and verified checkpoint restart.
15. Verify alerting for training progress stall, rank loss and network errors.
16. Produce incident report, runbooks, evidence and acceptance score.

## 11. Troubleshooting matrix

| Symptom | Inspect first | Avoid |
|---|---|---|
| NCCL watchdog timeout | earliest rank error, collective sequence | timeout-only “fix” |
| Hang after checkpoint | storage, barrier, rank timing | forced delete without checkpoint check |
| One rank OOM | per-rank memory and batch | blaming network |
| Rendezvous timeout | DNS/port/rank/world-size | changing GPU driver |
| IB transport error | RDMA link, GID, ACL, counters | globally disabling IB |
| All-reduce slow | topology, congestion, GPU/NIC affinity | assuming deadlock |
| Xid before NCCL failure | GPU/kernel evidence | resetting GPU automatically |
| Incomplete rank logs | logging gaps, pod restarts | claiming definitive RCA |

## 12. Production runbooks

**A — Collective timeout:** preserve all rank logs and sequence IDs; identify earliest divergent rank or error; classify code, CUDA, storage or fabric; recover from verified checkpoint with owner approval.

**B — Rendezvous failure:** verify launcher config, rank/world-size, endpoint DNS/ports, NetworkPolicies and pod scheduling; test isolated staging connectivity; restore matching configuration.

**C — RDMA/transport degradation:** correlate NIC/RDMA errors, GPU/NIC topology and link events; engage network team; validate staged network changes and throughput before rollout.

**D — Rank crash and checkpoint restore:** verify checkpoint completeness and framework compatibility; coordinate clean distributed-job restart; validate loss/step continuity and SLO.

**E — Data-loader/checkpoint straggler:** compare per-rank input and storage latencies, CPU throttling, I/O and barriers; fix source bottleneck; benchmark training step variance.

## 13. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Rank topology and evidence capture | 20 |
| Collective semantics and code diagnosis | 20 |
| Network/RDMA/GPU root-cause isolation | 20 |
| Checkpoint/recovery safety | 20 |
| Staging smoke tests and alerting | 10 |
| Runbooks, peer review and RCA quality | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical data-integrity, security, unsafe-job-termination or unapproved-production-impact failure. Written publication does not imply the tests ran.

## 14. References

- [PyTorch Distributed Communication Package](https://docs.pytorch.org/docs/stable/distributed.html)
- [PyTorch ProcessGroupNCCL Environment Variables](https://docs.pytorch.org/docs/stable/torch_nccl_environment_variables.html)
- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA NCCL Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html)
- [NVIDIA NCCL Tests](https://github.com/NVIDIA/nccl-tests)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)

**Next:** Chapter 079 — Network and Storage Bottleneck Incidents.
