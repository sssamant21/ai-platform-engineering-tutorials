# Lab 032 — GPU Workload Rollouts and Disruptions Acceptance Gate

**Python 3.11+ / pytest / offline synthetic policy.** No Kubernetes API, cloud credentials, GPU allocation or disruptive commands.

## Goal

Fail closed on insufficient GPU rollout surge, PDB/disruption risk, missing checkpoint/restore, incompatible model artifacts, stale telemetry, failed canary, unsafe node drain, missing N-1, missing rollback or unacceptable RPO/RTO.

## File: `gpu_rollout_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Rollout:
    namespace: str
    controller: str
    desired_replicas: int
    ready_replicas: int
    gpu_per_replica: int
    free_gpu_units: int
    max_surge: int
    max_unavailable: int
    pdb_min_available: int
    available_after_failure: int
    rpo_minutes: float
    max_rpo_minutes: float
    rto_minutes: float
    max_rto_minutes: float
    checkpoint_verified: bool
    restore_tested: bool
    artifact_compatible: bool
    immutable_image_verified: bool
    readiness_validated: bool
    canary_slo_verified: bool
    device_plugin_ready: bool
    cuda_verified: bool
    gpu_health_verified: bool
    n_minus_one_verified: bool
    disruption_budget_verified: bool
    tenant_security_verified: bool
    telemetry_age_s: float
    max_telemetry_age_s: float
    change_approved: bool
    rollback_tested: bool

def evaluate(r):
    if not isinstance(r, Rollout):
        raise TypeError("Rollout required")
    if not isinstance(r.namespace, str) or not r.namespace.strip():
        return {"accepted": False, "reason": "missing_namespace"}
    if r.controller not in ("Deployment", "StatefulSet", "Job", "TrainingOperator"):
        return {"accepted": False, "reason": "invalid_controller"}
    counts = (r.desired_replicas, r.ready_replicas, r.gpu_per_replica,
              r.free_gpu_units, r.max_surge, r.max_unavailable,
              r.pdb_min_available, r.available_after_failure)
    if (any(type(x) is not int or x < 0 for x in counts)
            or r.desired_replicas < 1 or r.gpu_per_replica < 1
            or r.ready_replicas > r.desired_replicas
            or r.max_unavailable > r.desired_replicas
            or r.pdb_min_available > r.desired_replicas):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (r.rpo_minutes, r.max_rpo_minutes, r.rto_minutes,
               r.max_rto_minutes, r.telemetry_age_s, r.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or r.max_rpo_minutes <= 0 or r.max_rto_minutes <= 0
            or r.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "replica_readiness": r.ready_replicas == r.desired_replicas,
        "surge_capacity": r.free_gpu_units >= r.max_surge * r.gpu_per_replica,
        "pdb": r.available_after_failure >= r.pdb_min_available,
        "rpo": r.rpo_minutes <= r.max_rpo_minutes,
        "rto": r.rto_minutes <= r.max_rto_minutes,
        "checkpoint": r.checkpoint_verified is True,
        "restore": r.restore_tested is True,
        "artifact": r.artifact_compatible is True,
        "image": r.immutable_image_verified is True,
        "readiness": r.readiness_validated is True,
        "canary_slo": r.canary_slo_verified is True,
        "device_plugin": r.device_plugin_ready is True,
        "cuda": r.cuda_verified is True,
        "gpu_health": r.gpu_health_verified is True,
        "n_minus_one": r.n_minus_one_verified is True,
        "disruption": r.disruption_budget_verified is True,
        "security": r.tenant_security_verified is True,
        "telemetry": r.telemetry_age_s <= r.max_telemetry_age_s,
        "change": r.change_approved is True,
        "rollback": r.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "namespace": r.namespace, "controller": r.controller,
            "required_surge_gpu_units": r.max_surge * r.gpu_per_replica}

def example():
    return Rollout(
        "synthetic-gpu-staging", "Deployment",
        2, 2, 1, 2, 1, 0, 1, 1,
        5.0, 15.0, 10.0, 30.0,
        *([True] * 13), 10.0, 120.0, True, True
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_gpu_rollout_policy.py`

```python
from dataclasses import replace
import pytest
from gpu_rollout_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_namespace():
    assert evaluate(replace(example(), namespace=" "))["reason"] == "missing_namespace"

def test_bad_controller():
    assert evaluate(replace(example(), controller="DaemonSet"))["reason"] == "invalid_controller"

def test_fractional_gpu():
    assert evaluate(replace(example(), gpu_per_replica=0.5))["reason"] == "invalid_count"

def test_boolean_gpu():
    assert evaluate(replace(example(), gpu_per_replica=True))["reason"] == "invalid_count"

def test_negative_surge():
    assert evaluate(replace(example(), max_surge=-1))["reason"] == "invalid_count"

def test_bad_pdb_count():
    assert evaluate(replace(example(), pdb_min_available=3))["reason"] == "invalid_count"

def test_nan_rpo():
    assert evaluate(replace(example(), rpo_minutes=float("nan")))["reason"] == "invalid_metric"

def test_insufficient_surge():
    assert "surge_capacity" in evaluate(replace(example(), free_gpu_units=0))["failed_gates"]

def test_pdb_failure():
    assert "pdb" in evaluate(replace(example(), available_after_failure=0))["failed_gates"]

def test_rpo_failure():
    assert "rpo" in evaluate(replace(example(), rpo_minutes=16))["failed_gates"]

def test_rto_failure():
    assert "rto" in evaluate(replace(example(), rto_minutes=31))["failed_gates"]

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nonboolean_change():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("checkpoint_verified", "checkpoint"),
    ("restore_tested", "restore"),
    ("artifact_compatible", "artifact"),
    ("immutable_image_verified", "image"),
    ("readiness_validated", "readiness"),
    ("canary_slo_verified", "canary_slo"),
    ("device_plugin_ready", "device_plugin"),
    ("cuda_verified", "cuda"),
    ("gpu_health_verified", "gpu_health"),
    ("n_minus_one_verified", "n_minus_one"),
    ("disruption_budget_verified", "disruption"),
    ("tenant_security_verified", "security"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_required_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run locally:

```bash
python -m pip install pytest
python gpu_rollout_policy.py
python -m pytest -q
```

**Expected 29 cases:** 15 standalone and 14 parametrized. Execution remains pending.

## Ten synthetic incident drills

1. Insufficient surge GPU capacity.
2. PDB cannot protect minimum availability.
3. RPO budget exceeded.
4. RTO budget exceeded.
5. Checkpoint missing or unverified.
6. Restore test fails.
7. Model artifact or image compatibility unknown.
8. GPU driver/plugin/CUDA canary unhealthy.
9. N-1 or disruption budget not verified.
10. Stale telemetry, change approval or rollback absent.

## Staging acceptance

1. Confirm controller, replicas, GPU resource key, affinity, node supply and quota.
2. Verify pinned image/model digest and startup/readiness probe behavior.
3. Measure available surge GPU capacity and canary SLO under representative load.
4. Exercise approved rollout and controlled drain; observe PDB eviction behavior.
5. Restore a disposable training checkpoint and verify actual RPO/RTO.
6. Confirm GPU UUID, driver, device-plugin, CUDA/NCCL and DCGM health.
7. Validate Karpenter disruption, N-1 recovery, security and rollback.
8. Record UTC evidence and signoff.

**Limitation:** This evaluator verifies declared synthetic evidence; it cannot attest actual scheduler capacity, model correctness, checkpoint integrity or production SLOs.

## 100-point rubric

Rollout architecture 20; PDB/capacity 20; checkpoint/RPO/RTO 20; incidents/runbooks 20; tests/staging 20. Pass >=90 with no critical correctness, data-loss or rollback blocker.
