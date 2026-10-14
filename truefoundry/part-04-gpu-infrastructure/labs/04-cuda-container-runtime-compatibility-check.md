# Lab 4.4 — CUDA, Container Runtime and Compatibility Validation

**Canonical:** [Chapter 4.4](../04-cuda-container-runtime-compatibility/04-cuda-container-runtime-compatibility.md)  
**Default mode:** [SAFE-READ]. Optional smoke test: [LAB-MUTATE] only in an approved disposable GPU-enabled cluster.

## Objectives

Prove GPU scheduling, device allocation, host/container visibility, PyTorch CUDA initialization and vLLM readiness as separate checks. Build a compatibility evidence matrix without changing shared GPU infrastructure.

## 0. Safety preflight

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
```

Record the exact cluster, namespace, TrueFoundry deployment and time. Never run `kubectl exec` into production Pods without explicit authorization. No package installs, privileged debugging, runtime restarts or driver changes in production.

## 1. Kubernetes GPU allocation [SAFE-READ]

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{" GPU allocatable\n"}'
```

**Record:** Pod owner, immutable image digest if available, GPU limit, assigned node, PodScheduled condition, container state and relevant events. If Pod is Pending and unscheduled, diagnose scheduling first.

## 2. Host versus container GPU visibility [SAFE-READ where authorized]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=name,driver_version,memory.total,memory.used --format=csv
```

Inside an approved existing GPU Pod containing the utility:

```bash
kubectl -n <lab-namespace> exec <gpu-pod> -- nvidia-smi
```

**Record:** host driver version, GPU model and whether the container can see the GPU. A missing `nvidia-smi` executable inside the container may be an image/tooling issue, not a GPU-access failure.

## 3. PyTorch build and runtime [SAFE-READ in an approved diagnostic container]

If PyTorch is installed:

```bash
python -c "import torch; print('torch=',torch.__version__); print('compiled_cuda=',torch.version.cuda); print('available=',torch.cuda.is_available()); print('count=',torch.cuda.device_count())"
```

For an **approved disposable** GPU container, test execution:

```bash
python -c "import torch; x=torch.ones(4,device='cuda'); print('CUDA_EXEC_PASS',x.sum().item())"
```

**Interpretation:** `torch.version.cuda` reports the build's CUDA version; `is_available` reports current runtime availability; a successful GPU operation proves more than a successful import. If PyTorch is not installed, skip rather than mutate a production image.

## 4. Version evidence worksheet

| Layer | Observed version / value | Evidence source | Compatible? |
|---|---|---|---|
| GPU model / compute capability | | Cloud/vendor inventory | |
| Node OS / kernel | | Approved node inventory | |
| Host NVIDIA driver | | `nvidia-smi` | |
| Device plugin / GPU Operator | | Installed deployment manifests | |
| Container image digest | | Rendered Pod spec | |
| CUDA runtime / libraries | | Approved image metadata | |
| PyTorch version / CUDA build | | Python diagnostic | |
| vLLM version | | Image metadata / startup logs | |

Consult official NVIDIA, PyTorch and vLLM version documentation. Mark unknown values **Unknown**, never infer compatibility from a single version string.

## 5. Optional disposable PyTorch GPU smoke test [LAB-MUTATE]

Use an approved, immutable, CUDA-enabled PyTorch image that includes Python and PyTorch; replace the placeholder with a real digest. The GPU device plugin must already be functioning.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-compat-ch04
  namespace: gpu-lab
spec:
  restartPolicy: Never
  containers:
    - name: pytorch
      image: <approved-pytorch-cuda-image@sha256:digest>
      command: ["python", "-c"]
      args:
        - |
          import torch
          print("torch", torch.__version__)
          print("build_cuda", torch.version.cuda)
          assert torch.cuda.is_available(), "CUDA unavailable"
          print("CUDA_EXEC_PASS", torch.ones(4, device="cuda").sum().item())
      resources:
        requests:
          cpu: "100m"
          memory: "512Mi"
        limits:
          memory: "2Gi"
          nvidia.com/gpu: 1
```

Create `gpu-lab` only if absent and authorized, then apply the manifest:

```bash
kubectl create namespace gpu-lab
kubectl apply -f cuda-compat-ch04.yaml
kubectl -n gpu-lab get pod cuda-compat-ch04 -o wide
kubectl -n gpu-lab describe pod cuda-compat-ch04
kubectl -n gpu-lab logs cuda-compat-ch04
```

Skip the create command if the namespace already exists. If the Pod remains Pending, check placement, resource availability and events; do not change shared node labels/taints.

**Success:** `CUDA_EXEC_PASS 4.0` appears in logs and Pod completes successfully. The exact printed floating-point formatting can vary.

Cleanup:

```bash
kubectl -n gpu-lab delete pod cuda-compat-ch04 --ignore-not-found
kubectl -n gpu-lab get pods
```

Delete the namespace only if created solely for this lab and confirmed empty.

## 6. Failure investigation exercises (non-destructive)

1. **CPU-only PyTorch:** `torch.version.cuda is None`. Identify the image/package owner and compatible CUDA-enabled build.
2. **Driver/runtime mismatch:** PyTorch CUDA build exists but initialization reports an insufficient driver. Record exact versions and validate official compatibility guidance.
3. **GPU invisible in container:** host GPU healthy, Pod scheduled, but CUDA unavailable. Inspect GPU allocation and runtime integration before changing the image.
4. **vLLM device inference failure:** PyTorch CUDA works, vLLM fails. Review vLLM release/platform compatibility and startup logs.
5. **CUDA OOM:** model fails after CUDA initialization. Distinguish VRAM exhaustion from host RAM and Kubernetes scheduling.

For each scenario, document **Lowest Proven Healthy Layer**, **First Failed Transition**, actionable owner and approved remediation.

## 7. Provider-specific checks

**EKS:** GPU node AMI, driver/container toolkit strategy, device plugin and Pod resource limits.  
**AKS:** GPU VM SKU, node image/driver integration and CUDA-enabled image support.  
**GKE:** GPU node-pool driver mode, device exposure and compatible framework image.

Do not infer that any cloud provider universally supplies the same driver/runtime configuration.

## 8. Acceptance and incident evidence

```text
Date/time / context / namespace:
TrueFoundry deployment / Kubernetes controller:
Pod image digest / GPU limit / assigned node:
GPU model / host driver version:
Device plugin and runtime ownership:
Host nvidia-smi:
Container GPU visibility:
PyTorch version / compiled CUDA:
torch.cuda.is_available / GPU operation:
vLLM version / initialization logs:
Lowest Proven Healthy Layer:
First Failed Transition:
Blast radius / current actionable owner:
Compatibility matrix reviewed:
Approved fix / rollback:
Inference validation:
Lab cleanup:
```

**Pass:** Each GPU layer is either evidenced or explicitly marked inaccessible, no production mutation occurs, and optional disposable resources are cleaned up.

**Next:** Chapter 4.5 — Dedicated GPU Node Pools and Resource Isolation.
