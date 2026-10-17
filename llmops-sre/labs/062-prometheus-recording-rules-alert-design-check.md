# Chapter 062 — Executable Prometheus Recording Rules and Alert Design Lab

**Status:** Published, not executed. **Environment:** isolated local test; no production Prometheus access.

## 1. Prerequisites
Prometheus `promtool` installed from a reviewed official Prometheus release. Check:
```bash
promtool --version
```
No live Prometheus server is needed for the rule unit tests.

## 2. Define the rule group
Create `llm-rules.yml`:

```yaml
groups:
  - name: llm-inference-recording
    interval: 1m
    rules:
      - record: llm:requests_per_second:rate5m
        expr: sum by (service, environment) (rate(llm_requests_total[5m]))
      - record: llm:errors_per_second:rate5m
        expr: sum by (service, environment) (rate(llm_requests_total{outcome="error"}[5m]))
      - record: llm:error_ratio:rate5m
        expr: llm:errors_per_second:rate5m / clamp_min(llm:requests_per_second:rate5m, 1e-9)
      - record: llm:p95_request_duration_seconds:rate5m
        expr: histogram_quantile(0.95, sum by (service, environment, le) (rate(llm_request_duration_seconds_bucket[5m])))
      - record: llm:queue_depth:max
        expr: max by (service, environment) (llm_queue_depth)
  - name: llm-inference-alerts
    interval: 1m
    rules:
      - alert: LLMHighErrorRatio
        expr: |
          (
            sum by (service, environment) (rate(llm_requests_total{outcome="error"}[5m]))
            /
            clamp_min(sum by (service, environment) (rate(llm_requests_total[5m])), 1e-9)
          ) > 0.05
          and on(service, environment)
          sum by (service, environment) (rate(llm_requests_total[5m])) > 0.1
        for: 2m
        labels:
          severity: warning
          team: inference-sre
        annotations:
          summary: "Elevated inference error ratio for {{ $labels.service }}"
          runbook: "Chapter 062 RB-062-A"
      - alert: LLMQueueBacklog
        expr: max by (service, environment) (llm_queue_depth) > 20
        for: 3m
        labels:
          severity: warning
          team: inference-sre
        annotations:
          summary: "Inference queue backlog for {{ $labels.service }}"
          runbook: "Chapter 062 RB-062-B"
```

The `llm_*` metrics are **synthetic contract names**. Production adoption requires exporter mapping, approval and threshold calibration. The alert expressions use raw metrics to avoid cross-group evaluation-order dependencies.

## 3. Unit-test alerts and recording rules
Create `llm-rules.test.yml`:

```yaml
rule_files:
  - llm-rules.yml
evaluation_interval: 1m
tests:
  - interval: 1m
    input_series:
      - series: 'llm_requests_total{service="gateway",environment="staging",outcome="ok"}'
        values: '0+60x12'
      - series: 'llm_requests_total{service="gateway",environment="staging",outcome="error"}'
        values: '0+12x12'
      - series: 'llm_queue_depth{service="gateway",environment="staging"}'
        values: '25x13'
    promql_expr_test:
      - expr: 'llm:requests_per_second:rate5m'
        eval_time: 10m
        exp_samples:
          - labels: '{service="gateway",environment="staging"}'
            value: 1.2
      - expr: 'llm:error_ratio:rate5m'
        eval_time: 10m
        exp_samples:
          - labels: '{service="gateway",environment="staging"}'
            value: 0.16666666666666666
    alert_rule_test:
      - eval_time: 10m
        alertname: LLMHighErrorRatio
        exp_alerts:
          - exp_labels:
              service: gateway
              environment: staging
              severity: warning
              team: inference-sre
            exp_annotations:
              summary: "Elevated inference error ratio for gateway"
              runbook: "Chapter 062 RB-062-A"
      - eval_time: 10m
        alertname: LLMQueueBacklog
        exp_alerts:
          - exp_labels:
              service: gateway
              environment: staging
              severity: warning
              team: inference-sre
            exp_annotations:
              summary: "Inference queue backlog for gateway"
              runbook: "Chapter 062 RB-062-B"
```

Run:
```bash
promtool check rules llm-rules.yml
promtool test rules llm-rules.test.yml
```
Expected: rule syntax validation succeeds and both unit tests and recorded-expression checks pass. Numeric floating-point tolerances are handled by promtool; pin your promtool version and record it in lab evidence.

## 4. Negative and operational tests
1. **Syntax failure:** in a disposable copy, change `rate(` to `rte(`; `promtool check rules` should fail.
2. **Alert timing:** change `for: 2m` to `for: 20m` in a disposable copy; the 10m alert assertion should fail.
3. **Traffic gate:** add a separate synthetic idle workload and confirm no error-ratio page when request rate is below 0.1/s.
4. **Counter reset:** construct an additional series with a reset and compare `sum(rate(counter[5m]))` against `rate(sum(counter)[5m])` to illustrate correct ordering.
5. **Missing metrics:** remove an input series and verify that absent data is not silently interpreted as zero.
6. **Histogram:** add synthetic classic histogram buckets and assert p95 with `promql_expr_test`; confirm `le` is retained until after `rate()` and aggregation.

## 5. Acceptance and cleanup
- [ ] `promtool check rules` passes.
- [ ] `promtool test rules` passes.
- [ ] Negative mutation tests fail as expected.
- [ ] Traffic gate, counter reset, histogram and missing-data scenarios reviewed.
- [ ] Alert owner, routing, dashboard, SLO threshold and runbook approved.
- [ ] Technical/security reviews and live execution recorded independently.

Delete disposable test files only after preserving sanitized validation output. No production rule reloads or Alertmanager changes are part of this lab.
