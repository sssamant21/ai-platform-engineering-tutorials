# 4.4 — CUDA, Container Runtime and Compatibility

**Part:** 04 — GPU Infrastructure  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, model-serving engineers  
**Safety:** Production inspection is **[SAFE-READ]**. Image changes, runtime configuration and driver upgrades require **[CHANGE-CONTROL]**. Optional lab mutation is confined to an approved disposable environment.

## 1. Learning objectives

Explain the difference between the host NVIDIA driver, CUDA driver API, CUDA runtime/toolkit, NVIDIA Container Toolkit, PyTorch CUDA build and vLLM device detection. Diagnose GPU initialization failures layer by layer without treating all CUDA errors as driver failures.

## 2. End-to-end GPU initialization path

```text
TrueFoundry workload configuration
    ↓
Rendered Kubernetes Pod + nvidia.com/gpu allocation
    ↓
Kubelet / container runtime (containerd, CRI-O where supported)
    ↓
NVIDIA Container Toolkit / GPU device injection
    ↓
Container: GPU device nodes + driver libraries
    ↓
PyTorch CUDA-enabled build + CUDA runtime dependencies
    ↓
CUDA driver API → host NVIDIA kernel driver
    ↓
Physical GPU
    ↓
vLLM model initialization and inference
```

**Do not collapse these into one “CUDA installation” layer.** The host driver, container CUDA runtime and framework wheel/image may be versioned independently. TrueFoundry reconciliation, Pod scheduling and model readiness are separate success criteria.

## 3. CUDA terminology and version interpretation

| Term | What it means | How to inspect | Common mistake |
|---|---|---|---|
| Host NVIDIA driver | Enables host communication with GPU | `nvidia-smi`, driver version | Assuming it is the container toolkit |
| CUDA driver API | Driver-facing GPU programming interface | Driver/library diagnostics | Treating driver API as a separately installed toolkit |
| CUDA runtime | Runtime used by applications/libraries | Container libraries, framework build | Assuming host toolkit must match exactly |
| CUDA toolkit / `nvcc` | Compiler and developer tools, optional in runtime images | `nvcc --version` if installed | Treating missing `nvcc` as proof CUDA cannot run |
| `nvidia-smi` “CUDA Version” | Maximum CUDA version supported by the driver (as reported) | `nvidia-smi` | Treating it as installed CUDA toolkit version |
| `torch.version.cuda` | CUDA version used to build the PyTorch package | Python import | Treating it as proof the GPU is accessible |
| `torch.cuda.is_available()` | PyTorch CUDA availability at runtime | Python check | Assuming it proves vLLM model will load |
| GPU compute capability | Hardware architecture compatibility target | GPU model/vendor specs | Assuming all wheels/kernels support all GPUs |

Driver/toolkit compatibility can include minor-version compatibility and other constraints; use official NVIDIA documentation for the **exact** driver branch, CUDA release, GPU architecture and runtime. PyTorch/vLLM compatibility is also release-specific. Never invent a universal version matrix.

## 4. Kubernetes resource and container prerequisites

The workload must receive an advertised and available GPU extended resource:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

The Pod must be placed on an eligible node and the runtime must expose the GPU devices and driver libraries. A GPU-equipped node with a healthy host driver does not guarantee container access.

An environment-specific placement example:

```yaml
nodeSelector:
  node-pool: gpu-reserved
  gpu: "true"
  gpu_type: A10G
```

These labels are illustrative, not Kubernetes/TrueFoundry defaults. If the pool is tainted, an approved matching toleration may also be required.

## 5. Production-safe investigation [SAFE-READ]

### 5.1 Establish workload identity and scheduler state

```bash
kubectl config current-context
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl describe node <gpu-node>
```

Capture image digest, Pod owner, node, GPU request/limit, scheduling conditions, container state and relevant events. A Pending Pod requires scheduling investigation first; a scheduled Pod with CUDA errors requires later-layer investigation.

### 5.2 Compare host and container GPU visibility

On an **authorized GPU node shell**:

```bash
nvidia-smi
nvidia-smi --query-gpu=name,driver_version,memory.total,memory.used --format=csv
```

On an **approved existing GPU-enabled diagnostic Pod**:

```bash
kubectl -n <namespace> exec <pod> -c <container> -- nvidia-smi
```

If `nvidia-smi` is not present in the image, that alone does not prove GPU allocation failed. Check the container's intended toolset and runtime device exposure via authorized diagnostics.

### 5.3 Inspect framework CUDA availability

Only if PyTorch is already installed in an approved diagnostic container:

```bash
python -c "import torch; print('torch',torch.__version__); print('build_cuda',torch.version.cuda); print('available',torch.cuda.is_available()); print('devices',torch.cuda.device_count())"
```

If CUDA is available, a more direct allocation smoke test in a disposable environment is:

```bash
python -c "import torch; x=torch.ones(4,device='cuda'); print(x.sum().item())"
```

**Interpretation:**
- `torch.version.cuda is None`: often a CPU-only PyTorch build.
- `torch.version.cuda` populated but `is_available() == False`: framework was built with CUDA support but cannot currently initialize an available GPU; investigate driver/runtime/device visibility and compatibility.
- CUDA smoke test passes but vLLM fails: investigate vLLM wheel/platform support, configuration, model dependencies, memory and version-specific startup logs.

Do not run package installations, driver updates or environment mutation in production to “test” a hypothesis.

## 6. Runtime integration and image provenance

Container runtime integration depends on the installed NVIDIA Container Toolkit, CRI/runtime configuration and GPU allocation strategy. Managed GPU node images and GPU Operator deployments may configure this differently. Do **not** blindly change the node's default runtime handler or edit `containerd` configuration on shared nodes.

For the image, capture:
- Immutable image digest and registry provenance
- Base OS and architecture
- Python version, PyTorch build and CUDA dependency versions
- vLLM version and supported GPU architectures
- Model precision, quantization and required memory
- Runtime environment settings that alter GPU visibility (for example `CUDA_VISIBLE_DEVICES`) without exposing secrets

**Security:** prefer signed, organization-approved images; do not copy credentials, secrets or complete environment dumps into incident tickets.

## 7. vLLM: “No CUDA runtime is found” / “Failed to infer device type”

These messages may arise from several layers; they are **not** proof of a single root cause.

### Decision tree

```text
vLLM cannot infer CUDA device
  ├─ Pod unscheduled?
  │    └─ Inspect nvidia.com/gpu, node eligibility and events
  └─ Pod scheduled
       ├─ Host nvidia-smi fails?
       │    └─ Driver / node-image / hardware owner
       ├─ GPU not advertised or allocated?
       │    └─ Device plugin / kubelet / Pod GPU limit
       ├─ Container cannot access GPU?
       │    └─ NVIDIA toolkit / runtime / allocation
       ├─ PyTorch CUDA unavailable?
       │    └─ CPU-only wheel, driver/runtime mismatch, unsupported GPU
       ├─ PyTorch CUDA works but vLLM fails?
       │    └─ vLLM version/platform/kernel/configuration
       └─ vLLM initializes but model fails?
            └─ VRAM, model format, dtype, quantization or model loading
```

### Evidence-led error matrix

| Symptom | Evidence to collect | Do not assume |
|---|---|---|
| `nvidia-smi: command not found` in container | Image contents and other GPU checks | Host driver is missing |
| `Failed to initialize NVML` | Host/container driver library and device access | Framework wheel alone is broken |
| `torch.version.cuda == None` | PyTorch package/build metadata | Hardware has failed |
| `torch.cuda.is_available() == False` | Runtime logs, driver, devices, package build | Driver upgrade always fixes it |
| `CUDA driver version is insufficient` | Actual driver/runtime versions and vendor matrix | Arbitrary toolkit upgrade is safe |
| `invalid device ordinal` | Visible device indices, allocation and application configuration | More physical GPUs are required |
| `CUDA out of memory` | VRAM allocation, model size, concurrency, KV cache | Kubernetes node RAM is exhausted |
| CUDA works, vLLM device inference fails | vLLM/PyTorch compatibility and detailed logs | GPU device is absent |

## 8. TrueFoundry deployment investigation

1. Identify workspace, deployment, namespace, rendered controller and immutable image digest.
2. Confirm Pod GPU resource allocation and node eligibility.
3. Check node driver/device-plugin/runtime health and container GPU visibility.
4. Check PyTorch CUDA build and runtime availability.
5. Review vLLM startup logs for the first failure, not only the last stack trace.
6. Assign the **Lowest Proven Healthy Layer**, **First Failed Transition** and current actionable owner.
7. Fix through authoritative TrueFoundry/GitOps/image build configuration after compatibility review.
8. Validate a successful model readiness check and inference request, not merely `Running` Pod state.

## 9. Cloud provider differences

- **EKS:** Review GPU instance, node AMI/driver strategy, device plugin, NVIDIA toolkit and runtime configuration. Bottlerocket and Linux GPU AMIs differ.
- **AKS:** Review GPU VM SKU, supported node image/driver mode and GPU operator/toolkit strategy.
- **GKE:** Review GPU accelerator node pool, driver installation mode and GPU runtime integration for the chosen GKE mode/version.

Do not copy driver-installation commands across providers without checking their supported node-image and operator ownership models.

## 10. Change, rollback and recovery [CHANGE-CONTROL]

**Before change:** preserve versions, Pod spec, image digest, node image, operator values, inference baseline and rollback artifact. Verify vendor compatibility matrix and identify canary node/pool.

**Change options:** rebuild the model image with a compatible CUDA-enabled PyTorch/vLLM stack; adjust reviewed GPU allocation; or repair node runtime/driver through its authoritative management plane. Choose the smallest supported blast radius.

**Rollback:** redeploy the last known-good immutable image or revert the approved node-image/operator/runtime configuration. Driver/runtime rollback may require node replacement or downtime. Do not restart shared container runtimes as a diagnostic shortcut.

**Recovery checks:** node Ready → GPU advertised → Pod scheduled → container GPU visible → PyTorch CUDA test passes → vLLM model Ready → inference succeeds → error/latency metrics within baseline.

## 11. Operational acceptance

- [ ] Distinguish driver-supported CUDA version, toolkit version and PyTorch CUDA build.
- [ ] Explain why `nvcc` can be absent in a working runtime container.
- [ ] Trace GPU exposure from Pod request through runtime to framework.
- [ ] Diagnose vLLM device inference errors with evidence, not assumptions.
- [ ] Capture exact image/driver/framework versions without exposing secrets.
- [ ] Propose reviewed remediation, rollback and end-to-end acceptance.

**Lab:** [4.4 — CUDA and Runtime Compatibility Validation](../labs/04-cuda-container-runtime-compatibility-check.md)  
**Next:** Chapter 4.5 — Dedicated GPU Node Pools and Resource Isolation.

## References

- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/)
- [NVIDIA System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [PyTorch CUDA](https://pytorch.org/docs/stable/cuda.html)
- [vLLM Documentation](https://docs.vllm.ai/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
