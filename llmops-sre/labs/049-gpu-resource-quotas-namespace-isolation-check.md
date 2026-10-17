# Chapter 049 — Hands-On Lab: GPU Resource Quotas and Namespace Isolation

**Part 07 | Chapter 049 | Published lab plan; live execution pending**

## Goal

Inventory namespace GPU quotas, RBAC, Pod Security, NetworkPolicies and workload identity; design safe GPU allocation and tenant admission tests; document sharing, model isolation, runtime token fairness, chargeback and rollback.

## Guardrails

**[SAFE-READ]** authorized Kubernetes inventory. **[LAB-MUTATE]** isolated staging with explicit approval and bounded GPU-hours. **[CHANGE-CONTROL]** production ResourceQuota, RBAC, NetworkPolicy, IAM, admission, GPU sharing and model routes. No PHI, customer prompts, Secrets or credential dumps.

## Lab 0 — Preflight

Record UTC, context, cluster/Kubernetes version, GPU plugin resource names, namespaces, quotas, RBAC owner, CNI, GPU sharing mode, cloud workload identity, SLO, cost ceiling and rollback.

## Lab 1 — Quota and policy inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get namespaces
kubectl get resourcequota -A
kubectl get limitrange -A
kubectl get networkpolicy -A
kubectl get serviceaccounts -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl describe node APPROVED_GPU_NODE
```

Do not enumerate Secret values or grant new permissions during discovery.

## Lab 2 — Namespace quota details [SAFE-READ]

```bash
kubectl -n APPROVED_NS get resourcequota,limitrange
kubectl -n APPROVED_NS describe resourcequota APPROVED_QUOTA
kubectl -n APPROVED_NS get pods -o wide
kubectl -n APPROVED_NS get events --sort-by=.lastTimestamp
```

Capture hard/used GPU, CPU, memory and Pod quotas; distinguish admission denial from scheduler Pending.

## Lab 3 — Quota design [DESIGN]

```yaml
# STAGING DESIGN ONLY — VERIFY RESOURCE NAMES
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-budget
  namespace: approved-staging
spec:
  hard:
    requests.nvidia.com/gpu: "2"
    pods: "12"
    requests.cpu: "16"
    requests.memory: 64Gi
```

Never overwrite existing quota without owner approval. Validate with installed API and admission policies.

## Lab 4 — Isolation matrix [DESIGN]

| Boundary | Control | Evidence |
|---|---|---|
| API access | namespace RBAC | pending |
| GPU allocation | ResourceQuota | pending |
| privileged containers | Pod Security | pending |
| network | CNI NetworkPolicy | pending |
| model artifacts | workload identity/IAM | pending |
| inference API | JWT/model authorization | pending |
| token/KV fairness | gateway/runtime limits | pending |
| sharing | MIG/device plugin policy | pending |
| costs | tenant GPU-hour attribution | pending |

## Lab 5 — Admission and rollback tabletop

Simulate quota exhaustion, wrong GPU resource key, denied DNS egress, cross-tenant model IAM, time-slicing contention and policy drift. Define expected events, containment, rollback, synthetic golden inference and reviewer sign-off.

## Lab 6 — Eight game days

01 quota denial; 02 quota RBAC escalation; 03 model egress blocked; 04 artifact IAM exposure; 05 sharing contention; 06 KV monopoly; 07 preemption; 08 incomplete rollback.

## Twenty-four failure checklist

F01 quota absent; F02 wrong key; F03 exhausted; F04 CPU/memory; F05 LimitRange; F06 RBAC quota; F07 Pod Security; F08 service account; F09 DNS; F10 storage egress; F11 workload identity; F12 cross-tenant IAM; F13 image; F14 unexpected sharing; F15 noisy neighbor; F16 MIG; F17 KV; F18 token quotas; F19 priority; F20 service exposure; F21 privacy; F22 chargeback; F23 rollback; F24 orphaned GPU.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | Namespace ownership | Pending |
| G03 | GPU resource key | Pending |
| G04 | ResourceQuota | Pending |
| G05 | CPU/memory/LimitRange | Pending |
| G06 | RBAC | Pending |
| G07 | Pod Security | Pending |
| G08 | NetworkPolicy | Pending |
| G09 | Workload identity | Pending |
| G10 | Storage IAM | Pending |
| G11 | Image integrity | Pending |
| G12 | Sharing/MIG | Pending |
| G13 | Tenant token quotas | Pending |
| G14 | Fairness/priority | Pending |
| G15 | SLO/terminal validity | Pending |
| G16 | Privacy/telemetry | Pending |
| G17 | FinOps/chargeback | Pending |
| G18 | Full rollback | Pending |
| G19 | Game-day evidence | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
Namespaces/owners:
GPU plugin resource names:
GPU quota hard/used:
CPU/memory/LimitRange:
RBAC/Pod Security/admission:
CNI NetworkPolicy/DNS/storage:
Workload identity/artifact IAM:
MIG/sharing configuration:
Tenant tokens/KV and SLO:
Privacy/telemetry:
GPU-hours/chargeback:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Approved staging tests must restore GitOps quota, RBAC, NetworkPolicy and identity configuration, remove only owned GPU Pods, verify no orphaned GPU nodes and reconcile GPU-hours. All acceptance gates remain pending until independent evidence review.

**Next: Chapter 050 — MIG, GPU Sharing and Multi-Tenant Risks.**
