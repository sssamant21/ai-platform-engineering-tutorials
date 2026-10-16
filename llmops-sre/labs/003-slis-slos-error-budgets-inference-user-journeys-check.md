# Chapter 003 — Hands-On Lab: SLIs, SLOs, Error Budgets and Inference User Journeys

**Part 01 | Chapter 003 | Status: Published plan; not executed**

## Objective

Build an auditable customer-journey SLI contract, calculate error budgets, draft safe Prometheus queries and alerts, and tabletop six inference reliability failures.

## Safety

**[SAFE-READ]** use only approved metrics and dashboards. **[LAB-MUTATE]** synthetic-only staging instrumentation requires authorization. **[CHANGE-CONTROL]** production alerts, releases and traffic shaping. Never record prompts, PHI, secrets or user IDs in metrics.

## Lab 0 — Preconditions

Record environment, UTC window, approved synthetic endpoint, model, owner, observability stack, existing metric names, data retention, telemetry freshness and review authority. Do not assume chapter metric names exist.

## Lab 1 — Journey and eligibility

Choose streaming interactive inference. Fill:

```text
Journey:
Request starts at:
Eligible request criteria:
Exclusions and their separate counters:
Good request criteria:
Valid complete SSE protocol marker:
Client cancel vs server truncation:
Retry semantics:
Prompt/output cohort buckets:
Measurement point and clock:
Owner and alert destination:
```

## Lab 2 — Budget arithmetic

Given a **synthetic** 30-day request sample of 1,000,000 eligible attempts, 600 failures and illustrative 99.9% SLO:

```text
Allowed failures = 1,000,000 × (1 − 0.999) = 1,000
Consumed = 600
Remaining = 400
Observed availability = (1,000,000 − 600) / 1,000,000 = 99.94%
Budget consumed = 60%
```

If a 5-minute window contains 100,000 eligible attempts and 1,000 failures, the error ratio is 1% and burn is 10× relative to 99.9%. Discuss whether the alert is sustained and whether traffic floors are satisfied.

## Lab 3 — Metrics audit

Use read-only Prometheus/Grafana queries in an approved environment, adapting to verified names:

```promql
sum(increase(llm_inference_eligible_requests_total{env="staging"}[5m]))
sum(increase(llm_inference_failed_eligible_requests_total{env="staging"}[5m]))
```

These are **proposed custom instrumentation names**. Confirm metric existence, labels, resets, gaps, cardinality and duplicate instrumentation before interpreting values.

## Lab 4 — Burn-rate policy

Draft (do not deploy) fast windows 5m/1h and medium 30m/6h, error-budget fraction, traffic floor, missing-telemetry detection, synthetic fallback, routing, runbook and release freeze decision. Include no-traffic behavior.

## Lab 5 — Complete-stream synthetic specification

Define a safe synthetic request (no sensitive content), authorized route, expected SSE event grammar, first-token deadline, stream completion, finish reason, maximum token budget and bounded probe frequency. Do not send traffic unless approved.

## Lab 6 — Six game days

| Game day | Mode | Expected signal |
|---|---|---|
| 01 SSE truncated despite HTTP 200 | Tabletop | incomplete stream |
| 02 429 misclassified | Tabletop | false availability |
| 03 metrics disappear | Tabletop | freshness alarm |
| 04 TTFT regression | Staging only if approved | TTFT cohort |
| 05 low-volume noisy burn | Tabletop | traffic-floor guard |
| 06 quality failure with green latency | Tabletop | evaluation gate |

## Sixteen failure checklist

F01 SSE; F02 TTFT; F03 TPOT; F04 429; F05 504; F06 ingress-only errors; F07 low traffic; F08 absent traffic; F09 scrape; F10 reset; F11 duplicate; F12 p95; F13 model revision; F14 cold start; F15 retries; F16 quality.

## Twelve acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G01 | User journey inventory | Pending |
| G02 | Eligibility/exclusion contract | Pending |
| G03 | SSE completion definition | Pending |
| G04 | TTFT/TPOT/E2E units | Pending |
| G05 | Cohort and cardinality review | Pending |
| G06 | Verified budget calculation | Pending |
| G07 | Multi-window burn policy | Pending |
| G08 | Missing-data and traffic-floor behavior | Pending |
| G09 | Alert owner and escalation | Pending |
| G10 | Release/error-budget policy | Pending |
| G11 | Incident tabletop evidence | Pending |
| G12 | Reviewer signoff | Pending |

## Evidence and cleanup

```text
UTC / environment / approval:
Model/region/journey:
SLI numerator/denominator/exclusions:
Sample traffic, failures, target, remaining budget:
Metric source, units and freshness:
Burn windows and thresholds:
Synthetic SSE completion rules:
Failure scenarios and game-day mode:
Findings, owners, next action, reviewer:
```

Read-only labs leave no resources. Stop approved synthetic traffic, sanitize evidence and keep acceptance pending until verified.

**Next: Chapter 004 — Hands-On Lab Environment and Safety Controls.**
