# Chapter 077 — GPU OOM and Memory Fragmentation

**Part 10 — Incident Response and Troubleshooting | AI Infrastructure Engineering & SRE**

**Publication status:** Documentation published. Offline exercises, GPU staging validation, independent technical review and production acceptance are **pending**.

## 1. Mission and objectives

Diagnose CUDA out-of-memory (OOM) events, allocator fragmentation, retained tensors, GPU memory leaks, activation peaks, MIG memory ceilings and distributed-training memory imbalance without risking production jobs or data integrity.

You will distinguish **true insufficient capacity**, **temporary peak allocation**, **allocator fragmentation**, **application retention/leak**, **other-process consumption**, **MIG partition limits** and **hardware/driver errors**. You will build a reproducible evidence trail and implement a safe, benchmarked remediation plan.

**Prerequisites:** Chapters 071 (capacity SLOs), 075 (GPU node failures), 076 (CUDA initialization); Python/PyTorch familiarity; staging GPU access for optional runtime tests. Never inject OOM into production.

## 2. GPU memory model: know the layers

\`\`\`text
Physical GPU framebuffer (FB)
  ├─ driver/context and non-framework allocations
  ├─ other processes / MPS / sharing overhead
  ├─ framework allocator reserved pool
  │   ├─ live tensors / model parameters / gradients / optimizer state
  │   ├─ activations / attention workspace / temporary kernels
  │   └─ cached/free allocator blocks (reusable, not necessarily releasable)
  └─ unavailable / reserved system memory
\`\`\`

In MIG mode, a workload's visible GPU memory limit corresponds to its assigned instance, **not** the total framebuffer of the physical GPU. CUDA context, driver, NCCL and other libraries may allocate memory outside PyTorch's caching allocator. Therefore, \`nvidia-smi\`, DCGM framebuffer gauges and \`torch.cuda.memory_*\` statistics need not match.

### 2.1 Key distinctions

| Signal | Meaning | Caution |
|---|---|---|
| \`nvidia-smi\` FB used | Device-reported framebuffer usage | Includes non-PyTorch and other-process usage |
| \`DCGM_FI_DEV_FB_USED\` | DCGM framebuffer used, if exported | Verify unit, labels, MIG scope and scrape freshness |
| \`torch.cuda.memory_allocated()\` | Memory occupied by live tensors managed by PyTorch allocator | Not all CUDA allocations |
| \`torch.cuda.memory_reserved()\` | Total memory managed by PyTorch caching allocator | Reserved does not mean all live |
| \`torch.cuda.max_memory_allocated()\` | Peak live-tensor allocation since reset/start | Reset only in controlled diagnostic window |
| \`torch.cuda.mem_get_info()\` | CUDA free/total memory reported for visible device | Can include non-framework consumers |
| \`torch.cuda.memory_summary()\` | Detailed allocator stats and fragmentation hints | Backend/version dependent |
| \`torch.cuda.memory_snapshot()\` | Allocator history/state when supported | Can be large and reveal workload details |

PyTorch memory statistics are scoped to a device and allocator backend. Under \`cudaMallocAsync\`, some allocator statistics are not meaningful; verify the installed PyTorch version and backend.

## 3. Failure taxonomy

| Failure class | Pattern | Diagnostic test |
|---|---|---|
| Capacity exhaustion | allocated + required peak exceeds device/MIG capacity | peak memory, model size, batch/sequence |
| Activation spike | OOM on long sequence or batch outlier | input shape distribution and max tokens |
| Fragmentation | substantial reserved-but-unallocated memory; allocation fails despite apparent headroom | allocator summary and snapshot |
| Tensor retention | allocated memory rises per iteration | graph/tensor references and profiling |
| Other-process contention | device FB high but PyTorch allocated low | process inventory and sharing policy |
| NCCL/communication overhead | distributed ranks consume unequal memory | per-rank stats, collectives and buffers |
| MIG mismatch | slice smaller than expected | pod resource name, MIG profile and CUDA-visible device |
| Kernel/driver fault | OOM-like application failure with Xid/illegal access | first CUDA error, NVRM journal |
| Host/cgroup OOM | Linux OOMKilled, not CUDA OOM | pod status, cgroup/kernel events |

**Important:** \`OOMKilled\` is a Kubernetes/container memory kill and is **not automatically a GPU framebuffer OOM**. A CUDA OOM exception does not normally require a node reboot.

## 4. Incident triage — first 15 minutes

### 4.1 Kubernetes and workload evidence

\`\`\`bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> get pod <gpu-pod> -o json
kubectl -n <namespace> logs <gpu-pod> --all-containers=true --timestamps=true --since=2h
kubectl -n <namespace> logs <gpu-pod> --previous --timestamps=true
kubectl describe node <gpu-node>
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

Capture **first error** (not just the last stack trace), GPU model/MIG profile, pod UID, image digest, model version, batch size, sequence length, precision, optimizer, gradient accumulation, GPU memory and last successful checkpoint. \`--previous\` only works for an available previous terminated container.

### 4.2 Host GPU evidence (approved access)

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,memory.total,memory.used,memory.free --format=csv
nvidia-smi --query-compute-apps=pid,process_name,used_gpu_memory --format=csv
nvidia-smi -q -d MEMORY,COMPUTE,ECC
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|nvidia|oom'
\`\`\`

Process visibility can be restricted by MIG, permissions or platform configuration. Avoid disclosing other tenants' process names/identifiers in shared tickets.

### 4.3 Prometheus/DCGM investigation

\`\`\`promql
# Verify these are present and confirm exporter units/labels first
DCGM_FI_DEV_FB_USED
DCGM_FI_DEV_FB_FREE
DCGM_FI_DEV_FB_TOTAL
DCGM_FI_DEV_GPU_UTIL

# Observed memory usage trend, not an independent capacity denominator
max_over_time(DCGM_FI_DEV_FB_USED[1h])
\`\`\`

Do not infer an application leak from aggregate FB usage alone. Correlate memory growth with the specific workload, input shape, allocator stats and other consumers. Scrape intervals can miss short allocation spikes.

## 5. PyTorch memory forensics

### 5.1 Non-disruptive in-application logging

In an **approved staging application**:

\`\`\`python
import torch

def gpu_memory_report(label: str, device: int = 0):
    if not torch.cuda.is_available():
        raise RuntimeError("CUDA unavailable")
    torch.cuda.synchronize(device)  # diagnostic only; may add overhead
    free, total = torch.cuda.mem_get_info(device)
    print({
        "label": label,
        "allocated_mib": round(torch.cuda.memory_allocated(device) / 2**20, 1),
        "reserved_mib": round(torch.cuda.memory_reserved(device) / 2**20, 1),
        "peak_allocated_mib": round(torch.cuda.max_memory_allocated(device) / 2**20, 1),
        "device_free_mib": round(free / 2**20, 1),
        "device_total_mib": round(total / 2**20, 1),
    })
\`\`\`

Instrument before and after forward, backward, optimizer step and checkpoint. The \`synchronize\` call is intentionally for diagnostics; it can distort performance and should not be placed unconditionally in production hot paths.

### 5.2 Common tensor-retention mistakes

\`\`\`python
# BAD: retains autograd graphs each step
loss_history.append(loss)

# BETTER: store a scalar detached from graph
loss_history.append(loss.detach().item())

# BAD: appends outputs that still reference computation graph
outputs.append(model(inputs))

# BETTER for metrics-only outputs (validate semantics first)
outputs.append(model(inputs).detach().cpu())
\`\`\`

Even detached CPU outputs consume host memory; use bounded buffers. \`torch.no_grad()\` or \`torch.inference_mode()\` may reduce inference memory but must not be used around training gradients.

### 5.3 Fragmentation versus capacity

\`reserved - allocated\` indicates cached/allocator-managed memory not occupied by live tensors, **not** a definitive fragmentation measure. A failed large allocation despite apparent free memory can result from allocator layout, context overhead, stream behavior or competing processes. Confirm with memory summary/snapshot and repeatable workload inputs.

\`torch.cuda.empty_cache()\` releases some unoccupied cached blocks to the driver; it **does not free live tensors** and is not a general OOM fix. Frequent calls can reduce performance and cause allocator churn. \`PYTORCH_CUDA_ALLOC_CONF\` is an alias for the current \`PYTORCH_ALLOC_CONF\` configuration interface in newer PyTorch versions; allocator options such as \`max_split_size_mb\` apply only to relevant backends and should be tested against the installed release. Never set arbitrary values fleet-wide without measurement.

## 6. Training memory budget

Approximate components for mixed-precision training:

\`\`\`text
peak_gpu_memory =
    model_parameters
  + gradients
  + optimizer_states
  + activations
  + temporary_workspaces
  + communication_buffers
  + runtime_context_and_allocator_overhead
\`\`\`

For a **simple FP32 Adam baseline** without weight sharing, sharding, fused optimizers or mixed precision, parameters (~4 bytes), gradients (~4), first/second moments (~8) total roughly **16 bytes per parameter** before activations and overhead. This is an illustration, not a universal model: mixed precision can retain master weights, and ZeRO/FSDP/offload materially change memory.

Peak activation memory depends strongly on batch size, sequence length, architecture, attention implementation, checkpointing and recomputation. Prefer actual per-rank peaks over formula-only sizing.

### 6.1 Remediation tradeoffs

| Change | Potential benefit | Cost/risk |
|---|---|---|
| Reduce microbatch size | lower activation peak | lower throughput, gradient noise if global batch changes |
| Gradient accumulation | preserve global batch with smaller microbatches | longer steps and optimizer semantics |
| Activation checkpointing | recompute activations instead of retaining | extra compute; RNG/determinism considerations |
| Mixed precision (BF16/FP16) | smaller activations/selected weights | numerical validation and hardware support |
| FSDP/ZeRO sharding | distribute parameters/gradients/states | communication, configuration complexity |
| CPU/NVMe offload | reduce GPU residency | bandwidth, host memory, checkpoint complexity |
| Sequence bucketing/caps | avoid pathological long-input peaks | product behavior and quality changes |
| Model/optimizer changes | reduce state size | requires correctness and accuracy acceptance |
| Allocator tuning | may reduce fragmentation | backend-specific, can worsen performance |

Do not change model precision, training batch semantics or checkpoint strategy without workload-owner approval and reproducible quality tests.

## 7. Inference-specific memory pressure

Inference OOM is often driven by **KV cache growth**, concurrent requests, context length, batching and model weight residency. Capacity plans must include worst-case tokens and active sequences, not average GPU utilization. A service can be low on SM busy yet unable to admit another request because memory is exhausted.

For LLM serving, distinguish model weights, KV cache, activation/workspace and memory reserved by serving engines. Validate admission control, maximum context, prefill/decode concurrency, memory utilization settings and latency SLO. Do not lower safety margins solely to maximize throughput.

## 8. MIG, sharing and multi-tenancy

Confirm whether a pod receives a full GPU, MIG slice or time-sliced/shared GPU. MIG framebuffer isolation and per-instance accounting differ from time-slicing, which does not provide the same memory isolation. A model sized for a full GPU may OOM on a smaller MIG instance even if the host GPU has free memory outside that instance. Never recommend cross-tenant process termination as a remediation.

## 9. Recovery sequence

1. Classify CUDA OOM vs Linux \`OOMKilled\` vs kernel/driver failure.
2. Capture first exception, workload inputs and allocator/device metrics.
3. Verify model/precision/batch/context, GPU SKU/MIG and other consumers.
4. Reproduce in isolated staging using sanitized, bounded inputs.
5. Test **one** remediation at a time; compare peak memory, throughput, latency, model quality and stability.
6. Validate checkpoint and training restart semantics or serving replica redundancy.
7. Canary rollout with rollback threshold and owner approval.
8. Confirm p99 latency, job progress, queue wait and GPU memory headroom.
9. Record the evidence and update capacity planning.

## 10. Sixteen hands-on exercises

1. Inventory GPU memory, MIG profile, driver and framework version.
2. Capture GPU framebuffer baseline and per-process use.
3. Verify DCGM FB metric units, labels and freshness.
4. Capture PyTorch allocated/reserved/peak memory.
5. Demonstrate a synthetic CUDA OOM with a bounded staging allocation.
6. Distinguish CUDA OOM from Kubernetes \`OOMKilled\`.
7. Demonstrate tensor retention in a small staging loop.
8. Correct retention and compare memory slopes.
9. Reproduce long-sequence activation peak with synthetic inputs.
10. Compare microbatch sizes and peak memory.
11. Compare activation checkpointing tradeoffs.
12. Assess mixed-precision memory versus numerical correctness.
13. Diagnose allocator fragmentation using snapshots or summary.
14. Diagnose a synthetic MIG slice capacity mismatch.
15. Tabletop distributed rank imbalance and safe checkpoint recovery.
16. Produce an SRE runbook, evidence, review and score.

## 11. Troubleshooting matrix

| Symptom | First check | Avoid |
|---|---|---|
| CUDA OOM with free FB | allocator layout, peak, other consumers | assuming metrics are synchronized |
| Reserved much larger than allocated | cache versus fragmentation | treating difference as leaked memory |
| OOM after many iterations | retained graphs, input variability | blanket \`empty_cache()\` |
| OOM only on one rank | uneven batch, model shard, NCCL buffers | restarting all nodes blindly |
| OOM on MIG slice | profile size and assigned device | using host physical FB total |
| Container \`OOMKilled\` | cgroup/host memory limits | debugging only CUDA |
| Inference OOM at low GPU util | KV cache, concurrency, context | autoscaling based on GPU busy alone |
| Xid coincides with OOM | kernel log, first error, GPU health | declaring memory cause without proof |

## 12. Production runbooks

**A — Training CUDA OOM:** preserve error, checkpoint and per-rank memory; classify capacity vs retention vs fragmentation; stage microbatch/precision/sharding changes; validate resumed training and loss behavior.

**B — Inference KV-cache exhaustion:** inspect context/concurrency and serving-engine memory, enforce approved admission limits, scale compatible capacity or adjust batching; verify p99 and error SLO.

**C — Fragmentation diagnosis:** collect allocator stats and memory snapshot in staging, confirm reproducibility, benchmark backend-specific allocator tuning, deploy canary only if measurable benefit.

**D — MIG memory mismatch:** map pod GPU resource to actual MIG profile, verify memory ceiling and workload requirement, choose compatible placement without changing other tenants' partitions.

**E — Host versus device OOM:** distinguish \`OOMKilled\` and CUDA exception via pod status, cgroup, driver and framework metrics; remediate correct layer; verify no data corruption.

## 13. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Memory accounting and device/MIG inventory | 20 |
| CUDA vs host OOM classification | 15 |
| PyTorch allocator and retention diagnostics | 20 |
| Training/inference memory budgeting | 15 |
| Staging remediation and rollback safety | 20 |
| Runbooks, evidence and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical security, data-integrity or unapproved-production-impact failure. Documentation publication is not a passed runtime test.

## 14. References

- [PyTorch CUDA Memory Management](https://docs.pytorch.org/docs/stable/notes/cuda.html#memory-management)
- [PyTorch CUDA Memory API](https://docs.pytorch.org/docs/stable/cuda.html#memory-management)
- [PyTorch CUDA Memory Snapshot](https://docs.pytorch.org/docs/stable/torch_cuda_memory.html)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

**Next:** Chapter 078 — NCCL Hangs and Distributed Deadlocks.
