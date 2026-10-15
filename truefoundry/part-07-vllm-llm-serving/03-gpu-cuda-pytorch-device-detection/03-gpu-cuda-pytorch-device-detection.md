# 7.3 — GPU, CUDA, PyTorch and Device Detection

**Part:** 07 — vLLM & LLM Serving · **Chapter:** 3 of 7  
**Audience:** TrueFoundry platform engineers, Kubernetes/GPU SREs, model-serving engineers  
**Safety:** [SAFE-READ] for observations; [LAB-MUTATE] only in approved isolated nonproduction; [CHANGE-CONTROL] for production changes.

## 1. Objectives and prerequisites

Trace the complete GPU enablement chain from Kubernetes node to vLLM runtime; distinguish NVIDIA driver, CUDA runtime, PyTorch wheel and vLLM build compatibility; diagnose `No CUDA runtime`, `Failed to infer device type`, `torch.cuda.is_available() == False`, and missing `nvidia.com/gpu`; and validate a safe, authoritative rollback.

Prerequisites: Chapters 7.1–7.2; read-only Kubernetes access; an approved TrueFoundry workspace; a known vLLM image digest and model revision; and an existing authorized GPU workload. **Do not install drivers, restart device plugins or modify shared GPU nodes while following read-only exercises.**

## 2. The complete GPU execution chain

```text
Cloud VM / bare-metal GPU physically present
  → NVIDIA kernel driver loaded on host
  → NVIDIA Container Toolkit / compatible runtime configuration
  → Kubernetes NVIDIA device plugin (or supported GPU operator)
  → Node allocatable nvidia.com/gpu
  → Scheduler + taints/tolerations/affinity + quota
  → Pod requests and receives GPU device allocation
  → GPU device nodes and driver libraries visible in container
  → PyTorch CUDA-enabled build and compatible driver
  → vLLM accelerator/device inference and engine initialization
  → model weights and KV cache fit GPU VRAM
  → readiness and golden inference
```

**Lowest Proven Healthy Layer (LPHL)** is the last layer supported by evidence. **First Failed Transition (FFT)** is the next broken handoff. A successful `nvidia-smi` on a node does not prove GPU visibility in a Pod; `torch.cuda.is_available()` does not prove model architecture support or adequate VRAM.

## 3. Component responsibility matrix

| Component | Responsibility | Typical failure |
|---|---|---|
| GPU hardware/cloud instance | Exposes physical accelerator | Wrong node SKU, GPU absent |
| NVIDIA host driver | Kernel/device and userspace driver interface | Driver unloaded, device fault |
| Container runtime/toolkit | Makes device and driver libraries available to container | Missing libcuda, runtime config |
| Kubernetes device plugin | Advertises `nvidia.com/gpu` allocatable | No allocatable GPUs |
| Scheduler | Places GPU-requesting Pod | Pending due to quota, taint, affinity |
| Container image | Supplies framework/runtime userspace | CPU-only torch, incompatible libraries |
| PyTorch | Discovers usable CUDA backend | CUDA unavailable |
| vLLM | Detects device and launches engine | Device inference exception |
| Model executor | Allocates weights, workspaces and KV cache | GPU OOM / unsupported kernel |

TrueFoundry reconciles deployments into Kubernetes; verify the supported version-specific GPU resource settings in the platform before changing configuration.

## 4. Host driver vs CUDA toolkit vs PyTorch

- **NVIDIA driver** runs on the GPU host and provides the kernel interface and driver API. The host driver's supported CUDA API level is not the same as a container's installed toolkit version.
- **CUDA runtime / user-space libraries** are generally supplied with the serving image or its dependencies; their requirements must be compatible with the host driver.
- **PyTorch build** may be CUDA-enabled or CPU-only. `torch.version.cuda` reports the CUDA version against which that build was compiled; it is not the host driver version.
- **vLLM wheel/image** must support the Python, PyTorch, CUDA, GPU architecture and backend combination. Rebuilding or mixing wheels ad hoc can introduce ABI incompatibilities.
- **`nvidia-smi` CUDA Version** commonly indicates the maximum CUDA driver API version supported by the installed driver, **not** the installed toolkit or PyTorch build.

Always compare against the official NVIDIA/PyTorch/vLLM compatibility documentation for the exact release. Do not assume newer drivers or wheels are universally compatible.

## 5. Kubernetes discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <vllm-pod>
kubectl -n <namespace> get resourcequota,limitrange
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Inspect `Capacity` and `Allocatable` for `nvidia.com/gpu`; verify Pod GPU requests/limits, node selectors, tolerations, priority, quota and event messages. An ordinary Kubernetes GPU request is an integer resource; sharing/MIG requires explicit supported setup.

Inspect plugin placement only where labels/namespace are known and approved:

```bash
kubectl get daemonsets -A
kubectl get pods -A -o wide
```

Do not blindly restart a device-plugin DaemonSet: doing so may disrupt workloads. Device plugin namespaces and names vary by deployment.

## 6. Container-level diagnostics [SAFE-READ]

Execute **only in an existing approved container** with permission; avoid shells that dump environment variables or mounted credentials.

```bash
kubectl -n <namespace> exec <vllm-pod> -c <container> -- nvidia-smi
kubectl -n <namespace> exec <vllm-pod> -c <container> -- python -c 'import torch; print("torch:", torch.__version__); print("compiled CUDA:", torch.version.cuda); print("available:", torch.cuda.is_available()); print("count:", torch.cuda.device_count())'
kubectl -n <namespace> exec <vllm-pod> -c <container> -- python -c 'import vllm; print(vllm.__version__)'
```

Commands may fail because the container is not running or utilities are absent; record **Not available**, not “GPU broken.” `torch.cuda.device_count()` is only meaningful when the container has the expected GPU allocation. Avoid importing vLLM in a crashing container if it adds resource pressure; inspect startup logs first.

## 7. Diagnosing “No CUDA runtime”

Potential failure layers:

1. Pod was not allocated a GPU because `nvidia.com/gpu` was omitted.
2. Node has no allocatable GPU due to device-plugin/driver issue.
3. Container runtime did not expose device nodes or driver libraries.
4. Image contains CPU-only PyTorch or incompatible CUDA userspace libraries.
5. CUDA initialization fails due to unsupported GPU architecture, driver/runtime incompatibility or hardware errors.

**Order:** check Pod request → node allocatable → scheduling events → container GPU visibility → PyTorch CUDA build → driver/runtime compatibility → vLLM logs. Do not start by changing model weights or reinstalling drivers.

## 8. Diagnosing “Failed to infer device type”

This error can arise during vLLM platform/device selection, before model loading. Investigate the exact vLLM version and complete traceback. Check GPU allocation, container device exposure, PyTorch CUDA availability, backend support and architecture. It is not proof that Kubernetes scheduling failed; a Running Pod can lack GPU access.

Illustrative decision tree:

```text
Pod Pending?
  ├─ yes → GPU allocatable/quota/taints/affinity
  └─ no → Pod Running
           ├─ no device visible → GPU request/device plugin/runtime
           ├─ nvidia-smi works but torch CUDA false → torch build/ABI/runtime
           ├─ torch CUDA true but vLLM device fails → vLLM version/backend
           ├─ vLLM detects GPU but load fails → model support/VRAM/precision
           └─ model loads but no readiness → probes/warmup/endpoint
```

A supported CPU-only vLLM configuration, if available for the installed release, is **not** a drop-in replacement for GPU serving; do not silently change execution target.

## 9. GPU memory and allocation

Differentiate host RAM, GPU framebuffer/VRAM, CUDA allocator reservations, model weights, temporary activations and KV cache. A simplified KV-cache estimate for a decoder-only transformer:

```text
KV bytes ≈ 2 × layers × KV heads × head_dim × live_tokens × bytes_per_element
```

Actual memory also depends on tensor parallelism, block granularity, quantization, sliding-window attention, allocator overhead and model architecture. A model that fits at startup may OOM at high concurrency or long contexts.

Investigate `nvidia-smi` memory, approved DCGM metrics, vLLM running/waiting requests, KV usage, model context limit and token distributions. Do not assume low GPU utilization means ample VRAM.

## 10. Safe runtime version matrix

Record the following as an **evidence worksheet**, not as invented compatible versions:

| Layer | Observed version / value | Source |
|---|---|---|
| GPU SKU and compute capability | | Approved inventory |
| Host NVIDIA driver | | Node GPU telemetry |
| Driver-supported CUDA API | | `nvidia-smi` |
| Container CUDA userspace | | Image build manifest |
| Python and PyTorch build | | Image/runtime metadata |
| `torch.version.cuda` | | Container diagnostic |
| vLLM version | | Image/runtime metadata |
| Model architecture and revision | | Registry/manifest |
| NVIDIA device plugin/operator | | Cluster inventory |
| Runtime class/toolkit | | Cluster configuration |

Compatibility is a **tested combination**, not a comparison of one version string.

## 11. Metrics and alerts

Observe GPU availability, VRAM occupancy, device errors, Pod Pending duration, CUDA initialization failures, model load time, readiness, 5xx, TTFT/ITL and queue depth. Alert on user impact first (zero ready endpoints, sustained error ratio, latency SLO breach), then attach GPU diagnostics. GPU utilization alone is not a sufficient page trigger.

## 12. Troubleshooting matrix

| ID | Symptom | Evidence | Likely first failed transition |
|---|---|---|---|
| G1 | GPU node absent | Node inventory | Provisioning → cluster |
| G2 | No `nvidia.com/gpu` allocatable | Node describe | Driver/plugin → Kubernetes |
| G3 | GPU Pod Pending | Scheduler events | Request → placement |
| G4 | GPU quota exceeded | Namespace quota | Admission → scheduling |
| G5 | Missing toleration | Taints and Pod spec | Scheduler → node |
| G6 | Device invisible inside Pod | GPU request and runtime | Plugin → container |
| G7 | `nvidia-smi` missing | Image tooling | Utility → diagnostics (not GPU proof) |
| G8 | `nvidia-smi` fails | Device/driver errors | Driver → container |
| G9 | `torch.version.cuda` null | PyTorch build | Image → CUDA-enabled framework |
| G10 | `torch.cuda.is_available()` false | Build/device | Framework → accelerator |
| G11 | `No CUDA runtime` | Traceback | Runtime → CUDA |
| G12 | `Failed to infer device type` | vLLM traceback | Engine → platform |
| G13 | Unsupported GPU architecture | Compatibility docs | Kernel → device |
| G14 | Model OOM at load | VRAM and model metadata | Weights → GPU memory |
| G15 | KV OOM under load | KV/queue/concurrency | Active tokens → VRAM |
| G16 | Device health error | DCGM/ECC/Xid if available | Hardware/driver → executor |
| G17 | Multi-GPU worker failure | Worker/NCCL logs | Parallel executor → interconnect |
| G18 | GPU ready, endpoint 503 | Pod probes/EndpointSlice | Engine → Service |

Do not interpret a symptom as a root cause without corroborating evidence.

## 13. Incident response and rollback

1. Record affected endpoint, image digest, model revision, GPU node, vLLM/PyTorch versions, onset and SLO impact.
2. Establish LPHL/FFT from Kubernetes events, driver/device telemetry and sanitized logs.
3. Freeze promotion; preserve healthy replicas and bound retries.
4. Fix the confirmed layer in an approved change. Node/driver/device-plugin upgrades require cluster-owner coordination and capacity/drain plans.
5. Roll back **image + vLLM/PyTorch/CUDA stack + model/tokenizer revision + GPU scheduling/resources** via authoritative TrueFoundry/GitOps configuration.
6. Verify GPU allocation, CUDA detection, model load, Ready EndpointSlice, authorized golden request, TTFT/ITL, queue and VRAM.
7. Document RCA, compatibility matrix and preventive validation.

## 14. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | Correct context/workspace and safe access |
| A2 | GPU hardware, driver and plugin inventoried |
| A3 | Node allocatable and Pod GPU allocation validated |
| A4 | Container device and PyTorch CUDA state recorded |
| A5 | vLLM device inference and version compatibility assessed |
| A6 | Model memory/KV budget and startup readiness reviewed |
| A7 | 18 failure scenarios classified by LPHL/FFT |
| A8 | Known-good rollback and post-recovery checks documented |

## 15. Evidence template

```text
Date/time, owner and approval:
TrueFoundry workspace / cluster / namespace:
GPU node/SKU / nvidia.com/gpu allocatable:
Pod GPU request, selector, tolerations and quota:
Driver / device plugin / container runtime:
PyTorch build / torch.version.cuda / torch.cuda.is_available:
vLLM version and complete sanitized error:
Model/tokenizer revision and image digest:
Host RAM / VRAM / KV cache / concurrency:
Lowest Proven Healthy Layer:
First Failed Transition:
Remediation owner / rollback release:
Golden inference, readiness and post-recovery SLO:
```

**Hands-on lab:** [7.3 — GPU CUDA PyTorch Device Detection Check](../labs/03-gpu-cuda-pytorch-device-detection-check.md)  
**Next:** Chapter 7.4 — Model Loading Tokenizers and Tensor Parallelism.

## References

- [vLLM Documentation](https://docs.vllm.ai/)
- [PyTorch CUDA Semantics](https://docs.pytorch.org/docs/stable/notes/cuda.html)
- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
