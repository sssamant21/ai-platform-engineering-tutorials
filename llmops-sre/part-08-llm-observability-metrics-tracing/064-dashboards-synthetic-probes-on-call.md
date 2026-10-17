# Chapter 064 — Dashboards, Synthetic Probes and On-Call

**Part 08 — LLM Observability, Metrics and Tracing**  
**Status:** Published tutorial and lab; technical/security review and live execution pending.

## Learning objectives
- Build user-journey and service dashboards for inference availability, TTFT, TPOT, throughput, latency, cost and saturation.
- Design synthetic probes that exercise gateway, authentication, inference and streaming completion safely.
- Separate black-box synthetic checks from white-box metrics, logs and traces.
- Implement executable local HTTP probes, unit tests, and evidence capture.
- Define actionable on-call alert ownership, incident severity, escalation and handoff.

## 1. Reference architecture

```text
                  synthetic probe scheduler
                         |
               safe test tenant / key
                         |
client -> gateway -> router -> inference worker -> stream
           |         |             |
       request SLI  queue       GPU/model
           \_________|_____________/
                     |
             Prometheus / OTEL
                     |
         recording rules + SLO alerts
                     |
          Grafana dashboards + alerting
                     |
          Alertmanager -> on-call
                     |
         runbooks, incident timeline
```

**Principle:** dashboards must represent what users experience, not only whether servers are alive. A green GPU graph cannot prove successful token delivery; a green HTTP 200 cannot prove a stream completed.

## 2. Dashboard specification

| Dashboard row | Essential panels | Diagnostic question |
|---|---|---|
| User experience | eligible requests, bad requests, SLO error budget, TTFT p50/p95/p99, stream completion | Can users get usable answers? |
| Traffic | RPS, active requests, model mix (bounded catalog), token input/output rates | What workload changed? |
| Latency | queue wait, prefill, TTFT, TPOT, end-to-end duration, timeouts | Where is the delay? |
| Capacity | GPU utilization, memory, KV cache, queue depth, admission rejects, replicas | Is the service saturated? |
| Reliability | burn-rate alerts, 5xx, 429, cancellations, probe success, telemetry gaps | Is degradation sustained? |
| Cost | tokens/sec, GPU-hours, cost/request (modeled), idle utilization | Is capacity economical? |

Use consistent dashboard variables `environment`, `service` and approved `model_family`; never use request IDs, prompt strings or tenant identifiers as metric labels. Display scrape freshness and last sample timestamps so a missing panel cannot masquerade as healthy zero. Include deployment annotations, runbook links, panel ownership and units. Differentiate TTFT, TPOT and total duration: they measure different stages.

### Example PromQL
```promql
# User-facing error ratio; match your approved SLI taxonomy
sum by (service, environment) (rate(llm_sli_bad_requests_total[5m]))
/
clamp_min(sum by (service, environment) (rate(llm_sli_requests_total[5m])), 1e-9)

# Request throughput
sum by (service, environment) (rate(llm_sli_requests_total[5m]))

# Queue pressure (illustrative metric name)
max by (service, environment) (llm_queue_depth)

# Scrape health
min by (job) (up{job=~"llm-gateway|inference-worker"})
```
Validate real exporter names and histogram semantics in your environment. Idle denominators require explicit handling; avoid showing error ratio as a reassuring zero when there are no requests.

## 3. Synthetic probe design

**Probe levels:**
1. **Transport:** DNS, TLS, connection and gateway reachability.
2. **Readiness:** health endpoint or non-sensitive model metadata (does not prove inference).
3. **Inference:** send an approved fixed synthetic prompt; validate response status, schema and expected bounded output.
4. **Streaming:** verify first-token deadline, chunk progression, end-of-stream marker and full completion.
5. **Regional journey:** execute from independent regions and correlate to customer experience.

Use a dedicated test identity, least-privilege credentials, fixed low token caps, hard timeout, strict request rate and cost budget. Never embed tokens in dashboard JSON or logs. Treat generated output as untrusted; log only status, duration and controlled result codes. Probe endpoints and input/output schemas vary by serving platform; validate before use. Avoid generating probe traffic during emergency capacity constraints without incident-command approval.

## 4. On-call signal quality
A page must have: user impact, service owner, severity, threshold/window, SLO link, diagnostic dashboard, runbook and escalation path. Route availability and streaming failures by service ownership; avoid paging for isolated worker restarts when redundancy protects users.

| Severity | Example | Response |
|---|---|---|
| SEV1 | widespread inability to serve inference | incident command, immediate mitigation |
| SEV2 | material SLO burn or region-wide degradation | on-call triage and owner escalation |
| SEV3 | isolated warning, no sustained user impact | ticket, investigation during working hours |

Severity thresholds and response times are organization-specific; define them in the on-call policy. Silences require reason, owner, expiry and audit trail.

## 5. Executable lab
The [Chapter 064 companion lab](../labs/064-dashboards-synthetic-probes-on-call-check.md) provides a local synthetic inference HTTP server, a dependency-free Python probe, tests for success, bad status, malformed output and timeout, and a sample dashboard panel inventory. It **does not** contact an external LLM or consume GPU capacity.

## 6. Troubleshooting and scenarios
- **Dashboard green but users fail:** compare SLI scope, synthetic inference probe, streaming completion and regional segmentation.
- **Probe fails but real traffic succeeds:** check synthetic identity, quotas, source IP, certificate, test prompt and rate limits.
- **TTFT spike:** inspect queue time, prefill, KV cache, batch scheduling and gateway buffering.
- **Telemetry disappears:** check collector/exporter/scrape freshness and separate no-data alerts.
- **Probe cost grows:** cap cadence, output tokens, concurrency and dedicated budget.
- **On-call alert storm:** group common root causes, preserve critical pages, improve thresholds and record silences.

## 7. Production runbooks

### RB-064-A — Synthetic probe failure
**[SAFE-READ]** check probe region, DNS/TLS, HTTP status, auth, timeout, gateway and backend metrics. Verify whether real-user SLIs corroborate impact. **[CHANGE-CONTROL]** repair credentials, route or deployment using approved procedure; never paste probe secrets into incident tickets.

### RB-064-B — Streaming degradation
Measure first-token delay, chunk cadence and completion; correlate queue pressure, GPU saturation, timeouts and rollout history. Roll back or adjust capacity only with incident authorization.

### RB-064-C — Dashboard discrepancy
Compare panel query labels, units, time range, datasource, missing data and histogram aggregation. Verify recording-rule freshness and dashboards-as-code revision.

### RB-064-D — On-call handoff
Transfer active incidents, unresolved pages, budget burn, risky deployments, silences with expiry, dashboards and escalation contacts. Record acknowledgments; avoid informal ownership gaps.

## 8. Security, FinOps and acceptance
- **[SAFE-READ]** dashboard/alert inspection; **[LAB-MUTATE]** synthetic local test; **[CHANGE-CONTROL]** production probes, credentials and routing.
- No prompt, response text, secret, request ID or tenant identifier in metric labels.
- Probe budgets, rate limits and timeouts reviewed before production.
- [ ] User-facing SLI panels and freshness indicators documented.
- [ ] Synthetic probe validates status and response shape.
- [ ] Failure and timeout tests pass.
- [ ] Alert ownership, runbooks and escalation routes reviewed.
- [ ] Security/technical review and live execution tracked independently.

## 9. References
- [Prometheus alerting practices](https://prometheus.io/docs/practices/alerting/)
- [Prometheus blackbox exporter](https://github.com/prometheus/blackbox_exporter)
- [Grafana dashboard documentation](https://grafana.com/docs/grafana/latest/dashboards/)
- [Google SRE Workbook — Monitoring](https://sre.google/workbook/monitoring/)

**Next:** Chapter 065 — LLM Observability, Metrics and Tracing Capstone.
