# Chapter 011 — Executable NVIDIA Driver Lifecycle and Compatibility Lab

**Status:** Published. Offline tests, vendor-matrix verification, staging upgrade and independent review pending.

## 1. Setup
Requires Python 3.11+ and pytest. On Linux:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
On PowerShell activate with `.venv\Scripts\Activate.ps1`.

## 2. Deterministic change-control evaluator
Save as `driver_lifecycle.py`:
```python
from dataclasses import dataclass
import json
import re

DIGEST = re.compile(r"^sha256:[0-9a-f]{64}$")

@dataclass(frozen=True)
class DriverChange:
    node_pool: str
    owner: str
    environment: str
    previous_image_digest: str
    candidate_image_digest: str
    expected_gpus: int
    observed_gpus: int
    single_driver_owner: bool
    vendor_matrix_verified: bool
    kernel_module_verified: bool
    secure_boot_validated: bool
    container_toolkit_verified: bool
    device_plugin_verified: bool
    cuda_smoke_passed: bool
    nccl_test_passed: bool
    checkpoint_verified: bool
    n_minus_one_capacity_verified: bool
    observability_healthy: bool
    security_approved: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(change):
    if not isinstance(change, DriverChange):
        raise TypeError("DriverChange required")
    if any(not isinstance(v, str) or not v.strip()
           for v in (change.node_pool, change.owner)):
        return {"accepted": False, "reason": "missing_ownership"}
    if change.environment not in {"dev", "staging", "prod"}:
        return {"accepted": False, "reason": "invalid_environment"}
    if (not isinstance(change.previous_image_digest, str)
            or not isinstance(change.candidate_image_digest, str)
            or not DIGEST.fullmatch(change.previous_image_digest)
            or not DIGEST.fullmatch(change.candidate_image_digest)):
        return {"accepted": False, "reason": "invalid_image_digest"}
    if change.previous_image_digest == change.candidate_image_digest:
        return {"accepted": False, "reason": "unchanged_image"}
    if (type(change.expected_gpus) is not int or type(change.observed_gpus) is not int
            or change.expected_gpus <= 0 or change.observed_gpus < 0):
        return {"accepted": False, "reason": "invalid_gpu_count"}
    gates = {
        "gpu_count": change.observed_gpus == change.expected_gpus,
        "driver_owner": change.single_driver_owner is True,
        "vendor_matrix": change.vendor_matrix_verified is True,
        "kernel_module": change.kernel_module_verified is True,
        "secure_boot": change.secure_boot_validated is True,
        "container_toolkit": change.container_toolkit_verified is True,
        "device_plugin": change.device_plugin_verified is True,
        "cuda_smoke": change.cuda_smoke_passed is True,
        "nccl": change.nccl_test_passed is True,
        "checkpoint": change.checkpoint_verified is True,
        "n_minus_one": change.n_minus_one_capacity_verified is True,
        "observability": change.observability_healthy is True,
        "security": change.security_approved is True,
        "change_approval": change.change_approved is True,
        "rollback": change.rollback_tested is True,
    }
    failed = sorted(k for k, passed in gates.items() if not passed)
    return {"accepted": not failed, "failed_gates": failed,
            "node_pool": change.node_pool, "environment": change.environment}

def example():
    return DriverChange(
        "gpu-staging", "gpu-platform-sre", "staging",
        "sha256:" + "a" * 64, "sha256:" + "b" * 64, 4, 4,
        True, True, True, True, True, True, True, True,
        True, True, True, True, True, True, True,
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

**Scope:** Every control is a synthetic assertion. This program does not install, compare or validate real NVIDIA drivers; it cannot independently prove vendor compatibility, cryptographic signatures, GPU health or checkpoint recovery. The compatibility gate must be backed by externally collected approved evidence.

## 3. Automated pytest suite
Save as `test_driver_lifecycle.py`:
```python
from dataclasses import replace
import pytest
from driver_lifecycle import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_owner():
    assert evaluate(replace(example(), owner=" "))["reason"] == "missing_ownership"

def test_invalid_environment():
    assert evaluate(replace(example(), environment="other"))["reason"] == "invalid_environment"

def test_invalid_digest():
    assert evaluate(replace(example(), candidate_image_digest="latest"))["reason"] == "invalid_image_digest"

def test_same_image():
    e = example()
    assert evaluate(replace(e, candidate_image_digest=e.previous_image_digest))["reason"] == "unchanged_image"

def test_boolean_gpu_rejected():
    assert evaluate(replace(example(), expected_gpus=True))["reason"] == "invalid_gpu_count"

def test_negative_gpu_rejected():
    assert evaluate(replace(example(), observed_gpus=-1))["reason"] == "invalid_gpu_count"

def test_missing_gpu():
    assert "gpu_count" in evaluate(replace(example(), observed_gpus=3))["failed_gates"]

def test_extra_gpu():
    assert "gpu_count" in evaluate(replace(example(), observed_gpus=5))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("single_driver_owner", "driver_owner"),
    ("vendor_matrix_verified", "vendor_matrix"),
    ("kernel_module_verified", "kernel_module"),
    ("secure_boot_validated", "secure_boot"),
    ("container_toolkit_verified", "container_toolkit"),
    ("device_plugin_verified", "device_plugin"),
    ("cuda_smoke_passed", "cuda_smoke"),
    ("nccl_test_passed", "nccl"),
    ("checkpoint_verified", "checkpoint"),
    ("n_minus_one_capacity_verified", "n_minus_one"),
    ("observability_healthy", "observability"),
    ("security_approved", "security"),
    ("change_approved", "change_approval"),
    ("rollback_tested", "rollback"),
])
def test_fail_closed_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_string_is_not_boolean():
    assert "vendor_matrix" in evaluate(
        replace(example(), vendor_matrix_verified="yes"))["failed_gates"]

def test_integer_is_not_boolean():
    assert "security" in evaluate(
        replace(example(), security_approved=1))["failed_gates"]
```
Run:
```bash
python driver_lifecycle.py
python -m pytest -q
```
Expected **26 tests** (12 standalone + 14 parameterized). These tests have not been executed.

## 4. Approved staging integration
1. Record exact GPU SKU, OS/kernel, installed driver, toolkit, GPU Operator and workload CUDA/NCCL versions.
2. Link vendor support documentation for each tested pair; record approval.
3. Build and sign candidate image; verify rollback image remains available.
4. Execute host driver and kernel module diagnostics, Secure Boot validation and GPU enumeration.
5. Run pinned GPU smoke workload, inference latency and multi-rank NCCL test where relevant.
6. Verify checkpoint restore, N-1 capacity, telemetry, tenant isolation and security controls.
7. Canary one GPU node in staging; exercise rollback, record UTC evidence and seek independent approval.

## 5. Final assessment
Architecture (20), compatibility (20), rollout safety (20), incidents/runbooks (20), lab/evidence (20). Pass >=90 with no hard-gate failure.
