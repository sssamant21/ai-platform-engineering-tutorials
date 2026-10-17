# Chapter 062 — Prometheus Recording Rules and Alert Design

**Part 08 — LLM Observability, Metrics and Tracing**  
**Publication:** Tutorial and companion lab published. **Reviews and live execution:** pending.

## Learning objectives
- Distinguish raw metrics, recording rules, alerting rules, and SLO burn-rate alerts.
- Design robust recording rules for LLM inference availability, throughput, latency and saturation.
- Understand rate windows, counter resets, histograms, missing data, label cardinality and aggregation.
- Test PromQL expressions and rule syntax with `promtool` in a reproducible synthetic lab.
- Operate safe alert routing, inhibition, ownership and incident runbooks.

## 1. Architecture
```text
gateway / inference / GPU exporters
             |
     Prometheus scrape
             |
  recording rules (precompute)
             |
 alert rules (for / severity)
             |
        Alertmanager
             |
  on-call routing / dedup / silence
             |
 dashboards + traces + logs + runbooks
```
Recording rules compute frequently used expressions into named time series, reducing query cost and promoting consistent definitions. Alert rules evaluate conditions; `for` guards against transient spikes but increases detection delay. Neither replaces correct SLI definitions, burn-rate policies or paging ownership.

## 2. Metrics and labels
Use a **documented canonical metric contract**. The lab intentionally uses **synthetic illustrative names**, not guaranteed vLLM/Triton exporter names:
- `llm_requests_total{service,environment,outcome}` — counter.
- `llm_request_duration_seconds_bucket{service,environment,le}` — classic histogram buckets.
- `llm_inflight_requests{service,environment}` — gauge.
- `llm_queue_depth{service,environment}` — gauge.
- `up{job}` — Prometheus target health.

Adapt recording rules to your actual deployed exporter metric names and histogram type after verifying `/metrics`, HELP/TYPE metadata, labels and versions. Never silently substitute names from another exporter. Use controlled `service`, `environment`, `outcome` dimensions; do not label with prompt, request ID, user ID, tenant ID or raw model name.

## 3. PromQL patterns

### Request throughput
```promql
sum by (service, environment) (rate(llm_requests_total[5m]))
```

### Error ratio (explicit outcome contract)
```promql
sum by (service, environment) (rate(llm_requests_total{outcome="error"}[5m]))
/
clamp_min(sum by (service, environment) (rate(llm_requests_total[5m])), 1e-9)
```
A near-zero denominator can create misleading ratios during idle traffic; gate alerts on a minimum request rate or count. This example counts `outcome="error"` only: define precisely whether cancellations, policy rejects, 429s and timeouts count against the user-journey SLI.

### Classic histogram p95 latency
```promql
histogram_quantile(
  0.95,
  sum by (service, environment, le) (
    rate(llm_request_duration_seconds_bucket[5m])
  )
)
```
Keep `le` in the aggregation for classic histograms. Histogram quantiles are estimates affected by bucket boundaries and low volume. Do not average p95 values across instances. Native histogram queries require different aggregation and version compatibility checks.

### Queue saturation
```promql
max by (service, environment) (llm_queue_depth)
```
Queue depth alone is not an SLO: correlate with arrival rate, time-in-queue, TTFT, rejection rate and replica/GPU utilization.

## 4. Recording rule design
Recommended naming: `service:metric:operation_window` (e.g. `llm:requests_per_second:rate5m`). Ensure that the recorded series' labels are stable and match downstream dashboard/alert assumptions. Evaluate `rate()` before summing counters to handle counter resets. Set evaluation intervals appropriate to scrape intervals and detection requirements. Avoid cascading recording rules that depend on execution order across groups.

## 5. Alerting policy
| Alert | Signal | Severity | Notes |
|---|---|---|---|
| LLMHighErrorRatio | elevated 5m error ratio with traffic gate | warning | initial lab threshold only |
| LLMHighP95Latency | 5m p95 latency > approved budget | warning | verify low-volume behavior |
| LLMQueueBacklog | queue depth above capacity threshold | warning | corroborate TTFT and replicas |
| LLMInferenceTargetDown | target unavailable | critical or warning per redundancy | `up` missing series needs separate treatment |

Do not page on every transient p95 or individual pod failure if redundancy protects users. Multi-window, multi-burn-rate SLO alerts are covered in Chapter 063. Alertmanager grouping, deduplication, routing, inhibition and silences need change control; silencing is not remediation.

## 6. Lab
See [Chapter 062 companion lab](../labs/062-prometheus-recording-rules-alert-design-check.md) for:
- self-contained rule YAML,
- `promtool check rules`,
- `promtool test rules` using synthetic time series,
- negative test and failure drills,
- expected results and cleanup.

## 7. Failure scenarios and troubleshooting
1. **No alert during incident:** verify scrape, absent series, label matchers, evaluation interval, `for`, Alertmanager route and receiver.
2. **Alert flaps:** inspect sample gaps, thresholds, traffic volume, `for` and workload patterns.
3. **Unexpectedly high error ratio:** verify numerator and denominator have identical aggregation labels and outcome taxonomy.
4. **Latency looks healthy but users complain:** investigate histogram buckets, streaming semantics, TTFT/TPOT and gateway buffering.
5. **Prometheus OOM or query slowness:** inspect active series, recording-rule cardinality, expensive regex selectors and high-frequency evaluation.
6. **Silent missing target:** `up == 0` only matches existing target series; supplement with expected-target inventory and `absent()` checks scoped to stable jobs.

## 8. Production runbooks

### RB-062-A — High error ratio
**[SAFE-READ]** confirm SLI definition and minimum traffic; segment by controlled service/environment, inspect recent deploys, 429/5xx/timeouts and correlated traces. **[CHANGE-CONTROL]** mitigate with approved rollback, capacity or routing changes. Record incident timeline and error budget impact.

### RB-062-B — Latency regression
Compare 5m/30m latency, TTFT, TPOT, queue depth, GPU utilization and upstream timeouts. Determine whether bottleneck is gateway, admission, prefill, decode or network using actual telemetry. Escalate with trace IDs, not prompt content.

### RB-062-C — Rule evaluation failure
Inspect Prometheus `/rules`, evaluation errors and config reload status. Run `promtool check rules` before applying corrected YAML; validate rule output after rollout. Never restart a production monitoring cluster without approval.

### RB-062-D — Alert storm
Confirm root cause and shared labels, adjust Alertmanager grouping/inhibition under change control, preserve critical pages and log temporary silences with owner and expiry. Fix noisy instrumentation rather than permanently hiding alerts.

## 9. Security, FinOps and acceptance
- **[SAFE-READ]** metrics/rules inspection; **[LAB-MUTATE]** local tests; **[CHANGE-CONTROL]** production rule/route modifications.
- Protect metrics endpoints and Alertmanager; avoid tenant identifiers and sensitive payloads in labels/annotations.
- Estimate added recorded-series cardinality and retention impact before deployment.
- [ ] Syntax validated with `promtool`.
- [ ] Synthetic unit tests pass, including expected alert timing.
- [ ] Missing/idle traffic and counter resets considered.
- [ ] Dashboards, ownership, runbooks and alert routes reviewed.
- [ ] Security and technical reviews and live execution tracked independently.

## 10. References
- [Prometheus recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
- [Prometheus alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Prometheus promtool unit testing](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/)
- [Prometheus alerting best practices](https://prometheus.io/docs/practices/alerting/)

**Next:** Chapter 063 — SLO Burn Rates and Error Budget Policy.
