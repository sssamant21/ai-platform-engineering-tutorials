# Chapter 052 — Hands-On Lab: Cluster Autoscaler, Karpenter and GPU Warmup

**Part 07 | Chapter 052 | Published lab plan; live execution pending**

## Goal

Discover GPU node provisioning ownership, inspect unschedulable Pods, model GPU node bootstrap and inference warmup, assess zone/instance capacity, design bounded scale-out and rollback experiments, and reconcile GPU-hours.

## Safety

**[SAFE-READ]** authorized Kubernetes and cloud inventory. **[LAB-MUTATE]** isolated staging with explicit approval and bounded GPU-hours. **[CHANGE-CONTROL]** production CA/Karpenter, NodePools/NodeClasses, quotas, node image, disruption and model artifact caching. No PHI, customer prompts, Secrets or unapproved GPU nodes.

## Lab 0 — Preflight

Record UTC, cluster/context, Kubernetes version, cloud/provider, CA/Karpenter version, GPU SKU, node pool ownership, region/AZ, quotas, model digest, SLO, cost ceiling, approval and rollback.

## Lab 1 — Provisioner discovery [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get crd | grep -Ei 'nodepool|nodeclaim|ec2nodeclass|karpenter'
kubectl get daemonsets -A
```

Discover controller namespace and actual CRDs before requesting custom resources.

## Lab 2 — GPU scheduling and node readiness [SAFE-READ]

```bash
kubectl describe node APPROVED_GPU_NODE
kubectl -n APPROVED_NS describe pod APPROVED_GPU_POD
kubectl -n APPROVED_NS get events --sort-by=.lastTimestamp
```

Record GPU Capacity/Allocatable, node conditions, Pod Pending reasons, image pull, model readiness and driver/plugin health.

## Lab 3 — Cold-start budget [CALCULATION]

Use a conservative sequential example:

```text
30 s scaling decision
150 s GPU node launch
90 s node/GPU readiness
45 s image pull
120 s model download
75 s CUDA warmup
15 s readiness
TOTAL = 525 s = 8 min 45 s
```

Replace each stage with measured p50/p95; some stages overlap. Compare to TTFT objective and document minimum warm replicas.

## Lab 4 — Provisioner constraints [DESIGN]

| Constraint | Expected evidence |
|---|---|
| GPU family/SKU | cloud availability and quota |
| NodePool/NodeClass or node group | correct ownership |
| node labels/taints | matching GPU Pod |
| GPU allocatable | driver/plugin registered |
| zone diversity | capacity after one-zone loss |
| interruption policy | streaming-safe drain |
| warm capacity | cold-start SLO |
| cost ceiling | GPU-hours |

## Lab 5 — Synthetic scale-out plan [DESIGN]

Plan one approved synthetic GPU Pod and bounded traffic, record scheduling → node launch → GPU readiness → image/model warmup → serving readiness. Set strict abort thresholds, max nodes/replicas, cloud spend cap and cleanup. Do not run without explicit staging approval.

## Lab 6 — Eight tabletop game days

01 cloud quota; 02 AZ GPU shortage; 03 NodeClaim bootstrap; 04 plugin delay; 05 slow model download; 06 scale-to-zero; 07 consolidation stream drop; 08 competing provisioners.

## Twenty-four failure checklist

F01 CA absent; F02 Karpenter absent; F03 overlap; F04 template GPU; F05 selector; F06 taint; F07 SKU; F08 quota; F09 AZ; F10 NodeClass; F11 NodeClaim; F12 bootstrap; F13 driver; F14 plugin; F15 image; F16 IAM/KMS; F17 download; F18 warmup; F19 zero; F20 consolidation; F21 spot; F22 PDB; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | Provisioner ownership | Pending |
| G03 | Node templates | Pending |
| G04 | GPU SKU/quota | Pending |
| G05 | Labels/taints | Pending |
| G06 | Pending detection | Pending |
| G07 | Node launch | Pending |
| G08 | Bootstrap/CNI | Pending |
| G09 | Driver/plugin | Pending |
| G10 | Image pull | Pending |
| G11 | Model download | Pending |
| G12 | CUDA warmup | Pending |
| G13 | Readiness | Pending |
| G14 | Warm pool sizing | Pending |
| G15 | Spot/disruption | Pending |
| G16 | TTFT/TPOT SLO | Pending |
| G17 | Telemetry | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
Cloud/GPU SKU/region/AZ:
CA/Karpenter versions/ownership:
Node templates/taints/labels:
Cloud quota/capacity:
GPU allocatable/driver/plugin:
Pending → node launch → Ready:
Image/model pull and warmup:
TTFT/TPOT/terminal-validity:
Spot/consolidation/disruption:
GPU-hours/idle cost:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Approved staging tests must restore GitOps node-pool/provisioner configuration, remove only owned synthetic Pods, verify no orphaned GPU nodes, reconcile GPU-hours and retain sanitized evidence. All acceptance gates remain pending until independent review.

**Next: Chapter 053 — Multi-Zone GPU Capacity and Disruption Budgets.**
