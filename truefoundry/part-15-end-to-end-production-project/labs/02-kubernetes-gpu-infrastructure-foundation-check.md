# Chapter 15.2 — Hands-on Lab: Kubernetes and GPU Infrastructure Foundation

**Part 15 — End-to-End Production Project | Approved staging only | Execution pending**

## Objective

Validate an approved Kubernetes GPU foundation across EKS, AKS or GKE, including IaC review, GPU node pools, driver/device plugin, scheduling, quota, telemetry, N-1 and cost cleanup.

**Safety:** read-only discovery by default. Applying manifests, provisioning GPUs, changing quotas or autoscalers requires staging approval, budget and teardown plan. No production mutations or secrets in lab evidence.

## Lab 0 — Preflight

Record Chapter 15.1 ADRs, cloud/region/AZ, cluster, Kubernetes version, GPU SKU, image/runtime/driver matrix, node labels/taints, quota, N-1 definition, GPU-hour cap and authorized reviewer.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L topology.kubernetes.io/zone,node.kubernetes.io/instance-type
kubectl get resourcequota,limitrange -A
kubectl get events -A --sort-by=.lastTimestamp
```

## Lab 1 — IaC and identity

Review version-pinned Terraform/OpenTofu or approved IaC plan, state encryption/locking, network private subnets, workload identity, Kubernetes RBAC and cloud policy validation. Do not execute an unreviewed apply. Capture PR, plan, approval and rollback.

## Lab 2 — GPU discovery and driver/plugin

```bash
kubectl get daemonsets -A
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" allocatable="}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
kubectl get pods -A -o wide
```

Confirm GPU resource name (including MIG variants), driver/device plugin ownership, node allocatable, DCGM telemetry and any XID/ECC evidence. Use an approved diagnostic Pod with pinned image digest only after staging change approval.

## Lab 3 — Scheduling and quotas

Check a disposable staging namespace with approved GPU quota, node selector, taints/tolerations, topology and requests. Verify Pending events for intentionally unsatisfiable scheduling using a **tabletop** if real capacity would be consumed. Do not remove production taints to force placement.

## Lab 4 — Capacity and cost

Compute steady GPUs, rolling overlap, per-zone capacity, N-1 loss scenario, cloud GPU quota and autoscaler max. Estimate GPU-hours and budget for warm replicas and lab. Check cleanup plan and cost labels.

## Lab 5 — Six game days

1. GPU quota/Pod Pending.
2. Driver or plugin mismatch (tabletop).
3. Node NotReady with failover.
4. Autoscaler AZ shortage.
5. XID/ECC and node drain decision (tabletop).
6. IaC drift and GPU cost orphan.

Record UTC timeline, evidence, blast radius, SLO, approved mitigation, rollback, GPU-hours, reviewer and executed/tabletop.

## Forty-eight scenario checklist

01 quota; 02 SKU; 03 AZ; 04 IPs; 05 endpoints; 06 IAM; 07 Kubernetes version; 08 RBAC; 09 CNI; 10 DNS; 11 NotReady; 12 cert; 13 driver; 14 CUDA; 15 plugin; 16 collision; 17 allocatable; 18 MIG; 19 Pending; 20 taints; 21 affinity; 22 topology; 23 namespace quota; 24 PDB; 25 image; 26 device; 27 CUDA init; 28 GPU OOM; 29 host OOM; 30 XID; 31 scale delay; 32 max; 33 register; 34 node template; 35 spot; 36 surge; 37 DCGM; 38 stale; 39 alerts; 40 logs; 41 utilization; 42 cost tags; 43 drift; 44 lock; 45 review; 46 N-1; 47 budget; 48 orphan.

## Fourteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved ADR/IaC plan | Pending |
| G2 | Private network and egress | Pending |
| G3 | Cluster version and RBAC | Pending |
| G4 | GPU SKU/driver/CUDA compatibility | Pending |
| G5 | GPU allocatable and device visibility | Pending |
| G6 | Plugin/DCGM and metrics freshness | Pending |
| G7 | Scheduling/taints/affinity | Pending |
| G8 | GPU quotas and admission | Pending |
| G9 | Autoscaler/AZ capacity | Pending |
| G10 | N-1 and rolling surge | Pending |
| G11 | Synthetic GPU smoke | Pending |
| G12 | Security/audit controls | Pending |
| G13 | GPU-hour budget and teardown | Pending |
| G14 | Reviewer sign-off/handoff | Pending |

## Evidence template

```text
Lab ID / approval / UTC:
Cloud, region, AZ, cluster/version:
IaC commit, plan, state protections:
GPU SKU, driver, CUDA, plugin, MIG:
Node allocatable and schedulable GPU:
Namespace quota, taints, affinity:
Autoscaler max, zone availability, N-1:
DCGM metrics and hardware alerts:
Synthetic diagnostic Pod result:
GPU-hours, cost labels and cleanup:
Executed/tabletop, blockers, reviewer:
```

## Cleanup

Remove only lab-created resources through approved IaC or namespace ownership, verify GPU nodes terminate where expected, budgets/alerts remain enabled and no production state changed. Leave gates pending until independently executed and reviewed.

**Next:** Chapter 15.3 — TrueFoundry Integration and Secure Model Serving.
