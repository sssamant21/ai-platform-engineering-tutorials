# Lab 061 — Ray Cluster Operations Gate

**Python 3.11+ · pytest · offline synthetic KubeRay policy.** No live Ray, Kubernetes, cloud or GPU operations.

## Goal

Fail closed on invalid GPU geometry, insufficient autoscaler/N-1 capacity, object-store/spill headroom, scheduling latency, GCS recovery, checkpoint RPO/RTO, telemetry staleness and missing security/governance evidence. Thresholds are synthetic site policy; Boolean fields are attestations, not live evidence.

## File: `ray_cluster_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    worker_nodes: int
    gpus_per_worker: int
    advertised_gpus: int
    requested_gpus: int
    max_autoscaled_gpus: int
    surviving_gpus: int
    required_n_minus_one_gpus: int
    object_store_used_gb: float
    object_store_limit_gb: float
    spill_used_gb: float
    spill_capacity_gb: float
    placement_pending_p95_min: float
    placement_slo_min: float
    gcs_recovery_min: float
    gcs_rto_min: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    restore_minutes: float
    restore_rto_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    head_ha_design_reviewed: bool
    kuberay_operator_healthy: bool
    raycluster_crd_verified: bool
    rayjob_lifecycle_verified: bool
    gpu_resource_advertisement_verified: bool
    placement_groups_verified: bool
    autoscaler_bounds_reviewed: bool
    k8s_quota_and_admission_verified: bool
    node_gpu_driver_compatible: bool
    object_store_memory_budgeted: bool
    spill_storage_verified: bool
    gcs_recovery_tested: bool
    actor_restart_semantics_reviewed: bool
    task_idempotency_verified: bool
    checkpoint_restore_tested: bool
    dataset_lineage_pinned: bool
    network_isolation_verified: bool
    workload_identity_verified: bool
    tenant_rbac_verified: bool
    ray_dashboard_secured: bool
    prometheus_metrics_verified: bool
    incident_drills_completed: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    counts = (p.worker_nodes, p.gpus_per_worker, p.advertised_gpus,
              p.requested_gpus, p.max_autoscaled_gpus, p.surviving_gpus,
              p.required_n_minus_one_gpus)
    if any(type(v) is not int or v <= 0 for v in counts):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.object_store_used_gb, p.object_store_limit_gb,
               p.spill_used_gb, p.spill_capacity_gb,
               p.placement_pending_p95_min, p.placement_slo_min,
               p.gcs_recovery_min, p.gcs_rto_min,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.restore_minutes, p.restore_rto_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.object_store_limit_gb, p.spill_capacity_gb,
                   p.placement_slo_min, p.gcs_rto_min,
                   p.checkpoint_rpo_min, p.restore_rto_min,
                   p.telemetry_max_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "gpu_inventory": p.advertised_gpus == p.worker_nodes * p.gpus_per_worker,
        "gpu_allocation": p.requested_gpus <= p.advertised_gpus,
        "autoscaler_bounds": p.max_autoscaled_gpus >= p.requested_gpus,
        "n_minus_one_capacity": p.surviving_gpus >= p.required_n_minus_one_gpus,
        "object_store_headroom": p.object_store_used_gb < p.object_store_limit_gb,
        "spill_headroom": p.spill_used_gb < p.spill_capacity_gb,
        "placement_slo": p.placement_pending_p95_min <= p.placement_slo_min,
        "gcs_rto": p.gcs_recovery_min <= p.gcs_rto_min,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "restore_rto": p.restore_minutes <= p.restore_rto_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "head_ha_design_reviewed": p.head_ha_design_reviewed is True,
        "kuberay_operator_healthy": p.kuberay_operator_healthy is True,
        "raycluster_crd_verified": p.raycluster_crd_verified is True,
        "rayjob_lifecycle_verified": p.rayjob_lifecycle_verified is True,
        "gpu_resource_advertisement_verified": p.gpu_resource_advertisement_verified is True,
        "placement_groups_verified": p.placement_groups_verified is True,
        "autoscaler_bounds_reviewed": p.autoscaler_bounds_reviewed is True,
        "k8s_quota_and_admission_verified": p.k8s_quota_and_admission_verified is True,
        "node_gpu_driver_compatible": p.node_gpu_driver_compatible is True,
        "object_store_memory_budgeted": p.object_store_memory_budgeted is True,
        "spill_storage_verified": p.spill_storage_verified is True,
        "gcs_recovery_tested": p.gcs_recovery_tested is True,
        "actor_restart_semantics_reviewed": p.actor_restart_semantics_reviewed is True,
        "task_idempotency_verified": p.task_idempotency_verified is True,
        "checkpoint_restore_tested": p.checkpoint_restore_tested is True,
        "dataset_lineage_pinned": p.dataset_lineage_pinned is True,
        "network_isolation_verified": p.network_isolation_verified is True,
        "workload_identity_verified": p.workload_identity_verified is True,
        "tenant_rbac_verified": p.tenant_rbac_verified is True,
        "ray_dashboard_secured": p.ray_dashboard_secured is True,
        "prometheus_metrics_verified": p.prometheus_metrics_verified is True,
        "incident_drills_completed": p.incident_drills_completed is True,
        "change_approved": p.change_approved is True,
        "rollback_tested": p.rollback_tested is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates), "physical_gpu_inventory": p.worker_nodes * p.gpus_per_worker}

def example():
    return Plan(
        "synthetic-ray-gpu-cluster",
        4, 8, 32, 16, 64, 24, 16,
        100.0, 200.0, 200.0, 500.0,
        5.0, 15.0, 2.0, 10.0,
        5.0, 30.0, 60.0, 120.0,
        10.0, 120.0,
        *([True] * 25)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_ray_cluster_policy.py`

```python
from dataclasses import replace
import pytest
from ray_cluster_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_bool_count():
    assert evaluate(replace(example(), worker_nodes=True))["reason"] == "invalid_count"

def test_fractional_count():
    assert evaluate(replace(example(), advertised_gpus=1.5))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), object_store_used_gb=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), checkpoint_age_min=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), gcs_rto_min=True))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 36

def test_nonboolean_review():
    assert "independent_review_approved" in evaluate(
        replace(example(), independent_review_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("advertised_gpus", "gpu_inventory", 31),
    ("requested_gpus", "gpu_allocation", 33),
    ("max_autoscaled_gpus", "autoscaler_bounds", 15),
    ("surviving_gpus", "n_minus_one_capacity", 15),
    ("object_store_used_gb", "object_store_headroom", 201.0),
    ("spill_used_gb", "spill_headroom", 501.0),
    ("placement_pending_p95_min", "placement_slo", 16.0),
    ("gcs_recovery_min", "gcs_rto", 11.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("restore_minutes", "restore_rto", 121.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("head_ha_design_reviewed", "head_ha_design_reviewed"),
    ("kuberay_operator_healthy", "kuberay_operator_healthy"),
    ("raycluster_crd_verified", "raycluster_crd_verified"),
    ("rayjob_lifecycle_verified", "rayjob_lifecycle_verified"),
    ("gpu_resource_advertisement_verified", "gpu_resource_advertisement_verified"),
    ("placement_groups_verified", "placement_groups_verified"),
    ("autoscaler_bounds_reviewed", "autoscaler_bounds_reviewed"),
    ("k8s_quota_and_admission_verified", "k8s_quota_and_admission_verified"),
    ("node_gpu_driver_compatible", "node_gpu_driver_compatible"),
    ("object_store_memory_budgeted", "object_store_memory_budgeted"),
    ("spill_storage_verified", "spill_storage_verified"),
    ("gcs_recovery_tested", "gcs_recovery_tested"),
    ("actor_restart_semantics_reviewed", "actor_restart_semantics_reviewed"),
    ("task_idempotency_verified", "task_idempotency_verified"),
    ("checkpoint_restore_tested", "checkpoint_restore_tested"),
    ("dataset_lineage_pinned", "dataset_lineage_pinned"),
    ("network_isolation_verified", "network_isolation_verified"),
    ("workload_identity_verified", "workload_identity_verified"),
    ("tenant_rbac_verified", "tenant_rbac_verified"),
    ("ray_dashboard_secured", "ray_dashboard_secured"),
    ("prometheus_metrics_verified", "prometheus_metrics_verified"),
    ("incident_drills_completed", "incident_drills_completed"),
    ("change_approved", "change_approved"),
    ("rollback_tested", "rollback_tested"),
    ("independent_review_approved", "independent_review_approved"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run offline:

```bash
python -m pip install pytest
python ray_cluster_policy.py
python -m pytest -q
```

**46 planned pytest cases:** 10 standalone + 11 numeric + 25 Boolean. Runtime execution, live Ray/GCS recovery and independent review are pending.

## Ten synthetic incident drills

1. GPU resource advertisement mismatch.
2. GPU allocation and autoscaler quota shortfall.
3. N-1 GPU capacity deficit.
4. Object store OOM.
5. Spill storage exhaustion.
6. Placement group scheduling SLO breach.
7. GCS recovery RTO breach.
8. Checkpoint RPO/restore RTO breach.
9. Stale telemetry.
10. Missing tenant security, idempotency or independent approval.

## 100-point acceptance

Ray control plane 20; GPU scheduling/autoscaling/object store 20; recovery/RPO/RTO 20; security/incidents 20; tests/staging/review 20. Require >=90, every critical gate green and signed live staging evidence for production.
