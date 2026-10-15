# Chapter 11.4 — OpenTelemetry Distributed Tracing and Correlation

**Part 11 — Observability | Canonical production SRE tutorial**

## Objectives and safety

Instrument and troubleshoot end-to-end requests traversing a TrueFoundry inference gateway, application service, model runtime, Kubernetes workloads and external dependencies. Deploy an OpenTelemetry (OTel) Collector pipeline, propagate W3C trace context, choose sampling and retention, correlate traces with logs and metrics, and preserve privacy and reliability.

**Scope:** platform control plane, cluster agent, gateway, model server and cloud services may expose different tracing capabilities. Verify actual TrueFoundry edition, deployed runtime and available integrations. Do not claim the platform automatically generates traces. **[SAFE-READ]** inspect; **[LAB-MUTATE]** synthetic staging; **[CHANGE-CONTROL]** production instrumentation or pipeline changes. Do not capture prompts, completions, PHI, credentials, model weights or Authorization headers.

## 1. End-to-end trace architecture

```text
Client → API gateway → inference application → model-serving runtime
              │                │                     │
              └──── W3C traceparent/tracestate ──────┘
                     │               │
                 OTel SDK        OTel SDK/bridge
                     └──── OTLP gRPC/HTTP ────┐
                                               ▼
                                      OTel Collector
                                  receive → process → export
                                               │
                                         Trace backend
                                     (Tempo/Jaeger/cloud)
                                               │
                           logs (trace_id) ↔ traces ↔ metrics/exemplars
```

Spans represent operations with parent-child relationships. A trace ID connects spans; a span ID identifies one operation. Propagating context does not automatically create spans. Asynchronous queues require explicit propagation and careful span links when causal relationships are not simple parent-child.

## 2. W3C trace context and service identity

Propagate `traceparent` and `tracestate` across trusted service boundaries using supported instrumentation. A `traceparent` header has version, 16-byte trace ID, 8-byte parent span ID and flags. Do not copy untrusted `baggage` blindly; it can carry sensitive or attacker-controlled content.

Use resource attributes such as `service.name`, `service.version`, `deployment.environment.name`, `k8s.namespace.name`, `k8s.pod.uid` and cluster identity where supported. Semantic conventions evolve by SDK/version; pin a supported instrumentation version and document naming. Avoid unbounded attributes such as raw user IDs or prompt bodies.

## 3. Trace boundaries for LLM inference

Capture approved metadata for request authorization, routing, queueing, model selection, prefill, decoding, tool invocation and cloud model-artifact fetch. Prefer **safe counts and timings**: request/response token counts, TTFT, total duration, model alias/version, status/error class and queue wait. Never record raw prompt/response content by default. When the runtime cannot expose an internal span, use surrounding application spans and label the visibility gap.

For streaming inference, distinguish first-byte/first-token latency from final completion; a single HTTP span may remain open for the whole stream. Verify that client disconnect and cancellation are represented correctly.

## 4. Collector deployment and configuration

Choose a supported topology:
- **Agent/DaemonSet:** node-local reception where required.
- **Gateway Deployment:** central processing, policy and export.
- **Sidecar:** only for specific isolation/compatibility needs; adds overhead.

Example staging Collector config (adjust exporter endpoint, TLS, auth and installed distribution):
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  batch:
    timeout: 5s
    send_batch_size: 512
exporters:
  otlp:
    endpoint: trace-backend.monitoring.svc.cluster.local:4317
    tls:
      insecure: false
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
```

This is a **template**, not a ready-to-apply deployment. Verify receiver exposure, certificate trust, exporter credentials, processor compatibility and resource limits. `tls.insecure: false` requires valid TLS at the backend; use supported mTLS/auth mechanisms. Protect Collector ingress with network policy and authentication where feasible. Memory limits and retry queues require sizing based on real traffic.

## 5. Safe discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl get deployments,daemonsets -A
kubectl get svc -A
kubectl get configmaps -A | grep -Ei 'otel|opentelemetry|collector' || true
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

After identifying approved collector namespace and deployment:
```bash
kubectl -n OTEL_NAMESPACE get deploy OTEL_COLLECTOR_DEPLOYMENT
kubectl -n OTEL_NAMESPACE logs deploy/OTEL_COLLECTOR_DEPLOYMENT --since=15m --tail=100
```
Review output for sensitive fields. Do not dump Secret-backed Collector environment variables or configuration values containing credentials.

## 6. Sampling and retention strategy

**Head sampling:** decisions near ingestion reduce cost but may miss rare failures. **Tail sampling:** can retain errors/high latency but requires buffering, consistent trace routing and extra memory; partial traces and high traffic can overwhelm the collector. Use a documented policy balancing error/latency coverage, representative baseline traffic, compliance and cost.

Sampling must be accounted for when interpreting counts. Do not use sampled trace counts as authoritative request-rate/error-rate SLIs; use metrics. Define retention by incident investigation need and data policy, with access control and deletion behavior.

## 7. Logs, metrics and traces correlation

Add trace/span IDs to **structured logs** (with redaction and access control). Use exemplars when the metrics pipeline supports them, linking selected observations to traces without introducing trace IDs as Prometheus labels. Correlate `service.name`, environment, deployment revision, image/model digest, request ID and time. Ensure clocks are synchronized; otherwise waterfall duration can appear misleading across services.

## 8. Performance, cost and reliability

Instrument only meaningful spans; avoid per-token spans or huge events that create runaway volume. Monitor SDK queue size, export failures, Collector receiver accepted/refused spans, processor drops, exporter retry, memory, CPU, backend ingestion lag and trace search latency. Collector availability is a dependency; application request success should not depend on a remote trace backend. Use bounded asynchronous export with defined drop behavior.

## 9. Twelve production troubleshooting runbooks

1. **No traces:** verify SDK enabled, exporter endpoint/protocol, collector target and backend ingest.
2. **Trace breaks at gateway:** inspect `traceparent` forwarding, proxy stripping and trust policy.
3. **Missing model-runtime spans:** check supported instrumentation and document runtime visibility gap.
4. **Duplicate root traces:** inspect propagation and service-side context extraction.
5. **Collector OTLP 401/403:** verify workload identity, auth headers and backend permissions without exposing tokens.
6. **Collector TLS handshake failure:** inspect certificate chain, hostname/SNI and mTLS.
7. **Collector memory pressure:** inspect batch, memory limiter, sampling, queues and burst capacity.
8. **High export drops:** check backend 429/5xx, retry, persistent queue options and receiver throttling.
9. **Traces missing under tail sampling:** inspect trace affinity, decision wait, buffers and sampling policy.
10. **High-cardinality attributes:** remove unbounded attributes and review backend index/storage costs.
11. **Misleading span duration:** verify clock sync, async span boundaries and streaming completion.
12. **Sensitive prompt in span:** contain exposure, stop collection, restrict backend access and follow security incident handling.

## 10. Change, rollback and acceptance

Before production: verify trace schema, redaction, service boundaries, W3C propagation, exporter TLS/auth, sampling policy, collector SLO and retention. Run staging synthetic traces across gateway/app/runtime; document unsupported internal spans. Measure overhead and storage impact. Rollback instrumentation/configuration without disabling mandatory security audit logs.

Complete the [hands-on lab](../labs/04-opentelemetry-distributed-tracing-correlation-check.md) with **30 failure scenarios and 10 acceptance gates**. Formal review and live execution remain pending.

**Next:** Chapter 11.5 — GPU, vLLM and Inference Performance Observability.
