# Chapter 025 — CUDA and NCCL Troubleshooting Capstone

**Part 03 — CUDA, GPU Memory and Communication · Final capstone · Chapter 025 of 100**

**Publication status:** Published. Execution of offline tests, hardware staging scenarios, security review and production acceptance remain pending.

## Mission

Operate as an on-call AI Infrastructure SRE during a compound production incident: an eight-rank training job stalls after a GPU node rollout, some workers report CUDA initialization errors, one GPU has Xid events, NCCL collective latency has increased, and GPU memory pressure appears elevated. The capstone integrates Chapters 017–024: driver/runtime compatibility, HBM and OOM, CUDA contexts/streams, MIG, NVLink/NVSwitch, NCCL semantics, hang recovery and profiling.

**Outcome:** produce an evidence-backed fault tree, prioritize safe containment, demonstrate coordinated recovery from a verified checkpoint, validate performance and correctness, and deliver an auditable RCA/COA.

## 1. Reference architecture

```text
Training controller / launcher / checkpoint orchestrator
                        |
           Kubernetes GPU worker pods
          rank 0 .. rank 7 (2 nodes x 4 GPUs)
                        |
           CUDA runtime / framework
                        |
       NCCL communicators + CUDA streams
          |                        |
       NVLink/NVSwitch         NIC/RDMA/EFA
          |                        |
      GPU driver + HBM       Cross-node network
          \                        /
             Linux GPU nodes
                    |
           Object storage checkpoint
                    |
       DCGM + Prometheus + traces + logs
```

This is a **synthetic** architecture. Actual GPU count, network, MIG, node pool and checkpoint behavior must be confirmed for the user's staging platform. Do not infer that all GPUs or cloud SKUs provide NVSwitch or MIG.

## 2. Incident scenario and timeline

| UTC | Observation | Do not assume |
|---|---|---|
| 14:00 | GPU node image canary begins | image is compatible |
| 14:07 | rank 5 reports CUDA init failure | CUDA toolkit alone is root cause |
| 14:09 | rank 2 reports Xid | every Xid requires a reset |
| 14:10 | training step stops advancing | NCCL caused the stall |
| 14:12 | watchdog times out on rank 0 | rank 0 failed first |
| 14:14 | GPU memory reaches 95% on one slice | host has insufficient total VRAM |
| 14:16 | NCCL allreduce p99 exceeds baseline | network is necessarily broken |
| 14:20 | checkpoint age 11 minutes | checkpoint is application-consistent |

**Priority:** protect workload correctness and recoverability before throughput optimization. Stop further rollout; do not perform GPU resets, MIG reconfiguration, driver changes or unapproved pod termination.

## 3. Fault tree

```text
Distributed job no longer progresses
  +-- application: divergent collective order / rank count / datatype
  +-- framework: launcher failure / watchdog / checkpoint inconsistency
  +-- CUDA: driver-runtime mismatch / context fault / stream dependency
  +-- GPU: Xid / ECC / HBM OOM / MIG isolation or profile mismatch
  +-- fabric: NVLink / NVSwitch / PCIe AER / Fabric Manager
  +-- network: NIC / RDMA / EFA / RoCE / CNI / security policy
  +-- orchestration: eviction / OOMKilled / node NotReady / resource mismatch
  +-- data: input loader / storage / checkpoint I/O stall
  +-- observability: stale telemetry / incomplete rank logs
```

**Rule:** an error reported by NCCL is often downstream of another fault. Preserve temporal ordering and correlate GPU UUID, node, pod UID, rank, process and checkpoint step.

## 4. Evidence collection (read-only)

```bash
date -u
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,name,driver_version,memory.used,memory.free,utilization.gpu --format=csv
nvidia-smi -q -d ECC
nvidia-smi -q -d NVLINK
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|ECC|AER|OOM|NCCL' | tail -n 150
ip -brief link
ip -s link
```

Kubernetes, on an authorized context:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node APPROVED_GPU_NODE
kubectl describe pod APPROVED_RANK_POD -n APPROVED_NAMESPACE
kubectl logs APPROVED_RANK_POD -n APPROVED_NAMESPACE --since=30m
```

Some NVIDIA commands and counters are unsupported on particular GPU generations or MIG profiles. Record unsupported as **unknown**, not healthy. Avoid indiscriminate process dumps or traces containing model data, credentials or confidential network details.

## 5. Eight integrated diagnostic gates

1. **CUDA compatibility:** verify host driver, toolkit/runtime, GPU compute capability and container runtime.
2. **GPU memory:** separate HBM from cgroup OOM, allocated vs reserved bytes, KV/activation/NCCL workspace and MIG slice limits.
3. **Context/stream correctness:** identify fork hazards, implicit synchronization, missing events and context loss.
4. **MIG capacity/isolation:** verify SKU, GI/CI profile, UUID, Kubernetes advertised resource and tenant approval.
5. **GPU fabric:** compare physical NVLink/NVSwitch/PCIe topology and P2P capability with known-good baseline.
6. **NCCL contract:** reconcile all ranks' world size, collective order, count, dtype and last completed operation.
7. **Hang recovery:** require complete rank evidence, verified checkpoint and coordinated communicator teardown/restart.
8. **Performance profiling:** compare bounded Nsight/PyTorch/DCGM evidence, application SLO and version-pinned NCCL canary.

A gate is **unknown** if the evidence is missing. Unknown is not a pass.

## 6. Hands-on lab: staging GPU validation

### Lab A — Baseline inventory

On a **dedicated, approved staging pool**, capture driver/CUDA/NCCL/framework, GPU UUIDs, topology, NIC mapping, Kubernetes allocation, GPU health, application image digest and checkpoint metadata. Compare with a documented supported-version matrix. Do not modify drivers or firmware as part of inventory.

### Lab B — CUDA device and P2P smoke

Compile and execute the read-only CUDA peer-access probe from Chapter 021. Check all expected device pairs and MIG restrictions. The probe tests peer-access capability, **not** link bandwidth or actual collective performance.

### Lab C — NCCL correctness

Run Chapter 022's two-GPU CUDA AllReduce smoke or an approved `nccl-tests` build with bounded sizes:

```bash
./build/all_reduce_perf -b 8 -e 16M -f 2 -g 2
```

Record exact GPU UUIDs, NCCL/CUDA versions, numeric correctness, elapsed time, `algbw` and `busbw`, driver/network errors and benchmark version. Do not treat `busbw` as an NVLink physical link counter.

### Lab D — Synthetic rank stall

Use the companion **offline** lab to simulate missing rank, collective divergence, stale telemetry, Xid, bad checkpoint, missing rollback and unapproved restart. For live fault injection, use a disposable training job only after separate approval.

### Lab E — Profiling regression

Collect an approved bounded Nsight Systems trace, optional Nsight Compute counters where supported, CUDA events and application step timings. Distinguish CPU launch gaps, memory copies, compute, NCCL, synchronization and data loading. Do not run unrestricted tracing on production.

### Lab F — Recovery acceptance

Confirm verified durable checkpoint, consistent global step, model/optimizer/RNG/data-shard state as required, complete rank reformation, CUDA/NCCL canary, application correctness, workload throughput and SLO. A successful process restart alone does not establish training correctness.

## 7. Twelve troubleshooting scenarios

| Scenario | Key discriminating evidence | First safe response |
|---|---|---|
| driver/runtime mismatch | compatibility matrix, init errors | stop rollout and revert image |
| GPU OOM vs OOMKilled | device/cgroup metrics | right-size correct resource |
| CUDA fork/context failure | process model, init stack | change startup method in staging |
| missing stream dependency | CUDA events, race symptoms | fix dependency ordering |
| MIG profile mismatch | device plugin allocatable, profile | correct placement/admission |
| NVLink topology regression | `nvidia-smi topo`, UUID map | quarantine canary and investigate |
| Fabric Manager not ready | platform-specific service evidence | follow vendor-approved lifecycle |
| NCCL collective divergence | all-rank ordered operations | coordinated job stop and code fix |
| rank pod eviction | pod events, node pressure | recover whole rank group |
| RDMA transport failure | NIC counters, network plugin | restore approved transport |
| GPU Xid followed by hang | UTC Xid vs rank timeout | isolate hardware failure domain |
| profiling-induced slowdown | trace overhead vs baseline | stop profiling and restore baseline |

## 8. Ten incident simulation exercises

1. A CUDA container image upgrade fails on one node.
2. A MIG partition is smaller than the model memory requirement.
3. Two CUDA streams have an omitted event dependency.
4. A GPU Xid precedes an NCCL watchdog timeout.
5. One rank skips AllReduce because of divergent control flow.
6. Network interface selection changes after node replacement.
7. NVLink/P2P connectivity differs from the expected node SKU.
8. GPU allocator reservation increases while useful throughput drops.
9. Checkpoint metadata is stale or incomplete.
10. Telemetry disappears during a canary promotion.

For each drill: define trigger, expected detector, blast radius, evidence, containment, recovery, SLO impact, rollback, RCA/COA and acceptance owner.

## 9. Ten production runbooks

**RB-025-A — Incident command:** declare severity, freeze rollout, assign roles and preserve UTC evidence.

**RB-025-B — Driver/CUDA compatibility:** compare supported matrix, container image, driver and device runtime.

**RB-025-C — GPU memory/OOM:** separate device VRAM, MIG partition, allocator, cgroup and workload budget.

**RB-025-D — CUDA context/stream:** validate process model, streams, events and synchronization.

**RB-025-E — MIG/fabric topology:** inventory GPU/MIG UUIDs, NVLink/PCIe, plugin strategy and vendor limits.

**RB-025-F — NCCL collective mismatch:** compare all ranks' operation sequence, count, dtype and launcher state.

**RB-025-G — Network transport:** investigate NIC, RDMA/EFA/IB/RoCE, CNI, DNS, firewall and congestion.

**RB-025-H — Coordinated recovery:** stop/reform rank group, verify checkpoint and resume consistent global step.

**RB-025-I — Profiling and SLO:** run bounded canary, compare identical workload shape and correct numeric output.

**RB-025-J — RCA/COA and change closure:** document root cause, contributing factors, timeline, evidence and prevention.

## 10. Production readiness checklist

- [ ] Hardware, driver, CUDA, NCCL and framework versions approved.
- [ ] GPU/MIG resource inventory and isolation requirements verified.
- [ ] Rank mapping and collective correctness proven on staging.
- [ ] GPU health, network and fabric telemetry available and fresh.
- [ ] Checkpoint integrity and restore RTO tested.
- [ ] N-1 capacity and change rollback validated.
- [ ] Incident escalation, vendor ownership and on-call handoff documented.
- [ ] SLOs, error budget and acceptance owner signed off.
- [ ] Security/privacy review of traces and logs completed.
- [ ] All critical blockers resolved.

## 11. Companion executable capstone

[Lab 025 — CUDA and NCCL Production Readiness Gate](../labs/025-cuda-nccl-troubleshooting-capstone-check.md) is an offline Python 3.11+ evaluator with pytest cases for compatibility, rank/collective integrity, memory, topology, checkpoint, health, observability, rollback and change approval. It does not contact a GPU or modify production resources.

## 12. 100-point enterprise acceptance

| Assessment | Points |
|---|---:|
| CUDA/driver/GPU architecture and compatibility | 20 |
| NCCL/stream correctness and diagnosis | 20 |
| Incident command and recovery | 20 |
| Profiling, telemetry, performance and SLO | 20 |
| Executable lab, RCA/COA and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 **and** no critical safety, checkpoint, security, correctness or SLO blocker. Documentation publication is not proof of this assessment.

## References

- [NVIDIA CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA NCCL Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [NVIDIA Nsight Systems](https://docs.nvidia.com/nsight-systems/)
- [NVIDIA Nsight Compute](https://docs.nvidia.com/nsight-compute/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)

**Part 03 complete (9/9 published). Next: Chapter 026 — GPU Device Plugin and Resource Discovery, beginning Part 04 — Kubernetes GPU Administration.**
