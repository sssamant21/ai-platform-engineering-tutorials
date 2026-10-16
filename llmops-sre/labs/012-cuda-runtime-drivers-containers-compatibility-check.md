# Chapter 012 — Hands-On Lab: CUDA Runtime, Drivers, Containers and Compatibility

**Part 03 | Chapter 012 | Published lab plan; execution pending**

## Goal

Build a GPU software compatibility inventory and troubleshoot synthetic driver, CUDA runtime, container toolkit and device-plugin incidents without modifying production.

## Safety

**[SAFE-READ]** approved Kubernetes, node and release metadata. **[LAB-MUTATE]** explicitly approved staging CUDA initialization and synthetic model validation. **[CHANGE-CONTROL]** production driver, kernel, toolkit, GPU operator, device plugin or node pool changes. Do not install software, enable privileged Pods or patch DaemonSets as part of read-only labs.

## Lab 0 — Preflight

Record UTC, approved context/namespace, GPU SKU, host OS/kernel, NVIDIA driver, framework/image digest, CUDA runtime, device plugin, NVIDIA toolkit, model revision, owner, reviewer and cost ceiling.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

Stop if permissions or context are unverified.

## Lab 1 — Driver and device evidence [SAFE-READ]

On an authorized GPU host or approved troubleshooting Pod:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total --format=csv
uname -r
```

Optionally, if installed:

```bash
nvcc --version
```

Document why `nvidia-smi` CUDA version is not the container toolkit version, and why missing `nvcc` may be normal.

## Lab 2 — Kubernetes GPU scheduling [SAFE-READ]

```bash
kubectl describe nodes
kubectl get pods -A -o wide
```

Record node GPU capacity and allocatable resources, pending Pod events, device-plugin health and scheduling constraints. Do not change taints or requests.

## Lab 3 — Compatibility worksheet [DESIGN]

| Component | Recorded version/source | Supported by matrix? |
|---|---|---|
| GPU model/compute capability | Pending | Pending |
| Kernel and NVIDIA driver | Pending | Pending |
| CUDA driver interface | Pending | Pending |
| Container CUDA runtime | Pending | Pending |
| cuDNN/cuBLAS/NCCL | Pending | Pending |
| Framework/runtime image | Pending | Pending |
| NVIDIA Container Toolkit | Pending | Pending |
| Kubernetes device plugin | Pending | Pending |
| Model/serving image digest | Pending | Pending |

Use actual vendor/framework compatibility documentation for the deployed releases. Do not guess.

## Lab 4 — Container validation [DESIGN; optional approved staging]

Create a test plan for GPU visibility, framework import, CUDA initialization, minimal synthetic inference, complete SSE, VRAM, TTFT/TPOT and error classification. Do not execute on production. Define hard abort thresholds, owner and rollback.

## Lab 5 — Incident triage [TABLETOP]

For each scenario, write symptom → safe checks → probable layer → escalation → authorized mitigation → rollback → customer-path verification:
- `nvidia-smi` works but CUDA initialization fails.
- Container cannot find `libcuda.so`.
- New image reports insufficient driver.
- Pod is Pending for GPU.
- NCCL times out after multi-GPU rollout.
- GPU reports XID/ECC error.

## Lab 6 — Eight game days

01 missing host driver; 02 newer CUDA image; 03 device plugin unavailable; 04 container missing GPU; 05 unsupported kernel; 06 NCCL failure; 07 XID after upgrade; 08 GitOps incompatible rollback. Default **tabletop**.

## Twenty-scenario checklist

F01 module; F02 kernel; F03 driver version; F04 devices; F05 libcuda; F06 libcudart; F07 cuDNN; F08 NCCL; F09 compute capability; F10 ordinal; F11 plugin crash; F12 allocatable; F13 pending; F14 MIG; F15 CUDA init; F16 OOM; F17 illegal access; F18 collective; F19 XID/ECC; F20 image regression.

## Sixteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | CUDA architecture map | Pending |
| G02 | Driver/runtime distinction | Pending |
| G03 | nvidia-smi interpretation | Pending |
| G04 | Vendor compatibility matrix | Pending |
| G05 | Library/framework inventory | Pending |
| G06 | Immutable image digest | Pending |
| G07 | Device plugin and scheduling | Pending |
| G08 | Toolkit integration | Pending |
| G09 | MIG considerations | Pending |
| G10 | Authorized diagnostics | Pending |
| G11 | CUDA init validation plan | Pending |
| G12 | NCCL/topology | Pending |
| G13 | Security controls | Pending |
| G14 | Upgrade/rollback plan | Pending |
| G15 | Customer-path acceptance | Pending |
| G16 | Reviewer signoff | Pending |

## Evidence template

```text
UTC/context/authorization:
GPU SKU/UUID/kernel/driver:
Container image digest/CUDA runtime:
Framework/cuDNN/cuBLAS/NCCL:
Device plugin/toolkit/MIG:
Vendor compatibility references:
Read-only diagnostics and findings:
Staging execution vs tabletop:
Risk, cost, rollback, reviewer and gate status:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop separately approved staging validation, reconcile GPU-hours and remove only resources owned by the change. All gates remain pending until independently reviewed.

**Next: Chapter 013 — GPU VRAM Sizing, Precision and Quantization.**
