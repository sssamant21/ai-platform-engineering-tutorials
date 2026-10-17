# Chapter 060 — OpenTelemetry Traces Across Inference Paths

**Part 08 — LLM Observability, Metrics and Tracing**  
**Status:** Tutorial and companion lab published; technical/security reviews and live execution pending.

## Learning objectives
1. Explain trace IDs, span IDs, parent/child relationships, span events, links, context propagation and sampling.
2. Trace gateway → auth → routing → queue → inference → streaming → response across service boundaries.
3. Separate observed spans from inferred model phases and correlate traces with TTFT, TPOT, ITL, GPU and queue metrics.
4. Build a privacy-safe OpenTelemetry Python demonstration with a local OTLP collector.
5. Diagnose missing spans, high-cardinality attributes, sampling gaps, clock skew and exporter backpressure.

## 1. Architecture and signal boundaries

```text
client (traceparent)
  -> ingress / API gateway [HTTP SERVER span]
    -> auth and policy [INTERNAL span]
      -> router / model selection [CLIENT span]
        -> inference service [SERVER span]
          -> admission / queue [INTERNAL span]
          -> prefill (only when instrumented)
          -> decode / stream (only when instrumented)
        <- first token / final chunk
  -> OTel SDK / OTLP -> Collector -> tracing backend
                              -> optional tail sampling / filtering
Prometheus / DCGM / Kubernetes metrics remain independent signals.
```

A trace shows causal relationships and observed timing. **Do not** label a GPU kernel or prefill span as measured unless the serving runtime actually emits that span. The example uses a synthetic inference phase; it does not instrument vLLM/Triton kernels.

### Span taxonomy
| Span | Kind | Recommended attributes | Avoid |
|---|---|---|---|
| `POST /v1/chat/completions` | SERVER | route template, status, service.name | raw path with IDs, prompts |
| `authorize_request` | INTERNAL | policy outcome, low-cardinality auth method | JWT, user email |
| `route_inference` | CLIENT/INTERNAL | model family, deployment tier | customer-provided model strings |
| `inference_queue` | INTERNAL | bounded queue class | request body |
| `synthetic_inference` | INTERNAL | simulated=true | claiming GPU execution |
| `stream_response` | INTERNAL | first-token event, completion status | generated text |

Use `traceparent` / `tracestate` per W3C Trace Context; `baggage` may propagate to downstream services and must not contain secrets or sensitive identifiers. Keep resource attributes stable (service.name, deployment.environment.name). A trace ID is a correlation key, **not** an authentication token.

## 2. SRE interpretation
- **High gateway duration, low inference duration:** investigate ingress, network, auth, response buffering.
- **Long queue span:** admission, concurrency limits, overload, insufficient replicas.
- **Slow first-token event:** investigate routing, queue, prefill, cold model loading and transport buffering; a timestamp alone cannot attribute root cause.
- **High duration after first token:** investigate token rate, streaming stalls, client backpressure and decode throughput.
- **Missing downstream child span:** inspect propagation headers, instrumentation, sampling, exporters and clock skew.
- **Healthy trace with high GPU utilization:** correlate with DCGM, vLLM/Triton metrics before deciding capacity changes.

Trace sampling makes trace-derived request rates incomplete; use metrics for SLO denominators. Avoid unbounded span names, model IDs, user IDs and request IDs as metric labels.

## 3. Prerequisites and safety
Python 3.11+, isolated virtual environment, Docker for optional collector, staging-only endpoints. No real prompts, credentials, production traffic or tenant data. **[SAFE-READ]** inspect logs and spans. **[LAB-MUTATE]** start local collector and synthetic request. **[CHANGE-CONTROL]** deploy SDK, collector, sampling or egress changes to production only after review.

## 4. Executable lab
Follow [Chapter 060 companion lab](../labs/060-opentelemetry-traces-across-inference-paths-check.md). It installs the OpenTelemetry SDK/exporter, runs a synthetic inference pipeline, checks parent-child span structure, validates redaction and demonstrates an OTLP Collector.

## 5. Sampling, reliability and cost
Head sampling makes the decision early and can lose rare errors; tail sampling requires collector-side buffering, memory and correct routing of all spans for a trace. Prefer a controlled sampling policy with bounded resource use. Set collector memory limits, batching, retries, queue limits and exporter timeouts; monitor refused/dropped spans and backend ingest costs. Collector outages must not block inference requests. Protect collector endpoints with network restrictions and TLS/authentication in nonlocal environments.

## 6. Troubleshooting decision tree
1. No spans: verify SDK provider initialization, sampler, shutdown/flush, OTLP endpoint/protocol and collector logs.
2. Broken parent-child chain: verify HTTP propagation across each hop and async task context.
3. Unexpected missing traces: verify sampling decisions and dropped spans; do not infer no traffic.
4. Inflated duration: check time source, buffering, overlapping spans and server/client boundaries.
5. Trace contains secrets: stop export, restrict access, rotate exposed secrets as appropriate, purge per retention policy, correct instrumentation and add regression tests.
6. High exporter overhead: reduce event volume, cardinality and sampling; inspect batch processor queues before tuning.

## 7. Production runbooks
**RB-060-A — Inference latency:** select a slow sampled trace, compare gateway/auth/queue/inference/stream spans, correlate p95 TTFT/TPOT and GPU metrics, record trace ID and time window, escalate to the owning layer.

**RB-060-B — Missing spans:** check propagated `traceparent`, collector health, sampler config, OTLP protocol mismatch and backend ingestion; validate with synthetic requests before changing production sampling.

**RB-060-C — Telemetry leak:** isolate exporter access, suspend sensitive attributes, involve security/privacy owner, rotate credentials if exposed and verify cleanup under retention policy.

**RB-060-D — Collector overload:** review queue/drop counters, collector CPU/memory, batch/export settings and backend throttling; preserve inference availability, scale collector within approved budgets.

## 8. Acceptance gates
- [ ] Synthetic request creates a root span with nested auth, routing, queue, inference and streaming spans.
- [ ] First-token event precedes stream completion.
- [ ] Parent-child relationships and trace IDs are consistent.
- [ ] No prompt, token, API key, bearer token or raw tenant identifier is exported.
- [ ] OTLP collector accepts spans in an isolated environment.
- [ ] Missing exporter is nonfatal to inference path.
- [ ] Operational investigation links a trace to independent metrics without claiming uninstrumented phases.
- [ ] Technical review, security review and live lab execution recorded separately.

## 9. Further reading
- [OpenTelemetry Python](https://opentelemetry.io/docs/languages/python/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Semantic conventions](https://opentelemetry.io/docs/specs/semconv/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)

**Next:** Chapter 061 — Prompt-Safe Logging, Redaction and Cardinality.
