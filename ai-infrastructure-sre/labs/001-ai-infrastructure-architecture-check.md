# Chapter 001 — Executable AI Infrastructure Architecture Lab

**Publication:** Complete. **Execution and live validation:** Pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Deterministic capacity and policy evaluator
Save as `infrastructure_acceptance.py`:
```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Platform:
    zones: dict[str, float]
    peak_demand: float
    reserve_fraction: float
    workload_identity: bool
    tenant_isolation: bool
    driver_compatibility_verified: bool
    gpu_device_plugin_healthy: bool
    observability_enabled: bool
    checkpoint_restore_tested: bool
    incident_owner_assigned: bool

def evaluate(p):
    if not isinstance(p, Platform):
        raise TypeError("Platform required")
    if not isinstance(p.zones, dict) or len(p.zones) < 3:
        return {"accepted": False, "reason": "zone_count"}
    if any(not isinstance(k, str) or not k or type(v) not in (int, float)
           or not math.isfinite(v) or v <= 0 for k, v in p.zones.items()):
        return {"accepted": False, "reason": "invalid_capacity"}
    if any(type(v) not in (int, float) or not math.isfinite(v)
           for v in (p.peak_demand, p.reserve_fraction)):
        return {"accepted": False, "reason": "invalid_demand"}
    if p.peak_demand < 0 or not 0 <= p.reserve_fraction < 1:
        return {"accepted": False, "reason": "invalid_demand"}
    n1 = sum(p.zones.values()) - max(p.zones.values())
    gates = {
        "n_minus_one": p.peak_demand <= n1 * (1 - p.reserve_fraction),
        "workload_identity": p.workload_identity is True,
        "tenant_isolation": p.tenant_isolation is True,
        "driver_compatibility": p.driver_compatibility_verified is True,
        "device_plugin": p.gpu_device_plugin_healthy is True,
        "observability": p.observability_enabled is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "incident_owner": p.incident_owner_assigned is True,
    }
    failed = sorted(k for k, v in gates.items() if not v)
    return {"accepted": not failed, "failed_gates": failed,
            "n_minus_one_capacity": round(n1, 3),
            "headroom": round(n1 - p.peak_demand, 3)}

def example():
    return Platform({"zone-a": 100, "zone-b": 100, "zone-c": 100},
                    150, 0.20, True, True, True, True, True, True, True)

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
Values and gates are synthetic; no GPU, Kubernetes or cloud verification occurs.

## 2. Automated tests
Save as `test_infrastructure_acceptance.py`:
```python
from dataclasses import replace
import pytest
from infrastructure_acceptance import evaluate, example

def test_reference_passes():
    assert evaluate(example())["accepted"]

def test_n1_shortfall():
    assert "n_minus_one" in evaluate(replace(example(), peak_demand=170))["failed_gates"]

def test_n1_boundary():
    assert evaluate(replace(example(), peak_demand=160))["accepted"]

def test_two_zones():
    assert evaluate(replace(example(), zones={"a": 100, "b": 100}))["reason"] == "zone_count"

def test_negative_capacity():
    assert evaluate(replace(example(), zones={"a": -1, "b": 100, "c": 100}))["reason"] == "invalid_capacity"

def test_nan_demand():
    assert evaluate(replace(example(), peak_demand=float("nan")))["reason"] == "invalid_demand"

@pytest.mark.parametrize("field", [
    "workload_identity", "tenant_isolation", "driver_compatibility_verified",
    "gpu_device_plugin_healthy", "observability_enabled",
    "checkpoint_restore_tested", "incident_owner_assigned",
])
def test_policy_fail_closed(field):
    assert not evaluate(replace(example(), **{field: False}))["accepted"]

def test_nonboolean_gate():
    assert not evaluate(replace(example(), workload_identity="yes"))["accepted"]

def test_invalid_type():
    with pytest.raises(TypeError):
        evaluate({})
```
Run:
```bash
python infrastructure_acceptance.py
python -m pytest -q
```
Expected **15 pytest cases** (7 parameterized + 8 standalone). Tests have not been executed.

## 3. Live staging follow-up
Collect GPU host inventory, `nvidia-smi` diagnostics, Kubernetes allocatable GPUs, driver/operator versions, per-zone capacity, checkpoint restore proof, SLO dashboards and ownership approvals. Compare actual evidence to synthetic gates before claiming readiness.

## 4. Assessment
Score architecture mapping, infrastructure discovery, SLO/capacity, incident response and runbook evidence at 20 points each. Educational pass >=90/100 with all critical security/integrity gates.
