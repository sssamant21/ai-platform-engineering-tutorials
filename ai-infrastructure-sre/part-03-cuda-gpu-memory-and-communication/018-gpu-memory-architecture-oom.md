# Chapter 018 — GPU Memory Architecture and OOM

**Part 03 — CUDA, GPU Memory and Communication · Chapter 018 of 100**

**Publication:** Complete. **Offline lab execution, live GPU validation, independent technical/security review and production acceptance:** Pending.

## Outcomes
Explain GPU memory hierarchy and allocation behavior; distinguish device-memory exhaustion from Linux/Kubernetes OOM; diagnose CUDA out-of-memory errors, fragmentation, framework caching and MIG isolation; create workload-aware memory budgets; and execute safe recovery and capacity runbooks.

## 1. GPU memory architecture
```text
CPU process / system RAM / page-locked host memory
                |
          PCIe / NVLink
                |
    GPU global memory (HBM / GDDR)
       /          |           \
   model weights  activations  KV cache / workspaces
                |
  GPU L2 cache / SM shared memory / registers
                |
       CUDA contexts / streams / kernels
```

GPU **device memory** is not the same as Linux RAM. Modern GPUs expose a memory hierarchy: device global memory (HBM or GDDR), L2 cache, per-SM shared memory, registers and local memory (which may spill to device memory). CUDA allocations, memory pools, contexts and framework allocators share device capacity with other workloads and driver overhead. Unified/managed memory changes placement and migration semantics but does **not** create unlimited free GPU capacity.

### Major allocation categories
| Category | Common workloads | Operational implication |
|---|---|---|
| Model parameters | inference/training | scales with parameter count and precision |
| Gradients | training | typically additional parameter-sized storage, implementation dependent |
| Optimizer state | training | may dominate with Adam-like optimizers |
| Activations | training/inference | sensitive to batch, sequence length and checkpointing |
| KV cache | autoregressive inference | grows with concurrent tokens and model architecture |
| CUDA contexts/kernels | all | nonzero per-process overhead |
| cuBLAS/cuDNN/NCCL workspaces | all | library-dependent temporary allocations |
| Allocator reserved/cached memory | frameworks | reserved is not equal to active tensors |
| Fragmentation | dynamic shapes/long-lived jobs | allocation may fail despite apparently free capacity |

**Illustrative math, not a production sizing guarantee:** raw parameter storage for a 7-billion-parameter model at 2 bytes/parameter is about 14 billion bytes (about 13.0 GiB), before activations, KV cache, temporary workspaces, quantization metadata, fragmentation, runtime overhead and replicas.

## 2. Four distinct OOM/failure domains
1. **CUDA device OOM:** an allocation on a selected GPU fails because usable device memory or allocator capacity is insufficient.
2. **Framework allocator OOM:** cached/reserved/fragmented memory prevents satisfying a request; inspect allocated, reserved and allocator diagnostics.
3. **Linux cgroup/Kubernetes OOMKilled:** a process exceeds **host RAM** cgroup limits; inspect pod termination reason, node memory pressure and kernel OOM logs.
4. **MIG partition memory exhaustion:** a workload exhausts its allocated GPU instance's memory despite free memory elsewhere on the physical GPU.

Also distinguish device-side assertions, illegal memory access, GPU reset/Xid and CUDA context failure: they can surface during allocation or synchronization but are **not** automatically OOM.

## 3. Read-only discovery and diagnostics
On an approved GPU host:
```bash
date -u
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,memory.total,memory.used,memory.free,utilization.gpu --format=csv
nvidia-smi --query-compute-apps=gpu_uuid,pid,process_name,used_gpu_memory --format=csv
nvidia-smi -q -d MEMORY
nvidia-smi topo -m
free -h
journalctl -k -b --no-pager | grep -Ei 'out of memory|oom-kill|NVRM|Xid' | tail -n 100
```
Per-process GPU memory accounting may be unsupported or restricted by MIG, driver and OS; some values are `N/A`. Treat missing telemetry as unknown, never zero. `nvidia-smi memory.used` is not a direct measurement of live tensors, and high utilization is not necessarily a memory bottleneck.

Kubernetes:
```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl describe pod APPROVED_POD -n APPROVED_NAMESPACE
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node APPROVED_GPU_NODE
kubectl top pod -A
kubectl top node
```
`kubectl top` reports CPU and **host RAM** via metrics-server, not GPU VRAM. A container memory limit does not automatically cap CUDA device memory. Verify pod termination reason (`OOMKilled`), exit code, cgroup counters and framework/CUDA error separately.

## 4. PyTorch memory diagnostics (approved staging)
```python
import torch

if not torch.cuda.is_available():
    raise SystemExit("No CUDA device available")
for i in range(torch.cuda.device_count()):
    with torch.cuda.device(i):
        props = torch.cuda.get_device_properties(i)
        free, total = torch.cuda.mem_get_info()
        print({
            "device": i, "name": props.name,
            "total_bytes": total, "free_bytes": free,
            "allocated_bytes": torch.cuda.memory_allocated(),
            "reserved_bytes": torch.cuda.memory_reserved(),
            "max_allocated_bytes": torch.cuda.max_memory_allocated(),
        })
```
Framework counters primarily describe allocations managed by that framework; they do not necessarily account for other processes, CUDA libraries or non-framework allocations. `torch.cuda.empty_cache()` can release unused cached blocks to the driver but **does not free live tensors**, guarantee a successful allocation or repair a leaking process. Do not run GPU memory stress scripts on shared production GPUs.

## 5. Workload sizing and admission
Use a budget with explicit safety margin:
```text
Required device memory =
  weights + gradients + optimizer state + activations
  + KV cache + CUDA/library workspaces + runtime overhead
  + measured fragmentation and safety margin
```
For inference, cap batch size, context length, concurrent sequences and KV-cache allocation according to measured p95/p99 memory and SLO behavior. For training, adjust microbatch, gradient accumulation, activation checkpointing, precision and optimizer strategy after correctness/performance validation. Tensor/model parallelism distributes some memory but adds communication and does not guarantee linear savings.

**Example planning calculation:** an 80 GiB GPU with 12 GiB reserved for measured overhead and a 10 GiB operational safety margin leaves **58 GiB** for the remaining modeled workload categories. These numbers are synthetic and should not be used as universal thresholds.

### MIG and Kubernetes
MIG partitions expose constrained compute/memory resources; GPU device-plugin resource naming and isolation vary by mode and GPU SKU. Reconcile the actual MIG device UUID, profile, workload visibility and allocatable resource. Do not sum MIG instances as if each were a full physical GPU. Kubernetes extended GPU resource requests are **counts**, not arbitrary byte-level VRAM reservations; use admission and workload controls for memory headroom.

## 6. Observability and SLOs
Track device-memory used/free, allocator allocated/reserved, allocation failures, CUDA OOM rate, per-pod workload identity, MIG profile, batch/sequence/concurrency, queue depth, training step time, inference TTFT/TPOT, host RSS, OOMKilled and telemetry freshness. Avoid unbounded model/prompt/request labels. Alert on repeated failures and SLO burn, not merely high allocated memory.

Correlate time series by UTC timestamp, node, GPU UUID, MIG device, pod and workload release. Account for driver metrics missing during a reset or exporter outage. Preserve privacy and least-privilege access to process metadata.

## 7. Eight incident simulations
1. **Inference KV-cache exhaustion:** increasing context/concurrency causes CUDA OOM; cap admission and tune KV budget.
2. **Training microbatch too large:** activation spike; reduce microbatch, validate gradient accumulation and convergence.
3. **Reserved memory far above allocated:** allocator fragmentation/caching; inspect framework stats and dynamic-shape behavior before tuning.
4. **Pod OOMKilled but GPU has free VRAM:** diagnose host cgroup memory and data-loader workers.
5. **MIG instance OOM:** verify profile/UUID and right-size workload or MIG partition through approved change.
6. **OOM after model rollout:** compare weight precision, tensor parallel layout, max context, image and runtime versions.
7. **GPU disappears after Xid:** distinguish device loss/reset from OOM and escalate hardware.
8. **Telemetry reports zero after exporter failure:** stale data must fail closed; restore DCGM/exporter and independent probes.

## 8. Troubleshooting matrix
| Symptom | Evidence | Likely domain | Action |
|---|---|---|---|
| CUDA out of memory | exception + device free bytes | VRAM/allocator | right-size and profile |
| `OOMKilled` | pod last state/cgroup | host RAM | inspect RSS/limits |
| allocated << reserved | framework memory summary | caching/fragmentation | examine allocator |
| low free VRAM, no listed processes | driver/MIG/permissions | accounting/context | verify visibility |
| OOM on one GPU only | UUID/placement/MIG | skewed allocation | rebalance topology |
| NCCL fails after OOM | rank logs/checkpoints | distributed failure | coordinated recovery |
| illegal memory access | CUDA/Xid logs | kernel/device | debug, not OOM assumption |
| missing memory metrics | DCGM scrape | observability | restore collector |

## 9. Operational runbooks
**RB-018-A — Initial classification:** capture UTC time, exact error, GPU UUID, node, workload, container status and driver logs; distinguish CUDA OOM from Linux OOMKilled.

**RB-018-B — GPU allocation analysis:** collect device free/used, framework allocated/reserved, process list and MIG profile, preserving unsupported metrics as unknown.

**RB-018-C — Inference protection:** reduce admission, batch/concurrency/context through approved controls; validate TTFT/TPOT, error rate and rollback.

**RB-018-D — Training recovery:** confirm durable checkpoint and optimizer state; change microbatch/precision only after correctness review; restart coordinated ranks.

**RB-018-E — MIG and tenancy:** confirm resource request, profile, UUID and isolation; never modify live MIG configuration without approval.

**RB-018-F — Host OOM:** inspect cgroup limits, pod termination, data-loader processes and node memory pressure; avoid changing GPU parameters blindly.

**RB-018-G — Memory leak/fragmentation:** profile repeated iterations and allocator counters; compare before/after release and code paths.

**RB-018-H — Acceptance/RCA:** document memory budget, load-test envelope, SLO, telemetry, owner, regression tests and prevention.

## 10. Companion executable lab
[Chapter 018 — GPU Memory and OOM Admission Gate](../labs/018-gpu-memory-architecture-oom-check.md) provides a Python 3.11+ synthetic workload memory-budget evaluator and pytest tests. It performs **no real GPU allocation** and does not claim hardware readiness.

## 11. 100-point assessment
| Category | Points |
|---|---:|
| GPU memory architecture | 20 |
| OOM failure-domain diagnosis | 20 |
| Budgeting, admission and MIG | 20 |
| Incident response and runbooks | 20 |
| Executable lab and validation evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical tenant isolation, data integrity, observability or SLO blocker.

## References
- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [CUDA Runtime Memory Management](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__MEMORY.html)
- [PyTorch CUDA Memory Management](https://docs.pytorch.org/docs/stable/notes/cuda.html#memory-management)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

**Next:** Chapter 019 — CUDA Processes, Streams and Contexts.
