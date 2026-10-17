# Chapter 056 — Golden Signals for LLM Inference

**Part 08 — LLM Observability, Metrics and Tracing | Chapter 056/100**

> **Status:** Written tutorial published. Live lab execution, independent technical/security review and operational acceptance pending. Metric names, histogram labels, PromQL and tracing support vary across vLLM, Triton, KServe, Ray Serve and gateway versions. Do not change production instrumentation or logging without approval.

## Learning objectives

Translate the four golden signals into LLM-specific user and infrastructure telemetry; measure TTFT, TPOT, ITL, E2E latency, terminal-valid completions, token throughput, queueing, saturation and GPU cost; design privacy-safe Prometheus/OTel dashboards and actionable SLO alerts.

## 1. Reference observability architecture

```text
Client → API gateway → inference router → model server → GPU
            │              │              │         │
       auth / 429s       queue depth    token events  DCGM
            └──────────────┼──────────────┼──────────┘
                           ▼
                   Prometheus + OTel
                           │
                 metrics/traces/logs
                           │
                recording rules / SLOs
                           │
            dashboards → alerting → runbooks
                           │
                  incident + FinOps
```

Correlate request IDs across layers without logging raw prompts, completions, credentials or PHI.

## 2. Four golden signals adapted to inference

| Signal | LLM-specific indicators | Typical interpretation |
|---|---|---|
| Latency | TTFT, TPOT, ITL, E2E, queue wait | user-perceived speed |
| Traffic | requests/sec, input/output tokens/sec, active streams | offered and completed work |
| Errors | 5xx, 429, CUDA OOM, timeout, truncated streams, invalid completions | correctness and availability |
| Saturation | waiting requests, KV-cache utilization, GPU VRAM, memory bandwidth, GPU allocatable | bottlenecks and capacity |

GPU utilization alone is not a reliable saturation or user-experience metric.

## 3. Request lifecycle and timing

```text
request received → admission → queue → prefill
   → first token → token 2 → ... → final token
   → terminal completion / cancellation / error
```

Define timestamps at a consistent measurement boundary. **TTFT** = first-token time minus request acceptance (or documented client-send time). **E2E** = terminal response minus acceptance. **ITL** is the interval between successive streamed tokens. **TPOT** may be defined as average post-first-token generation time per output token; explicitly document denominator and whether first token is excluded. Different exporters use different definitions.

## 4. Terminal-valid completion

A request is terminal-valid only if it reaches a valid terminal state, conforms to API/schema requirements and is not silently truncated. Distinguish successful completion, explicit user cancellation, server cancellation, deadline exceeded, partial stream and transport disconnect. Define SLI denominator and cancellation policy with product owners.

## 5. Latency SLOs

Measure p50/p95/p99 TTFT, p95 TPOT/ITL and E2E by model, route, request class and region. Use server-side histograms where possible; validate bucket coverage, counter monotonicity and sparse traffic. Tail latency is more important than averages for user experience.

## 6. Traffic and token work

Count admitted requests, completed requests, prompt tokens, generated tokens, requests in flight, streaming sessions and queue depth. Separate **offered load** from **terminal-valid goodput**. Long-context prefill and decode demand should not be treated as identical work units.

## 7. Error taxonomy

| Category | Examples | Response |
|---|---|---|
| admission | quota 429, model unavailable | policy/capacity |
| client | invalid request, disconnect | distinguish from server errors |
| server | 5xx, crash, OOM | incident |
| streaming | interrupted or truncated output | terminal-state validation |
| model | invalid schema/tool output | correctness SLI |
| dependency | object store/KMS/network | startup/availability |

Do not label every 4xx as server failure or every HTTP 200 as successful streaming completion.

## 8. Saturation telemetry

Capture GPU SM/memory utilization, VRAM allocated/free, KV-cache occupancy, prefill/decode queue lengths, active sequences, batch sizes, CPU memory, disk/network bandwidth, device Xid/ECC and node GPU allocatable. Use NVIDIA DCGM exporter where supported; MIG/time-slicing labels and cardinality need validation.

## 9. Metric naming and labels

Prefer stable, low-cardinality labels such as model ID (bounded), service, cluster, region, status class and route. Avoid request ID, user ID, raw prompt, session ID, token text or unbounded tenant identifiers as Prometheus labels. Tenant drill-down may require controlled exemplars or separate secured analytics.

## 10. Illustrative metric contract [DESIGN]

```text
llm_requests_total{model,route,outcome}
llm_terminal_valid_completions_total{model,route}
llm_ttft_seconds_bucket{model,route,le}
llm_e2e_seconds_bucket{model,route,le}
llm_output_tokens_total{model,route}
llm_input_tokens_total{model,route}
llm_waiting_requests{model,route}
llm_active_sequences{model,route}
llm_kv_cache_utilization_ratio{model}
```

These names are **proposed canonical recording/normalization metrics**, not guaranteed native vLLM/Triton metrics. Map and test actual exporters before deploying queries.

## 11. PromQL examples [REQUIRES NORMALIZED METRICS]

```promql
sum by (model) (rate(llm_requests_total[5m]))
```

```promql
sum by (model) (rate(llm_terminal_valid_completions_total[5m]))
```

```promql
histogram_quantile(
  0.95,
  sum by (model, le) (rate(llm_ttft_seconds_bucket[5m]))
)
```

```promql
sum by (model) (llm_waiting_requests)
```

Do not sum a per-replica gauge across duplicate scrape targets without deduplication. Handle absent metrics, zero traffic and reset behavior explicitly.

## 12. OpenTelemetry trace model

Create spans for gateway admission, queue, prefill, first token, decode, streaming and finalization. Record sanitized model/version, token counts, queue duration, status and bounded error categories. Sampling must preserve slow/error traces while protecting privacy and limiting cost. Never attach raw prompt/response by default.

## 13. Dashboard design

A production inference dashboard should display: traffic and token mix; TTFT/TPOT/E2E percentiles; queue depth; terminal-valid completion and 429/5xx; KV/VRAM/GPU utilization; node/Pod readiness; per-model/region breakdown; deployment markers; GPU-hours and cost per valid completion. Pair symptom panels with likely causes.

## 14. Alerting and error budgets

Use multi-window burn-rate alerts for terminal-valid availability and sustained latency SLO violations, plus symptom-linked saturation alerts. Avoid paging solely on high GPU utilization. Route model-serving pages to SRE/on-call with runbook links; keep capacity/cost warnings distinct from customer-impacting pages.

## 15. SLO math

```text
terminal-valid availability =
  terminal-valid completions / eligible admitted requests

error budget = (1 - SLO target) × eligible requests
```

Example: 99.9% objective over 100,000 eligible requests permits at most 100 non-valid completions. Document exclusions and how cancellations/partial streams are counted.

## 16. Safe-read diagnostics

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

Inspect Prometheus/OTel targets only with authorized access. Avoid dumping model payloads or sensitive environment variables.

## 17. Troubleshooting decision tree

```text
Inference experience degrades?
 ├─ TTFT high → admission/queue/prefill/model warmup
 ├─ TPOT/ITL high → decode/KV/GPU contention
 ├─ E2E high → long output or queue
 ├─ valid completion low → partial streams/errors
 ├─ 429 high → tenant limits/capacity
 ├─ GPU high but SLO good → no incident by itself
 ├─ GPU low but queue high → routing/scheduler bottleneck
 ├─ metrics absent → exporter/scrape/labels
 └─ trace gaps → propagation/sampling/privacy
```

## 18. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | TTFT measurement boundary inconsistent | instrumentation |
| F02 | TPOT denominator incorrect | metric spec |
| F03 | ITL omitted for streaming | trace |
| F04 | E2E excludes queue wait | timestamps |
| F05 | HTTP 200 masks truncated stream | terminal state |
| F06 | user cancellation counted as 5xx | taxonomy |
| F07 | 429 hidden from error dashboard | gateway |
| F08 | CUDA OOM misclassified | logs |
| F09 | long-context work mixed with short | token histograms |
| F10 | queue depth missing | exporter |
| F11 | KV-cache utilization absent | runtime |
| F12 | GPU utilization mistaken for goodput | SLO |
| F13 | DCGM MIG labels inaccurate | exporter |
| F14 | duplicate scrapes inflate counters | targets |
| F15 | counter used as gauge | PromQL |
| F16 | histogram buckets inadequate | p99 |
| F17 | histogram aggregation wrong | PromQL |
| F18 | high-cardinality request IDs | TSDB |
| F19 | prompt data leaked in trace | privacy |
| F20 | trace context lost at gateway | spans |
| F21 | alert pages on healthy GPU usage | alert review |
| F22 | SLO denominator excludes failures | audit |
| F23 | dashboard not tied to runbook | on-call |
| F24 | telemetry cost exceeds budget | FinOps |

## 19. Sixteen operational runbooks

1. Inventory inference metric and trace coverage.
2. Validate TTFT timestamp boundaries.
3. Recover TPOT/ITL/E2E definitions.
4. Restore terminal-valid completion accounting.
5. Reconcile 429/4xx/5xx and cancellation taxonomy.
6. Diagnose queue and prefill latency.
7. Diagnose decode/KV/GPU saturation.
8. Recover model/route/region breakdowns.
9. Restore DCGM/MIG telemetry.
10. Repair Prometheus targets and duplicate scrapes.
11. Validate counters, gauges and histograms.
12. Contain metric cardinality explosion.
13. Restore OTel trace propagation and sampling.
14. Remove sensitive prompt/tenant data from telemetry.
15. Tune burn-rate alerts and on-call routing.
16. Reconcile observability cost and rollback.

## 20. Eight tabletop game days

1. TTFT p99 doubles but GPU utilization stays low.
2. Streaming HTTP 200 masks truncated completions.
3. Queue backlog grows without matching metric.
4. DCGM exporter loses MIG labels.
5. Prometheus scrape duplication doubles request rate.
6. Trace sampling drops all slow/error spans.
7. New label leaks user IDs and explodes cardinality.
8. Alert routing pages on GPU usage while real errors are hidden.

Default tabletop; live injection requires approved isolated staging.

## 21. Twenty acceptance gates

G01 scope/versions; G02 metric contract; G03 TTFT; G04 TPOT/ITL; G05 E2E; G06 terminal validity; G07 traffic/tokens; G08 429/5xx taxonomy; G09 queue; G10 KV; G11 GPU/DCGM; G12 histogram accuracy; G13 PromQL semantics; G14 cardinality; G15 OTel propagation; G16 privacy; G17 SLO/error budget; G18 alert/runbook; G19 telemetry FinOps; G20 independent review.

See [companion lab](../labs/056-golden-signals-llm-inference-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 057 — TTFT, TPOT, ITL, E2E Latency and Completion.**
