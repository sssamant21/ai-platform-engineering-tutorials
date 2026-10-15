# Lab 7.3 — GPU CUDA PyTorch and Device Detection Check

**Canonical:** [Chapter 7.3](../03-gpu-cuda-pytorch-device-detection/03-gpu-cuda-pytorch-device-detection.md)  
**Default:** [SAFE-READ]. Changes to runtime, device plugin, GPU nodes or workloads require approval; fault injection is [LAB-MUTATE] only on isolated nonproduction resources.

## 1. Goal

Prove or isolate the GPU device chain from node driver through Kubernetes allocation, container CUDA, PyTorch and vLLM. Diagnose `No CUDA runtime` and `Failed to infer device type` without unsafe host changes.

## 2. Preflight

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> get resourcequota,limitrange
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record approved TrueFoundry workspace, namespace, GPU workload, owner, image digest and model revision. No production mutation, secrets or sensitive prompts.

## 3. Node and scheduler inventory

```bash
kubectl describe node <gpu-node>
kubectl -n <namespace> describe pod <vllm-pod>
kubectl get daemonsets -A
```

| Layer | Observed evidence | Pass/Fail/Unknown |
|---|---|---|
| GPU node SKU | | |
| NVIDIA driver | | |
| Device plugin/operator | | |
| Node GPU allocatable | | |
| Pod GPU request/limit | | |
| Namespace GPU quota | | |
| Taints/tolerations/affinity | | |
| Pod scheduled and Running | | |

Record events verbatim with secrets redacted. Do not restart or modify GPU DaemonSets.

## 4. Container runtime checks

Run only when an **existing authorized container** is available:

```bash
kubectl -n <namespace> exec <vllm-pod> -c <container> -- nvidia-smi
kubectl -n <namespace> exec <vllm-pod> -c <container> -- python -c 'import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available(), torch.cuda.device_count())'
kubectl -n <namespace> exec <vllm-pod> -c <container> -- python -c 'import vllm; print(vllm.__version__)'
```

If the Pod is crashing, use sanitized logs and image build metadata instead. Missing diagnostic utilities are **Not installed**, not automatic GPU failure.

## 5. Compatibility matrix

| Component | Value | Verified against |
|---|---|---|
| GPU model/architecture | | |
| Driver version | | |
| Driver-supported CUDA API | | |
| Container CUDA userspace | | |
| PyTorch build and CUDA | | |
| vLLM version | | |
| Model/tokenizer revision | | |
| GPU resource allocation | | |
| VRAM headroom | | |

Explain why `nvidia-smi` CUDA Version differs from `torch.version.cuda` when applicable.

## 6. Fault analysis: two critical errors

### A. “No CUDA runtime”

Collect Pod request, node allocatable, container device visibility, PyTorch build, driver compatibility and full sanitized traceback. Identify LPHL/FFT. Never jump directly to reinstalling drivers.

### B. “Failed to infer device type”

Collect vLLM version, platform/backend support, PyTorch CUDA availability, GPU allocation and complete sanitized traceback. Identify whether failure precedes model loading.

## 7. Optional isolated runtime experiment [LAB-MUTATE]

In a disposable approved workspace, compare a verified CUDA-capable vLLM image with an intentionally incompatible **lab-only** image or GPU allocation configuration. Do not run disruptive driver/plugin experiments on shared nodes. Capture startup logs, readiness and recovery after restoring known-good release.

## 8. Failure classification

| ID | Scenario | LPHL | FFT | Evidence | Safe response |
|---|---|---|---|---|---|
| G1 | GPU node absent | | | | |
| G2 | No GPU allocatable | | | | |
| G3 | Pod Pending | | | | |
| G4 | GPU quota | | | | |
| G5 | Missing toleration | | | | |
| G6 | Device invisible | | | | |
| G7 | Missing nvidia-smi utility | | | | |
| G8 | nvidia-smi failure | | | | |
| G9 | CPU-only PyTorch | | | | |
| G10 | torch CUDA false | | | | |
| G11 | No CUDA runtime | | | | |
| G12 | Device inference fails | | | | |
| G13 | Unsupported GPU architecture | | | | |
| G14 | Load-time VRAM OOM | | | | |
| G15 | KV-cache OOM | | | | |
| G16 | GPU device error | | | | |
| G17 | Multi-GPU worker failure | | | | |
| G18 | Endpoint 503 after GPU ready | | | | |

Use tabletop where live injection is unsafe. Do not claim executed evidence for tabletop cases.

## 9. Acceptance gates

| Gate | Pass/Fail/Not run | Evidence |
|---|---|---|
| A1 Approved context/access | | |
| A2 Driver/plugin inventory | | |
| A3 Node/Pod GPU allocation | | |
| A4 Container/PyTorch CUDA state | | |
| A5 vLLM inference/compatibility | | |
| A6 VRAM/KV/startup | | |
| A7 LPHL/FFT classification | | |
| A8 Rollback/recovery | | |

## 10. Evidence and cleanup

```text
Date/time, owner, workspace:
Node GPU SKU and driver:
Device plugin and allocatable:
Pod requests, quota and scheduling:
Container device visibility:
PyTorch and vLLM version/build:
CUDA error and sanitized traceback:
Model/image revision, VRAM/KV:
G1–G18 classification:
Lowest Proven Healthy Layer:
First Failed Transition:
Known-good release and rollback:
Post-recovery GPU/model/endpoint checks:
Unexecuted gates and cleanup:
```

**Next:** Chapter 7.4 — Model Loading Tokenizers and Tensor Parallelism.
