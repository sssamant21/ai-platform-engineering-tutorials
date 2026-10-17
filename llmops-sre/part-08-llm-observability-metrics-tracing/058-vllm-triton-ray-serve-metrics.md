# Chapter 058 — vLLM, Triton and Ray Serve Metrics

**Part 08 — LLM Observability, Metrics and Tracing | Chapter 058/100**

> **Publication status:** Written tutorial only. Live execution, independent technical/security review and operational acceptance pending. Exported metric names and labels are **version-specific**. Never assume the illustrative queries work without inspecting actual targets.

## Learning outcomes

Build reliable observability across vLLM, NVIDIA Triton Inference Server and Ray Serve; distinguish inference metrics from GPU, Ray and Kubernetes metrics; normalize request, token, queue, latency and completion semantics; implement SLO dashboards, alerting, troubleshooting and evidence-based acceptance.

## 1. Architecture

```text
clients → ingress / auth / admission → inference router
                                          │
                       ┌──────────────────┼──────────────────┐
                       ▼                  ▼                  ▼
                     vLLM              Triton             Ray Serve
                  scheduler/KV       model/backend       replicas/queues
                       │                  │                  │
                  /metrics            /metrics       Ray/Serve metrics
                       └──────────────────┼──────────────────┘
                                          ▼
                         Prometheus scrape / OTel Collector
                                          │
                             normalization + recording rules
                                          │
                           SLO dashboards / alerts / runbooks
                                          │
                               GPU DCGM + K8s + FinOps
```

An inference gateway's request count is not necessarily equal to backend request count because of retries, routing and batching. Preserve layer and attempt semantics.

## 2. Metric source taxonomy

| Layer | Signal | Examples |
|---|---|---|
| gateway | admitted/rejected traffic | 429, 5xx, request duration |
| vLLM | scheduler/token workload | running/waiting requests, KV usage, TTFT, output tokens |
| Triton | model/backend execution | request count, queue/compute durations, execution count, pending requests |
| Ray Serve | deployment/router/replica | request counts, errors, latency, queued requests, replica health |
| DCGM | GPU hardware | VRAM, utilization, temperature, ECC/Xid |
| Kubernetes | workload and node | Pod restarts, Ready, GPU allocatable |
| synthetic client | user outcome | first token, terminal-valid completion, E2E |

Do not interpret Triton inference execution count as equivalent to client requests: dynamic batching and ensembles change the relationship.

## 3. vLLM observability

vLLM deployments may export `vllm:`-prefixed metrics, including request counts, TTFT/E2E histograms, running/waiting requests, token counts and KV-cache indicators. Names can change across engine generations and releases. Verify metric HELP/TYPE, histogram bucket schema, label sets and whether counters count requests, attempts or tokens.

Inspect authorized local endpoint:

```bash
# [SAFE-READ] run only against an approved internal metrics endpoint
curl -fsS --max-time 10 http://127.0.0.1:8000/metrics | grep -E '^(# HELP|# TYPE|vllm:)' | head -80
```

Port-forwarding requires authorization and an approved Pod/Service; never expose the metrics endpoint publicly.

## 4. Triton observability

Triton typically exposes Prometheus metrics for model requests, execution, queue/compute durations and memory/GPU telemetry when enabled. Investigate `nv_inference_request_success`, `nv_inference_request_failure`, `nv_inference_queue_duration_us`, `nv_inference_compute_infer_duration_us`, `nv_inference_exec_count` and `nv_gpu_` families **only if present**. Some duration families are counters in microseconds, not histograms. Backend configuration and server version affect availability.

```bash
# [SAFE-READ] approved internal endpoint, common Triton metrics port
curl -fsS --max-time 10 http://127.0.0.1:8002/metrics | grep -E '^(# HELP|# TYPE|nv_inference_|nv_gpu_)' | head -100
```

Compute mean queue duration only by dividing a matching duration counter's rate by the matching event counter rate; do not mix incompatible labels or assume request and execution cardinality match.

## 5. Ray Serve observability

Ray and Ray Serve export component and application metrics via Ray's metrics infrastructure. Discover the actual dashboard/metrics endpoint and check metrics such as `ray_serve_num_http_requests`, `ray_serve_num_http_error_requests`, `ray_serve_deployment_request_latency_ms`, `ray_serve_num_ongoing_requests_at_replicas` and `ray_serve_num_queued_queries` **when available**. Naming and aggregation differ across Ray releases and Serve ingress modes.

```bash
# [SAFE-READ] only against an approved Ray metrics endpoint
curl -fsS --max-time 10 http://127.0.0.1:8080/metrics | grep -E '^(# HELP|# TYPE|ray_serve_)' | head -100
```

The example port is illustrative; discover the real metrics endpoint before use.

## 6. Cross-platform normalization contract

Define a version-controlled mapping:

```yaml
# Design contract, not an executable exporter configuration
schema_version: 1
units:
  duration: seconds
  tokens: count
  queue_depth: requests
dimensions:
  - cluster
  - service
  - model
  - route
  - region
  - outcome
required:
  - admitted_requests_total
  - terminal_valid_completions_total
  - first_token_latency_seconds
  - e2e_latency_seconds
  - waiting_requests
  - input_tokens_total
  - output_tokens_total
```

Every mapping needs source metric, version, units, event boundary, label mapping, cardinality limit, freshness and test evidence. A platform lacking a native signal must use an approved gateway or synthetic-client instrument; **do not fabricate it**.

## 7. Metrics that cannot be equated blindly

- Triton request duration may cover backend execution rather than end-user E2E.
- Ray Serve request latency may reflect HTTP or deployment processing depending on instrumentation.
- vLLM TTFT is generally a server-side signal, not client-observed TTFT.
- Request counters can count retries, internal invocations or batched executions.
- Queue depth snapshots cannot be compared directly with total queued request counters.
- GPU utilization does not establish terminal-valid goodput.

## 8. PromQL examples [NORMALIZED METRICS REQUIRED]

```promql
sum by (platform, model) (
  rate(llm_admitted_requests_total[5m])
)
```

```promql
histogram_quantile(
  0.95,
  sum by (platform, model, le) (
    rate(llm_first_token_latency_seconds_bucket[5m])
  )
)
```

```promql
sum by (platform, model) (
  rate(llm_terminal_valid_completions_total[5m])
)
```

```promql
sum by (platform, model) (llm_waiting_requests)
```

These are **proposed normalized metric names**, not built-in exporter names. Only use after validating mapping and scrape health.

## 9. Histogram and unit correctness

Check seconds versus milliseconds versus microseconds, bucket coverage, counter resets, duplicate scrapes, classic versus native histograms and p95/p99 sample counts. Avoid converting cumulative duration counters into percentile estimates. Use a normalized duration histogram only when real observations support it.

## 10. Completion semantics

Define terminal-valid completion at the gateway/client boundary. A 200 response may start a stream that later truncates. Track explicit cancellation, timeouts, partial streams, invalid schema/tool output, retry attempts and request deduplication. Server metrics alone often cannot prove terminal-valid completion.

## 11. Labels and cardinality

Use bounded `platform`, `cluster`, `service`, `model`, `route`, `region`, `outcome` and `version` where appropriate. Do not use prompt, output, raw tenant ID, session ID, request ID or user ID as Prometheus labels. Separate restricted tracing for request-level investigation.

## 12. Metrics security

Metrics endpoints can reveal infrastructure, model names and traffic volumes. Enforce network restrictions, mTLS/authentication where supported, workload identity, tenant isolation and least-privilege scrape permissions. Redact sensitive labels; set retention and access audit policies.

## 13. Dashboards

Build a platform comparison dashboard with per-model request rate, terminal-valid completion, TTFT and E2E p95, queue depth, input/output token rate, error taxonomy, replica health, GPU memory/utilization and deployment markers. Add separate platform-native panels for vLLM KV, Triton backend queue/compute and Ray Serve router/replica saturation.

## 14. SLO and alerting

Page on customer-impacting terminal-valid availability or sustained latency burn, not isolated high GPU utilization. Alert on missing critical scrape targets, stalled counters under confirmed traffic, sustained queues with degraded TTFT, model-unready replicas and error spikes. Link each alert to its platform-specific runbook.

## 15. Capacity and FinOps

Normalize GPU-hours and terminal-valid output tokens per platform/model/SKU. Account for idle warm pools, dynamic batching, replica overhead, model caching, cross-zone traffic and metrics ingestion cost. Benchmark with matched request distributions; do not compare raw tokens/sec across different models as if equivalent.

## 16. Safe Kubernetes discovery

```bash
date -u
kubectl config current-context
kubectl get pods -A -o wide
kubectl get services -A
kubectl get servicemonitors -A 2>/dev/null || true
kubectl get podmonitors -A 2>/dev/null || true
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n APPROVED_NS describe pod APPROVED_INFERENCE_POD
```

Read-only discovery is preferred. Verify monitoring CRDs exist; a missing CRD is not evidence of missing telemetry.

## 17. Troubleshooting decision tree

```text
Missing or inconsistent inference metrics?
 ├─ endpoint unreachable → service/network/auth
 ├─ target DOWN → scrape config/TLS/RBAC
 ├─ exporter present, series absent → feature/version/traffic
 ├─ latency differs by platform → boundaries/units
 ├─ request counts differ → retries/batching/layer
 ├─ GPU metrics missing → DCGM/driver/MIG
 ├─ queue high, TTFT normal → sampling/label mismatch
 ├─ p99 nonsensical → buckets/sample count
 ├─ completion inflated → partial streams/dedup
 └─ cardinality spike → unsafe labels/rollout
```

## 18. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | vLLM metrics disabled | endpoint |
| F02 | vLLM metric renamed after upgrade | HELP/TYPE |
| F03 | vLLM KV cache signal absent | version |
| F04 | vLLM TTFT boundary differs from client | traces |
| F05 | Triton metrics endpoint inaccessible | scrape |
| F06 | Triton microseconds read as seconds | units |
| F07 | Triton exec count treated as requests | batching |
| F08 | Triton ensemble double counting | model labels |
| F09 | Ray Serve endpoint wrong | config |
| F10 | Ray Serve metric renamed | HELP/TYPE |
| F11 | Ray router and replica counters duplicated | labels |
| F12 | Ray request latency mistaken for TTFT | semantics |
| F13 | gateway retries inflate backend traffic | correlation |
| F14 | 200 stream truncation counted success | terminal |
| F15 | exporter missing after rollout | targets |
| F16 | Prometheus duplicate scraping | target labels |
| F17 | histogram units/buckets wrong | p99 |
| F18 | counter/gauge type mismatch | TYPE |
| F19 | high-cardinality model/tenant labels | TSDB |
| F20 | sensitive prompt in telemetry | privacy |
| F21 | DCGM MIG GPU labels mismatched | topology |
| F22 | stale scrape masks saturation | freshness |
| F23 | alert pages on GPU usage alone | alerts |
| F24 | telemetry cost exceeds budget | FinOps |

## 19. Sixteen operational runbooks

1. Inventory vLLM/Triton/Ray Serve metrics endpoints.
2. Recover vLLM metric exposure and schema drift.
3. Validate vLLM TTFT, queue and KV metrics.
4. Recover Triton scrape target and authentication.
5. Correct Triton duration units and execution counts.
6. Diagnose Triton batching/ensemble metric semantics.
7. Recover Ray Serve metrics endpoint and target.
8. Reconcile Ray router/replica counters.
9. Normalize gateway/backend retries and attempts.
10. Restore terminal-valid completion accounting.
11. Repair Prometheus scrape deduplication.
12. Validate histogram buckets, units and resets.
13. Restore GPU/DCGM/MIG correlation.
14. Contain cardinality and telemetry privacy exposure.
15. Restore SLO alerts, dashboards and on-call links.
16. Execute schema migration rollback and cost reconciliation.

## 20. Eight tabletop game days

1. vLLM upgrade silently renames a TTFT metric.
2. Triton microsecond duration interpreted as seconds.
3. Triton batching makes execution count look like lost requests.
4. Ray Serve router and replica request counts are summed twice.
5. Gateway retry storm inflates backend errors.
6. Prometheus scrape duplication doubles throughput.
7. Metrics labels leak tenant identifiers.
8. Cross-platform dashboard reports healthy while streams truncate.

Tabletop by default; live injection requires isolated staging approval.

## 21. Twenty acceptance gates

G01 environment/versions; G02 endpoint inventory; G03 scrape health; G04 vLLM scheduler/KV; G05 vLLM latency; G06 Triton request/exec; G07 Triton queue/compute units; G08 Ray router/replica; G09 Ray latency semantics; G10 gateway attempts; G11 terminal validity; G12 token accounting; G13 histogram accuracy; G14 PromQL/dedup; G15 GPU/DCGM correlation; G16 labels/privacy; G17 SLO alerts; G18 rollback/schema drift; G19 telemetry FinOps; G20 independent review.

See [companion lab](../labs/058-vllm-triton-ray-serve-metrics-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 059 — GPU DCGM, Node and Kubernetes Observability.**
