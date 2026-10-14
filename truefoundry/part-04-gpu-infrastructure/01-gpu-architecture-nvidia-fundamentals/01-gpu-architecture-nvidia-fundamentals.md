# 4.1 — GPU Architecture and NVIDIA GPU Fundamentals

**Part:** 04 — GPU Infrastructure  
**Audience:** TrueFoundry platform engineers, Kubernetes administrators, ML engineers, SRE/DBRE  
**Safety:** Production commands are read-only unless explicitly marked **[LAB-MUTATE]**.  
**Scope:** GPU fundamentals and evidence gathering. Kubernetes GPU scheduling, driver installation and CUDA repair are covered in Chapters 4.2–4.4.

## 1. Learning objectives

By the end, explain how GPUs execute AI workloads, identify the GPU hardware and software layers, interpret NVIDIA health and memory evidence, and locate the first failed transition when a TrueFoundry-managed model cannot use a GPU.

## 2. Architecture: from deployment intent to GPU execution

```text
TrueFoundry model/service configuration
       │ management path (control plane / tfy-agent)
       ▼
Kubernetes Deployment / Pod specification
       │ resource requests and placement
       ▼
Scheduler → GPU-capable Kubernetes node
       │ device allocation (device plugin)
       ▼
Container runtime + NVIDIA runtime integration
       │ device nodes and driver libraries exposed to container
       ▼
Framework (PyTorch / vLLM) → CUDA runtime / libraries
       │ kernel launches
       ▼
NVIDIA driver → physical GPU → GPU memory and compute units
```

**Critical distinctions:**
- A GPU instance type does **not** prove that Kubernetes advertises an allocatable GPU.
- A Pod running on a GPU node does **not** prove its container can see the device.
- A visible GPU in `nvidia-smi` does **not** prove that the application framework can initialize CUDA.
- TrueFoundry deployment success does **not** prove model readiness or inference success.
- A Kubernetes resource request for `nvidia.com/gpu` is an *extended resource*, not a CPU/memory request. It depends on device-plugin registration and compatible cluster configuration.

Treat TrueFoundry-specific controls as configuration intent; inspect the **rendered Kubernetes objects** for authoritative runtime evidence.

## 3. GPU hardware fundamentals

A GPU executes many parallel operations using streaming multiprocessors (SMs), cores and specialized accelerators (such as Tensor Cores on supported NVIDIA architectures). AI inference and training commonly involve large matrix operations and high memory bandwidth.

| Component | Operational meaning | Failure or capacity signal |
|---|---|---|
| GPU compute / SMs | Parallel execution capacity | Low utilization despite queued requests may indicate upstream bottlenecks |
| VRAM (device memory) | Holds model weights, activations, KV cache and buffers | CUDA out-of-memory or model load failure |
| Memory bandwidth | Rate at which data moves within GPU memory | Memory-bound kernels and lower throughput |
| PCIe / NVLink (where present) | Host-to-GPU / GPU-to-GPU communication | Slow transfers or multi-GPU communication bottlenecks |
| GPU driver | Host kernel/user-space interface to GPU | Driver not loaded, mismatch, inaccessible device |
| CUDA runtime/toolkit | Libraries and runtime used by GPU applications | Framework/CUDA compatibility failures |
| Device plugin | Advertises GPUs to Kubernetes | Missing `nvidia.com/gpu` capacity/allocatable |
| Container runtime integration | Makes GPU devices/libraries accessible inside container | GPU visible on host but not in workload |
| PyTorch / vLLM | Application/framework GPU detection and execution | `torch.cuda.is_available() == False`, vLLM device detection error |

**VRAM is not node RAM.** A node with abundant system RAM can still fail model loading when VRAM is insufficient. GPU utilization, memory allocation and GPU memory bandwidth are different measurements.

### 3.1 NVIDIA GPU families and compatibility

NVIDIA GPU models differ in architecture, VRAM, supported compute capability, tensor-core features, MIG support and interconnects. Do not assume a capability (including MIG or NVLink) is present just because a node has an NVIDIA GPU. Confirm the **exact GPU model** and vendor documentation before enabling features.

### 3.2 GPU partitioning and sharing

- **Whole GPU:** Common Kubernetes allocation model using integer `nvidia.com/gpu` resources.
- **MIG:** Hardware-supported GPU partitioning on *specific* NVIDIA models; resource names and exposure depend on plugin configuration.
- **Time slicing:** Shares execution time among workloads but does not inherently provide MIG-style memory isolation.
- **Multi-GPU:** Tensor parallelism and distributed serving introduce topology and interconnect requirements.

**Do not configure GPU sharing or MIG in production as a troubleshooting shortcut.** Changing these settings can disrupt other tenants.

## 4. Prerequisites and safety

- Read access to the intended Kubernetes context, nodes, Pods and events.
- Permission to use `kubectl debug` or exec **only** for approved disposable lab workloads; production access may be restricted.
- GPU node with compatible drivers/device plugin for optional lab.
- Existing TrueFoundry deployment identity, namespace and workload name if investigating a real incident.

Commands below use explicit placeholders. Before execution, set your own values and verify the context:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
```

Never assume `dev`, `staging` and `prod` are separate clusters; they may be namespaces in one cluster.

## 5. GPU discovery and read-only inventory [SAFE-READ]

### 5.1 Inventory nodes and advertised GPU resources

```bash
kubectl get nodes -o wide
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl describe node <gpu-node-name>
kubectl get node <gpu-node-name> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}'
```

If the JSONPath returns nothing, inspect `kubectl describe node` and check device-plugin registration. A blank value is **not** proof that hardware is absent.

### 5.2 Inventory workload requests and scheduling state

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod-name>
kubectl -n <namespace> get pod <pod-name> -o jsonpath='{range .spec.containers[*]}{.name}{" GPU limit: "}{.resources.limits.nvidia\.com/gpu}{"\n"}{end}'
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Inspect node selectors, affinity, taints, tolerations, resource limits, scheduling events and the actual assigned node. Do not conclude that a GPU request exists solely from a TrueFoundry UI setting.

### 5.3 Inspect driver and GPU evidence

If authorized on an existing GPU node or GPU-enabled lab Pod:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

`nvidia-smi` typically reports a **driver-supported maximum CUDA version**, not proof of the CUDA toolkit version installed in a particular container.

When a GPU container already exists and access is approved:

```bash
kubectl -n <namespace> logs <pod-name> --tail=200
kubectl -n <namespace> exec <pod-name> -- nvidia-smi
```

Avoid `exec` into production Pods unless permitted by your access and incident procedures; logs and Pod status are safer first steps.

## 6. Layer-by-layer failure classification

| Symptom | First checks | Likely owner boundary |
|---|---|---|
| GPU node not present | Cloud node pool, autoscaler events, node registration | Cloud / cluster infrastructure |
| Node present, no `nvidia.com/gpu` | Node capacity, device-plugin DaemonSet, driver health | GPU platform operations |
| Pod Pending | Events, GPU requests, affinity, taints, allocatable GPUs | Kubernetes scheduling / capacity |
| Pod Running, no device visible | Container resource limit, runtime integration, device exposure | GPU runtime integration |
| `nvidia-smi` works, PyTorch CUDA false | Framework build, CUDA compatibility, library loading | Image / ML runtime |
| vLLM: `No CUDA runtime is found` or `Failed to infer device type` | Pod placement → GPU allocation → container device → PyTorch CUDA | Identify first failed layer; do not assume one cause |
| CUDA OOM during model load | GPU memory used/free, model size, dtype, tensor parallelism | Model configuration / capacity |
| GPU appears idle, inference slow | Request traffic, queueing, CPU preprocessing, model metrics | Serving performance |

**Diagnostic rule:** find the **Lowest Proven Healthy Layer**, then the **First Failed Transition**. Record timestamped evidence and the current actionable owner.

## 7. TrueFoundry-specific operational example

A deployment targeting a dedicated GPU pool might expect labels:

```yaml
nodeSelector:
  node-pool: gpu-reserved
  gpu: "true"
  gpu_type: A10G
containers:
  - name: model
    resources:
      limits:
        nvidia.com/gpu: 1
```

This is a **fragment of a Kubernetes Pod spec**, not a complete TrueFoundry deployment configuration. These labels are environment-specific examples, not Kubernetes or TrueFoundry defaults. Verify actual labels with `kubectl get nodes --show-labels`; ensure GPU resource advertising, matching tolerations and capacity. GPU resources are commonly requested via limits; Kubernetes treats an omitted GPU request as equal to the specified GPU limit for this extended resource.

## 8. Production incident runbook [SAFE-READ]

1. Confirm cluster context, namespace, deployment identity and affected service.
2. Determine blast radius: one Pod, one GPU node, one node pool or all GPU workloads.
3. Capture Pod status, events, owner reference, image and resource limits.
4. Capture target-node labels, taints, Ready state and `nvidia.com/gpu` allocatable.
5. Review NVIDIA device-plugin/operator status and logs **without changing configuration**.
6. If authorized, compare host GPU health with container visibility.
7. Check framework initialization errors and model-load logs.
8. Classify the first failed transition and assign the actionable owner.
9. Apply a reviewed fix through the authoritative owner (IaC/Helm/TrueFoundry configuration), not an undocumented hot patch.
10. Validate recovery: Pod scheduled → GPU allocated → CUDA available → model Ready → successful inference; capture post-change evidence.

### Escalation evidence template

```text
Incident / timestamp:
Cluster / namespace / workload:
TrueFoundry deployment and Kubernetes owner:
GPU model / node pool / node:
Pod state / events:
Requested vs allocatable nvidia.com/gpu:
Driver and device-plugin evidence:
Container nvidia-smi / framework CUDA evidence:
Lowest Proven Healthy Layer:
First Failed Transition:
Blast radius / customer impact:
Current actionable owner:
Proposed safe action / rollback:
Recovery acceptance evidence:
```

## 9. Monitoring and capacity signals

Track per-GPU utilization, VRAM used/free, temperature, power, ECC/error signals where supported, node health, allocatable GPUs, pending GPU Pods, device-plugin health, and application-level inference latency, queueing and errors. Use NVIDIA DCGM exporter / Prometheus where deployed; metric names and support vary by GPU and exporter version.

**Avoid one-metric diagnoses:** high VRAM usage alone is not proof of a leak; low GPU utilization alone is not proof of underprovisioning.

## 10. Failure simulations and rollback

Run failure simulations **only in disposable environments**. In this chapter, use non-destructive observation and optional disposable GPU smoke-test Pod; defer driver/plugin removal, MIG reconfiguration, taint changes and node replacement to controlled later labs.

Rollback for the optional lab is simply deleting the disposable smoke-test Pod and confirming no remaining lab resources. Never restart shared GPU drivers or operators as part of this foundational lab.

## 11. Acceptance criteria

- [ ] Can distinguish GPU hardware, driver, CUDA, device plugin, container runtime and application layers.
- [ ] Can identify actual GPU node model and Kubernetes advertised GPU capacity.
- [ ] Can explain host vs container vs framework GPU visibility.
- [ ] Can identify the first failed transition for a vLLM GPU-detection error.
- [ ] Can collect read-only evidence without changing shared GPU nodes.
- [ ] Can describe blast radius, remediation owner and end-to-end recovery evidence.

## 12. Hands-on lab and next chapter

**Lab:** [01 — GPU Architecture and NVIDIA Fundamentals Check](../labs/01-gpu-architecture-nvidia-fundamentals-check.md)  
**Next:** Chapter 4.2 — GPU Nodes and Kubernetes GPU Scheduling.

## References

- [Kubernetes — Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [NVIDIA — System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [NVIDIA — Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA — CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [NVIDIA — GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
