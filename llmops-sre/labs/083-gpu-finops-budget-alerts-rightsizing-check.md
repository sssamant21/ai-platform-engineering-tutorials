# Chapter 083 — Executable GPU FinOps, Budget Alerts and Rightsizing Lab

**Status:** Published; execution, live GPU tests and FinOps review pending. Synthetic prices only.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Offline cost and rightsizing evaluator
Save as `gpu_finops.py`:
```python
from dataclasses import dataclass
from decimal import Decimal
import json

@dataclass(frozen=True)
class Workload:
    gpu_count: int
    billed_hours: Decimal
    usd_per_gpu_hour: Decimal
    completed_tokens: int
    good_requests: int
    p95_latency_ms: int
    goodput_rps: Decimal
    n_minus_one_pass: bool

def evaluate_cost(w):
    if type(w.gpu_count) is not int or w.gpu_count <= 0:
        raise ValueError("gpu_count must be positive")
    if w.billed_hours < 0 or w.usd_per_gpu_hour < 0:
        raise ValueError("negative cost input")
    if type(w.completed_tokens) is not int or w.completed_tokens <= 0:
        raise ValueError("completed_tokens must be positive")
    if type(w.good_requests) is not int or w.good_requests <= 0:
        raise ValueError("good_requests must be positive")
    total = Decimal(w.gpu_count) * w.billed_hours * w.usd_per_gpu_hour
    return {
        "total_usd": total,
        "usd_per_million_tokens": total * Decimal(1_000_000) / Decimal(w.completed_tokens),
        "usd_per_good_request": total / Decimal(w.good_requests),
    }

def evaluate_rightsizing(baseline, candidate):
    base = evaluate_cost(baseline)
    cand = evaluate_cost(candidate)
    if type(baseline.p95_latency_ms) is not int or baseline.p95_latency_ms <= 0:
        raise ValueError("invalid baseline latency")
    if type(candidate.p95_latency_ms) is not int or candidate.p95_latency_ms <= 0:
        raise ValueError("invalid candidate latency")
    if baseline.goodput_rps <= 0 or candidate.goodput_rps < 0:
        raise ValueError("invalid goodput")
    gates = {
        "lower_cost_per_good_request": cand["usd_per_good_request"] < base["usd_per_good_request"],
        "p95_regression_under_10pct": Decimal(candidate.p95_latency_ms) <= Decimal(baseline.p95_latency_ms) * Decimal("1.10"),
        "goodput_regression_under_5pct": candidate.goodput_rps >= baseline.goodput_rps * Decimal("0.95"),
        "n_minus_one_pass": candidate.n_minus_one_pass is True,
    }
    return {"approved": all(gates.values()), "gates": gates}

def budget_alert(actual_usd, projected_usd, monthly_budget_usd):
    if monthly_budget_usd <= 0 or actual_usd < 0 or projected_usd < 0:
        raise ValueError("invalid budget")
    ratio = max(actual_usd, projected_usd) / monthly_budget_usd
    if ratio >= Decimal("1.00"):
        return "critical"
    if ratio >= Decimal("0.95"):
        return "high"
    if ratio >= Decimal("0.85"):
        return "warning"
    if ratio >= Decimal("0.70"):
        return "notice"
    return "normal"

def sample():
    base = Workload(4, Decimal("24"), Decimal("2"), 10_000_000, 10_000, 200, Decimal("20"), True)
    candidate = Workload(3, Decimal("24"), Decimal("2"), 10_000_000, 10_000, 210, Decimal("19.5"), True)
    return base, candidate

if __name__ == "__main__":
    base, candidate = sample()
    result = evaluate_rightsizing(base, candidate)
    print(json.dumps(result, indent=2))
    print("baseline_cost", evaluate_cost(base)["total_usd"])
    print("budget_alert", budget_alert(Decimal("700"), Decimal("860"), Decimal("1000")))
```

**Limitations:** No real cloud pricing, idle allocation, discounts, GPU profiling or quality evaluation is performed. Cost-per-good-request assumes the supplied request count is trusted. Production rightsizing requires real SLO benchmarks, N-1 validation, billing reconciliation and safety evaluation.

## 3. Automated tests
Save as `test_gpu_finops.py`:
```python
from dataclasses import replace
from decimal import Decimal
import pytest
from gpu_finops import evaluate_cost, evaluate_rightsizing, budget_alert, sample

def test_gpu_cost():
    b, _ = sample()
    assert evaluate_cost(b)["total_usd"] == Decimal("192")

def test_cost_per_million_tokens():
    b, _ = sample()
    assert evaluate_cost(b)["usd_per_million_tokens"] == Decimal("19.2")

def test_good_rightsizing():
    b, c = sample()
    assert evaluate_rightsizing(b, c)["approved"]

def test_latency_regression():
    b, c = sample()
    assert not evaluate_rightsizing(b, replace(c, p95_latency_ms=240))["approved"]

def test_goodput_regression():
    b, c = sample()
    assert not evaluate_rightsizing(b, replace(c, goodput_rps=Decimal("18")))["approved"]

def test_n_minus_one_failure():
    b, c = sample()
    assert not evaluate_rightsizing(b, replace(c, n_minus_one_pass=False))["approved"]

def test_cost_increase():
    b, c = sample()
    assert not evaluate_rightsizing(b, replace(c, gpu_count=5))["approved"]

def test_zero_tokens_rejected():
    b, _ = sample()
    with pytest.raises(ValueError):
        evaluate_cost(replace(b, completed_tokens=0))

def test_negative_price_rejected():
    b, _ = sample()
    with pytest.raises(ValueError):
        evaluate_cost(replace(b, usd_per_gpu_hour=Decimal("-1")))

@pytest.mark.parametrize("projected,expected", [
    ("600", "normal"),
    ("700", "notice"),
    ("850", "warning"),
    ("950", "high"),
    ("1000", "critical"),
])
def test_budget_alert_levels(projected, expected):
    assert budget_alert(Decimal("100"), Decimal(projected), Decimal("1000")) == expected

def test_invalid_budget():
    with pytest.raises(ValueError):
        budget_alert(Decimal("10"), Decimal("20"), Decimal("0"))
```

Run:
```bash
python gpu_finops.py
python -m pytest -q
```
Expected: **15 pytest cases** (10 standalone and 5 parameterized). Execution has not been performed.

## 4. SRE/FinOps failure drills
1. Reduce GPU replicas and validate cost savings against p95 and goodput guardrails.
2. Trigger latency and throughput regressions independently.
3. Fail N-1 availability and confirm rollout rejection.
4. Simulate projected spend crossing 70%, 85%, 95% and 100% budget.
5. Reject invalid pricing, zero token volume and zero budget.
6. Explain why GPU utilization alone is not a rightsizing approval criterion.
7. Plan a live staging benchmark with realistic prompt lengths, output lengths, concurrency and GPU memory pressure.
8. Reconcile GPU-hour ledger with invoice and record shared idle/HA allocation policy.

## 5. 100-point rubric
| Area | Points |
|---|---:|
| Cost inventory and allocation | 20 |
| Unit economics and trusted metering | 20 |
| Budget forecasts and alerts | 20 |
| SLO-safe rightsizing and N-1 | 20 |
| Tests, runbooks and audit | 20 |
| **Total** | **100** |

Educational acceptance >=90/100 with no critical SLO regression. Production acceptance requires real GPU measurements, cost validation and review.
