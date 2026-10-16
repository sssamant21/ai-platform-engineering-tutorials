# Chapter 019 — Hands-On Lab: vLLM Streaming and OpenAI-Compatible APIs

**Part 04 | Chapter 019 | Written lab published; not executed**

## Goal

Build a robust streaming API acceptance plan, validate SSE framing and completion, classify partial HTTP 200 streams, test cancellation/retry safety, and correlate TTFT/TPOT with gateway, scheduler and GPU behavior.

## Safety

**[SAFE-READ]** approved ingress and vLLM configuration, metrics and redacted logs. **[LAB-MUTATE]** synthetic requests against isolated staging with approved token, concurrency and GPU-hour limits. **[CHANGE-CONTROL]** production gateway timeouts, buffering, auth, retry and serving changes. Never log secrets, customer prompts or model output.

## Lab 0 — Preflight

Record UTC, context/namespace, vLLM version/image digest, model alias/tokenizer/template, gateway, authentication method, expected streaming contract, cost ceiling, owner and rollback.

```bash
kubectl config current-context
kubectl get pods,deploy,svc,ingress -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — API contract worksheet [DESIGN]

| Field | Expected verification |
|---|---|
| Endpoint | pinned vLLM route |
| Model alias | approved served model |
| Auth | gateway/backend policy |
| Chat template | synthetic role fixture |
| Stream content type | text/event-stream |
| Chunk schema | endpoint/version-specific |
| Finish reason | valid terminal state |
| Terminal marker | complete SSE contract |
| Timeouts | gateway/client/backend |
| Cancellation | work release |
| Retry | safe before/after first chunk |
| Observability | sanitized request correlation |

## Lab 2 — Non-streaming comparison [DESIGN; optional staging]

For approved staging only:

```bash
curl --fail-with-body --max-time 120 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $STAGING_API_TOKEN" \
  -d '{"model":"APPROVED_MODEL_ALIAS","messages":[{"role":"user","content":"Say READY."}],"max_tokens":32,"stream":false}' \
  "$STAGING_BASE_URL/v1/chat/completions"
```

Validate status, schema, model alias, finish reason and JSON completeness. Do not include authorization headers in saved evidence.

## Lab 3 — SSE validation [DESIGN; optional staging]

```bash
curl --fail-with-body --no-buffer --max-time 120 \
  -H "Content-Type: application/json" \
  -H "Accept: text/event-stream" \
  -H "Authorization: Bearer $STAGING_API_TOKEN" \
  -d '{"model":"APPROVED_MODEL_ALIAS","messages":[{"role":"user","content":"Say READY."}],"max_tokens":32,"stream":true}' \
  "$STAGING_BASE_URL/v1/chat/completions"
```

Capture sanitized event timing only. Confirm complete SSE event boundaries, JSON chunks, finish reason, terminal [DONE] and clean close. A successful curl exit does not replace parser-level completion verification.

## Lab 4 — Offline SSE parser acceptance [DESIGN]

Test a parser against synthetic fixtures (no network required):
- Event split across two transport reads.
- Two SSE events in one read.
- UTF-8 code point split across reads.
- LF and CRLF delimiters.
- Comment/heartbeat line.
- Multi-line data field.
- Malformed JSON event.
- Finish reason but missing [DONE].
- [DONE] followed by unexpected data.
- Connection close after HTTP 200 but before completion.

Mark a stream successful only when the version-specific completion contract is met.

## Lab 5 — Gateway and cancellation [TABLETOP]

Review buffering, response flushing, connect/header/idle/total deadlines, retry behavior, client cancellation propagation and backend work release. Define a bounded staging plan for a disconnect mid-generation; do not execute fault injection on production.

## Lab 6 — SLO and FinOps worksheet [DESIGN]

Synthetic 100,000 eligible streams, 300 failures, 99.9% SLO:

```text
allowed failures = 100
observed success = 99.7%
error-budget burn = 3×
```

Record TTFT, TPOT/ITL, E2E, complete-stream success, cancellation rate, 429/5xx, retries and GPU-hours per completed valid request. Distinguish deliberate cancellation from involuntary incomplete streams in the approved SLI contract.

## Lab 7 — Eight tabletop game days

01 buffering; 02 idle timeout; 03 split UTF-8; 04 crash after chunks; 05 retry storm; 06 cancellation leak; 07 invalid tool-call stream; 08 green HTTP 200 with failed SSE.

## Twenty-four failure checklist

F01 API field; F02 alias; F03 template; F04 context; F05 auth; F06 policy; F07 quota; F08 overload; F09 upstream; F10 timeout; F11 buffering; F12 compression; F13 UTF-8; F14 multi-line; F15 terminal; F16 finish reason; F17 partial 200; F18 cancellation; F19 retry; F20 TTFT; F21 TPOT; F22 tool call; F23 privacy; F24 GPU cost.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned vLLM release | Pending |
| G02 | API endpoint/schema | Pending |
| G03 | Model alias | Pending |
| G04 | Authentication | Pending |
| G05 | Quotas/admission | Pending |
| G06 | SSE framing | Pending |
| G07 | Incremental UTF-8 parsing | Pending |
| G08 | Finish reason | Pending |
| G09 | Terminal completion | Pending |
| G10 | HTTP 200 partial failure | Pending |
| G11 | TTFT | Pending |
| G12 | TPOT | Pending |
| G13 | Proxy timeouts/buffering | Pending |
| G14 | Cancellation propagation | Pending |
| G15 | Retry safety | Pending |
| G16 | Tool/JSON streaming | Pending |
| G17 | Privacy | Pending |
| G18 | SLO and GPU FinOps | Pending |
| G19 | Rollback | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/authorization:
vLLM/model/gateway versions:
Endpoint/auth/model alias:
Non-streaming response validation:
SSE parser fixtures and terminal evidence:
TTFT/TPOT/E2E/complete-stream SLI:
429/5xx/cancellation/retry:
GPU cost and privacy:
Game-day mode and findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop separately approved synthetic load, remove only owned staging artifacts and reconcile GPU-hours. All acceptance gates remain pending until independently reviewed.

**Next: Chapter 020 — vLLM GPU Memory and Context Tuning.**
