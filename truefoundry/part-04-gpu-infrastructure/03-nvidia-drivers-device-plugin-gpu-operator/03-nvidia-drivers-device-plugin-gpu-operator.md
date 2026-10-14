# 4.3 — NVIDIA Drivers, Device Plugin and GPU Operator

**Part:** 04 — GPU Infrastructure  
**Audience:** TrueFoundry platform engineering, Kubernetes SRE, ML platform operations  
**Safety:** Production examples are **[SAFE-READ]**. Changes to GPU drivers, operators, device plugins or runtimes are **[CHANGE-CONTROL]** and belong only in an approved maintenance plan.

## 1. Objectives

Understand the host-to-container GPU software chain; distinguish NVIDIA driver, container toolkit, device plugin and GPU Operator responsibilities; diagnose missing `nvidia.com/gpu` capacity; and plan safe GPU infrastructure recovery without disrupting unrelated tenants.

## 2. Architecture and failure boundaries

```text
GPU-equipped node / compatible OS kernel
      ↓
NVIDIA host driver (kernel modules + user-space libraries)
      ↓
NVIDIA container toolkit / compatible runtime integration
      ↓
NVIDIA Kubernetes device plugin → kubelet device registration
      ↓
Node capacity / allocatable: nvidia.com/gpu
      ↓
Kubernetes scheduler → GPU Pod assigned to node
      ↓
Kubelet device allocation + container creation
      ↓
CUDA framework (PyTorch / vLLM) → GPU execution
```

**GPU Operator** can manage multiple parts of this chain through its components and operands, including driver, toolkit, device plugin, GPU Feature Discovery, DCGM exporter and other optional features. **It is not mandatory in every cluster**: managed GPU node images or separately managed driver/plugin stacks are also valid. Installing the operator over an existing managed stack without reviewing component ownership risks conflicts.

**TrueFoundry boundary:** TrueFoundry configures and orchestrates workloads, but successful platform reconciliation is not proof of correct GPU host-driver registration, container device exposure or framework compatibility.

## 3. Component responsibilities

| Component | Primary job | Key evidence | Typical failure |
|---|---|---|---|
| GPU hardware | Physical accelerator | Cloud SKU, PCI inventory, model | Missing/unhealthy GPU |
| NVIDIA driver | OS/kernel ↔ GPU communication | Driver module, `nvidia-smi`, node logs | Driver absent, module mismatch |
| NVIDIA container toolkit | Integrates NVIDIA devices/libraries into containers | Runtime configuration, container startup events | GPU works on host but not in container |
| NVIDIA device plugin | Registers allocatable GPU resources with kubelet | DaemonSet Pod/logs, node `nvidia.com/gpu` | GPU absent from node resource inventory |
| GPU Operator | Reconciles configured GPU software components | ClusterPolicy and operand status where deployed | Operand failures, conflicting ownership |
| GPU Feature Discovery | Publishes GPU-related node labels where configured | Node labels and discovery Pod logs | Missing or stale feature labels |
| DCGM exporter | Exposes supported GPU telemetry | Exporter Pod/metrics | Missing metrics, not necessarily GPU outage |
| CUDA libraries/framework | Application-side GPU computation | PyTorch CUDA checks, application logs | CUDA runtime/device detection failure |

The exact operands, custom resources, labels and namespaces depend on GPU Operator version and installation configuration.

## 4. Driver versus CUDA compatibility

A host NVIDIA driver must support the CUDA runtime and framework stack used by the container. The `CUDA Version` shown in `nvidia-smi` reflects the maximum CUDA level supported by the installed driver, **not** the CUDA toolkit necessarily installed in the container.

Do not assume a driver update is needed just because a vLLM workload says `No CUDA runtime is found`. First prove:
1. GPU hardware is present and the node is Ready.
2. The host driver communicates with the GPU.
3. The device plugin advertises the GPU resource.
4. The Pod requests and receives a GPU.
5. The container runtime exposes GPU devices and libraries.
6. PyTorch/vLLM initializes CUDA with compatible builds.

Compatibility must be validated against the **exact** NVIDIA driver branch, CUDA runtime, GPU compute capability, PyTorch build, vLLM version, Kubernetes distribution and OS kernel. Use vendor matrices before a production change.

## 5. GPU Operator deployment modes and ownership

Common patterns:

- **Operator-managed driver and toolkit:** operator controls driver/toolkit operands. Requires careful OS, secure-boot, kernel and cluster compatibility review.
- **Preinstalled/managed driver:** cloud GPU node image or image-baking pipeline owns driver; operator driver management must be configured accordingly.
- **Standalone device plugin:** driver/toolkit provisioned separately; plugin DaemonSet registers GPUs.
- **Managed cloud integration:** EKS, AKS and GKE GPU provisioning vary by node image, release and configuration; never assume operator is installed or that all components are managed automatically.

**Rule:** Only one authoritative installation path should own each host component. Avoid overlapping driver DaemonSets, competing device plugins or simultaneous unmanaged and operator-controlled changes.

## 6. Discovery and evidence collection [SAFE-READ]

### 6.1 Confirm target and GPU resource inventory

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}'
```

A blank GPU resource field is a **registration symptom**, not conclusive proof of missing physical GPU. Record node Ready state, allocatable resources, taints and timestamp.

### 6.2 Discover installed GPU components

```bash
kubectl get daemonsets -A
kubectl get deployments -A
kubectl get pods -A -o wide
kubectl get crd | grep -i nvidia
```

If a GPU Operator is installed, discover its actual namespace and available custom resources:

```bash
kubectl api-resources | grep -i -E 'clusterpolic|nvidia'
kubectl get clusterpolicies.nvidia.com
```

The ClusterPolicy query applies only if that resource exists; absence is not necessarily a fault in a standalone plugin installation.

### 6.3 Inspect component health and events

```bash
kubectl -n <gpu-system-namespace> get pods -o wide
kubectl -n <gpu-system-namespace> describe pod <gpu-component-pod>
kubectl -n <gpu-system-namespace> logs <gpu-component-pod> --tail=200
kubectl -n <gpu-system-namespace> get events --sort-by=.lastTimestamp
kubectl describe node <gpu-node>
```

Identify the plugin/operator Pods by their actual labels and owners. When the Pod has multiple containers, use `-c <container>` to select the intended one. Avoid dumping Secrets or verbose environment variables.

### 6.4 Inspect host driver where authorized

On an approved GPU node shell or existing diagnostics environment:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,memory.used --format=csv
lsmod | grep nvidia
cat /proc/driver/nvidia/version
```

These host commands require suitable node access; they are not `kubectl` commands. If the driver is absent or the host is inaccessible, capture that limitation and escalate to the node image/driver owner. Do not launch privileged debugging Pods in production without authorization.

## 7. Fault isolation matrix

| Symptom | First checks | Probable failure boundary |
|---|---|---|
| GPU instance exists, `nvidia-smi` fails on host | Driver module, OS/kernel compatibility, device health | Driver / node image |
| Host GPU works, no `nvidia.com/gpu` | Plugin Pods/logs, kubelet registration, operator operands | Device plugin / kubelet |
| GPU resource exists, Pod Pending | Requests, allocatable vs allocated, selectors, taints | Scheduling / capacity |
| Pod bound, container cannot access GPU | GPU limits, runtime/toolkit, device allocation errors | Container integration |
| Container `nvidia-smi` works, PyTorch CUDA false | PyTorch build, runtime libraries, supported driver | Framework compatibility |
| Operator healthy, one node lacks GPUs | Node-specific driver/plugin health, taints, OS image | Node-level drift |
| Operator component CrashLoopBackOff | Component logs, image pull, permissions, compatibility | Operator/operand |
| Metrics missing, model inference healthy | Exporter/service/metrics pipeline | Observability, not necessarily GPU failure |

Use the **Lowest Proven Healthy Layer** and **First Failed Transition** to assign the current actionable owner.

## 8. Production incident runbook [SAFE-READ]

1. Confirm cluster context, node pool, affected workloads, time window and blast radius.
2. Preserve Kubernetes Pod events and node GPU capacity/allocatable.
3. Determine whether GPU Operator, standalone plugin or managed cloud node images own the stack.
4. Inspect GPU operator/device-plugin/driver operand state and logs.
5. Where authorized, compare host `nvidia-smi` with container visibility and framework initialization.
6. Correlate failures with recent node-image, kernel, driver, operator, plugin or workload-image changes.
7. Classify the first failed transition and current actionable owner.
8. Write a change plan with version compatibility evidence, canary node/pool, rollback and workload-drain strategy.
9. Implement only after change approval; avoid restarting shared GPU components as a diagnostic experiment.
10. Validate resource registration, scheduling, container GPU access, model readiness and successful inference.

### Evidence template

```text
Incident / timestamp / severity:
Cluster / namespace / TrueFoundry deployment:
GPU node pool and affected nodes:
GPU model / node OS image / kernel:
GPU stack ownership model:
Driver version / host nvidia-smi:
GPU Operator / ClusterPolicy status (if present):
Device plugin DaemonSet and logs:
Node GPU capacity / allocatable:
Pod requested GPUs / assigned node:
Container GPU visibility:
Framework CUDA result:
Lowest Proven Healthy Layer:
First Failed Transition:
Blast radius and current actionable owner:
Approved remediation and rollback:
Post-change inference and metrics evidence:
```

## 9. Safe maintenance, rollback and upgrades [CHANGE-CONTROL]

**Prechange:** record versions, source-of-truth configuration, node image, operator values, GPU resource advertisement, existing workloads and available capacity in another failure domain. Verify driver/CUDA/Kubernetes compatibility and review node-drain/disruption budgets.

**Canary:** prefer a dedicated nonproduction GPU node or canary pool. Do not perform in-place driver restarts on nodes serving active tenants without approved downtime or evacuation. Upgrade sequencing is environment-specific; follow the vendor's supported upgrade path.

**Rollback:** revert the authoritative Helm/IaC/node-image change to a known-compatible version; recover or replace the canary node using the supported provider workflow; verify driver and plugin registration before returning workloads. Driver rollback can require reboot/node replacement. Never promise zero downtime.

**Acceptance:** node Ready, expected `nvidia.com/gpu` allocatable, operator/plugin healthy, disposable GPU smoke test passes, model CUDA initializes, inference succeeds and baseline alerts remain healthy.

## 10. Observability and proactive controls

Alert on unexpected loss of advertised GPUs, unhealthy GPU driver/plugin/operator operands, node NotReady, GPU allocation failures, CUDA initialization errors and GPU hardware errors. Use DCGM exporter where available and supported. GPU resource inventory, Pod scheduling and application inference are separate signals; combine them to avoid false conclusions.

## 11. Common mistakes

- Treating `nvidia-smi` CUDA version as the installed container CUDA toolkit.
- Installing GPU Operator without checking whether cloud images already manage drivers.
- Assuming every node has the same driver/kernel/image after partial rollouts.
- Deleting device-plugin Pods or rebooting GPU nodes as an unapproved first response.
- Treating missing GPU metrics as proof the GPU is unavailable.
- Treating TrueFoundry reconciliation success as proof CUDA is healthy.

## 12. Acceptance checklist

- [ ] Distinguish driver, toolkit, device plugin, operator and CUDA responsibilities.
- [ ] Identify GPU stack ownership and component health without mutating production.
- [ ] Explain why host GPU visibility differs from Kubernetes advertising and container visibility.
- [ ] Collect evidence for missing `nvidia.com/gpu` and GPU device allocation failures.
- [ ] Propose canary, compatibility, rollback and end-to-end recovery checks.
- [ ] Avoid claiming vendor validation without an actual compatibility review.

**Lab:** [4.3 — NVIDIA GPU Stack Validation](../labs/03-nvidia-drivers-device-plugin-gpu-operator-check.md)  
**Next:** Chapter 4.4 — CUDA, Container Runtime and Compatibility.

## References

- [NVIDIA GPU Operator documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/)
- [NVIDIA Kubernetes device plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/)
- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [NVIDIA System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [Kubernetes GPU scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [TrueFoundry documentation](https://docs.truefoundry.com/)
