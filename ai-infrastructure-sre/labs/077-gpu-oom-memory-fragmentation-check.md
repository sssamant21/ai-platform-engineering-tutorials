# Lab 077 — GPU OOM and Memory Fragmentation Acceptance Gate

**Chapter:** [077 — GPU OOM and Memory Fragmentation](../part-10-incident-response-and-troubleshooting/077-gpu-oom-memory-fragmentation.md)

**Execution status:** NOT RUN. Synthetic examples and code do not establish live GPU validation.

## 1. Safety

All memory stress tests must run in a disposable, quota-limited **staging GPU workload**. Do not intentionally exhaust production GPUs, interfere with shared MIG devices or terminate other tenants' processes. Agree on GPU memory budget, timeout, rollback and owner approval. Never copy customer prompts or model weights into public evidence.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | Device and MIG inventory | SKU, FB, driver, PyTorch | Pending |
| E02 | Framebuffer baseline | redacted process and FB samples | Pending |
| E03 | DCGM semantics | HELP/TYPE, units, labels | Pending |
| E04 | PyTorch allocator | allocated/reserved/peak report | Pending |
| E05 | Synthetic bounded CUDA OOM | staging-only expected exception | Pending |
| E06 | Host vs CUDA OOM | pod status and exception comparison | Pending |
| E07 | Tensor retention | memory trend and code evidence | Pending |
| E08 | Retention fix | before/after slopes | Pending |
| E09 | Activation spike | bounded sequence length test | Pending |
| E10 | Microbatch tuning | peak memory and throughput | Pending |
| E11 | Activation checkpointing | peak/compute tradeoff | Pending |
| E12 | Precision test | memory and numerical checks | Pending |
| E13 | Allocator analysis | memory summary/snapshot | Pending |
| E14 | MIG mismatch | synthetic profile capacity reasoning | Pending |
| E15 | Rank imbalance tabletop | checkpoint/restart decision | Pending |
| E16 | Acceptance | reviewer, runbook and score | Pending |

## 3. Read-only commands

\`\`\`bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> logs <gpu-pod> --timestamps=true --since=2h
kubectl -n <namespace> get pod <gpu-pod> -o json
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

Approved GPU host only:

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,memory.total,memory.used,memory.free --format=csv
nvidia-smi --query-compute-apps=pid,process_name,used_gpu_memory --format=csv
\`\`\`

## 4. Bounded PyTorch diagnostic (staging only)

Run in an approved GPU staging environment with a **small**, configurable memory budget. This script does not deliberately fill the GPU.

\`\`\`python
import torch

if not torch.cuda.is_available():
    raise SystemExit("SKIP: no CUDA GPU available")
device = "cuda:0"
print("PyTorch", torch.__version__)
free, total = torch.cuda.mem_get_info(device)
print("GPU free/total MiB:", round(free/2**20), round(total/2**20))
# Allocate at most 16 MiB and no more than 1% of reported free memory.
bytes_to_allocate = min(16 * 2**20, int(free * 0.01))
if bytes_to_allocate < 4 * 1024:
    raise SystemExit("SKIP: insufficient staging memory headroom")
elements = bytes_to_allocate // 4
x = torch.zeros(elements, dtype=torch.float32, device=device)
torch.cuda.synchronize()
print("allocated MiB:", round(torch.cuda.memory_allocated()/2**20, 2))
print("reserved MiB:", round(torch.cuda.memory_reserved()/2**20, 2))
print("peak allocated MiB:", round(torch.cuda.max_memory_allocated()/2**20, 2))
del x
\`\`\`

This is a **safe memory accounting probe**, not a deliberate OOM injection. For E05, use a reviewed synthetic error fixture or a separately approved tightly bounded staging-only test; never allocate until device exhaustion.

## 5. Synthetic memory budget exercise

Assume a simple FP32 Adam training configuration with 1 billion parameters and no sharding/offload:

\`\`\`python
params = 1_000_000_000
bytes_per_param = 4 + 4 + 8  # parameter + gradient + two Adam moments
base_gib = params * bytes_per_param / 2**30
print("Approximate baseline GiB before activations:", round(base_gib, 2))
\`\`\`

Expected baseline is approximately **14.90 GiB** before activations, workspaces, fragmentation, CUDA context and communication buffers. This is an illustration, not a model-specific capacity estimate.

## 6. Failure classification worksheet

**Case A:** \`torch.cuda.OutOfMemoryError\` but pod was not \`OOMKilled\`. Investigate GPU memory and allocator, not Linux cgroup OOM.

**Case B:** \`OOMKilled\` exit 137 and no CUDA exception. Investigate host/container RAM pressure and limits.

**Case C:** allocated memory grows monotonically each training step. Inspect retained autograd graphs and unbounded collections.

**Case D:** MIG instance has 10 GiB usable FB but model assumes a full GPU. Correct placement/model budget, not host memory totals.

**Case E:** only one distributed rank OOMs. Check rank-local inputs, shards, activations and NCCL buffers.

## 7. Evidence template

\`\`\`yaml
chapter: "077"
environment: "STAGING_OR_OFFLINE"
execution_date_utc: null
reviewer: null
gpu_model: null
mig_profile: null
driver_version: null
pytorch_version: null
allocator_backend: null
workload_image_digest: null
baseline_fb_used_mib: null
baseline_fb_free_mib: null
torch_allocated_mib: null
torch_reserved_mib: null
torch_peak_allocated_mib: null
oom_classification: "NOT_RUN"
retention_analysis: "NOT_RUN"
activation_peak_analysis: "NOT_RUN"
staging_remediation: "NOT_RUN"
numerical_correctness: "NOT_RUN"
rollback_review: "PENDING"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 8. Acceptance and stop conditions

Chapter score: **20 + 15 + 20 + 15 + 20 + 10 = 100**. Pass at **≥85** with no critical safety, security or data-integrity issue. Stop for uncontrolled GPU memory pressure, impact on shared workloads, data exposure or unauthorized remediation. Mark offline execution, staging validation and peer review separately.

**Next:** Chapter 078 — NCCL Hangs and Distributed Deadlocks.
