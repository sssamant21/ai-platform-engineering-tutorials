# Chapter 11.7 — Enterprise Observability and Incident Detection Capstone

**Part 11 — Observability | Final enterprise SRE capstone**

## Mission and outcomes

Integrate Chapters 11.1–11.6 into a production-oriented observability and incident-detection system for TrueFoundry-hosted inference on Kubernetes. Deliver reproducible evidence that telemetry covers the full request path, alerts identify customer impact, responders can distinguish platform, application, GPU and dependency failures, and recovery restores service without losing accountability.

**Publication is not certification.** This tutorial and its lab are design artifacts; technical/security review, actual staging execution, incident game days and production acceptance require separate approval.

## 1. Scope, safety and prerequisites

- **[SAFE-READ]** inspect namespaces, workloads, events, metrics, logs, traces, dashboards and alert rules.
- **[LAB-MUTATE]** use isolated staging with approved synthetic requests, bounded load and disposable monitoring changes.
- **[CHANGE-CONTROL]** obtain authorization for production deployments, scrape/rule updates, GPU disruptions, logging changes or alert routing.
- Never record raw prompts, completions, patient data, customer identifiers, credentials, access tokens, model weights or confidential payloads.
- Verify TrueFoundry edition, cluster agent, gateway, vLLM engine, Kubernetes, Prometheus, OpenTelemetry Collector, Grafana, Alertmanager, NVIDIA driver and DCGM exporter versions. No integration or metric name is assumed to exist until confirmed.

**Required artifacts:** service map; telemetry source inventory; SLI contract; dashboards; recording/alert rules; runbooks; incident timeline; recovery evidence; acceptance scorecard.

## 2. Enterprise reference architecture

```text
Synthetic client / real user traffic
                │
       Ingress / API gateway ── gateway RED, timeout and streaming signals
                │
       TrueFoundry deployment ── agent/control-plane reconciliation logs*
                │
      Inference application ── structured logs + W3C trace context
                │
         vLLM runtime ── TTFT, TPOT, queues, token throughput, KV cache*
                │
        CUDA / NVIDIA GPU ── DCGM/device plugin/node metrics*
                │
 Kubernetes scheduling/events ── kube-state-metrics + Events exporter*
                │
      Prometheus / OTel / log collectors
                │
      TSDB + logs + trace backend
                │
   Grafana / Alertmanager / incident workflow
*Availability and metric schemas depend on installed versions/configuration.
```

Keep the data plane functioning when a telemetry backend is unavailable; bounded asynchronous telemetry is preferred. Document unavoidable blind windows, sampling loss, event expiry and collector buffer limits.

## 3. Telemetry coverage matrix

| Layer | Metrics | Logs/events | Traces | Owner |
|---|---|---|---|---|
| Gateway | RPS, error ratio, latency, streaming completion | Access/timeout errors | Root/request span | API SRE |
| TrueFoundry agent | Health/reconciliation if exposed | Bounded agent logs | Only if instrumented | Platform |
| Kubernetes | Pod readiness, restarts, resource usage | Scheduling, probe, image, volume Events | Usually no native app trace | K8s SRE |
| Model application | HTTP, queue, request duration | Structured request/error categories | Application spans | Serving team |
| vLLM | TTFT, TPOT, tokens, scheduler/KV where exposed | Runtime errors, model load | Version-dependent | ML platform |
| GPU | Utilization, memory, power, ECC/XID if exposed | Driver/node events | Not a standard GPU trace | GPU platform |
| Dependencies | Availability, latency, retry | Auth/storage/network errors | Dependency spans if instrumented | Cloud SRE |
| Telemetry stack | UP, drops, lag, queue, rule duration | Collector/exporter errors | Export pipeline if instrumented | Observability |

Mark each cell **verified**, **not available**, **not applicable** or **pending**; do not turn a missing signal into a false zero.

## 4. Instrumentation and data contracts

### Logs
JSON lines with UTC timestamp, severity, service, environment, namespace, Pod UID, deployment revision, safe model alias, trace ID, error category and latency units. Use redaction at source and collector, tenant RBAC and retention controls. Kubernetes Events have limited retention and must be exported if long-term evidence is needed.

### Metrics
Counters for eligible request outcomes; histograms for request latency and TTFT where supported; gauges for queues, KV cache and GPU state. Do not use raw request IDs, prompts or user IDs as labels. Use verified exporter metric names and units, and track TSDB cardinality, storage and scrape health.

### Traces
W3C traceparent/tracestate, approved service/resource attributes and OTLP export with TLS/auth. Head/tail sampling policies must be documented. A propagated context does not guarantee the model runtime exposes internal spans. Sampled trace counts are not authoritative SLO denominators.

## 5. SLIs, SLOs and alerting

Define service-owned availability, streaming completion and TTFT SLIs with explicit eligible requests, timeouts, client cancellations, retries, denominator, rolling window and no-data policy. Distinguish customer-facing reliability from internal GPU/Pod health.

**Example 99.9% availability SLO over 30 days:**
```text
budget_fraction = 1 - 0.999 = 0.001
bad_request_budget = eligible_requests * 0.001
burn_rate = observed_bad_fraction / 0.001
```

A 14.4 burn rate sustained for 1 hour consumes about 2% of a 30-day budget. Validate both 1h and 5m windows for a fast-burn page, with minimum traffic checks. Do not interpret missing metrics as success. Use the real service metric dictionary rather than assuming an `inference_*` series exists.

## 6. Standard dashboards

1. **Executive reliability:** SLO attainment, 30-day budget remaining, user impact and incident count.
2. **Gateway RED:** eligible traffic, 5xx/timeouts, p95/p99, retries and streaming completion.
3. **Inference:** TTFT/TPOT, tokens/sec, running/waiting requests, KV cache and model revision.
4. **GPU/Kubernetes:** allocation, GPU memory/utilization, device faults, Pod readiness, restart/OOM and scheduler Events.
5. **Telemetry health:** target UP, collector accepted/dropped, backend ingestion lag, query/rule duration and alert delivery.

Every panel requires units, data freshness, owner, runbook link, version filters and a no-data state distinct from zero.

## 7. Incident detection and triage decision tree

```text
Customer-facing SLO burn or availability alert?
 ├─ YES → validate eligible traffic and metric health
 │        ├─ Gateway errors/timeouts → ingress, auth, network, downstream
 │        ├─ TTFT/queue regression → prefill, concurrency, KV cache, GPU
 │        ├─ TPOT regression → decode, batching, GPU/driver
 │        ├─ Pod unavailable → Events, probes, image, resource pressure
 │        └─ GPU faults → DCGM, XID/ECC, driver/device plugin
 └─ NO → telemetry missing?
          ├─ YES → collector/scrape/retention incident; avoid false healthy
          └─ NO → watch trends, capacity and dependency errors
```

Start with customer impact, not a single hardware utilization threshold. Cross-check logs, metrics, traces and Kubernetes Events using UTC timestamps and deployment revisions. Record whether a finding is an observation, hypothesis or confirmed root cause.

## 8. Twelve incident response runbooks

**IR-01 — Gateway 5xx spike:** segment by route/model/revision, correlate ingress and runtime errors, roll back only if change-linked.

**IR-02 — TTFT p95 regression:** inspect input-token mix, queue wait, prefill, autoscaling and GPU/CPU saturation.

**IR-03 — TPOT/streaming slowdown:** inspect decode scheduling, GPU load, network buffering and client cancellation.

**IR-04 — vLLM KV cache exhaustion:** inspect concurrency/context limits, preemption, memory fit and safe capacity options.

**IR-05 — CUDA OOM or runtime crash:** separate device OOM from Kubernetes OOMKilled; inspect model/version and resource limits.

**IR-06 — GPU XID/ECC/thermal event:** preserve node evidence, coordinate safe drain/replacement with platform team.

**IR-07 — Pod Pending/CrashLoopBackOff:** inspect scheduling, images, probes, PVC, events and `--previous` logs.

**IR-08 — TrueFoundry agent reconciliation failure:** verify actual agent namespace, permissions, identity and backend reachability.

**IR-09 — Metrics pipeline outage:** validate target discovery, scrape auth, TSDB/WAL and remote-write queue; document SLO blind window.

**IR-10 — Logs or traces missing:** inspect collector buffer/drop, sampling, propagation, retention and redaction.

**IR-11 — Alert storm or silent page:** inspect recording rules, no-data, Alertmanager routing, grouping and silences.

**IR-12 — Security/privacy exposure in telemetry:** restrict access, stop emission, escalate incident, rotate exposed secrets where needed and follow approved purge/retention policy.

For each runbook record: detection, scope, impact, evidence, owner, containment, recovery, rollback, verification and follow-up.

## 9. Production change and rollback

Version all dashboards, recording rules, collector configs and runbooks in Git. Validate YAML/PromQL with supported tools; review high-cardinality labels, TLS/auth, resource budgets and alert routing. Use staging/canary promotion, preserve prior known-good configurations and verify rollback. Avoid disabling mandatory audit logs or exposing metrics endpoints.

## 10. Acceptance scorecard (100 points)

| Domain | Points | Evidence |
|---|---:|---|
| Telemetry architecture and source inventory | 10 | Approved service map and versions |
| Logs, Events and privacy controls | 10 | Redaction, retention, source proof |
| Prometheus metrics and cardinality | 10 | Scrape/recording-rule validation |
| OpenTelemetry tracing/correlation | 10 | Synthetic cross-service trace |
| GPU/vLLM inference metrics | 15 | TTFT/TPOT, KV, GPU attribution |
| Dashboards and telemetry coverage | 10 | Screenshots/query evidence |
| SLI/SLO/error budget design | 10 | Approved definitions/calculations |
| Alerts, routing and escalation | 10 | Staging notification test |
| Incident response and recovery | 10 | Game-day timeline and rollback |
| Governance and final sign-off | 5 | Reviewer approval |
| **Total** | **100** | |

**Proposed acceptance:** 85/100 or better **and** all critical gates passed; no outstanding privacy exposure, missing customer-impact SLI, failed alert delivery or unverified recovery. This is a curriculum scoring policy, not an external certification.

Complete the [capstone lab](../labs/07-enterprise-observability-incident-detection-capstone-check.md) with **36 failure scenarios, 12 acceptance gates and five staging game days**.

**Part 11 publication complete (7/7).** Formal technical/security review and live execution remain pending.
