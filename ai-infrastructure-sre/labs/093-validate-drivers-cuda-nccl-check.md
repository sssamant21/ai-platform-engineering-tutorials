# Lab 093 — Drivers, CUDA and NCCL Validation Acceptance Gate

**Chapter:** [093 — Validate Drivers CUDA and NCCL](../part-12-enterprise-operations-and-final-capstones/093-validate-drivers-cuda-nccl.md)

**Status:** NOT EXECUTED. Requires approved GPU staging hardware, driver/framework test images, multi-node fabric access and independent review.

## 1. Safety

Do not install/downgrade GPU drivers, change MIG settings, disable RDMA, reboot production nodes or run fabric-saturating benchmarks without an approved change window. Bound GPU count, payload size, duration and cost; use non-sensitive logs. Capture node identity and approved context.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | Status |
|---|---|---|---|
| E01 | Context/GPU inventory | node/SKU matrix | Pending |
| E02 | Compatibility versions | driver/CUDA/NCCL list | Pending |
| E03 | Host GPU discovery | nvidia-smi output | Pending |
| E04 | Xid/ECC health | sanitized logs | Pending |
| E05 | Kubernetes allocatable | node JSONPath | Pending |
| E06 | Operator/device plugin | pod/events | Pending |
| E07 | Container GPU visibility | smoke logs | Pending |
| E08 | CUDA matmul | correctness output | Pending |
| E09 | GPU/NIC topology | topo matrix | Pending |
| E10 | Intra-node NCCL | all_reduce_perf output | Pending |
| E11 | NIC/RDMA/MTU | fabric inventory | Pending |
| E12 | Multi-node NCCL | rank and transport logs | Pending |
| E13 | Repeatability | median/variance | Pending |
| E14 | Compatibility fault | failure diagnosis | Pending |
| E15 | Canary rollback | change record | Pending |
| E16 | Independent review | rubric and COAs | Pending |

## 3. Read-only Kubernetes preflight

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
kubectl get pods -A | grep -Ei 'gpu-operator|device-plugin|dcgm' || true
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

Record missing permissions or components as \`NOT_ACCESSIBLE\`/\`NOT_INSTALLED\`, not PASS.

## 4. Host-side discovery (approved GPU node only)

\`\`\`bash
uname -r
nvidia-smi
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,name,driver_version,memory.total --format=csv
\`\`\`

Compare actual count, driver and topology with approved hardware/compatibility inventory.

## 5. CUDA compute smoke test

In a CUDA-enabled PyTorch container with an allocated GPU:

\`\`\`python
import torch
assert torch.cuda.is_available()
x = torch.ones((256,256), device="cuda")
y = x @ x
torch.cuda.synchronize()
assert torch.allclose(y, torch.full_like(y, 256.0))
print("CUDA_MATMUL_PASS")
\`\`\`

Expected output **only if executed successfully:** \`CUDA_MATMUL_PASS\`. Record the actual container digest, GPU UUID and framework CUDA build.

## 6. NCCL test plan

**Intra-node:** approved \`nccl-tests\` binary, exact allocated GPU count, bounded 8 MiB–1 GiB payload, warmup and repeated measurements.

\`\`\`bash
# Example for a dedicated eight-GPU staging node:
./build/all_reduce_perf -b 8M -e 1G -f 2 -g 8 -w 5 -n 20
\`\`\`

**Inter-node:** run a separately approved MPI/Kubernetes distributed test with verified rank-to-GPU mapping, interface and fabric. Capture correctness, algbw/busbw, network selection and timeouts. Never claim RDMA passed merely because TCP all-reduce completed.

## 7. Offline scoring arithmetic

\`\`\`python
weights = [15,15,15,15,15,15,10]
assert sum(weights) == 100
scores = [0]*len(weights)  # not executed
assert sum(scores) == 0
print("UNTESTED: score=0/100, acceptance=NOT_EVALUATED")
\`\`\`

This is a rubric sanity check, not a hardware test.

## 8. Evidence manifest

\`\`\`yaml
chapter: "093"
environment: "GPU_STAGING_REQUIRED"
publication: "PUBLISHED"
execution: "NOT_RUN"
execution_date_utc: null
cluster: null
gpu_sku: null
gpu_uuids: []
driver_version: null
kernel_version: null
cuda_runtime: null
framework_build: null
nccl_version: null
operator_version: null
gpu_health: "NOT_VERIFIED"
container_gpu_smoke: "NOT_RUN"
cuda_matmul: "NOT_RUN"
intra_node_nccl: "NOT_RUN"
inter_node_nccl: "NOT_RUN"
fabric_transport_verified: false
baseline_variance: null
rollback_rehearsal: "NOT_RUN"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
corrective_actions: []
\`\`\`

## 9. Acceptance

Score **15 + 15 + 15 + 15 + 15 + 15 + 10 = 100**. Pass ≥85 only after all critical GPU/CUDA/NCCL correctness, isolation, fabric and rollback gates pass with independent review. Publishing the chapter does not validate drivers, CUDA or NCCL.

**Next:** Chapter 094 — Deploy Kubernetes GPU Workloads.
