# Lab 076 — CUDA Initialization and Driver Errors Acceptance Gate

**Chapter:** [076 — CUDA Initialization and Driver Errors](../part-10-incident-response-and-troubleshooting/076-cuda-initialization-driver-errors.md)

**Execution status:** NOT RUN. The following are controlled exercises and evidence requirements, not completed tests.

## 1. Safety and prerequisites

Use a dedicated staging GPU workload for any CUDA probes or changes. Read-only Kubernetes/host diagnostics are preferred in production. No production driver reinstall, GPU reset, containerd restart, node reboot, kernel module unload, MIG reconfiguration or forced workload deletion. Confirm checkpoint, replica, rollback and incident owner before approved remediation.

## 2. Exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | Version inventory | kernel, driver, toolkit, GPU, runtime | Pending |
| E02 | Version semantics | \`nvidia-smi\` vs \`nvcc\` interpretation | Pending |
| E03 | Compatibility matrix | pinned vendor release mapping | Pending |
| E04 | Kubernetes GPU allocation | allocatable and pod resources | Pending |
| E05 | Toolkit ownership | Operator/runtime configuration | Pending |
| E06 | CUDA error capture | first failing API and numeric code | Pending |
| E07 | CUDA device count | staging probe stdout/exit code | Pending |
| E08 | CUDA kernel smoke | bounded staging kernel result | Pending |
| E09 | Synthetic driver too old | diagnosis and safe remediation | Pending |
| E10 | Synthetic invalid ordinal | MIG/container visibility mapping | Pending |
| E11 | Synthetic OCI error | toolkit/runtime fault tree | Pending |
| E12 | Synthetic NVML mismatch | module/user-space evidence | Pending |
| E13 | Xid correlation | UTC timeline and workload impact | Pending |
| E14 | Rollback tabletop | checkpoint, change, recovery plan | Pending |
| E15 | Alerting/recovery | test routing and acceptance | Pending |
| E16 | Review | peer sign-off and score | Pending |

## 3. Read-only diagnostics

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl get pods -A -o wide --field-selector spec.nodeName=<gpu-node>
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> logs <gpu-pod> --timestamps=true --since=2h
kubectl get runtimeclasses
kubectl get pods -A | grep -Ei 'nvidia|gpu-operator|toolkit|device-plugin'
\`\`\`

Approved host only:

\`\`\`bash
uname -r
lsmod | grep '^nvidia'
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version --format=csv
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|nvidia|AER'
\`\`\`

## 4. Staging CUDA probe

Create \`cuda-probe.cu\`:

\`\`\`cpp
#include <cuda_runtime.h>
#include <cstdio>
int main() {
    int count = 0;
    cudaError_t rc = cudaGetDeviceCount(&count);
    std::printf("code=%d message=%s count=%d\n",
                static_cast<int>(rc), cudaGetErrorString(rc), count);
    return rc == cudaSuccess && count > 0 ? 0 : 1;
}
\`\`\`

Compile and execute only in an approved CUDA development image on a staging GPU:

\`\`\`bash
nvcc -o cuda-probe cuda-probe.cu
./cuda-probe
echo "exit_code=$?"
\`\`\`

Record image digest, CUDA toolkit, host driver, GPU SKU, stdout, stderr, return code and test UTC time. This probe verifies enumeration only; a separate approved bounded kernel test is required for kernel execution acceptance.

## 5. Synthetic diagnosis worksheet

**Case A:** host driver supports an older CUDA runtime than the application image requires. Identify compatible driver/toolkit pairing from NVIDIA documentation; propose a pinned image or staged driver change. No live driver upgrade.

**Case B:** host \`nvidia-smi\` succeeds, but container has \`invalid device ordinal\`. Inspect pod GPU request, \`CUDA_VISIBLE_DEVICES\`, device enumeration and MIG strategy.

**Case C:** Pod stuck \`ContainerCreating\` with \`nvidia-container-cli\` error. Inspect toolkit/runtime and GPU Operator ownership; do not restart containerd blindly.

**Case D:** after kernel upgrade, NVML reports a driver/library mismatch. Compare loaded module, user-space libraries and rollout history; stage rollback.

**Case E:** Xid and CUDA init failure coincide. Determine whether the hardware event preceded the CUDA error and whether independent evidence supports causality.

## 6. Evidence template

\`\`\`yaml
chapter: "076"
environment: "STAGING_OR_OFFLINE"
execution_date_utc: null
reviewer: null
kernel_version: null
driver_version: null
cuda_toolkit_version: null
cuda_runtime_version: null
gpu_model: null
gpu_compute_capability: null
container_image_digest: null
container_runtime_version: null
gpu_operator_version: null
device_plugin_version: null
cuda_probe_result: "NOT_RUN"
kernel_smoke_result: "NOT_RUN"
compatibility_matrix_review: "PENDING"
synthetic_cases: "NOT_RUN"
rollback_tabletop: "NOT_RUN"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 7. Acceptance

Chapter gates: **20 + 20 + 20 + 10 + 20 + 10 = 100**. Pass at **≥85** with no critical safety/security failure, validated compatibility and documented rollback. Never mark staging validation complete based on synthetic examples alone.

**Next:** Chapter 077 — GPU OOM and Memory Fragmentation.
