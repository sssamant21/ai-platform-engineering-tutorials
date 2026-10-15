# 6.3 — CPU and GPU Inference Runtimes and Scheduling

**Part:** 06 — Model Serving · **Chapter:** 3 of 7  
**Audience:** TrueFoundry platform engineers, MLOps engineers, GPU infrastructure engineers and SREs  
**Safety:** Default diagnostics are **[SAFE-READ]**. Deployment, node-pool, GPU allocation, driver and runtime changes are **[CHANGE-CONTROL]**. Experiments must use approved nonproduction resources.

## 1. Learning outcomes

Select and validate CPU/GPU inference runtimes; understand PyTorch and ONNX Runtime execution providers; trace Kubernetes accelerator scheduling and NVIDIA runtime dependencies; size host RAM and GPU VRAM; diagnose device-detection failures, OOM, Pending Pods and latency regressions; and perform safe rollback of the full runtime stack.

## 2. Inference execution architecture

```text
TrueFoundry model deployment / approved image + model version
       ↓
Kubernetes scheduler → node placement / extended resources
       ↓
Container runtime → device-plugin allocation / CDI or runtime hooks
       ↓
Application process
       ├── request validation / preprocessing
       ├── inference runtime: PyTorch / ONNX Runtime / other
       ├── CPU execution or CUDA execution provider
       ├── weights, activations, buffers and caches
       └── postprocessing / response
       ↓
Service endpoints → authenticated client → SLO telemetry
```

TrueFoundry configures and manages supported deployments; the actual framework/runtime stack resides in the workload image and cluster. A successful platform deployment does not establish that CUDA can initialize or that a model fits GPU memory.

## 3. CPU versus GPU decision matrix

| Dimension | CPU inference | GPU inference |
|---|---|---|
| Best fit | Smaller models, low/moderate concurrency, portability | Compute-intensive models, high parallelism and throughput |
| Scheduler resources | CPU, memory, ephemeral storage | CPU, memory, GPU extended resource and GPU-compatible node |
| Runtime dependencies | CPU-optimized framework and math libraries | Compatible NVIDIA driver, CUDA runtime/framework and device access |
| Memory | Host RAM | Host RAM plus device VRAM |
| Bottlenecks | CPU saturation, thread oversubscription, memory bandwidth | VRAM, GPU compute, PCIe transfer, batching, CPU preprocessing |
| Cost | Often simpler, lower fixed cost | Higher fixed cost; utilization and batching critical |
| Failure modes | OOMKilled, CPU throttling, contention | Device detection, GPU OOM, CUDA mismatch, kernel errors |

Benchmark **your** model and input distribution before choosing hardware. GPU is not universally faster for tiny batches or preprocessing-heavy workloads.

## 4. PyTorch and ONNX Runtime fundamentals

**PyTorch:** model/device placement must be explicit and compatible with installed builds. `torch.cuda.is_available()` tests CUDA availability for that process; it does not prove sufficient VRAM or correct model execution. Use inference/eval mode and validated input dtype/device.

**ONNX Runtime:** execution providers (e.g., `CPUExecutionProvider`, `CUDAExecutionProvider`) determine where supported operators execute. A provider can be listed as available but session initialization or specific operators may still fail or fall back. Validate the **active session providers**, operator placement where observable, correctness and latency.

Example **read-only runtime diagnostics inside an approved existing container**:

```bash
python -c "import torch; print('torch', torch.__version__); print('cuda build', torch.version.cuda); print('cuda available', torch.cuda.is_available()); print('devices', torch.cuda.device_count())"
```

```bash
python -c "import onnxruntime as ort; print('onnxruntime', ort.__version__); print('providers', ort.get_available_providers())"
```

These require the libraries in that container. Do not install or modify packages inside production containers merely to run diagnostics.

## 5. NVIDIA GPU dependency chain

```text
Physical GPU and node hardware
 → NVIDIA kernel driver / device nodes
 → Kubernetes node registration
 → NVIDIA device plugin / GPU Operator where installed
 → node allocatable nvidia.com/gpu
 → scheduler places Pod requesting GPU
 → kubelet device allocation / runtime integration
 → container CUDA libraries + framework compatibility
 → runtime initializes device
 → model weights allocate / inference executes
```

Failure at any earlier layer can surface as a framework error such as “No CUDA runtime” or “Failed to infer device type.” Distinguish host driver visibility from container runtime visibility.

### Inspect node allocation [SAFE-READ]

```bash
kubectl get nodes -o wide
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\\.com/gpu
kubectl describe node <gpu-node>
kubectl -n <namespace> describe pod <gpu-pod>
```

Note: the custom-column GPU field can be sensitive to kubectl JSONPath escaping and Kubernetes versions; `kubectl describe node` is the reliable fallback for Allocatable and Allocated resources.

```bash
kubectl -n <namespace> get pod <gpu-pod> -o jsonpath='{.spec.containers[*].resources}{"\n"}'
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

GPU extended resources are integer-valued for standard `nvidia.com/gpu` allocations; request and limit rules differ from ordinary CPU resources. GPU sharing, MIG and time-slicing require explicit platform/device-plugin configuration and change isolation; do not assume one advertised GPU equals exclusive access to a physical GPU in every cluster.

## 6. Illustrative GPU Pod configuration

The following is a **Kubernetes Pod-template fragment**, not a TrueFoundry API schema:

```yaml
containers:
  - name: inference
    image: registry.example.com/ml/inference@sha256:<approved-digest>
    resources:
      requests:
        cpu: "1000m"
        memory: 8Gi
        nvidia.com/gpu: 1
      limits:
        memory: 12Gi
        nvidia.com/gpu: 1
    ports:
      - name: http
        containerPort: 8080
    startupProbe:
      httpGet:
        path: /startup
        port: http
      periodSeconds: 10
      failureThreshold: 90
    readinessProbe:
      httpGet:
        path: /ready
        port: http
      periodSeconds: 5
      timeoutSeconds: 2
```

Replace the image digest and sizing values with measured, approved settings. Validate actual health endpoints and the node's allocatable GPU count. Node selectors, tolerations and affinity must match the GPU node pool. A Pod may remain Pending even when a GPU exists if CPU, RAM, topology, taints or quotas prevent scheduling.

## 7. Driver, CUDA and framework compatibility

Three distinct versions matter:
1. **Host NVIDIA driver** must support the CUDA user-space runtime used by the container.
2. **Container CUDA libraries/runtime** must match the framework build and inference engine requirements.
3. **Framework/model stack** must support the GPU architecture, precision and kernels being used.

`nvidia-smi` typically reports the driver and maximum supported CUDA API version; it does **not** prove the container's CUDA toolkit or framework build is compatible. Check official compatibility matrices for exact versions before changing drivers or images.

### Approved container diagnostics

```bash
nvidia-smi
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
```

If `nvidia-smi` is absent from an image, that alone does not prove GPU allocation failure. Use a pre-approved diagnostics image or platform metrics rather than installing tools in production.

## 8. GPU memory and inference sizing

VRAM is consumed by model weights, activations, temporary kernels/workspaces, runtime context, framework allocator, and—especially for generative serving—KV cache. Actual footprint depends on dtype, quantization, batch size, input sequence length, concurrent requests, engine and fragmentation.

A rough **weights-only** lower bound is:

```text
weight_bytes ≈ parameter_count × bytes_per_parameter
```

For example, a 7-billion-parameter model at FP16 has roughly 14 GB decimal (about 13.0 GiB) of weights before runtime overhead, KV cache, buffers or fragmentation. This does **not** imply it fits on a 16 GiB GPU in a real serving configuration.

**Host memory and VRAM are separate.** A container can be OOMKilled due to host memory while the GPU has free VRAM, or encounter CUDA OOM while Kubernetes host-memory usage appears safe.

## 9. CPU sizing and threading

Tune worker processes, intra-op/inter-op threads, preprocessing pools and batch/concurrency limits. Excess threads can oversubscribe CPUs, amplify context switching and increase P99 latency. Compare CPU requests, CPU limits, throttling, runtime thread count and queue depth.

Avoid treating increased replica count as a substitute for per-replica model footprint and startup time. A scale-out event may simultaneously download large model artifacts and exhaust registry/network/storage throughput.

## 10. GPU scheduling and placement

Inspect:
- GPU node labels and taints; required/preferred affinity and tolerations.
- Allocatable versus requested GPUs and resource quotas.
- Device plugin/Operator health, GPU feature discovery and MIG configuration.
- CPU/memory headroom and topology spread.
- GPU node autoscaling latency and model cold-start time.
- Cluster autoscaler/Karpenter compatibility and capacity availability.
- Isolation, tenant quotas and utilization/cost tradeoffs.

An autoscaler cannot provision an unavailable GPU SKU, satisfy a conflicting affinity rule or bypass cloud quota.

## 11. Model runtime readiness

**Startup:** allow time for model fetch, checksum verification, framework initialization, GPU allocation and warmup.  
**Readiness:** report success only when the model is loaded and the process can serve valid inference.  
**Liveness:** detect a stuck process without restarting all replicas during transient object-store or downstream outages.

For production, test graceful SIGTERM handling, request draining, rollout surge and model-loading concurrency. A Kubernetes Ready Pod alone is not evidence of model correctness; run an approved golden inference request.

## 12. Performance observability

| Signal | Meaning | Caution |
|---|---|---|
| Request P95/P99 | User-perceived tail latency | Compare same payload mix |
| Queue wait | Admission/batching delay | May dominate at high concurrency |
| CPU throttling | Enforced CPU bandwidth | Requires cgroup metrics |
| Host RSS/working set | Container memory pressure | Different from VRAM |
| GPU utilization | Device busy time | Not equivalent to useful throughput |
| GPU memory used/free | VRAM footprint | Does not show fragmentation alone |
| GPU memory OOM events | Allocation failure | Correlate with model and batch |
| Model load time | Cold-start impact | Separate fetch and initialization |
| Throughput / batch size | Serving capacity | Higher throughput may worsen latency |
| Restarts / readiness | Availability | Distinguish probe from runtime failure |

Use approved GPU telemetry such as DCGM exporter where deployed. Metric names and labels depend on exporter/version. Do not infer idle GPU from a single short sampling window.

## 13. Troubleshooting matrix

| Symptom | Evidence | First failed transition |
|---|---|---|
| GPU Pod Pending | Scheduler events, taints, allocatable | Request → node placement |
| `nvidia.com/gpu` absent | Node resources, device plugin | Driver/plugin → registration |
| `nvidia-smi` fails in container | Device allocation, runtime integration | Kubelet → container device |
| CUDA unavailable in PyTorch | Framework build, device visibility | Container → CUDA runtime |
| “Failed to infer device type” | Runtime logs, CUDA availability | Framework → device selection |
| Unsupported GPU architecture | Framework kernels, device model | Runtime → GPU ISA |
| CUDA OOM during load | VRAM metrics, model footprint | Model allocation → VRAM |
| CUDA OOM at high concurrency | Batch/sequence/KV cache | Admission → VRAM |
| Container OOMKilled | Pod last state, host RAM | Process → memory cgroup |
| Slow P99, low GPU utilization | Queue/preprocess/CPU/network | Upstream → GPU execution |
| Slow P99, high GPU saturation | Queue depth, kernel time | Execution capacity → response |
| High CPU throttling | cgroup metrics, CPU limit | CPU cap → runtime |
| Inference wrong on GPU | Golden outputs, dtype/precision | Execution → correctness |
| Model not Ready | Startup/warmup/probe logs | Runtime → readiness |
| Rollback still fails | Driver/model/image mismatch | Release → compatibility contract |

## 14. Production incident runbook

1. Record TrueFoundry workspace/deployment, model version, image digest, node and GPU SKU.
2. Capture first failure time, error rate, P95/P99, throughput, Pod events and previous logs.
3. Verify scheduler placement and `nvidia.com/gpu` allocation before debugging CUDA.
4. Verify node driver/device plugin/runtime, then container CUDA/framework and model load.
5. Separate **host OOM**, **GPU VRAM OOM**, **probe failure** and **application crash**.
6. Compare baseline versus failing revision's model size, precision, batch, concurrency and runtime versions.
7. Select smallest approved remediation or restore a known-good **image + model + resource/placement configuration** via TrueFoundry/GitOps.
8. Verify readiness, golden inference, GPU/CPU memory, queue depth, error rate and latency over an agreed observation window.

Do not restart shared GPU nodes, change host drivers, enable broad privileged containers or disable security controls without an approved change.

## 15. Acceptance gates

| Gate | Evidence |
|---|---|
| G1 | Runtime/framework and model version pinned |
| G2 | CPU/GPU selection based on workload benchmark |
| G3 | GPU node/device plugin and resource allocation verified |
| G4 | Driver/CUDA/framework compatibility documented |
| G5 | Host memory and VRAM budget validated separately |
| G6 | Startup/readiness and golden inference verified |
| G7 | P95/P99, queue, CPU/GPU and OOM telemetry captured |
| G8 | Compatible rollback and cleanup plan documented |

## 16. Evidence template

```text
Timestamp / TrueFoundry workspace / cluster:
Deployment revision / image digest / model version:
CPU/GPU inference runtime and versions:
GPU SKU, node, driver and allocatable resources:
Pod requests, limits, affinity and tolerations:
Host memory and GPU VRAM usage:
Framework CUDA/device detection:
Model load/warmup time:
Readiness / golden inference:
P95/P99, queue, throughput, CPU/GPU utilization:
Lowest Proven Healthy Layer:
First Failed Transition:
Approved remediation / owner:
Known-good image+model+runtime rollback:
Post-recovery acceptance and cleanup:
```

**Hands-on lab:** [6.3 — CPU/GPU Runtime and Scheduling Validation](../labs/03-cpu-gpu-inference-runtimes-scheduling-check.md)  
**Next:** Chapter 6.4 — Inference Endpoints, Request Lifecycle and API Reliability.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [PyTorch CUDA Semantics](https://docs.pytorch.org/docs/stable/notes/cuda.html)
- [ONNX Runtime Execution Providers](https://onnxruntime.ai/docs/execution-providers/)
- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
