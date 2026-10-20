# Chapter 062 — PyTorch Distributed and NCCL

**AI Infrastructure Engineering & SRE · Part 08 — Distributed Training Operations · Chapter 062/100**

> **Validation boundary:** Production-oriented tutorial and offline synthetic acceptance gate. No live GPU, NCCL, PyTorch multi-node or cloud failure injection is implied.

## Objectives and prerequisites

Build and operate multi-node PyTorch distributed training with DistributedDataParallel (DDP), Fully Sharded Data Parallel (FSDP), `torchrun`, process groups, NCCL collectives, GPU/NIC affinity, Kubernetes GPU scheduling, dataset sharding, mixed precision, checkpoint recovery and SRE observability. Diagnose collective hangs without unsafe production changes.

**Prerequisites:** Chapters 017–025, 043–050, 055–056 and 059–061; Linux GPU hosts, CUDA/NCCL, Kubernetes and RDMA/EFA networking.

## 1. Distributed training architecture

```text
approved image + pinned PyTorch/CUDA/NCCL + dataset
                         |
             scheduler / gang admission
                         |
            torchrun rendezvous + membership
                         |
     +-------------------+-------------------+
     | node A ranks 0..7 | node B ranks 8..15|
     | GPU <-> NVLink    | GPU <-> NVLink    |
     | NIC <-> PCIe/NUMA | NIC <-> PCIe/NUMA |
     +---------RDMA/EFA/IB/RoCE fabric-------+
                         |
           NCCL all-reduce / all-gather
             reduce-scatter / broadcast
                         |
      model/optimizer/RNG/sampler checkpoint
                         |
         durable manifest + isolated restore
                         |
       DCGM / NIC metrics / NCCL logs / SLO
```

The rendezvous service, process-group lifecycle, data pipeline and checkpoint archive are separate failure domains. A successful `nvidia-smi` does not establish distributed training readiness.

## 2. Core process-group concepts

- **Global rank:** unique process identifier within the process group.
- **Local rank:** process index within a node, normally used to select the assigned GPU.
- **World size:** number of participating processes.
- **Backend:** NCCL for common CUDA GPU collective operations; Gloo for selected CPU/control scenarios.
- **Collective:** an operation requiring compatible participation and ordering among the intended ranks.
- **Rendezvous:** coordinated membership/initialization; elastic membership behavior depends on launch and training design.

A conventional one-process-per-GPU DDP run requires coherent rank geometry and exactly one assigned GPU per process. Ensure the selected distributed training operator and launcher do not double-spawn processes.

## 3. Minimal torchrun-compatible DDP program

```python
# ddp_smoke.py — example requires supported CUDA GPUs and torchrun.
import os
import torch
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

def main():
    dist.init_process_group(backend="nccl")
    local_rank = int(os.environ["LOCAL_RANK"])
    torch.cuda.set_device(local_rank)
    try:
        model = torch.nn.Linear(8, 4).to(local_rank)
        ddp = DDP(model, device_ids=[local_rank])
        optimizer = torch.optim.SGD(ddp.parameters(), lr=0.01)
        x = torch.randn(16, 8, device=f"cuda:{local_rank}")
        loss = ddp(x).square().mean()
        loss.backward()
        optimizer.step()
        if dist.get_rank() == 0:
            print("DDP smoke step completed")
    finally:
        dist.destroy_process_group()

if __name__ == "__main__":
    main()
```

```bash
# Single-node staging example with two visible GPUs:
torchrun --standalone --nnodes=1 --nproc-per-node=2 ddp_smoke.py
```

This checks a small collective path, not dataset correctness, multi-node networking or production stability. Run only on an authorized GPU test host.

## 4. DDP vs FSDP and collective patterns

| Mode | Memory | Communication | Reliability risk |
|---|---|---|---|
| DDP | full model replica/rank | gradient all-reduce | straggler/hang and redundant state |
| FSDP | sharded parameters/gradients/optimizer | all-gather + reduce-scatter | shard/reshard compatibility |
| Tensor parallel | partitioned tensors | intra-layer collectives | latency-sensitive topology |
| Pipeline parallel | stage-local parameters | activation transfers | bubble and stage failure |

Choose based on model size, GPU memory, network bisection bandwidth, training framework support and verified checkpoint restore. Do not assume a FSDP checkpoint can be safely loaded with any arbitrary world size.

## 5. NCCL topology and performance

Compare NVLink/NVSwitch, PCIe root complex, NIC NUMA affinity, EFA/InfiniBand/RoCE capabilities, interface naming, MTU and network congestion. Select NCCL network environment settings only after identifying the correct backend and fabric; forcing `NCCL_SOCKET_IFNAME` is not a universal fix for RDMA transports.

Read-only inspection:

```bash
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,utilization.gpu,memory.used --format=csv
lspci | grep -Ei 'nvidia|ethernet|infiniband'
ip -br link
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

Capture per-rank timestamps, NCCL version, first failing collective, GPU Xid, NIC error counters, step-time breakdown and topology. Enable verbose NCCL logging temporarily and scope log volume.

## 6. Global batch, throughput and scaling

```text
global_batch = per_rank_microbatch * gradient_accumulation * data_parallel_replicas
scaling_efficiency = observed_samples_per_second / (world_size * one_gpu_samples_per_second)
collective_fraction = collective_p95_ms / step_p95_ms
RPO = incident_time - last_verified_durable_checkpoint_time
```

**Synthetic example:** 16 ranks × microbatch 2 × accumulation 8 = global batch 256. If one GPU yields 100 samples/s and 16 GPUs yield 1,200 samples/s, scaling efficiency is 75%. Validate equivalence of workload, precision, batch semantics and convergence.

## 7. Dataset and checkpoint consistency

Use a pinned immutable dataset manifest, rank-aware sampler and stable epoch/cursor semantics. Save model, optimizer, scheduler, RNG, GradScaler where applicable, sampler state, global step and a committed distributed checkpoint manifest. Restore into an isolated staging environment, verify hashes/shards and replay correctness, then resume only under supported topology and version constraints.

## 8. Reliability, Kubernetes and security

Use GPU resource limits, device-plugin isolation, topology-aware scheduling, quotas, admission controls, network policy and workload identity. A single rank failure often requires coordinated group recovery. Fence stale rendezvous membership and protect checkpoint RPO/RTO before preemption or rolling changes. Distinguish driver errors, CUDA OOM, NCCL transport faults and application collective ordering bugs.

## 9. Observability and SLOs

Track GPU utilization/memory/Xid, NIC errors, collective p95/p99, rank skew, data wait, process-group init time, watchdog timeouts, failed steps, checkpoint age, restore duration and cost per successful step. Correlate job ID, rank, node, GPU UUID, NIC, dataset release, image digest and UTC. Stale telemetry is unknown, not healthy.

## 10. Fifteen troubleshooting scenarios

1. NCCL collective hangs when one rank crashes.
2. All-reduce p99 increases after network placement change.
3. CUDA initialization fails on one worker.
4. NCCL chooses the wrong NIC interface.
5. EFA/RDMA plugin unavailable on a subset of nodes.
6. GPU/NIC NUMA mismatch increases collective latency.
7. World-size mismatch causes rendezvous timeout.
8. Collective calls execute in different order across ranks.
9. DistributedSampler repeats data after restart.
10. DDP gradient buckets cause memory pressure.
11. FSDP shard restore fails after world-size change.
12. Mixed precision overflow diverges optimizer state.
13. Checkpoint manifest has missing rank shards.
14. NCCL watchdog reports async error after link flap.
15. Stale telemetry masks a collective performance regression.

For each record symptom, first failed rank, SLI, read-only inspection, containment, recovery, rollback and evidence.

## 11. Ten incident simulations

1. **INC-062-01:** Rank loss mid-all-reduce.
2. **INC-062-02:** Rendezvous endpoint failure.
3. **INC-062-03:** Wrong NIC selection.
4. **INC-062-04:** RDMA/EFA interface degradation.
5. **INC-062-05:** GPU Xid during collective.
6. **INC-062-06:** Collective ordering divergence.
7. **INC-062-07:** FSDP checkpoint shard loss.
8. **INC-062-08:** Mixed precision overflow.
9. **INC-062-09:** Worker preemption and replay.
10. **INC-062-10:** Monitoring outage during NCCL timeout.

Run disruptive drills only in authorized staging. Record incident commander, UTC timeline, job/rank impact, checkpoint RPO, restore RTO, GPU-hours lost and RCA/COA.

## 12. Eight production runbooks

1. **RB-062-A — Rank and rendezvous triage:** Verify global/local rank, world size, rendezvous ID, fencing and logs.
2. **RB-062-B — NCCL hang:** Identify first failed rank, collective sequence, watchdog, NIC counters and GPU health.
3. **RB-062-C — GPU/NIC topology:** Compare PCIe/NUMA affinity, NVLink, EFA/IB/RoCE and placement.
4. **RB-062-D — Driver and framework compatibility:** Verify image digest, PyTorch, CUDA runtime, driver and NCCL versions.
5. **RB-062-E — DDP/FSDP memory:** Inspect batch size, gradient buckets, activation memory and shard ownership.
6. **RB-062-F — Dataset and checkpoint recovery:** Verify immutable dataset, sampler cursor, optimizer, RNG and global commit.
7. **RB-062-G — Security and isolation:** Validate device isolation, workload identity, network policy and tenant boundaries.
8. **RB-062-H — Incident closure:** Measure step SLO, RPO/RTO, rollback and independent review.

## 13. Twelve hands-on labs

1. Map world size, global/local rank and GPU device assignments.
2. Run single-node DDP smoke test in approved GPU staging.
3. Inspect GPU-to-NIC topology and NCCL transport dependencies.
4. Compare DDP and FSDP communication and memory tradeoffs.
5. Calculate global batch and scaling efficiency.
6. Model collective latency and degraded fabric bandwidth.
7. Validate immutable dataset and sampler replay.
8. Review globally consistent FSDP/DDP checkpoint state.
9. Simulate rank failure and rendezvous fencing.
10. Simulate NCCL timeout, network skew and CUDA incompatibility.
11. Run companion offline policy gate and pytest suite.
12. Complete incident evidence, rollback and enterprise readiness review.

**Companion:** [Lab 062 — PyTorch Distributed and NCCL Gate](../labs/062-pytorch-distributed-nccl-check.md).

## 14. Enterprise acceptance — 100 points

| Domain | Points |
|---|---:|
| Process groups, rank geometry and DDP/FSDP | 20 |
| NCCL fabric, collective performance and GPU memory | 20 |
| Dataset, checkpoints and RPO/RTO | 20 |
| Security, SLOs and incident response | 20 |
| Offline tests, live GPU evidence and independent review | 20 |
| **Total** | **100** |

Require >=90 and zero critical correctness, isolation, collective, checkpoint, rollback or independent review blockers. Offline policy passing is not live multi-node certification.

## References

- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)
- [PyTorch DDP](https://docs.pytorch.org/docs/stable/generated/torch.nn.parallel.DistributedDataParallel.html)
- [PyTorch FSDP](https://docs.pytorch.org/docs/stable/fsdp.html)
- [PyTorch Distributed Checkpoint](https://docs.pytorch.org/docs/stable/distributed.checkpoint.html)
- [NVIDIA NCCL](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)

**Next: Chapter 063 — Training Job Scheduling and Fairness.**
