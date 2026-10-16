# Chapter 009 — Streaming, Backpressure and API Gateway Behavior

**Part 02 — LLM Architecture and Inference Internals | Chapter 009 of 100**

**Status:** Written tutorial published. Technical/security review, hands-on execution and operational acceptance remain pending.

## Learning objectives

Trace streaming inference through clients, gateways, proxies and runtimes; define valid SSE completion; diagnose buffering, backpressure and timeout failures; protect the inference fleet from slow consumers and retry storms; and design a safe synthetic streaming acceptance test.

## 1. The streaming request path

```text
Client → DNS/TLS → API gateway/auth/quota → ingress/reverse proxy
      → Kubernetes Service/EndpointSlice → inference runtime
      → tokenizer → prefill → decode → token events
      → proxy flush/buffering → client parser → final completion

Control plane: GitOps, operator, gateway configuration, model routing
Observability: synthetic client, edge metrics, runtime metrics, traces, GPU
```

Streaming is a **customer-visible protocol**, not simply an HTTP status. HTTP 200 may be returned before the first token; a subsequent transport failure can leave a partial answer with a successful status code.

## 2. SSE protocol fundamentals

Server-Sent Events typically use `Content-Type: text/event-stream` and UTF-8 text frames separated by a blank line. Fields can include `event:`, `data:`, `id:` and `retry:`. A single event may contain multiple `data:` lines. Keepalive comments beginning with `:` are allowed.

Many OpenAI-compatible inference APIs stream data events containing JSON and finish with a `data: [DONE]` sentinel; **this is an API-specific convention, not a universal SSE rule**. Define success using the actual deployed API contract, including finish reason, expected terminal marker and clean stream closure.

An SSE parser must handle fragmented TCP chunks, partial UTF-8 sequences, multiple frames in one read and event delimiters across reads. Do not assume each network chunk is one token or one SSE event.

## 3. First byte, first token and completion

| Signal | Measurement point | Operational meaning |
|---|---|---|
| TTFB | request to first response byte | edge/proxy responsiveness |
| TTFT | request to first generated output token | queue + prefill + delivery |
| ITL | successive output-token arrival gap | decode/scheduling/network |
| TPOT | defined average post-first-token duration | decode efficiency |
| E2E | request to valid final completion | whole customer journey |
| Stream completion | valid terminal event and protocol finish | correctness/availability |

A heartbeat or empty SSE event must not be mistaken for the first generated token. Request-based good/total counters should classify partial streams as failures even when HTTP status is 200.

## 4. Proxy buffering and flushing

Reverse proxies, ingress controllers, CDNs and gateways may buffer upstream data, compress responses or coalesce chunks. Buffering can make GPU decoding healthy while clients receive bursts of tokens and experience poor TTFT/ITL.

Investigate **per-hop**:
1. Runtime first-token timestamp.
2. Gateway upstream response timing.
3. Proxy buffering/compression behavior.
4. Client first valid token arrival.
5. Final SSE completion.

Buffering configuration is proxy-specific; verify the exact ingress/gateway implementation and release before proposing changes. Never disable buffering globally without considering memory, security and multi-tenant effects.

## 5. Backpressure and slow consumers

Backpressure occurs when a downstream consumer cannot drain streamed bytes as quickly as upstream generation produces them. TCP flow control, proxy queues and runtime streaming buffers can accumulate memory and hold request resources.

Possible policies: bounded per-connection buffers, read/write deadlines, maximum stream duration, cancellation propagation, client flow control, concurrency caps and tenant fairness. The runtime may continue generating tokens after the client disconnects unless cancellation propagates; this wastes GPU compute and token budget.

**Operational contract:** track active streams, bytes queued, disconnect reasons, cancellation propagation delay, wasted generated tokens, per-tenant concurrency and incomplete-stream rate.

## 6. Timeout alignment

| Boundary | Typical concern |
|---|---|
| Client overall deadline | customer gives up before completion |
| Gateway request timeout | stream terminated mid-generation |
| Proxy idle/read timeout | long gap between tokens appears idle |
| Upstream connect timeout | runtime cold start/connection |
| Runtime queue timeout | overload before scheduling |
| Model generation deadline | output budget/slow decode |
| Graceful shutdown/drain | in-flight streams interrupted |

Do not invent universal timeout values. Establish a documented deadline hierarchy, allow bounded headroom, and test legitimate long-generation requests. Heartbeats can maintain an idle connection but **must not** disguise a stalled model or override a contractual end-to-end deadline.

## 7. Retry, idempotency and quota behavior

Retrying a streaming generation can duplicate cost, produce different output and violate application semantics. A client should distinguish failures **before any output** from failures **after partial output**. Automatic replay after partial output needs an explicit product contract and should not silently concatenate incompatible responses.

Use exponential backoff with jitter, bounded attempts, circuit breakers and overload shedding. Treat contractual tenant-quota 429 differently from internal capacity 429 in SLO accounting; define and audit both.

## 8. Gateway security and privacy

Validate authentication before forwarding, model/tenant authorization, maximum request body and token budgets, safe CORS where applicable, TLS and certificate renewal, egress restrictions, prompt/log redaction, and rate limits. Avoid putting prompts, generated content, bearer tokens or user IDs in metric labels or generic access logs.

A request ID can aid tracing but must be stored in an access-controlled tracing system, not high-cardinality Prometheus labels.

## 9. Observability contract

Proposed **instrumentation categories**, not guaranteed vendor-native metric names:
- Eligible and failed streaming requests, including partial 200 responses.
- Complete SSE success and terminal finish-reason classification.
- TTFB, TTFT, ITL, E2E histograms by bounded cohorts.
- Client disconnect, upstream cancellation and wasted token counters.
- Active streams, proxy buffered bytes, gateway 429/5xx/504 and runtime queue.
- Synthetic probe freshness, trace sampling and missing telemetry.

Avoid averaging per-Pod p95s. Correlate by bounded model, region, gateway revision, streaming mode and prompt/output length buckets.

## 10. Read-only investigation

**[SAFE-READ]** after verifying authorization:

```bash
kubectl config current-context
kubectl get ingress,svc,endpointslices -A
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For approved namespace/deployment:

```bash
kubectl -n GATEWAY_NAMESPACE describe deployment GATEWAY_DEPLOYMENT
kubectl -n GATEWAY_NAMESPACE logs deployment/GATEWAY_DEPLOYMENT --since=15m --tail=100
```

Redact credentials and prompt content. Do not assume ingress objects exist when a cluster uses Gateway API or a managed edge; inspect the actual deployment architecture.

## 11. Failure decision tree

```text
Streaming request degraded?
 ├─ no connection → DNS/TLS/route/auth
 ├─ 401/403 → tenant/model permission
 ├─ 429 → quota versus capacity overload
 ├─ 502/503/504 → endpoints, upstream health, timeout
 ├─ 200 but no tokens → queue/prefill or buffering
 ├─ tokens arrive in bursts → proxy buffering/compression
 ├─ long inter-token gaps → decode, scheduler, network
 ├─ partial 200 stream → timeout, drain, crash, cancellation
 ├─ client disconnect but GPU busy → cancellation propagation
 ├─ retry storm → amplification and duplicate work
 └─ green HTTP metrics, failing synthetic → incomplete-stream SLI
```

## 12. Twenty production failure scenarios

| ID | Symptom | First check |
|---|---|---|
| F01 | DNS failure | resolution and route |
| F02 | TLS handshake failure | certificate/edge |
| F03 | 401/403 after policy change | auth/model entitlement |
| F04 | 429 quota misclassification | gateway policy |
| F05 | upstream 503 | endpoints/readiness |
| F06 | 504 before first token | deadline and queue |
| F07 | proxy buffers SSE | flush/buffering |
| F08 | compression delays tokens | content encoding |
| F09 | heartbeat counted as token | parser/SLI |
| F10 | fragmented UTF-8 parsing | client parser |
| F11 | multiple SSE events per chunk | framing parser |
| F12 | missing terminal marker | completion contract |
| F13 | HTTP 200 with truncated stream | client synthetic |
| F14 | idle timeout mid-generation | timeout hierarchy |
| F15 | slow consumer memory growth | proxy buffers |
| F16 | cancellation not propagated | runtime active requests |
| F17 | retry after partial output | idempotency contract |
| F18 | rolling deploy kills streams | draining policy |
| F19 | sensitive prompt in logs | redaction controls |
| F20 | stale telemetry hides failures | synthetic freshness |

## 13. Fourteen operational runbooks

1. Verify customer-path synthetic and API completion contract.
2. Diagnose DNS/TLS/auth failures.
3. Triage quota 429 versus overload 429.
4. Triage upstream 502/503/504.
5. Separate TTFB from TTFT.
6. Inspect proxy buffering/compression.
7. Validate SSE framing and UTF-8 parser.
8. Investigate partial HTTP 200 streams.
9. Align client/proxy/runtime deadlines.
10. Diagnose slow consumers and bounded buffers.
11. Verify cancellation propagation.
12. Contain retry amplification and token waste.
13. Protect in-flight streams during rollout/drain.
14. Validate recovery, privacy and error-budget impact.

Each runbook needs named owner, safe checks, escalation, approved mitigation, rollback and completion verification.

## 14. Eight game days

1. Proxy buffering causes delayed first token.
2. HTTP 200 ends without valid terminal event.
3. Slow client causes proxy buffer growth.
4. Client cancellation fails to stop generation.
5. Gateway idle timeout interrupts a long stream.
6. Rolling deployment terminates active streams.
7. Retry storm doubles generated token cost.
8. HTTP metrics stay green while SSE synthetic fails.

Default mode is **tabletop**. Synthetic staging tests require explicit approval, traffic/token limits, privacy review and cleanup.

## 15. Sixteen acceptance gates

G01 streaming architecture; G02 API-specific completion contract; G03 SSE framing; G04 TTFB/TTFT distinction; G05 buffering investigation; G06 backpressure policy; G07 cancellation propagation; G08 timeout hierarchy; G09 retry/idempotency policy; G10 429 classification; G11 privacy and auth; G12 synthetic stream validation; G13 observability freshness; G14 safe diagnostics; G15 rollback/runbook; G16 reviewer signoff.

All gates are pending until executed and reviewed. See [companion lab](../labs/009-streaming-backpressure-api-gateway-behavior-check.md).

**Next: Chapter 010 — Inference Internals Performance and Reliability Acceptance Project.**
