# Lab 4.3 — NVIDIA Drivers, Device Plugin and GPU Operator Validation

**Companion:** [Chapter 4.3](../03-nvidia-drivers-device-plugin-gpu-operator/03-nvidia-drivers-device-plugin-gpu-operator.md)  
**Default:** [SAFE-READ]. Optional GPU smoke test is [LAB-MUTATE] on a disposable, approved cluster only.

## Objectives

Determine GPU software-stack ownership; verify driver and device-plugin registration; compare host, Kubernetes and container GPU visibility; diagnose a missing GPU resource without changing shared GPU nodes.

## 0. Safety and scope

Do not restart GPU drivers, delete operator/device-plugin Pods, change Helm releases, reboot nodes, modify taints or install privileged DaemonSets in production. Record exact context and obtain authorization before host inspection or Pod exec.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
```

## 1. Identify GPU stack ownership [SAFE-READ]

```bash
kubectl get daemonsets -A
kubectl get deployments -A
kubectl get pods -A -o wide
kubectl api-resources | grep -i -E 'clusterpolic|nvidia'
```

Document whether this environment uses GPU Operator, a standalone device plugin, a cloud-managed GPU node image or a combination with clearly divided ownership. Do not infer ownership from Pod names alone; verify Helm/IaC documentation if available.

## 2. Verify GPU resource registration [SAFE-READ]

```bash
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}'
```

Record Ready status, GPU capacity, allocatable, labels, taints and allocated resources. Blank GPU resource fields are registration evidence, not proof of missing hardware.

## 3. Inspect plugin/operator health [SAFE-READ]

Discover actual namespace and Pod names before running:

```bash
kubectl -n <gpu-system-namespace> get pods -o wide
kubectl -n <gpu-system-namespace> describe pod <device-plugin-pod>
kubectl -n <gpu-system-namespace> logs <device-plugin-pod> --tail=150
kubectl -n <gpu-system-namespace> get events --sort-by=.lastTimestamp
```

If GPU Operator is installed and its CRD is available:

```bash
kubectl get clusterpolicies.nvidia.com
kubectl describe clusterpolicies.nvidia.com <clusterpolicy-name>
```

Do not run the ClusterPolicy commands if the CRD is absent. Record component states and errors without modifying them.

## 4. Host driver validation [SAFE-READ when authorized]

On an approved GPU node shell:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,driver_version,memory.total,memory.used --format=csv
lsmod | grep nvidia
cat /proc/driver/nvidia/version
```

**Interpretation:** a host failure points toward hardware/driver/OS. A healthy host with missing Kubernetes GPU resources points toward registration/plugin/kubelet. The `nvidia-smi` CUDA version is **not** the container toolkit version.

## 5. Container and framework checks [SAFE-READ in an approved existing lab Pod]

```bash
kubectl -n <lab-namespace> describe pod <gpu-pod>
kubectl -n <lab-namespace> exec <gpu-pod> -- nvidia-smi
```

If PyTorch exists in that container:

```bash
kubectl -n <lab-namespace> exec <gpu-pod> -- python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
```

A missing `nvidia-smi` executable may mean the container image lacks the utility; do not automatically diagnose device failure. If PyTorch is absent, do not install it in a production Pod.

## 6. Optional disposable smoke Pod [LAB-MUTATE]

Use an approved disposable GPU cluster and immutable, organization-approved CUDA image. Substitute the placeholder with a real image digest and ensure your cluster's GPU scheduling policy allows this Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-stack-ch04
  namespace: gpu-lab
spec:
  restartPolicy: Never
  containers:
    - name: gpu-check
      image: <approved-cuda-image@sha256:digest>
      command: ["/bin/sh", "-c"]
      args: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

Only if `gpu-lab` does not exist and creation is approved:

```bash
kubectl create namespace gpu-lab
```

Then:

```bash
kubectl apply -f gpu-stack-ch04.yaml
kubectl -n gpu-lab get pod gpu-stack-ch04 -o wide
kubectl -n gpu-lab describe pod gpu-stack-ch04
kubectl -n gpu-lab logs gpu-stack-ch04
```

If scheduling fails, preserve events and compare GPU resources/taints. **Do not** change shared nodes to force the smoke Pod to run.

Cleanup:

```bash
kubectl -n gpu-lab delete pod gpu-stack-ch04 --ignore-not-found
kubectl -n gpu-lab get pods
```

Delete `gpu-lab` only if created exclusively for this lab and confirmed empty.

## 7. Failure classification exercise

| Observation | First suspected boundary | Required next evidence |
|---|---|---|
| Host `nvidia-smi` fails | Driver / node image / hardware | Kernel/driver version and node events |
| Host works; `nvidia.com/gpu` missing | Device plugin / kubelet | Plugin logs and node registration |
| Resource advertised; Pod Pending | Scheduler/capacity | Scheduler events and GPU allocations |
| Pod assigned; container device unavailable | Runtime/device allocation | Container events, toolkit/runtime configuration |
| Container sees GPU; PyTorch CUDA false | Framework/CUDA build | Framework version, CUDA compatibility |
| Operator exporter unhealthy; inference healthy | Metrics pipeline | Exporter/collector logs |

Explain **Lowest Proven Healthy Layer**, **First Failed Transition**, blast radius and current actionable owner for each case.

## 8. Provider-specific evidence

**EKS:** GPU instance and node group/Karpenter, AMI/kernel, driver/toolkit strategy, plugin DaemonSet.  
**AKS:** GPU VM SKU, node image/driver mode, plugin/operator configuration, Kubernetes allocatable GPUs.  
**GKE:** accelerator node pool, driver installation strategy, resource advertising and runtime behavior.

Managed behavior varies by version; verify actual provider documentation and installed components.

## 9. Acceptance report

```text
Date / context / namespace:
GPU node and hardware model:
GPU stack owner (operator / standalone / managed):
Host OS / kernel / NVIDIA driver:
Host nvidia-smi:
Device plugin Pod status and logs:
Operator/ClusterPolicy status if installed:
Node GPU capacity and allocatable:
Pod request and assignment:
Container nvidia-smi:
Framework CUDA result:
Lowest Proven Healthy Layer:
First Failed Transition:
Blast radius and actionable owner:
Approved remediation and rollback:
Smoke Pod outcome / cleanup:
```

**Pass:** Evidence establishes the relevant software boundary, no production mutation occurs, and optional disposable resources are cleaned up.

**Next:** Chapter 4.4 — CUDA, Container Runtime and Compatibility.
