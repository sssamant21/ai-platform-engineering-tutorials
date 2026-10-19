# Chapter 015 — Executable GPU Node Replacement and Recovery Lab

**Status:** Published. Synthetic tests and staging recovery validation pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Synthetic replacement acceptance evaluator
Save as `node_recovery.py`:
```python
from dataclasses import dataclass
import json
import re

DIGEST = re.compile(r"^sha256:[0-9a-f]{64}$")

@dataclass(frozen=True)
class Recovery:
    old_node: str
    new_node: str
    owner: str
    environment: str
    platform: str
    approved_image_digest: str
    expected_gpus: int
    detected_gpus: int
    fault_evidence_preserved: bool
    change_approved: bool
    n_minus_one_verified: bool
    checkpoint_restore_verified: bool
    local_data_risk_resolved: bool
    disruption_budget_verified: bool
    workload_eviction_verified: bool
    image_signature_verified: bool
    workload_identity_verified: bool
    driver_compatible: bool
    container_runtime_verified: bool
    device_plugin_healthy: bool
    gpu_health_verified: bool
    cuda_smoke_passed: bool
    telemetry_verified: bool
    application_slo_verified: bool
    rollback_plan_tested: bool

def evaluate(r):
    if not isinstance(r, Recovery):
        raise TypeError("Recovery required")
    if any(not isinstance(v, str) or not v.strip()
           for v in (r.old_node, r.new_node, r.owner)):
        return {"accepted": False, "reason": "missing_identity"}
    if r.old_node == r.new_node:
        return {"accepted": False, "reason": "node_not_replaced"}
    if r.environment not in {"dev", "staging", "prod"}:
        return {"accepted": False, "reason": "invalid_environment"}
    if r.platform not in {"eks", "aks", "bare-metal", "self-managed"}:
        return {"accepted": False, "reason": "invalid_platform"}
    if not isinstance(r.approved_image_digest, str) or not DIGEST.fullmatch(r.approved_image_digest):
        return {"accepted": False, "reason": "invalid_image"}
    if (type(r.expected_gpus) is not int or type(r.detected_gpus) is not int
            or r.expected_gpus <= 0 or r.detected_gpus < 0):
        return {"accepted": False, "reason": "invalid_gpu_count"}
    gates = {
        "gpu_count": r.detected_gpus == r.expected_gpus,
        "fault_evidence": r.fault_evidence_preserved is True,
        "change_approval": r.change_approved is True,
        "n_minus_one": r.n_minus_one_verified is True,
        "checkpoint_restore": r.checkpoint_restore_verified is True,
        "local_data": r.local_data_risk_resolved is True,
        "disruption_budget": r.disruption_budget_verified is True,
        "workload_eviction": r.workload_eviction_verified is True,
        "image_signature": r.image_signature_verified is True,
        "workload_identity": r.workload_identity_verified is True,
        "driver": r.driver_compatible is True,
        "container_runtime": r.container_runtime_verified is True,
        "device_plugin": r.device_plugin_healthy is True,
        "gpu_health": r.gpu_health_verified is True,
        "cuda_smoke": r.cuda_smoke_passed is True,
        "telemetry": r.telemetry_verified is True,
        "application_slo": r.application_slo_verified is True,
        "rollback": r.rollback_plan_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "old_node": r.old_node, "new_node": r.new_node}

def example():
    return Recovery("gpu-old", "gpu-new", "gpu-sre", "staging", "eks",
                    "sha256:" + "a" * 64, 4, 4, *([True] * 17))

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
**Safety:** Flags represent external attestations, not independently verified facts. This program performs **no** Kubernetes/cloud mutations, checkpoint operations, hardware diagnostics or cryptographic verification. It cannot authorize a production replacement.

## 2. pytest regression suite
Save as `test_node_recovery.py`:
```python
from dataclasses import replace
import pytest
from node_recovery import evaluate, example

def test_healthy():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_owner():
    assert evaluate(replace(example(), owner=" "))["reason"] == "missing_identity"

def test_same_node():
    r = example()
    assert evaluate(replace(r, new_node=r.old_node))["reason"] == "node_not_replaced"

def test_environment():
    assert evaluate(replace(example(), environment="invalid"))["reason"] == "invalid_environment"

def test_platform():
    assert evaluate(replace(example(), platform="unknown"))["reason"] == "invalid_platform"

def test_image():
    assert evaluate(replace(example(), approved_image_digest="latest"))["reason"] == "invalid_image"

def test_bool_gpu():
    assert evaluate(replace(example(), expected_gpus=True))["reason"] == "invalid_gpu_count"

def test_negative_gpu():
    assert evaluate(replace(example(), detected_gpus=-1))["reason"] == "invalid_gpu_count"

def test_missing_gpu():
    assert "gpu_count" in evaluate(replace(example(), detected_gpus=3))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("fault_evidence_preserved", "fault_evidence"),
    ("change_approved", "change_approval"),
    ("n_minus_one_verified", "n_minus_one"),
    ("checkpoint_restore_verified", "checkpoint_restore"),
    ("local_data_risk_resolved", "local_data"),
    ("disruption_budget_verified", "disruption_budget"),
    ("workload_eviction_verified", "workload_eviction"),
    ("image_signature_verified", "image_signature"),
    ("workload_identity_verified", "workload_identity"),
    ("driver_compatible", "driver"),
    ("container_runtime_verified", "container_runtime"),
    ("device_plugin_healthy", "device_plugin"),
    ("gpu_health_verified", "gpu_health"),
    ("cuda_smoke_passed", "cuda_smoke"),
    ("telemetry_verified", "telemetry"),
    ("application_slo_verified", "application_slo"),
    ("rollback_plan_tested", "rollback"),
])
def test_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_approval():
    assert "change_approval" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_nonboolean_checkpoint():
    assert "checkpoint_restore" in evaluate(replace(example(), checkpoint_restore_verified=1))["failed_gates"]
```
Run:
```bash
python node_recovery.py
python -m pytest -q
```
Expected **29 cases** (12 standalone + 17 parameterized); execution pending.

## 3. Approved staging acceptance
1. Capture current node/GPU inventory, Xid/ECC, kubelet and workload state.
2. Confirm checkpoint durability, local NVMe/PV dependencies, disruption budget and N-1 capacity.
3. Exercise **synthetic** data-loss, capacity, driver and identity gate failures with test fixtures.
4. Under separate change approval, perform provider-supported staging node replacement.
5. Validate signed image, GPU count/UUID, driver, runtime, plugin, CUDA smoke and training restore.
6. Verify application SLO, telemetry, alert routing, rollback, cost and asset reconciliation.
7. Attach sanitized UTC evidence, independent reviewer and owner signoff.

## 4. 100-point assessment
Architecture (20), data/capacity (20), platform operations (20), incidents/runbooks (20), lab/evidence (20). Pass >=90 with no hard-stop failure.
