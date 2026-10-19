# Chapter 014 — Executable GPU Firmware, BIOS and Host Maintenance Lab

**Status:** Published. Offline tests and live maintenance validation pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Synthetic maintenance change gate
Save as `firmware_maintenance.py`:
```python
from dataclasses import dataclass
import json
import re

DIGEST = re.compile(r"^sha256:[0-9a-f]{64}$")

@dataclass(frozen=True)
class Change:
    node: str
    owner: str
    environment: str
    previous_artifact: str
    candidate_artifact: str
    expected_gpus: int
    observed_gpus: int
    vendor_matrix_verified: bool
    artifact_signature_verified: bool
    bmc_recovery_verified: bool
    bios_settings_backed_up: bool
    pcie_numa_baseline_verified: bool
    secure_boot_verified: bool
    driver_compatible: bool
    nic_storage_healthy: bool
    checkpoint_restore_verified: bool
    n_minus_one_capacity_verified: bool
    telemetry_healthy: bool
    workload_smoke_passed: bool
    security_approved: bool
    change_approved: bool
    recovery_plan_tested: bool

def evaluate(change):
    if not isinstance(change, Change):
        raise TypeError("Change required")
    if any(not isinstance(v, str) or not v.strip() for v in (change.node, change.owner)):
        return {"accepted": False, "reason": "missing_ownership"}
    if change.environment not in {"dev", "staging", "prod"}:
        return {"accepted": False, "reason": "invalid_environment"}
    if any(not isinstance(d, str) or not DIGEST.fullmatch(d)
           for d in (change.previous_artifact, change.candidate_artifact)):
        return {"accepted": False, "reason": "invalid_artifact_digest"}
    if change.previous_artifact == change.candidate_artifact:
        return {"accepted": False, "reason": "unchanged_artifact"}
    if (type(change.expected_gpus) is not int or type(change.observed_gpus) is not int
            or change.expected_gpus <= 0 or change.observed_gpus < 0):
        return {"accepted": False, "reason": "invalid_gpu_count"}
    gates = {
        "gpu_count": change.observed_gpus == change.expected_gpus,
        "vendor_matrix": change.vendor_matrix_verified is True,
        "artifact_signature": change.artifact_signature_verified is True,
        "bmc_recovery": change.bmc_recovery_verified is True,
        "bios_backup": change.bios_settings_backed_up is True,
        "topology_baseline": change.pcie_numa_baseline_verified is True,
        "secure_boot": change.secure_boot_verified is True,
        "driver": change.driver_compatible is True,
        "nic_storage": change.nic_storage_healthy is True,
        "checkpoint": change.checkpoint_restore_verified is True,
        "n_minus_one": change.n_minus_one_capacity_verified is True,
        "telemetry": change.telemetry_healthy is True,
        "workload_smoke": change.workload_smoke_passed is True,
        "security": change.security_approved is True,
        "change_approval": change.change_approved is True,
        "recovery_plan": change.recovery_plan_tested is True,
    }
    failed = sorted(k for k, passed in gates.items() if not passed)
    return {"accepted": not failed, "failed_gates": failed,
            "node": change.node, "environment": change.environment}

def example():
    return Change("gpu-staging-01", "gpu-hardware-sre", "staging",
                  "sha256:" + "a" * 64, "sha256:" + "b" * 64, 4, 4,
                  *([True] * 15))

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

**Important:** Every Boolean is an assertion supplied by the operator; the tool does not independently verify signatures, compatibility, BMC recovery, or approval. It performs no firmware changes. Vendor matrix and recovery evidence must be independently collected.

## 2. Automated regression tests
Save as `test_firmware_maintenance.py`:
```python
from dataclasses import replace
import pytest
from firmware_maintenance import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_owner():
    assert evaluate(replace(example(), owner=" "))["reason"] == "missing_ownership"

def test_bad_environment():
    assert evaluate(replace(example(), environment="unknown"))["reason"] == "invalid_environment"

def test_bad_digest():
    assert evaluate(replace(example(), candidate_artifact="latest"))["reason"] == "invalid_artifact_digest"

def test_unchanged():
    c = example()
    assert evaluate(replace(c, candidate_artifact=c.previous_artifact))["reason"] == "unchanged_artifact"

def test_negative_gpu():
    assert evaluate(replace(example(), observed_gpus=-1))["reason"] == "invalid_gpu_count"

def test_bool_gpu():
    assert evaluate(replace(example(), expected_gpus=True))["reason"] == "invalid_gpu_count"

def test_gpu_mismatch():
    assert "gpu_count" in evaluate(replace(example(), observed_gpus=3))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("vendor_matrix_verified", "vendor_matrix"),
    ("artifact_signature_verified", "artifact_signature"),
    ("bmc_recovery_verified", "bmc_recovery"),
    ("bios_settings_backed_up", "bios_backup"),
    ("pcie_numa_baseline_verified", "topology_baseline"),
    ("secure_boot_verified", "secure_boot"),
    ("driver_compatible", "driver"),
    ("nic_storage_healthy", "nic_storage"),
    ("checkpoint_restore_verified", "checkpoint"),
    ("n_minus_one_capacity_verified", "n_minus_one"),
    ("telemetry_healthy", "telemetry"),
    ("workload_smoke_passed", "workload_smoke"),
    ("security_approved", "security"),
    ("change_approved", "change_approval"),
    ("recovery_plan_tested", "recovery_plan"),
])
def test_fail_closed_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_approval():
    assert "change_approval" in evaluate(
        replace(example(), change_approved="yes"))["failed_gates"]

def test_nonboolean_signature():
    assert "artifact_signature" in evaluate(
        replace(example(), artifact_signature_verified=1))["failed_gates"]
```
Run:
```bash
python firmware_maintenance.py
python -m pytest -q
```
Expected **26 pytest cases** (11 standalone and 15 parameterized); tests have not been executed.

## 3. Approved staging exercises
1. Record vendor SKU, BIOS/BMC/GPU/NIC/NVMe firmware versions, PCIe topology and GPU UUID.
2. Verify signed candidate, compatibility matrix, change ticket and independent recovery access.
3. Simulate a missing BMC recovery path, unsigned artifact, GPU count mismatch and insufficient N-1 headroom **by changing input flags**, not by disrupting hardware.
4. In a separately approved maintenance window, rehearse vendor-specific upgrade and recovery on a staging server.
5. Validate POST, Linux boot, Secure Boot, GPU enumeration, CUDA smoke, NIC/RDMA, NVMe checkpoint restore and app SLO.
6. Attach sanitized UTC evidence, risk review, rollback procedure and owner approval.

## 4. 100-point rubric
Architecture (20), inventory/security (20), change/rollback (20), incidents/runbooks (20), executable evidence (20). Pass >=90 with no hard stop.
