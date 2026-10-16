# Chapter 029 — Hands-On Lab: Triton GPU Metrics and Model Performance

**Part 05 | Chapter 029 | Published lab plan; live execution pending**

## Goal

Inventory Triton Prometheus metrics, validate duration semantics, correlate model queue/compute with GPU telemetry, design reproducible benchmarks and create SLO, dashboard, privacy and FinOps acceptance evidence.

## Safety

**[SAFE-READ]** authorized metrics, GPU inventory and redacted logs. **[LAB-MUTATE]** isolated staging with synthetic models, approved concurrency/duration and GPU-hour ceiling. **[CHANGE-CONTROL]** production scrape, metrics exposure, alerts, benchmark load and model/runtime changes. No PHI, customer tensors or secrets.

## Lab 0 — Preflight

Record UTC, cluster/context, Triton image/backend versions, model revision, metrics endpoint, Prometheus/OTel/DCGM versions, GPU SKU/UUID, owner, approval, cost ceiling and rollback.

```bash
date -u
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Metric schema verification [SAFE-READ; authorized port-forward]

```bash
kubectl -n MODEL_NAMESPACE port-forward svc/APPROVED_TRITON_METRICS_SERVICE 18002:8002
```

In another terminal:

```bash
curl --fail --silent --show-error http://127.0.0.1:18002/metrics | head -100
```

Use localhost only, confirm the Service and port, close promptly and redact output. Record HELP/TYPE, units, labels, freshness and counter semantics.

## Lab 2 — Correct PromQL [DESIGN]

If the pinned exporter exposes matching metric families:

```promql
sum by (model) (rate(nv_inference_request_success[5m]))
```

For cumulative queue duration in microseconds, design a mean queue-time query using the matching request counter and divide by 1e6. Do **not** present the result as p95/p99. Use verified histograms or client measurements for tail latency.

## Lab 3 — GPU correlation [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

Map device UUID to Pod/model instances and DCGM metrics where supported. Shared GPU usage cannot automatically be attributed to a single model.

## Lab 4 — Performance experiment matrix [DESIGN]

| Variant | Concurrency | Input cohort | Measure |
|---|---:|---|---|
| A | 1 | small | baseline |
| B | 4 | small | queue/compute |
| C | 8 | small | saturation |
| D | 4 | large | shape sensitivity |
| E | 4 | mixed | representative load |

Hold runtime/model/GPU/warmup/duration constant. Record valid completions, client E2E p95/p99, request/execution rate, batch efficiency, GPU memory and GPU-hours.

## Lab 5 — SLO worksheet [DESIGN]

```text
Target valid completion = 99.9%
Eligible = 200,000
Failed/incomplete = 500
Allowed = 200
Observed success = 99.75%
Burn = 2.5×
```

Define eligibility, partial decoupled responses, low-traffic safeguards, multi-window alerts and missing-scrape detection.

## Lab 6 — Privacy and FinOps [DESIGN]

Audit model/version label cardinality, raw tensors in logs/traces, gateway auth, metrics exposure, retention and cost. Compute GPU-hours per valid completed request; distinguish shared-device attribution estimates from measured isolated workloads.

## Lab 7 — Eight tabletop game days

01 metric rename; 02 unit error; 03 average mistaken for p99; 04 DCGM outage; 05 batching tail regression; 06 incomplete decoupled response; 07 stale scrape; 08 GPU cost spike.

## Twenty-four failure checklist

F01 endpoint; F02 rename; F03 units; F04 p99 confusion; F05 resets; F06 stale; F07 cardinality; F08 GPU mapping; F09 DCGM; F10 XID/ECC coverage; F11 attribution; F12 queue; F13 batching; F14 backend; F15 transfers; F16 cold start; F17 gRPC; F18 decoupled completion; F19 low sample; F20 SLO alert; F21 OTel; F22 privacy; F23 FinOps; F24 benchmark bias.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned schema | Pending |
| G02 | Metrics reachability | Pending |
| G03 | Freshness | Pending |
| G04 | Units/types | Pending |
| G05 | Average vs percentile | Pending |
| G06 | Request/execution semantics | Pending |
| G07 | Queue metrics | Pending |
| G08 | Compute metrics | Pending |
| G09 | Batch efficiency | Pending |
| G10 | GPU/DCGM | Pending |
| G11 | Device attribution | Pending |
| G12 | Client E2E | Pending |
| G13 | Valid completion | Pending |
| G14 | SLO burn | Pending |
| G15 | Missing-data alert | Pending |
| G16 | Privacy | Pending |
| G17 | Cardinality/cost | Pending |
| G18 | Reproducible benchmark | Pending |
| G19 | Rollback | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Triton/backend/model versions:
Metric names/types/units/freshness:
Request/execution/batch:
Queue/compute and client E2E:
GPU/DCGM/UUID:
SLO/alert and missing-data:
Benchmark cohorts/valid completions:
Privacy/cardinality/retention:
GPU-hours/cost:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Close port-forwards; stop separately approved staging load, restore authoritative observability configuration and reconcile GPU-hours. Gates remain pending until independently reviewed.

**Next: Chapter 030 — Triton Kubernetes Deployment and Operations.**
