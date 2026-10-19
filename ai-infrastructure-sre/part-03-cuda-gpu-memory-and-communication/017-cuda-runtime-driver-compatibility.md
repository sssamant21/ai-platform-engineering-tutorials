# Chapter 017 — CUDA Runtime and Driver Compatibility

**Part 03 — CUDA, GPU Memory and Communication · Chapter 017 of 100**

**Status:** Published documentation. Offline tests, GPU execution, independent technical/security review and production acceptance remain pending.

## Learning objectives
Explain CUDA driver/runtime/toolkit boundaries; establish supported driver–CUDA–GPU compatibility evidence; troubleshoot CUDA initialization and device visibility in Linux, containers and Kubernetes; design version-pinned image and node upgrades; test canaries and rollback; and distinguish compatibility failures from hardware and workload failures.

## 1. CUDA software architecture
```text
Application: PyTorch / TensorRT / vLLM / custom CUDA
                   |
Framework + CUDA runtime (libcudart) + cuBLAS/cuDNN/NCCL
                   |
       CUDA Driver API (libcuda.so)
                   |
    NVIDIA user-space driver + kernel modules
                   |
        GPU firmware + compute capability
                   |
        PCIe / NVLink / host platform
```

The **NVIDIA driver** provides the kernel integration and CUDA Driver API. The **CUDA Toolkit** supplies compilers, headers and development libraries; applications may bundle CUDA runtime and framework libraries in container images. The version shown by `nvidia-smi` as “CUDA Version” represents the maximum CUDA version supported by the installed driver according to NVIDIA's reporting conventions; it **does not** prove that the corresponding toolkit is installed, that a container is compatible, or that an application kernel will run.

CUDA Runtime API and Driver API differ in initialization and version interfaces. Distinguish `cudaRuntimeGetVersion` (runtime) from `cudaDriverGetVersion` (driver API). `nvcc --version` reports the compiler/toolkit if installed, and may not exist on runtime-only nodes.

## 2. Compatibility is a matrix, not a single version number
| Boundary | Required evidence | Typical failure |
|---|---|---|
| GPU architecture | device SKU, compute capability, framework binary support | no kernel image available |
| driver ↔ CUDA runtime | NVIDIA compatibility guide, exact versions | insufficient driver |
| kernel ↔ driver module | OS/kernel ABI, signing, DKMS or prebuilt module | driver not loaded |
| container ↔ host | NVIDIA Container Toolkit/CRI and injected libraries | GPU invisible in pod |
| framework ↔ CUDA | official PyTorch/TensorRT/vendor build matrix | symbol or ABI error |
| NCCL ↔ fabric | supported NCCL, GPU/NIC/RDMA stack | collective hang |
| PTX ↔ driver JIT | PTX ISA/toolkit, driver capability | JIT compilation error |
| operator ↔ node image | supported GPU Operator/driver ownership | conflicting installers |

NVIDIA documents **minor-version compatibility** within certain CUDA major families and **forward-compatibility** packages for supported deployments. These have limitations (including feature requirements and PTX JIT constraints). Do not assume all combinations of newer toolkit and older driver work. Use the official NVIDIA CUDA Compatibility documentation and application-vendor support matrix, with explicit approval of each tested pair.

### Required production contract
For each pool record: GPU model/compute capability, OS/kernel, exact driver build, CUDA toolkit/runtime version, framework and NCCL build, GPU Operator/device-plugin version, NVIDIA Container Toolkit version, immutable node image digest, application image digest, Secure Boot state, support evidence URL, staging test result, owner and change ticket.

## 3. Read-only host diagnostics
Run on an authorized GPU node:
```bash
date -u
uname -r
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,compute_cap --format=csv
nvidia-smi
command -v nvcc && nvcc --version || true
cat /proc/driver/nvidia/version 2>/dev/null || true
lsmod | grep '^nvidia' || true
ldconfig -p | grep -E 'libcuda.so|libnvidia-ml.so' || true
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|nvidia' | tail -n 100
```
Not all `nvidia-smi --query-gpu` fields are supported on every version; use `nvidia-smi --help-query-gpu` and omit unsupported fields. Missing `nvcc` on a runtime-only host is not automatically an error. A successful `nvidia-smi` proves NVML access, **not** CUDA kernel execution.

### Optional minimal CUDA C smoke (requires a supported toolkit and GPU)
```cuda
// cuda_smoke.cu — staging only
#include <cstdio>
#include <cuda_runtime.h>
__global__ void add_one(int *x) { if (threadIdx.x == 0) *x += 1; }
int main() {
    int n = 0;
    cudaError_t e = cudaGetDeviceCount(&n);
    if (e != cudaSuccess || n < 1) {
        std::fprintf(stderr, "CUDA discovery: %s, count=%d\n", cudaGetErrorString(e), n);
        return 1;
    }
    int *p = nullptr;
    if (cudaMallocManaged(&p, sizeof(int)) != cudaSuccess) return 2;
    *p = 41;
    add_one<<<1, 1>>>(p);
    e = cudaDeviceSynchronize();
    int result = (e == cudaSuccess) ? *p : -1;
    cudaFree(p);
    if (e != cudaSuccess || result != 42) return 3;
    std::printf("CUDA smoke passed: %d GPUs\n", n);
    return 0;
}
```
Build and run only in an approved staging environment:
```bash
nvcc -O2 cuda_smoke.cu -o cuda_smoke
./cuda_smoke
```
The sample validates device enumeration, allocation, kernel launch and synchronization. It does not certify every GPU, library or workload. If your target GPU/driver cannot run managed-memory code, use a vendor-supported alternative smoke test.

## 4. Containers and Kubernetes
The host driver remains a host responsibility even when the application image includes CUDA runtime libraries. The NVIDIA Container Toolkit integrates device and driver-library access into the supported runtime. Do not install a competing host driver inside application images.

**Read-only checks:**
```bash
nvidia-ctk --version 2>/dev/null || true
crictl info 2>/dev/null | head -n 60 || true
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node APPROVED_GPU_NODE
kubectl get daemonsets -A
kubectl get pods -A -o wide
```
For a staging smoke pod, pin a **verified image digest** from a trusted registry, request `nvidia.com/gpu: 1`, use approved runtime/security controls, and validate CUDA initialization plus an actual kernel. Avoid mutable `:latest` tags. MIG profiles may expose distinct resource names. GPU allocation does not imply compatibility with the workload's compiled architecture.

## 5. Upgrade and rollback
1. Inventory exact driver, runtime, framework, GPU SKU and application image versions; collect compatibility evidence.
2. Select a vendor-supported combination; pin and sign host/application images.
3. Reproduce on staging GPU hardware; test driver module, NVML, CUDA smoke, framework import, inference or training workload and NCCL where applicable.
4. Verify spare GPU capacity, quotas, checkpoints, PDBs, local storage, workload identity and rollback.
5. Canary one node/pool and one representative application image; monitor CUDA errors, Xid, GPU allocation, latency/throughput and error budget.
6. Stop on initialization failure, performance regression, unexpected GPU count, telemetry loss or capacity shortfall.
7. Roll back the **approved host image and/or workload image as a tested combination**, not an arbitrary package downgrade. Preserve UTC evidence and obtain signoff.

## 6. Troubleshooting matrix
| Symptom | First checks | Likely cause |
|---|---|---|
| CUDA driver insufficient | driver version vs app runtime | unsupported pairing |
| `nvidia-smi` fails | kernel module, NVRM, Secure Boot | driver initialization |
| `nvidia-smi` works, CUDA fails | CUDA smoke, runtime libraries | API/runtime mismatch |
| no CUDA devices in pod | device plugin, toolkit, CRI | container integration |
| invalid device ordinal | visible-device mapping/MIG | index mismatch |
| no kernel image available | GPU compute capability, fatbin | unsupported architecture |
| PTX JIT error | driver JIT/toolkit versions | feature limitation |
| NVML library mismatch | loaded user-space and kernel driver | mixed driver installation |
| NCCL timeout | driver/NCCL/fabric/ranks | distributed communication |
| CUDA OOM | allocations, fragmentation, MIG | memory pressure |
| framework import error | pinned wheel/CUDA ABI | binary mismatch |
| Xid after update | kernel logs, workload | driver/hardware regression |

### CUDA errors and safe interpretation
Errors such as insufficient driver, initialization failure, unsupported forward-compatibility path or unsupported GPU architecture must be investigated against the exact NVIDIA documentation and build. Error codes may vary across CUDA releases and APIs; avoid hard-coding a universal mapping from numeric codes to remediation.

## 7. Eight incident simulations
1. **New application image, old driver:** matrix gate rejects rollout; use compatible app image or approved driver canary.
2. **Kernel patch invalidates driver:** restore signed compatible node image after preserving module and NVRM evidence.
3. **GPU visible on host, not in container:** trace device plugin, toolkit, CRI and resource allocation.
4. **PTX JIT failure:** identify target GPU architecture and driver support; rebuild or upgrade through validated path.
5. **NCCL collective stalls:** correlate rank versions, NIC/RDMA, topology and checkpoint; do not classify solely as CUDA mismatch.
6. **MIG profile changes:** reconcile visible devices, memory and plugin resources; requalify workload.
7. **NVML user-space mismatch:** detect competing driver installation ownership and revert to one supported stack.
8. **Canary inference SLO regression:** halt wave, restore old image/pool, validate throughput and latency.

## 8. Operational runbooks
**RB-017-A — Compatibility inventory:** capture driver, runtime, framework, CUDA libraries, GPU SKU, node image and support evidence.

**RB-017-B — CUDA initialization triage:** compare NVML, module, CUDA smoke, logs, runtime version and workload binary.

**RB-017-C — Container GPU failure:** inspect plugin, CRI, toolkit, image digest and resource requests without exposing secrets.

**RB-017-D — Safe driver change:** pin supported image, verify signature, stage, canary, monitor, stop and roll back.

**RB-017-E — Framework/NCCL mismatch:** verify build matrix and fabric, reproduce with minimal smoke and restore checkpoints.

**RB-017-F — GPU architecture/PTX issue:** identify compute capability, compiled targets, driver JIT support and supported rebuild.

**RB-017-G — Post-change acceptance:** prove CUDA kernel, application SLO, GPU count, Xid/ECC, telemetry and rollback.

## 9. Companion executable lab
[Chapter 017 — CUDA Runtime and Driver Compatibility Gate](../labs/017-cuda-runtime-driver-compatibility-check.md) implements an offline **explicit allowlist** validator. It does not infer NVIDIA support from version-number ordering; the approved combinations must come from official vendor matrices and staging evidence.

## 10. 100-point assessment
| Category | Points |
|---|---:|
| CUDA stack and API architecture | 20 |
| Version/architecture compatibility evidence | 20 |
| Host/container/Kubernetes diagnostics | 20 |
| Change safety and incident runbooks | 20 |
| Executable lab and acceptance evidence | 20 |
| **Total** | **100** |

Pass >=90 with no compatibility, security, recovery, capacity or observability hard stop. Published documentation and synthetic tests do **not** equal production qualification.

## References
- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [NVIDIA CUDA Runtime API](https://docs.nvidia.com/cuda/cuda-runtime-api/)
- [NVIDIA CUDA Driver API](https://docs.nvidia.com/cuda/cuda-driver-api/)
- [NVIDIA CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next:** Chapter 018 — GPU Memory Architecture and OOM.
