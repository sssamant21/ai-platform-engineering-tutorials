# Chapter 029 — Triton GPU Metrics and Model Performance

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 029 of 100**

> **Status:** Written tutorial published. Live labs, independent technical/security review and operational acceptance pending. Metric names, units and availability depend on pinned Triton, backend, GPU and exporter versions.

## Learning objectives

Build a production observability and performance methodology for Triton inference: request/queue/compute metrics, GPU telemetry, batch efficiency, model-version SLIs, HTTP/gRPC and decoupled-response correctness, Prometheus/Grafana/OTel correlation, performance experiments, SLO burn alerts and FinOps.

## 1. Observability architecture

```text
Synthetic/production client
      │ end-to-end latency and valid completion
      ▼
Authenticated gateway / ingress
      │ HTTP/gRPC errors, retry, quota
      ▼
Triton front end + model scheduler
      │ queue time, request and batch counts
      ▼
Model instance + backend
      │ compute input / infer / output
      ▼
GPU / CPU / memory / device runtime
      │ DCGM, node metrics, GPU faults
      ▼
Prometheus / OTel / structured logs
      │
Grafana + alert routing + SLO burn + FinOps
```

Use model name/version and bounded workload classes for aggregation. Do not put request IDs, user IDs, tensor values or raw prompts in metric labels.

## 2. Model performance golden signals

| Signal | Interpretation | Caution |
|---|---|---|
| completed/failed inference requests | serving success | distinguish client vs backend |
| queue duration | waiting before dispatch | not the same as compute |
| compute input | input preparation/copy | may depend on backend |
| compute infer | execution duration | not full E2E |
| compute output | output handling | may omit gateway time |
| request latency p50/p95/p99 | customer experience | client vs server boundary |
| execution and batch size | scheduler efficiency | request vs execution count |
| model load/unload | availability/cold start | process ready ≠ model ready |
| GPU utilization and VRAM | device pressure | high util not always bad |
| XID/ECC, throttling, power | hardware health | supported metric coverage varies |
| valid completions/GPU-hour | FinOps | exclude invalid partial responses |

Triton reports model/backend metrics according to the pinned version. Do not assume every backend emits all fields.

## 3. Metric semantics: request, execution and batch

A request is not necessarily one backend execution. With batching, several requests may be executed together; for decoupled models, a request may yield multiple responses. Count eligible requests and valid completed responses according to the client contract.

```text
request rate = valid eligible requests / interval
execution rate = backend executions / interval
batch efficiency = requests executed / backend executions
```

Batch efficiency is an aggregate ratio, not proof of improved p99 or correctness. Record input shapes, concurrency and model versions.

## 4. Triton Prometheus endpoint

Triton commonly exposes a Prometheus metrics endpoint, often on a dedicated port such as 8002, but ports and exposure are configurable. The endpoint may contain model metadata and must remain internal.

For an **approved localhost port-forward** to the metrics port:

```bash
kubectl -n MODEL_NAMESPACE port-forward svc/APPROVED_TRITON_METRICS_SERVICE 18002:8002
```

Then, if the pinned runtime serves the endpoint:

```bash
curl --fail --silent --show-error http://127.0.0.1:18002/metrics | head -100
```

Use an approved Service/port, localhost only, and close the tunnel. Inspect `# HELP` and `# TYPE` to confirm counter/gauge/histogram semantics, labels and units.

## 5. Representative metric families

Common Triton metric families in supported releases include `nv_inference_request_success`, `nv_inference_request_failure`, `nv_inference_count`, `nv_inference_exec_count`, `nv_inference_request_duration_us`, `nv_inference_queue_duration_us`, `nv_inference_compute_input_duration_us`, `nv_inference_compute_infer_duration_us` and `nv_inference_compute_output_duration_us`. These are **representative**, not guaranteed across all releases/backends/configurations.

Many `*_duration_us` metrics are cumulative microsecond counters, **not latency histograms**. Dividing a duration rate by a corresponding request or execution rate can yield an average, not a p95/p99. Use verified Triton histograms if enabled/supported, client-side histograms or tracing for tail latency.

## 6. Correct PromQL patterns

Illustrative queries **only after validating actual metric names, labels and types**:

```promql
# Model-level successful request rate
sum by (model) (rate(nv_inference_request_success[5m]))

# Mean queue time in seconds, when duration and request counters align
sum by (model) (rate(nv_inference_queue_duration_us[5m]))
/
clamp_min(sum by (model) (rate(nv_inference_count[5m])), 1e-9)
/
1e6

# Approximate requests per backend execution
sum by (model) (rate(nv_inference_count[5m]))
/
clamp_min(sum by (model) (rate(nv_inference_exec_count[5m])), 1e-9)
```

Check numerator/denominator semantics, zero traffic, failed requests, counter resets, labels and whether durations include canceled or rejected requests. Do not label these average ratios as p99.

For an actual classic histogram exported with `_bucket`:

```promql
histogram_quantile(
  0.99,
  sum by (le, model) (rate(<verified_latency_bucket>[5m]))
)
```

The placeholder must be replaced with a verified metric. Native histograms require their supported query form.

## 7. GPU telemetry with DCGM

Combine Triton metrics with NVIDIA DCGM exporter where supported:
- GPU memory used/free and utilization;
- SM activity and memory bandwidth proxies;
- power, temperature and throttle reasons;
- XID/ECC errors and device health;
- per-device UUID, node and Pod attribution where available.

Not every GPU/exporter exposes every field. A missing metric is unknown, not zero. Validate exporter freshness and GPU identity before correlating with a model instance.

## 8. Model and GPU attribution

A Pod may host multiple models, model versions and instance groups. GPU utilization measured per device cannot automatically be attributed to one model. Correlate:
1. model request/execution rates;
2. model instance placement;
3. GPU UUID and device metrics;
4. workload isolation windows;
5. shared-device contention and background work.

Use isolated benchmarks to estimate model-specific GPU efficiency; avoid false per-model cost precision in shared environments.

## 9. Performance experiment methodology

Record a reproducible test manifest:

```yaml
model: approved_synthetic_model
model_version: pinned
triton_image_digest: pinned
backend_version: pinned
gpu_sku: recorded
gpu_uuid: recorded
request_protocol: http_or_grpc
input_shape_cohorts: [small, medium, large]
concurrency_levels: [1, 4, 8]
warmup: defined
duration_seconds: bounded
gpu_hour_ceiling: approved
```

Use a supported benchmarking client such as Triton Performance Analyzer only after checking its version and flags. Avoid invented universal command options. Hold model, backend, GPU, concurrency, warmup and input distribution constant when comparing changes.

## 10. Queue, compute and latency diagnosis

```text
E2E latency regression?
 ├─ gateway time high → TLS/auth/routing/network
 ├─ Triton queue high → batching, instances, admission
 ├─ compute input high → preprocessing/transfers
 ├─ compute infer high → model/GPU/backend
 ├─ compute output high → output size/transfers
 ├─ GPU util high + p99 high → saturation or poor batching
 ├─ GPU util low + queue high → scheduler/model availability
 ├─ model load slow → artifact store/backend init
 ├─ gRPC errors → protocol/deadline/config
 ├─ decoupled partial response → completion contract
 └─ metrics flatline → scrape/exporter freshness
```

Do not assume that component timings add exactly to E2E: concurrency, sampling boundaries and transport can create gaps or overlaps.

## 11. SLOs and error budgets

Example **synthetic** model success SLO:

```text
Target = 99.9% valid completions
Eligible requests = 200,000
Failed/incomplete = 500
Allowed failures = 200
Observed success = 99.75%
Burn rate = (500/200,000) / 0.001 = 2.5×
```

For decoupled/streaming models, define terminal completion at the client, not merely HTTP 200. Build multi-window burn alerts with low-volume guards, missing-data alerts and distinct 429/5xx classifications.

## 12. Grafana dashboards

Recommended panels:
1. valid request rate and complete response success;
2. p50/p95/p99 client E2E;
3. model request failures and error taxonomy;
4. average queue/compute input/infer/output (correct units);
5. batch efficiency and backend execution rate;
6. model load time/readiness;
7. GPU VRAM, utilization, power, temperature and faults;
8. Pod restarts, device-plugin health and scheduling;
9. scrape freshness and trace/log ingestion;
10. SLO burn, GPU-hours and cost per valid completion.

## 13. Security and telemetry cost

Restrict metrics to trusted observability networks. Scrub tensor payloads, authorization tokens and user identifiers from logs/traces. Bound label cardinality, trace sampling, log retention and scrape interval. Audit changes that expose model metadata or administrative endpoints.

## 14. Read-only checks

```bash
date -u
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

For an approved deployment:

```bash
kubectl -n MODEL_NAMESPACE logs deployment/TRITON_DEPLOYMENT --since=15m --tail=100
kubectl -n MODEL_NAMESPACE describe deployment TRITON_DEPLOYMENT
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

Redact logs and never dump Secrets or customer tensors.

## 15. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | metrics endpoint unreachable | scrape target |
| F02 | metric renamed on upgrade | schema diff |
| F03 | duration unit misread as seconds | HELP/type |
| F04 | cumulative duration treated as p99 | query audit |
| F05 | counter reset causes false spike | restart |
| F06 | stale scrape treated as zero | freshness |
| F07 | excessive model-version cardinality | TSDB |
| F08 | missing GPU UUID mapping | device inventory |
| F09 | DCGM exporter incompatible | exporter logs |
| F10 | GPU XID/ECC metric absent | support matrix |
| F11 | model shared GPU attribution wrong | placement |
| F12 | queue saturation | queue duration |
| F13 | dynamic batching p99 regression | client histogram |
| F14 | backend compute regression | compute duration |
| F15 | input/output transfer bottleneck | stage metrics |
| F16 | model cold start latency | readiness |
| F17 | gRPC deadline exceeded | client/gateway |
| F18 | decoupled response incomplete | terminal validator |
| F19 | low sample count p99 noise | volume |
| F20 | SLO burn alert missing | alert test |
| F21 | OTel collector drops spans | collector metrics |
| F22 | sensitive tensors in logs | privacy audit |
| F23 | GPU cost rises despite throughput | FinOps |
| F24 | benchmark workload not representative | cohort manifest |

## 16. Sixteen operational runbooks

1. Inventory pinned Triton/backend/metric schema.
2. Diagnose metrics endpoint and scrape freshness.
3. Validate metric units/types and counter resets.
4. Correct average versus percentile queries.
5. Diagnose label cardinality and retention.
6. Restore DCGM exporter and GPU mapping.
7. Investigate GPU faults and throttling.
8. Investigate queue saturation and batching.
9. Investigate compute input bottlenecks.
10. Investigate compute inference bottlenecks.
11. Investigate compute output bottlenecks.
12. Diagnose model cold start/readiness.
13. Diagnose gRPC and decoupled completion.
14. Validate SLO burn and alert delivery.
15. Recover tracing/log privacy and telemetry cost.
16. Execute performance rollback and RCA.

## 17. Eight tabletop game days

1. Upgrade renames a critical Triton metric.
2. A microsecond duration is charted as seconds.
3. Cumulative average is mislabeled p99.
4. DCGM exporter fails during GPU XID incident.
5. Batching increases p99 while mean improves.
6. Decoupled model returns incomplete response.
7. Prometheus scrapes stop during an outage.
8. GPU cost doubles after a throughput optimization.

Default tabletop; live failure injection requires separately approved isolated staging.

## 18. Twenty acceptance gates

G01 pinned schema; G02 metrics reachability; G03 freshness; G04 units/types; G05 average vs percentile; G06 request/execution semantics; G07 queue metrics; G08 compute metrics; G09 batch efficiency; G10 GPU/DCGM; G11 device attribution; G12 client E2E; G13 valid completion; G14 SLO burn; G15 missing-data alert; G16 privacy; G17 cardinality/cost; G18 reproducible benchmark; G19 rollback; G20 independent review.

See [companion lab](../labs/029-triton-gpu-metrics-model-performance-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 030 — Triton Kubernetes Deployment and Operations.**
