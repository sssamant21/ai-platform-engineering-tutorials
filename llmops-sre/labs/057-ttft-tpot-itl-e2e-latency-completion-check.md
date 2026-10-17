# Chapter 057 — Hands-On Lab: TTFT, TPOT, ITL, E2E Latency and Completion

**Part 08 | Chapter 057 | Published lab plan; live execution pending**

## Goal

Build a precise client/server latency contract, validate streamed token timing, completion semantics, histogram quantiles, SLO error budgets and privacy-safe tracing using synthetic traffic.

## Safety

**[SAFE-READ]** authorized metrics/Kubernetes inventory. **[LAB-MUTATE]** approved isolated staging, synthetic prompts and bounded load. **[CHANGE-CONTROL]** production instrumentation, exporters, alert rules and gateway behavior. No PHI, customer prompts, Secrets or raw completions in logs.

## Lab 0 — Preflight

Record UTC, context, model/server/exporter versions, gateway, tokenizer, streaming protocol, clock boundaries, SLO, histogram buckets, approvals, budget and rollback.

## Lab 1 — Metric and service inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get pods -A -o wide
kubectl get services -A
kubectl get servicemonitors -A 2>/dev/null || true
kubectl get podmonitors -A 2>/dev/null || true
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Discover actual metrics; canonical names in the chapter are design-only.

## Lab 2 — Latency math [CALCULATION]

Synthetic token emission times after server acceptance: `t1=0.8s`, `t2=0.9s`, `t3=1.1s`, `t4=1.2s`; terminal response `1.4s`.

```text
TTFT = 0.8 s
ITL = [0.1, 0.2, 0.1] s
TPOT = (1.2 - 0.8) / (4 - 1) = 0.1333 s
E2E_server = 1.4 s
```

A one-token response has undefined post-first-token TPOT. Record client and server timing separately.

## Lab 3 — Streaming completion [DESIGN]

Define terminal-valid marker, truncated stream, cancellation, timeout, 429, 5xx, invalid schema/tool output and retries. Test that HTTP 200 without a valid terminal marker is not a success.

## Lab 4 — PromQL histogram validation [DESIGN]

```promql
histogram_quantile(0.95,
  sum by (model, le) (rate(llm_ttft_seconds_bucket[5m]))
)
```

Map actual exporter names, confirm bucket monotonicity and SLO threshold coverage, inspect sample count and avoid comparing incompatible histogram definitions.

## Lab 5 — Trace privacy and clocks [DESIGN]

Design gateway/admission/queue/prefill/first-token/decode/terminal spans with sanitized model/version, token counts and outcome. Verify no prompt, completion, PHI, secrets or unbounded identifiers; account for cross-node clock skew.

## Lab 6 — Eight tabletop game days

01 queue TTFT; 02 decode ITL; 03 SSE chunks; 04 truncated stream; 05 single-token TPOT; 06 histogram p99; 07 clock skew; 08 metric rollback.

## Twenty-four failure checklist

F01 boundary; F02 queue; F03 client/server; F04 TPOT formula; F05 one token; F06 chunks; F07 tokenizer; F08 E2E flush; F09 length skew; F10 partial stream; F11 cancellation; F12 429; F13 invalid output; F14 retries; F15 buckets; F16 quantile; F17 sparse p99; F18 duplicate scrape; F19 cardinality; F20 privacy; F21 clock; F22 overhead; F23 alert; F24 rollback.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | Timestamp boundaries | Pending |
| G03 | Client/server TTFT | Pending |
| G04 | TPOT formula | Pending |
| G05 | ITL/tokenization | Pending |
| G06 | E2E | Pending |
| G07 | Terminal marker | Pending |
| G08 | Cancellations | Pending |
| G09 | 429/errors | Pending |
| G10 | Schema validity | Pending |
| G11 | Request classes | Pending |
| G12 | Histogram buckets | Pending |
| G13 | PromQL | Pending |
| G14 | Low traffic | Pending |
| G15 | Retry/dedup | Pending |
| G16 | OTel tracing | Pending |
| G17 | Privacy | Pending |
| G18 | SLO/alerts | Pending |
| G19 | Rollback/overhead | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
Gateway/model/exporter/tokenizer versions:
Server/client clock boundaries:
Synthetic first/last/terminal timestamps:
TTFT/TPOT/ITL/E2E:
Streaming completion taxonomy:
Histogram buckets/sample count/PromQL:
Retry/dedup/cancellation/429:
OTel propagation/privacy:
SLO/error budget/alerting:
Instrumentation overhead:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Approved staging tests must restore original instrumentation, exporters, alerts and sampling, remove only owned synthetic workloads, validate no sensitive data leaked and retain sanitized evidence. All gates remain pending until independent review.

**Next: Chapter 058 — vLLM, Triton and Ray Serve Metrics.**
