# Chapter 091 — Executable High Availability, N-1 and Multi-Zone Recovery Lab

**Status:** Published; offline tests, staging failover and independent review pending. No real cloud/GPU actions.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic N-1 evaluator
Save as `ha_recovery.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class Capacity:
    zones: dict[str, int]
    peak_demand: int
    minimum_zones: int = 3

def evaluate_capacity(c):
    if not isinstance(c, Capacity):
        raise TypeError("Capacity required")
    if not isinstance(c.zones, dict) or any(not isinstance(k, str) or not k for k in c.zones):
        raise ValueError("named zones required")
    if type(c.peak_demand) is not int or c.peak_demand < 0:
        raise ValueError("nonnegative demand required")
    if type(c.minimum_zones) is not int or c.minimum_zones < 2:
        raise ValueError("minimum_zones must be at least two")
    if any(type(v) is not int or v < 0 for v in c.zones.values()):
        raise ValueError("nonnegative integer capacity required")
    total = sum(c.zones.values())
    n_minus_one = total - max(c.zones.values(), default=0)
    enough_zones = len(c.zones) >= c.minimum_zones
    return {
        "total_capacity": total,
        "n_minus_one_capacity": n_minus_one,
        "headroom": n_minus_one - c.peak_demand,
        "n_minus_one_pass": enough_zones and n_minus_one >= c.peak_demand,
        "zones_present": len(c.zones),
    }

@dataclass(frozen=True)
class Recovery:
    n_minus_one_pass: bool
    healthy_endpoints: bool
    model_ready: bool
    tenant_isolation_verified: bool
    signed_model_verified: bool
    budget_controls_verified: bool
    slo_verified: bool
    measured_rto_seconds: int
    target_rto_seconds: int
    owner_approved: bool

def can_restore_traffic(r):
    if not isinstance(r, Recovery):
        raise TypeError("Recovery required")
    if type(r.measured_rto_seconds) is not int or type(r.target_rto_seconds) is not int:
        return False
    if r.measured_rto_seconds < 0 or r.target_rto_seconds <= 0:
        return False
    gates = (
        r.n_minus_one_pass, r.healthy_endpoints, r.model_ready,
        r.tenant_isolation_verified, r.signed_model_verified,
        r.budget_controls_verified, r.slo_verified, r.owner_approved,
    )
    return all(type(x) is bool and x for x in gates) and r.measured_rto_seconds <= r.target_rto_seconds

def example():
    return Capacity({"zone-a": 120, "zone-b": 120, "zone-c": 120}, 200)

if __name__ == "__main__":
    print(json.dumps(evaluate_capacity(example()), indent=2))
```

**Limitations:** The capacities are supplied, synthetic safe throughput values. This arithmetic does not account for queueing, cold starts, heterogeneous GPUs, zone-correlated failures, storage dependencies, latency, live cloud quotas or network congestion. `True` recovery flags are operator assertions, not verified integrations.

## 3. Automated tests
Save as `test_ha_recovery.py`:
```python
from dataclasses import replace
import pytest
from ha_recovery import Capacity, Recovery, evaluate_capacity, can_restore_traffic, example

def good():
    return Recovery(True, True, True, True, True, True, True, 45, 60, True)

def test_n_minus_one_capacity():
    r = evaluate_capacity(example())
    assert r["n_minus_one_capacity"] == 240
    assert r["headroom"] == 40 and r["n_minus_one_pass"]

def test_overloaded_n_minus_one():
    assert not evaluate_capacity(replace(example(), peak_demand=260))["n_minus_one_pass"]

def test_two_zone_design_fails_three_zone_policy():
    assert not evaluate_capacity(Capacity({"a": 120, "b": 120}, 100))["n_minus_one_pass"]

def test_asymmetric_zone_loss():
    r = evaluate_capacity(Capacity({"a": 300, "b": 100, "c": 100}, 220))
    assert r["n_minus_one_capacity"] == 200 and not r["n_minus_one_pass"]

def test_exact_capacity_boundary():
    assert evaluate_capacity(replace(example(), peak_demand=240))["n_minus_one_pass"]

def test_restore_passes():
    assert can_restore_traffic(good())

@pytest.mark.parametrize("field", [
    "n_minus_one_pass", "healthy_endpoints", "model_ready",
    "tenant_isolation_verified", "signed_model_verified",
    "budget_controls_verified", "slo_verified", "owner_approved",
])
def test_failed_recovery_gate(field):
    assert not can_restore_traffic(replace(good(), **{field: False}))

def test_rto_breach():
    assert not can_restore_traffic(replace(good(), measured_rto_seconds=61))

def test_rto_boundary():
    assert can_restore_traffic(replace(good(), measured_rto_seconds=60))

@pytest.mark.parametrize("zones", [
    {"a": -1, "b": 10, "c": 10},
    {"a": 1.5, "b": 10, "c": 10},
    {"": 10, "b": 10, "c": 10},
])
def test_invalid_zones(zones):
    with pytest.raises(ValueError):
        evaluate_capacity(Capacity(zones, 10))

def test_invalid_demand():
    with pytest.raises(ValueError):
        evaluate_capacity(Capacity({"a": 10, "b": 10, "c": 10}, -1))

def test_invalid_rto():
    assert not can_restore_traffic(replace(good(), target_rto_seconds=0))

def test_invalid_recovery_type():
    with pytest.raises(TypeError):
        can_restore_traffic({})
```

Run:
```bash
python ha_recovery.py
python -m pytest -q
```
Expected: **23 pytest cases** (8 recovery-gate parameterizations, 3 invalid-zone parameterizations and 12 standalone tests). Execution has not been performed.

## 4. Production-oriented staging extensions
1. Discover actual GPU placement per zone and measure safe per-zone token throughput.
2. Run synthetic peak traffic; capture success, p95 TTFT, TPOT, queue age and stream completion.
3. Simulate controlled zone traffic loss; measure detection, diversion and model warmup.
4. Check RTO, retrieval freshness/RPO and tenant isolation.
5. Validate quota reservations, cost ledger and retry ceilings during failover.
6. Restore zone with canary, record signed model digest and independent reviewer approval.
7. Capture UTC incident timeline, rollback evidence, cost impact and RCA/COA.

## 5. 100-point assessment
| Dimension | Points |
|---|---:|
| Capacity and failure-domain modeling | 20 |
| N-1 failure testing | 20 |
| Security and budget gates | 20 |
| RTO/SLO recovery | 20 |
| Tests, runbooks and evidence | 20 |
| **Total** | **100** |

Pass >=90/100 with no critical isolation or policy bypass. Live staging and independent review remain pending.
