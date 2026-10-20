# Lab 084 — GPU Rightsizing and Utilization Acceptance Gate

**Chapter:** [084 — GPU Rightsizing and Utilization](../part-11-capacity-finops-and-reliability/084-gpu-rightsizing-utilization.md)

**Status:** NOT EXECUTED. Offline analysis, isolated GPU staging benchmark, independent technical/security review and acceptance remain pending.

## 1. Safety

Use synthetic or approved redacted telemetry. Do not resize live inference, change MIG profiles, enable time-slicing, modify node pools, terminate training jobs or alter quotas without change approval and rollback. Preserve checkpoints, customer SLOs, tenant isolation and failure headroom.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | GPU/SKU/MIG inventory | compatible resource matrix | Pending |
| E02 | Allocation reconciliation | allocated GPU-hours | Pending |
| E03 | Utilization baseline | coverage and weighted rate | Pending |
| E04 | Low-util training analysis | CPU/storage/NCCL hypothesis | Pending |
| E05 | Inference KV-cache pressure | HBM and latency evidence | Pending |
| E06 | GPU SKU fit | HBM/interconnect/kernel matrix | Pending |
| E07 | Training cost per step | useful-work metric | Pending |
| E08 | Inference cost per request | SLO-compliant cost | Pending |
| E09 | MIG profile simulation | stranded resources | Pending |
| E10 | Sharing isolation review | MIG vs time-slicing | Pending |
| E11 | Kubernetes scheduling review | requests/quotas/pending | Pending |
| E12 | Training batch/precision test | convergence and throughput | Pending |
| E13 | Serving canary | peak p99 and error rate | Pending |
| E14 | HA/rollback tabletop | node/AZ-loss capacity | Pending |
| E15 | Savings reconciliation | actual vs nominal savings | Pending |
| E16 | Independent review | score and risk register | Pending |

## 3. Offline utilization and GPU-hour calculations

\`\`\`python
jobs = [
    {"gpus": 4, "hours": 24, "util": 0.30},
    {"gpus": 2, "hours": 12, "util": 0.75},
]
allocated = sum(j["gpus"] * j["hours"] for j in jobs)
weighted = sum(j["gpus"] * j["hours"] * j["util"] for j in jobs) / allocated
print("allocated_gpu_hours:", allocated)
print("weighted_utilization:", round(weighted, 3))
assert allocated == 120
assert round(weighted, 3) == 0.39
\`\`\`

Expected: **120 GPU-hours** and **39% weighted GPU activity**. This does not establish that 61% of cost is avoidable.

## 4. Offline rightsizing calculation

\`\`\`python
baseline_gpus = 4
candidate_gpus = 3
hours = 24
baseline = baseline_gpus * hours
candidate = candidate_gpus * hours
released = baseline - candidate
fraction = released / baseline
print(baseline, candidate, released, f"{fraction:.0%}")
assert (baseline, candidate, released) == (96, 72, 24)
assert fraction == 0.25
\`\`\`

Expected: 96 vs 72 GPU-hours/day; 25% resource reduction. **Not** automatically 25% invoice savings. Validate p99, availability and committed charges before any cost claim.

## 5. Synthetic diagnosis fixtures

**A — Low GPU SM utilization:** 20% busy, high data-loader wait, low memory use. Investigate CPU/storage before GPU removal.

**B — Memory-bound serving:** high KV-cache occupancy, low SM utilization, p99 spike at long context. Evaluate admission and memory headroom.

**C — MIG fragmentation:** remaining slices do not match queued workload profile. Identify compatible placement; no production MIG reconfiguration.

**D — Time-slicing interference:** two tenants share a GPU and p99 rises. Explain why time-slicing is not MIG isolation.

**E — Reserved fleet:** downsizing releases GPUs but contractual bill remains unchanged. Calculate opportunity value separately from realized savings.

## 6. Staging canary plan

- Pin model/serving image, GPU SKU, traffic distribution and baseline.
- Set measurable acceptance thresholds for p99, errors, tokens/sec, memory headroom and numerical quality.
- Confirm one-node/AZ-loss redundancy, rollback artifact and owner.
- Compare baseline and candidate over representative peak and off-peak loads.
- Abort immediately if SLO, accuracy, checkpoint or tenant isolation gates fail.
- Record actual billed cost changes only after FinOps reconciliation.

If isolated GPU staging is unavailable, mark the canary **NOT_RUN** rather than PASS.

## 7. Evidence template

\`\`\`yaml
chapter: "084"
environment: "OFFLINE_OR_STAGING"
execution_date_utc: null
reviewer: null
gpu_sku: null
gpu_allocation_hours: null
utilization_coverage_percent: null
weighted_utilization: null
baseline_cost_per_unit: null
candidate_cost_per_unit: null
mig_isolation_review: "PENDING"
training_quality_test: "NOT_RUN"
inference_p99_canary: "NOT_RUN"
ha_headroom_test: "NOT_RUN"
realized_savings_verified: false
rollback_test: "NOT_RUN"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 8. Acceptance

Score **20 + 20 + 20 + 20 + 10 + 10 = 100**. Pass at ≥85 with no critical SLO, model quality, tenant isolation, data integrity, unapproved disruption or false savings claim. Documentation publication does not constitute executed lab evidence.

**Next:** Chapter 085 — Capacity Reservation and Quota Governance.
