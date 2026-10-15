# Lab 8.4 — GPU Node Autoscaling and Cluster Capacity Check

**Canonical:** [Chapter 8.4](../04-gpu-node-autoscaling-cluster-capacity/04-gpu-node-autoscaling-cluster-capacity.md)  
**Default:** [SAFE-READ]. Node-pool or workload changes require [LAB-MUTATE] in approved isolated staging. Production changes require [CHANGE-CONTROL].

## 1. Mission and safety

Verify GPU provisioning, Kubernetes scheduler fit, NVIDIA allocatable resources, cloud quota, cold-start time, N-1 reserve and node disruption safety. Do **not** reboot shared GPU nodes, install drivers cluster-wide, change shared node pools or generate unbounded load.

## 2. Approval record

```text
Date/time / owner / change approval:
TrueFoundry workspace, EKS/AKS cluster, namespace:
GPU node pool and autoscaler type/version:
GPU SKU, cloud quota, node max, subnet capacity:
Immutable model/image and known-good rollback:
Maximum test duration, node/GPU-hours cost:
TTFT/ITL/error SLO and stop thresholds:
Cleanup owner and evidence destination:
```

## 3. Read-only preflight

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L node.kubernetes.io/instance-type
kubectl -n <namespace> get deployments,pods,hpa -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get resourcequota,pdb
kubectl -n <namespace> get endpointslices -o wide
kubectl describe node <gpu-node>
kubectl -n <namespace> describe pod <gpu-pod>
```

Optional Karpenter inventory **only if installed**:

```bash
kubectl get crd nodepools.karpenter.sh
kubectl get nodepools.karpenter.sh
kubectl get nodeclaims.karpenter.sh
```

If Karpenter is absent, document the actual node-scaling mechanism rather than installing it.

## 4. Cloud/node-pool checklist

| Check | Expected | Observed | Evidence |
|---|---|---|---|
| EKS/AKS and autoscaler ownership | | | |
| GPU SKU and regional capacity | | | |
| Node group min/max / NodePool limits | | | |
| Cloud instance/GPU quota | | | |
| Subnet IP and CNI headroom | | | |
| Node IAM/managed identity | | | |
| Driver/runtime/device plugin | | | |
| GPU allocatable | | | |
| GPU Pod request and quota | | | |
| Taints/tolerations and affinity | | | |
| Zone/PVC topology | | | |
| Known-good rollback | | | |

## 5. GPU resource verification

For an authorized existing Pod only:

```bash
kubectl -n <namespace> exec <gpu-pod> -c <container> -- nvidia-smi
kubectl -n <namespace> exec <gpu-pod> -c <container> -- python -c 'import torch; print(torch.cuda.is_available(), torch.cuda.device_count())'
```

Missing utility is Unknown, not proof of absent GPU. Record GPU resource name, MIG/time-slicing mode if any, and actual device visibility.

## 6. Capacity worksheet

```text
replicas = ceil(peak accepted output tokens/s
                / sustainable output tokens/s per replica at SLO)
GPUs = replicas × GPUs per Pod
```

| Input | Assumption | Observed |
|---|---|---|
| Peak output tokens/s | | |
| Per-replica throughput at SLO | | |
| Required serving replicas | | |
| GPUs per replica/TP topology | | |
| Largest-node/failure-domain loss | | |
| N-1 reserve | | |
| Rolling-update surge | | |
| Node count and allocatable GPUs | | |
| CPU/RAM/IP and GPU fit | | |
| Cloud quota and max node count | | |

Explain why five GPUs on one node do not provide node-level HA.

## 7. Time-to-usable-capacity worksheet

| Stage | Time | Evidence |
|---|---|---|
| Pod unschedulable | | |
| Autoscaler decision | | |
| Cloud instance allocation | | |
| Node Ready | | |
| GPU allocatable | | |
| Pod scheduled | | |
| Image/model fetched | | |
| CUDA/model warmup | | |
| Endpoint Ready | | |
| Successful synthetic inference | | |

Do not infer times. Mark Not run where live provisioning is not approved.

## 8. Optional bounded staging scale-up [LAB-MUTATE]

1. Confirm lab-owned node pool, quota, max GPU spend, approved model and rollback.
2. Record baseline nodes, GPU allocatable, Ready replicas, queue and SLO.
3. Trigger a bounded **lab-owned** GPU Pod request that exceeds current free capacity but fits approved pool limits.
4. Observe scheduler event → autoscaler decision → cloud instance → node Ready → device plugin → Pod placement.
5. Verify model-aware readiness and synthetic inference without using sensitive prompts.
6. Stop for cloud quota error, prolonged Pending, unexpected cost, node instability or SLO breach.
7. Remove only lab-owned resources; verify graceful drain and return to baseline.
8. Capture recovery, cleanup and cost evidence.

If a separate pool cannot be isolated, conduct tabletop only.

## 9. Failure tabletop

| ID | Scenario | LPHL | FFT | Evidence | Safe response |
|---|---|---|---|---|---|
| G1 | Insufficient GPU | | | | |
| G2 | Pool max reached | | | | |
| G3 | Cloud quota exceeded | | | | |
| G4 | GPU SKU unavailable | | | | |
| G5 | Subnet IP exhausted | | | | |
| G6 | Cloud IAM denied | | | | |
| G7 | Karpenter requirements exclude SKU | | | | |
| G8 | CA template mismatch | | | | |
| G9 | Missing toleration | | | | |
| G10 | Strict affinity | | | | |
| G11 | Topology unsatisfiable | | | | |
| G12 | PVC zone mismatch | | | | |
| G13 | NVIDIA driver absent | | | | |
| G14 | Device plugin unhealthy | | | | |
| G15 | CUDA incompatible | | | | |
| G16 | Model cold-start timeout | | | | |
| G17 | Artifact download bottleneck | | | | |
| G18 | Consolidation breaks streams | | | | |
| G19 | Spot loss overwhelms reserve | | | | |
| G20 | GPU cost runaway | | | | |
| G21 | Multi-GPU Pod cannot fit | | | | |
| G22 | MIG resource mismatch | | | | |
| G23 | Node Ready, GPU missing | | | | |
| G24 | Surge rollout deadlock | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition. Mark tabletop vs live evidence accurately.

## 10. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Autoscaler/provider/version and ownership | | |
| A2 Quota/SKU/IAM/subnet validation | | |
| A3 NVIDIA driver/device plugin/allocatable | | |
| A4 Scheduling and multi-GPU topology | | |
| A5 N-1 reserve/surge/cost | | |
| A6 Provision-to-Ready timing | | |
| A7 G1–G24 failure classification | | |
| A8 Safe drain/rollback/SLO | | |

## 11. Final report

```text
Date/time / approval / owner:
Cluster/namespace and GPU node pool:
CA/Karpenter/AKS autoscaler and version:
Cloud quota, instance supply, subnet and identity:
GPU allocatable, Pod request, affinity/taints:
Driver/plugin and CUDA/PyTorch validation:
Replicas, GPUs, N-1 reserve and surge:
Node provisioning and model time-to-Ready:
TTFT/ITL, errors, stream completion, GPU-hours:
G1–G24 LPHL/FFT and A1–A8 results:
Rollback, cleanup, outstanding risks:
```

**Next:** Chapter 8.5 — Inference Concurrency, Latency and Throughput Optimization.
