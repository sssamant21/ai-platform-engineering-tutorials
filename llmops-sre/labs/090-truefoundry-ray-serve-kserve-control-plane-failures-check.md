# Chapter 090 — Executable TrueFoundry, Ray Serve and KServe Control Plane Failure Lab

**Status:** Published. Offline execution, staging drills and independent review pending.

## Prerequisites
Python 3.11+ and pytest. Use synthetic observations; no Kubernetes or platform credentials required.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```

PowerShell activation: `.venv\Scripts\Activate.ps1`.

## 1. Incident classifier and recovery gates
Save as `control_plane_incident.py`:

```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class Signals:
    api_available: bool = True
    admission_healthy: bool = True
    reconciler_healthy: bool = True
    ray_head_healthy: bool = True
    ray_serve_healthy: bool = True
    kserve_ready: bool = True
    gpu_allocatable: bool = True
    endpoint_ready: bool = True
    inference_healthy: bool = True

def classify(s):
    if not isinstance(s, Signals):
        raise TypeError("Signals required")
    if any(type(v) is not bool for v in vars(s).values()):
        raise ValueError("signals must be booleans")
    if not s.api_available:
        return "kubernetes_api_unavailable"
    if not s.admission_healthy:
        return "admission_webhook_failure"
    if not s.reconciler_healthy:
        return "reconciliation_failure"
    if not s.ray_head_healthy:
        return "ray_head_or_gcs_failure"
    if not s.ray_serve_healthy:
        return "ray_serve_failure"
    if not s.kserve_ready:
        return "kserve_not_ready"
    if not s.gpu_allocatable:
        return "gpu_scheduling_failure"
    if not s.endpoint_ready:
        return "routing_endpoint_failure"
    if not s.inference_healthy:
        return "inference_data_plane_failure"
    return "healthy"

@dataclass(frozen=True)
class Recovery:
    approved_revision: bool
    admission_policy_intact: bool
    tenant_isolation_verified: bool
    signed_artifact_verified: bool
    healthy_capacity: int
    required_capacity: int
    inference_probe_passed: bool
    slo_verified: bool
    owner_approved: bool

def can_promote(r):
    if not isinstance(r, Recovery):
        raise TypeError("Recovery required")
    counts = (r.healthy_capacity, r.required_capacity)
    if any(type(v) is not int or v < 0 for v in counts):
        return False
    checks = (
        r.approved_revision, r.admission_policy_intact,
        r.tenant_isolation_verified, r.signed_artifact_verified,
        r.inference_probe_passed, r.slo_verified, r.owner_approved,
    )
    return all(type(v) is bool and v for v in checks) and r.healthy_capacity >= r.required_capacity

if __name__ == "__main__":
    signals = Signals(admission_healthy=False)
    recovery = Recovery(True, True, True, True, 4, 3, True, True, True)
    print(json.dumps({"fault": classify(signals), "promotion_allowed": can_promote(recovery)}, indent=2))
```

The classifier deliberately prioritizes one fault at a time. Multiple simultaneous faults require separate evidence and operator investigation. A `True` value is an operator assertion, **not verified evidence**.

## 2. Pytest suite
Save as `test_control_plane_incident.py`:

```python
from dataclasses import replace
import pytest
from control_plane_incident import Signals, Recovery, classify, can_promote

@pytest.mark.parametrize("change,expected", [
    ({"api_available": False}, "kubernetes_api_unavailable"),
    ({"admission_healthy": False}, "admission_webhook_failure"),
    ({"reconciler_healthy": False}, "reconciliation_failure"),
    ({"ray_head_healthy": False}, "ray_head_or_gcs_failure"),
    ({"ray_serve_healthy": False}, "ray_serve_failure"),
    ({"kserve_ready": False}, "kserve_not_ready"),
    ({"gpu_allocatable": False}, "gpu_scheduling_failure"),
    ({"endpoint_ready": False}, "routing_endpoint_failure"),
    ({"inference_healthy": False}, "inference_data_plane_failure"),
])
def test_faults(change, expected):
    assert classify(replace(Signals(), **change)) == expected

def good():
    return Recovery(True, True, True, True, 4, 3, True, True, True)

def test_healthy():
    assert classify(Signals()) == "healthy"

def test_recovery_allowed():
    assert can_promote(good())

@pytest.mark.parametrize("field", [
    "approved_revision", "admission_policy_intact",
    "tenant_isolation_verified", "signed_artifact_verified",
    "inference_probe_passed", "slo_verified", "owner_approved",
])
def test_failed_gate_blocks(field):
    assert not can_promote(replace(good(), **{field: False}))

def test_capacity_shortfall():
    assert not can_promote(replace(good(), healthy_capacity=2))

def test_invalid_capacity():
    assert not can_promote(replace(good(), healthy_capacity=-1))

def test_invalid_signal_type():
    with pytest.raises(ValueError):
        classify(replace(Signals(), api_available="yes"))

def test_invalid_recovery_type():
    with pytest.raises(TypeError):
        can_promote({})
```

Run:

```bash
python control_plane_incident.py
python -m pytest -q
```

Expected: **22 pytest cases** (9 platform-fault parameterizations, 7 recovery-gate parameterizations and 6 standalone tests). Execution has not been performed.

## 3. Staging extensions
1. Correlate TrueFoundry deployment status with Kubernetes resources and logs.
2. Reproduce a Ray Serve controller/proxy failure in an isolated cluster.
3. Verify KServe admission failure and InferenceService condition recovery.
4. Inject a synthetic GPU scheduling failure; test healthy capacity and PDB constraints.
5. Test a revoked model artifact and assert deployment promotion remains blocked.
6. Confirm a broken endpoint returns to service only after synthetic inference and SLO recovery.
7. Capture UTC timeline, immutable revisions, reviewer approvals and rollback evidence.
8. Confirm no bypass of tenant isolation, model signature or admission policies.

## 4. Assessment
| Area | Points |
|---|---:|
| Failure identification | 20 |
| Platform-specific diagnosis | 20 |
| Safe remediation | 20 |
| Recovery gates and SLOs | 20 |
| Tests and audit evidence | 20 |
| **Total** | **100** |

Educational pass: >=90/100 and no critical security/policy bypass. Real control-plane validation and independent review remain pending.
