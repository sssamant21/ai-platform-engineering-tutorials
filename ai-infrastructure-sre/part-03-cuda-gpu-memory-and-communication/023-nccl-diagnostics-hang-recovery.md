# Chapter 023 — NCCL Diagnostics and Hang Recovery

**Part 03 — CUDA, GPU Memory and Communication · Chapter 023 of 100**

**Publication status:** Published documentation. Offline tests, GPU/network staging exercises, security review, and production acceptance are independent pending milestones.

## Objectives

1. Classify a slow, stalled, or failed NCCL collective without assuming NCCL is the root cause.
2. Correlate rank progress, communicator lifecycle, GPU Xid, CUDA errors, Kubernetes events, network transport, and checkpoint state.
3. Use version-compatible NCCL/PyTorch diagnostic controls with bounded overhead and appropriate privacy.
4. Design coordinated timeout, failover, rollback, and checkpoint-recovery procedures.
5. Validate safe escalation using an offline Python 3.11+ synthetic hang-triage policy and live staging runbooks.

## 1. Failure-domain architecture

```text
Application progress / training step / inference request
                      |
             Framework distributed runtime
          ranks, process groups, watchdogs
                      |
            NCCL communicator / collectives
               |                 |
       CUDA streams/events    Transport discovery
               |                 |
       GPU/driver/Xid      NVLink/PCIe/NIC/RDMA
               \                 /
                  Host OS / Kubernetes
                      |
        Checkpoint storage / orchestration
```

A collective hang is an **observation**, not a diagnosis. The first rank to log a timeout may be the **victim** of a different rank's failure. Common initiating failures include inconsistent collective order/count, a dead process, GPU Xid, device OOM, network partition, bad interface selection, resource pressure, or framework launch skew.

A communicator's correctness depends on compatible rank participation and operation ordering. Retrying a collective independently on one rank can deepen a mismatch. Treat communicator error states as potentially unrecoverable and follow the framework/NCCL-version-specific recovery contract.

## 2. Distinguish five classes of incidents

| Failure | Evidence | Recovery boundary |
|---|---|---|
| **Progress stall** | no completed training step or collective for a bounded interval | identify stalled rank and dependency |
| **Collective mismatch** | ranks disagree on operation, count, datatype, or order | correct application contract; coordinated restart |
| **Transport fault** | NIC link, RDMA errors, routing, firewall, socket logs | restore network and rebuild communicator |
| **GPU/CUDA fault** | Xid, ECC, CUDA illegal access/OOM | quarantine hardware if necessary; checkpoint recovery |
| **Orchestration fault** | pod eviction, OOMKilled, restart, node loss | recover whole job/rank group |

Distinguish host-side launch delays from device-side execution, and distinguish high collective latency from **zero forward progress**. A long collective may be legitimate for unusually large payloads or transient congestion; thresholds must be workload- and environment-specific.

## 3. Evidence collection: read-only first

On the authorized GPU host:

```bash
date -u
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,name,driver_version,memory.used,utilization.gpu --format=csv
nvidia-smi --query-compute-apps=gpu_uuid,pid,process_name,used_gpu_memory --format=csv
ip -brief link
ip -s link
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|ECC|AER|oom|rdma' | tail -n 120
```

For Kubernetes:

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl describe pod APPROVED_RANK_POD -n APPROVED_NAMESPACE
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node APPROVED_GPU_NODE
kubectl logs APPROVED_RANK_POD -n APPROVED_NAMESPACE --since=30m
kubectl logs APPROVED_RANK_POD -n APPROVED_NAMESPACE --previous --since=30m
```

The previous-container log is only available when retained. Collect logs from **all** ranks, not just the first failed pod. Map UTC time, rank, world size, GPU UUID, node, pod UID, image digest, CUDA/NCCL/framework version, communicator ID where safely exposed, last completed step, checkpoint age and incident start.

**Privacy:** logs and process arguments can expose IPs, credentials, filesystem paths, model identifiers or training data. Redact before attaching to tickets; never dump tensors or secrets by default.

## 4. NCCL and framework diagnostic controls

Diagnostic settings vary by NCCL and PyTorch version. Confirm supported variables against the deployed release. For an approved staging experiment, representative controls include:

```bash
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,NET,COLL
# Optional diagnostic log path: ensure private writable directory and retention.
# export NCCL_DEBUG_FILE=/approved/private/path/nccl.%h.%p.log
```

For PyTorch distributed jobs, version-supported watchdog/flight-recorder controls can help identify the last enqueued/completed collective; consult the exact framework documentation. Avoid inventing universal names or assuming a setting exists on all releases.

**Do not** set broad debug tracing permanently in production: it can create substantial logs, change timing, and leak infrastructure metadata. `NCCL_SOCKET_IFNAME`, `NCCL_IB_DISABLE`, `NCCL_P2P_DISABLE`, `NCCL_ALGO`, and `NCCL_PROTO` are powerful experimental controls, not general fixes. Changing transport may mask the cause or degrade performance. Preserve original values and test rollback.

## 5. A disciplined rank-by-rank investigation

1. **Freeze the evidence:** preserve logs, current state, checkpoint metadata, image/version, and UTC timestamps.
2. **Establish last good progress:** compare step counters and collective sequence IDs across ranks.
3. **Find the earliest divergence:** rank missing, different operation, slow data loader, CUDA fault, transport timeout, or node event.
4. **Classify hardware:** correlate GPU UUID with Xid/ECC, PCIe AER, device memory and driver reset.
5. **Classify transport:** compare intra-node and inter-node behavior; NIC errors, RDMA registration, DNS, firewall, CNI, EFA/IB/RoCE and interface choice.
6. **Classify orchestration:** pod restart, eviction, host OOM, node NotReady, PDB/drain and autoscaler events.
7. **Choose safe recovery:** abort failed communicator through supported framework path; do not independently resume one rank.
8. **Restore consistent state:** verify checkpoint includes model, optimizer, scheduler, RNG, step and data-shard progress as required by the application.
9. **Canary and verify:** exact world size, rank mapping, NCCL smoke, correctness, step throughput and SLO.
10. **Document RCA/COA:** primary failure, contributing factors, blast radius, prevention and regression test.

## 6. Timeout and watchdog design

A useful timeout should account for expected collective size, network variance, checkpoint duration, initialization skew and worst-case training step. Define separate limits for startup, collective progress, application heartbeat, job completion and recovery. Timeout expiration should trigger **diagnosis and controlled job coordination**, not automatic destructive node resets.

A watchdog may observe progress indirectly; it must distinguish telemetry failure from real job progress failure. When logs or rank heartbeat are missing, mark evidence **unknown** and fail closed on production promotion.

## 7. Staging NCCL reproduction

Use a dedicated, approved GPU staging pool with no production traffic. Confirm:

- All ranks share compatible NCCL/CUDA/framework versions and matching collective contracts.
- GPU UUIDs, topology, node/NIC affinity and communicator bootstrap are recorded.
- The pinned `nccl-tests` version supports the environment; run `all_reduce_perf` at bounded message sizes and a controlled world size.
- Capture numeric correctness, latency, throughput, errors and per-rank completion.
- Exercise only **safe fault simulations** such as deliberately missing synthetic heartbeats or stopping a disposable staging rank with explicit approval; never kill a shared host process or disrupt a production fabric.
- Confirm that the job recovers from an independently verified checkpoint and that all ranks resume at a consistent step.

Example **staging-only** single-node command, assuming an already built compatible nccl-tests tree:

```bash
./build/all_reduce_perf -b 8 -e 16M -f 2 -g 2
```

This is a bounded functional/performance smoke, not proof of multi-node recovery. Do not compare `busbw` to physical NVLink line rate as if they were identical.

## 8. Observability and SLOs

| Signal | Source | Interpretation |
|---|---|---|
| Last completed training step | application | primary progress signal |
| Last enqueued/completed collective | framework trace where supported | locate divergence |
| Rank heartbeat age | orchestrator/application | missing rank or delayed progress |
| NCCL error/timeout count | framework logs | symptom; not necessarily root cause |
| Xid/ECC/PCIe AER | driver/DCGM/kernel | device/fabric evidence |
| NIC drops/retransmits/RDMA errors | network telemetry | transport fault evidence |
| Checkpoint age and restore success | training/checkpoint service | recoverability |
| Job restart and MTTR | orchestration/SRE | reliability |
| GPU utilization and memory | DCGM/driver | context, not proof of forward progress |

Define SLOs for successful distributed steps, training availability, checkpoint recovery RTO, and time-to-detect rank stalls. Set example alert thresholds only after baseline characterization. For instance, a 120-second heartbeat age is a **synthetic lab default**, not a production recommendation.

## 9. Twelve troubleshooting cases

1. **One rank skips a collective:** compare ordered operation IDs; correct control-flow divergence.
2. **Different tensor counts:** validate shape/count/dtype across all ranks before launch.
3. **One rank OOMKilled:** correlate cgroup and GPU VRAM separately; restore the group.
4. **Xid precedes watchdog timeout:** isolate GPU/driver fault and evaluate node replacement.
5. **Slow allreduce after node change:** compare NVLink topology, NIC mapping and clocks.
6. **Wrong network interface:** inspect NCCL bootstrap logs and approved interface policy.
7. **RDMA transport fails:** inspect NIC/driver/plugin and cloud security policy.
8. **TCP fallback hides issue:** verify actual transport and performance; avoid permanent workaround.
9. **Container image drift:** compare CUDA/NCCL/framework versions and immutable digests.
10. **Process group initialized with wrong world size:** reconcile launcher membership and rank map.
11. **Repeated recovery loop:** validate checkpoint completeness and consistent global step.
12. **Telemetry gap during hang:** use independent application/driver evidence; fail closed.

## 10. Ten production incident simulations

1. Synthetic missing rank heartbeat.
2. Collective sequence divergence across ranks.
3. Single rank CUDA OOM and coordinated recovery.
4. Rank pod eviction while peers remain running.
5. GPU Xid followed by distributed watchdog timeout.
6. Inter-node packet loss or RDMA degradation on a disposable test fabric.
7. Incorrect NCCL socket interface after configuration rollout.
8. Checkpoint missing optimizer or RNG state.
9. NCCL version mismatch after partial deployment.
10. Missing telemetry during a canary, requiring promotion halt.

For each drill, capture trigger, detection time, rank/GPU identity, error classification, recovery action, SLO impact, checkpoint integrity, rollback, and RCA owner.

## 11. Eight operational runbooks

**RB-023-A — Initial triage:** identify affected job, rank count, last progress, UTC start, blast radius and current checkpoint.

**RB-023-B — Collective mismatch:** compare per-rank sequence, datatype/count, code path and launcher; stop unsafe partial retry.

**RB-023-C — GPU/driver fault:** correlate Xid/ECC, memory, CUDA error and device UUID; quarantine only with approval.

**RB-023-D — Network transport:** verify NIC, RDMA/EFA/IB, routing, security policy, packet errors and transport selection.

**RB-023-E — Kubernetes rank loss:** inspect pod status, OOMKilled, eviction, node events and restart policy; coordinate group recovery.

**RB-023-F — Checkpoint restoration:** verify durable checkpoint, application-consistent global step and required optimizer/data state.

**RB-023-G — Safe canary/rollback:** pin versions, use bounded nccl-tests, confirm all-rank progress and restore previous image if needed.

**RB-023-H — RCA and prevention:** document initiating fault, downstream timeouts, owner, SLO, action items and test evidence.

## 12. Executable companion lab

[Lab 023 — NCCL Hang Triage and Recovery Policy](../labs/023-nccl-diagnostics-hang-recovery-check.md) provides an offline Python 3.11+ validator with pytest cases covering rank identity, progress, collective ordering, error signals, checkpoint readiness, observability and approval gates. It does **not** run NCCL or perform live fault injection.

## 13. 100-point assessment

| Category | Points |
|---|---:|
| Failure domains and collective correctness | 20 |
| All-rank evidence and observability | 20 |
| Safe coordinated recovery/checkpoint | 20 |
| Incident drills and runbooks | 20 |
| Executable lab and staging proof | 20 |
| **Total** | **100** |

Pass >=90 with no critical correctness, checkpoint, tenant security, recovery or SLO blocker.

## References

- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NCCL Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html)
- [NCCL Environment Variables](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)
- [NVIDIA nccl-tests](https://github.com/NVIDIA/nccl-tests)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)
- [PyTorch ProcessGroupNCCL Environment Variables](https://docs.pytorch.org/docs/stable/torch_nccl_environment_variables.html)
- [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

**Next:** Chapter 024 — GPU Profiling and Bottleneck Analysis.
