# Chapter 048 — Hands-On Lab: GPU Node Pools, Taints, Tolerations and Affinity

**Part 07 | Chapter 048 | Published lab plan; live execution pending**

## Goal

Inventory GPU node pools and placement rules, identify taint/selector/affinity/topology constraints, design a bounded synthetic GPU placement test, evaluate autoscaler capacity and produce rollback evidence.

## Safety

**[SAFE-READ]** authorized Kubernetes and cloud inventory. **[LAB-MUTATE]** approved isolated staging with bounded GPU-hours. **[CHANGE-CONTROL]** production node labels/taints, node pools, priority, affinity and autoscaler changes. No PHI, customer prompts, Secrets or unapproved GPU node operations.

## Lab 0 — Preflight

Record UTC, cluster/context, Kubernetes/cloud provider version, GPU SKU, node pool labels/taints, quotas, active workloads, SLO, change approval, reviewer, cost ceiling and rollback.

## Lab 1 — Node and Pod inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -A -o wide
kubectl get priorityclasses
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Record node allocatable/allocated GPUs, labels, taints, zones and Pod Pending reasons.

## Lab 2 — Namespace quota and placement [SAFE-READ]

```bash
kubectl -n APPROVED_NS get resourcequota,limitrange
kubectl -n APPROVED_NS get pods -o wide
kubectl -n APPROVED_NS describe pod APPROVED_POD
```

Compare Pod tolerations, selectors, affinity and topology spread against actual nodes.

## Lab 3 — Synthetic Pod design [DESIGN]

```yaml
# EXAMPLE ONLY — NOT READY TO APPLY
apiVersion: v1
kind: Pod
metadata:
  name: gpu-placement-smoke
  namespace: approved-staging
spec:
  restartPolicy: Never
  nodeSelector:
    workload-tier: gpu-inference
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu-inference
    effect: NoSchedule
  containers:
  - name: cuda
    image: registry.example.invalid/cuda@sha256:REPLACE
    command: ["sh", "-c", "nvidia-smi && sleep 10"]
    resources:
      limits:
        nvidia.com/gpu: 1
```

Before any staging execution, replace placeholders with approved digest, namespace, actual labels and taints. Require explicit abort and GPU-hour limits.

## Lab 4 — Topology and autoscaler plan [DESIGN]

| Constraint | Evidence | Expected action |
|---|---|---|
| taints/tolerations | node/Pod spec | verify matching |
| selector/affinity | labels and expressions | verify eligible nodes |
| topology spread | zone labels | check feasible distribution |
| multi-GPU | per-node allocatable | validate same-node fit |
| node provisioner | cloud quota/template | verify scale-out feasibility |
| warm capacity | cold-start timing | compare against TTFT SLO |

## Lab 5 — Rollback and recovery [TABLETOP]

Simulate missing label, unexpected taint, GPU AZ shortage, hard topology conflict and wrong pool placement. Restore the complete GitOps workload + node-pool configuration only under approved change control.

## Lab 6 — Eight game days

01 missing label; 02 taint drift; 03 AZ GPU shortage; 04 topology deadlock; 05 multi-GPU fit; 06 spot interruption; 07 broad toleration; 08 incomplete rollback.

## Twenty-four failure checklist

F01 pool absent; F02 SKU; F03 quota; F04 node image; F05 taint; F06 broad toleration; F07 missing toleration; F08 selector; F09 affinity; F10 soft affinity; F11 anti-affinity; F12 topology; F13 multi-GPU; F14 plugin; F15 autoscaler template; F16 spot; F17 priority; F18 PDB; F19 image; F20 model; F21 VRAM; F22 tenant; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | Node-pool inventory | Pending |
| G03 | GPU SKU/VRAM | Pending |
| G04 | Quota/capacity | Pending |
| G05 | Taints | Pending |
| G06 | Tolerations | Pending |
| G07 | Selectors | Pending |
| G08 | Affinity | Pending |
| G09 | Anti-affinity | Pending |
| G10 | Topology spread | Pending |
| G11 | Multi-GPU fit | Pending |
| G12 | Autoscaler template | Pending |
| G13 | Driver/plugin | Pending |
| G14 | Startup/warm pool | Pending |
| G15 | Priority/disruption | Pending |
| G16 | Tenant/security | Pending |
| G17 | Telemetry/SLO | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
GPU node pools/SKU/zone:
GPU allocatable/allocated:
Node labels/taints:
Pod tolerations/selectors/affinity:
Topology and multi-GPU:
Cloud quota/autoscaler:
Driver/plugin and startup:
Security/tenant isolation:
TTFT/TPOT/terminal validity:
GPU-hours/cost:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop activities create no resources. Approved staging tests must remove only owned Pods, restore GitOps source of truth and node-pool configuration, verify no orphaned GPU nodes, reconcile GPU-hours and retain sanitized evidence. No gate is complete without independent review.

**Next: Chapter 049 — GPU Resource Quotas and Namespace Isolation.**
