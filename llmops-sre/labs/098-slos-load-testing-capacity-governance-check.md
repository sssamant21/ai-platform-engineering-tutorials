# Chapter 098 — Executable SLO, Load Testing and Capacity Governance Lab

**Status:** Published. Offline pytest execution and live GPU load validation pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Offline acceptance evaluator
Save as `slo_capacity.py`:
```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Measurement:
    eligible_requests: int
    completed_successfully: int
    p95_ttft_ms: float
    p95_tpot_ms: float
    interrupted_streams: int

@dataclass(frozen=True)
class Governance:
    measurement: Measurement
    zone_safe_rps: dict[str, float]
    forecast_peak_rps: float
    min_headroom_fraction: float
    availability_target: float
    max_ttft_ms: float
    max_tpot_ms: float
    max_stream_failure_fraction: float
    tenant_isolation_verified: bool
    budget_guard_verified: bool
    signed_model_verified: bool
    load_test_approved: bool
    owner_approved: bool

def evaluate(g):
    if not isinstance(g, Governance) or not isinstance(g.measurement, Measurement):
        raise TypeError("Governance with Measurement required")
    m = g.measurement
    ints = (m.eligible_requests, m.completed_successfully, m.interrupted_streams)
    if any(type(v) is not int or v < 0 for v in ints):
        return {"accepted": False, "reason": "invalid_counts"}
    if m.eligible_requests == 0 or m.completed_successfully > m.eligible_requests or m.interrupted_streams > m.eligible_requests:
        return {"accepted": False, "reason": "invalid_counts"}
    nums = (m.p95_ttft_ms, m.p95_tpot_ms, g.forecast_peak_rps,
            g.min_headroom_fraction, g.availability_target,
            g.max_ttft_ms, g.max_tpot_ms, g.max_stream_failure_fraction)
    if any(type(x) not in (int, float) or not math.isfinite(x) for x in nums):
        return {"accepted": False, "reason": "invalid_threshold"}
    if (min(m.p95_ttft_ms, m.p95_tpot_ms, g.forecast_peak_rps,
            g.max_ttft_ms, g.max_tpot_ms) < 0
        or not 0 <= g.min_headroom_fraction < 1
        or not 0 < g.availability_target <= 1
        or not 0 <= g.max_stream_failure_fraction <= 1):
        return {"accepted": False, "reason": "invalid_threshold"}
    if not isinstance(g.zone_safe_rps, dict) or len(g.zone_safe_rps) < 3:
        return {"accepted": False, "reason": "insufficient_zones"}
    if any(not isinstance(k, str) or not k or type(v) not in (int, float)
           or not math.isfinite(v) or v <= 0 for k, v in g.zone_safe_rps.items()):
        return {"accepted": False, "reason": "invalid_capacity"}
    n1 = sum(g.zone_safe_rps.values()) - max(g.zone_safe_rps.values())
    success = m.completed_successfully / m.eligible_requests
    stream_failure = m.interrupted_streams / m.eligible_requests
    checks = {
        "availability": success >= g.availability_target,
        "ttft": m.p95_ttft_ms <= g.max_ttft_ms,
        "tpot": m.p95_tpot_ms <= g.max_tpot_ms,
        "stream_completion": stream_failure <= g.max_stream_failure_fraction,
        "n_minus_one": g.forecast_peak_rps <= n1 * (1 - g.min_headroom_fraction),
        "tenant_isolation": g.tenant_isolation_verified is True,
        "budget_guard": g.budget_guard_verified is True,
        "signed_model": g.signed_model_verified is True,
        "load_test_approval": g.load_test_approved is True,
        "owner_approval": g.owner_approved is True,
    }
    failed = sorted(k for k, v in checks.items() if not v)
    return {"accepted": not failed, "failed_gates": failed,
            "availability": round(success, 6),
            "stream_failure_fraction": round(stream_failure, 6),
            "n_minus_one_safe_rps": round(n1, 3),
            "headroom_rps": round(n1 - g.forecast_peak_rps, 3)}

def example():
    return Governance(
        Measurement(10000, 9995, 900, 45, 2),
        {"zone-a": 120.0, "zone-b": 120.0, "zone-c": 120.0},
        180.0, 0.20, 0.999, 1800, 100, 0.001,
        True, True, True, True, True,
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

**Limits:** The script evaluates synthetic aggregate metrics. p95 values must come from a correctly aggregated histogram or raw measurements; averaging per-pod p95 is invalid. Signed artifacts, approvals, budgets and tenant controls are asserted, not independently verified.

## 2. Automated tests
Save as `test_slo_capacity.py`:
```python
from dataclasses import replace
import pytest
from slo_capacity import Measurement, Governance, evaluate, example

def test_baseline_passes():
    r = evaluate(example())
    assert r["accepted"] and r["n_minus_one_safe_rps"] == 240

def test_availability_breach():
    g = example()
    assert "availability" in evaluate(replace(g, measurement=replace(g.measurement, completed_successfully=9800)))["failed_gates"]

def test_ttft_breach():
    g = example()
    assert "ttft" in evaluate(replace(g, measurement=replace(g.measurement, p95_ttft_ms=1801)))["failed_gates"]

def test_tpot_breach():
    g = example()
    assert "tpot" in evaluate(replace(g, measurement=replace(g.measurement, p95_tpot_ms=101)))["failed_gates"]

def test_stream_breach():
    g = example()
    assert "stream_completion" in evaluate(replace(g, measurement=replace(g.measurement, interrupted_streams=20)))["failed_gates"]

def test_n1_shortfall():
    assert "n_minus_one" in evaluate(replace(example(), forecast_peak_rps=200))["failed_gates"]

def test_n1_boundary():
    assert evaluate(replace(example(), forecast_peak_rps=192))["accepted"]

def test_asymmetric_zone_failure():
    g = replace(example(), zone_safe_rps={"a": 250.0, "b": 100.0, "c": 100.0})
    assert "n_minus_one" in evaluate(g)["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("tenant_isolation_verified", "tenant_isolation"),
    ("budget_guard_verified", "budget_guard"),
    ("signed_model_verified", "signed_model"),
    ("load_test_approved", "load_test_approval"),
    ("owner_approved", "owner_approval"),
])
def test_policy_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_zero_requests():
    g = example()
    assert evaluate(replace(g, measurement=replace(g.measurement, eligible_requests=0)))["reason"] == "invalid_counts"

def test_negative_count():
    g = example()
    assert evaluate(replace(g, measurement=replace(g.measurement, interrupted_streams=-1)))["reason"] == "invalid_counts"

def test_invalid_p95():
    g = example()
    assert evaluate(replace(g, measurement=replace(g.measurement, p95_ttft_ms=float("nan"))))["reason"] == "invalid_threshold"

def test_invalid_zone_capacity():
    assert evaluate(replace(example(), zone_safe_rps={"a": 100, "b": 100, "c": -1}))["reason"] == "invalid_capacity"

def test_insufficient_zones():
    assert evaluate(replace(example(), zone_safe_rps={"a": 100, "b": 100}))["reason"] == "insufficient_zones"

def test_invalid_input():
    with pytest.raises(TypeError):
        evaluate({})
```

Run:
```bash
python slo_capacity.py
python -m pytest -q
```
Expected: **19 pytest cases** (5 parameterized plus 14 standalone). Execution not performed.

## 3. Live staging extensions
1. Obtain test window, rate ceiling, budget and rollback approval.
2. Generate synthetic workloads by prompt/output length and model route.
3. Measure request and stream completion, TTFT, TPOT, queue, GPU/KV pressure and token spend.
4. Determine safe zone throughput only where SLOs pass.
5. Test largest-zone loss and confirm N-1 headroom.
6. Validate fair admission, 429 behavior, bounded retries and budget limits.
7. Run soak test, verify memory stability and autoscaler convergence.
8. Produce results, SLO burn-rate graphs, uncertainty, owners and remediation plan.

## 4. 100-point assessment
| Domain | Points |
|---|---:|
| SLO definitions and instrumentation | 20 |
| Load methodology and safety | 20 |
| N-1 capacity governance | 20 |
| Security, fairness and budget | 20 |
| Runbooks and evidence | 20 |
| **Total** | **100** |

Pass >=90/100 with all critical policy gates passing. Real GPU measurements and review remain pending.
