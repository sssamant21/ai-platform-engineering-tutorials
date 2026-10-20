# Chapter 059 — Distributed Training Architecture

**AI Infrastructure Engineering & SRE · Part 08 — Distributed Training Operations · Chapter 059/100**

> **Publication boundary:** This is an enterprise architecture tutorial and offline synthetic gate. No live GPU cluster, Slurm/Ray/PyTorch distributed training, cloud deployment or disaster-recovery test is implied.

## Learning outcomes

Design and operate multi-node GPU training platforms, choose data/model/tensor/pipeline parallelism, map PyTorch ranks and process groups to Kubernetes GPU nodes, establish reliable rendezvous and collective communication, plan placement and network/storage topology, quantify scaling efficiency and protect checkpoint RPO/RTO. Identify production SRE responsibilities for incident command, security, capacity and rollback.

**Prerequisites:** Parts 01–07, CUDA/NCCL, Kubernetes GPU scheduling, high-performance networking, dataset loading and distributed checkpoint consistency.

## 1. Reference architecture

```text
training API / GitOps / approved job spec
                |
 admission: quota, fair-share, priority, budget
                |
 K8s Job / Kubeflow Training Operator / Ray / Slurm
                |
 rendezvous + membership + rank/world-size control
                |
  +-------------+-------------+-------------+
  | GPU node A  | GPU node B  | GPU node C  |
  | ranks 0..7  | ranks 8..15 | ranks 16..23|
  | NVLink      | NVLink      | NVLink      |
  +------RDMA/EFA/IB/RoCE network fabric----+
                | NCCL collectives / all-reduce
                |
 dataset cache / Lustre / versioned object storage
                |
 globally consistent distributed checkpoints
                |
 DCGM + Prometheus + OpenTelemetry + SLOs
                |
 on-call, incident command, DR and FinOps
```

Define the exact training framework, accelerator SKU, GPU interconnect, NIC count, NUMA domains, container image digest, driver/CUDA/NCCL compatibility and supported cloud/cluster topology. This diagram is illustrative, not a deployment manifest.

## 2. Parallelism taxonomy

| Strategy | Work partition | Main bottleneck | Reliability concern |
|---|---|---|---|
| Data parallel (DDP) | batch across ranks | gradient all-reduce | straggler/hung collective |
| FSDP / ZeRO-style sharding | parameters, gradients, optimizer | all-gather / reduce-scatter | shard completeness at checkpoint |
| Tensor parallel | tensor operations | low-latency intra-layer collectives | topology and network skew |
| Pipeline parallel | model stages | pipeline bubbles / microbatches | stage restart consistency |
| Hybrid 3D parallel | combined strategies | orchestration and contention | rank-group mapping and restore |

Choose based on model memory, global batch semantics, communication volume, checkpoint format, failure domain and measurable scaling efficiency. Increasing GPU count can reduce efficiency if data or collectives dominate.

## 3. PyTorch distributed process topology

In a conventional one-process-per-GPU DDP setup, each process has a global rank and local GPU index; the world size is the number of participating processes. A process group provides collective communication. `torchrun` supplies rendezvous environment variables in supported deployments. Avoid hardcoding rank membership or assuming elastic restart semantics for arbitrary training scripts.

```python
# Minimal application fragment; launch with a supported torchrun configuration.
import os
import torch
import torch.distributed as dist

def main():
    dist.init_process_group(backend="nccl")
    local_rank = int(os.environ["LOCAL_RANK"])
    torch.cuda.set_device(local_rank)
    rank = dist.get_rank()
    world_size = dist.get_world_size()
    try:
        # Build model, DistributedSampler and DDP wrapper here.
        # sampler.set_epoch(epoch) at every epoch boundary.
        print(f"rank={rank} world_size={world_size}")
    finally:
        dist.destroy_process_group()

if __name__ == "__main__":
    main()
```

**Note:** This is not a complete training program and requires compatible NVIDIA GPUs, PyTorch, NCCL and an initialized distributed launch. Ensure all ranks follow compatible collective ordering and fail together on unrecoverable errors.

## 4. Communication and topology

Inspect NVLink/NVSwitch locality, GPU PCIe root complexes, NIC NUMA affinity, EFA/InfiniBand/RoCE connectivity, MTU and congestion control. Measure collective p50/p95/p99 and compare with single-node baselines. A job can have healthy GPU utilization yet poor scaling due to all-reduce time.

**Useful read-only commands** (when installed and authorized):

```bash
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,name,utilization.gpu,memory.used --format=csv
lscpu
numactl --hardware
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

For collective tests use a reviewed, bounded NCCL-tests run in isolated staging. Do not saturate production fabric.

## 5. Kubernetes scheduling and reliability

Define GPU resource requests, CPU/memory requests, topology spread, affinity, taints/tolerations, priority classes, quotas and admission controls. Use a framework-specific operator or batch scheduler for coordinated distributed job lifecycle rather than assuming a vanilla Kubernetes Job provides all-rank gang scheduling. Plan GPU node autoscaler/Karpenter behavior, preemption, graceful termination and checkpoint window.

A pod restart is not necessarily a valid distributed recovery. Fence stale ranks, clean up rendezvous state and restore a globally committed checkpoint with optimizer, RNG and sampler state.

## 6. Capacity and scaling mathematics

```text
global_batch = microbatch_per_gpu * gradient_accumulation * data_parallel_replicas
ideal_step_time = single_gpu_step_time / gpu_count  # only for fixed total work
scaling_efficiency = throughput_N / (N * throughput_1)
collective_fraction = collective_time / total_step_time
checkpoint_write_GBps = checkpoint_GB / checkpoint_window_seconds
```

**Synthetic example:** 8 GPUs each process 100 samples/s; a 32-GPU run delivers 2,400 samples/s. Relative scaling efficiency is 2400 / (32 × 100) = 75%. Compare equivalent model, batch, data precision and training quality; throughput alone does not capture convergence or cost per useful token.

## 7. Dataset and checkpoint correctness

Map immutable dataset shards to rank and worker assignments. Use distributed samplers correctly; set epoch seeds and record iterator/cursor progress. For FSDP and tensor-parallel models, verify framework-specific distributed checkpoint APIs, resharding support and complete global manifests. Define exactly what changes are permitted across world-size restarts.

## 8. Observability and SLOs

Instrument GPU SM/memory, Xid/ECC, network throughput and retransmits, collective duration, data-loader wait, per-rank step skew, rendezvous failures, admission queue latency, checkpoint age, restart attempts and cost per successful training step. Set site-specific SLOs for successful job completion, p95 step time, RPO/RTO and scheduling latency. Correlate logs using job, rank, node, dataset release, image digest and UTC.

## 9. Security and governance

Use scoped Kubernetes service accounts and cloud workload identity, least privilege for checkpoint/object storage, pod security, network policy, tenant GPU quotas, secrets management, signed images and auditable change approvals. Protect data and optimizer state from cross-tenant access. Fault injection requires staging authorization.

## 10. Fifteen troubleshooting scenarios

1. GPU pods placed across an untested network fabric.
2. NCCL all-reduce hangs after one rank exits.
3. CUDA and NCCL versions differ across nodes.
4. Master rendezvous endpoint becomes unavailable.
5. World size and rank assignment diverge after restart.
6. GPU-to-NIC NUMA mismatch increases collective latency.
7. Oversubscribed network fabric degrades all-reduce p99.
8. Object-store reads starve GPUs before compute starts.
9. CPU decoding bottleneck looks like GPU underutilization.
10. Checkpoint shards capture inconsistent global steps.
11. A spot node eviction strands distributed ranks.
12. GPU quota and scheduler queue produce admission starvation.
13. A tenant saturates shared network or storage.
14. Stale metrics hide collective retry storms.
15. Restart from checkpoint duplicates dataset samples.

For each: symptom, affected SLI, first safe inspection, hypothesis, containment, restoration, rollback trigger and evidence.

## 11. Ten incident simulations

1. **INC-059-01:** Rank loss during collective.
2. **INC-059-02:** Rendezvous leader failure.
3. **INC-059-03:** Network partition between worker groups.
4. **INC-059-04:** NCCL timeout and watchdog.
5. **INC-059-05:** GPU Xid/ECC failure.
6. **INC-059-06:** Driver/CUDA version skew.
7. **INC-059-07:** Checkpoint write interruption.
8. **INC-059-08:** Preemptible worker eviction.
9. **INC-059-09:** Tenant congestion and fairness.
10. **INC-059-10:** Telemetry blackout during RPO breach.

Record incident commander, UTC timeline, first failed rank, blast radius, training loss window, SLO/error-budget impact, RPO/RTO, remediation, communications and RCA/COA. Run disruptive drills in isolated staging only.

## 12. Eight production runbooks

1. **RB-059-A — Admission and placement:** Inspect quotas, taints, GPU discovery, topology and queue state.
2. **RB-059-B — Collective hang:** Correlate first failed rank, NCCL logs, NIC counters and timeout.
3. **RB-059-C — Rendezvous recovery:** Fence stale members and restart under a supported membership model.
4. **RB-059-D — Node failure:** Preserve evidence, drain or replace unhealthy GPU host, restore job.
5. **RB-059-E — Checkpoint restart:** Verify global commit, optimizer, RNG, sampler and dataset release.
6. **RB-059-F — Performance regression:** Compare step breakdown, compute, data wait and collective time.
7. **RB-059-G — Security and isolation:** Verify workload identity, pod security, network policy and tenant quotas.
8. **RB-059-H — Incident closure:** Measure RPO/RTO and SLO, validate rollback and obtain independent review.

## 13. Twelve hands-on labs

1. Draw GPU node, NIC, fabric, rendezvous, dataset and checkpoint dependencies.
2. Select DDP, FSDP, tensor, pipeline or hybrid parallelism for example models.
3. Map global/local ranks, world size and data-parallel groups.
4. Calculate global batch and scaling efficiency.
5. Model network bandwidth and collective budget for N-1 degraded mode.
6. Define Kubernetes GPU admission, quotas and topology constraints.
7. Validate immutable dataset sharding and epoch correctness.
8. Model globally consistent checkpoint and RPO/RTO acceptance.
9. Simulate rank failure and rendezvous fencing.
10. Simulate collective timeout, network partition and driver skew.
11. Run offline fail-closed evaluator and pytest suite.
12. Assemble evidence, incident runbooks, cost model and 100-point readiness review.

**Companion:** [Lab 059 — Distributed Training Architecture Gate](../labs/059-distributed-training-architecture-check.md).

## 14. Enterprise acceptance — 100 points

| Domain | Points |
|---|---:|
| Architecture, rank groups and parallelism selection | 20 |
| Network topology, GPU scaling and scheduling | 20 |
| Dataset/checkpoint correctness and recovery | 20 |
| Security, observability and incident response | 20 |
| Offline tests, live staging evidence and independent review | 20 |
| **Total** | **100** |

Require >=90 with zero critical correctness, isolation, collective, checkpoint, rollback or review blockers. A synthetic pass does **not** certify a production distributed training platform.

## References

- [PyTorch Distributed Overview](https://docs.pytorch.org/tutorials/beginner/dist_overview.html)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)
- [PyTorch Distributed Checkpoint](https://docs.pytorch.org/docs/stable/distributed.checkpoint.html)
- [Kubernetes Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [NVIDIA NCCL](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)

**Next: Chapter 060 — Slurm Cluster Administration.**
