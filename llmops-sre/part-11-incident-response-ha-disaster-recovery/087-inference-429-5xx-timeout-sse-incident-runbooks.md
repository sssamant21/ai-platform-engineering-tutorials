# Chapter 087 — Inference 429, 5xx, Timeout and SSE Incident Runbooks

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 087 of 100**  
**Status:** Chapter and offline executable lab published. Live traffic replay, staging SSE verification and independent operational review pending.

## Objectives
Diagnose and mitigate production LLM inference failures involving HTTP 429, 5xx, timeouts and Server-Sent Events (SSE) streaming. Separate gateway policy denials from backend overload, trace request deadlines across services, recover streaming reliability and prevent retry amplification or unexpected GPU spend.

## 1. Inference request architecture
```text
client / agent
   |
   | HTTPS / SSE
   v
WAF / LB -> AI gateway -> token quota + budget ledger
                         |
                   model router
                         |
              inference scheduler / queue
                         |
                  GPU model server
                         |
              streamed tokens / usage
                         |
              SSE proxy -> client
```
Trace the entire path, including gateway, load balancer, proxy buffering, token admission, backend queue, GPU runtime, client disconnect and usage reconciliation. Do not log raw prompts or generated content while debugging.

## 2. Failure taxonomy
| Symptom | Likely causes | Evidence | First safe action |
|---|---|---|---|
| 429 with quota reason | per-tenant RPS, token budget, concurrency or spend ceiling | reason code, tenant tier, retry-after | verify entitlement; do not blindly increase |
| 429 with overload reason | admission shedding, bounded queue | queue age, GPU load, capacity | reduce burst and retries |
| 500 | application exception, model runtime failure | sanitized exception class, deployment revision | rollback recent change |
| 502 | bad upstream, unhealthy backend, protocol mismatch | LB upstream errors, health probes | inspect routing and endpoints |
| 503 | no ready capacity, circuit breaker, model loading | replica readiness, cold starts, GPU memory | route to healthy capacity |
| 504 | proxy deadline exceeded | hop-by-hop timeouts, backend queue, TTFT | locate timeout owner |
| client timeout | deadline mismatch, long context, network | client and server timestamps | align deadlines |
| SSE stalls | proxy buffering, flush, heartbeat, backend starvation | time-to-first-event, inter-event gap | validate streaming path |
| SSE disconnect | client abort, LB idle timeout, connection churn | disconnect reason, usage reconciliation | stop work and account usage |

Status codes can vary by provider and gateway implementation; reason codes and distributed traces are more reliable than code alone.

## 3. Triage sequence (first 10 minutes)
1. Establish UTC onset, affected region/model/tenant tier and customer symptoms.
2. Compare total requests, 429 by reason, 5xx, timeout rate, p95/p99 latency, TTFT and inter-token latency.
3. Inspect changes to gateway policies, model revision, autoscaler, ingress and network.
4. Compare queue depth, concurrency, KV cache/GPU memory, CPU tokenization and replica readiness.
5. Verify SSE response headers, proxy buffering, idle timeouts and client reconnect behavior.
6. Review retries, idempotency and cost reservations to detect amplification.
7. Pick lowest-risk mitigation with an incident commander if severity warrants.
8. Validate recovery with synthetic and real-user SLOs, including completed streams.

## 4. Gateway 429 diagnosis
Separate **policy-limited** from **capacity-limited** requests. Policy-limited traffic may be correct even if clients complain; do not bypass tenant budgets. Capacity-limited traffic can be mitigated with bounded admission, queue shedding, additional healthy capacity or approved routing to cheaper/alternate models. `Retry-After` must reflect actual availability when possible. Clients should honor exponential backoff with jitter, deadlines and a retry budget; never retry an exhausted spend budget indefinitely.

## 5. 5xx and timeout diagnosis
Measure elapsed time at each hop: ingress, auth, quota check, queue wait, model load, prefill, decode and response write. Distinguish connection timeout, request deadline, proxy idle timeout and backend execution timeout. Streaming may run longer than a non-streaming request; align client, gateway, ingress and backend policies without creating unbounded requests. Use deadline propagation and cancellation to release GPU work where supported.

## 6. SSE streaming contract
A typical SSE response uses `Content-Type: text/event-stream`, UTF-8, `data:` fields and blank-line event delimiters. Keep intermediaries from buffering application events, emit safe keepalives where supported, handle disconnects and avoid double billing on reconnect. Completion markers are implementation-specific; do not assume every provider emits `[DONE]`. Partial streams are not necessarily safe to replay or resume without application-level idempotency.

### Synthetic stream probe
```bash
curl --no-buffer --max-time 30 -H 'Accept: text/event-stream' \
  'http://127.0.0.1:8080/synthetic-stream'
```
This endpoint is illustrative. In staging, use a controlled test endpoint with no real user content. Inspect first-event delay, event framing, idle gaps and final completion signal.

## 7. Observability and alerting
Track `inference_requests_total` by status/reason/model tier, queue age, time-to-first-token, time-per-output-token, stream completion and abort counts, upstream 502/503/504, active requests, GPU memory and KV cache, retry amplification, outstanding budget reservations and cost per good request. Use privacy-safe low-cardinality labels. Alert on sustained burn and customer impact rather than single transient 429 events.

## 8. Incident runbooks

### RB-087-A — Sustained 429
Verify policy reason and tenant scope; distinguish correct quota enforcement from overload; inspect retry storms; apply bounded retries and approved quota/capacity changes. Validate budget ledger and fairness.

### RB-087-B — Inference 5xx surge
Compare deployment timeline, readiness and upstream health. Isolate failing revision, restore known-good model/gateway configuration, monitor 5xx, latency and GPU memory.

### RB-087-C — Timeout/504 escalation
Correlate client/gateway/backend deadlines with queue wait, prefill and decode. Shed excessive concurrency, restore capacity or revert recent change. Avoid simply raising all timeouts.

### RB-087-D — SSE stalls/disconnects
Check `text/event-stream`, proxy buffering, idle timeout, heartbeat support, client abort handling and completion accounting. Reproduce with synthetic curl probe; validate full-stream success after fix.

### RB-087-E — Retry and cost amplification
Identify retry loops and duplicate work; apply per-workflow retry budgets, idempotency, cancellation and cost admission controls from Chapter 082. Reconcile outstanding reservations.

## 9. Failure injection game day
| Scenario | Expected response |
|---|---|
| quota 429 | no uncontrolled retry |
| overload 429 | bounded backoff and shedding |
| backend 503 | failover only to approved healthy capacity |
| 504 after long queue | identify queue wait and deadline owner |
| SSE delayed first event | TTFT alert and proxy/backend diagnosis |
| SSE missing final marker | incomplete-stream detection |
| client disconnect | cancellation and usage reconciliation |
| 5xx after model rollout | approved rollback |
| recursive agent retry | spend ceiling and loop termination |
| gateway exporter outage | retain safe local evidence without raw prompts |

## 10. Acceptance (100 points)
| Domain | Points |
|---|---:|
| HTTP status/reason classification | 20 |
| End-to-end latency and timeout diagnostics | 20 |
| SSE framing, streaming and disconnect handling | 20 |
| Safe mitigations, retry/cost controls | 20 |
| Tests, communications, rollback and evidence | 20 |
| **Total** | **100** |

Critical blockers: unbounded retries, bypassed spend/tenant controls, streaming privacy leakage or a recovery change that breaks isolation. Educational target >=90/100. Staging traffic replay and production sign-off pending.

## 11. Companion lab
[Chapter 087 — Executable Inference 429, 5xx, Timeout and SSE Incident Lab](../labs/087-inference-429-5xx-timeout-sse-incident-runbooks-check.md). Offline deterministic triage and SSE framing tests; not a real inference server or network probe.

## References
- [RFC 6585 — HTTP 429](https://www.rfc-editor.org/rfc/rfc6585.html)
- [MDN — Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [OpenTelemetry Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)

**Next:** Chapter 088 — GPU XID, ECC, Driver and CUDA Incident Response.
