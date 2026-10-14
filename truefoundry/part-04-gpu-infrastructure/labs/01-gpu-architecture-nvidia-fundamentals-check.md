# Lab 4.1 — GPU Architecture and NVIDIA Fundamentals Check

**Companion:** [Canonical Chapter 4.1](../01-gpu-architecture-nvidia-fundamentals/01-gpu-architecture-nvidia-fundamentals.md)  
**Production mode:** [SAFE-READ]  
**Optional smoke test:** [LAB-MUTATE] disposable cluster/namespace only; approved GPU image required.

## Outcomes

Prove the difference between (a) GPU hardware existence, (b) Kubernetes GPU advertising, (c) Pod GPU allocation, (d) container visibility, and (e) framework CUDA availability. Record the first failed transition, not just the final error.

## 0. Preflight and safety

- Use a known cluster context; never assume `dev` and `prod` are separate clusters.
- Do not change GPU drivers, device plugins, node labels, taints, GPU sharing or workloads in production.
- Collect only non-sensitive evidence. Redact tenant names, registry credentials and tokens from shared output.
- `kubectl exec` may be disallowed in production; use existing logs or authorized debug environments.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
```

**Record:** context, cluster, date/time, intended namespace, authorized scope.

## 1. GPU node inventory [SAFE-READ]

```bash
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl get nodes -o custom-columns='NAME:.metadata.name,READY:.status.conditions[-1].type,GPU_CAPACITY:.status.capacity.nvidia\.com/gpu,GPU_ALLOCATABLE:.status.allocatable.nvidia\.com/gpu'
kubectl describe node <gpu-node-name>
```

**Interpretation:** custom-columns output is an inventory aid; use `describe node` to verify actual Ready condition, capacity and allocatable values. Missing `nvidia.com/gpu` means the Kubernetes GPU resource is not advertised; it does not independently prove missing hardware.

**Evidence:** GPU model from authorized host inspection or cloud inventory, labels, taints, capacity, allocatable, node Ready condition.

## 2. Inspect an existing workload [SAFE-READ]

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod-name>
kubectl -n <namespace> get pod <pod-name> -o jsonpath='{range .spec.containers[*]}{.name}{": "}{.resources.limits.nvidia\.com/gpu}{" GPUs\n"}{end}'
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod-name> --tail=100
```

**Evidence:** actual Pod owner, image, node, GPU limit, placement constraints, conditions, recent errors. If multiple containers exist, inspect each container and any init containers separately.

## 3. Driver/device visibility [SAFE-READ where authorized]

On an already authorized GPU host or existing GPU-enabled lab container:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,driver_version,memory.total,memory.used,utilization.gpu --format=csv
```

For an existing approved lab Pod:

```bash
kubectl -n <lab-namespace> exec <gpu-pod> -- nvidia-smi
```

**Checkpoint:** A healthy host result and failed container result narrows the fault to allocation/runtime exposure or container configuration. Neither alone proves framework compatibility.

## 4. Optional PyTorch validation [SAFE-READ in approved existing lab container]

Only if Python and PyTorch are already installed:

```bash
python -c "import torch; print('torch=',torch.__version__); print('compiled_cuda=',torch.version.cuda); print('cuda_available=',torch.cuda.is_available()); print('device_count=',torch.cuda.device_count())"
```

If `torch.cuda.is_available()` is false, capture the complete exception/logs and check whether the PyTorch build supports CUDA. Do not install or replace packages in production during diagnosis.

## 5. Optional disposable GPU smoke test [LAB-MUTATE]

**Only in a disposable, approved GPU-enabled cluster.** The example assumes a working device plugin and an approved NVIDIA CUDA image with `nvidia-smi`. Substitute an organization-approved, immutable image digest. Avoid floating tags and untrusted registries.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-ch04-smoke
  namespace: gpu-lab
  labels:
    app: gpu-ch04-smoke
spec:
  restartPolicy: Never
  containers:
    - name: check
      image: <approved-cuda-image@sha256:digest>
      command: ["/bin/sh", "-c"]
      args: ["nvidia-smi && echo GPU_SMOKE_PASS"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

Create the `gpu-lab` namespace only if authorized and absent, then apply the manifest after replacing the image placeholder:

```bash
kubectl create namespace gpu-lab
kubectl apply -f gpu-ch04-smoke.yaml
kubectl -n gpu-lab get pod gpu-ch04-smoke -o wide
kubectl -n gpu-lab describe pod gpu-ch04-smoke
kubectl -n gpu-lab logs gpu-ch04-smoke
```

If the namespace already exists, **skip** `kubectl create namespace`. Do not delete a pre-existing namespace. If GPU pool requires labels/taints, adapt only the disposable Pod with reviewed nodeSelector/tolerations; do not alter shared nodes.

### Controlled failure interpretation

| Observed result | Investigate |
|---|---|
| Pending: Insufficient nvidia.com/gpu | GPU allocatable, existing allocations, node-pool capacity |
| Pending: affinity/taint mismatch | Pod selector/affinity/tolerations vs node constraints |
| ContainerCreateError | Runtime/device injection and node events |
| `nvidia-smi` command missing | Image does not contain utility; not proof of GPU failure |
| `nvidia-smi` cannot communicate with driver | Driver/device exposure/runtime integration |
| GPU smoke passes, model fails | Framework build, CUDA libraries, model loading and app configuration |

### Cleanup [LAB-MUTATE]

```bash
kubectl -n gpu-lab delete pod gpu-ch04-smoke --ignore-not-found
kubectl -n gpu-lab get pods
```

Delete the `gpu-lab` namespace **only if created exclusively for this lab** and confirmed to contain no other resources.

## 6. Cloud-provider interpretation

- **EKS:** Verify GPU instance family, node group/Karpenter provisioning, AMI driver support, device plugin and node allocatable GPUs. Do not assume every EKS AMI includes drivers.
- **AKS:** Verify GPU VM SKU, supported node image/driver provisioning, GPU operator/plugin strategy and allocatable GPUs. Configuration varies by AKS version.
- **GKE:** Verify GPU node-pool accelerator configuration, driver installation mode, plugin integration and allocatable GPUs. Managed behavior varies by GKE mode/version.

For all three: hardware inventory, Kubernetes advertising, Pod allocation and framework availability are separate checks.

## 7. Incident exercise: vLLM cannot infer device

Given `No CUDA runtime is found` or `Failed to infer device type`, document:
1. Does the node have a physical GPU?
2. Does Kubernetes advertise `nvidia.com/gpu`?
3. Did the affected Pod request and receive a GPU on a compatible node?
4. Can its container access the GPU device and driver?
5. Can PyTorch initialize CUDA?
6. Can vLLM load the model and serve an inference request?

**Do not assume the error proves a missing GPU node.** Record the earliest failed check.

## 8. Acceptance record

```text
Cluster/context:
Namespace/workload:
Timestamp:
GPU model:
GPU advertised capacity/allocatable:
Pod GPU limit and assigned node:
Node taints/labels:
Host nvidia-smi:
Container nvidia-smi:
PyTorch CUDA availability:
First failed transition:
Blast radius:
Current actionable owner:
Recommended action (no unapproved mutation):
Lab smoke outcome:
Cleanup confirmed:
```

**Pass:** Each layer has evidence or an explicitly marked access limitation; no shared-node mutation occurred; optional smoke Pod cleaned up.

**Next:** Chapter 4.2 — GPU Nodes and Kubernetes GPU Scheduling.
