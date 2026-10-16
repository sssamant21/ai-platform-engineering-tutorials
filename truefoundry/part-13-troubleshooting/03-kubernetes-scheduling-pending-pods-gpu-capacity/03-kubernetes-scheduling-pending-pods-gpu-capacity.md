# Chapter 13.3 — Kubernetes Scheduling, Pending Pods and GPU Capacity Failures

**Part 13 — Troubleshooting | TrueFoundry SRE canonical tutorial**

## Objectives

Diagnose Pending inference Pods and GPU capacity failures across Kubernetes scheduling, extended resources, taints, affinity, topology spread, ResourceQuota, autoscalers, cloud quotas and scarce GPU SKUs. Restore service without violating availability, security, SLO or cost guardrails.

**[SAFE-READ]** inspect approved Kubernetes and cloud state. **[LAB-MUTATE]** bounded synthetic staging. **[CHANGE-CONTROL]** production GPU pool sizes, quotas, scheduling rules, node drain, priorities, reservations or TrueFoundry deployment spec. Never remove PDB/affinity protections or raise cost limits merely to force a Pod to schedule.

## 1. Scheduling path

```text
TrueFoundry desired deployment
     │
     ▼
Pod created → scheduler filters nodes
     ├─ GPU extended resource nvidia.com/gpu
     ├─ CPU/memory requests and allocatable
     ├─ taints/tolerations, selectors and affinity
     ├─ topology spread and volume constraints
     └─ namespace quotas and admission policy
     │
     ├─ fit → Bind node → kubelet/device plugin → model starts
     └─ no fit → Pending / FailedScheduling Event
                   │
              node autoscaler
                   ├─ matching node group + cloud quota → provision
                   └─ max limits, supply, subnet/IAM/launch failure
```

A Pending Pod may be **unscheduled**, or assigned to a node but blocked by image pulls, volumes or container setup. Always check `spec.nodeName`, phase, conditions and Events before blaming the scheduler.

## 2. Read-only triage

```bash
kubectl config current-context
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o wide
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get resourcequota,limitrange -n MODEL_NAMESPACE
kubectl get priorityclass
kubectl get pdb -n MODEL_NAMESPACE
```

Read `FailedScheduling` messages precisely: `Insufficient nvidia.com/gpu`, `Insufficient cpu`, taints, selector mismatch, topology constraints or volume affinity require different remediations. Events may expire; capture timestamps promptly.

## 3. GPU extended resources and allocatable capacity

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
kubectl describe node NODE_NAME
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o yaml
```

Review `nvidia.com/gpu` requests/limits, device plugin health, MIG resource names (when configured), node labels, existing Pod GPU allocations and CPU/memory. `kubectl top nodes` reports CPU/memory, **not** GPU utilization. GPU usage requires validated DCGM/vendor telemetry; Kubernetes scheduler allocates declared resources, not based on instantaneous GPU utilization.

For standard extended GPU resources, limits imply requests if requests are omitted, and a GPU request without an equal limit is not a valid configuration in ordinary Kubernetes GPU allocation. Verify the actual runtime and GPU-sharing mechanism before assuming fractional GPU support.

## 4. Taints, selectors, affinity and spread

```bash
kubectl describe node NODE_NAME
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o jsonpath='{.spec.nodeSelector}{"\n"}'
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o jsonpath='{.spec.affinity}{"\n"}'
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o jsonpath='{.spec.tolerations}{"\n"}'
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o jsonpath='{.spec.topologySpreadConstraints}{"\n"}'
```

Differentiate hard `requiredDuringSchedulingIgnoredDuringExecution` affinity from soft preferred rules. `DoNotSchedule` topology spread can block placement when a zone lacks eligible GPUs; relaxing it may reduce HA and needs approval. GPU node taints should normally be matched by explicit tolerations only for authorized workloads.

## 5. Admission, quotas and priority

Namespace `ResourceQuota` and `LimitRange` can reject Pod creation **before** scheduling, so the workload may have no Pending Pod. Inspect Deployment/ReplicaSet Events and admission errors. Evaluate priority/preemption carefully: a high-priority GPU Pod may displace lower-priority services and cause a different outage. PDBs constrain voluntary evictions; they do not guarantee that preemption cannot occur.

## 6. Autoscaler and cloud supply

Trace Pending Pod → matching GPU node group/provisioner → scaling decision → cloud API → node registration → device plugin advertisement → readiness. Check:
- min/max nodes, GPU instance type, architecture, AZ, taints and labels
- account/region GPU quotas and instance availability
- IAM, launch template, subnets, IP exhaustion and image bootstrap
- Karpenter/Cluster Autoscaler version-specific logs and events
- scale-up latency, model artifact warmup and SLO headroom

Avoid assuming a specific autoscaler. A cloud instance running without `nvidia.com/gpu` allocatable is a device-plugin/driver/bootstrap issue, not a lack of physical GPU supply.

## 7. SLO-constrained capacity

Compute eligible peak traffic and measured inference throughput per ready replica. Reserve capacity for the largest failure domain (N-1) and model cold start. A node count that fits all Pods under healthy conditions can still be insufficient after a zone loss. Keep spend caps and cloud quota separate from Kubernetes namespace quota.

## 8. Safe remediation sequence

1. Capture affected scope, UTC timeline, Events and change history.
2. Confirm Pod created, scheduled or unscheduled.
3. Classify scheduler constraint or cloud provision failure.
4. Check matching healthy node capacity and device-plugin resources.
5. Check autoscaler and cloud quota, SKU/AZ supply.
6. Propose lowest-risk fix through TrueFoundry desired state or approved IaC.
7. Validate replica spread, PDB, warm reserve, budget and SLO.
8. Canary, monitor and roll back if inference degrades.

Do not delete healthy GPU Pods as a first-line fix; doing so may force expensive model reloads and interrupt streaming requests.

## 9. Fourteen runbooks

1. Pending Pod classification.
2. FailedScheduling: insufficient GPU.
3. CPU/memory pressure on GPU nodes.
4. Device plugin missing GPU allocatable.
5. MIG profile/resource mismatch.
6. GPU taint/toleration mismatch.
7. Node selector/affinity mismatch.
8. Topology spread deadlock.
9. PVC zone affinity conflict.
10. ResourceQuota/admission rejection.
11. Autoscaler does not scale.
12. Cloud GPU quota/SKU shortage.
13. Node bootstraps but remains NotReady.
14. N-1 GPU capacity and cost-safe recovery.

## 10. Thirty-six failure scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | Insufficient GPU | 19 | Autoscaler max reached |
| 02 | Insufficient CPU | 20 | Cloud quota exhausted |
| 03 | Insufficient memory | 21 | GPU SKU unavailable |
| 04 | GPU request/limit invalid | 22 | AZ capacity unavailable |
| 05 | Device plugin unhealthy | 23 | Subnet IP exhausted |
| 06 | NVIDIA driver missing | 24 | Node bootstrap failed |
| 07 | MIG resource mismatch | 25 | Node NotReady |
| 08 | GPU sharing misconfigured | 26 | Kubelet registration failure |
| 09 | Taint not tolerated | 27 | Autoscaler permissions denied |
| 10 | Node selector mismatch | 28 | Node group labels wrong |
| 11 | Required node affinity | 29 | Node group taints wrong |
| 12 | Required Pod affinity | 30 | Provisioning too slow |
| 13 | Pod anti-affinity | 31 | Model cold start exceeds SLO |
| 14 | Topology spread blocked | 32 | PDB/maintenance conflict |
| 15 | PVC zone affinity | 33 | Preemption harms tenant |
| 16 | Namespace ResourceQuota | 34 | Spot interruption |
| 17 | LimitRange/admission denial | 35 | N-1 capacity insufficient |
| 18 | PriorityClass conflict | 36 | GPU cost cap exceeded |

## 11. Twelve acceptance gates

G1 approved scope; G2 Pod scheduling phase correctly classified; G3 scheduler Events preserved; G4 GPU allocatable and Pod requests reconciled; G5 affinity/taints/spread assessed; G6 quota/admission assessed; G7 autoscaler decision traced; G8 cloud quota/SKU checked; G9 N-1 reserve validated; G10 safe rollback defined; G11 SLO and GPU costs verified; G12 reviewer sign-off.

Complete the [hands-on lab](../labs/03-kubernetes-scheduling-pending-pods-gpu-capacity-check.md). Formal review and live execution pending.

**Next:** Chapter 13.4 — Deployment Failures, CrashLoopBackOff and ImagePullBackOff.
