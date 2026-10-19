# Chapter 049 — NCCL Network Performance Troubleshooting

**AI Infrastructure Engineering & SRE | Part 06 — High Performance Networking | Chapter 049/100**

> **Release status:** Published learning material and offline synthetic lab. Live NCCL benchmark execution, failure injection, independent security review, and production signoff are pending.

## Objectives

Troubleshoot NCCL collective hangs, bandwidth regressions, slow ranks, topology mismatch, GPU/NIC failures, RDMA provider fallback, RoCE congestion, native InfiniBand faults, AWS EFA integration, Kubernetes scheduling and checkpoint recovery. Build reproducible evidence and a fail-closed production acceptance process.

**Prerequisites:** Chapters 017–025 (CUDA/NCCL), 026–034 (Kubernetes GPU), and 043–048 (network fabrics).

## 1. End-to-end architecture

```text
PyTorch DDP / FSDP / distributed trainer
           | collective: all-reduce / all-gather / reduce-scatter
      NCCL ranks and communicators
           | topology / channels / transport selection
     CUDA streams + GPU buffers
           | PCIe / NVLink / NVSwitch
     NIC / HCA / provider plugin
           | IB verbs / RoCEv2 / EFA libfabric / sockets
     leaf-spine fabric / AWS EFA network
           | remote GPU ranks
  observability + checkpoint / restart + incident command
```

A timeout is a **symptom**, not proof of a network defect. GPU kernel faults, inconsistent rank participation, CPU starvation, incompatible libraries, OOM, storage stalls, rendezvous errors and topology changes can produce similar symptoms.

## 2. NCCL collective performance model

For each collective, record operation, element count, datatype, rank count, nodes, GPUs per node, message sizes, channel count, GPU topology, NCCL version and measured `algbw`/`busbw`. Interpret `busbw` according to the collective's algorithm-specific accounting, not as raw wire throughput. Application step p95/p99 and successful completion rate are the primary user-facing signals.

| Symptom | Competing hypotheses | First evidence |
|---|---|---|
| Collective hangs | missing rank, GPU Xid, transport fault, synchronization mismatch | rank logs, GPU health, timeout timestamps |
| Low `busbw` | PCIe locality, TCP fallback, congestion, wrong message size | topology, NCCL transport, fabric counters |
| One slow rank | host CPU pressure, NIC errors, GPU clocks, storage | per-rank timelines and metrics |
| Intermittent failures | link flap, PFC pause, ECN drift, capacity churn | correlated network and pod events |
| Only multi-node fails | security group/ACL, RDMA provider, MTU, fabric | cross-node path and plugin evidence |
| Works then regresses | driver/firmware/NCCL change, scheduling, contention | immutable version diff and change timeline |

## 3. Safe read-only baseline collection

On approved nodes or clusters, gather the following without mutating state:

```bash
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,utilization.gpu,memory.used,temperature.gpu --format=csv
rdma link show
ibv_devices
ip -s link
ethtool -S APPROVED_NIC | grep -Ei 'drop|crc|ecn|pfc|pause|error'
kubectl get pods -A -o wide
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

Use `fi_info -p efa` only where AWS EFA/libfabric is installed, `ibstat` for native InfiniBand hosts, and switch counters through approved fabric administration. Missing counters are **unknown**, not zero. Scrub internal addresses and workload secrets from diagnostic artifacts.

## 4. Transport-specific diagnosis

**Native InfiniBand:** validate HCA/port state, subnet manager, partition membership, route/path health, QP/CQ errors, GPU/HCA locality and supported memory-registration path.

**RoCEv2:** validate end-to-end DSCP/PCP/traffic class mapping, MTU, ECN/DCQCN, scoped PFC, pause duration, switch queues and congestion drops. PFC marks or pauses alone do not prove failure; correlate with latency and throughput.

**AWS EFA:** verify EFA-capable instance, attached EFA ENI, kernel driver, `libfabric` EFA provider, AWS OFI NCCL plugin, same-AZ compatible rank placement and peer security groups. EFA is not native InfiniBand and does not use its subnet-manager procedures.

**Sockets/TCP fallback:** verify from actual NCCL logs/provider instrumentation. A successful training job is not proof of RDMA or GPUDirect RDMA.

## 5. NCCL debugging without unsafe production changes

In an **approved isolated staging** run, pin the workload image digest, driver, CUDA, NCCL, plugin and firmware versions. NCCL debug settings can reveal transport and initialization choices; excessive tracing can change performance and expose topology, so use limited windows and sanitize logs.

```bash
# Staging-only: collect NCCL initialization and network transport diagnostics.
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,NET
# Execute an approved nccl-tests build with explicit GPU count and message range.
# Example syntax only; confirm test binary and host allocation first:
./build/all_reduce_perf -b 8 -e 1G -f 2 -g 1
```

The single-process example does **not** validate multi-node operation. For multi-node benchmarks use an approved launcher, explicit rank/world-size mapping, matching versions, controlled placement and monitored traffic budget. Capture per-rank logs and test completion; do not set `NCCL_IB_DISABLE`, interface overrides or timeout tuning as permanent workarounds without diagnosis.

## 6. Decision tree for collective hangs

1. Identify first failed collective, affected rank(s), world size and synchronized timestamps.
2. Verify all ranks entered the same collective in the same order; inspect application exceptions and rendezvous logs.
3. Check GPU Xid/ECC, OOM, CUDA errors and host CPU/memory pressure.
4. Confirm NCCL/CUDA/driver and transport plugin versions.
5. Check effective NCCL transport and GPU/NIC locality.
6. Inspect physical link errors, switch queue congestion, ECN/PFC or EFA provider events as appropriate.
7. Compare matched healthy and failed runs; identify first divergence.
8. Preserve checkpoint and isolate or restart only with workload owner approval.
9. Validate collective correctness, step-time SLO, checkpoint RPO/RTO and rollback.

## 7. Performance methodology

Use representative all-reduce, all-gather and reduce-scatter sizes; repeat runs with fixed rank counts, warmup, GPU clocks/power context, affinity, fabric load and storage conditions. Record throughput, p50/p95/p99, rank skew, retransmits where applicable, queue pressure, GPU utilization and cost per successful job. Test under an approved largest-failure-domain outage (N-1) in staging. Do not assume bandwidth scales linearly with rank count or rail count.

## 8. Kubernetes and platform controls

Validate GPU device plugin, RDMA/EFA plugin if applicable, extended-resource allocatable, node affinity, topology placement, namespace quotas, network policy, CNI/IPAM and immutable images. Coordinated training ranks require consistent scheduling/admission. GPU autoscaler churn or rolling driver upgrades can break collectives; gate rollouts with checkpoint/restart and canary validation. Do not universally use privileged pods or host networking.

## 9. Observability, SLOs and alerting

Track collective completion rate, p95 step latency, NCCL bus bandwidth against a **matched baseline**, per-rank skew, GPU Xid/ECC, host saturation, link/queue counters, transport fallback, checkpoint age and telemetry freshness. Correlate by job, rank, GPU UUID, node, NIC, switch path and UTC timestamp. Page on user impact and supporting evidence, not every benign ECN mark.

## 10. Twelve troubleshooting scenarios

1. NCCL hangs because one rank never joins a collective.
2. GPU Xid on a single rank mimics network deadlock.
3. EFA plugin upgrade triggers TCP fallback.
4. RoCE PFC storm causes all-reduce tail latency.
5. InfiniBand partition or route mismatch.
6. GPU/NIC PCIe/NUMA locality regression.
7. NCCL/CUDA/driver incompatibility.
8. MTU or peer security-group policy mismatch.
9. NCCL performance collapses at larger message sizes.
10. Kubernetes device plugin loses RDMA allocation.
11. Checkpoint storage burst competes with training traffic.
12. Telemetry gap obscures the first failing rank.

## 11. Ten production-style incident simulations

1. Missing rank during all-reduce.
2. GPU driver regression after node upgrade.
3. Network rail loss breaches N-1 bandwidth.
4. NCCL transport silently falls back to sockets.
5. EFA provider initialization failure.
6. RoCE queue drops under incast.
7. IB HCA port flap.
8. CPU affinity misplacement causes stragglers.
9. Checkpoint RPO violation after a collective timeout.
10. Security policy drift plus stale monitoring.

For each: record incident commander, severity, UTC timeline, affected jobs/ranks, SLO/error-budget impact, first evidence, containment, rollback, checkpoint integrity, owner approval and RCA/COA. Disruptive tests require authorized staging.

## 12. Eight operational runbooks

**RB-049-A — Rank rendezvous:** identify missing/misaligned ranks, preserve logs and stop unsafe retries.

**RB-049-B — GPU health:** inspect Xid/ECC, OOM, driver, CUDA and host pressure.

**RB-049-C — Transport discovery:** confirm NCCL provider, libfabric/verbs, plugin and fallback.

**RB-049-D — Fabric congestion:** inspect path errors, queue drops, ECN/PFC, routing and MTU.

**RB-049-E — Performance regression:** compare matched `nccl-tests` baselines, rank skew and locality.

**RB-049-F — Kubernetes scheduling:** check device allocatable, affinity, CNI, quotas and autoscaler churn.

**RB-049-G — Recovery:** preserve checkpoints, validate RPO/RTO, restart safely and prove SLO.

**RB-049-H — Postincident:** collect signed evidence, budget/cost, root cause, changes and rollback.

## 13. Eight hands-on labs

1. Draw collective, GPU, NIC, fabric and checkpoint dependency graph.
2. Capture baseline inventory and versions using read-only commands.
3. Analyze per-rank collective timestamps and identify first divergence.
4. Compare InfiniBand, RoCE, EFA and socket failure signatures.
5. Benchmark NCCL in approved staging with a controlled baseline.
6. Analyze simulated GPU, rank, queue and checkpoint incidents.
7. Run offline fail-closed NCCL readiness tests.
8. Produce an enterprise acceptance packet and 100-point scorecard.

**Companion:** [Lab 049 — NCCL Network Performance Troubleshooting Gate](../labs/049-nccl-network-performance-troubleshooting-check.md).

## 14. Enterprise assessment (100 points)

| Domain | Points |
|---|---:|
| Collective semantics and reproducible baselines | 20 |
| Transport, GPU and fabric diagnosis | 20 |
| Kubernetes scheduling, security and change controls | 20 |
| SLO, incident response, N-1 and checkpoint recovery | 20 |
| Tests, evidence, live staging and reviewer signoff | 20 |
| **Total** | **100** |

Require >=90 **and** no critical blocker for correctness, transport, security, N-1, telemetry, checkpoint or rollback. Synthetic lab results alone cannot authorize production.

## Official references

- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA nccl-tests](https://github.com/NVIDIA/nccl-tests)
- [NVIDIA GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)
- [AWS OFI NCCL](https://github.com/aws/aws-ofi-nccl)
- [Linux RDMA documentation](https://docs.kernel.org/infiniband/index.html)
- [Kubernetes device plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next: Chapter 050 — AI Networking Reliability Capstone.**
