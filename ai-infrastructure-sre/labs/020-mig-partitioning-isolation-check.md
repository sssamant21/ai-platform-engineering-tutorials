# Chapter 020 — MIG Partitioning and Isolation Executable Lab

**Status:** Published. Offline tests and live MIG staging acceptance pending.

## Scope and prerequisites
Python 3.11+ and pytest. This lab uses **synthetic** MIG profiles, not claims about any real NVIDIA SKU. Approved profile layouts must come from `nvidia-smi mig -lgip`, `-lcip`, the NVIDIA MIG guide and platform policy. It performs no GPU reset or MIG mutation.

## File: `mig_policy.py`
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class MIGChange:
    node: str
    owner: str
    gpu_uuid: str
    profile: str
    requested: int
    available: int
    legal_profiles: frozenset[str]
    sku_supported: bool
    mig_mode_verified: bool
    controller_unique: bool
    plugin_strategy_verified: bool
    tenant_isolation_approved: bool
    checkpoint_verified: bool
    drain_verified: bool
    n_minus_one_verified: bool
    telemetry_fresh: bool
    gpu_health_verified: bool
    change_approved: bool
    rollback_tested: bool
    workload_slo_verified: bool

def evaluate(c):
    if not isinstance(c, MIGChange):
        raise TypeError("MIGChange required")
    if any(not isinstance(v, str) or not v.strip()
           for v in (c.node, c.owner, c.gpu_uuid, c.profile)):
        return {"accepted": False, "reason": "missing_identity"}
    if not c.gpu_uuid.startswith("GPU-"):
        return {"accepted": False, "reason": "invalid_gpu_uuid"}
    if (type(c.requested) is not int or type(c.available) is not int
            or c.requested <= 0 or c.available < 0):
        return {"accepted": False, "reason": "invalid_capacity"}
    if not isinstance(c.legal_profiles, frozenset):
        return {"accepted": False, "reason": "invalid_policy"}
    gates = {
        "legal_profile": c.profile in c.legal_profiles,
        "capacity": c.requested <= c.available,
        "sku": c.sku_supported is True,
        "mig_mode": c.mig_mode_verified is True,
        "controller": c.controller_unique is True,
        "device_plugin": c.plugin_strategy_verified is True,
        "isolation": c.tenant_isolation_approved is True,
        "checkpoint": c.checkpoint_verified is True,
        "drain": c.drain_verified is True,
        "n_minus_one": c.n_minus_one_verified is True,
        "telemetry": c.telemetry_fresh is True,
        "gpu_health": c.gpu_health_verified is True,
        "change_approval": c.change_approved is True,
        "rollback": c.rollback_tested is True,
        "workload_slo": c.workload_slo_verified is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "node": c.node, "profile": c.profile,
            "remaining_instances": c.available - c.requested}

def example():
    return MIGChange(
        "gpu-staging-01", "gpu-platform-sre",
        "GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
        "synthetic-small", 2, 3, frozenset({"synthetic-small", "synthetic-large"}),
        *([True] * 13)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_mig_policy.py`
```python
from dataclasses import replace
import pytest
from mig_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_owner():
    assert evaluate(replace(example(), owner=" "))["reason"] == "missing_identity"

def test_uuid():
    assert evaluate(replace(example(), gpu_uuid="invalid"))["reason"] == "invalid_gpu_uuid"

def test_negative():
    assert evaluate(replace(example(), available=-1))["reason"] == "invalid_capacity"

def test_bool_capacity():
    assert evaluate(replace(example(), requested=True))["reason"] == "invalid_capacity"

def test_zero_request():
    assert evaluate(replace(example(), requested=0))["reason"] == "invalid_capacity"

def test_policy_type():
    assert evaluate(replace(example(), legal_profiles=set()))["reason"] == "invalid_policy"

def test_unknown_profile():
    assert "legal_profile" in evaluate(replace(example(), profile="unknown"))["failed_gates"]

def test_insufficient_capacity():
    assert "capacity" in evaluate(replace(example(), requested=4))["failed_gates"]

def test_remaining():
    assert evaluate(example())["remaining_instances"] == 1

@pytest.mark.parametrize("field,gate", [
    ("sku_supported", "sku"),
    ("mig_mode_verified", "mig_mode"),
    ("controller_unique", "controller"),
    ("plugin_strategy_verified", "device_plugin"),
    ("tenant_isolation_approved", "isolation"),
    ("checkpoint_verified", "checkpoint"),
    ("drain_verified", "drain"),
    ("n_minus_one_verified", "n_minus_one"),
    ("telemetry_fresh", "telemetry"),
    ("gpu_health_verified", "gpu_health"),
    ("change_approved", "change_approval"),
    ("rollback_tested", "rollback"),
    ("workload_slo_verified", "workload_slo"),
])
def test_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_approval():
    assert "change_approval" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_nonboolean_isolation():
    assert "isolation" in evaluate(replace(example(), tenant_isolation_approved=1))["failed_gates"]
```

Run:
```bash
python -m pip install pytest
python mig_policy.py
python -m pytest -q
```
**Expected 26 pytest cases** (13 standalone + 13 parameterized); not executed as part of publication.

## Staging-only acceptance exercises
1. Capture physical GPU UUID, SKU, driver, MIG mode, supported GI/CI profiles and placement constraints.
2. Compare Kubernetes advertised resource names against configured device-plugin strategy.
3. Validate synthetic unsupported profile, insufficient capacity, stale telemetry and unapproved tenant isolation.
4. Under a **separate authorized maintenance change**, verify checkpoint, workload evacuation and N-1 before MIG layout change.
5. Confirm new MIG UUIDs, CUDA device visibility, workload memory isolation, DCGM telemetry, inference SLO and rollback.
6. Attach UTC evidence, compatibility references, asset reconciliation, security approval and owner signoff.

**Limitations:** The synthetic capacity integer represents precomputed available instances. It does not solve real MIG placement geometry, fragmentation, cross-GI constraints or GPU architecture support. These require vendor-specific evidence and staging tests.

## 100-point assessment
Architecture 20; scheduling/lifecycle 20; isolation/capacity 20; incident runbooks 20; tests/evidence 20. Pass >=90 with no critical safety failure.
