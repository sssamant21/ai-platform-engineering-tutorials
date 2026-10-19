# Lab 034 — Kubernetes GPU Platform Capstone Acceptance Gate

**Python 3.11+ · pytest · offline synthetic evidence.** No Kubernetes, AWS, Azure, GPU or filesystem mutations.

## Goal

Evaluate the integrated Part 04 platform: physical GPU and device-plugin readiness, Operator ownership, runtime/CDI, tenant security, quota, scheduling, N-1, disruption, checkpoints, SLO, telemetry, FinOps and rollback. Invalid input and missing critical evidence must fail closed.

## File: `gpu_platform_capstone.py`

```python
from dataclasses import dataclass
import json
import math
import re

RESOURCE = re.compile(r"^nvidia\.com/(?:gpu|mig-[A-Za-z0-9.-]+)$")

@dataclass(frozen=True)
class Evidence:
    cluster: str
    gpu_resource: str
    requested_gpu_units: int
    allocatable_gpu_units: int
    n_minus_one_gpu_units: int
    tenant_quota_remaining: int
    checkpoint_age_minutes: float
    max_checkpoint_age_minutes: float
    telemetry_age_seconds: float
    max_telemetry_age_seconds: float
    projected_gpu_hour_cost: float
    max_gpu_hour_cost: float
    driver_healthy: bool
    device_plugin_registered: bool
    operator_or_managed_driver_verified: bool
    toolkit_cdi_verified: bool
    cuda_canary_passed: bool
    nccl_test_passed_or_not_applicable: bool
    mig_sharing_reviewed: bool
    pod_security_verified: bool
    rbac_verified: bool
    network_isolation_verified: bool
    image_provenance_verified: bool
    scheduler_topology_verified: bool
    autoscaler_verified: bool
    disruption_budget_verified: bool
    checkpoint_restore_tested: bool
    inference_or_training_slo_healthy: bool
    dcgm_alerts_verified: bool
    gpu_xid_ecc_reviewed: bool
    cost_attribution_verified: bool
    incident_game_day_passed: bool
    change_approved: bool
    rollback_tested: bool
    independent_security_reviewed: bool

def evaluate(e):
    if not isinstance(e, Evidence):
        raise TypeError("Evidence required")
    if not isinstance(e.cluster, str) or not e.cluster.strip():
        return {"accepted": False, "reason": "invalid_cluster"}
    if not isinstance(e.gpu_resource, str) or not RESOURCE.fullmatch(e.gpu_resource):
        return {"accepted": False, "reason": "invalid_gpu_resource"}
    counts = (e.requested_gpu_units, e.allocatable_gpu_units,
              e.n_minus_one_gpu_units, e.tenant_quota_remaining)
    if any(type(x) is not int or x < 0 for x in counts) or e.requested_gpu_units < 1:
        return {"accepted": False, "reason": "invalid_gpu_count"}
    metrics = (e.checkpoint_age_minutes, e.max_checkpoint_age_minutes,
               e.telemetry_age_seconds, e.max_telemetry_age_seconds,
               e.projected_gpu_hour_cost, e.max_gpu_hour_cost)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or e.max_checkpoint_age_minutes <= 0
            or e.max_telemetry_age_seconds <= 0 or e.max_gpu_hour_cost <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "gpu_capacity": e.requested_gpu_units <= e.allocatable_gpu_units,
        "n_minus_one": e.requested_gpu_units <= e.n_minus_one_gpu_units,
        "tenant_quota": e.requested_gpu_units <= e.tenant_quota_remaining,
        "checkpoint_freshness": e.checkpoint_age_minutes <= e.max_checkpoint_age_minutes,
        "telemetry_freshness": e.telemetry_age_seconds <= e.max_telemetry_age_seconds,
        "cost_guardrail": e.projected_gpu_hour_cost <= e.max_gpu_hour_cost,
        "driver": e.driver_healthy is True,
        "device_plugin": e.device_plugin_registered is True,
        "driver_ownership": e.operator_or_managed_driver_verified is True,
        "toolkit_cdi": e.toolkit_cdi_verified is True,
        "cuda_canary": e.cuda_canary_passed is True,
        "nccl": e.nccl_test_passed_or_not_applicable is True,
        "mig_sharing": e.mig_sharing_reviewed is True,
        "pod_security": e.pod_security_verified is True,
        "rbac": e.rbac_verified is True,
        "network_isolation": e.network_isolation_verified is True,
        "image_provenance": e.image_provenance_verified is True,
        "scheduler_topology": e.scheduler_topology_verified is True,
        "autoscaler": e.autoscaler_verified is True,
        "disruption_budget": e.disruption_budget_verified is True,
        "checkpoint_restore": e.checkpoint_restore_tested is True,
        "workload_slo": e.inference_or_training_slo_healthy is True,
        "dcgm_alerts": e.dcgm_alerts_verified is True,
        "xid_ecc": e.gpu_xid_ecc_reviewed is True,
        "cost_attribution": e.cost_attribution_verified is True,
        "game_day": e.incident_game_day_passed is True,
        "change": e.change_approved is True,
        "rollback": e.rollback_tested is True,
        "security_review": e.independent_security_reviewed is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "cluster": e.cluster, "gpu_resource": e.gpu_resource,
            "remaining_gpu_after_request": e.allocatable_gpu_units - e.requested_gpu_units}

def example():
    return Evidence(
        "synthetic-gpu-staging", "nvidia.com/gpu",
        4, 8, 4, 6, 10.0, 30.0, 15.0, 120.0, 40.0, 100.0,
        *([True] * 23)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_gpu_platform_capstone.py`

```python
from dataclasses import replace
import pytest
from gpu_platform_capstone import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "invalid_cluster"

def test_bad_resource():
    assert evaluate(replace(example(), gpu_resource="gpu"))["reason"] == "invalid_gpu_resource"

def test_fractional_gpu():
    assert evaluate(replace(example(), requested_gpu_units=1.5))["reason"] == "invalid_gpu_count"

def test_boolean_gpu():
    assert evaluate(replace(example(), requested_gpu_units=True))["reason"] == "invalid_gpu_count"

def test_nan_cost():
    assert evaluate(replace(example(), projected_gpu_hour_cost=float("nan")))["reason"] == "invalid_metric"

def test_negative_checkpoint_age():
    assert evaluate(replace(example(), checkpoint_age_minutes=-1))["reason"] == "invalid_metric"

def test_capacity():
    assert "gpu_capacity" in evaluate(replace(example(), requested_gpu_units=9))["failed_gates"]

def test_n_minus_one():
    assert "n_minus_one" in evaluate(replace(example(), requested_gpu_units=5))["failed_gates"]

def test_quota():
    assert "tenant_quota" in evaluate(replace(example(), tenant_quota_remaining=3))["failed_gates"]

def test_checkpoint_stale():
    assert "checkpoint_freshness" in evaluate(replace(example(), checkpoint_age_minutes=31))["failed_gates"]

def test_telemetry_stale():
    assert "telemetry_freshness" in evaluate(replace(example(), telemetry_age_seconds=121))["failed_gates"]

def test_cost():
    assert "cost_guardrail" in evaluate(replace(example(), projected_gpu_hour_cost=101))["failed_gates"]

def test_nonboolean_security():
    assert "security_review" in evaluate(replace(example(), independent_security_reviewed="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("driver_healthy", "driver"),
    ("device_plugin_registered", "device_plugin"),
    ("operator_or_managed_driver_verified", "driver_ownership"),
    ("toolkit_cdi_verified", "toolkit_cdi"),
    ("cuda_canary_passed", "cuda_canary"),
    ("nccl_test_passed_or_not_applicable", "nccl"),
    ("mig_sharing_reviewed", "mig_sharing"),
    ("pod_security_verified", "pod_security"),
    ("rbac_verified", "rbac"),
    ("network_isolation_verified", "network_isolation"),
    ("image_provenance_verified", "image_provenance"),
    ("scheduler_topology_verified", "scheduler_topology"),
    ("autoscaler_verified", "autoscaler"),
    ("disruption_budget_verified", "disruption_budget"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("inference_or_training_slo_healthy", "workload_slo"),
    ("dcgm_alerts_verified", "dcgm_alerts"),
    ("gpu_xid_ecc_reviewed", "xid_ecc"),
    ("cost_attribution_verified", "cost_attribution"),
    ("incident_game_day_passed", "game_day"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("independent_security_reviewed", "security_review"),
])
def test_critical_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run locally:

```bash
python -m pip install pytest
python gpu_platform_capstone.py
python -m pytest -q
```

**Planned 38 pytest cases** (15 standalone + 23 parameterized). Execution is pending.

## Ten incident game-day exercises

1. Device plugin disappears.
2. Driver/toolkit incompatible with CUDA.
3. CDI allocation fails.
4. MIG resource key drifts.
5. Tenant security control is missing.
6. GPU quota or N-1 capacity exhausted.
7. Karpenter/managed pool scale-out fails.
8. Checkpoint age breaches RPO.
9. DCGM telemetry or GPU health evidence is missing.
10. Workload SLO, cost guardrail or rollback fails.

## Staging evidence checklist

Capture reviewed versions, GPU UUID and allocatable keys, Pod Security/RBAC, quota, scheduler events, CUDA/NCCL canary, DCGM/Xid/ECC, checkpoint restore and RPO/RTO, autoscaler scale-out, disruption/N-1, cost attribution, SLO, rollback and signed go/no-go. Do not store secrets or sensitive patient/model data in public GitHub.

**Limitations:** The synthetic evaluator trusts input attestations and cannot prove hardware isolation, GPU correctness, Kubernetes policy enforcement, cloud quota, real checkpoint recovery or production SLO.

## 100-point assessment

GPU platform 20; tenant/security 20; scaling/recovery 20; SLO/observability 20; tests/game days/signoff 20. >=90 and zero critical blockers required.
