# Chapter 11.4 — Hands-on Lab: OpenTelemetry Distributed Tracing and Correlation

**Part 11 | Isolated staging | Live execution pending**

## Mission and safety

Validate W3C trace context, OTel SDK/Collector, inference request spans, logs-to-traces correlation, sampling, privacy, telemetry failures and recovery. Use synthetic prompts only. **[SAFE-READ]** inspect; **[LAB-MUTATE]** approved staging; **[CHANGE-CONTROL]** production approval. Never collect raw prompts, completions, PHI, credentials or model weights.

## Lab 0 — Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl get deployments,daemonsets -A
kubectl get svc -A
kubectl get configmaps -A | grep -Ei 'otel|collector' || true
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Record SDK/runtime versions, TrueFoundry agent/inference gateway topology, Collector endpoint, trace backend, service names, retention, sampling, IAM/TLS and reviewer. Missing instrumentation is a gap, not a pass.

## Lab 1 — Context propagation

Send one approved synthetic inference request with generated trace context. In an authorized trace UI, identify gateway, app, model-runtime and dependency spans that actually exist. Verify parent-child relationships, trace IDs and timestamps. Record missing boundaries without inventing spans.

## Lab 2 — Collector pipeline [SAFE-READ]

Inspect receiver/protocol, processors, exporter, TLS/auth, network policy, batching, memory limits and telemetry self-metrics. Review bounded collector logs and check accepted/refused/exported/dropped span counters where supported. Do not export configuration secrets.

## Lab 3 — Correlation and streaming

Correlate trace ID to structured gateway/application logs and deployment revision. For synthetic streaming inference, distinguish TTFT, overall duration and cancellation. Validate that request IDs are not Prometheus metric labels and no prompt text appears in spans.

## Lab 4 — Sampling and failure tabletop

Compare head vs tail sampling coverage and cost; rehearse backend 429, exporter timeout, TLS mismatch, collector memory pressure and trace fragmentation using disposable staging resources or tabletop. Never disrupt production telemetry. Confirm application availability is independent of trace backend.

## Lab 5 — Recovery and cleanup

Restore lab-only configuration, verify new synthetic traces and backend ingestion, inspect queues/drops and remove test workloads. Capture sanitized evidence, UTC timestamps, trace ID, span count, sampling policy, expected/actual and reviewer.

## Thirty failure scenarios

| # | Scenario | Focus |
|---|---|---|
| 01 | SDK not enabled | Instrumentation |
| 02 | Wrong OTLP endpoint | Networking |
| 03 | gRPC vs HTTP protocol mismatch | Transport |
| 04 | Collector Service absent | Discovery |
| 05 | Collector network policy blocks traffic | Connectivity |
| 06 | Collector TLS certificate invalid | Trust |
| 07 | Backend exporter 401/403 | Auth |
| 08 | Backend exporter 429 | Throttling |
| 09 | Backend exporter 5xx | Retry |
| 10 | Collector memory limiter drops | Capacity |
| 11 | Collector batch oversized | Latency/memory |
| 12 | Collector Pod restarts | Availability |
| 13 | Traceparent stripped at gateway | Propagation |
| 14 | Duplicate root spans | Context extraction |
| 15 | Missing app span | SDK scope |
| 16 | Missing vLLM internal span | Runtime support |
| 17 | Async job loses context | Span links |
| 18 | Trace ID malformed | W3C format |
| 19 | Tracestate untrusted | Boundary |
| 20 | Baggage contains sensitive field | Privacy |
| 21 | Trace sampled out | Sampling |
| 22 | Tail-sampling buffer too small | Decision window |
| 23 | Tail-sampling trace fragments | Trace affinity |
| 24 | High-cardinality span attributes | Cost |
| 25 | Log trace ID missing | Correlation |
| 26 | Metrics exemplar unavailable | Backend support |
| 27 | Streaming span closes early | Lifecycle |
| 28 | Clock skew distorts waterfall | Time sync |
| 29 | Prompt/response captured in span | Incident |
| 30 | Trace backend outage | Degradation/recovery |

For each scenario: expected/actual, environment, UTC timestamp, sanitized trace/log reference, diagnosis, remediation, rollback and reviewer. Unsafe scenarios must be tabletop.

## Ten acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved scope and tracing source inventory | Pending |
| G2 | Gateway/app propagation validated | Pending |
| G3 | Model/dependency spans or documented gaps | Pending |
| G4 | Collector OTLP pipeline and TLS/auth verified | Pending |
| G5 | Log-to-trace correlation demonstrated | Pending |
| G6 | Sampling, retention and cost reviewed | Pending |
| G7 | Redaction and safe attribute policy validated | Pending |
| G8 | Collector drop/retry/lag observability verified | Pending |
| G9 | Recovery, rollback and cleanup demonstrated | Pending |
| G10 | SRE/security review and sign-off | Pending |

**Pass** requires timestamped evidence and reviewer approval; blocked/unexecuted tests are not passes.

**Next:** Chapter 11.5 — GPU, vLLM and Inference Performance Observability.
