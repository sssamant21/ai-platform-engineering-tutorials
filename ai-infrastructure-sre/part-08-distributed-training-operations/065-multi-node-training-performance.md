# Chapter 065 — Multi-Node Training Performance

**AI Infrastructure Engineering & SRE · Part 08 — Distributed Training Operations · Chapter 065/100**

> **Evidence boundary:** Detailed architecture, staging commands and offline synthetic readiness lab. No live GPU benchmark, NCCL test, cloud profiling or independent acceptance has been executed.

## Learning objectives

Measure and optimize multi-node distributed training throughput without sacrificing correctness. Decompose step time into compute, collective, data-loader, CPU, synchronization and checkpoint overhead. Diagnose NCCL transport and topology, GPU/NIC NUMA locality, stragglers, storage backpressure, memory, thermal throttling, scaling efficiency, cost and SLO regressions.

**Prerequisites:** Chapters 043–050, 053–057, 059–064; PyTorch DDP/FSDP, NCCL, CUDA, Kubernetes GPU scheduling and checkpoint recovery.

## 1. Enterprise performance architecture

```text
pinned model / precision / global batch / dataset
                      |
 scheduler / gang placement / GPU quota
                      |
  GPU node A ---- high-speed fabric ---- GPU node B
  ranks 0..7        NCCL collectives      ranks 8..15
  NVLink, PCIe      RDMA/EFA/IB/RoCE      NVLink, PCIe
  CPU/NUMA/NIC                              CPU/NUMA/NIC
                      |
 dataset cache + parallel filesystem + object store
                      |
 distributed checkpoint / archive / restore
                      |
 DCGM + NIC + NCCL + profiler + OTel traces
                      |
 SLOs / capacity / FinOps / incident response
```

Measure at **rank, node, fabric and end-to-end job** levels. GPU utilization alone is insufficient: high utilization can coincide with poor throughput, while synchronization and input starvation can be hidden by coarse averages.

## 2. Baseline design and reproducibility

Pin container digest, driver, CUDA, NCCL, framework version, GPU SKU, precision, sequence length, optimizer, model, dataset manifest, global batch, gradient accumulation, rank topology, network transport and checkpoint schedule. Repeat tests, exclude warmup, record confidence intervals and report both throughput and convergence quality. Compare equivalent useful work.

## 3. Step-time decomposition

```text
step_wall_ms ~= max(per_rank_compute + communication + data_wait + synchronization)
               + checkpoint_amortization
global_batch = microbatch_per_rank * gradient_accumulation * data_parallel_replicas
scaling_efficiency = observed_throughput_N / (N * throughput_1)
collective_fraction = collective_p95_ms / step_p95_ms
data_wait_fraction = loader_wait_p95_ms / step_p95_ms
GPU_hours_per_million_tokens = allocated_GPU_hours / (tokens_processed / 1e6)
```

The decomposition is an approximation: communication and computation can overlap; p95 component latencies do not necessarily add to p95 end-to-end latency. Use traces and synchronized per-rank events for rigorous attribution.

**Synthetic example:** Single GPU: 100 samples/s; 32 GPUs: 2,400 samples/s => 75% scaling efficiency. With 200 ms step p95, 50 ms collective p95 and 20 ms loader wait p95, collective and loader fractions are 25% and 10% respectively. These are indicators, not an additive causal breakdown.

## 4. NCCL and fabric profiling

Inspect all-reduce, all-gather, reduce-scatter, broadcast and point-to-point patterns. Establish NVLink/NVSwitch intra-node bandwidth, GPU-NIC PCIe locality, RDMA/EFA/InfiniBand/RoCE path, link errors, retransmits, ECN/PFC where applicable and bisection bandwidth. NCCL algorithm/protocol selection is version- and topology-dependent. Compare approved `nccl-tests` baselines in isolated staging; never flood production fabric.

Read-only commands on authorized hosts:

```bash
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,utilization.gpu,utilization.memory,memory.used,temperature.gpu,power.draw --format=csv
lscpu
numactl --hardware
ip -s link
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

Use Nsight Systems, PyTorch Profiler and NCCL logs with bounded sampling. Avoid publishing sensitive environment variables or credentials in debug bundles.

## 5. GPU, CPU and memory performance

Investigate SM utilization, occupancy, HBM pressure, memory fragmentation, CUDA streams, gradient bucketing, activation checkpointing, CPU preprocessing, pinned memory, thread oversubscription, NUMA affinity, GPU clocks and thermal/power caps. A high collective fraction can be caused by a slow rank rather than network saturation.

## 6. Data pipeline and checkpoint contention

Measure object-store request rate, throttling, FSx/Lustre metadata operations, cache hit rate, dataloader queue depth, CPU decode time, checkpoint write bandwidth and restore read bandwidth. Separate data and checkpoint I/O budgets from collective fabric capacity where architecture permits. Validate cache warming, dataset sharding and rank replay correctness before tuning throughput.

## 7. Topology and scheduling

Gang admission, GPU SKU homogeneity, NVLink/NIC locality, cross-zone placement, tenant noisy neighbors, cloud quota and node autoscaler placement directly affect throughput. Use topology-aware scheduling and approved node labels/affinity, but validate actual GPU/NIC mapping after placement.

## 8. SLOs, cost and safe optimization

Track throughput, p95/p99 step latency, scaling efficiency, rank skew, NCCL collective latency, GPU memory/temperature, data wait, checkpoint age, job success, GPU-hours and cost per successful token. Prioritize correctness and recovery over nominal peak throughput. Canary optimizations, measure variance, define rollback thresholds and capture before/after evidence.

## 9. Fifteen troubleshooting scenarios

1. **TS-065-01:** All-reduce latency doubles after NIC replacement. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
2. **TS-065-02:** GPU utilization low despite full allocations. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
3. **TS-065-03:** One rank consistently straggles due to NUMA locality. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
4. **TS-065-04:** Object store request throttling stalls data loader. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
5. **TS-065-05:** FSx/Lustre metadata saturation during epoch transitions. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
6. **TS-065-06:** Network ECN/PFC misconfiguration impacts RoCE. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
7. **TS-065-07:** NCCL falls back to slower transport. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
8. **TS-065-08:** NVLink peer access unexpectedly unavailable. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
9. **TS-065-09:** Mixed precision change invalidates throughput baseline. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
10. **TS-065-10:** Gradient accumulation changes convergence semantics. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
11. **TS-065-11:** Checkpoint upload saturates training fabric. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
12. **TS-065-12:** GPU thermal throttling reduces step throughput. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
13. **TS-065-13:** CPU preprocessing causes high data-wait p95. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
14. **TS-065-14:** Autoscaler places workers across slow failure domains. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.
15. **TS-065-15:** Metrics pipeline misses rank-level tail latency. Record first affected rank, baseline delta, read-only inspection, containment, test and rollback.

## 10. Ten incident simulations

1. **INC-065-01:** NIC degradation and all-reduce p95 breach. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.
2. **INC-065-02:** NCCL fallback to TCP. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.
3. **INC-065-03:** GPU/NIC NUMA misplacement. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.
4. **INC-065-04:** Data loader backpressure. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.
5. **INC-065-05:** Object store throttling. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.
6. **INC-065-06:** Checkpoint network contention. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.
7. **INC-065-07:** Thermal throttling. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.
8. **INC-065-08:** Rank straggler injection. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.
9. **INC-065-09:** N-1 GPU/node capacity loss. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.
10. **INC-065-10:** Observability outage during performance regression. Execute only in approved staging; capture UTC timeline, affected ranks, p95 step/collective, throughput, GPU-hours, SLO and RCA.

## 11. Eight production runbooks

1. **RB-065-A — Baseline and comparability:** Pin model, precision, batch, data and image digest before comparing.
2. **RB-065-B — Collective latency:** Identify first slow rank, NCCL algorithm/transport, NIC errors and topology.
3. **RB-065-C — Data starvation:** Correlate loader wait, object IOPS, cache hit rate and CPU preprocessing.
4. **RB-065-D — GPU bottleneck:** Inspect SM occupancy, memory pressure, clocks, Xid and thermal limits.
5. **RB-065-E — NUMA and fabric:** Validate GPU-NIC locality, NVLink and RDMA/EFA/IB/RoCE routing.
6. **RB-065-F — Checkpoint contention:** Separate data, training collectives and checkpoint I/O budgets.
7. **RB-065-G — Regression rollback:** Canary approved changes and restore last known-good configuration.
8. **RB-065-H — Incident closure:** Recompute SLO, efficiency, GPU-hours, root cause and independent review.

## 12. Twelve hands-on labs

1. Pin a reproducible distributed training baseline and dataset manifest.
2. Map GPU/NIC NUMA, NVLink and failure domains.
3. Compute scaling efficiency for 1, 8, 16 and 32 GPUs.
4. Decompose step p95 into compute, collective and data wait indicators.
5. Simulate a straggler rank and collective tail latency.
6. Model N-1 surviving fabric capacity.
7. Investigate object-store and filesystem input starvation.
8. Model checkpoint I/O contention and retention tradeoffs.
9. Compare DDP/FSDP memory and collective costs.
10. Calculate GPU-hours per million useful tokens.
11. Run offline fail-closed policy and pytest.
12. Produce regression incident evidence, rollback and independent review.

**Companion:** [Lab 065 — Multi-Node Training Performance Gate](../labs/065-multi-node-training-performance-check.md).

## 13. Enterprise acceptance — 100 points

| Domain | Points |
|---|---:|
| Reproducible benchmark and scaling efficiency | 20 |
| NCCL fabric, GPU/NIC topology and rank profiling | 20 |
| Data pipeline, checkpoint contention and N-1 resilience | 20 |
| SLOs, cost, security and incident response | 20 |
| Offline tests, live benchmark and independent signoff | 20 |
| **Total** | **100** |

Require >=90, no critical correctness, security, SLO, recovery or rollback failures, reproducible staging benchmarks and independent review. Synthetic policy success is not production validation.

## References

- [PyTorch Profiler](https://docs.pytorch.org/docs/stable/profiler.html)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)
- [NVIDIA NCCL](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [Kubernetes scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)

**Next: Chapter 066 — Distributed Training SRE Capstone.**
