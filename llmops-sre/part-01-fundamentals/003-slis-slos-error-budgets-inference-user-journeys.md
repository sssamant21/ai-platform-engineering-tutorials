# Chapter 003 — SLIs, SLOs, Error Budgets and Inference User Journeys

**Part 01 — AI/LLM SRE Fundamentals | Chapter 003 of 100**

> **Publication:** Tutorial and companion lab drafted for GitHub. Illustrative thresholds are not approved production objectives. Technical/security review and live execution remain pending.

## Objectives

By completion, engineers can identify distinct inference user journeys, define eligible events and good events, calculate request-based and latency SLOs, measure complete streaming responses, interpret error budgets and burn rates, configure safe Prometheus-style observability, and apply error-budget policies during incidents and releases.

## 1. Start with customer journeys

An SLO must represent user experience, not simply Pod readiness. Different workloads have different latency and completion needs.

| Journey | User expectation | Critical failure |
|---|---|---|
| Interactive streaming chat | fast first token, steady tokens, complete answer | truncated SSE, long TTFT |
| Synchronous non-streaming API | usable final response within deadline | timeout, 5xx, invalid response |
| Batch summarization | correct durable result by deadline | partial output, lost job, bad artifact |
| Agent tool workflow | complete authorized tool sequence | repeated tool calls, loop, unsafe action |
| Retrieval-augmented response | grounded answer and acceptable latency | missing retrieval, hallucinated citation |
| Internal model gateway | authorized routing, quota fairness | wrong model/tenant, 429 policy defect |

For this chapter's core exercise, select one **approved synthetic** interactive chat endpoint. Define a separate objective for batch/agent workloads rather than blending incomparable traffic.

## 2. Establish the measurement contract

```text
request_id, service, environment, region, model_revision,
streaming_mode, prompt_length_bucket, output_length_bucket,
status_class, failure_category, completed_stream,
first_token_timestamp, completion_timestamp, observed_at
```

Use bounded-cardinality labels and no prompt text, token content, user ID, PHI or secret. Prefer exemplars/traces with access controls for individual request IDs.

Specify **measurement point**: client edge (best for customer experience), gateway, or serving runtime. These are not interchangeable. A gateway 504 may not appear in the runtime error counter. Define a common UTC window, clock-skew tolerance, retry treatment and telemetry freshness.

### Eligibility rules

A useful starting contract (must be approved by service owner):

- Include valid, authorized requests that the service agreed to serve, including internal capacity 429 and gateway 5xx.
- Exclude malformed input rejected before admission and intentional contractual quota enforcement, **only when separately metered and independently auditable**.
- Distinguish client cancellation from server disconnect; count premature server-terminated SSE as failure.
- Count retries as separate customer attempts for an attempt-based SLI; use a request-group identifier for an additional user-journey SLI.
- Classify timeouts, transport failures, partial streams, missing final protocol event and invalid output contract.
- Never mark unknown/missing telemetry as successful traffic.

Document how a request is considered complete for the actual streaming protocol. A valid HTTP 200 with an interrupted stream is a failure.

## 3. Core SLIs

**Request availability:**

```text
availability = good_eligible_requests / all_eligible_requests
error_ratio = 1 - availability
```

**Complete-stream success:**

```text
stream_completion = valid_complete_streams / eligible_stream_requests
```

**Latency success (request-based):**

```text
TTFT_success = eligible_requests_with_TTFT_under_target / eligible_requests
E2E_success = eligible_requests_with_valid_completion_under_target / eligible_requests
```

Failed and timed-out eligible requests should not silently disappear from latency success denominators. Where possible, use event-based good/total counters in addition to latency histograms.

**Token-level diagnostic metrics** (not necessarily SLOs): TTFT, time per output token (TPOT), inter-token latency (ITL), prefill/decode time, queue delay, token throughput, KV-cache occupancy and GPU VRAM. Define TPOT calculation and exclude non-comparable zero-token responses.

Use cohorts: model, region, streaming mode, bounded prompt/output size buckets, priority class and serving revision. Avoid one global percentile hiding long-prompt or cold-start regressions.

## 4. Percentiles versus request-based SLOs

A p95 TTFT of 1.5 seconds is a diagnostic summary, not automatically equivalent to “95% of requests complete within 1.5 seconds” unless the sample, eligibility, window and aggregation are consistent. Never average p95 values across replicas. Prefer aggregate histogram buckets or event-based counters.

A histogram for TTFT can approximate the fraction at a threshold; confirm units and buckets, and whether failed requests emit TTFT at all. Request-based good/total event counters often make error budgets more auditable.

## 5. SLO and budget examples

**Illustrative only**:

| SLI | Example objective | Window | Owner |
|---|---|---|---|
| Eligible request availability | 99.9% | rolling 30d | service owner |
| Valid complete streaming | 99.5% | rolling 30d | API/inference owner |
| TTFT within 2s for short-prompt interactive requests | 95% | rolling 7d | inference owner |
| E2E under 20s for defined short-output cohort | 95% | rolling 7d | application owner |

For 1,000,000 eligible requests and a 99.9% target, budget is 1,000 failed requests. If 600 have failed, 400 remain. Request-weighted budgets are **not automatically** minutes of downtime.

```text
budget_fraction = 1 - target
budget_requests = eligible_requests × budget_fraction
spent = failed_eligible_requests
remaining = budget_requests - spent
burn_rate = observed_error_ratio / budget_fraction
```

At 99.9%, an observed error ratio of 1% implies 10× burn. At 100 eligible requests with one failure, the ratio is 1%, but low-volume confidence and alert minimum-event thresholds must be considered.

## 6. Multi-window burn-rate alerting

Alert on **two windows** to distinguish a brief spike from sustained impact. Example windows for a 30-day SLO: 5m and 1h (fast), 30m and 6h (medium). Common illustrative burn multipliers include 14.4× for urgent and 6× for sustained high burn, but thresholds, traffic floors, evaluation intervals and notification policies require tuning against real data.

```text
fast_alert = burn(5m) > 14.4 AND burn(1h) > 14.4
medium_alert = burn(30m) > 6 AND burn(6h) > 6
```

For a 99.9% availability SLO, burn(5m) = error_ratio(5m) / 0.001.

**Do not deploy these as production alert rules without review.** Zero traffic is not 100% availability; alert separately on missing synthetic probes and telemetry gaps.

## 7. Prometheus reference patterns

The following metrics are **proposed instrumentation names**, not vendor-standard vLLM/Triton/TrueFoundry metrics. Implement them in the gateway or synthetic instrumentation first; verify actual labels.

```promql
# eligible requests per 5 minutes
sum(increase(llm_inference_eligible_requests_total{env="staging"}[5m]))

# failed eligible attempts (includes partial SSE and timeout)
sum(increase(llm_inference_failed_eligible_requests_total{env="staging"}[5m]))

# 5-minute failure fraction; clamp_min prevents division by zero,
# but guard separately against absent/low traffic before paging
sum(increase(llm_inference_failed_eligible_requests_total{env="staging"}[5m]))
/
clamp_min(sum(increase(llm_inference_eligible_requests_total{env="staging"}[5m])), 1)

# illustrative fast burn (0.1% budget)
(
  sum(rate(llm_inference_failed_eligible_requests_total{env="staging"}[5m]))
  /
  clamp_min(sum(rate(llm_inference_eligible_requests_total{env="staging"}[5m])), 0.001)
) / 0.001
```

In production, use consistent service/model cohort filtering and require matching metric availability. Counter monotonicity, restarts, missing series and event-time semantics should be tested. Do not sum request counters across gateway and runtime (double counting).

### Streaming probe

A synthetic probe must validate authorized access, expected content-type, first token deadline, token event parsing, completion marker, and final usage/finish reason where protocol guarantees them. Measure first-byte separately from first-token. Use non-sensitive synthetic prompts, controlled budgets and a bounded output length.

## 8. Error-budget policy and releases

Example policy, to be approved by owners:

- **Healthy budget:** proceed with progressive delivery and canary comparison.
- **Rapid burn:** freeze risky changes, page service owner and contain incident.
- **Budget exhausted:** pause nonessential releases, prioritize recovery work and approve exceptions explicitly.
- **Unknown telemetry:** block automatic promotion; validate independent probes.
- **Model-quality regression:** apply evaluation guardrail even if HTTP SLO remains green.
- **Security issue:** invoke security incident response irrespective of error budget.

Maintain separate availability and model-quality gates; a fast response can still be unsafe or incorrect.

## 9. Operational triage

```text
SLO alert
  → verify traffic and telemetry freshness
  → confirm customer-path synthetic
  → segment model/region/tenant-class/prompt cohort
  → classify gateway/serving/GPU/artifact/network failure
  → correlate rollout, capacity, queue, TTFT, TPOT, SSE
  → stop risky rollout or contain traffic under change control
  → verify good-event recovery and budget burn decay
  → incident timeline, owner, RCA/COA
```

## 10. Sixteen failure scenarios

| ID | Symptom | Investigation |
|---|---|---|
| F01 | HTTP 200 but truncated SSE | completion marker and client disconnect |
| F02 | high TTFT, low error rate | prefill, queue, warmup |
| F03 | high TPOT | decode, GPU, cache |
| F04 | 429 spike | contractual quota vs saturation |
| F05 | gateway 504 | timeout boundary |
| F06 | 5xx at ingress only | edge vs runtime counters |
| F07 | low traffic with noisy burn | minimum-event thresholds |
| F08 | absent traffic | synthetic and demand |
| F09 | metrics scrape failure | target freshness |
| F10 | counter reset | increase/rate handling |
| F11 | duplicate instrumentation | double counting |
| F12 | p95 averaging | histogram aggregation |
| F13 | model revision regression | cohort diff |
| F14 | cold start during rollout | warmup and readiness |
| F15 | client retry storm | original attempts and grouping |
| F16 | quality failure with green SLO | evaluation guardrail |

## 11. Ten SRE runbooks

1. Define and approve eligible event contract.
2. Validate synthetic complete-stream monitoring.
3. Investigate 5xx/429 classification.
4. Investigate TTFT regression.
5. Investigate TPOT/ITL regression.
6. Audit metric cardinality, completeness and reset handling.
7. Diagnose low-volume or missing-traffic alerts.
8. Respond to rapid multi-window burn.
9. Freeze/rollback a regressing release under ownership controls.
10. Conduct error-budget review, incident RCA and SLO recalibration.

Each runbook should record severity, owner, safe commands, evidence, containment, rollback, validation and escalation.

## 12. Six game days

1. Tabletop: truncated SSE returns HTTP 200.
2. Tabletop: internal overload 429 classified as success.
3. Tabletop: missing telemetry causes apparent zero errors.
4. Staging synthetic: controlled TTFT degradation, if approved.
5. Tabletop: noisy low-volume burn-rate alert.
6. Tabletop: model-quality failure with healthy latency.

## 13. Twelve acceptance gates

G01 journey catalog; G02 eligibility contract; G03 valid SSE completion; G04 TTFT/TPOT units; G05 cohort and cardinality plan; G06 budget arithmetic; G07 fast/medium burn policy; G08 traffic-floor and missing-data alerts; G09 owner/escalation; G10 canary release policy; G11 incident tabletop; G12 independent review.

See the [companion hands-on lab](../labs/003-slis-slos-error-budgets-inference-user-journeys-check.md). All gates are pending until evidence is collected and reviewed.

**Next: Chapter 004 — Hands-On Lab Environment and Safety Controls.**
