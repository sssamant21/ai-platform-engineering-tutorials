# Lab 029 — GPU Resource Quotas and Multitenancy Admission Gate

**Python 3.11+ / pytest / offline synthetic policy.** No cluster access, GPU allocation, RBAC changes, or privileged operations.

## Goal

Fail closed on quota/capacity violations, mismatched GPU/MIG resource keys, invalid integer requests, shared-GPU isolation policy, stale telemetry, tenant security, N-1 capacity, checkpoint, rollback and change approval. This model is a **simplified policy**; Kubernetes ResourceQuota is admission accounting and not physical GPU reservation.

## File: `tenant_gpu_policy.py`

```python
from dataclasses import dataclass
import json
import math
import re

RESOURCE = re.compile(r"^nvidia\.com/(?:gpu|mig-[A-Za-z0-9.-]+)$")

@dataclass(frozen=True)
class Tenant:
    namespace: str
    resource_key: str
    node_resource_key: str
    quota_hard: int
    quota_used: int
    requested: int
    pool_allocatable: int
    pool_reserved: int
    share_mode: str
    mig_profile_verified: bool
    sharing_approved: bool
    pool_isolation_verified: bool
    rbac_verified: bool
    pod_security_verified: bool
    network_policy_verified: bool
    metrics_access_verified: bool
    n_minus_one_verified: bool
    checkpoint_verified: bool
    telemetry_age_s: float
    max_telemetry_age_s: float
    gpu_health_verified: bool
    slo_healthy: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(t):
    if not isinstance(t, Tenant):
        raise TypeError("Tenant required")
    if not isinstance(t.namespace, str) or not t.namespace.strip():
        return {"accepted": False, "reason": "missing_namespace"}
    if (not isinstance(t.resource_key, str) or not RESOURCE.fullmatch(t.resource_key)
            or not isinstance(t.node_resource_key, str) or not RESOURCE.fullmatch(t.node_resource_key)):
        return {"accepted": False, "reason": "invalid_resource"}
    counts = (t.quota_hard, t.quota_used, t.requested, t.pool_allocatable, t.pool_reserved)
    if (any(type(x) is not int for x in counts)
            or t.quota_hard < 0 or t.quota_used < 0 or t.requested <= 0
            or t.pool_allocatable < 0 or t.pool_reserved < 0
            or t.quota_used > t.quota_hard or t.pool_reserved > t.pool_allocatable):
        return {"accepted": False, "reason": "invalid_count"}
    if t.share_mode not in ("full", "mig", "time-slicing", "mps"):
        return {"accepted": False, "reason": "invalid_share_mode"}
    if (any(type(x) not in (int, float) or not math.isfinite(x)
            for x in (t.telemetry_age_s, t.max_telemetry_age_s))
            or t.telemetry_age_s < 0 or t.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_telemetry"}
    gates = {
        "resource_match": t.resource_key == t.node_resource_key,
        "quota": t.quota_used + t.requested <= t.quota_hard,
        "pool_capacity": t.pool_reserved + t.requested <= t.pool_allocatable,
        "mig_profile": t.share_mode != "mig" or t.mig_profile_verified is True,
        "sharing": t.share_mode in ("full", "mig") or t.sharing_approved is True,
        "pool_isolation": t.pool_isolation_verified is True,
        "rbac": t.rbac_verified is True,
        "pod_security": t.pod_security_verified is True,
        "network_policy": t.network_policy_verified is True,
        "metrics_access": t.metrics_access_verified is True,
        "n_minus_one": t.n_minus_one_verified is True,
        "checkpoint": t.checkpoint_verified is True,
        "telemetry": t.telemetry_age_s <= t.max_telemetry_age_s,
        "gpu_health": t.gpu_health_verified is True,
        "slo": t.slo_healthy is True,
        "change": t.change_approved is True,
        "rollback": t.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "namespace": t.namespace,
            "quota_remaining_before_request": t.quota_hard - t.quota_used,
            "pool_remaining_before_request": t.pool_allocatable - t.pool_reserved}

def example():
    return Tenant(
        "synthetic-tenant-a", "nvidia.com/gpu", "nvidia.com/gpu",
        4, 1, 1, 8, 3, "full",
        *([True] * 9), 10.0, 120.0, *([True] * 4)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_tenant_gpu_policy.py`

```python
from dataclasses import replace
import pytest
from tenant_gpu_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_namespace():
    assert evaluate(replace(example(), namespace=" "))["reason"] == "missing_namespace"

def test_resource_key():
    assert evaluate(replace(example(), resource_key="gpu"))["reason"] == "invalid_resource"

def test_fractional_request():
    assert evaluate(replace(example(), requested=0.5))["reason"] == "invalid_count"

def test_boolean_request():
    assert evaluate(replace(example(), requested=True))["reason"] == "invalid_count"

def test_overused_quota():
    assert evaluate(replace(example(), quota_used=5))["reason"] == "invalid_count"

def test_quota_exceeded():
    assert "quota" in evaluate(replace(example(), requested=4))["failed_gates"]

def test_pool_exceeded():
    assert "pool_capacity" in evaluate(replace(example(), requested=6))["failed_gates"]

def test_wrong_resource_type():
    assert "resource_match" in evaluate(replace(example(), node_resource_key="nvidia.com/mig-1g.10gb"))["failed_gates"]

def test_unknown_share_mode():
    assert evaluate(replace(example(), share_mode="unsafe"))["reason"] == "invalid_share_mode"

def test_mig_requires_profile():
    assert "mig_profile" in evaluate(replace(example(), share_mode="mig", mig_profile_verified=False))["failed_gates"]

def test_shared_requires_approval():
    assert "sharing" in evaluate(replace(example(), share_mode="time-slicing", sharing_approved=False))["failed_gates"]

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nan_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=float("nan")))["reason"] == "invalid_telemetry"

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("pool_isolation_verified", "pool_isolation"),
    ("rbac_verified", "rbac"),
    ("pod_security_verified", "pod_security"),
    ("network_policy_verified", "network_policy"),
    ("metrics_access_verified", "metrics_access"),
    ("n_minus_one_verified", "n_minus_one"),
    ("checkpoint_verified", "checkpoint"),
    ("gpu_health_verified", "gpu_health"),
    ("slo_healthy", "slo"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_hard_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run locally:

```bash
python -m pip install pytest
python tenant_gpu_policy.py
python -m pytest -q
```

**Expected: 27 pytest cases** (16 standalone, 11 parameterized). Execution has not been performed as part of publication.

## Ten synthetic incident drills

1. Namespace quota exceeded.
2. Pool allocatable insufficient despite quota.
3. Wrong MIG resource key.
4. Shared GPU policy not approved.
5. Tenant placement isolation fails.
6. RBAC/Pod Security/NetworkPolicy evidence missing.
7. Cross-tenant metric access not reviewed.
8. N-1 or checkpoint readiness missing.
9. GPU health/telemetry/SLO degraded.
10. Change or rollback evidence absent.

## Staging acceptance

Inventory actual resource keys, quota used/hard, GPU pool reservations, node labels, device-plugin strategy and tenant access. Deploy an authorized disposable canary; verify a valid GPU request succeeds, quota-exceeding request is rejected by admission, within-quota unschedulable request remains Pending, and tenant cannot bypass protected node pools. Verify RBAC, PSA, network isolation, metrics access, N-1 and rollback with security approval.

**Limitation:** Synthetic assertions do not validate Kubernetes ResourceQuota accounting, MIG hardware isolation or multi-tenant security.

## 100-point rubric

Quota semantics 20; sharing/MIG 20; tenant security 20; incidents/FinOps 20; automated/staging evidence 20. Pass >=90 with no critical security or capacity blocker.
