# Lab 028 — GPU Scheduling, Taints and Affinity Admission Gate

**Python 3.11+ · pytest · offline synthetic scheduling policy.** No Kubernetes API calls, node changes or GPU allocation.

## Policy model

Check node/pod GPU resource key, allocatable capacity, taint/toleration compatibility, required node labels, zone, GPU device-plugin readiness, MIG profile, N-1 capacity, tenant isolation, telemetry, checkpoint, security, and rollback. The model intentionally does not replace Kubernetes scheduler plugins or prove real device health.

## File: `gpu_scheduler_policy.py`

```python
from dataclasses import dataclass
import json
import math
import re

RESOURCE = re.compile(r"^nvidia\.com/(?:gpu|mig-[A-Za-z0-9.-]+)$")

@dataclass(frozen=True)
class Scheduling:
    pod: str
    node: str
    resource_key: str
    node_resource_key: str
    requested: int
    allocatable: int
    already_reserved: int
    required_pool: str
    node_pool: str
    required_zone: str
    node_zone: str
    taints: tuple[str, ...]
    tolerations: tuple[str, ...]
    device_plugin_ready: bool
    mig_profile_verified: bool
    topology_verified: bool
    n_minus_one_verified: bool
    tenant_isolation_verified: bool
    checkpoint_verified: bool
    telemetry_age_s: float
    max_telemetry_age_s: float
    cuda_canary_passed: bool
    change_approved: bool
    rollback_tested: bool
    slo_healthy: bool

def evaluate(s):
    if not isinstance(s, Scheduling):
        raise TypeError("Scheduling required")
    if any(not isinstance(x, str) or not x.strip() for x in
           (s.pod, s.node, s.required_pool, s.node_pool, s.required_zone, s.node_zone)):
        return {"accepted": False, "reason": "missing_identity"}
    if (not isinstance(s.resource_key, str) or not RESOURCE.fullmatch(s.resource_key)
            or not isinstance(s.node_resource_key, str) or not RESOURCE.fullmatch(s.node_resource_key)):
        return {"accepted": False, "reason": "invalid_resource_key"}
    if (any(type(v) is not int for v in (s.requested, s.allocatable, s.already_reserved))
            or s.requested <= 0 or s.allocatable < 0 or s.already_reserved < 0
            or s.already_reserved > s.allocatable):
        return {"accepted": False, "reason": "invalid_capacity"}
    if (not isinstance(s.taints, tuple) or not isinstance(s.tolerations, tuple)
            or any(not isinstance(t, str) or not t for t in s.taints + s.tolerations)):
        return {"accepted": False, "reason": "invalid_taints"}
    if (any(type(v) not in (int, float) or not math.isfinite(v)
            for v in (s.telemetry_age_s, s.max_telemetry_age_s))
            or s.telemetry_age_s < 0 or s.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_telemetry"}
    gates = {
        "resource_type": s.resource_key == s.node_resource_key,
        "capacity": s.requested <= s.allocatable - s.already_reserved,
        "pool": s.required_pool == s.node_pool,
        "zone": s.required_zone == s.node_zone,
        "taints": set(s.taints).issubset(set(s.tolerations)),
        "device_plugin": s.device_plugin_ready is True,
        "mig_profile": s.mig_profile_verified is True,
        "topology": s.topology_verified is True,
        "n_minus_one": s.n_minus_one_verified is True,
        "tenant_isolation": s.tenant_isolation_verified is True,
        "checkpoint": s.checkpoint_verified is True,
        "telemetry": s.telemetry_age_s <= s.max_telemetry_age_s,
        "cuda_canary": s.cuda_canary_passed is True,
        "change": s.change_approved is True,
        "rollback": s.rollback_tested is True,
        "slo": s.slo_healthy is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "available_gpu_units": s.allocatable - s.already_reserved,
            "resource_key": s.resource_key}

def example():
    return Scheduling(
        "synthetic-training", "synthetic-gpu-node",
        "nvidia.com/gpu", "nvidia.com/gpu", 1, 4, 2,
        "staging-gpu", "staging-gpu", "zone-a", "zone-a",
        ("dedicated=gpu:NoSchedule",), ("dedicated=gpu:NoSchedule",),
        True, True, True, True, True, True, 10.0, 120.0,
        True, True, True, True
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

The taint/toleration gate uses **exact string matching as a deliberate simplified synthetic model**. Kubernetes supports key/effect/operator/value matching and `NoExecute` tolerationSeconds; live validation must use the real scheduler semantics.

## File: `test_gpu_scheduler_policy.py`

```python
from dataclasses import replace
import pytest
from gpu_scheduler_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_pod():
    assert evaluate(replace(example(), pod=" "))["reason"] == "missing_identity"

def test_invalid_resource():
    assert evaluate(replace(example(), resource_key="gpu"))["reason"] == "invalid_resource_key"

def test_fractional_gpu_rejected():
    assert evaluate(replace(example(), requested=0.5))["reason"] == "invalid_capacity"

def test_boolean_gpu_rejected():
    assert evaluate(replace(example(), requested=True))["reason"] == "invalid_capacity"

def test_negative_capacity():
    assert evaluate(replace(example(), allocatable=-1))["reason"] == "invalid_capacity"

def test_overreserved():
    assert evaluate(replace(example(), already_reserved=5))["reason"] == "invalid_capacity"

def test_insufficient_capacity():
    assert "capacity" in evaluate(replace(example(), requested=3))["failed_gates"]

def test_wrong_resource_key():
    assert "resource_type" in evaluate(replace(example(), node_resource_key="nvidia.com/mig-1g.10gb"))["failed_gates"]

def test_wrong_pool():
    assert "pool" in evaluate(replace(example(), node_pool="cpu"))["failed_gates"]

def test_wrong_zone():
    assert "zone" in evaluate(replace(example(), node_zone="zone-b"))["failed_gates"]

def test_missing_toleration():
    assert "taints" in evaluate(replace(example(), tolerations=()))["failed_gates"]

def test_invalid_taints():
    assert evaluate(replace(example(), taints=("x", 1)))["reason"] == "invalid_taints"

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nan_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=float("nan")))["reason"] == "invalid_telemetry"

def test_nonboolean_change():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("device_plugin_ready", "device_plugin"),
    ("mig_profile_verified", "mig_profile"),
    ("topology_verified", "topology"),
    ("n_minus_one_verified", "n_minus_one"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("checkpoint_verified", "checkpoint"),
    ("cuda_canary_passed", "cuda_canary"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("slo_healthy", "slo"),
])
def test_hard_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python gpu_scheduler_policy.py
python -m pytest -q
```

**Expected 27 cases:** 17 standalone and 10 parameterized. Test execution has not been performed as part of publication.

## Eight offline incident drills

1. Missing GPU toleration.
2. Required affinity selects wrong node pool.
3. GPU quota/capacity insufficient.
4. Wrong MIG resource key.
5. Wrong availability zone.
6. Device plugin not ready.
7. Stale telemetry or missing checkpoint.
8. Tenant isolation, N-1, approval or rollback gate fails.

## Live staging acceptance

1. Inventory actual GPU node labels, taints, allocatable resource keys, GPU Operator/plugin and MIG strategy.
2. Validate ResourceQuota, PriorityClass, PDB and topology spread constraints.
3. Render approved manifests and run server-side dry-run with cluster authorization; note dry-run does **not** guarantee scheduling.
4. Deploy a pinned-image disposable canary in a staging namespace and inspect scheduler events.
5. Confirm pod binds to expected GPU node, CUDA sees the intended GPU, and workload correctness/SLO are healthy.
6. Exercise a controlled Pending case and verify diagnostic signal without modifying production.
7. Confirm N-1 capacity, checkpoint/rollback, security review and owner signoff.

## 100-point assessment

Resource semantics 20; taints/affinity 20; MIG/topology/tenant policy 20; incident runbooks 20; tests and staging evidence 20. Pass >=90 with no critical security, scheduling or recovery blocker.
