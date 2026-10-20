# Lab 087 — GPU Cost Allocation and Unit Economics Acceptance Gate

**Chapter:** [087 — GPU Cost Allocation and Unit Economics](../part-11-capacity-finops-and-reliability/087-gpu-cost-allocation-unit-economics.md)

**Status:** NOT EXECUTED. Offline ledger exercises, actual billing reconciliation, staging performance measurement, security/FinOps review and independent acceptance remain pending.

## 1. Safety and data handling

Use synthetic rates and workloads unless approved billing exports are available. Do not publish invoices, cost-center mappings, customer/model identifiers, API keys or provider account numbers. Do not change production labels, chargebacks, cloud commitments or serving replica counts as part of the lab.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | Cost taxonomy | cost pool inventory | Pending |
| E02 | Ownership mapping | UID-to-owner history | Pending |
| E03 | GPU-hours/idle | allocation worksheet | Pending |
| E04 | Invoice reconciliation | ledger and variance | Pending |
| E05 | Shared-cost allocation | approved allocation keys | Pending |
| E06 | Training cost/step | successful step denominator | Pending |
| E07 | Cost to target quality | convergence/quality gate | Pending |
| E08 | Cost/1M tokens | token definition | Pending |
| E09 | Cost/SLO request | eligible request denominator | Pending |
| E10 | Retry/checkpoint cost | attempt-level ledger | Pending |
| E11 | Commitment accounting | coverage and amortization | Pending |
| E12 | Realized savings | invoice vs freed capacity | Pending |
| E13 | Tenant showback | versioned report | Pending |
| E14 | Dispute correction | audit history | Pending |
| E15 | Alerts | anomaly/reconciliation rule | Pending |
| E16 | Independent review | score and risk register | Pending |

## 3. Offline training unit-economics exercise

\`\`\`python
gpus = 8
successful_hours = 4
failed_hours = 2
synthetic_rate_per_gpu_hour = 2.0
other_cost = 12.0
successful_steps = 12000

gpu_hours = gpus * (successful_hours + failed_hours)
total_cost = gpu_hours * synthetic_rate_per_gpu_hour + other_cost
cost_per_step = total_cost / successful_steps

print(gpu_hours, total_cost, cost_per_step)
assert gpu_hours == 48
assert total_cost == 108.0
assert cost_per_step == 0.009
\`\`\`

Expected: **48 GPU-hours, $108 synthetic total, $0.009 per successful step**. Rate is fictional, not a current cloud price.

## 4. Offline inference unit economics

\`\`\`python
synthetic_daily_cost = 600.0
successful_output_tokens = 120_000_000
cost_per_million = synthetic_daily_cost / successful_output_tokens * 1_000_000
print(cost_per_million)
assert cost_per_million == 5.0
\`\`\`

Expected: **$5 per million successful output tokens** using synthetic input. Report prompt tokens, cached tokens and output tokens separately in a real system.

## 5. Invoice-to-ledger reconciliation

\`\`\`python
invoice = 10_000.00
allocated = 7_500.00
shared = 1_500.00
unallocated = 900.00
variance = invoice - allocated - shared - unallocated
print("reconciliation_variance:", variance)
assert variance == 100.00
\`\`\`

Expected: **$100 unreconciled**. Do not silently spread this amount across tenants. Investigate credits, taxes, FX, time windows and missing workloads before closure.

## 6. Synthetic governance cases

**A — Fixed commitment:** workload releases 25% GPU-hours but bill is unchanged. Distinguish freed capacity from actual savings.

**B — Missing owner:** deleted namespace leaves GPU spend without a current owner. Use historical UID mapping and suspense pool.

**C — Retry inflation:** spot interruptions increase failed training attempts. Include retries in cost numerator.

**D — Serving p99 regression:** cheaper GPU configuration breaches latency SLO. Reject optimization despite lower hourly rate.

**E — Chargeback dispute:** tenant disputes shared storage cost. Reproduce the ledger version and obtain approved correction.

## 7. Evidence template

\`\`\`yaml
chapter: "087"
environment: "OFFLINE_OR_STAGING"
execution_date_utc: null
finops_reviewer: null
security_reviewer: null
billing_period: null
currency: null
rate_source: "SYNTHETIC"
cost_allocation_policy_version: null
invoice_reconciliation: "NOT_RUN"
unallocated_cost_percent: null
training_cost_per_step: null
inference_cost_per_million_tokens: null
slo_compliant_request_cost: null
commitment_review: "PENDING"
showback_review: "PENDING"
chargeback_authorization: "NOT_GRANTED"
staging_benchmark: "NOT_RUN"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 8. Acceptance

Score **20 + 20 + 20 + 15 + 15 + 10 = 100**. Pass at ≥85 with no critical invoice double-counting, unauthorized chargeback, privacy exposure, unsupported rates, false realized-savings claim or customer SLO/quality regression.

**Next:** Chapter 088 — Scaling and Queue Admission Control.
