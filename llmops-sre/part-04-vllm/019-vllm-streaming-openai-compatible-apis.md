# Chapter 019 — vLLM Streaming and OpenAI-Compatible APIs

**Part 04 — vLLM Administration and Operations | Chapter 019 of 100**

> **Status:** Tutorial published. Lab execution, technical/security review and operational acceptance remain pending. Endpoint support, OpenAI-compatible behavior, streaming fields and metric names depend on the pinned vLLM release.

## Learning objectives

Operate a version-pinned vLLM OpenAI-compatible serving interface; distinguish non-streaming JSON from Server-Sent Events (SSE); validate terminal stream completion; troubleshoot proxy buffering, timeouts, disconnects and cancellation; define request/response contracts, authentication, admission and observability; safely test streaming failure recovery and cost.

## 1. End-to-end API architecture

```text
Application client
  ├─ request ID / auth / model alias / deadline
  ├─ JSON non-streaming or SSE streaming
  ▼
TLS ingress / API gateway
  ├─ authentication, quotas, request-size limits
  ├─ retry policy, proxy buffering and idle timeout
  └─ tenant-aware routing
  ▼
Kubernetes Service → vLLM API server
                      ├─ request validation/tokenization
                      ├─ scheduler/queue/continuous batching
                      ├─ prefill/decode GPU engine
                      └─ token chunks / finish reason
  ▼
SSE client parser → validated terminal completion
  ▼
Customer-path success SLI + tracing + GPU/runtime metrics
```

An HTTP 200 response confirms only that headers were returned successfully. It does **not** establish that all expected stream events arrived, that the generation completed, or that a client consumed the answer.

## 2. OpenAI-compatible API contract

vLLM exposes an OpenAI-compatible API for supported models/endpoints. Compatibility is not identical feature parity with every OpenAI API version. Verify against your pinned vLLM release:

| Contract element | Verification |
|---|---|
| Endpoint and method | supported route and JSON schema |
| Model name | approved served-model alias |
| Authentication | gateway and backend trust boundary |
| Chat template | model-specific roles and formatting |
| Token limits | context + max output admission |
| Sampling | temperature/top-p/seed support and reproducibility caveats |
| Tool calling | supported parser, template and schema |
| Structured outputs | JSON/schema correctness and unsupported options |
| Streaming | event order, finish reason, terminal marker |
| Usage | whether and when usage appears for the chosen mode |
| Error codes | 400, 401, 403, 404, 408, 429, 5xx semantics |
| Cancellation | downstream disconnect and GPU resource release |

Never assume a parameter supported by one serving backend is accepted by another. Unsupported fields should fail predictably rather than being silently ignored.

## 3. Non-streaming versus streaming

**Non-streaming:** a JSON response is delivered when generation completes or errors. The client can validate the full body and status together.

**Streaming:** response chunks arrive over SSE, usually with content type `text/event-stream`. A standard SSE event uses lines such as `data: {...}` followed by a blank line. Chat completion chunks can carry incremental deltas, an eventual finish reason, and a terminal `data: [DONE]` marker in common OpenAI-compatible streaming behavior.

```text
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"id":"example","choices":[{"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"example","choices":[{"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

**Illustrative, not a recorded vLLM response.** Exact chunk structure, finish reasons and usage fields depend on endpoint and release. A terminal marker without a valid finish state may still require contract-specific investigation. Do not assume each network read corresponds to one complete SSE event.

## 4. Correct SSE parsing and completion

A production client must:
1. Decode UTF-8 incrementally across arbitrary TCP chunks.
2. Buffer until an SSE event delimiter, not until one socket read.
3. Support CRLF and LF line endings and `data:` fields, with multi-line data when applicable.
4. Ignore permitted comment/heartbeat lines.
5. Parse JSON only after a complete event is assembled.
6. Track model ID, request ID, chunk order, output tokens and finish state.
7. Treat transport close before the expected terminal event as **incomplete**, even after HTTP 200.
8. Enforce absolute and idle deadlines, bounded buffers and cancellation.
9. Avoid duplicating already streamed content when retrying.
10. Record sanitized completion evidence without storing prompts or sensitive output.

Some proxies compress, buffer or coalesce data; network chunk boundaries are not SSE event boundaries.

## 5. Synthetic streaming API test

**[LAB-MUTATE]** Only against an approved isolated staging endpoint, with a synthetic prompt and approved token:

```bash
curl --fail-with-body --no-buffer --max-time 120 \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -H "Authorization: Bearer $STAGING_API_TOKEN" \
  -d '{"model":"APPROVED_MODEL_ALIAS","messages":[{"role":"user","content":"Say READY."}],"max_tokens":32,"stream":true}' \
  "$STAGING_BASE_URL/v1/chat/completions"
```

The model alias and endpoint must be verified for the pinned release. Do not echo credentials or write request headers into logs. Curl is useful for visual inspection, but automated acceptance needs a robust SSE parser and explicit terminal-event checks.

A corresponding non-streaming test uses `"stream":false` and validates the complete JSON body, model alias, finish reason and response schema.

## 6. Gateway and ingress behavior

A streaming-capable ingress must preserve timely flushing and reasonable connection lifetime while enforcing security:

- TLS, authentication and request-size limits.
- No unwanted response buffering; verify the *actual* gateway behavior.
- Explicit connect, header, idle and total request deadlines.
- Client disconnect propagation and backend cancellation.
- Consistent request IDs and trace propagation.
- Tenant-aware quotas, token budgets and 429 behavior.
- Limits on retry amplification and duplicate generation.

Do not blindly disable buffering globally or set infinite timeouts. Review gateway-specific syntax and security policy. Long generations can exceed idle timeouts even when backend compute is healthy.

## 7. Retry and idempotency

A client may retry a request **before** any output is consumed if policy allows. Once chunks have been delivered, replaying the request can produce duplicate text, additional cost and inconsistent outputs. Treat streaming inference as **non-idempotent by default** unless the application implements an explicit deduplication/replay contract.

Define a retry matrix:

| Failure | Recommended classification |
|---|---|
| 401/403 | auth/configuration; do not blind retry |
| 400 invalid request | fix schema/token budget |
| 429 | bounded backoff, quota-aware admission |
| 502/503 before chunks | bounded retry if safe |
| 504 before chunks | inspect timeout and server state |
| disconnect after chunks | incomplete; avoid transparent replay |
| backend completes, client cancels | cancellation accounting and resource release |
| 200 with missing terminal event | failed complete-stream SLI |

Never log bearer tokens or raw customer prompts during retry debugging.

## 8. Streaming SLIs and SLOs

Define **eligible requests** and exclude only explicitly agreed categories (for example intentional client cancellations, with a separate cancellation SLI). Recommended measurements:

- **Complete-stream success rate:** terminally completed and schema-valid streams / eligible streams.
- **TTFT:** elapsed time from a defined client or gateway start until first *meaningful generated token*, not merely HTTP headers.
- **TPOT/ITL:** time between generated output tokens, excluding metadata-only events.
- **E2E latency:** request start to validated completion.
- **429/5xx:** distinguish quota rejection, overload and backend failure.
- **Cancellation propagation:** time from disconnect to release of engine work.
- **GPU cost per completed valid request:** include partial streams and retries.

Prometheus histogram labels should use bounded model/route/status classes, never raw prompt text, user IDs or unbounded request IDs.

## 9. Example failure budget

Synthetic: 99.9% complete-stream success target, 100,000 eligible streams in a window, 300 incomplete or failed streams:

```text
Allowed failures = 100,000 × 0.001 = 100
Observed success = 99,700 / 100,000 = 99.7%
Failure-rate burn = 0.3% / 0.1% = 3×
```

This is an illustrative calculation. Alert policies should use real traffic volume, multi-window burn and minimum sample safeguards.

## 10. Read-only operational investigation

**[SAFE-READ]** on an approved Kubernetes cluster:

```bash
kubectl config current-context
kubectl get pods,deploy,svc,ingress -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For a verified workload:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Inspect gateway response buffering, timeouts, ingress target health, backend queue, token timing, cancellations and GPU/KV cache metrics using approved dashboards. Sanitize log evidence. Never enable full request/response body logging in production to debug SSE.

## 11. Failure decision tree

```text
Streaming response broken?
 ├─ no HTTP response → DNS/TLS/ingress/upstream
 ├─ 401/403 → auth and identity
 ├─ 400 → schema, model alias, context
 ├─ 429 → tenant quota vs backend admission
 ├─ 502/503 → service endpoints, readiness, crashes
 ├─ 504 → gateway/backend timeout
 ├─ headers fast, first token slow → queue/prefill
 ├─ chunks delayed in bursts → proxy buffering
 ├─ tokens slow → decode/KV/GPU contention
 ├─ JSON parse errors → SSE framing/client parser
 ├─ stream closes before [DONE] → timeout/cancel/crash
 ├─ missing finish reason → endpoint contract
 ├─ client disconnect leaks work → cancellation propagation
 ├─ duplicated answer after retry → non-idempotent replay
 └─ 200 appears healthy → verify terminal completion SLI
```

## 12. Production change and rollback

1. Freeze pinned vLLM image, model/tokenizer/template, API contract and gateway configuration.
2. Obtain approval, set token/cost limits, SLO thresholds and rollback owner.
3. Run synthetic non-streaming and streaming contract tests in isolated staging.
4. Test short/long generations, timeout boundaries and cancellation.
5. Compare first-token latency, inter-token latency, complete SSE, 429/5xx and GPU-hours.
6. Canary gateway/backend changes with separate cohort visibility.
7. Abort on incomplete streams, auth regressions, tail-latency burn or cancellation leaks.
8. Roll back through the authoritative GitOps/ingress controller.
9. Validate terminal completion and sustained customer-path recovery.
10. Preserve sanitized RCA/COA evidence.

## 13. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | unsupported API parameter | 400/schema |
| F02 | wrong served-model alias | 404/model list |
| F03 | chat template mismatch | synthetic fixtures |
| F04 | tokenizer/context overrun | admission logs |
| F05 | auth token invalid | 401 |
| F06 | gateway policy denies request | 403 |
| F07 | tenant quota exhausted | 429 |
| F08 | backend overloaded | queue/429/503 |
| F09 | ingress upstream unavailable | 502/503 |
| F10 | gateway idle timeout | 504/early close |
| F11 | proxy buffers SSE | delayed first chunk |
| F12 | compression breaks flush | chunk timing |
| F13 | partial UTF-8 decode bug | parser errors |
| F14 | multi-line SSE mishandled | framing |
| F15 | terminal [DONE] missing | completion validator |
| F16 | finish reason missing/invalid | schema |
| F17 | HTTP 200 partial stream | completion SLI |
| F18 | cancellation not propagated | active engine requests |
| F19 | retry duplicates partial answer | request tracing |
| F20 | slow TTFT from prefill | prompt cohorts |
| F21 | high TPOT from KV pressure | decode/cache |
| F22 | tool-call stream invalid | structured-output eval |
| F23 | sensitive output logged | privacy audit |
| F24 | retries increase GPU cost | good-request FinOps |

## 14. Sixteen operational runbooks

1. Validate API contract and pinned version.
2. Investigate model alias and chat-template errors.
3. Diagnose authentication and authorization.
4. Diagnose 429 quotas and overload.
5. Triage 502/503/504 and ingress endpoints.
6. Diagnose proxy buffering and flush behavior.
7. Validate SSE parser/framing.
8. Investigate missing terminal marker.
9. Investigate HTTP 200 incomplete streams.
10. Investigate TTFT/queue/prefill.
11. Investigate TPOT/decode/KV.
12. Diagnose cancellation resource leaks.
13. Prevent duplicate retries and cost amplification.
14. Validate tool/JSON streaming and privacy.
15. Execute approved gateway/backend rollback.
16. Verify sustained complete-stream SLO and RCA.

## 15. Eight tabletop game days

1. Gateway buffers all SSE chunks until completion.
2. Idle timeout terminates a long generation.
3. Client parser splits UTF-8 code points incorrectly.
4. Backend crashes after HTTP 200 and two chunks.
5. 429 retry storm overloads the GPU.
6. Client disconnect fails to cancel engine work.
7. Tool-call streaming produces invalid JSON.
8. Rollout shows green HTTP 200 but complete-stream SLO burns.

Default **tabletop**; live fault injection requires approved staging and bounded GPU-hours.

## 16. Twenty acceptance gates

G01 pinned vLLM release; G02 endpoint schema; G03 model alias; G04 authentication; G05 quota/admission; G06 SSE framing; G07 UTF-8 incremental parsing; G08 finish reason; G09 terminal completion; G10 HTTP 200 partial classification; G11 TTFT; G12 TPOT; G13 proxy timeout/buffering; G14 cancellation; G15 retry safety; G16 structured output; G17 privacy; G18 SLO/FinOps; G19 rollback; G20 independent review.

See [companion lab](../labs/019-vllm-streaming-openai-compatible-apis-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 020 — vLLM GPU Memory and Context Tuning.**
