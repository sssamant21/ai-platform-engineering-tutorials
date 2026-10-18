# Chapter 094 — Executable Chaos Engineering and Recovery Game-Day Lab

**Status:** Published. Offline tests and real staging fault injection pending.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Safety gate simulator
Save as `chaos_game_day.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class Preflight:
    staging_isolated: bool
    synthetic_data_only: bool
    owner_approved: bool
    security_approved: bool
    rollback_ready: bool
    kill_switch_ready: bool
    monitoring_healthy: bool
    tenant_isolation_verified: bool
    budget_guard_active: bool
    healthy_replicas: int
    max_affected_replicas: int

def can_inject(p):
    if not isinstance(p, Preflight):
        raise TypeError("Preflight required")
    if type(p.healthy_replicas) is not int or type(p.max_affected_replicas) is not int:
        return False
    if p.healthy_replicas < 2 or not 0 < p.max_affected_replicas < p.healthy_replicas:
        return False
    gates = (
        p.staging_isolated, p.synthetic_data_only, p.owner_approved,
        p.security_approved, p.rollback_ready, p.kill_switch_ready,
        p.monitoring_healthy, p.tenant_isolation_verified,
        p.budget_guard_active,
    )
    return all(type(x) is bool and x for x in gates)

@dataclass(frozen=True)
class Observation:
    success_rate: float
    p95_ttft_ms: int
    stream_failure_rate: float
    security_violation: bool
    data_integrity_violation: bool
    tenant_isolation_violation: bool
    budget_violation: bool

def must_abort(o, min_success=0.99, max_ttft_ms=1800, max_stream_fail=0.01):
    if not isinstance(o, Observation):
        raise TypeError("Observation required")
    if any(type(x) not in (float, int) for x in (o.success_rate, o.stream_failure_rate)):
        return True
    if not 0 <= o.success_rate <= 1 or not 0 <= o.stream_failure_rate <= 1:
        return True
    if type(o.p95_ttft_ms) is not int or o.p95_ttft_ms < 0:
        return True
    flags = (o.security_violation, o.data_integrity_violation,
             o.tenant_isolation_violation, o.budget_violation)
    if any(type(x) is not bool for x in flags):
        return True
    return (any(flags) or o.success_rate < min_success
            or o.p95_ttft_ms > max_ttft_ms
            or o.stream_failure_rate > max_stream_fail)

def example():
    return Preflight(True, True, True, True, True, True, True, True, True, 3, 1)

if __name__ == "__main__":
    o = Observation(0.995, 900, 0.002, False, False, False, False)
    print(json.dumps({"injection_allowed": can_inject(example()), "abort": must_abort(o)}, indent=2))
```

**Limitations:** The simulator neither injects faults nor queries production telemetry. Approval, SLO and isolation flags are assertions; live experiments require independent authoritative checks and a human kill switch.

## 3. Automated tests
Save as `test_chaos_game_day.py`:
```python
from dataclasses import replace
import pytest
from chaos_game_day import Preflight, Observation, can_inject, must_abort, example

def good_obs():
    return Observation(0.995, 900, 0.002, False, False, False, False)

def test_safe_preflight():
    assert can_inject(example())

@pytest.mark.parametrize("field", [
    "staging_isolated", "synthetic_data_only", "owner_approved",
    "security_approved", "rollback_ready", "kill_switch_ready",
    "monitoring_healthy", "tenant_isolation_verified", "budget_guard_active",
])
def test_preflight_gate(field):
    assert not can_inject(replace(example(), **{field: False}))

def test_excess_blast_radius():
    assert not can_inject(replace(example(), max_affected_replicas=3))

def test_no_redundancy():
    assert not can_inject(replace(example(), healthy_replicas=1))

def test_healthy_observation():
    assert not must_abort(good_obs())

def test_success_slo_breach():
    assert must_abort(replace(good_obs(), success_rate=0.98))

def test_latency_slo_breach():
    assert must_abort(replace(good_obs(), p95_ttft_ms=1801))

def test_stream_failure_breach():
    assert must_abort(replace(good_obs(), stream_failure_rate=0.02))

@pytest.mark.parametrize("field", [
    "security_violation", "data_integrity_violation",
    "tenant_isolation_violation", "budget_violation",
])
def test_critical_abort(field):
    assert must_abort(replace(good_obs(), **{field: True}))

def test_invalid_rate():
    assert must_abort(replace(good_obs(), success_rate=1.5))

def test_invalid_latency():
    assert must_abort(replace(good_obs(), p95_ttft_ms=-1))

def test_invalid_preflight_type():
    with pytest.raises(TypeError):
        can_inject({})

def test_invalid_observation_type():
    with pytest.raises(TypeError):
        must_abort({})
```

Run:
```bash
python chaos_game_day.py
python -m pytest -q
```
Expected: **25 pytest cases** (9 preflight-gate cases, 4 critical-abort cases and 12 standalone tests). Execution has not been performed.

## 4. Staging game-day extension
1. Define a hypothesis and a safe synthetic traffic mix.
2. Obtain owner/security approval and select a disposable namespace.
3. Confirm baseline inference success, TTFT, stream completion, quality and cost.
4. Validate kill switch and rollback before injecting one bounded fault.
5. Simulate GPU worker loss, registry outage, retrieval lag or tool timeout.
6. Observe abort thresholds and automatically stop only if the **real** injector supports verified stop controls.
7. Restore healthy routing, validate signed model revision and tenant isolation.
8. Record UTC detection, mitigation and recovery times; calculate error-budget impact.
9. Produce RCA/COA and named follow-up tasks.

## 5. 100-point assessment
| Domain | Points |
|---|---:|
| Hypothesis and preflight | 20 |
| Safety and blast-radius controls | 20 |
| SLO observation and abort | 20 |
| Recovery and validation | 20 |
| Evidence, tests and runbooks | 20 |
| **Total** | **100** |

Pass >=90/100 with no critical security/privacy/integrity breach. Real injection and independent review pending.
