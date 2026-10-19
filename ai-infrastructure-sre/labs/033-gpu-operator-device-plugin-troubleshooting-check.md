# Lab 033 — GPU Operator and Device Plugin Troubleshooting Gate

**Python 3.11+ · pytest · offline synthetic readiness policy.** No Kubernetes access, cloud credentials, driver changes, GPU allocation or node disruption.

## Objective

Fail closed when operator reconciliation, driver, toolkit, CDI, device plugin, GPU UUID, MIG, DCGM, security, N-1, checkpoint or rollback evidence is missing.

## File: `gpu_operator_diagnostics.py`

```python
from dataclasses import dataclass, replace
import json
import math
import re

RESOURCE = re.compile(r"^nvidia\.com/(?:gpu|mig-[A-Za-z0-9.-]+)$")

@dataclass(frozen=True)
class Evidence:
    node: str
    resource_key: str
    expected_gpu_units: int
    allocatable_gpu_units: int
    reserved_gpu_units: int
    telemetry_age_s: float
    max_telemetry_age_s: float
    operator_installed: bool
    clusterpolicy_reconciled: bool
    driver_healthy: bool
    toolkit_healthy: bool
    device_plugin_registered: bool
    duplicate_plugin_absent: bool
    cdi_compatible: bool
    mig_strategy_verified: bool
    gpu_uuid_verified: bool
    cuda_canary_passed: bool
    nccl_smoke_passed: bool
    dcgm_healthy: bool
    xid_ecc_reviewed: bool
    security_verified: bool
    n_minus_one_verified: bool
    checkpoint_verified: bool
    change_approved: bool
    rollback_tested: bool
    workload_slo_healthy: bool

def evaluate(e):
    if not isinstance(e, Evidence):
        raise TypeError("Evidence required")
    if not isinstance(e.node, str) or not e.node.strip():
        return {"accepted": False, "reason": "invalid_node"}
    if not isinstance(e.resource_key, str) or not RESOURCE.fullmatch(e.resource_key):
        return {"accepted": False, "reason": "invalid_resource"}
    counts = (e.expected_gpu_units, e.allocatable_gpu_units, e.reserved_gpu_units)
    if (any(type(x) is not int or x < 0 for x in counts)
            or e.expected_gpu_units < 1 or e.reserved_gpu_units > e.allocatable_gpu_units):
        return {"accepted": False, "reason": "invalid_gpu_count"}
    metrics = (e.telemetry_age_s, e.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) for x in metrics)
            or e.telemetry_age_s < 0 or e.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_telemetry"}
    checks = {
        "resource_advertisement": e.allocatable_gpu_units >= e.expected_gpu_units,
        "operator": e.operator_installed is True,
        "clusterpolicy": e.clusterpolicy_reconciled is True,
        "driver": e.driver_healthy is True,
        "toolkit": e.toolkit_healthy is True,
        "device_plugin": e.device_plugin_registered is True,
        "no_duplicate_plugin": e.duplicate_plugin_absent is True,
        "cdi": e.cdi_compatible is True,
        "mig": e.mig_strategy_verified is True,
        "gpu_uuid": e.gpu_uuid_verified is True,
        "cuda": e.cuda_canary_passed is True,
        "nccl": e.nccl_smoke_passed is True,
        "dcgm": e.dcgm_healthy is True,
        "xid_ecc": e.xid_ecc_reviewed is True,
        "security": e.security_verified is True,
        "n_minus_one": e.n_minus_one_verified is True,
        "checkpoint": e.checkpoint_verified is True,
        "change": e.change_approved is True,
        "rollback": e.rollback_tested is True,
        "workload_slo": e.workload_slo_healthy is True,
        "telemetry": e.telemetry_age_s <= e.max_telemetry_age_s,
    }
    failed = sorted(k for k, value in checks.items() if not value)
    return {"accepted": not failed, "failed_gates": failed,
            "node": e.node, "resource_key": e.resource_key,
            "available_gpu_units": e.allocatable_gpu_units - e.reserved_gpu_units}

def example():
    return Evidence(
        "synthetic-gpu-node", "nvidia.com/gpu", 4, 4, 1, 10.0, 120.0,
        *([True] * 19)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

The fixture contains exactly 19 boolean fields, from `operator_installed` through `workload_slo_healthy`. This is an intentionally conservative **operator-managed** profile. Cloud-managed driver/toolkit installations need a different reviewed evidence profile rather than pretending the Operator is installed.

## File: `test_gpu_operator_diagnostics.py`

```python
from dataclasses import replace
import pytest
from gpu_operator_diagnostics import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_empty_node():
    assert evaluate(replace(example(), node=" "))["reason"] == "invalid_node"

def test_invalid_resource():
    assert evaluate(replace(example(), resource_key="gpu"))["reason"] == "invalid_resource"

def test_fractional_gpu():
    assert evaluate(replace(example(), expected_gpu_units=1.5))["reason"] == "invalid_gpu_count"

def test_boolean_gpu():
    assert evaluate(replace(example(), expected_gpu_units=True))["reason"] == "invalid_gpu_count"

def test_overreserved():
    assert evaluate(replace(example(), reserved_gpu_units=5))["reason"] == "invalid_gpu_count"

def test_missing_gpu_advertisement():
    assert "resource_advertisement" in evaluate(
        replace(example(), expected_gpu_units=5))["failed_gates"]

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nan_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=float("nan")))["reason"] == "invalid_telemetry"

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("operator_installed", "operator"),
    ("clusterpolicy_reconciled", "clusterpolicy"),
    ("driver_healthy", "driver"),
    ("toolkit_healthy", "toolkit"),
    ("device_plugin_registered", "device_plugin"),
    ("duplicate_plugin_absent", "no_duplicate_plugin"),
    ("cdi_compatible", "cdi"),
    ("mig_strategy_verified", "mig"),
    ("gpu_uuid_verified", "gpu_uuid"),
    ("cuda_canary_passed", "cuda"),
    ("nccl_smoke_passed", "nccl"),
    ("dcgm_healthy", "dcgm"),
    ("xid_ecc_reviewed", "xid_ecc"),
    ("security_verified", "security"),
    ("n_minus_one_verified", "n_minus_one"),
    ("checkpoint_verified", "checkpoint"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("workload_slo_healthy", "workload_slo"),
])
def test_required_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python gpu_operator_diagnostics.py
python -m pytest -q
```

**Planned: 30 pytest cases** (11 standalone + 19 parameterized). Runtime execution and live GPU validation have not been performed as part of publication.

## Ten synthetic incidents

1. ClusterPolicy fails reconciliation.
2. Host driver unhealthy.
3. Toolkit/CRI mismatch.
4. Device plugin registration missing.
5. Duplicate plugin ownership.
6. CDI injection incompatible.
7. MIG profile unverified.
8. CUDA/NCCL smoke fails.
9. DCGM telemetry stale or Xid/ECC not reviewed.
10. Security, checkpoint, N-1, change or rollback gate fails.

## Staging acceptance

Inventory actual GPU Operator ownership, ClusterPolicy, driver/toolkit versions, kubelet resources, MIG strategy, GPU UUID and DCGM. Perform approved read-only diagnostics and a disposable pinned-image CUDA canary. Simulate plugin/CDI failures only on disposable staging capacity. Verify application correctness, checkpoint, N-1, rollback and SLO. Document findings and owner signoff.

## 100-point rubric

Operator/device plugin 20; driver/CDI/MIG 20; GPU correctness/telemetry 20; incidents/runbooks 20; tests/staging 20. Pass >=90 with no critical isolation, data-loss, GPU health or rollback blocker.
