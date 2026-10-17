# Chapter 057 — TTFT, TPOT, ITL, E2E Latency and Completion

**Part 08 — LLM Observability, Metrics and Tracing | Chapter 057/100**

> **Status:** Written tutorial published; live execution, independent technical/security review and operational acceptance pending. Model servers and SDKs vary in event boundaries, tokenization, chunking and metric names. Validate definitions before comparing systems or using latency metrics for paging.

## Learning objectives

Define and measure TTFT, TPOT, ITL, end-to-end latency and terminal-valid completion; reconcile client/server measurements; build accurate Prometheus histograms and traces; distinguish queue, prefill and decode bottlenecks; design SLOs and incident runbooks for streaming and non-streaming inference.

## 1. Request timeline

```text
client send
    │ network + TLS
gateway receive / authentication
    │ admission / rate limit
model request accepted ────────────────┐
    │ queue wait                       │
prefill start                          │
    │ prompt processing                │
first output token emitted ◄── TTFT ───┘
    │ decode token 2
    │ decode token 3
    │ ...
final output token
    │ serialization / flush
terminal response ◄──────────── E2E ───┘
```

Every latency metric requires a declared clock, start/end event, scope, streaming mode and cancellation policy.

## 2. TTFT — Time to First Token

```text
TTFT_server = first_token_emitted_at - request_accepted_at
TTFT_client = first_token_received_at - client_request_sent_at
```

The client view includes network, gateway and transport delays; server TTFT may include admission and queueing depending on the exporter. Do not compare client and server TTFT without documenting boundaries.

## 3. ITL — Inter-Token Latency

For streamed output token timestamps `t1..tn`, `ITL_i = t_i - t_(i-1)` for `i >= 2`. ITL can expose jitter, batching interference, GPU contention and network flush effects. HTTP/SSE chunks may contain multiple tokens; **chunk inter-arrival is not automatically token ITL**.

## 4. TPOT — Time per Output Token

A common approximation for requests with at least two output tokens:

```text
TPOT = (last_output_token_time - first_output_token_time) /
       (output_token_count - 1)
```

Some exporters instead use generation duration divided by all output tokens or report batch-level decode timing. Record exact definition, tokenization and exclusions; a single-token response has no post-first-token TPOT under this definition.

## 5. E2E — End-to-End Latency

```text
E2E_client = terminal_response_received_at - client_request_sent_at
E2E_server = terminal_response_finished_at - request_accepted_at
```

E2E depends on output length and stream termination. Compare latency within request classes, context lengths and output token ranges rather than aggregating all requests into one misleading percentile.

## 6. Completion taxonomy

| Outcome | Definition | Default SLI handling |
|---|---|---|
| terminal-valid | completed protocol and semantic checks | success |
| server 5xx | server error | failure |
| timeout | deadline reached without valid terminal state | failure |
| truncated stream | transport ends without valid terminal marker | failure |
| model-invalid output | schema/tool contract violated | failure if in scope |
| explicit client cancellation | client intentionally aborts | document exclusion policy |
| rate limit 429 | admission policy | separate admission SLI or failure per contract |
| invalid client request | malformed input | usually excluded if contract allows |

HTTP 200 at stream start is not proof of successful terminal completion.

## 7. Golden signal interactions

High TTFT with normal TPOT often indicates queue/prefill/admission delay. Normal TTFT with high ITL/TPOT suggests decode contention. High E2E with normal TTFT/TPOT may reflect long output. Low terminal-valid completion with healthy latencies suggests truncated streams, cancellations or invalid outputs.

## 8. Token and request classes

Partition metrics by bounded model, route, region, prompt-token bucket, output-token bucket and streaming mode. Avoid raw prompt, user ID, request ID and unbounded session identifiers as Prometheus labels. Use exemplars/traces for per-request drill-down.

## 9. Histogram design

Prefer seconds as base unit and histogram buckets covering target SLO thresholds and tails. Keep consistent bucket layouts when aggregating classic histograms across replicas. Quantiles are estimated; sparse traffic and poor bucket resolution can distort p99.

## 10. Canonical metric contract [DESIGN]

```text
llm_ttft_seconds_bucket{model,route,le}
llm_tpot_seconds_bucket{model,route,le}
llm_itl_seconds_bucket{model,route,le}
llm_e2e_seconds_bucket{model,route,le}
llm_requests_total{model,route,outcome}
llm_terminal_valid_completions_total{model,route}
llm_output_tokens_total{model,route}
llm_input_tokens_total{model,route}
llm_streams_inflight{model,route}
```

These are proposed normalized metrics, **not guaranteed native exporter names**. Map vLLM/Triton/Ray Serve/gateway versions explicitly.

## 11. PromQL [REQUIRES NORMALIZED METRICS]

```promql
histogram_quantile(0.95,
  sum by (model, le) (rate(llm_ttft_seconds_bucket[5m]))
)
```

```promql
histogram_quantile(0.95,
  sum by (model, le) (rate(llm_e2e_seconds_bucket[5m]))
)
```

```promql
sum by (model) (rate(llm_terminal_valid_completions_total[5m]))
/
clamp_min(sum by (model) (rate(llm_requests_total{outcome!="excluded"}[5m])), 1e-9)
```

The availability ratio is valid only when the denominator metric counts each eligible admitted request exactly once and numerator/denominator windows align. For low traffic, use count-based windows and avoid treating absent data as zero success.

## 12. OpenTelemetry tracing

Recommended spans: `gateway.accept`, `admission`, `queue`, `prefill`, `first_token`, `decode`, `stream.finalize`. Record sanitized token counts, model digest, outcome, latency and bounded error class. Cross-service propagation and clock skew must be validated. Do not emit raw prompts or completions.

## 13. Client-side measurement

Use a monotonic clock for durations. A client streaming harness should record request start, response headers, each chunk arrival, protocol terminal marker, output token count and error/cancellation. **Token timestamps require tokenizer-aware parsing**; SSE chunk timestamps are only chunk-level latency.

## 14. Server-side measurement

Instrument admission, queue, prefill, first-token emission, decode and finalization. Avoid expensive per-token synchronous logging. Use histograms and sampled traces; measure instrumentation overhead and protect data privacy.

## 15. SLO design

Define separate objectives for TTFT p95/p99, TPOT/ITL p95, E2E p95 by request class and terminal-valid availability. Specify traffic minimums, time windows, exclusions, cancellation rules and alert burn rates. Tie alerts to customer impact and runbooks.

## 16. Safe-read inspection

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

Inspect exporter endpoint only with authorization. Do not print sensitive environment variables or raw model payloads.

## 17. Troubleshooting decision tree

```text
Latency/completion incident?
 ├─ TTFT high, TPOT normal → queue/prefill/network
 ├─ TTFT normal, TPOT high → decode/KV/GPU
 ├─ ITL jitter → batching/flush/network
 ├─ E2E high, per-token normal → output length
 ├─ HTTP 200, no terminal marker → truncated stream
 ├─ completion low → error/cancel taxonomy
 ├─ p99 nonsensical → buckets/aggregation
 ├─ client-server discrepancy → clock/boundary
 └─ metric absent → exporter/scrape/version
```

## 18. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | TTFT starts at inconsistent boundary | spec |
| F02 | TTFT excludes queue | traces |
| F03 | client TTFT conflated with server | compare |
| F04 | TPOT divides by all tokens incorrectly | formula |
| F05 | TPOT on single-token response | guard |
| F06 | chunk gap mislabeled ITL | parser |
| F07 | tokenizer counts mismatch | audit |
| F08 | E2E excludes terminal flush | client |
| F09 | output length biases E2E | buckets |
| F10 | HTTP 200 masks truncated stream | terminal |
| F11 | cancellations counted as 5xx | taxonomy |
| F12 | 429 omitted from admission SLI | gateway |
| F13 | invalid JSON/tool output counted success | validation |
| F14 | retry counted twice | IDs |
| F15 | histogram buckets miss SLO threshold | config |
| F16 | histogram quantiles aggregated incorrectly | PromQL |
| F17 | low-traffic p99 misleading | samples |
| F18 | duplicate scrape doubles request counts | targets |
| F19 | metric label cardinality explodes | TSDB |
| F20 | prompt data leaks into trace | privacy |
| F21 | clock skew corrupts spans | tracing |
| F22 | high instrumentation overhead | benchmark |
| F23 | alert ignores terminal validity | alert |
| F24 | rollback changes metric contract silently | GitOps |

## 19. Sixteen operational runbooks

1. Inventory latency/completion metric contracts.
2. Restore TTFT event boundaries.
3. Diagnose queue versus prefill delays.
4. Validate TPOT denominator and token count.
5. Recover ITL measurement from true token events.
6. Diagnose E2E and output-length skew.
7. Recover terminal marker and partial stream accounting.
8. Reconcile cancellation/429/error taxonomy.
9. Validate schema/tool completion correctness.
10. Restore histogram buckets and quantile aggregation.
11. Diagnose low-sample percentile instability.
12. Recover duplicate scrape/retry accounting.
13. Contain cardinality and sensitive labels.
14. Restore OTel clock and trace propagation.
15. Tune SLO burn-rate alerts and dashboards.
16. Execute metric-contract rollback and RCA.

## 20. Eight tabletop game days

1. Queue spike doubles TTFT but TPOT stays normal.
2. Decode contention triples ITL during long generation.
3. SSE chunks batch tokens and ITL dashboard lies.
4. HTTP 200 stream disconnects without terminal marker.
5. Single-token responses cause TPOT divide-by-zero.
6. Histogram buckets hide p99 TTFT regression.
7. Client/server clocks disagree across trace spans.
8. New metric release breaks SLO alerts and rollback.

Default tabletop; live injection requires approved isolated staging.

## 21. Twenty acceptance gates

G01 scope/versions; G02 timestamp boundaries; G03 client/server TTFT; G04 TPOT formula; G05 ITL/tokenization; G06 E2E; G07 terminal marker; G08 cancellations; G09 429/errors; G10 schema validity; G11 request classes; G12 histogram buckets; G13 PromQL; G14 low traffic; G15 retry/dedup; G16 OTel tracing; G17 privacy; G18 SLO/alerts; G19 rollback/overhead; G20 independent review.

See [companion lab](../labs/057-ttft-tpot-itl-e2e-latency-completion-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 058 — vLLM, Triton and Ray Serve Metrics.**
