# Lab 071 — GPU Utilization and Capacity SLOs: Acceptance Evidence

**Chapter:** [071 — GPU Utilization and Capacity SLOs](../part-09-gpu-observability/071-gpu-utilization-capacity-slos.md)

**Execution status:** NOT RUN. This worksheet does not claim real capacity measurements or SLO compliance.

## Safety preflight

Use read-only cluster discovery first. Capacity or autoscaling changes require explicit owner approval, a disposable staging workload, a rollback plan and a known GPU budget. Never evict production training jobs or reduce serving replicas to run the lab. Protect tenant/job names and infrastructure identifiers in shared evidence.

## 14-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | GPU pool inventory | model, memory, MIG, region and quotas | Pending |
| E02 | Capacity reconciliation | installed/healthy/allocatable/eligible counts | Pending |
| E03 | DCGM semantics | metric HELP/TYPE and labels | Pending |
| E04 | Utilization baselines | by-SKU time series and time window | Pending |
| E05 | Queue admission | queue wait histogram/counter definition | Pending |
| E06 | Fragmentation | concrete unschedulable placement example | Pending |
| E07 | Fleet availability SLI | independent inventory denominator | Pending |
| E08 | Workload SLO | numerator, denominator, owner, exclusions | Pending |
| E09 | Error budget | reproducible synthetic calculation | Pending |
| E10 | Dashboard | utilization/saturation/queue/coverage panels | Pending |
| E11 | Forecast | baseline, peak and N+1 scenario | Pending |
| E12 | Staging canary | approved plan, outcome and rollback evidence | Pending |
| E13 | Failure tabletop | node failure and quota shortage decisions | Pending |
| E14 | Acceptance | reviewer and score | Pending |

## Read-only discovery commands

\`\`\`bash
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl get pods -A -o wide
kubectl get pods -A --field-selector=status.phase=Pending
kubectl get resourcequota -A
kubectl get priorityclass
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get pods -A | grep -Ei 'dcgm|prometheus|gpu-operator'
\`\`\`

Verify the kube-state-metrics labels for GPU resources before using any published PromQL examples. Collect only approved, redacted samples.

## Synthetic calculation: reproduce without GPU access

**Illustrative inputs:** 20 eligible GPU devices observed for a 30-day month; 600 GPU-minutes are classified unavailable based on a validated independent inventory. Expected = 20 × 30 × 24 × 60 = 864,000 GPU-minutes. Availability = (864,000 − 600) / 864,000 = **99.9306%**. At a 99.5% objective, budget = 0.005 × 864,000 = **4,320 GPU-minutes**; remaining = **3,720 GPU-minutes**. This is an arithmetic exercise, **not** an observed SLO result.

\`\`\`python
devices = 20
days = 30
unavailable_gpu_minutes = 600
objective = 0.995
expected = devices * days * 24 * 60
availability = 1 - unavailable_gpu_minutes / expected
budget = expected * (1 - objective)
print({"expected_gpu_minutes": expected,
       "availability_percent": round(availability * 100, 4),
       "budget_gpu_minutes": round(budget),
       "remaining_gpu_minutes": round(budget - unavailable_gpu_minutes)})
assert expected == 864000
assert round(budget) == 4320
assert round(budget - unavailable_gpu_minutes) == 3720
\`\`\`

A real fleet with changing inventory requires time-varying eligible GPU counts, maintenance policies and independent health records; do not copy this fixed-population model into production unchanged.

## Staging canary and failure drill

Before a staging capacity change, record baseline eligible jobs, queue age, p95/p99 serving latency, GPU/MIG inventory, utilization, failure reserve and cloud quota. Use a bounded canary, measure the same signals after change, and roll back if queue SLO, latency SLO or failover reserve is violated. A **tabletop** node failure is sufficient if real staging GPU resources are unavailable; label it as tabletop, not live resilience validation.

## Evidence record

\`\`\`yaml
chapter: "071"
environment: "STAGING_OR_OFFLINE"
date_utc: null
reviewer: null
gpu_skus: []
mig_profiles: []
expected_gpu_minutes: null
unavailable_gpu_minutes: null
fleet_availability_sli: null
slo_target: null
error_budget_gpu_minutes: null
telemetry_coverage: null
queue_admission_sli: null
capacity_forecast_review: "PENDING"
staging_canary: "NOT_RUN"
node_failure_tabletop: "NOT_RUN"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## Acceptance and stop conditions

Score Chapter 071's gates **20 + 20 + 15 + 15 + 15 + 15 = 100**. Pass at **≥85**, with no critical security or workload safety failure. Stop for unexpected production impact, unauthorized quota changes, workload eviction, missing capacity reserves, or evidence that the SLO denominator is based only on observed exporter series.

**Next:** Chapter 072 — Cluster Telemetry and Event Correlation.
