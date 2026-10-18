# Chapter 006 — Executable AI Infrastructure SLO and Error-Budget Lab

**Status:** Published. Offline pytest execution, production telemetry integration and independent review pending.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic SLO evaluator
Save as `slo_budget.py`:
```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class SLO:
    target: float
    eligible_events: int
    bad_events: int
    short_eligible: int
    short_bad: int
    long_eligible: int
    long_bad: int
    critical_security_incident: bool
    checkpoint_restore_verified: bool
    telemetry_complete: bool
    owner_approved: bool

def evaluate(s):
    if not isinstance(s, SLO):
        raise TypeError("SLO required")
    if type(s.target) not in (int, float) or not math.isfinite(s.target) or not 0 < s.target < 1:
        return {"accepted": False, "reason": "invalid_target"}
    counts = (s.eligible_events, s.bad_events, s.short_eligible,
              s.short_bad, s.long_eligible, s.long_bad)
    if any(type(v) is not int or v < 0 for v in counts):
        return {"accepted": False, "reason": "invalid_counts"}
    if (s.eligible_events == 0 or s.short_eligible == 0 or s.long_eligible == 0
            or s.bad_events > s.eligible_events
            or s.short_bad > s.short_eligible
            or s.long_bad > s.long_eligible):
        return {"accepted": False, "reason": "invalid_counts"}
    budget_fraction = 1 - s.target
    allowed_bad = math.floor(s.eligible_events * budget_fraction + 1e-9)
    burn_short = (s.short_bad / s.short_eligible) / budget_fraction
    burn_long = (s.long_bad / s.long_eligible) / budget_fraction
    gates = {
        "within_budget": s.bad_events <= allowed_bad,
        "fast_burn": not (burn_short >= 14.4 and burn_long >= 14.4),
        "security": s.critical_security_incident is False,
        "checkpoint_restore": s.checkpoint_restore_verified is True,
        "telemetry": s.telemetry_complete is True,
        "owner": s.owner_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {
        "accepted": not failed,
        "failed_gates": failed,
        "allowed_bad_events": allowed_bad,
        "remaining_bad_events": allowed_bad - s.bad_events,
        "budget_consumed_fraction": None if allowed_bad == 0 else round(s.bad_events / allowed_bad, 6),
        "short_burn_rate": round(burn_short, 4),
        "long_burn_rate": round(burn_long, 4),
    }

def example():
    return SLO(0.999, 1_000_000, 700, 100_000, 100, 500_000, 350,
               False, True, True, True)

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
**Interpretation:** The synthetic baseline has 700 bad events against a 1,000-event budget (70% consumed). The illustrative fast-burn rule is intentionally simplified: production multi-window alerts need properly selected time windows, volume thresholds, reset behavior and paging routing. Zero allowed bad events yields an undefined consumed fraction; use a longer evaluation window or a more appropriate statistical approach rather than inventing a percentage.

## 3. Automated pytest suite
Save as `test_slo_budget.py`:
```python
from dataclasses import replace
import pytest
from slo_budget import evaluate, example

def test_baseline():
    r = evaluate(example())
    assert r["accepted"] and r["allowed_bad_events"] == 1000

def test_consumption():
    assert evaluate(example())["budget_consumed_fraction"] == 0.7

def test_budget_exhausted():
    r = evaluate(replace(example(), bad_events=1001))
    assert "within_budget" in r["failed_gates"]

def test_budget_boundary():
    assert evaluate(replace(example(), bad_events=1000))["accepted"]

def test_fast_burn():
    r = evaluate(replace(example(), short_bad=2000, long_bad=10000))
    assert "fast_burn" in r["failed_gates"]

def test_single_window_not_enough():
    r = evaluate(replace(example(), short_bad=2000))
    assert "fast_burn" not in r["failed_gates"]

def test_security_stop():
    assert "security" in evaluate(replace(example(), critical_security_incident=True))["failed_gates"]

def test_missing_checkpoint():
    assert "checkpoint_restore" in evaluate(replace(example(), checkpoint_restore_verified=False))["failed_gates"]

def test_missing_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_complete=False))["failed_gates"]

def test_missing_owner():
    assert "owner" in evaluate(replace(example(), owner_approved=False))["failed_gates"]

def test_bad_target():
    assert evaluate(replace(example(), target=1))["reason"] == "invalid_target"

def test_nan_target():
    assert evaluate(replace(example(), target=float("nan")))["reason"] == "invalid_target"

def test_negative_counts():
    assert evaluate(replace(example(), bad_events=-1))["reason"] == "invalid_counts"

def test_more_bad_than_total():
    assert evaluate(replace(example(), bad_events=1_000_001))["reason"] == "invalid_counts"

def test_zero_events():
    assert evaluate(replace(example(), eligible_events=0))["reason"] == "invalid_counts"

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

@pytest.mark.parametrize("field", ["eligible_events", "bad_events", "short_eligible",
                                   "short_bad", "long_eligible", "long_bad"])
def test_boolean_counts_rejected(field):
    assert evaluate(replace(example(), **{field: True}))["reason"] == "invalid_counts"
```
Run:
```bash
python slo_budget.py
python -m pytest -q
```
Expected: **22 pytest cases** (16 standalone + 6 parameterized). Execution has not been performed.

## 4. Approved staging validation
1. Instrument completed, failed and truncated inference streams, including terminal states.
2. Verify eligible population and avoid double-counting retries.
3. Measure TTFT, TPOT, scheduling latency, GPU allocatable capacity and checkpoint restores.
4. Export counters to Prometheus and inspect short/long burn windows with minimum-volume safeguards.
5. Simulate zone loss and measure N-1 throughput and stream completion at bounded load.
6. Verify independent synthetic probes, missing-data alerts and pager acknowledgement.
7. Attach UTC reports, change approvals, rollback and incident response evidence.

## 5. 100-point assessment
SLI design (20), error-budget math (20), capacity and training SLOs (20), governance (20), evidence and runbooks (20). Educational pass >=90 with no critical security or integrity violation.
