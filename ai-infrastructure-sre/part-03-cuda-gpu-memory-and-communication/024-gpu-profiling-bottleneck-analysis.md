# Chapter 024 — GPU Profiling and Bottleneck Analysis

**Part 03 — CUDA, GPU Memory and Communication · Chapter 024 of 100**

**Status:** Published documentation. Live GPU profiling, independent technical/security review, and production acceptance remain pending until supported by executed evidence.

## Learning objectives

- Distinguish CPU launch overhead, GPU compute, memory bandwidth, data loading, PCIe/NVLink, NCCL communication, and scheduling bottlenecks.
- Use NVIDIA Nsight Systems (nsys), Nsight Compute (ncu), DCGM, PyTorch Profiler and CUDA events safely.
- Interpret kernel occupancy, achieved throughput, warp stalls, SM utilization, memory throughput, synchronization, and communication overlap without confusing correlation with causation.
- Design baseline, canary, regression, and remediation experiments for SRE/DBRE-operated AI infrastructure.
- Implement a fail-closed synthetic profiling acceptance policy and incident runbooks.

## 1. GPU workload critical path

```text
Request / training step
   |
   +--> input ingest --> CPU preprocess --> pinned-memory staging
   |                                        |
   +--> framework dispatcher --> CUDA kernel launch --> GPU streams
                                                   |      |
                                            SM compute   HBM/L2
                                                   \      /
                                               synchronization
                                                     |
                                  NCCL collectives / NVLink / NIC
                                                     |
                                           output / checkpoint
```

A GPU with 95% utilization may still have poor useful throughput. Utilization is often a sampled busy-time indicator; it is not the same as **SM occupancy**, **arithmetic intensity**, **memory bandwidth saturation**, **tensor-core efficiency**, or **application-level throughput**. Conversely, low utilization may be normal for latency-sensitive, bursty inference.

### Common bottleneck classes

| Class | Typical symptom | Distinguishing evidence |
|---|---|---|
| CPU / launch-bound | GPU gaps between kernels | CPU trace, launch API durations, queueing |
| Data-loader / storage-bound | idle GPU before batches | batch wait, disk/network throughput, worker queues |
| Compute-bound | sustained kernel execution | arithmetic throughput, instruction mix, roofline evidence |
| HBM / L2 bandwidth-bound | long memory-intensive kernels | memory throughput, cache hit rate, bytes moved |
| Occupancy / latency-bound | poor latency hiding | active warps, registers, shared memory, stall reasons |
| PCIe / host transfer-bound | slow host↔device copies | copy duration, pinned buffers, NUMA |
| NVLink / NCCL-bound | collectives dominate step | per-rank NCCL time, link and NIC metrics |
| Synchronization-bound | GPU waits or serialized streams | CUDA event/stream timeline, barriers |
| Capacity / contention | p99 worsens under colocation | per-tenant load, MIG profile, context count |

**Do not** label a kernel compute-bound from GPU utilization alone. Confirm with multiple measurements and the relevant profiler counters.

## 2. Baselines, SLOs, and experimental design

Capture: UTC timestamp, host, node pool, GPU UUID/model, MIG profile, driver, CUDA, framework, NCCL, image digest, model revision, precision, batch size, sequence length, concurrency, dataset, network path, and ambient load. A useful baseline includes p50/p95/p99 latency, training steps/sec or tokens/sec, TTFT/TPOT where relevant, GPU memory headroom, GPU utilization, kernel/copy/NCCL durations, and measurement uncertainty.

**Controlled comparison:** change one factor at a time, warm up kernels/JIT and caches, repeat trials, compare distributions rather than one run, and use the same workload shape and hardware. CUDA launches are asynchronous; CPU wall-clock around an unsynchronized kernel launch is **not** kernel execution time. Synchronize with CUDA events or explicit stream synchronization as appropriate.

## 3. Read-only triage

On an authorized node:

```bash
date -u
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,name,driver_version,memory.used,memory.free,utilization.gpu,utilization.memory,temperature.gpu,power.draw --format=csv
nvidia-smi --query-compute-apps=gpu_uuid,pid,process_name,used_gpu_memory --format=csv
nvidia-smi dmon -c 5
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|ECC|AER|oom' | tail -n 100
```

GPU/MIG/driver support for individual metrics varies. `N/A` means unknown, not zero. `dmon` is a short sampling tool and cannot substitute for application traces.

On Kubernetes:

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl describe pod APPROVED_POD -n APPROVED_NAMESPACE
kubectl describe node APPROVED_GPU_NODE
kubectl get events -A --sort-by=.lastTimestamp
```

Map pod UID, host PID, GPU UUID and GPU allocation before comparing workload and node metrics. Never share process command lines, request payloads, or traces containing sensitive tensor data without redaction.

## 4. Nsight Systems: end-to-end timeline

**Nsight Systems** helps attribute CPU activity, CUDA API calls, kernel execution, memory copies, and supported network/NVTX events. It is well suited for identifying idle gaps, synchronization, launch overhead and communication overlap.

Only profile an approved staging workload with bounded duration, storage, trace permissions and overhead budget:

```bash
nsys --version
nsys profile --help
# After confirming version-supported options:
nsys profile --trace=cuda,nvtx,osrt --output=approved_trace \
  python approved_staging_workload.py
```

Use a dedicated private output directory; traces may contain process names, kernel arguments, filesystem paths, model names and infrastructure details. The command is an example; verify option support and output policy for the installed version. Avoid always-on production tracing.

### Interpreting timelines

1. Identify CPU-side preparation and enqueue delay.
2. Locate GPU idle intervals between kernels.
3. Separate memory copies, compute kernels and collective kernels.
4. Inspect unexpected device-wide synchronizations.
5. Check whether concurrency is real or merely separate streams with dependencies.
6. Correlate rank timelines; one slow rank may stall all peers.

## 5. Nsight Compute: kernel-level analysis

**Nsight Compute** captures supported per-kernel metrics, including SM occupancy, memory throughput, instruction mix and stall reasons. Counter availability, replay requirements and permission restrictions depend on architecture and driver.

```bash
ncu --version
ncu --help
# Only after explicit approval and bounded kernel selection:
ncu --set basic --target-processes application-only \
  python approved_staging_workload.py
```

Some platforms require performance-counter access permissions, and collection can materially perturb execution. Avoid privileged access changes without security approval. Interpret achieved occupancy against kernel-specific theoretical occupancy, register pressure, shared memory and instruction-level parallelism. High occupancy does not guarantee high performance.

### Roofline intuition

Arithmetic intensity = operations / bytes transferred. A kernel with low arithmetic intensity may be bandwidth-limited even on a high-FLOPS GPU. For compute-intensive kernels, inspect tensor-core usage, dtype, alignment and kernel fusion. A roofline estimate is an **upper bound model**, not proof of root cause.

## 6. CUDA events and PyTorch Profiler

CUDA events time work on a device stream. They require correct stream placement and synchronization.

```python
# Run only on an approved CUDA-enabled staging environment.
import torch

assert torch.cuda.is_available(), "staging CUDA device required"
device = torch.device("cuda:0")
x = torch.randn((256, 256), device=device)
y = torch.randn((256, 256), device=device)
start = torch.cuda.Event(enable_timing=True)
end = torch.cuda.Event(enable_timing=True)

# Warm up compilation, caches and clocks.
for _ in range(5):
    _ = x @ y
torch.cuda.synchronize()
start.record()
for _ in range(20):
    z = x @ y
end.record()
end.synchronize()
print("GPU ms for 20 matmuls:", start.elapsed_time(end))
print("output shape:", tuple(z.shape))
```

The snippet is deliberately small; it is **not** a throughput benchmark or stress test. PyTorch Profiler can correlate CPU operators, CUDA kernels and memory events. Profiling options and tensor-shape/stack capture can expose sensitive metadata and add overhead; use a staging-only, reviewed trace configuration.

## 7. Memory, allocator, and fragmentation

Separate:
- Framework allocated bytes from allocator reserved bytes.
- Live tensors from cached allocator blocks.
- CUDA context/driver memory from application allocations.
- GPU device OOM from Linux cgroup `OOMKilled`.
- Model weights, activations, optimizer state, gradients, KV cache and NCCL workspace.

A profiler trace may reveal excessive allocations or copies; it does not by itself prove fragmentation. Collect framework allocator statistics and repeat a controlled workload. Avoid `empty_cache()` as a blanket performance fix: it may not release live tensors and can affect reuse/performance.

## 8. NCCL and multi-node profiling

For distributed training, correlate all ranks' compute/communication timelines with NIC counters, NVLink topology and checkpoint events. Distinguish communication that **overlaps** compute from communication that lies on the critical path. Do not attribute an NCCL hang to GPU saturation without rank-order, network and CUDA error evidence.

Compare version-pinned `nccl-tests` with the application workload at matching message sizes and topology. Benchmark-derived `busbw` is not a physical link utilization counter.

## 9. Twelve troubleshooting scenarios

| Symptom | First evidence | Safe next step |
|---|---|---|
| high GPU util, low tokens/sec | application SLO, kernel mix | compare useful throughput and batching |
| low GPU util, high latency | CPU timeline, queue | inspect data loader and launch gaps |
| p99 inference spikes | concurrency, KV cache, scheduler | isolate request shapes |
| HBM near capacity | allocated/reserved/context | inspect memory budget and fragmentation |
| frequent host→device copies | nsys copy timeline | evaluate pinned buffers/NUMA |
| kernels serialize unexpectedly | stream/event trace | inspect implicit synchronization |
| low achieved occupancy | ncu registers/shared memory | review kernel geometry |
| high occupancy, slow kernel | bandwidth and instruction counters | check roofline limits |
| NCCL dominates step | per-rank trace, NIC, topology | inspect collective size and overlap |
| GPU clocks drop | power/thermal counters | check node cooling/power policy |
| profiler changes performance | trace overhead/replay | reduce scope and compare unprofiled |
| metrics missing for MIG | DCGM/MIG capability | mark unknown; restore supported telemetry |

## 10. Eight production incident simulations

1. **CPU preprocessing stalls:** trace batch gaps; right-size workers and validate throughput.
2. **Host↔device transfer regression:** identify increased copies and NUMA mismatch.
3. **Kernel occupancy regression:** compare register/shared-memory changes across releases.
4. **KV cache pressure:** correlate p99 latency, allocation headroom and request mix.
5. **Unexpected stream synchronization:** detect serialization and validate dependency correctness.
6. **NCCL collective dominates:** correlate rank timelines and NIC transport before tuning.
7. **Thermal/power throttling:** correlate clocks, temperature and performance with vendor guidance.
8. **Profiler-induced SLO regression:** stop trace, restore baseline and tighten approved profiling window.

## 11. Eight operational runbooks

**RB-024-A — Baseline capture:** record hardware/software identity, workload shape, SLO and ambient load.

**RB-024-B — CPU/data bottleneck:** compare input queue, CPU, storage, kernel idle time and batch delivery.

**RB-024-C — Kernel bottleneck:** use bounded Nsight Compute, roofline, registers, occupancy and stalls.

**RB-024-D — Memory bottleneck:** compare HBM, L2, allocator, transfers, KV cache and fragmentation.

**RB-024-E — NCCL/communication:** correlate all ranks, collective timing, NIC, NVLink and topology.

**RB-024-F — Safe profiling:** obtain approval, minimize collection, protect traces, record profiler overhead.

**RB-024-G — Regression and rollback:** compare equivalent baselines, test one change, revert on SLO/correctness failure.

**RB-024-H — RCA and acceptance:** document bottleneck evidence, uncertainty, fix, validation, owners and follow-up.

## 12. Executable companion lab

[Lab 024 — GPU Profiling and Bottleneck Acceptance](../labs/024-gpu-profiling-bottleneck-check.md) contains a synthetic Python 3.11+ measurement-policy validator and pytest cases. It validates evidence completeness and SLO guardrails, **not** real GPU utilization or profiler correctness.

## 13. 100-point assessment

| Area | Points |
|---|---:|
| Critical-path and bottleneck reasoning | 20 |
| Profiling tools and metric interpretation | 20 |
| Controlled experiments and SLOs | 20 |
| Incident simulations and runbooks | 20 |
| Executable lab and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical data integrity, security, correctness, performance or recovery blocker.

## References

- [NVIDIA Nsight Systems](https://docs.nvidia.com/nsight-systems/)
- [NVIDIA Nsight Compute](https://docs.nvidia.com/nsight-compute/)
- [NVIDIA CUDA Events](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__EVENT.html)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [PyTorch Profiler](https://docs.pytorch.org/docs/stable/profiler.html)
- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA nccl-tests](https://github.com/NVIDIA/nccl-tests)

**Next:** Chapter 025 — CUDA and NCCL Troubleshooting Capstone.
