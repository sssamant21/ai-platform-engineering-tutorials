# Chapter 011 — GPU Architecture, SMs, HBM and Memory Bandwidth

**Part 03 — GPU Architecture, CUDA and VRAM | Chapter 011 of 100**

> **Status:** Tutorial published. Technical/security review, real lab execution and acceptance pending. Hardware numbers in exercises are synthetic; verify vendor datasheets and actual devices before operational decisions.

## Learning objectives

Explain streaming multiprocessors (SMs), tensor cores, high-bandwidth memory (HBM), memory hierarchy, GPU interconnects and CPU-to-GPU transfers; distinguish compute-bound and bandwidth-bound inference; interpret DCGM/nvidia-smi signals; estimate memory and token throughput; troubleshoot GPU saturation, throttling and out-of-memory failures; and create a production SRE capacity worksheet.

## 1. GPU architecture for inference SRE

```text
Client → Gateway → Inference Scheduler
                        │
                        ▼
                  CPU host / RAM
                        │ PCIe or NVLink / fabric
                        ▼
GPU device
 ├─ SMs: warp scheduling, CUDA cores, tensor cores
 ├─ Registers and shared memory / L1 cache
 ├─ L2 cache and memory controllers
 ├─ HBM / VRAM: weights, KV cache, activations, workspace
 └─ DMA, copy engines, interconnects
                        │
                        ▼
             Generated tokens / SSE
```

A GPU has many streaming multiprocessors (SMs). Each SM schedules groups of threads (warps), uses registers and shared memory, and executes arithmetic on CUDA cores and tensor cores where supported. A kernel's achieved occupancy depends on registers, shared memory, block size, architecture and scheduling; high occupancy alone does not guarantee performance.

Tensor cores accelerate supported matrix operations and numeric formats. Tensor-core availability does not mean every operator runs on tensor cores; inspect profiler evidence before attributing a regression.

## 2. Memory hierarchy and bandwidth

| Tier | Role | SRE implications |
|---|---|---|
| Registers | per-thread fast storage | register pressure reduces residency |
| Shared memory/L1 | on-chip data reuse | kernel tiling and contention |
| L2 cache | device-wide reuse | cache hit rate influences HBM traffic |
| HBM/VRAM | weights, KV, activations | capacity, bandwidth, ECC |
| Host RAM | staging/offload | page-locked memory and transfer overhead |
| PCIe/NVLink | host/device or GPU/GPU transport | transfer, tensor parallelism, collectives |
| Network fabric | cross-node distributed inference | latency, congestion, topology |

**Capacity and bandwidth are different.** A GPU can have enough VRAM to load a model yet be bandwidth-constrained during decode. Conversely, high nominal HBM bandwidth does not guarantee high effective bandwidth under random access, small batches or poor kernels.

## 3. Prefill versus decode

Prefill performs prompt-wide computation and may approach compute throughput limits when matrix sizes are large. Decode often reads substantial weights/KV state to generate a small number of tokens per active sequence, making memory bandwidth important. Continuous batching can improve weight reuse and arithmetic intensity but can increase queueing, KV-cache pressure and p99 latency.

The bottleneck is workload-specific: architecture, quantization, attention implementation, batch size, context length, kernel fusion, parallelism and GPU SKU matter. Measure rather than assume.

## 4. Roofline reasoning

A simplified performance bound is:

```text
achievable_operations_per_second
 ≤ min(peak_compute_ops_per_second,
       arithmetic_intensity_ops_per_byte × sustainable_memory_bandwidth)
```

Arithmetic intensity is useful operations divided by bytes moved from the relevant memory level. The roofline is a model, not a benchmark guarantee. Peak vendor compute figures may refer to particular precisions and sparsity assumptions; compare like for like.

### Synthetic example

Assume sustainable memory bandwidth 2.0 TB/s and arithmetic intensity 20 operations/byte:

```text
bandwidth roof = 2.0 × 10^12 bytes/s × 20 ops/byte
               = 40 × 10^12 ops/s = 40 TOPS
```

If relevant peak compute is 100 TOPS, the memory roof dominates under these assumptions. The example does not represent any specific GPU.

## 5. VRAM budget and KV cache

```text
VRAM required ≈ weights + KV cache + activations/workspace
              + CUDA/runtime buffers + fragmentation + headroom
```

For conventional attention:

```text
KV bytes/token ≈ 2 × layers × KV_heads × head_dim × bytes_per_element
```

For synthetic 32 layers, 8 KV heads, 128 head dimension, BF16 (2 bytes):

```text
KV/token = 2 × 32 × 8 × 128 × 2 = 131,072 bytes = 128 KiB
8,192 resident tokens ≈ 1 GiB logical KV state
```

This excludes paged block rounding, cache sharing, sharding, sliding windows, offload, quantization and fragmentation. **Never treat logical KV bytes as measured per-device VRAM.**

## 6. GPU interconnect and multi-GPU inference

Tensor parallelism splits some model operations across GPUs and adds communication such as all-reduce. Pipeline parallelism partitions layers and introduces pipeline scheduling. Host-to-device transfers can dominate when model loading or offload is involved.

Check GPU topology, NUMA placement, NVLink/PCIe bandwidth, collective performance, network latency and per-device memory balance. A model fitting on two GPUs does not imply 2× throughput or half latency.

## 7. GPU health signals

| Signal | Interpret carefully |
|---|---|
| GPU utilization | fraction of sampled time with active kernels, not efficiency |
| SM active / occupancy | scheduling activity, not direct token throughput |
| HBM bandwidth utilization | pressure on memory subsystem |
| VRAM used/free | capacity, not fragmentation or allocatable KV blocks |
| Power draw / power limit | throttling or workload intensity |
| Temperature / clocks | thermal or power throttling |
| ECC / XID | potential hardware/driver health incidents |
| PCIe/NVLink counters | transfer pressure or topology issues |
| GPU process list | competing workloads, memory allocation |
| DCGM exporter freshness | monitoring coverage and scrape health |

Metric availability depends on GPU model, driver, DCGM/exporter and permissions. Never assume a particular DCGM metric exists without checking the deployed version.

## 8. Safe operational commands

**[SAFE-READ]** On an approved GPU node or authorized GPU troubleshooting Pod:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu,power.draw,clocks.sm --format=csv
nvidia-smi topo -m
```

On approved Kubernetes context:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Access to the device from inside containers requires the correct NVIDIA device plugin/runtime and scheduling; **do not install drivers or privileged DaemonSets as part of read-only diagnostics**.

For a verified inference deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Redact sensitive output. GPU allocation through Kubernetes extended resources is not equivalent to device health or model readiness.

## 9. GPU sizing and FinOps workflow

1. Define input/output token length cohorts, concurrency, model and quantization.
2. Measure model weights and per-runtime workspace.
3. Estimate KV cache with architecture-specific assumptions.
4. Include spare VRAM, fragmentation and model-loading peaks.
5. Benchmark TTFT, TPOT, complete-stream success and tokens/s.
6. Observe GPU clocks, power, bandwidth, temperature and throttling.
7. Account for host CPU/RAM, PCIe/NVLink and storage/model-load time.
8. Model GPU-hours per successful request and per million output tokens.
9. Test failure and recovery capacity; don't size to average utilization only.
10. Record assumptions, approval, rollback and cost envelope.

## 10. GPU incident decision tree

```text
Inference slow or unavailable?
 ├─ no GPU resource on node → device plugin, node labels, allocatable
 ├─ CUDA initialization error → driver/runtime/library compatibility
 ├─ CUDA OOM → weights, KV cache, concurrency, fragmentation
 ├─ VRAM free but alloc fails → allocator/fragmentation/runtime logs
 ├─ high SM utilization → kernel contention / compute pressure
 ├─ high HBM traffic + poor TPOT → bandwidth-limited decode
 ├─ GPU low, queue high → scheduler/admission/CPU/tokenizer
 ├─ power/thermal clocks drop → throttling
 ├─ XID/ECC errors → hardware/driver escalation
 ├─ multi-GPU slower than single → topology/collectives
 ├─ host transfer bottleneck → PCIe/NUMA/offload
 └─ telemetry missing → exporter, permissions, independent synthetic
```

## 11. Twenty production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | no GPU allocatable | device plugin and node |
| F02 | Pod pending GPU | requests/taints/affinity |
| F03 | GPU visible but CUDA fails | driver/runtime compatibility |
| F04 | wrong GPU SKU | node/device inventory |
| F05 | model weights exceed VRAM | weight footprint |
| F06 | KV cache exhaustion | runtime cache blocks |
| F07 | fragmentation allocation failure | allocator logs |
| F08 | decode bandwidth saturation | TPOT/HBM throughput |
| F09 | prefill compute saturation | prompt cohorts/SM |
| F10 | GPU utilization high but tokens/s low | kernels/batching |
| F11 | GPU utilization low with queue growth | scheduler/CPU |
| F12 | thermal throttling | clocks/temperature |
| F13 | power throttling | clocks/power limits |
| F14 | ECC/XID event | device health logs |
| F15 | PCIe transfer pressure | transfer counters |
| F16 | NVLink/topology mismatch | nvidia-smi topo |
| F17 | collective communication slowdown | distributed traces |
| F18 | noisy-neighbor GPU contention | process and tenant inventory |
| F19 | DCGM exporter stale | scrape freshness |
| F20 | GPU cost spike with low success | GPU-hours/good requests |

## 12. Fourteen operational runbooks

1. Verify context and GPU inventory.
2. Triage pending GPU Pods.
3. Validate NVIDIA driver/runtime/device plugin compatibility.
4. Investigate GPU model load failure.
5. Investigate CUDA OOM and KV-cache pressure.
6. Diagnose memory fragmentation and allocation.
7. Separate prefill compute from decode bandwidth pressure.
8. Investigate low utilization with high queue.
9. Investigate thermal and power throttling.
10. Escalate ECC/XID hardware errors.
11. Diagnose multi-GPU topology/collective slowdown.
12. Restore missing DCGM/GPU observability.
13. Perform GPU capacity and cost review.
14. Execute approved mitigation, rollback and customer-path recovery.

Each runbook requires owner, safe evidence, impact, authorization, rollback, verification and follow-up.

## 13. Eight game days

1. GPU resource disappears from allocatable (tabletop).
2. Driver/runtime mismatch after node replacement (tabletop).
3. KV-cache pressure causes CUDA OOM (tabletop).
4. Decode becomes bandwidth constrained (tabletop).
5. GPU thermal throttling increases TPOT (tabletop).
6. Multi-GPU topology regression (tabletop).
7. DCGM exporter goes stale during degradation (tabletop).
8. Low GPU utilization hides high idle cost (tabletop).

Actual fault injection requires separate approved staging change control.

## 14. Sixteen acceptance gates

G01 GPU architecture diagram; G02 SM/tensor core distinctions; G03 HBM hierarchy; G04 roofline arithmetic; G05 prefill/decode bottlenecks; G06 logical KV sizing; G07 VRAM headroom; G08 PCIe/NVLink topology; G09 approved GPU diagnostics; G10 hardware health signals; G11 Kubernetes GPU scheduling; G12 incident triage; G13 safe capacity experiment; G14 FinOps model; G15 rollback/recovery; G16 independent reviewer.

See [companion lab](../labs/011-gpu-architecture-sms-hbm-memory-bandwidth-check.md). All gates remain pending until independently evidenced.

## Next

**Chapter 012 — CUDA Runtime, Drivers, Containers and Compatibility.**
