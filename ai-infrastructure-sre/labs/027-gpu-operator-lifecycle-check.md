# Lab 027 — GPU Operator Lifecycle and Upgrade Gate

**Python 3.11+ · pytest · offline synthetic model.** No cluster credentials, Helm install, GPU access, driver modifications, host reboots or destructive operations.

## Goal

Fail closed on conflicting component ownership, missing Operator reconciliation, incompatible driver/toolkit/plugin versions, unhealthy operands, stale telemetry, unsafe MIG changes, missing checkpoint/rollback, or absent production approval. A passing synthetic gate does **not** prove GPU Operator compatibility or live CUDA functionality.

## File: `gpu_operator_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Upgrade:
    cluster: str
    operator_version: str
    target_operator_version: str
    driver_management: str
    toolkit_management: str
    device_plugin_management: str
    duplicate_device_plugin: bool
    compatibility_matrix_verified: bool
    clusterpolicy_reconciled: bool
    operator_ready: bool
    driver_ready: bool
    toolkit_ready: bool
    device_plugin_ready: bool
    gpu_resources_verified: bool
    mig_strategy_verified: bool
    cuda_canary_passed: bool
    dcgm_telemetry_fresh: bool
    gpu_health_verified: bool
    checkpoint_verified: bool
    n_minus_one_verified: bool
    security_review_approved: bool
    change_approved: bool
    rollback_tested: bool
    host_driver_rollback_defined: bool
    application_slo_verified: bool
    telemetry_age_s: float
    max_telemetry_age_s: float

def evaluate(u):
    if not isinstance(u, Upgrade):
        raise TypeError("Upgrade required")
    if any(not isinstance(v, str) or not v.strip()
           for v in (u.cluster, u.operator_version, u.target_operator_version)):
        return {"accepted": False, "reason": "missing_identity"}
    valid_owners = ("operator", "external")
    if any(v not in valid_owners for v in (
        u.driver_management, u.toolkit_management, u.device_plugin_management
    )):
        return {"accepted": False, "reason": "invalid_owner"}
    if (type(u.telemetry_age_s) not in (int, float)
            or type(u.max_telemetry_age_s) not in (int, float)
            or not math.isfinite(u.telemetry_age_s)
            or not math.isfinite(u.max_telemetry_age_s)
            or u.telemetry_age_s < 0 or u.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_telemetry"}
    gates = {
        "single_plugin_owner": u.duplicate_device_plugin is False,
        "compatibility": u.compatibility_matrix_verified is True,
        "reconciliation": u.clusterpolicy_reconciled is True,
        "operator": u.operator_ready is True,
        "driver": u.driver_ready is True,
        "toolkit": u.toolkit_ready is True,
        "device_plugin": u.device_plugin_ready is True,
        "resource_discovery": u.gpu_resources_verified is True,
        "mig_strategy": u.mig_strategy_verified is True,
        "cuda_canary": u.cuda_canary_passed is True,
        "dcgm": u.dcgm_telemetry_fresh is True,
        "gpu_health": u.gpu_health_verified is True,
        "checkpoint": u.checkpoint_verified is True,
        "n_minus_one": u.n_minus_one_verified is True,
        "security": u.security_review_approved is True,
        "change": u.change_approved is True,
        "rollback": u.rollback_tested is True,
        "host_rollback": u.host_driver_rollback_defined is True,
        "slo": u.application_slo_verified is True,
        "telemetry": u.telemetry_age_s <= u.max_telemetry_age_s,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {
        "accepted": not failed,
        "failed_gates": failed,
        "cluster": u.cluster,
        "from_version": u.operator_version,
        "to_version": u.target_operator_version,
        "driver_owner": u.driver_management,
        "device_plugin_owner": u.device_plugin_management,
    }

def example():
    return Upgrade(
        "synthetic-gpu-staging", "v-old", "v-reviewed",
        "operator", "operator", "operator", False,
        *([True] * 18),
        10.0, 120.0
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_gpu_operator_policy.py`

```python
from dataclasses import replace
import pytest
from gpu_operator_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_identity"

def test_missing_target_version():
    assert evaluate(replace(example(), target_operator_version=" "))["reason"] == "missing_identity"

def test_invalid_driver_owner():
    assert evaluate(replace(example(), driver_management="both"))["reason"] == "invalid_owner"

def test_external_driver_allowed():
    assert evaluate(replace(example(), driver_management="external"))["accepted"]

def test_external_plugin_allowed():
    assert evaluate(replace(example(), device_plugin_management="external"))["accepted"]

def test_duplicate_plugin():
    assert "single_plugin_owner" in evaluate(replace(example(), duplicate_device_plugin=True))["failed_gates"]

def test_nonboolean_duplicate():
    assert "single_plugin_owner" in evaluate(replace(example(), duplicate_device_plugin="no"))["failed_gates"]

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nan_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=float("nan")))["reason"] == "invalid_telemetry"

def test_negative_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=-1))["reason"] == "invalid_telemetry"

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("compatibility_matrix_verified", "compatibility"),
    ("clusterpolicy_reconciled", "reconciliation"),
    ("operator_ready", "operator"),
    ("driver_ready", "driver"),
    ("toolkit_ready", "toolkit"),
    ("device_plugin_ready", "device_plugin"),
    ("gpu_resources_verified", "resource_discovery"),
    ("mig_strategy_verified", "mig_strategy"),
    ("cuda_canary_passed", "cuda_canary"),
    ("dcgm_telemetry_fresh", "dcgm"),
    ("gpu_health_verified", "gpu_health"),
    ("checkpoint_verified", "checkpoint"),
    ("n_minus_one_verified", "n_minus_one"),
    ("security_review_approved", "security"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("host_driver_rollback_defined", "host_rollback"),
    ("application_slo_verified", "slo"),
])
def test_hard_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python gpu_operator_policy.py
python -m pytest -q
```

**Expected: 31 pytest cases** (13 standalone and 18 parameterized). Runtime execution remains a separate validation milestone.

## Ten synthetic incident drills

1. Operator cannot reconcile ClusterPolicy.
2. Driver DaemonSet fails after node kernel update.
3. External driver ownership conflicts with planned rollout.
4. Duplicate device-plugin installation.
5. Container Toolkit/CDI mismatch.
6. MIG strategy and resource keys drift.
7. CUDA canary fails despite healthy operator.
8. DCGM telemetry is stale.
9. Checkpoint/N-1 capacity is unverified.
10. Helm rollback exists but host driver rollback is undefined.

## Staging acceptance

1. Inventory actual chart/operator and operand versions, node OS, driver/toolkit, runtime/CDI and GPU SKU.
2. Confirm version-specific NVIDIA platform support and cloud managed-driver ownership.
3. Render Helm manifests, inspect privileged DaemonSets, RBAC, host mounts, image digests and security controls.
4. Canary a dedicated staging GPU node pool with approved change and rollback.
5. Confirm ClusterPolicy reconciliation, DaemonSet readiness, resource keys and CUDA canary.
6. Validate MIG, DCGM/Xid/ECC, NCCL where needed, application correctness and SLO.
7. Verify chart rollback **and separate host driver/node-image recovery**.
8. Record audit evidence, RCA/COA, ownership and production acceptance.

**Limitation:** The synthetic gate verifies declared attestations only; it does not independently discover actual Kubernetes or NVIDIA state.

## 100-point rubric

Operator architecture 20; version/driver lifecycle 20; runtime/MIG/observability 20; incident recovery 20; executable tests/staging 20. Pass >=90 with no critical security, GPU resource or rollback failure.
