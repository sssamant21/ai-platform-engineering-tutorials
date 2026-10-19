# Chapter 016 — Executable GPU Hardware Operations Capstone Lab

**Status:** Published; offline tests, staging game day and independent acceptance pending.

## 1. Setup
Python 3.11+ and pytest. No GPU, cloud credentials or network are required for synthetic checks.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```

On PowerShell use `.venv\Scripts\Activate.ps1`.

## 2. Build the fail-closed acceptance evaluator

Save as `gpu_capstone.py`:

```python
from dataclasses import dataclass
import json
import re

DIGEST = re.compile(r"^sha256:[0-9a-f]{64}$")
UUID = re.compile(r"^GPU-[0-9a-fA-F-]{8,}$")

@dataclass(frozen=True)
class Capstone:
    node: str
    owner: str
    platform: str
    environment: str
    image_digest: str
    gpu_uuids: tuple[str, ...]
    expected_gpus: int
    allocatable_gpus: int
    vendor_matrix_verified: bool
    topology_verified: bool
    firmware_approved: bool
    driver_compatible: bool
    single_driver_owner: bool
    image_signature_verified: bool
    gpu_health_verified: bool
    no_critical_xid: bool
    no_uncorrectable_ecc: bool
    device_plugin_healthy: bool
    cuda_smoke_passed: bool
    nccl_test_passed: bool
    n_minus_one_capacity_verified: bool
    checkpoint_restore_verified: bool
    local_data_risk_resolved: bool
    disruption_budget_verified: bool
    workload_identity_verified: bool
    telemetry_fresh: bool
    application_slo_verified: bool
    rollback_tested: bool
    change_approved: bool
    independent_review_approved: bool

def evaluate(r):
    if not isinstance(r, Capstone):
        raise TypeError("Capstone required")
    if any(not isinstance(s, str) or not s.strip()
           for s in (r.node, r.owner)):
        return {"accepted": False, "reason": "missing_ownership"}
    if r.platform not in {"eks", "aks", "bare-metal", "self-managed"}:
        return {"accepted": False, "reason": "invalid_platform"}
    if r.environment not in {"dev", "staging", "prod"}:
        return {"accepted": False, "reason": "invalid_environment"}
    if not isinstance(r.image_digest, str) or not DIGEST.fullmatch(r.image_digest):
        return {"accepted": False, "reason": "invalid_image_digest"}
    if (type(r.expected_gpus) is not int or type(r.allocatable_gpus) is not int
            or r.expected_gpus <= 0 or r.allocatable_gpus < 0):
        return {"accepted": False, "reason": "invalid_gpu_count"}
    if (not isinstance(r.gpu_uuids, tuple)
            or any(not isinstance(u, str) or not UUID.fullmatch(u) for u in r.gpu_uuids)):
        return {"accepted": False, "reason": "invalid_gpu_uuid"}
    if len(set(r.gpu_uuids)) != len(r.gpu_uuids):
        return {"accepted": False, "reason": "duplicate_gpu_uuid"}
    gates = {
        "gpu_inventory": len(r.gpu_uuids) == r.expected_gpus,
        "gpu_allocatable": r.allocatable_gpus == r.expected_gpus,
        "vendor_matrix": r.vendor_matrix_verified is True,
        "topology": r.topology_verified is True,
        "firmware": r.firmware_approved is True,
        "driver": r.driver_compatible is True,
        "driver_owner": r.single_driver_owner is True,
        "image_signature": r.image_signature_verified is True,
        "gpu_health": r.gpu_health_verified is True,
        "critical_xid": r.no_critical_xid is True,
        "uncorrectable_ecc": r.no_uncorrectable_ecc is True,
        "device_plugin": r.device_plugin_healthy is True,
        "cuda_smoke": r.cuda_smoke_passed is True,
        "nccl": r.nccl_test_passed is True,
        "n_minus_one": r.n_minus_one_capacity_verified is True,
        "checkpoint": r.checkpoint_restore_verified is True,
        "local_data": r.local_data_risk_resolved is True,
        "disruption_budget": r.disruption_budget_verified is True,
        "workload_identity": r.workload_identity_verified is True,
        "telemetry": r.telemetry_fresh is True,
        "application_slo": r.application_slo_verified is True,
        "rollback": r.rollback_tested is True,
        "change_approval": r.change_approved is True,
        "independent_review": r.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "node": r.node, "platform": r.platform,
            "gpu_count": len(r.gpu_uuids)}

def example():
    return Capstone(
        "gpu-staging-01", "gpu-platform-sre", "eks", "staging",
        "sha256:" + "a" * 64,
        ("GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
         "GPU-ffffffff-bbbb-cccc-dddd-eeeeeeeeeeee"),
        2, 2, *([True] * 22)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

**Critical scope limitation:** These are synthetic attestations. The evaluator does not check actual signatures, GPU UUID authenticity, NVIDIA support matrices, checkpoint durability, security authorization or SLOs. A true flag is **not evidence** without linked test results and reviewer approval.

## 3. Automated tests

Save as `test_gpu_capstone.py`:

```python
from dataclasses import replace
import pytest
from gpu_capstone import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_owner():
    assert evaluate(replace(example(), owner=" "))["reason"] == "missing_ownership"

def test_invalid_platform():
    assert evaluate(replace(example(), platform="unknown"))["reason"] == "invalid_platform"

def test_invalid_environment():
    assert evaluate(replace(example(), environment="unknown"))["reason"] == "invalid_environment"

def test_invalid_digest():
    assert evaluate(replace(example(), image_digest="latest"))["reason"] == "invalid_image_digest"

def test_bool_gpu_count():
    assert evaluate(replace(example(), expected_gpus=True))["reason"] == "invalid_gpu_count"

def test_negative_gpu_count():
    assert evaluate(replace(example(), allocatable_gpus=-1))["reason"] == "invalid_gpu_count"

def test_invalid_uuid():
    assert evaluate(replace(example(), gpu_uuids=("bad",)))["reason"] == "invalid_gpu_uuid"

def test_duplicate_uuid():
    g = example().gpu_uuids[0]
    assert evaluate(replace(example(), gpu_uuids=(g, g)))["reason"] == "duplicate_gpu_uuid"

def test_missing_gpu():
    g = example()
    assert "gpu_inventory" in evaluate(replace(g, gpu_uuids=g.gpu_uuids[:1]))["failed_gates"]

def test_allocatable_mismatch():
    assert "gpu_allocatable" in evaluate(
        replace(example(), allocatable_gpus=1))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("vendor_matrix_verified", "vendor_matrix"),
    ("topology_verified", "topology"),
    ("firmware_approved", "firmware"),
    ("driver_compatible", "driver"),
    ("single_driver_owner", "driver_owner"),
    ("image_signature_verified", "image_signature"),
    ("gpu_health_verified", "gpu_health"),
    ("no_critical_xid", "critical_xid"),
    ("no_uncorrectable_ecc", "uncorrectable_ecc"),
    ("device_plugin_healthy", "device_plugin"),
    ("cuda_smoke_passed", "cuda_smoke"),
    ("nccl_test_passed", "nccl"),
    ("n_minus_one_capacity_verified", "n_minus_one"),
    ("checkpoint_restore_verified", "checkpoint"),
    ("local_data_risk_resolved", "local_data"),
    ("disruption_budget_verified", "disruption_budget"),
    ("workload_identity_verified", "workload_identity"),
    ("telemetry_fresh", "telemetry"),
    ("application_slo_verified", "application_slo"),
    ("rollback_tested", "rollback"),
    ("change_approved", "change_approval"),
    ("independent_review_approved", "independent_review"),
])
def test_fail_closed_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_string_not_approval():
    assert "change_approval" in evaluate(
        replace(example(), change_approved="yes"))["failed_gates"]

def test_integer_not_signature():
    assert "image_signature" in evaluate(
        replace(example(), image_signature_verified=1))["failed_gates"]
```

Run:
```bash
python gpu_capstone.py
python -m pytest -q
```

**Expected:** 36 pytest cases (14 standalone and 22 parameterized). Runtime execution has not been performed as part of publication.

## 4. Offline game-day exercise

Change one field at a time using `dataclasses.replace` and confirm each gate fails. For example:

```python
from dataclasses import replace
from gpu_capstone import evaluate, example
print(evaluate(replace(example(), no_uncorrectable_ecc=False)))
print(evaluate(replace(example(), checkpoint_restore_verified=False)))
print(evaluate(replace(example(), telemetry_fresh=False)))
```

These are **safe synthetic simulations**, not fault injection against a real GPU.

## 5. Real staging acceptance (separate authorization required)
1. Establish asset/GPU UUID and PCIe/NUMA baseline.
2. Validate pinned driver, firmware, Secure Boot and single driver owner against vendor support evidence.
3. Verify DCGM/Xid/ECC monitoring, application SLO and checkpoint restore.
4. Confirm zone-specific N-1 capacity, quota, PDB and local PV protection.
5. Execute provider-supported canary node replacement with approved change and rollback.
6. Verify GPU allocatable, CUDA smoke, NCCL where relevant, data integrity and workload SLO.
7. Attach timestamps, sanitized logs, dashboards, change ticket, reviewer and owner signoff.

## 6. 100-point assessment
Architecture (20), GPU health (20), workload recovery (20), incidents/runbooks (20), executable evidence (20). **Pass >=90** and no critical gate failure. Synthetic tests alone do not establish production readiness.
