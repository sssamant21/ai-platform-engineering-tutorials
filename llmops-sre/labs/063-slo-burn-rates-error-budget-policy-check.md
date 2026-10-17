# Chapter 063 — Executable SLO Burn-Rate and Error-Budget Lab

**Status:** Published, not executed. **Environment:** local synthetic counters; no production access.

## 1. Prerequisites
Use a reviewed Prometheus release containing `promtool`, plus Python 3.11+.
```bash
promtool --version
python3 --version
```

## 2. Rule file
Save as `slo-rules.yml`:

```yaml
groups:
  - name: llm-slo-burn
    interval: 1m
    rules:
      - record: llm:sli_error_ratio:rate5m
        expr: sum by (service, environment) (rate(llm_sli_bad_requests_total[5m])) / clamp_min(sum by (service, environment) (rate(llm_sli_requests_total[5m])), 1e-9)
      - record: llm:sli_error_ratio:rate1h
        expr: sum by (service, environment) (rate(llm_sli_bad_requests_total[1h])) / clamp_min(sum by (service, environment) (rate(llm_sli_requests_total[1h])), 1e-9)
      - record: llm:sli_burn_rate:rate1h
        expr: llm:sli_error_ratio:rate1h / 0.001
      - alert: LLMAvailabilityFastBurn
        expr: |
          (
            sum by (service, environment) (rate(llm_sli_bad_requests_total[1h]))
            / clamp_min(sum by (service, environment) (rate(llm_sli_requests_total[1h])), 1e-9)
          ) > (14.4 * 0.001)
          and on(service, environment)
          (
            sum by (service, environment) (rate(llm_sli_bad_requests_total[5m]))
            / clamp_min(sum by (service, environment) (rate(llm_sli_requests_total[5m])), 1e-9)
          ) > (14.4 * 0.001)
          and on(service, environment)
          sum by (service, environment) (rate(llm_sli_requests_total[5m])) > 0.1
        for: 2m
        labels:
          severity: page
          team: inference-sre
        annotations:
          summary: "Fast availability SLO burn for {{ $labels.service }}"
          runbook: "Chapter 063 RB-063-A"
      - alert: LLMAvailabilitySlowBurn
        expr: |
          (
            sum by (service, environment) (rate(llm_sli_bad_requests_total[6h]))
            / clamp_min(sum by (service, environment) (rate(llm_sli_requests_total[6h])), 1e-9)
          ) > (6 * 0.001)
          and on(service, environment)
          (
            sum by (service, environment) (rate(llm_sli_bad_requests_total[30m]))
            / clamp_min(sum by (service, environment) (rate(llm_sli_requests_total[30m])), 1e-9)
          ) > (6 * 0.001)
          and on(service, environment)
          sum by (service, environment) (rate(llm_sli_requests_total[30m])) > 0.1
        for: 2m
        labels:
          severity: page
          team: inference-sre
        annotations:
          summary: "Slow availability SLO burn for {{ $labels.service }}"
          runbook: "Chapter 063 RB-063-A"
```

This compact lab tests fast and slow paging pairs. Extend it with 1d/2h and 3d/6h ticket pairs before production adoption. All metric names and thresholds are **synthetic** until mapped to your service.

## 3. promtool unit tests
Save as `slo-rules.test.yml`:

```yaml
rule_files:
  - slo-rules.yml
evaluation_interval: 1m
tests:
  - interval: 1m
    input_series:
      - series: 'llm_sli_requests_total{service="gateway",environment="staging"}'
        values: '0+600x75'
      - series: 'llm_sli_bad_requests_total{service="gateway",environment="staging"}'
        values: '0+12x75'
    promql_expr_test:
      - expr: 'llm:sli_burn_rate:rate1h'
        eval_time: 70m
        exp_samples:
          - labels: '{service="gateway",environment="staging"}'
            value: 20
    alert_rule_test:
      - eval_time: 70m
        alertname: LLMAvailabilityFastBurn
        exp_alerts:
          - exp_labels:
              service: gateway
              environment: staging
              severity: page
              team: inference-sre
            exp_annotations:
              summary: "Fast availability SLO burn for gateway"
              runbook: "Chapter 063 RB-063-A"
      - eval_time: 70m
        alertname: LLMAvailabilitySlowBurn
        exp_alerts:
          - exp_labels:
              service: gateway
              environment: staging
              severity: page
              team: inference-sre
            exp_annotations:
              summary: "Slow availability SLO burn for gateway"
              runbook: "Chapter 063 RB-063-A"
  - interval: 1m
    input_series:
      - series: 'llm_sli_requests_total{service="idle",environment="staging"}'
        values: '0+0x75'
      - series: 'llm_sli_bad_requests_total{service="idle",environment="staging"}'
        values: '0+0x75'
    alert_rule_test:
      - eval_time: 70m
        alertname: LLMAvailabilityFastBurn
        exp_alerts: []
      - eval_time: 70m
        alertname: LLMAvailabilitySlowBurn
        exp_alerts: []
```

Run:
```bash
promtool check rules slo-rules.yml
promtool test rules slo-rules.test.yml
```
Expected: rules validate, burn rate equals 20 at 70m, both alerts fire for sustained high error ratio and neither fires for idle traffic.

## 4. Independent error-budget calculator
Save as `budget.py`:

```python
from decimal import Decimal

def budget(target, eligible, bad):
    target = Decimal(str(target))
    if not Decimal("0") < target < Decimal("1"):
        raise ValueError("target must be between 0 and 1")
    if isinstance(eligible, bool) or isinstance(bad, bool):
        raise ValueError("counts must be integers")
    if not isinstance(eligible, int) or not isinstance(bad, int):
        raise ValueError("counts must be integers")
    if not (0 < eligible and 0 <= bad <= eligible):
        raise ValueError("invalid counts")
    allowance = Decimal(eligible) * (1 - target)
    consumed = Decimal(bad) / allowance
    return {
        "allowed_bad": allowance,
        "consumed_fraction": consumed,
        "remaining_fraction": 1 - consumed,
        "burn_rate": (Decimal(bad) / Decimal(eligible)) / (1 - target),
    }

if __name__ == "__main__":
    print(budget("0.999", 1_000_000, 1_500))
```

Save as `test_budget.py`:

```python
from decimal import Decimal
import pytest
from budget import budget

def test_budget_exhausted():
    result = budget("0.999", 1_000_000, 1_500)
    assert result["allowed_bad"] == Decimal("1000.000")
    assert result["consumed_fraction"] == Decimal("1.5")
    assert result["remaining_fraction"] == Decimal("-0.5")
    assert result["burn_rate"] == Decimal("1.5")

@pytest.mark.parametrize("target,eligible,bad", [
    ("1", 100, 0), ("0", 100, 0),
    ("0.999", 0, 0), ("0.999", 100, 101),
    ("0.999", 100, -1), ("0.999", True, 0)
])
def test_invalid_inputs(target, eligible, bad):
    with pytest.raises(ValueError):
        budget(target, eligible, bad)
```

Run `python -m pip install pytest && python -m pytest -q`. Expected: 7 passing tests.

## 5. Failure drills
1. Raise alert burn threshold to 100 in a disposable copy: expected fast-burn alert test must fail.
2. Remove the short-window clause: demonstrate how a stale long-window incident could continue paging.
3. Remove the traffic gate: inspect why low-volume ratios may be noisy.
4. Remove one metric series: observe missing data and ensure it is not treated as zero errors.
5. Simulate a counter reset in synthetic data; confirm `rate()` handles reset before aggregation.
6. Recalculate the budget example manually and compare with Python output.

## 6. Production acceptance and cleanup
- [ ] `promtool check rules` passes.
- [ ] `promtool test rules` passes.
- [ ] Seven Python tests pass.
- [ ] Missing-data and traffic-gate behavior documented.
- [ ] Service-specific SLO target and eligible/bad taxonomy approved.
- [ ] Alert routing, ownership, retention and change policy reviewed.
- [ ] Technical/security reviews and live execution evidence recorded separately.

Remove disposable lab files only after saving sanitized evidence. **Do not apply lab alert thresholds directly to production.**
