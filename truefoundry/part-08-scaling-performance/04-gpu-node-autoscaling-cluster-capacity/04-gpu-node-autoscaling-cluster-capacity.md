# 8.4 — GPU Node Autoscaling and Cluster Capacity

**Part 08 — Scaling & Performance | Chapter 4 of 7**  
**Audience:** TrueFoundry platform engineers, GPU infrastructure SREs, Kubernetes EKS/AKS operators  
**Execution labels:** **[SAFE-READ]** observation; **[LAB-MUTATE]** isolated approved staging; **[CHANGE-CONTROL]** production changes.

## 1. Learning objectives

Design the node-scaling layer beneath inference HPA/KEDA; compare Cluster Autoscaler and Karpenter; plan EKS/AKS GPU node pools; validate NVIDIA drivers/device plugins, GPU allocatable and scheduler constraints; estimate usable capacity and cold-start delays; protect workloads during consolidation and interruption; troubleshoot and recover without destabilizing shared clusters.

**Prerequisites:** Chapters 8.1–8.3, a documented TrueFoundry workspace/cluster, approved GPU node pool, Kubernetes read access, measured inference SLO and model warmup time, quota visibility and an immutable known-good release. **Never assume the cluster uses Karpenter, Cluster Autoscaler or a particular GPU operator.**

## 2. Scaling chain and responsibility boundaries

```text
Client demand → TrueFoundry/vLLM → HPA/KEDA desired replicas
                                      │
                              Pending GPU Pods
                                      │
                  Kubernetes scheduler: fit constraints
                                      │
                  Node provisioner (CA/Karpenter/other)
                                      │
               Cloud instance availability, IAM, quotas
                                      │
                GPU node boot + CNI + kubelet + runtime
                                      │
                  NVIDIA driver / device plugin
                                      │
               nvidia.com/gpu allocatable / placement
                                      │
                 Image + model pull / CUDA warmup
                                      │
                   Ready EndpointSlice → capacity
```

**HPA/KEDA increases desired Pods, not GPUs.** Node autoscalers add nodes when scheduling constraints and provisioner configuration permit. A running GPU node is not serving capacity until the device is allocatable and the inference Pod is Ready.

## 3. Cluster Autoscaler vs Karpenter

| Dimension | Cluster Autoscaler (CA) | Karpenter (where supported) |
|---|---|---|
| Primary abstraction | Existing node groups / autoscaling groups | Provisioning constraints, NodePools/NodeClasses (version dependent) |
| Scale-up trigger | Unschedulable Pods against node group templates | Unschedulable Pods and eligible provisioning rules |
| Instance choice | Configured node group options | Eligible instance types from provisioner requirements |
| Consolidation | Node group scale-down logic | Consolidation/disruption policies |
| Cloud support | Many managed Kubernetes platforms | Karpenter support is provider/version specific; AWS provider common |
| Operational risk | Wrong group template, max size, ASG IAM | Wrong NodePool requirements, limits, disruption budgets |

Do not run multiple independent node autoscalers against the same node group/provisioning ownership without a documented coexistence design. On AKS, use the supported AKS node-pool autoscaler and verify any separate Karpenter integration against the deployed provider version. **Karpenter is not a generic drop-in AKS configuration.**

## 4. EKS GPU capacity architecture

Typical EKS options: managed/self-managed GPU node groups with CA, or Karpenter provisioned GPU EC2 instances with an AWS-specific NodeClass/NodePool. Confirm AWS region/AZ instance supply, EC2 quotas, subnets/IP capacity, launch template/AMI, node IAM role, ECR/S3 access, EBS bandwidth and supported GPU driver/runtime stack.

For Karpenter, inspect installed CRDs and versions before authoring NodePool or EC2NodeClass resources; schemas and disruption settings change across releases. NodePool CPU/memory limits are not the same as guaranteed GPU quota or cloud instance availability.

## 5. AKS GPU capacity architecture

Use approved AKS GPU node pools, cluster autoscaler profile and node-pool min/max, supported VM GPU SKUs, region quota, VNet/subnet IP headroom, node OS image and NVIDIA driver/device plugin provisioning strategy. Confirm supported Azure GPU VM/driver/AKS versions. Azure VM quota, regional supply and network IP exhaustion can all prevent scale-up even when Kubernetes wants more nodes.

## 6. NVIDIA GPU resource lifecycle

1. Cloud VM presents physical GPU(s).
2. Host driver and compatible container runtime expose devices.
3. NVIDIA device plugin advertises `nvidia.com/gpu` or configured MIG resource names.
4. Kubernetes scheduler places Pods requesting integer extended resources.
5. CUDA/PyTorch/vLLM initialize GPU and model.
6. Startup/readiness gates mark endpoint usable.

GPU count is not equal to free VRAM. For MIG/time-slicing, resource accounting, isolation and scheduling differ; do not infer exclusive GPU ownership from a resource name without verifying policy.

### Generic Pod GPU resource fragment (illustrative)

```yaml
resources:
  requests:
    cpu: "4"
    memory: 24Gi
    nvidia.com/gpu: 1
  limits:
    memory: 32Gi
    nvidia.com/gpu: 1
```

CPU/memory values are examples, **not production sizing**. Check GPU resource request/limit semantics and namespace quota. A Pod requesting 2 GPUs cannot be placed on two separate one-GPU nodes; all requested devices must be allocated to the Pod on one node.

## 7. Placement constraints and topology

Evaluate `nodeSelector`, required/preferred node affinity, taints/tolerations, topology spread, PVC zone binding, GPU SKU compatibility, node architecture, device-plugin labels and workload isolation. GPU nodes may be intentionally tainted; a missing toleration can make an otherwise idle GPU unavailable. **Required** topology rules can block placement when no eligible zones remain.

For tensor-parallel Pods, plan GPUs **per Pod** and NCCL/topology compatibility. Multiple Pods on different nodes are not equivalent to one multi-GPU Pod.

## 8. Capacity math and failure reserve

```text
required_serving_replicas =
  ceil(peak_accepted_tokens_per_second / sustainable_tokens_per_second_per_replica)

required_GPUs = required_serving_replicas × GPUs_per_Pod
```

**Hypothetical example:** 1,100 output tokens/s demand and 280 sustainable output tokens/s per one-GPU replica at the accepted TTFT/ITL SLO requires ceil(1100/280) = **4** serving replicas. With one replica reserved for failure, plan **5 allocatable GPUs** before considering rolling-update surge. If two replicas share a node, loss of that node may require **two** spare replicas rather than one. Verify N-1 for the *largest failure domain*, not merely one Pod.

Cloud quotas, node-pool limits, DaemonSet overhead, Pod density, CPU/RAM and subnet IP capacity can reduce usable capacity. A 4-GPU node does not guarantee four independent model replicas will fit.

## 9. Time to usable capacity

```text
Pod Pending → scale trigger → cloud capacity allocation
→ VM boot → kubelet/CNI → driver/device plugin → allocatable GPU
→ Pod schedule → image pull → artifact download
→ CUDA/model load → warmup → model-aware readiness
→ successful synthetic inference
```

Measure timestamps for every stage. Cold-start time can dominate autoscaling responsiveness. A **warm minimum capacity**, bounded admission queue, load shedding, pre-pulled immutable artifacts where appropriate, and tested scale-up policy may be necessary for synchronous low-TTFT services.

## 10. Read-only diagnostics [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L node.kubernetes.io/instance-type
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get deployments,hpa,pdb,resourcequota
kubectl -n <namespace> describe pod <pending-pod>
kubectl describe node <gpu-node>
kubectl -n <namespace> get endpointslices -o wide
```

If Karpenter is installed, inspect supported CRDs first:

```bash
kubectl get crd nodepools.karpenter.sh
kubectl get nodepools.karpenter.sh
kubectl get nodeclaims.karpenter.sh
```

If absent, do not assume an outage. For CA, discover its actual deployment namespace and logs using approved cluster inventory; do not hardcode its installation. Cloud CLI commands require authenticated read-only accounts and the actual provider profile/subscription.

## 11. GPU health validation

On an existing authorized GPU workload:

```bash
kubectl -n <namespace> exec <pod> -c <container> -- nvidia-smi
kubectl -n <namespace> exec <pod> -c <container> -- python -c 'import torch; print(torch.cuda.is_available(), torch.cuda.device_count())'
```

If minimal images lack `nvidia-smi`, use approved driver/device-plugin and DCGM telemetry. Missing a diagnostic utility is **Unknown**, not proof of missing GPU. Check driver/device-plugin health before changing application code for `No CUDA runtime` or `Failed to infer device type`.

## 12. Scale-down, disruption and consolidation

Node autoscaler consolidation or interruption can terminate long-running streams unless drain, graceful termination and readiness removal are designed. Verify:

- Node provisioner disruption/consolidation policy and budgets.
- Kubernetes PDB and `terminationGracePeriodSeconds`.
- Gateway streaming drain and retry behavior.
- Model load/warmup cost for replacement nodes.
- Zone diversity, spare GPU capacity and critical workload priority.
- Spot/preemptible node interruption handling and on-demand fallback.
- Authoritative GitOps/TrueFoundry settings before adjusting scale.

PDBs constrain certain voluntary evictions but do not guarantee protection from node failure, cloud interruption or every provisioning action.

## 13. Cost and security guardrails

Use GPU-hours, warm idle capacity, model-download/storage cost and **cost per million completed output tokens at SLO**. Keep a max node/replica budget, quota alarms, idle-node cleanup rules and fault-domain reserve. Enforce node IAM/managed identity least privilege, encrypted artifacts, trusted AMIs/images, workload identity, node isolation and controlled outbound model registry access. Do not put model credentials into user data or node bootstrap scripts.

## 14. Failure scenarios and first failed transition

| ID | Symptom | First failed transition | Evidence |
|---|---|---|---|
| G1 | Pod Pending: insufficient GPU | Scheduler → allocatable | Pod events |
| G2 | Node group max size reached | Provisioner → pool limit | Autoscaler status |
| G3 | EC2/Azure quota exceeded | Provisioner → cloud | Cloud quota error |
| G4 | GPU SKU out of stock | Cloud → VM launch | Allocation failure |
| G5 | Subnet IP exhaustion | Cloud/CNI → node/Pod | IP events |
| G6 | IAM/identity permission denied | Provisioner → cloud API | 403/AccessDenied |
| G7 | Karpenter NodePool requirements exclude SKU | NodePool → instance fit | Requirements |
| G8 | CA node template mismatch | CA → simulated scheduling | CA logs |
| G9 | GPU node taint not tolerated | Scheduler → node | Pod events |
| G10 | Required affinity too strict | Scheduler → node | Affinity |
| G11 | Topology spread unsatisfiable | Scheduler → zone | Constraints |
| G12 | PVC zone mismatch | Storage → Pod placement | Volume events |
| G13 | NVIDIA driver absent | Node → GPU runtime | Driver inventory |
| G14 | Device plugin unhealthy | Plugin → allocatable GPU | Node describe |
| G15 | CUDA/PyTorch incompatible | GPU runtime → engine | CUDA check |
| G16 | Model cold-start timeout | Pod → readiness | Startup timeline |
| G17 | Scale-up model download saturates storage | Registry → Pod | Download latency |
| G18 | Consolidation interrupts streaming | Node drain → client | Stream failures |
| G19 | Spot interruption exceeds reserve | Node loss → SLO | Capacity timeline |
| G20 | Unexpected GPU cost explosion | Autoscaler → spend | GPU-hours |
| G21 | Multi-GPU Pod cannot fit | Scheduler → topology | GPU per node |
| G22 | MIG resource mismatch | Plugin → Pod request | Resource names |
| G23 | Node becomes Ready but GPU unavailable | Kubelet → plugin | Allocatable |
| G24 | Rollout surge deadlock | Deployment → capacity | Pending + maxSurge |

Record **LPHL, FFT, impact, mitigation, rollback, owner and recovery evidence**. Do not inject faults into shared node pools.

## 15. Production incident runbook

1. **Declare:** endpoint, TTFT/ITL, errors, queue, desired/Ready replicas, Pending GPU Pods and onset.
2. **Trace:** HPA/KEDA → Deployment → scheduler → node autoscaler → cloud quota/supply → node runtime → GPU allocatable → model Ready.
3. **Preserve:** Pod/node events, provisioner decisions, immutable model/image, cloud errors, GPU telemetry and timestamps.
4. **Contain:** retain known-good warm replicas, cap admission/retries with jitter, pause unsafe rollout/consolidation where approved.
5. **Recover:** correct confirmed quota/identity/placement/plugin issue or restore authoritative TrueFoundry/GitOps configuration.
6. **Validate:** actual GPU allocatable, Ready EndpointSlices, correct model, queue drain, TTFT/ITL, stream completion, N-1 reserve and cost ceiling.
7. **Review:** LPHL/FFT, time-to-usable-capacity, missed alarms, ownership and preventive changes.

Do not reboot shared GPU nodes, replace fleet-wide drivers or modify production node pools as an unapproved diagnostic step.

## 16. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | Node autoscaler ownership/provider/version verified |
| A2 | GPU node-pool quota, SKU, IAM and subnet capacity checked |
| A3 | Driver/device plugin and allocatable GPU validated |
| A4 | Placement and multi-GPU topology reviewed |
| A5 | N-1 reserve, surge and cost ceiling calculated |
| A6 | Time-to-usable-capacity measured or Not run |
| A7 | G1–G24 classified with LPHL/FFT |
| A8 | Drain, rollback and SLO recovery documented |

## 17. Evidence template

```text
Date/time / change / owner:
TrueFoundry workspace / EKS or AKS / region / namespace:
CA/Karpenter/AKS autoscaler and version:
GPU node group/NodePool, SKU, quota and max size:
Node IAM/identity, subnet IP headroom, driver/plugin:
GPU allocatable vs requests, taints, affinity and topology:
Peak token demand, sustainable throughput, GPU per Pod:
Steady replicas, largest-domain N-1 reserve and surge:
Node launch, GPU allocatable, model Ready and inference times:
Pending Pod reason, LPHL, FFT, cloud/provisioner error:
TTFT/ITL, queue, stream completion, GPU-hours and cost:
Rollback release, cleanup, gate results and owner signoff:
```

**Hands-on lab:** [8.4 — GPU Node Autoscaling and Cluster Capacity Check](../labs/04-gpu-node-autoscaling-cluster-capacity-check.md)  
**Next:** Chapter 8.5 — Inference Concurrency, Latency and Throughput Optimization.

## References

- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
- [Karpenter Documentation](https://karpenter.sh/docs/)
- [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Azure AKS Cluster Autoscaler](https://learn.microsoft.com/azure/aks/cluster-autoscaler)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
