# Chapter 076 — CUDA Initialization and Driver Errors

**Part 10 — Incident Response and Troubleshooting | AI Infrastructure Engineering & SRE**

**Status:** Published documentation. Offline execution, live staging tests, security review and independent technical acceptance remain pending.

## 1. Objectives

Diagnose failures between NVIDIA GPU hardware, kernel driver, NVML, CUDA Driver API, CUDA Runtime API, NVIDIA Container Toolkit and Kubernetes GPU workloads. Distinguish unsupported versions, device visibility, container injection, permissions, driver initialization and actual GPU hardware faults; recover with minimal disruption and evidence-backed rollback.

**Prerequisites:** Chapter 075, GPU Operator/device plugin basics, Linux kernel troubleshooting and Kubernetes workload administration. **Scope:** approved read-only production diagnostics; any driver change, reboot, pod restart or toolkit modification only in an authorized staging or maintenance window.

## 2. Compatibility model: five independent layers

\`\`\`text
GPU hardware / firmware / PCIe
  → NVIDIA kernel modules + host driver + NVML
     → CUDA Driver API (libcuda.so.1, supplied by host driver)
        → CUDA Runtime / application libraries (libcudart, cuBLAS, cuDNN)
           → NVIDIA Container Toolkit / OCI runtime injection
              → Kubernetes device plugin / GPU allocation
                 → application initialization and kernels
\`\`\`

A successful \`nvidia-smi\` proves NVML can communicate with at least some GPUs; it does **not** prove that the CUDA application can load its libraries, allocate memory or execute kernels. The “CUDA Version” displayed by \`nvidia-smi\` is the **maximum CUDA driver API version supported by the driver**, not the installed toolkit version. \`nvcc --version\` describes a toolkit compiler when present; production runtime images may not include \`nvcc\`.

### 2.1 Compatibility pitfalls

| Layer | Failure pattern | Verification |
|---|---|---|
| Kernel ↔ NVIDIA module | module missing, Secure Boot signing, DKMS mismatch | \`uname -r\`, \`lsmod\`, \`modinfo\`, \`dmesg\` |
| Driver ↔ user-space NVML | driver/library version mismatch | \`nvidia-smi\`, installed driver packages, journal |
| Driver ↔ CUDA runtime | insufficient driver for toolkit/runtime | official NVIDIA CUDA compatibility matrix |
| GPU architecture ↔ binary | unsupported compute capability, no compatible cubin/PTX | GPU model, binary build flags, CUDA error |
| Host ↔ container toolkit | OCI injection failure, runtime handler mismatch | containerd config, toolkit logs, pod events |
| Kubelet ↔ device plugin | GPU resource not advertised | plugin logs, node capacity/allocatable |
| Workload ↔ device visibility | wrong \`CUDA_VISIBLE_DEVICES\`, MIG profile, permissions | env, device enumeration, allocation |
| Driver ↔ hardware | Xid, GPU fallen off bus, failed initialization | kernel journal, PCIe and vendor diagnostics |

**Important:** CUDA minor-version compatibility and forward-compatibility packages have restrictions and GPU/driver requirements. Do not assume every toolkit works with any older driver. Pin the actual matrix for your CUDA release, OS, GPU generation and container image.

## 3. Failure taxonomy and error mapping

| Symptom / message | Common causes | First safe action |
|---|---|---|
| \`CUDA driver version is insufficient for CUDA runtime version\` | runtime newer than supported host driver | check actual driver/runtime matrix |
| \`no CUDA-capable device is detected\` | no visible GPU, device permissions, driver failure | \`nvidia-smi -L\`, allocation and device visibility |
| \`CUDA initialization error\` / \`cudaErrorInitializationError\` | driver init, runtime state, device failure | correlate host logs, container runtime and GPU status |
| \`cudaErrorNoDevice\` | no compatible visible device | check GPU assignment and environment |
| \`invalid device ordinal\` | process selects GPU index not visible in container | check enumerated device count and mapping |
| \`CUDA out of memory\` | allocation exceeds free usable memory | see Chapter 077; do not assume driver bug |
| \`unspecified launch failure\` | prior asynchronous kernel error, hardware or illegal access | preserve earliest CUDA error and logs |
| \`invalid device function\` / \`no kernel image available\` | binary compiled without target GPU support | inspect compute capability and build |
| \`Failed to initialize NVML\` | driver/module or NVML library mismatch, access | inspect host driver, package versions |
| \`nvidia-container-cli: initialization error\` | toolkit/runtime/driver injection failure | pod events and OCI/runtime logs |

Error strings are context-dependent; use exact numeric codes, API call, driver/toolkit version and first failure time to avoid misclassification. CUDA kernel errors may surface **asynchronously** at a later API call.

## 4. Read-only triage: 15-minute runbook

### 4.1 Kubernetes scope and scheduling

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl get pods -A -o wide --field-selector spec.nodeName=<gpu-node>
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> logs <gpu-pod> --all-containers=true --timestamps=true --since=2h
kubectl get pods -A | grep -Ei 'nvidia|gpu-operator|device-plugin|toolkit'
\`\`\`

Record pod UID, container image **digest**, requested GPU/MIG resource, node, start time, restart count and runtime handler. A Pod in \`Pending\` is a scheduling/placement issue; \`ContainerCreating\` with OCI errors suggests toolkit/runtime injection; a Running pod that fails CUDA initialization suggests application/driver/device visibility.

### 4.2 Host driver and GPU evidence

On an approved host:

\`\`\`bash
uname -r
lsmod | grep -E '^nvidia'
modinfo nvidia | head -35
nvidia-smi -L
nvidia-smi --query-gpu=index,uuid,name,driver_version,memory.total --format=csv
nvidia-smi -q -d COMPUTE,ECC,PCI,PERFORMANCE
journalctl -k --since '-2 hours' --output=short-iso-precise | grep -Ei 'NVRM|Xid|nvidia|AER'
\`\`\`

If \`nvidia-smi\` fails, avoid repeated disruptive probes. Correlate driver installation, kernel version, Secure Boot module signing, GPU PCIe visibility and NVRM/Xid logs. Escalate Xid 79/serious ECC indicators to GPU hardware support; do not attempt unapproved reset.

### 4.3 Container and CUDA evidence

Within an **already-running, approved** GPU workload:

\`\`\`bash
# Use only if authorized to exec into the workload container:
kubectl -n <namespace> exec <gpu-pod> -- sh -c \
  'env | grep -E "^(CUDA_VISIBLE_DEVICES|NVIDIA_VISIBLE_DEVICES|NVIDIA_DRIVER_CAPABILITIES)=" || true'
kubectl -n <namespace> exec <gpu-pod> -- sh -c \
  'command -v nvidia-smi >/dev/null && nvidia-smi -L || true'
kubectl -n <namespace> exec <gpu-pod> -- sh -c \
  'ldconfig -p 2>/dev/null | grep -E "libcuda.so|libcudart.so" || true'
\`\`\`

Minimal images may not contain \`sh\`, \`ldconfig\` or \`nvidia-smi\`; absence of these tools is not proof of GPU failure. Avoid dumping full environment variables, which may expose secrets. CUDA device enumeration and kernel execution must be tested with a reviewed application-specific probe.

## 5. Driver API versus runtime API diagnostics

The CUDA Driver API typically initializes via \`cuInit(0)\`, then queries device count; the Runtime API uses functions such as \`cudaGetDeviceCount()\`. A tiny staging probe can identify which layer fails:

\`\`\`cpp
// Build only inside an approved CUDA development image in staging.
#include <cuda_runtime.h>
#include <cstdio>
int main() {
    int count = 0;
    cudaError_t rc = cudaGetDeviceCount(&count);
    std::printf("cudaGetDeviceCount: code=%d message=%s devices=%d\n",
                static_cast<int>(rc), cudaGetErrorString(rc), count);
    return rc == cudaSuccess && count > 0 ? 0 : 1;
}
\`\`\`

Compile with the matching toolkit (\`nvcc -o cuda-probe cuda-probe.cu\`) and execute only in an approved GPU staging pod. A successful device count does **not** prove model execution, memory stability or full CUDA kernel compatibility; extend to a bounded kernel smoke test if authorized.

## 6. NVIDIA Container Toolkit and Kubernetes

The NVIDIA Container Toolkit configures runtime integration to expose GPU devices and host driver libraries inside containers. GPU Operator installations may manage toolkit configuration; manual host changes can conflict with Operator reconciliation.

Read-only checks:

\`\`\`bash
kubectl get runtimeclasses
kubectl get pods -A | grep -Ei 'toolkit|gpu-operator|device-plugin'
kubectl -n <operator-namespace> logs <toolkit-pod> --timestamps=true --since=2h
kubectl -n <plugin-namespace> logs <plugin-pod> --timestamps=true --since=2h
\`\`\`

If \`nvidia-container-cli\` reports an OCI hook error, inspect containerd/CRI configuration, toolkit/driver version and host permissions. Do not blindly run \`nvidia-ctk runtime configure\` or restart containerd on production nodes: such actions can disrupt all containers on the node.

## 7. MIG and CUDA visibility

For MIG-enabled GPUs, CUDA device enumeration and memory availability reflect assigned MIG instances, not necessarily physical GPUs. Confirm device-plugin MIG strategy, pod resource requests and the GPU UUID/MIG mapping. \`CUDA_VISIBLE_DEVICES\` values are container-relative indices/identifiers; never assume host index 0 equals container index 0. GPU memory errors on a MIG slice are not evidence that the whole physical GPU is full.

## 8. Recovery strategy and rollback

**Decision sequence:**
1. **Classify:** host driver failure, container toolkit failure, device plugin, CUDA version, application binary, GPU hardware.
2. **Contain:** isolate impact at workload/node scope without evicting healthy jobs.
3. **Preserve:** exact image digest, driver/kernel/toolkit versions, first error, GPU UUID and Xid logs.
4. **Validate compatibility:** vendor matrix and supported GPU architecture.
5. **Plan change:** checkpoint state, replicas, PodDisruptionBudgets, node headroom, maintenance window and rollback image.
6. **Canary:** one disposable staging GPU workload/node.
7. **Restore:** verify NVML, CUDA device count, bounded kernel smoke test, job progress and service SLO.
8. **Handoff:** document root cause confidence, vendor escalation and prevention.

Do **not** reset GPUs, unload kernel modules, change drivers or reboot nodes automatically based on a single CUDA initialization error. Any change must be explicitly approved and reversible.

## 9. Alerting and dashboards

Correlate:
- Node Ready, GPU capacity/allocatable and plugin DaemonSet health.
- Driver/NVRM Xid logs and DCGM health (confirm metric semantics).
- Container creation failures, CrashLoopBackOff, CUDA probe failures.
- Inference errors, p99 latency, training step stalls and queue admission SLO.
- Recent node-image/driver/toolkit rollout.

Alert severity follows **workload impact and recurrence**; a single failed canary is not necessarily a fleet-wide page. Distinguish telemetry blindness from healthy GPU state.

## 10. Sixteen hands-on exercises

1. Inventory driver, kernel, CUDA toolkit, GPU SKU and container runtime.
2. Verify \`nvidia-smi\` CUDA-version interpretation versus \`nvcc\`.
3. Map the CUDA driver/runtime/toolkit compatibility matrix.
4. Reconcile node GPU allocatable, plugin and pod requests.
5. Inspect NVIDIA Container Toolkit/Operator ownership.
6. Capture a redacted CUDA initialization error with exact API code.
7. Run approved staging \`cudaGetDeviceCount\` probe.
8. Validate a bounded CUDA kernel smoke test in staging.
9. Diagnose synthetic “insufficient driver” error.
10. Diagnose synthetic “invalid device ordinal” under MIG.
11. Diagnose synthetic \`nvidia-container-cli\` OCI error.
12. Diagnose synthetic NVML mismatch after kernel upgrade.
13. Correlate an Xid event with CUDA failure and job impact.
14. Tabletop driver rollback during distributed training.
15. Validate alert routing and post-recovery acceptance criteria.
16. Complete evidence, runbooks, reviewer and 100-point score.

## 11. Troubleshooting matrix

| Symptom | Check | Avoid |
|---|---|---|
| \`nvidia-smi\` works, CUDA fails | runtime version, binary, permissions, container | declaring hardware healthy |
| \`nvidia-smi\` fails | kernel module, NVML, PCIe, logs | reboot-first diagnosis |
| OCI GPU injection fails | toolkit, runtime class, device plugin | unapproved containerd restart |
| Driver too old | compatibility matrix, pinned image | random package upgrades |
| CUDA no device | allocation, visibility, MIG | changing host device indices |
| CUDA invalid function | compute capability and build flags | driver-only fix |
| CUDA OOM | memory usage, allocator, MIG | treating as init error |
| GPU falls off bus | Xid/PCIe, vendor diagnostics | auto GPU reset |

## 12. Production runbooks

**A — CUDA initialization failure:** capture first error and API, inspect GPU assignment and driver/runtime versions, run approved staging probe, remediate narrowest layer, confirm workload SLO.

**B — NVIDIA Container Toolkit/OCI failure:** inspect pod events, toolkit and CRI logs, runtime handler and GPU Operator ownership; stage a pinned compatible fix; validate container creation without restarting production runtime unexpectedly.

**C — Driver/kernel mismatch:** compare kernel/module/NVML package versions and upgrade timeline; select known-good pinned image/driver via approved rollback; verify NVML, CUDA and workloads.

**D — MIG device visibility:** map physical GPU, MIG profile, plugin strategy, pod resources and CUDA-visible devices; restore correct profile allocation under owner approval.

**E — Hardware-related CUDA failure:** preserve Xid/ECC/PCIe evidence, protect training checkpoints, escalate vendor, validate repaired node before uncordon.

## 13. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Compatibility and failure-domain inventory | 20 |
| Host/driver/NVML diagnostic accuracy | 20 |
| Container runtime and CUDA probe correctness | 20 |
| MIG and GPU scheduling diagnosis | 10 |
| Staging recovery and rollback safety | 20 |
| Runbooks, evidence and peer review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical safety, security, data-integrity or unsupported-driver-change failure. Lab execution and independent review remain pending.

## 14. References

- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [NVIDIA CUDA Runtime API](https://docs.nvidia.com/cuda/cuda-runtime-api/)
- [NVIDIA CUDA Installation Guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)

**Next:** Chapter 077 — GPU OOM and Memory Fragmentation.
