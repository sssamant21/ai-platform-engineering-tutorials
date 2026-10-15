# Chapter 11.6 — Dashboards, SLIs, SLOs, Alerting and Error Budgets

**Part 11 — Observability | Canonical production SRE tutorial**

## Objectives and change controls

Build actionable reliability dashboards and service-level objectives (SLOs) for TrueFoundry-hosted inference, including streaming responses, Kubernetes/GPU health, error budgets, multi-window burn-rate alerts and on-call escalation. Define what constitutes a successful request, measure user impact, validate instrumentation, and test alert routing without generating production incidents.

**[SAFE-READ]** inspect metrics and alerts; **[LAB-MUTATE]** synthetic staging; **[CHANGE-CONTROL]** production SLOs, routing and alert rules. Verify deployed metric names and label schema, TrueFoundry integrations, Prometheus/Alertmanager/Grafana versions and model runtime instrumentation. Never expose prompts, responses, patient data, secrets or user IDs in metrics labels or alert annotations.

## 1. Service-level terminology

- **SLI:** numerical indicator of user experience (for example successful eligible inference requests / eligible inference requests).
- **SLO:** target for an SLI over a defined rolling or calendar window.
- **SLA:** external contractual commitment; do not equate it with an internal SLO.
- **Error budget:** tolerated unreliability within the SLO window.
- **Burn rate:** how quickly error budget is consumed relative to a uniform spending rate.

An SLO is a decision tool, not a promise that every request meets a latency threshold. Define exclusions and low-traffic handling with service owners. Distinguish service-level, platform-level and dependency-level responsibilities.

## 2. Inference SLI design

| SLI | Numerator | Denominator | Important boundary |
|---|---|---|---|
| Availability | Eligible successful completed requests | All eligible requests | Exclude explicitly defined client errors; count timeout/cancel policy consistently |
| Request latency | Eligible requests completing within target | Eligible completed requests or all eligible requests per approved policy | Include failures/timeouts as specified; don't silently exclude bad outcomes |
| Streaming TTFT | Eligible streams with first token within target | Eligible streaming requests | Handle aborted/no-token streams explicitly |
| Streaming completion | Eligible streams successfully completed | Eligible started streams | HTTP 200 before stream completion is not success |
| Model deployment readiness | Desired ready replicas at required capacity | Desired replicas | Separate infrastructure SLO |
| Telemetry coverage | Healthy expected scrape/collector sources | Expected sources | Monitoring SLO, not customer availability |

Define request identity, observation point (gateway or client), retry semantics, time window, model/tenant aggregation, timeouts and 4xx treatment. A retry may improve end-user success while inflating backend request counts; measure both when possible. If a metric cannot distinguish completed streams from started streams, do not claim to measure completion reliability.

## 3. SLO and error-budget math

For a 30-day rolling window and **99.9% availability SLO**, the allowed failed-request fraction is **0.1%**. For request-based SLIs:

```text
error_budget_fraction = 1 - SLO_target
budgeted_bad_requests = eligible_requests * error_budget_fraction
remaining_budget = budgeted_bad_requests - observed_bad_requests
burn_rate = observed_bad_fraction / error_budget_fraction
```

With 10,000,000 eligible requests, the budget is 10,000 bad requests. If 3,000 bad requests have occurred, 7,000 remain, assuming the counts and window align. Do not convert request-based budgets into fixed minutes of downtime; time-based and request-based SLOs are different.

For a 99.9% SLO, a 5% bad-request ratio yields burn rate 50. Sustained burn rate 14.4 for one hour consumes roughly 2% of a 30-day budget (14.4 × 1/720 ≈ 0.02). Multi-window alerts require a short and long window to confirm persistent impact.

## 4. Metrics contract and PromQL

These are **illustrative custom metrics**, not asserted TrueFoundry/vLLM names:

```promql
sum(rate(inference_requests_total{eligible="true",outcome="bad"}[5m]))
/
clamp_min(sum(rate(inference_requests_total{eligible="true"}[5m])), 1e-9)
```

```promql
sum(rate(inference_requests_total{eligible="true",outcome="bad"}[1h]))
/
clamp_min(sum(rate(inference_requests_total{eligible="true"}[1h])), 1e-9)
```

Use verified labels, instrumented stream completion, consistent denominators and adequate request volume. A missing metric is **unknown**, not 0% errors. Treat `clamp_min` as protection against division by zero, not a substitute for low-volume/no-data checks. For latency SLIs, histogram buckets can estimate a proportion under a threshold; classic histogram buckets must contain a suitable `le` boundary.

## 5. Multi-window multi-burn alerts

A common starting pattern for a 30-day 99.9% SLO is:
- **Fast page:** burn rate greater than 14.4 over both 1 hour and 5 minutes.
- **Slow page:** burn rate greater than 6 over both 6 hours and 30 minutes.
- **Ticket:** burn rate greater than 3 over both 1 day and 2 hours, or greater than 1 over both 3 days and 6 hours.

These are design examples, not universal defaults. Validate expected volume, budget consumption, notification policy and SLO window with the service owner. Prefer precomputed recording rules for the bad-request ratio and gate on minimum eligible request count. Ensure alerts evaluate both windows using the same SLI definition and labels.

Example **illustrative** Prometheus alert (assumes the named recording rules exist and have consistent `service` labels):

```yaml
groups:
- name: inference-slo-alerts
  rules:
  - alert: InferenceAvailabilityFastBurn
    expr: |
      (service:inference_bad_ratio:rate1h > 14.4 * 0.001)
      and on(service)
      (service:inference_bad_ratio:rate5m > 14.4 * 0.001)
      and on(service)
      (service:inference_eligible_requests:increase1h > 100)
    for: 2m
    labels:
      severity: page
      team: model-serving
    annotations:
      summary: "Inference availability error budget burning rapidly"
      runbook_url: "https://example.invalid/runbooks/inference-slo"
```

Replace the placeholder runbook URL with an approved real URL before deploying. The example is an ordinary Prometheus rule file; for Prometheus Operator, wrap under `PrometheusRule.spec.groups` and supply the CRD metadata/selector. Check with `promtool check rules`, and confirm Alertmanager routing. A high burn-rate alert can fire even when GPU utilization is normal.

## 6. Grafana dashboard architecture

**Executive reliability:** SLO target, 30-day attainment, budget consumed/remaining, incidents and user impact.

**Gateway RED:** eligible RPS, 4xx/5xx, p50/p95/p99 latency, timeout, retry and request completion.

**Inference UX:** TTFT p50/p95/p99, TPOT, end-to-end duration, stream completion, output tokens/sec, queue depth and running requests.

**Capacity:** GPU allocation, utilization/memory, KV cache, Pod readiness, scheduler wait, autoscaling and model load.

**Telemetry health:** scrape UP, collector accepted/dropped, missing metrics, alert evaluation and notification delivery.

Every dashboard should have environment, cluster, namespace, service/model and deployment version filters; clear units, data freshness, ownership and runbook links. Avoid user-ID, prompt text and request-ID labels. Show no-data distinctly from zero.

## 7. Alertmanager and escalation

Use labels for `team`, `service`, `severity`, `environment` and safe model alias. Route to an on-call schedule, group related alerts, suppress known downstream noise, and use silences with owner, reason and expiry. Do not silence critical security signals to hide reliability issues. Test notification delivery, deduplication, acknowledgments, escalation and incident linkage in staging.

## 8. Error-budget policy and release decisions

Define a documented response ladder: healthy budget → normal releases; rapid burn → investigate and freeze risky changes; exhausted budget → reliability-first work and executive/owner exception process. Policy must be agreed with product/security/SRE stakeholders; budget alone should not block urgent security patches or incident mitigations. Record exception owner and expiry.

## 9. Twelve operational runbooks

1. **Availability SLO breach:** validate denominator and affected users; segment by gateway/model/revision.
2. **TTFT SLO breach:** inspect queue, prefill load, input-token mix and GPU/runtime saturation.
3. **Streaming completion failures:** check gateway timeouts, disconnects, runtime cancellation and client semantics.
4. **Alert never fires:** verify rule loading, evaluation, labels, `for`, Alertmanager routing and silences.
5. **Alert storm:** tune grouping/inhibition, error budgets, traffic floors and dependency correlation.
6. **No-data mistaken for healthy:** check `up`, `absent`, scrape and telemetry coverage alerts.
7. **SLO numerator/denominator mismatch:** inspect labels, retries, 4xx exclusions and metric source.
8. **Low-traffic false pages:** add volume gates and longer windows without hiding real outages.
9. **Error budget suddenly consumed:** verify release timing, instrumentation change, denominator and window.
10. **Grafana dashboard stale:** inspect datasource, query errors, caching, timezone and refresh interval.
11. **Notification routing failure:** check receivers, secrets, webhook auth, schedules and escalation.
12. **SLO reporting disputed:** reconcile gateway/client logs, metric cardinality, sampling and approved definition.

## 10. Production acceptance

Publish an SLO specification (owner, target, window, source, numerator, denominator, exclusions, traffic floor, data-quality policy), dashboard, alert rules, routing, runbooks, budget policy and rollback. Test synthetic failures in staging, review user impact, validate alerts and record sign-off. Do not claim SLO compliance from unverified metrics.

Complete the [hands-on lab](../labs/06-dashboards-slis-slos-alerting-error-budgets-check.md), containing **32 failure scenarios and 12 acceptance gates**. Formal review and live execution remain pending.

**Next:** Chapter 11.7 — Enterprise Observability and Incident Detection Capstone.
