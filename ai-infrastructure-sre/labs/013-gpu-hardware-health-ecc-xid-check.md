# Chapter 013 — Executable GPU Hardware Health, ECC and Xid Acceptance Lab

**Status:** Published. Tests and live hardware validation pending.

## 1. Setup
Requires Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Synthetic GPU fault evaluator
Save as `gpu_health.py`:
```python
from dataclasses import dataclass
import json
import math
import re

UUID = re.compile(r"^GPU-[0-9a-fA-F-]{8,}$")

@dataclass(frozen=True)
class Health:
    gpu_uuid: str
    node: str
    ecc_supported: bool
    correctable_ecc_delta: int | None
    uncorrectable_ecc_delta: int | None
    critical_xid_count: int
    pcie_aer_errors: int
    telemetry_age_seconds: float
    gpu_present: bool
    thermal_healthy: bool
    workload_healthy: bool
    checkpoint_verified: bool
    isolation_approved: bool

def evaluate(records, expected_uuids, max_telemetry_age=120):
    if not isinstance(records, list) or not records:
        return {"accepted": False, "reason": "empty_inventory"}
    if any(not isinstance(r, Health) for r in records):
        return {"accepted": False, "reason": "invalid_record"}
    if (not isinstance(expected_uuids, (set, frozenset)) or not expected_uuids
            or any(not isinstance(u, str) or not UUID.fullmatch(u) for u in expected_uuids)):
        return {"accepted": False, "reason": "invalid_expected_inventory"}
    if (type(max_telemetry_age) not in (int, float)
            or not math.isfinite(max_telemetry_age) or max_telemetry_age <= 0):
        return {"accepted": False, "reason": "invalid_age_limit"}
    uuids = [r.gpu_uuid for r in records]
    if any(not isinstance(u, str) or not UUID.fullmatch(u) for u in uuids):
        return {"accepted": False, "reason": "invalid_uuid"}
    if len(set(uuids)) != len(uuids):
        return {"accepted": False, "reason": "duplicate_uuid"}
    failures = {}
    for r in records:
        if not isinstance(r.node, str) or not r.node.strip():
            return {"accepted": False, "reason": "missing_node"}
        if (type(r.critical_xid_count) is not int or r.critical_xid_count < 0
                or type(r.pcie_aer_errors) is not int or r.pcie_aer_errors < 0):
            return {"accepted": False, "reason": "invalid_error_count"}
        if (type(r.telemetry_age_seconds) not in (int, float)
                or not math.isfinite(r.telemetry_age_seconds) or r.telemetry_age_seconds < 0):
            return {"accepted": False, "reason": "invalid_telemetry_age"}
        for count in (r.correctable_ecc_delta, r.uncorrectable_ecc_delta):
            if count is not None and (type(count) is not int or count < 0):
                return {"accepted": False, "reason": "invalid_ecc_count"}
        gates = {
            "gpu_present": r.gpu_present is True,
            "telemetry_fresh": r.telemetry_age_seconds <= max_telemetry_age,
            "ecc_support_known": type(r.ecc_supported) is bool,
            "ecc_evidence": (r.ecc_supported is False or
                             (r.ecc_supported is True and r.correctable_ecc_delta is not None
                              and r.uncorrectable_ecc_delta is not None)),
            "uncorrectable_ecc": r.ecc_supported is False or r.uncorrectable_ecc_delta == 0,
            "critical_xid": r.critical_xid_count == 0,
            "pcie_aer": r.pcie_aer_errors == 0,
            "thermal": r.thermal_healthy is True,
            "workload": r.workload_healthy is True,
            "checkpoint": r.checkpoint_verified is True,
            "isolation_approval": r.isolation_approved is True,
        }
        failures[r.gpu_uuid] = sorted(k for k, passed in gates.items() if not passed)
    missing = sorted(expected_uuids - set(uuids))
    unexpected = sorted(set(uuids) - expected_uuids)
    return {"accepted": not missing and not unexpected and all(not v for v in failures.values()),
            "missing": missing, "unexpected": unexpected,
            "failed_gates_by_gpu": failures, "gpu_count": len(records)}

def example():
    r = Health("GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee", "gpu-node-staging-01",
               True, 0, 0, 0, 0, 10.0, True, True, True, True, True)
    return [r], {r.gpu_uuid}

if __name__ == "__main__":
    records, expected = example()
    print(json.dumps(evaluate(records, expected), indent=2))
```

**Interpretation:** This is a conservative *synthetic acceptance policy*, not a universal NVIDIA diagnostic threshold. `critical_xid_count` must be classified by an approved code-specific policy before input; `pcie_aer_errors == 0` is a lab simplification. ECC-unsupported devices should be assessed through other supported health signals. A `True` checkpoint or approval flag is not evidence of actual recovery or authorization.

## 3. Automated tests
Save as `test_gpu_health.py`:
```python
from dataclasses import replace
import pytest
from gpu_health import evaluate, example

def test_baseline():
    r, e = example()
    assert evaluate(r, e)["accepted"]

def test_empty():
    assert evaluate([], set())["reason"] == "empty_inventory"

def test_wrong_record():
    _, e = example()
    assert evaluate([{}], e)["reason"] == "invalid_record"

def test_bad_expected():
    r, _ = example()
    assert evaluate(r, {"bad"})["reason"] == "invalid_expected_inventory"

def test_duplicate():
    r, e = example()
    assert evaluate(r + r, e)["reason"] == "duplicate_uuid"

def test_invalid_uuid():
    r, e = example()
    assert evaluate([replace(r[0], gpu_uuid="gpu0")], e)["reason"] == "invalid_uuid"

def test_missing():
    r, e = example()
    assert evaluate(r, e | {"GPU-ffffffff-ffff-ffff-ffff-ffffffffffff"})["missing"]

def test_unexpected():
    r, _ = example()
    assert evaluate(r, {"GPU-ffffffff-ffff-ffff-ffff-ffffffffffff"})["unexpected"]

def test_invalid_ecc():
    r, e = example()
    assert evaluate([replace(r[0], uncorrectable_ecc_delta=-1)], e)["reason"] == "invalid_ecc_count"

def test_invalid_xid():
    r, e = example()
    assert evaluate([replace(r[0], critical_xid_count=-1)], e)["reason"] == "invalid_error_count"

def test_stale():
    r, e = example()
    result = evaluate([replace(r[0], telemetry_age_seconds=121)], e)
    assert "telemetry_fresh" in result["failed_gates_by_gpu"][r[0].gpu_uuid]

def test_unsupported_ecc():
    r, e = example()
    assert evaluate([replace(r[0], ecc_supported=False,
        correctable_ecc_delta=None, uncorrectable_ecc_delta=None)], e)["accepted"]

def test_missing_ecc_evidence():
    r, e = example()
    result = evaluate([replace(r[0], uncorrectable_ecc_delta=None)], e)
    assert "ecc_evidence" in result["failed_gates_by_gpu"][r[0].gpu_uuid]

@pytest.mark.parametrize("field,gate,value", [
    ("gpu_present", "gpu_present", False),
    ("uncorrectable_ecc_delta", "uncorrectable_ecc", 1),
    ("critical_xid_count", "critical_xid", 1),
    ("pcie_aer_errors", "pcie_aer", 1),
    ("thermal_healthy", "thermal", False),
    ("workload_healthy", "workload", False),
    ("checkpoint_verified", "checkpoint", False),
    ("isolation_approved", "isolation_approval", False),
])
def test_failure_gates(field, gate, value):
    r, e = example()
    result = evaluate([replace(r[0], **{field: value})], e)
    assert gate in result["failed_gates_by_gpu"][r[0].gpu_uuid]

def test_nonboolean_health():
    r, e = example()
    result = evaluate([replace(r[0], thermal_healthy="yes")], e)
    assert "thermal" in result["failed_gates_by_gpu"][r[0].gpu_uuid]

def test_nan_age():
    r, e = example()
    assert evaluate([replace(r[0], telemetry_age_seconds=float("nan"))], e)["reason"] == "invalid_telemetry_age"

def test_bad_age_limit():
    r, e = example()
    assert evaluate(r, e, max_telemetry_age=0)["reason"] == "invalid_age_limit"
```
Run:
```bash
python gpu_health.py
python -m pytest -q
```
Expected **24 tests** (16 standalone + 8 parameterized); tests have not been executed.

## 4. Staging acceptance evidence
1. Record supported ECC metrics, GPU UUID/model, driver, firmware and Xid policy.
2. Collect NVML/DCGM, NVRM kernel logs and Kubernetes events with UTC timestamps.
3. Simulate **alert inputs**, not physical memory corruption, GPU resets or hardware faults.
4. Correlate affected workloads, application SLO, checkpoint readiness and available N-1 capacity.
5. Validate isolation and vendor escalation procedure with an approved change.
6. Verify recovery, GPU inventory, smoke workload, metrics and pager routing.
7. Attach evidence, review findings, owner, ticket and independent signoff.

## 5. 100-point rubric
ECC/Xid theory (20), diagnostics (20), alert correlation (20), runbooks (20), lab/evidence (20). Pass >=90 without critical blocker.
