# Chapter 054 — Hands-On Lab: GPU Scheduling, Quota and Pending Pod Troubleshooting

**Part 07 | Chapter 054 | Published lab plan; live execution pending**

## Goal

Classify GPU Pod admission and Pending failures, collect scheduler/device-plugin/autoscaler evidence, build reversible incident decisions, validate GPU capacity and capture rollback/acceptance records.

## Safety

**[SAFE-READ]** authorized Kubernetes inventory and sanitized events. **[LAB-MUTATE]** approved isolated staging only. **[CHANGE-CONTROL]** production ResourceQuota, RBAC, taints, affinity, GPU node pools, device plugin, driver, MIG and autoscaler. No PHI, customer prompts, Secrets or unapproved GPU workloads.

## Lab 0 — Preflight

Record UTC, context, cluster/provider version, namespace, GPU resource key, Pod owner, node-pool ownership, quotas, SLO, cost ceiling, approval and rollback.

## Lab 1 — Admission versus Pending [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl -n APPROVED_NS get pods -o wide
kubectl -n APPROVED_NS get events --sort-by=.lastTimestamp
kubectl -n APPROVED_NS describe pod APPROVED_GPU_POD
kubectl -n APPROVED_NS get resourcequota,limitrange
```

If no Pod exists, inspect approved controller/GitOps errors for admission rejection rather than assuming scheduling.

## Lab 2 — Node and GPU resources [SAFE-READ]

```bash
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl describe node APPROVED_GPU_NODE
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Record Capacity/Allocatable/allocated, driver/device-plugin presence, labels, taints and scheduler event messages.

## Lab 3 — Storage, topology and priority [SAFE-READ]

```bash
kubectl get pvc -A
kubectl get priorityclasses
kubectl get nodes -L topology.kubernetes.io/zone
kubectl -n APPROVED_NS get pods -o wide
```

Compare Pod selector, required affinity, topology spread and PVC zone binding against eligible GPU nodes.

## Lab 4 — Autoscaler discovery [SAFE-READ]

```bash
kubectl get crd | grep -Ei 'nodepool|nodeclaim|ec2nodeclass|karpenter'
kubectl get pods -A -o wide
```

Identify CA/Karpenter controller ownership and inspect authorized logs only after discovering namespace/pod names. Do not assume controller installation.

## Lab 5 — Failure classification [DESIGN]

| Failure | Root-cause evidence | Approved remediation |
|---|---|---|
| admission quota | Forbidden + used/hard | owner-approved quota/capacity |
| insufficient GPU | allocatable/allocated | capacity/provisioner |
| taint mismatch | Pod/node spec | exact toleration |
| affinity mismatch | labels/required rules | policy correction |
| topology conflict | eligible zones | capacity or approved rule |
| PVC mismatch | PVC/PV topology | storage owner plan |
| plugin failure | daemonset/NVML | driver/plugin runbook |
| cloud quota | autoscaler/cloud events | capacity request |

## Lab 6 — Eight tabletop game days

01 GPU quota; 02 plugin failure; 03 taint drift; 04 label mismatch; 05 topology deadlock; 06 PVC zone; 07 cloud quota; 08 incomplete rollback.

## Twenty-four failure checklist

F01 GPU quota; F02 Pod count; F03 CPU/memory; F04 LimitRange; F05 RBAC; F06 Pod Security; F07 wrong resource; F08 allocatable zero; F09 plugin; F10 driver; F11 allocated; F12 taint; F13 selector; F14 affinity; F15 anti-affinity; F16 topology; F17 PVC; F18 preemption; F19 CA; F20 Karpenter; F21 cloud SKU; F22 MIG; F23 warmup; F24 rollback.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/context | Pending |
| G02 | Admission/Pending classification | Pending |
| G03 | ResourceQuota | Pending |
| G04 | CPU/memory/LimitRange | Pending |
| G05 | RBAC/Pod Security | Pending |
| G06 | GPU resource key | Pending |
| G07 | Allocatable/allocated | Pending |
| G08 | Driver/plugin | Pending |
| G09 | Taints/tolerations | Pending |
| G10 | Node selectors | Pending |
| G11 | Affinity | Pending |
| G12 | Topology | Pending |
| G13 | PVC/zone | Pending |
| G14 | Priority | Pending |
| G15 | CA/Karpenter | Pending |
| G16 | Cloud SKU/quota | Pending |
| G17 | MIG/model readiness | Pending |
| G18 | Rollback | Pending |
| G19 | SLO/FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
Namespace/Pod/owner:
API admission result:
Scheduler events:
GPU Capacity/Allocatable/allocated:
ResourceQuota/LimitRange:
RBAC/Pod Security:
Taints/selectors/affinity/topology:
PVC/zone/priority:
Device plugin/NVML/MIG:
CA/Karpenter/cloud quota:
Model readiness/TTFT/TPOT:
GPU-hours/cost:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop activities create no resources. Approved staging tests must restore original GitOps quota, placement and provisioner settings, remove only owned test workloads, verify GPU node readiness, reconcile GPU-hours and retain sanitized evidence. All gates remain pending until independent review.

**Next: Chapter 055 — GPU Autoscaling Acceptance Lab.**
