# Chapter 089 — Executable Model Quality and Release Regression Incident Lab

**Status:** Published; offline tests and live staging evaluation not yet executed.

## Setup
Python 3.11+ with pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Regression evaluator
Save as `quality_incident.py`:
```python
from dataclasses import dataclass
from decimal import Decimal
import json

@dataclass(frozen=True)
class Cohort:
    name: str
    samples: int
    successful: int
    grounded: int
    safety_violations: int
    p95_latency_ms: int
    revision: str

def validate(c):
    if not isinstance(c, Cohort):
        raise TypeError("Cohort required")
    values = (c.samples, c.successful, c.grounded, c.safety_violations, c.p95_latency_ms)
    if any(type(v) is not int for v in values):
        raise ValueError("integer counts required")
    if c.samples <= 0 or c.p95_latency_ms <= 0:
        raise ValueError("positive sample count and latency required")
    if any(v < 0 or v > c.samples for v in (c.successful, c.grounded, c.safety_violations)):
        raise ValueError("counts outside cohort")
    if not c.name or not c.revision:
        raise ValueError("name and revision required")

def metrics(c):
    validate(c)
    return {
        "task_success": Decimal(c.successful) / Decimal(c.samples),
        "grounded_rate": Decimal(c.grounded) / Decimal(c.samples),
        "safety_violations": c.safety_violations,
    }

def evaluate(baseline, canary, min_samples=100,
             max_success_drop=Decimal("0.03"),
             max_grounded_drop=Decimal("0.04"),
             max_latency_increase=Decimal("0.10")):
    validate(baseline)
    validate(canary)
    if type(min_samples) is not int or min_samples <= 0:
        raise ValueError("min_samples must be positive")
    b, c = metrics(baseline), metrics(canary)
    gates = {
        "sample_size": baseline.samples >= min_samples and canary.samples >= min_samples,
        "task_success": c["task_success"] >= b["task_success"] - max_success_drop,
        "grounded_rate": c["grounded_rate"] >= b["grounded_rate"] - max_grounded_drop,
        "p95_latency": Decimal(canary.p95_latency_ms) <= Decimal(baseline.p95_latency_ms) * (1 + max_latency_increase),
        "zero_safety_violations": canary.safety_violations == 0,
    }
    return {
        "promote": all(gates.values()),
        "rollback_or_hold": not all(gates.values()),
        "failed_gates": sorted(k for k, passed in gates.items() if not passed),
        "baseline": {k: str(v) for k, v in b.items()},
        "canary": {k: str(v) for k, v in c.items()},
    }

def sample():
    baseline = Cohort("baseline", 1000, 950, 920, 0, 400, "model-v1")
    canary = Cohort("canary", 1000, 940, 910, 0, 420, "model-v2")
    return baseline, canary

if __name__ == "__main__":
    print(json.dumps(evaluate(*sample()), indent=2))
```

**Trust limitation:** Counts and safety results are supplied by the operator. The evaluator does not validate dataset independence, label correctness, evaluator drift, subgroup fairness or statistical significance. Production promotion requires those checks plus policy approvals.

## 2. Pytest suite
Save as `test_quality_incident.py`:
```python
from dataclasses import replace
from decimal import Decimal
import pytest
from quality_incident import Cohort, evaluate, metrics, sample

def test_healthy_canary():
    assert evaluate(*sample())["promote"]

def test_task_regression():
    b, c = sample()
    assert "task_success" in evaluate(b, replace(c, successful=850))["failed_gates"]

def test_grounding_regression():
    b, c = sample()
    assert "grounded_rate" in evaluate(b, replace(c, grounded=800))["failed_gates"]

def test_latency_regression():
    b, c = sample()
    assert "p95_latency" in evaluate(b, replace(c, p95_latency_ms=500))["failed_gates"]

def test_safety_is_hard_gate():
    b, c = sample()
    assert not evaluate(b, replace(c, safety_violations=1))["promote"]

def test_low_sample_hold():
    b, c = sample()
    assert "sample_size" in evaluate(b, replace(c, samples=50, successful=47, grounded=45))["failed_gates"]

def test_exact_rate():
    b, _ = sample()
    assert metrics(b)["task_success"] == Decimal("0.95")

def test_recovery_after_rollback():
    b, c = sample()
    assert not evaluate(b, replace(c, successful=700))["promote"]
    assert evaluate(b, replace(c, successful=950, grounded=920))["promote"]

@pytest.mark.parametrize("change", [
    {"samples": 0}, {"successful": -1}, {"grounded": 1001},
    {"safety_violations": -1}, {"p95_latency_ms": 0},
])
def test_invalid_cohort(change):
    b, c = sample()
    with pytest.raises(ValueError):
        evaluate(b, replace(c, **change))

def test_invalid_type():
    with pytest.raises(TypeError):
        metrics({})

def test_invalid_min_samples():
    with pytest.raises(ValueError):
        evaluate(*sample(), min_samples=0)
```

Run:
```bash
python quality_incident.py
python -m pytest -q
```
The suite defines **15 test cases** (10 standalone and 5 parameterized). Execution is pending.

## 3. Failure injection
1. Degrade task success by 10 percentage points and verify hold.
2. Degrade grounding independently of HTTP success.
3. Inject a safety violation and confirm critical rollback gate.
4. Increase p95 latency above allowed regression.
5. Use insufficient sample size and verify hold.
6. Restore baseline scores and confirm candidate becomes eligible.
7. Run equivalent evaluations per tenant/language/task slice in staging.
8. Record release SHA, prompt revision, retrieval index, model digest, evaluator version and incident timeline.

## 4. 100-point assessment
| Dimension | Points |
|---|---:|
| Baseline and metrics | 20 |
| Quality and safety gates | 20 |
| Release correlation | 20 |
| Recovery validation | 20 |
| Tests, evidence and incident communications | 20 |
| **Total** | **100** |

Educational target >=90/100 with no critical safety/privacy bypass. Real model evaluation, statistical validation and independent review remain pending.
