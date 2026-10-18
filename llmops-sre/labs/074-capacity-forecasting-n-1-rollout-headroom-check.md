# Chapter 074 — Executable Capacity Forecasting, N-1 and Rollout Headroom Lab

**Status:** Published; execution pending. **Safety:** offline Python planning calculator only; no cloud calls or production changes.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
Windows PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Capacity calculator
Save as `capacity_lab.py`:

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Scenario:
    baseline_peak_rps: float
    growth_factor: float
    seasonal_factor: float
    safe_rps_per_replica: float
    ready_replicas: int
    failure_domain_replicas: int = 1
    rollout_max_unavailable: int = 0

def validate(s):
    for name in ("baseline_peak_rps", "growth_factor", "seasonal_factor", "safe_rps_per_replica"):
        value = getattr(s, name)
        if not isinstance(value, (int, float)) or not math.isfinite(value) or value <= 0:
            raise ValueError(f"invalid {name}")
    for name in ("ready_replicas", "failure_domain_replicas", "rollout_max_unavailable"):
        value = getattr(s, name)
        if type(value) is not int or value < 0:
            raise ValueError(f"invalid {name}")
    if s.failure_domain_replicas > s.ready_replicas:
        raise ValueError("failure domain larger than ready fleet")
    if s.rollout_max_unavailable > s.ready_replicas:
        raise ValueError("unavailable larger than ready fleet")

def forecast(s):
    validate(s)
    peak = s.baseline_peak_rps * s.growth_factor * s.seasonal_factor
    serving = math.ceil(peak / s.safe_rps_per_replica)
    resilient = serving + s.failure_domain_replicas
    surviving = s.ready_replicas - s.failure_domain_replicas
    during_rollout = s.ready_replicas - s.rollout_max_unavailable
    return {
        "forecast_peak_rps": peak,
        "serving_replicas": serving,
        "resilient_ready_replicas": resilient,
        "additional_ready_replicas": max(0, resilient - s.ready_replicas),
        "n_minus_domain_capacity_rps": surviving * s.safe_rps_per_replica,
        "rollout_available_capacity_rps": during_rollout * s.safe_rps_per_replica,
        "n_minus_domain_safe": surviving * s.safe_rps_per_replica >= peak,
        "rollout_safe": during_rollout * s.safe_rps_per_replica >= peak,
    }

def required_surge_slots(max_surge, free_gpu_slots):
    if any(type(x) is not int or x < 0 for x in (max_surge, free_gpu_slots)):
        raise ValueError("invalid GPU slots")
    return max(0, max_surge - free_gpu_slots)

if __name__ == "__main__":
    scenario = Scenario(60, 1.25, 1.2, 15, 7, 1, 0)
    print(json.dumps(forecast(scenario), indent=2))
    print("additional_gpu_slots_for_surge", required_surge_slots(2, 1))
```

**Limitations:** homogeneous replicas, fixed safe RPS, multiplicative forecast, static failure-domain size. No token-level demand, actual GPU availability, autoscaling time, cross-zone routing or probabilistic confidence intervals. Real SLO-safe capacity must be measured in staging.

## 3. Automated tests
Save as `test_capacity_lab.py`:

```python
import pytest
from capacity_lab import Scenario, forecast, required_surge_slots

S = Scenario(60, 1.25, 1.2, 15, 7, 1, 0)

def test_peak_forecast():
    assert forecast(S)["forecast_peak_rps"] == 90

def test_serving_replicas():
    assert forecast(S)["serving_replicas"] == 6

def test_resilient_replicas():
    assert forecast(S)["resilient_ready_replicas"] == 7

def test_n_minus_one_safe():
    assert forecast(S)["n_minus_domain_safe"]

def test_capacity_deficit():
    s = Scenario(60, 1.25, 1.2, 15, 5, 1, 0)
    assert forecast(s)["additional_ready_replicas"] == 2
    assert not forecast(s)["n_minus_domain_safe"]

def test_rollout_unavailable_degrades():
    s = Scenario(60, 1.25, 1.2, 15, 7, 1, 2)
    assert not forecast(s)["rollout_safe"]

def test_correlated_failure_domain():
    s = Scenario(60, 1.25, 1.2, 15, 7, 2, 0)
    assert not forecast(s)["n_minus_domain_safe"]

def test_surge_slots():
    assert required_surge_slots(3, 1) == 2

@pytest.mark.parametrize("s", [
    Scenario(0, 1, 1, 15, 7),
    Scenario(60, -1, 1, 15, 7),
    Scenario(60, 1, 1, 0, 7),
    Scenario(60, 1, 1, 15, 2, 3),
    Scenario(60, 1, 1, 15, 2, 1, 3),
])
def test_invalid_scenario(s):
    with pytest.raises(ValueError):
        forecast(s)

def test_invalid_surge():
    with pytest.raises(ValueError):
        required_surge_slots(-1, 1)
```

Run:
```bash
python capacity_lab.py
python -m pytest -q
```
Expected: **14 pytest cases** (9 standalone plus 5 parameterized); actual execution pending.

## 4. Experiments
1. Increase growth factor from 1.25 to 1.5; compare resilient ready replicas.
2. Decrease safe RPS/replica from 15 to 10 to represent a longer-context workload.
3. Model loss of two co-located replicas rather than one.
4. Set `rollout_max_unavailable=2`; determine rollout SLO risk.
5. Require two surge GPU slots with one free; identify provisioning gap.
6. Compare optimistic/expected/adverse seasonal factors and document uncertainty.
7. Evaluate warmup lead time of 10 minutes against a 30-second demand spike; explain why forecast alone cannot solve it.
8. Price additional ready GPU replicas using your organization's approved rate sheet (do not embed confidential rates in repo).

## 5. Evidence and 100-point acceptance
Record input assumptions, calculator output, test run, safe capacity provenance, failure-domain size, rollout settings, GPU quota evidence and change-control plan.

| Dimension | Points |
|---|---:|
| Demand forecasting and uncertainty | 20 |
| SLO-safe capacity and N-1 | 20 |
| Rollout and GPU surge headroom | 20 |
| Automated tests and failure drills | 20 |
| FinOps, security and runbook readiness | 20 |
| **Total** | **100** |

Educational acceptance target >=90/100. Production acceptance requires real GPU benchmark data, staging N-1 drills and owner/security review.
