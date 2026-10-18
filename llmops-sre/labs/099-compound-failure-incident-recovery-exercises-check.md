# Chapter 099 — Executable Compound Failure and Incident Recovery Lab

**Status:** Published; local pytest execution, live staging game days and independent review pending.

## 1. Prerequisites
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Offline incident evaluator
Save as `compound_recovery.py`:
```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Recovery:
    zone_safe_rps: dict[str, float]
    forecast_peak_rps: float
    reserve_fraction: float
    eligible_requests: int
    completed_requests: int
    p95_ttft_ms: float
    rpo_lag_seconds: int
    rpo_target_seconds: int
    measured_rto_seconds: int
    rto_target_seconds: int
    staging_isolated: bool
    owner_approved: bool
    rollback_ready: bool
    observability_healthy: bool
    signed_artifacts: bool
    retrieval_acl_verified: bool
    primary_writers_fenced: bool
    tool_workers_fenced: bool
    idempotency_reconciled: bool
    tenant_isolation_verified: bool
    residency_approved: bool
    budget_guard_verified: bool
    recovery_probes_passed: bool

def evaluate(r):
    if not isinstance(r, Recovery):
        raise TypeError("Recovery required")
    if not isinstance(r.zone_safe_rps, dict) or len(r.zone_safe_rps) < 3:
        return {"accepted": False, "reason": "zone_inventory"}
    if any(not isinstance(k, str) or not k or type(v) not in (int, float)
           or not math.isfinite(v) or v <= 0 for k, v in r.zone_safe_rps.items()):
        return {"accepted": False, "reason": "invalid_capacity"}
    floats = (r.forecast_peak_rps, r.reserve_fraction, r.p95_ttft_ms)
    ints = (r.eligible_requests, r.completed_requests, r.rpo_lag_seconds,
            r.rpo_target_seconds, r.measured_rto_seconds, r.rto_target_seconds)
    if any(type(x) not in (int, float) or not math.isfinite(x) for x in floats):
        return {"accepted": False, "reason": "invalid_metric"}
    if any(type(x) is not int for x in ints):
        return {"accepted": False, "reason": "invalid_metric"}
    if (r.forecast_peak_rps < 0 or not 0 <= r.reserve_fraction < 1
        or r.p95_ttft_ms < 0 or any(x < 0 for x in ints)
        or r.eligible_requests == 0 or r.completed_requests > r.eligible_requests
        or r.rto_target_seconds == 0):
        return {"accepted": False, "reason": "invalid_metric"}
    n1 = sum(r.zone_safe_rps.values()) - max(r.zone_safe_rps.values())
    gates = {
        "n_minus_one": r.forecast_peak_rps <= n1 * (1-r.reserve_fraction),
        "availability": r.completed_requests / r.eligible_requests >= 0.999,
        "ttft": r.p95_ttft_ms <= 1800,
        "rpo": r.rpo_lag_seconds <= r.rpo_target_seconds,
        "rto": r.measured_rto_seconds <= r.rto_target_seconds,
        "staging_isolated": r.staging_isolated,
        "owner_approved": r.owner_approved,
        "rollback_ready": r.rollback_ready,
        "observability_healthy": r.observability_healthy,
        "signed_artifacts": r.signed_artifacts,
        "retrieval_acl_verified": r.retrieval_acl_verified,
        "primary_writers_fenced": r.primary_writers_fenced,
        "tool_workers_fenced": r.tool_workers_fenced,
        "idempotency_reconciled": r.idempotency_reconciled,
        "tenant_isolation_verified": r.tenant_isolation_verified,
        "residency_approved": r.residency_approved,
        "budget_guard_verified": r.budget_guard_verified,
        "recovery_probes_passed": r.recovery_probes_passed,
    }
    failed = sorted(k for k, v in gates.items() if type(v) is not bool or not v)
    return {"accepted": not failed, "failed_gates": failed,
            "n_minus_one_safe_rps": round(n1, 2),
            "availability": round(r.completed_requests / r.eligible_requests, 6)}

def example():
    return Recovery(
        {"a": 120.0, "b": 120.0, "c": 120.0},
        180.0, 0.20, 10000, 9995, 900.0, 15, 30, 210, 300,
        True, True, True, True, True, True, True, True,
        True, True, True, True, True,
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
**Important:** The evaluator is a deterministic educational gate checker. All metrics, approval, integrity, fencing and security flags are **synthetic operator assertions**, not measurements. No actual cloud fault injection, restore, artifact signature verification or external tool execution occurs.

## 3. Automated tests
Save as `test_compound_recovery.py`:
```python
from dataclasses import replace
import pytest
from compound_recovery import Recovery, evaluate, example

def test_baseline():
    r = evaluate(example())
    assert r["accepted"] and r["n_minus_one_safe_rps"] == 240

def test_n1_shortfall():
    assert "n_minus_one" in evaluate(replace(example(), forecast_peak_rps=200))["failed_gates"]

def test_n1_boundary():
    assert evaluate(replace(example(), forecast_peak_rps=192))["accepted"]

def test_availability_breach():
    assert "availability" in evaluate(replace(example(), completed_requests=9900))["failed_gates"]

def test_ttft_breach():
    assert "ttft" in evaluate(replace(example(), p95_ttft_ms=1801))["failed_gates"]

def test_rpo_breach():
    assert "rpo" in evaluate(replace(example(), rpo_lag_seconds=31))["failed_gates"]

def test_rto_breach():
    assert "rto" in evaluate(replace(example(), measured_rto_seconds=301))["failed_gates"]

def test_rpo_boundary():
    assert evaluate(replace(example(), rpo_lag_seconds=30))["accepted"]

def test_rto_boundary():
    assert evaluate(replace(example(), measured_rto_seconds=300))["accepted"]

@pytest.mark.parametrize("field", [
    "staging_isolated", "owner_approved", "rollback_ready",
    "observability_healthy", "signed_artifacts", "retrieval_acl_verified",
    "primary_writers_fenced", "tool_workers_fenced",
    "idempotency_reconciled", "tenant_isolation_verified",
    "residency_approved", "budget_guard_verified", "recovery_probes_passed",
])
def test_critical_gate(field):
    r = evaluate(replace(example(), **{field: False}))
    assert not r["accepted"] and field in r["failed_gates"]

def test_invalid_zone_count():
    assert evaluate(replace(example(), zone_safe_rps={"a": 120, "b": 120}))["reason"] == "zone_inventory"

def test_negative_capacity():
    assert evaluate(replace(example(), zone_safe_rps={"a": -1, "b": 120, "c": 120}))["reason"] == "invalid_capacity"

def test_zero_requests():
    assert evaluate(replace(example(), eligible_requests=0))["reason"] == "invalid_metric"

def test_invalid_rto_target():
    assert evaluate(replace(example(), rto_target_seconds=0))["reason"] == "invalid_metric"

def test_invalid_type():
    with pytest.raises(TypeError):
        evaluate({})
```

Run:
```bash
python compound_recovery.py
python -m pytest -q
```
Expected **27 pytest cases** (13 parameterized + 14 standalone). Execution not performed.

## 4. Authorized staging extensions
1. Record immutable model, tokenizer, image, prompt and retrieval-index revisions.
2. Obtain change/security approval, isolate synthetic tenant and cap request rate, duration and spend.
3. Establish external client SLO baseline and GPU N-1 capacity.
4. Inject one bounded GPU worker or zone fault; capture queue and TTFT.
5. Simulate model registry outage and stale vector checkpoint; verify policy-based fail-closed behavior.
6. Simulate tool timeout; prove idempotency and no duplicate external effects.
7. Practice primary writer fencing and DR recovery to isolated staging with measured RPO/RTO.
8. Abort immediately on tenant, privacy, integrity, security, provenance or budget violation.
9. Recover signed serving revision, retrieval ACLs, tool state, stream completion and telemetry.
10. Record UTC timeline, RCA/COA, owner, deadline, failed gates and retest evidence.

## 5. 100-point rubric
| Area | Points |
|---|---:|
| Experiment hypothesis and safety | 15 |
| SLO measurement and incident command | 15 |
| GPU/control-plane resilience | 15 |
| RAG/artifact/tool integrity | 15 |
| DR and failback | 20 |
| Evidence, RCA/COA and retest | 20 |
| **Total** | **100** |

Pass >=90/100 **and all critical gates pass**. Real production readiness requires live staging drills and independent review.
