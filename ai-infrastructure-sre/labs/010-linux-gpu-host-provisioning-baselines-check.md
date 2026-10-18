# Chapter 010 — Executable Linux GPU Host Provisioning and Baselines Lab

**Status:** Published. Offline test execution and live node validation pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Fail-closed host baseline evaluator
Save as `host_baseline.py`:
```python
from dataclasses import dataclass
import json
import re

DIGEST = re.compile(r"^sha256:[0-9a-f]{64}$")

@dataclass(frozen=True)
class Host:
    host_id: str
    owner: str
    environment: str
    image_digest: str
    expected_gpus: int
    detected_gpus: int
    kernel_approved: bool
    driver_compatible: bool
    runtime_compatible: bool
    gpu_plugin_healthy: bool
    os_patched: bool
    time_sync_healthy: bool
    network_verified: bool
    storage_verified: bool
    workload_identity_verified: bool
    telemetry_verified: bool
    gpu_smoke_test_passed: bool
    rollback_image_available: bool
    capacity_gate_passed: bool

def evaluate(h):
    if not isinstance(h, Host):
        raise TypeError("Host required")
    if any(type(v) is not str or not v.strip() for v in (h.host_id, h.owner)):
        return {"accepted": False, "reason": "missing_identity"}
    if h.environment not in {"dev", "staging", "prod"}:
        return {"accepted": False, "reason": "invalid_environment"}
    if not isinstance(h.image_digest, str) or not DIGEST.fullmatch(h.image_digest):
        return {"accepted": False, "reason": "invalid_image_digest"}
    if (type(h.expected_gpus) is not int or type(h.detected_gpus) is not int
            or h.expected_gpus <= 0 or h.detected_gpus < 0):
        return {"accepted": False, "reason": "invalid_gpu_counts"}
    gates = {
        "gpu_count": h.detected_gpus == h.expected_gpus,
        "kernel": h.kernel_approved is True,
        "driver": h.driver_compatible is True,
        "runtime": h.runtime_compatible is True,
        "device_plugin": h.gpu_plugin_healthy is True,
        "os_patches": h.os_patched is True,
        "time_sync": h.time_sync_healthy is True,
        "network": h.network_verified is True,
        "storage": h.storage_verified is True,
        "identity": h.workload_identity_verified is True,
        "telemetry": h.telemetry_verified is True,
        "smoke_test": h.gpu_smoke_test_passed is True,
        "rollback": h.rollback_image_available is True,
        "capacity": h.capacity_gate_passed is True,
    }
    failed = sorted(k for k, passed in gates.items() if not passed)
    return {"accepted": not failed, "failed_gates": failed,
            "host_id": h.host_id, "gpu_count": h.detected_gpus}

def example():
    return Host("gpu-node-staging-01", "gpu-platform-sre", "staging",
                "sha256:" + "a" * 64, 4, 4,
                True, True, True, True, True, True, True, True,
                True, True, True, True, True)

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
**Limitations:** Boolean gates are declarations, not independently verified facts. Digest syntax is not cryptographic signature verification. This lab does not run cloud-init, install drivers, reboot hosts or invoke Kubernetes.

## 2. Automated tests
Save as `test_host_baseline.py`:
```python
from dataclasses import replace
import pytest
from host_baseline import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_host_id():
    assert evaluate(replace(example(), host_id=""))["reason"] == "missing_identity"

def test_missing_owner():
    assert evaluate(replace(example(), owner=" "))["reason"] == "missing_identity"

def test_invalid_environment():
    assert evaluate(replace(example(), environment="unknown"))["reason"] == "invalid_environment"

def test_invalid_digest():
    assert evaluate(replace(example(), image_digest="latest"))["reason"] == "invalid_image_digest"

def test_negative_gpu():
    assert evaluate(replace(example(), detected_gpus=-1))["reason"] == "invalid_gpu_counts"

def test_boolean_gpu_rejected():
    assert evaluate(replace(example(), expected_gpus=True))["reason"] == "invalid_gpu_counts"

def test_missing_gpu():
    assert "gpu_count" in evaluate(replace(example(), detected_gpus=3))["failed_gates"]

def test_extra_gpu():
    assert "gpu_count" in evaluate(replace(example(), detected_gpus=5))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("kernel_approved", "kernel"),
    ("driver_compatible", "driver"),
    ("runtime_compatible", "runtime"),
    ("gpu_plugin_healthy", "device_plugin"),
    ("os_patched", "os_patches"),
    ("time_sync_healthy", "time_sync"),
    ("network_verified", "network"),
    ("storage_verified", "storage"),
    ("workload_identity_verified", "identity"),
    ("telemetry_verified", "telemetry"),
    ("gpu_smoke_test_passed", "smoke_test"),
    ("rollback_image_available", "rollback"),
    ("capacity_gate_passed", "capacity"),
])
def test_required_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_gate():
    assert "telemetry" in evaluate(replace(example(), telemetry_verified="yes"))["failed_gates"]
```
Run:
```bash
python host_baseline.py
python -m pytest -q
```
Expected: **24 pytest cases** (11 standalone + 13 parameterized). Tests have not been executed.

## 3. Approved staging acceptance
1. Build a signed, versioned GPU host image and capture the exact kernel/driver/runtime compatibility matrix.
2. Boot an authorized staging node and capture Linux topology, PCIe, NUMA and GPU inventory.
3. Validate container runtime, kubelet, device plugin and allocatable GPU count.
4. Run a pinned GPU compute smoke test, record health/temperature/Xid/ECC.
5. Validate network, dataset storage, checkpoint access and scoped workload identity.
6. Exercise a controlled reboot, approved drain and rollback with N-1 headroom.
7. Compare metrics and SLO before/after; attach UTC evidence, approvals and incident contacts.

## 4. 100-point rubric
Provisioning (20), discovery (20), security (20), incident and rollback (20), executable evidence (20). Pass >=90 with no critical safety gate.
