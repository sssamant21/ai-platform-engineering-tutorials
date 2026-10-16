# Chapter 012 — CUDA Runtime, Drivers, Containers and Compatibility

**Part 03 — GPU Architecture, CUDA and VRAM | Chapter 012 of 100**

> **Status:** Written tutorial published. Technical/security review, real lab execution and operational acceptance remain pending. Version examples are illustrative; use NVIDIA and framework compatibility documentation for your exact stack.

## Learning objectives

Explain the NVIDIA kernel driver, CUDA driver API, CUDA runtime, user-space libraries, container toolkit and Kubernetes device plugin; validate compatibility without changing production; diagnose CUDA initialization, driver/library mismatch and GPU scheduling failures; build an immutable GPU serving release contract; and plan safe driver upgrades with rollback.

## 1. CUDA and container architecture

```text
Kubernetes scheduler / node allocatable nvidia.com/gpu
  → NVIDIA device plugin / GPU operator (if installed)
  → kubelet + container runtime + NVIDIA Container Toolkit
  → inference container
      ├─ serving framework (vLLM / Triton / PyTorch)
      ├─ CUDA runtime and CUDA-enabled libraries
      ├─ user-space CUDA driver interface (libcuda.so from host)
      └─ application kernels
  → host NVIDIA kernel driver
  → /dev/nvidia* and physical GPU
```

The kernel-mode NVIDIA driver runs on the host. CUDA user-space runtime and framework libraries may ship in the container. The NVIDIA Container Toolkit arranges device and driver-library access. A container cannot compensate for an incompatible or missing host driver simply by bundling a newer CUDA toolkit.

The CUDA **driver API** and **runtime API** are distinct; applications can use either. Driver compatibility and CUDA toolkit minor-version compatibility are conditional on release, feature set and platform. Never infer compatibility from a single version string.

## 2. Understand the version chain

| Layer | Example evidence | Compatibility risk |
|---|---|---|
| GPU hardware | model, architecture, compute capability | unsupported kernels |
| Host kernel and driver | `uname -r`, `nvidia-smi` | missing module, unsupported driver |
| Driver interface | `libcuda.so` availability | runtime initialization |
| CUDA runtime/toolkit | `nvcc --version` if installed | library ABI / kernel compatibility |
| cuDNN / NCCL / cuBLAS | package/runtime manifest | mismatched libraries |
| Framework | PyTorch, vLLM, Triton build | compiled CUDA target |
| Container runtime/toolkit | containerd, NVIDIA toolkit | missing devices/libraries |
| Kubernetes device plugin | DaemonSet/allocatable GPU | scheduling and discovery |
| Serving image | immutable digest, SBOM | drift, supply chain |
| MIG mode (where supported) | partition inventory | resource and memory mismatch |

**Important:** The CUDA version shown by `nvidia-smi` is the *maximum CUDA version supported by the installed driver interface*, not proof that a matching CUDA toolkit is installed inside the container. `nvcc` reports the compiler toolkit when present; many production inference images omit it.

## 3. Driver, runtime and library compatibility

NVIDIA publishes compatibility tables and minimum-driver requirements for CUDA toolkit releases. Some newer runtime features or PTX JIT paths require newer drivers even when limited minor-version compatibility applies. Check framework release notes, GPU architecture support, driver branch, host OS/kernel and CUDA libraries together.

A reproducible release manifest should pin:

```yaml
model_revision: "APPROVED_MODEL_REVISION"
serving_image_digest: "sha256:APPROVED_DIGEST"
gpu_sku: "APPROVED_GPU_SKU"
nvidia_driver_branch: "APPROVED_BRANCH"
cuda_runtime_version: "APPROVED_VERSION"
framework_version: "APPROVED_VERSION"
nvidia_container_toolkit_version: "APPROVED_VERSION"
device_plugin_version: "APPROVED_VERSION"
kubernetes_version: "APPROVED_VERSION"
validation_evidence: "CHANGE_RECORD_ID"
```

These are example documentation fields, **not** a Kubernetes resource or deployment manifest.

## 4. GPU discovery and Kubernetes scheduling

Kubernetes generally schedules NVIDIA GPUs through extended resources advertised by a device plugin, commonly `nvidia.com/gpu`. A healthy Pod schedule is not proof that CUDA can initialize. Conversely, a Pod pending with insufficient GPU capacity is a scheduling issue, not necessarily a driver crash.

Check:
- GPU node labels, taints, affinity and topology.
- Device plugin health and `allocatable` versus `capacity`.
- RuntimeClass configuration only if used by the cluster.
- NVIDIA toolkit and container runtime integration.
- Device files, library mounts and process visibility.
- MIG or time-slicing resource naming if configured.
- Security contexts and least-privilege access.

Never install the NVIDIA operator, edit device-plugin DaemonSets or switch runtime configuration during a read-only investigation.

## 5. Read-only compatibility checks

**[SAFE-READ]** only in an approved context/node:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

On an authorized GPU host or approved troubleshooting container:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total --format=csv
uname -r
```

If installed and authorized:

```bash
nvcc --version
```

A missing `nvcc` alone is **not** a failure. To inspect a known inference deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Do not collect prompts, tokens, secrets or credentials from logs. Use verified namespace names rather than copying placeholders verbatim.

## 6. Container and framework validation

Before promotion, use an approved staging workload and synthetic inference:
1. Verify image digest, SBOM, provenance and vulnerability policy.
2. Verify expected GPU devices and runtime libraries are exposed.
3. Check framework import and CUDA initialization without customer traffic.
4. Confirm the expected GPU architecture is supported by compiled kernels.
5. Load the approved model revision under a bounded memory budget.
6. Validate first-token, complete-stream, throughput and error handling.
7. Inspect CUDA OOM, illegal memory access, NCCL and kernel launch errors.
8. Compare TTFT/TPOT and GPU memory to the previous known-good revision.
9. Record rollback artifact and GitOps reconciliation owner.

Never interpret a successful `nvidia-smi` call as complete serving validation.

## 7. Failure decision tree

```text
GPU inference fails?
 ├─ Pod Pending → allocatable GPU, affinity, taints, quota
 ├─ GPU not visible → device plugin, toolkit, runtimeClass, devices
 ├─ nvidia-smi fails on host → kernel driver/module/hardware
 ├─ nvidia-smi works, CUDA init fails → libcuda/runtime/framework
 ├─ "driver version insufficient" → CUDA minimum-driver matrix
 ├─ missing libcudart/cuDNN/NCCL → image/library dependency
 ├─ "no kernel image available" → GPU compute capability/build
 ├─ invalid device ordinal → CUDA_VISIBLE_DEVICES/resource assignment
 ├─ NCCL init timeout → topology/network/collectives
 ├─ CUDA OOM → weights/KV/workspace/concurrency
 ├─ XID/ECC → GPU hardware/driver escalation
 ├─ only new image fails → image/framework/runtime regression
 └─ metrics absent → exporter/device plugin/scrape freshness
```

## 8. Driver upgrade and rollback plan

**[CHANGE-CONTROL]** Driver changes affect every GPU workload on a node. Prefer controlled node-pool replacement or approved maintenance procedures rather than modifying a busy production node.

- Inventory GPU SKUs, driver/kernel versions, workloads, topology and disruption budgets.
- Verify cloud/Kubernetes/GPU operator support matrix and kernel module signing requirements.
- Build and validate a staging canary with pinned serving image and synthetic requests.
- Establish drain, workload migration, rollback node pool, spare capacity and outage communications.
- Observe device plugin readiness, CUDA initialization, complete-stream success, TTFT/TPOT and XID/ECC.
- Abort on incompatible images, failed model loads, missing GPU resources or SLO regression.
- Roll back through the authoritative infrastructure/GitOps controller; avoid manual drift.

## 9. Security and FinOps controls

Use signed/scanned images, minimal capabilities, controlled GPU device access, approved registries, immutable digests, kernel patch policy and workload identity. Do not run privileged debug containers without authorization. Track node-pool idle GPU-hours, driver-induced restarts, failed model loads, wasted retries and extra canary capacity. Driver upgrades have availability and GPU cost implications.

## 10. Twenty production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | driver module absent | host `nvidia-smi` |
| F02 | driver/kernel mismatch | kernel/module logs |
| F03 | CUDA minimum driver unmet | compatibility matrix |
| F04 | container missing GPU device | runtime/toolkit config |
| F05 | `libcuda.so` unavailable | mounted driver libraries |
| F06 | `libcudart` mismatch | image dependencies |
| F07 | cuDNN ABI mismatch | framework import logs |
| F08 | NCCL version mismatch | distributed startup |
| F09 | unsupported GPU compute capability | kernel build |
| F10 | invalid device ordinal | device visibility |
| F11 | device plugin crash | DaemonSet/events |
| F12 | node GPU allocatable missing | node description |
| F13 | Pod pending GPU | scheduler events |
| F14 | MIG resource mismatch | partition/resource inventory |
| F15 | framework CUDA initialization failure | application traceback |
| F16 | CUDA OOM after startup | model/KV/VRAM |
| F17 | illegal memory access | kernel/runtime logs |
| F18 | NCCL collective timeout | network/topology |
| F19 | driver XID/ECC error | host/GPU health |
| F20 | image upgrade breaks inference | canary vs baseline |

## 11. Fourteen operational runbooks

1. Verify cluster context and GPU host inventory.
2. Investigate missing NVIDIA driver/module.
3. Validate CUDA/driver minimum-version matrix.
4. Triage container toolkit and GPU visibility.
5. Investigate missing CUDA user-space libraries.
6. Triage framework and compiled-kernel compatibility.
7. Diagnose Kubernetes GPU scheduling and device plugin.
8. Investigate MIG/resource naming mismatch.
9. Diagnose CUDA initialization and device ordinal.
10. Investigate NCCL/topology/distributed startup.
11. Triage CUDA OOM and illegal memory access.
12. Escalate XID/ECC hardware events.
13. Execute staged driver/image compatibility validation.
14. Roll back driver/node/image change and verify customer path.

Every runbook needs owner, impact, read-only evidence, escalation, change approval, rollback and synthetic completion verification.

## 12. Eight game days

1. Host driver absent after node replacement.
2. New image requires a newer driver.
3. Device plugin advertises no GPU.
4. Container has no visible devices.
5. Unsupported kernel for older GPU SKU.
6. NCCL initialization fails across GPUs.
7. Driver upgrade triggers XID errors.
8. GitOps restores an incompatible serving image.

Default **tabletop**; destructive tests require separately approved staging.

## 13. Sixteen acceptance gates

G01 architecture map; G02 driver vs runtime distinction; G03 `nvidia-smi` interpretation; G04 compatibility matrix; G05 framework/library inventory; G06 immutable image digest; G07 device plugin and scheduling; G08 runtime/toolkit integration; G09 MIG considerations; G10 read-only diagnostics; G11 CUDA initialization; G12 NCCL/topology; G13 security controls; G14 staged upgrade/rollback; G15 customer-path validation; G16 independent review.

See [companion lab](../labs/012-cuda-runtime-drivers-containers-compatibility-check.md). All gates remain pending until evidenced and reviewed.

**Next: Chapter 013 — GPU VRAM Sizing, Precision and Quantization.**
