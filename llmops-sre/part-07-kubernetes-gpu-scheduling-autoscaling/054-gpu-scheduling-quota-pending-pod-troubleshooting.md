# Chapter 054 — GPU Scheduling, Quota and Pending Pod Troubleshooting

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 054 of 100**

> **Status:** Written tutorial published. Live lab execution, independent technical/security review and operational acceptance pending. Kubernetes scheduler messages, NVIDIA device-plugin resource names, admission policies and autoscaler behavior vary by version/provider. Production fixes require approval and an auditable rollback.

## Learning outcomes

Diagnose GPU workloads rejected by admission or stuck Pending; distinguish quota, resource, taint, affinity, topology, PVC, priority and autoscaler failures; trace device-plugin and GPU node readiness; restore service safely; build incident runbooks and measurable acceptance gates.

## 1. Incident architecture

```text
GitOps/CI → Kubernetes API
                │
         authn/RBAC/admission
                │
    ResourceQuota / LimitRange / policy
                │
          Pod persisted?
           ├─ no → Forbidden / rejected
           └─ yes → scheduler
                    ├─ GPU allocatable / requested
                    ├─ taints and tolerations
                    ├─ nodeSelector / affinity
                    ├─ topology spread
                    ├─ PV/PVC and volumes
                    ├─ priority / preemption
                    └─ provisionable node templates
                               │
                  CA / Karpenter / cloud capacity
                               │
                    GPU driver / device plugin
                               │
                    Pod scheduled / runtime
                               │
                image + model load + readiness
```

**First diagnostic split:** Did the API reject the workload, or did it create a Pod that cannot schedule? These have different owners and remediation paths.

## 2. Symptoms and first response

| Symptom | Likely layer | First evidence |
|---|---|---|
| API `Forbidden: exceeded quota` | admission | ResourceQuota used/hard |
| API `Forbidden` RBAC | authorization | role/access audit |
| Pod Pending, `FailedScheduling` | scheduler | Pod events |
| `Insufficient nvidia.com/gpu` | GPU resources | node allocatable/allocated |
| untolerated taint | placement | node taints/Pod tolerations |
| affinity/selector mismatch | placement | labels/required rules |
| volume binding error | storage | PVC/storage class |
| `NotTriggerScaleUp` or provisioner error | autoscaler | CA/Karpenter logs |
| Pod scheduled but `ContainerCreating` | runtime | image/CSI/CNI |
| Pod Running but model unready | model startup | model logs/readiness |

Avoid guessing from a single event: scheduler failures may list multiple simultaneous constraints.

## 3. Safe-read triage bundle

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl get pods -A -o wide
kubectl get resourcequota -A
kubectl get limitrange -A
kubectl get priorityclasses
kubectl get pvc -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n APPROVED_NS describe pod APPROVED_GPU_POD
kubectl describe node APPROVED_GPU_NODE
```

Scope discovery to authorized clusters and namespaces; sanitize Pod events and logs before sharing.

## 4. Admission failure versus Pending

An admission rejection can prevent Pod creation entirely. Check deployment/controller events, CI/GitOps apply errors and quota usage. A created Pending Pod requires scheduler events, actual resource requests and matching node inventory. Do not increase GPU quotas simply because a Pod is Pending.

## 5. GPU resource accounting

Kubernetes schedules using advertised GPU extended resources such as `nvidia.com/gpu` or MIG-specific names, not live utilization. Compare Capacity, Allocatable and already allocated requests for each candidate node. A physical GPU visible to `nvidia-smi` but absent from Allocatable points toward driver/device-plugin/kubelet registration.

## 6. Quota troubleshooting

```bash
kubectl -n APPROVED_NS get resourcequota,limitrange
kubectl -n APPROVED_NS describe resourcequota APPROVED_QUOTA
kubectl -n APPROVED_NS get events --sort-by=.lastTimestamp
```

Check `requests.nvidia.com/gpu` and the actual advertised resource key, CPU/memory, Pod count and any admission constraints. Extended resource request/limit semantics are Kubernetes-version dependent; verify rather than assuming defaults.

## 7. Taints and tolerations

A toleration allows placement on a tainted node; it does not select that node. Match key, operator, value and effect. Avoid broad `Exists` tolerations that bypass tenant isolation. `NoExecute` can evict existing workloads; treat production taint changes as disruptive.

## 8. Node selectors and affinity

`nodeSelector` and required node affinity must match actual node labels. Preferred affinity is advisory. Hard anti-affinity can block replicas when GPU node count is small. Compare Pod rules against eligible nodes, not the intended node-pool design alone.

## 9. Topology spread

Hard `DoNotSchedule` topology constraints can block rollout if required zones lack GPU capacity. Validate `topology.kubernetes.io/zone`, `maxSkew`, label selector, eligible domains and autoscaler templates. Do not relax zone resilience without documenting failure risk.

## 10. PVC and storage constraints

Some GPU model-serving Pods require PVCs for model cache. `WaitForFirstConsumer`, zone-bound PVs and storage topology can constrain placement. Investigate PVC binding and zone compatibility before blaming GPUs.

## 11. Priority and preemption

High-priority Pods may preempt lower-priority workloads, but preemption cannot fix impossible affinity, wrong GPU profile or unavailable cloud capacity. Audit PriorityClass, victim disruption and model stream safety.

## 12. Cluster Autoscaler and Karpenter

CA and Karpenter can provision only feasible node templates/SKUs with available cloud quota and zone capacity. Inspect controller ownership, node groups/NodePools, taints, labels, limits, NodeClaims and provisioning events. Avoid dual ownership of the same capacity domain.

## 13. GPU driver/device plugin

```bash
kubectl get daemonsets -A
kubectl get crd | grep -Ei 'nvidia|clusterpolic'
kubectl describe node APPROVED_GPU_NODE
```

Inspect discovered device-plugin Pods and logs for NVML failures, driver incompatibility and kubelet registration. Do not restart GPU Operator or driver DaemonSets fleet-wide as a first response.

## 14. MIG and GPU sharing

A Pod requesting `nvidia.com/gpu` may not match nodes advertising MIG-specific resources, depending on plugin strategy. Time-sliced advertised units are not equivalent to physical GPU count or guaranteed VRAM. Verify actual resource names, GPU SKU, MIG mode and tenant sharing policy.

## 15. Diagnostic decision tree

```text
GPU workload cannot start?
 ├─ API rejected → RBAC/quota/admission
 ├─ Pod Pending
 │    ├─ insufficient GPU → allocatable/allocated
 │    ├─ taint mismatch → tolerations
 │    ├─ label/affinity mismatch → placement
 │    ├─ topology conflict → zones
 │    ├─ PVC unbound → storage
 │    └─ autoscaler fails → SKU/quota/template
 ├─ Pod scheduled, not Running → CNI/CSI/image/runtime
 ├─ Running, no CUDA → driver/toolkit/CDI
 └─ Running, unready → model artifact/warmup/KV
```

## 16. Safe remediation order

1. Capture UTC, context, workload owner and incident scope.
2. Preserve Pod events, controller errors and resource accounting.
3. Identify one or more blocking constraints; verify independently.
4. Choose smallest reversible change in isolated staging first.
5. Obtain production approval for quota, placement, GPU node or driver changes.
6. Canary one workload/node pool; validate terminal-valid inference and TTFT/TPOT.
7. Monitor regression, cost and tenant isolation; restore GitOps source of truth.
8. Roll back immediately if error budget, cost or security thresholds are crossed.

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | GPU Pod rejected by ResourceQuota | API error |
| F02 | Pod count quota exhausted | used/hard |
| F03 | CPU/memory quota blocks GPU Pod | admission |
| F04 | LimitRange conflicts | admission |
| F05 | RBAC prevents creation | Forbidden |
| F06 | Pod Security rejects runtime | admission |
| F07 | wrong GPU resource key | Pod spec |
| F08 | GPU Allocatable zero | node status |
| F09 | device plugin crash loop | DaemonSet |
| F10 | driver/NVML failure | logs |
| F11 | GPU fully allocated | node describe |
| F12 | untolerated taint | events |
| F13 | selector label absent | labels |
| F14 | required affinity impossible | events |
| F15 | anti-affinity blocks replicas | events |
| F16 | topology spread deadlock | zone inventory |
| F17 | PVC cannot bind in GPU zone | PVC events |
| F18 | preemption cannot help | scheduler |
| F19 | CA node group template mismatch | CA logs |
| F20 | Karpenter NodeClaim fails | status |
| F21 | cloud GPU SKU/quota unavailable | cloud |
| F22 | MIG profile mismatch | allocatable |
| F23 | image/model warmup mistaken for Pending | Pod status |
| F24 | rollback leaves quota/taint drift | GitOps diff |

## 18. Sixteen operational runbooks

1. Capture incident and classify admission versus scheduling.
2. Recover GPU ResourceQuota denial.
3. Resolve Pod/CPU/memory/LimitRange constraints.
4. Restore RBAC and Pod Security compliance.
5. Verify GPU resource keys and per-node accounting.
6. Recover device-plugin registration and NVML.
7. Diagnose GPU allocation saturation.
8. Restore taint/toleration matching.
9. Repair node selectors and required affinity.
10. Resolve anti-affinity and topology deadlocks.
11. Recover PVC zone and binding constraints.
12. Analyze priority/preemption safely.
13. Restore CA node-group template scaling.
14. Recover Karpenter NodeClaim/cloud capacity.
15. Diagnose MIG resource and model warmup issues.
16. Execute full placement/quota rollback and RCA.

## 19. Eight tabletop game days

1. Critical inference Pod rejected by exhausted GPU quota.
2. Device plugin crashes and GPUs disappear from Allocatable.
3. GPU pool gains an untolerated taint.
4. Required affinity points to a nonexistent GPU label.
5. Hard topology spread blocks zone failover.
6. PVC binds to a zone without compatible GPUs.
7. Cloud GPU quota blocks CA/Karpenter scale-out.
8. Rollback restores Deployment but not quota/taints.

Default tabletop; live injection requires approved isolated staging.

## 20. Twenty acceptance gates

G01 scope/context; G02 admission versus Pending classification; G03 ResourceQuota; G04 CPU/memory/LimitRange; G05 RBAC/Pod Security; G06 GPU resource key; G07 allocatable/allocated; G08 driver/plugin; G09 taints/tolerations; G10 node selectors; G11 affinity; G12 topology; G13 PVC/zone; G14 priority; G15 CA/Karpenter; G16 cloud SKU/quota; G17 MIG/model readiness; G18 rollback; G19 SLO/FinOps; G20 independent review.

See [companion lab](../labs/054-gpu-scheduling-quota-pending-pod-troubleshooting-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 055 — GPU Autoscaling Acceptance Lab.**
