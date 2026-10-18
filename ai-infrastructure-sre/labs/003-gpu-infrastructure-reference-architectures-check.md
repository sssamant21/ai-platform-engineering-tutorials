# Chapter 003 — Executable GPU Reference Architecture Acceptance Lab

**Status:** Published. Execution on a workstation, real GPU benchmarking and independent review pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Reference architecture evaluator
Save as `gpu_architecture.py`:
```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Architecture:
    zone_safe_rps: dict[str, float]
    forecast_peak_rps: float
    reserve_fraction: float
    workload_class: str
    identity_verified: bool
    tenant_isolation_verified: bool
    artifact_signatures_verified: bool
    gpu_runtime_compatible: bool
    network_tested: bool
    checkpoint_restore_tested: bool
    rpo_rto_defined: bool
    telemetry_healthy: bool
    budget_approved: bool
    owner_approved: bool

def evaluate(a):
    if not isinstance(a, Architecture):
        raise TypeError("Architecture required")
    if a.workload_class not in {"online_inference", "distributed_training"}:
        return {"accepted": False, "reason": "invalid_workload"}
    if not isinstance(a.zone_safe_rps, dict) or len(a.zone_safe_rps) < 3:
        return {"accepted": False, "reason": "insufficient_zones"}
    if any(not isinstance(k, str) or not k or type(v) not in (int, float)
           or not math.isfinite(v) or v <= 0 for k, v in a.zone_safe_rps.items()):
        return {"accepted": False, "reason": "invalid_zone_capacity"}
    if any(type(v) not in (int, float) or not math.isfinite(v)
           for v in (a.forecast_peak_rps, a.reserve_fraction)):
        return {"accepted": False, "reason": "invalid_demand"}
    if a.forecast_peak_rps < 0 or not 0 <= a.reserve_fraction < 1:
        return {"accepted": False, "reason": "invalid_demand"}
    n1 = sum(a.zone_safe_rps.values()) - max(a.zone_safe_rps.values())
    gates = {
        "n_minus_one": a.forecast_peak_rps <= n1 * (1 - a.reserve_fraction),
        "identity": a.identity_verified is True,
        "tenant_isolation": a.tenant_isolation_verified is True,
        "artifact_signatures": a.artifact_signatures_verified is True,
        "gpu_runtime": a.gpu_runtime_compatible is True,
        "network": a.network_tested is True,
        "checkpoint_restore": a.checkpoint_restore_tested is True,
        "recovery_targets": a.rpo_rto_defined is True,
        "telemetry": a.telemetry_healthy is True,
        "budget": a.budget_approved is True,
        "owner": a.owner_approved is True,
    }
    failed = sorted(k for k, passed in gates.items() if not passed)
    return {"accepted": not failed, "failed_gates": failed,
            "n_minus_one_safe_rps": round(n1, 3),
            "remaining_headroom_rps": round(n1 - a.forecast_peak_rps, 3)}

def example():
    return Architecture(
        {"zone-a": 100.0, "zone-b": 100.0, "zone-c": 100.0},
        150.0, 0.20, "online_inference",
        True, True, True, True, True, True, True, True, True, True,
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

The input values and policy flags are **synthetic assertions**. The script neither deploys infrastructure nor verifies real GPU drivers, network, storage, artifact signatures, SLOs or checkpoint restores.

## 2. Automated tests
Save as `test_gpu_architecture.py`:
```python
from dataclasses import replace
import pytest
from gpu_architecture import evaluate, example

def test_baseline():
    r = evaluate(example())
    assert r["accepted"] and r["n_minus_one_safe_rps"] == 200

def test_n1_shortfall():
    assert "n_minus_one" in evaluate(replace(example(), forecast_peak_rps=180))["failed_gates"]

def test_n1_boundary():
    assert evaluate(replace(example(), forecast_peak_rps=160))["accepted"]

def test_asymmetric_zone():
    r = evaluate(replace(example(), zone_safe_rps={"a": 200, "b": 50, "c": 50}))
    assert "n_minus_one" in r["failed_gates"]

def test_too_few_zones():
    assert evaluate(replace(example(), zone_safe_rps={"a": 100, "b": 100}))["reason"] == "insufficient_zones"

def test_negative_capacity():
    assert evaluate(replace(example(), zone_safe_rps={"a": -1, "b": 100, "c": 100}))["reason"] == "invalid_zone_capacity"

def test_nan_capacity():
    assert evaluate(replace(example(), zone_safe_rps={"a": float("nan"), "b": 100, "c": 100}))["reason"] == "invalid_zone_capacity"

def test_negative_demand():
    assert evaluate(replace(example(), forecast_peak_rps=-1))["reason"] == "invalid_demand"

def test_invalid_reserve():
    assert evaluate(replace(example(), reserve_fraction=1))["reason"] == "invalid_demand"

def test_invalid_workload():
    assert evaluate(replace(example(), workload_class="unknown"))["reason"] == "invalid_workload"

def test_distributed_training():
    assert evaluate(replace(example(), workload_class="distributed_training"))["accepted"]

@pytest.mark.parametrize("field,gate", [
    ("identity_verified", "identity"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("artifact_signatures_verified", "artifact_signatures"),
    ("gpu_runtime_compatible", "gpu_runtime"),
    ("network_tested", "network"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("rpo_rto_defined", "recovery_targets"),
    ("telemetry_healthy", "telemetry"),
    ("budget_approved", "budget"),
    ("owner_approved", "owner"),
])
def test_fail_closed_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_gate():
    assert "identity" in evaluate(replace(example(), identity_verified="yes"))["failed_gates"]

def test_invalid_type():
    with pytest.raises(TypeError):
        evaluate({})
```

Run:
```bash
python gpu_architecture.py
python -m pytest -q
```
Expected: **23 test cases** (13 standalone + 10 parameterized). Test execution has not been performed.

## 3. Live staging acceptance checklist
1. Record GPU hardware, driver/CUDA/NCCL versions, device-plugin and operator health.
2. Measure **safe** per-zone throughput with the target workload mix, latency and stream-completion SLOs.
3. Test largest-zone loss, warmup, quota, queueing and admission at bounded synthetic load.
4. Validate multi-node training fabric, checkpoint restore, optimizer/data cursor consistency and RTO.
5. Verify tenant identity, network segmentation, signed artifacts, secrets and residency.
6. Reconcile GPU-hours, token delivery, retry overhead and budget.
7. Attach UTC incident timeline, metrics, rollback evidence, approvals and independent review.

## 4. Assessment
Requirements and workload mapping (20), topology and storage/network (20), capacity/SLO/cost (20), security/recovery (20), evidence and runbooks (20). Pass >=90/100 with no critical gate violation.
