# Lab 083 — GPU Demand Forecasting Acceptance Gate

**Chapter:** [083 — GPU Demand Forecasting](../part-11-capacity-finops-and-reliability/083-gpu-demand-forecasting.md)

**Status:** NOT EXECUTED. Offline forecasting exercises, staging/scheduler validation and independent review remain pending.

## 1. Safety and prerequisites

Use synthetic workload data or approved, redacted historical metrics. Do not change production GPU quotas, scale node pools, purchase reservations, drain nodes or preempt training jobs as part of the lab. Financial values are placeholders until actual contract rates are supplied.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | GPU SKU/failure-domain inventory | node/SKU/MIG matrix | Pending |
| E02 | Request vs utilization reconciliation | coverage report | Pending |
| E03 | GPU-hour dataset | versioned CSV | Pending |
| E04 | Average vs peak concurrency | arithmetic | Pending |
| E05 | Moving/seasonal baseline | forecast output | Pending |
| E06 | Driver-based forecast | assumptions and roadmap | Pending |
| E07 | Quantile uncertainty | p50/p90/p95 | Pending |
| E08 | Backtest | MAE/WAPE/coverage | Pending |
| E09 | Retry inflation | sensitivity | Pending |
| E10 | Node bundle rounding | feasible node count | Pending |
| E11 | Gang scheduling | queue/deadline model | Pending |
| E12 | Inference replica sizing | measured SLO throughput | Pending |
| E13 | Maintenance/AZ-loss headroom | resilience scenario | Pending |
| E14 | FinOps scenarios | contract-rate placeholders | Pending |
| E15 | Capacity trigger alert | routing and lead time | Pending |
| E16 | Peer review | score and risk register | Pending |

## 3. Offline capacity math lab

\`\`\`python
from math import ceil

peak_gpu = 60
node_gpu = 8
maintenance = 0.05
headroom = 0.15

required_gpu = ceil(peak_gpu / ((1-maintenance)*(1-headroom)))
required_nodes = ceil(required_gpu / node_gpu)
installed_gpu = required_nodes * node_gpu
usable_gpu = installed_gpu * (1-maintenance)*(1-headroom)

print(required_gpu, required_nodes, installed_gpu, round(usable_gpu, 2))
assert (required_gpu, required_nodes, installed_gpu) == (75, 10, 80)
assert usable_gpu >= peak_gpu
\`\`\`

Expected output: \`75 10 80 64.6\`. This is a simplified fleet model, **not** a proof of gang scheduling feasibility.

## 4. Synthetic GPU-hour calculation

\`\`\`python
jobs = [
    {"gpus": 8, "hours": 4, "attempts": 1},
    {"gpus": 2, "hours": 6, "attempts": 1},
    {"gpus": 16, "hours": 3, "attempts": 2},
]
gpu_hours = sum(j["gpus"] * j["hours"] * j["attempts"] for j in jobs)
print("GPU-hours:", gpu_hours)
assert gpu_hours == 140
\`\`\`

Assumes each retry has the same runtime. For real workloads use measured attempt-level GPU-hours.

## 5. Synthetic scenarios

**A — Peak concurrency:** 128-GPU training job for two hours vs 16-GPU fleet. Explain why daily GPU-hours do not ensure schedulability.

**B — Queue fragmentation:** 8-GPU node bundles split by incompatible MIG profiles. Identify placement failure.

**C — Inference burst:** requests/sec rises 3× and model context length doubles. Demand must be measured against p99-compliant per-GPU throughput, not scaled linearly without testing.

**D — Supply shock:** one AZ unavailable and cloud quota request pending. Model reserve capacity and lead time.

**E — FinOps:** reserved GPUs idle while spot jobs are interrupted; calculate useful work per dollar from approved rates.

## 6. Evidence template

\`\`\`yaml
chapter: "083"
environment: "OFFLINE_OR_STAGING"
execution_date_utc: null
reviewer: null
dataset_version: null
data_coverage_percent: null
gpu_sku_mix: []
forecast_horizon_days: null
forecast_method: null
forecast_mae: null
forecast_wape: null
p90_coverage: null
peak_concurrent_gpu: null
required_nodes: null
maintenance_fraction: null
headroom_fraction: null
inference_slo_test: "NOT_RUN"
queue_simulation: "NOT_RUN"
capacity_alert_delivery: "NOT_RUN"
finops_rate_source: "NOT_SUPPLIED"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 7. Acceptance

Score **20 + 20 + 20 + 15 + 15 + 10 = 100**. Pass at ≥85 with no critical data privacy, invalid GPU interchangeability, unsafe procurement assumption or unapproved production change. Written forecasts are not evidence of actual staging execution.

**Next:** Chapter 084 — GPU Rightsizing and Utilization.
