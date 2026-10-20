# Lab 085 — Capacity Reservation and Quota Governance Acceptance Gate

**Chapter:** [085 — Capacity Reservation and Quota Governance](../part-11-capacity-finops-and-reliability/085-capacity-reservation-quota-governance.md)

**Status:** NOT EXECUTED. Offline policy exercises, staging quota tests, cloud reservation review, security review and independent acceptance remain pending.

## 1. Safety

Use read-only production inspection and isolated staging for Kubernetes quota/admission tests. Do not create billable cloud reservations, modify production quotas, raise PriorityClass privileges, preempt active training jobs or change protected serving capacity without explicit approvals. Financial figures and cloud guarantees must be validated against real contracts.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | GPU/AZ inventory | SKU/node/zone matrix | Pending |
| E02 | Six capacity planes | distinctions worksheet | Pending |
| E03 | ResourceQuota audit | read-only manifests | Pending |
| E04 | Priority/RBAC review | privilege findings | Pending |
| E05 | Kueue queue policy | staged manifests | Pending |
| E06 | Gang admission simulation | Pending reason | Pending |
| E07 | Tenant tiers | floor/ceiling rules | Pending |
| E08 | Headroom calculation | protected/burst model | Pending |
| E09 | Borrow/reclaim tabletop | checkpoint-safe plan | Pending |
| E10 | Node/AZ loss | compatible capacity model | Pending |
| E11 | Provider comparison | quota/reservation/discount | Pending |
| E12 | Reservation register | owner/expiry/terms | Pending |
| E13 | Orphan/expiry audit | exception report | Pending |
| E14 | Commitment cost exposure | actual vs nominal | Pending |
| E15 | Policy/alert tabletop | denied and escalated requests | Pending |
| E16 | Independent review | score and risk register | Pending |

## 3. Read-only Kubernetes audit

\`\`\`bash
kubectl config current-context
kubectl get resourcequota -A
kubectl get limitrange -A
kubectl get priorityclass
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase=Pending
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

If Kueue is installed, inspect its versioned CRDs and relevant ClusterQueues/LocalQueues/ResourceFlavors. Record \`NOT_INSTALLED\` where unavailable; do not apply manifests to production.

## 4. Staging-only ResourceQuota example

\`\`\`yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-team-a
  namespace: gpu-team-a
spec:
  hard:
    requests.nvidia.com/gpu: "8"
    pods: "50"
\`\`\`

Create namespace and quota **only** in approved staging; verify denied requests at 9 GPUs and successful admission at or below quota **only if compatible physical GPUs exist**. ResourceQuota admission success does not prove the pod will schedule.

## 5. Offline headroom calculation

\`\`\`python
installed = 64
gpus_per_node = 8
protected = 16
training = 24
research = 8
free = installed - protected - training - research
after_node_failure = installed - gpus_per_node
print("unallocated:", free, "after_failure:", after_node_failure)
assert free == 16
assert after_node_failure == 56
assert protected + training + research <= after_node_failure
\`\`\`

Expected: 16 GPUs unallocated and 56 after one 8-GPU node failure. This **does not prove** topology feasibility, zone resilience or preservation of all workloads; verify compatible node bundles.

## 6. Synthetic governance cases

**A — Cloud quota vs supply:** quota approved, GPU VM unavailable in requested AZ. Distinguish the two control planes.

**B — Borrowing violation:** research borrows 16 GPUs and P0 serving demand rises. Draft reclaim decision with checkpoint and PDB safety.

**C — MIG mismatch:** available slices cannot satisfy a full-GPU job. Quota does not fix profile incompatibility.

**D — Expiring reservation:** launch forecast exceeds renewal lead time. Identify owner, approver and fallback.

**E — Billing mismatch:** committed spend continues after capacity release. Report capacity value separately from invoice savings.

## 7. Evidence template

\`\`\`yaml
chapter: "085"
environment: "OFFLINE_OR_STAGING"
execution_date_utc: null
reviewer: null
gpu_sku_zone_inventory: []
quota_audit: "NOT_RUN"
priority_rbac_review: "PENDING"
kueue_policy_test: "NOT_RUN"
gang_admission_test: "NOT_RUN"
tenant_floor_ceiling_policy: "NOT_REVIEWED"
borrowing_tabletop: "NOT_RUN"
node_az_resilience: "NOT_EVALUATED"
reservation_contract_review: "PENDING"
reservation_expiry_audit: "NOT_RUN"
commitment_cost_reconciliation: "NOT_RUN"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 8. Acceptance

Score **20 + 20 + 20 + 15 + 15 + 10 = 100**. Pass at ≥85 with no critical tenant isolation, unsafe preemption, quota privilege escalation, unapproved cloud spending, checkpoint integrity or protected SLO violation. A documented policy is not a tested policy.

**Next:** Chapter 086 — N-1 and Multi-Zone GPU Resilience.
