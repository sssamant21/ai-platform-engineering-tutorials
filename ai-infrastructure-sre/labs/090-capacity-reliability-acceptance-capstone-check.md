# Lab 090 — Capacity and Reliability Acceptance Capstone Gate

**Chapter:** [090 — Capacity and Reliability Acceptance Capstone](../part-11-capacity-finops-and-reliability/090-capacity-reliability-acceptance-capstone.md)

**Execution:** NOT RUN. Staging benchmarks, node/AZ fault injection, billing reconciliation, DR restoration, security review and independent acceptance are pending.

## 1. Safety and prerequisites

Use isolated staging and synthetic workload data for fault injection. Production may be inspected read-only with authorization. Do not evict live GPUs, disable an AZ, alter routing, create cloud commitments, delete checkpoints or change queue priorities without explicit change control. Confirm incident commander, workload owners, rollback and cost limits before any test.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | Status |
|---|---|---|---|
| E01 | SLO/RTO/RPO contracts | signed definitions | Pending |
| E02 | GPU/zone/quota inventory | compatible matrix | Pending |
| E03 | Demand forecast | history, error bounds | Pending |
| E04 | Utilization baseline | allocated GPU-hours, throughput | Pending |
| E05 | Rightsizing canary | quality/p99 comparison | Pending |
| E06 | MIG/sharing review | tenant isolation gate | Pending |
| E07 | Quota/borrowing review | approved policy | Pending |
| E08 | N-1/AZ capacity math | worst-case feasibility | Pending |
| E09 | Node-loss serving test | p99/errors | Pending |
| E10 | Training resume | complete checkpoint restore | Pending |
| E11 | Cost ledger | reconciled invoice | Pending |
| E12 | Queue/overload test | admission and backpressure | Pending |
| E13 | Provisioning/warmup | measured lead time | Pending |
| E14 | Artifact restore | SHA-256, optimizer state | Pending |
| E15 | Regional DR/failback | RTO/RPO and fencing | Pending |
| E16 | Independent signoff | score, COAs and owners | Pending |

## 3. Offline arithmetic gate

\`\`\`python
from math import ceil

nodes_per_zone = {"az-a": 4, "az-b": 4}
gpus_per_node = 8
installed = sum(nodes_per_zone.values()) * gpus_per_node
n_minus_one = installed - gpus_per_node
survivor_by_failed_zone = {
    failed: sum(n * gpus_per_node for zone, n in nodes_per_zone.items() if zone != failed)
    for failed in nodes_per_zone
}
worst_az = min(survivor_by_failed_zone.values())

protected_inference = 24
non_preemptible_training = 16
base_replicas = ceil(900 / 150)
n1_replicas = base_replicas + 2

gpu_hours = 8 * (4 + 2)
cost_per_step = (gpu_hours * 2 + 12) / 12000

print(installed, n_minus_one, worst_az, base_replicas, n1_replicas, cost_per_step)
assert (installed, n_minus_one, worst_az) == (64, 56, 32)
assert protected_inference <= worst_az
assert protected_inference + non_preemptible_training > worst_az
assert (base_replicas, n1_replicas) == (6, 8)
assert cost_per_step == 0.009
\`\`\`

Expected: **64 installed**, **56 after one node loss**, **32 after one AZ loss**, **6 base serving replicas**, **8 simplified N-1 replicas**, and **$0.009 per successful training step using a fictional rate**. The aggregate AZ test **fails** when 16 non-preemptible training GPUs remain protected alongside 24 inference GPUs.

## 4. Read-only Kubernetes inventory

\`\`\`bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get resourcequota -A
kubectl get priorityclass
kubectl get hpa -A
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

Inspect GPU Operator, Kueue, KEDA and node provisioner CRDs only when installed. Record \`NOT_INSTALLED\` instead of treating absence as a pass.

## 5. Staging fault-injection matrix

| Drill | Required evidence | Abort immediately if |
|---|---|---|
| Peak inference traffic | p99, errors, throughput, ready GPUs | protected SLO breach |
| GPU rightsizing | baseline/candidate quality and cost | OOM/quality regression |
| Quota exhaustion | admission denial and audit | privilege bypass |
| Training gang placement | schedulability and fairness | P0 starvation |
| GPU node failure | serving continuity and restart | capacity shortfall |
| Synthetic AZ loss | survivor placement and p99 | protected traffic fails |
| Cost reconciliation | invoice variance and owner mapping | double-counting |
| Corrupt checkpoint copy | checksum rejection | invalid restore accepted |
| Regional recovery | measured end-to-end RTO/RPO | security/fencing failure |
| Failback | single writer and safe traffic shift | duplicate writes |

## 6. Evidence manifest

\`\`\`yaml
chapter: "090"
environment: "STAGING_REQUIRED"
publication: "PUBLISHED"
execution: "NOT_RUN"
execution_date_utc: null
platform_owner: null
ml_owner: null
finops_reviewer: null
security_reviewer: null
independent_reviewer: null
gpu_sku_zone_inventory: []
forecast_validation: "NOT_RUN"
rightsizing_canary: "NOT_RUN"
tenant_isolation_review: "PENDING"
quota_governance_test: "NOT_RUN"
n_minus_one_test: "NOT_RUN"
az_loss_test: "NOT_RUN"
invoice_reconciliation: "NOT_RUN"
queue_overload_test: "NOT_RUN"
checkpoint_restore: "NOT_RUN"
regional_dr_test: "NOT_RUN"
failback_fencing: "NOT_RUN"
rollback_test: "NOT_RUN"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
corrective_actions: []
\`\`\`

## 7. Acceptance score

| Domain | Points |
|---|---:|
| Forecasting | 10 |
| Rightsizing and quality/SLO | 15 |
| Quota and reservation governance | 15 |
| N-1/AZ resilience | 20 |
| Unit economics | 10 |
| Scaling and admission | 10 |
| DR and checkpoint integrity | 15 |
| Independent review/rollback | 5 |
| **Total** | **100** |

**Pass:** ≥85/100 and every critical gate passes. Do not mark PASSED without timestamped staging results, independent review and approved evidence. Publication completes Part 11 **documentation**, not production readiness.

**Next:** Chapter 091 — Enterprise AI Infrastructure Reference Architecture.
