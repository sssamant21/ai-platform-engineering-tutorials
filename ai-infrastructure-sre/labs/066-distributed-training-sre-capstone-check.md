# Lab 066 — Distributed Training SRE Capstone Gate

**Python 3.11+ · pytest · offline synthetic acceptance.** No GPU, scheduler, storage, Kubernetes or cloud mutations.

## Goal

Fail closed on rank geometry, GPU ownership, gang capacity, N-1 fabric, NCCL collective SLO, scaling efficiency, checkpoint shard integrity, checkpoint bandwidth, RPO/RTO, telemetry freshness and critical enterprise control attestations. All thresholds are synthetic; Boolean flags require separate evidence.

## File: `distributed_training_capstone_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    platform: str
    nodes: int
    gpus_per_node: int
    world_size: int
    admitted_gpus: int
    gang_required_gpus: int
    expected_checkpoint_shards: int
    verified_checkpoint_shards: int
    single_gpu_samples_per_second: float
    observed_samples_per_second: float
    minimum_scaling_efficiency: float
    required_fabric_gbps: float
    surviving_fabric_gbps: float
    collective_p95_ms: float
    collective_slo_ms: float
    checkpoint_gb: float
    checkpoint_window_s: float
    checkpoint_write_gbps: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    recovery_minutes: float
    recovery_rto_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    architecture_approved: bool
    slurm_control_plane_verified: bool
    ray_gcs_recovery_verified: bool
    kubernetes_gpu_admission_verified: bool
    gang_scheduling_verified: bool
    tenant_fairshare_verified: bool
    gpu_inventory_verified: bool
    driver_cuda_nccl_pinned: bool
    rank_topology_verified: bool
    fabric_n_minus_one_verified: bool
    collective_slo_verified: bool
    data_loader_slo_verified: bool
    checkpoint_global_commit_verified: bool
    checkpoint_digest_verified: bool
    optimizer_rng_sampler_restored: bool
    rendezvous_fencing_verified: bool
    preemption_requeue_tested: bool
    cross_zone_restore_tested: bool
    rpo_rto_evidence_verified: bool
    gpu_telemetry_verified: bool
    queue_telemetry_verified: bool
    security_identity_verified: bool
    tenant_network_isolation_verified: bool
    finops_chargeback_verified: bool
    incident_drills_completed: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.platform, str) or not p.platform.strip():
        return {"accepted": False, "reason": "missing_platform"}
    counts = (p.nodes, p.gpus_per_node, p.world_size, p.admitted_gpus,
              p.gang_required_gpus, p.expected_checkpoint_shards, p.verified_checkpoint_shards)
    if any(type(x) is not int or x <= 0 for x in counts):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.single_gpu_samples_per_second, p.observed_samples_per_second,
               p.minimum_scaling_efficiency, p.required_fabric_gbps,
               p.surviving_fabric_gbps, p.collective_p95_ms, p.collective_slo_ms,
               p.checkpoint_gb, p.checkpoint_window_s, p.checkpoint_write_gbps,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.recovery_minutes, p.recovery_rto_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or min(p.single_gpu_samples_per_second, p.minimum_scaling_efficiency,
                   p.collective_slo_ms, p.checkpoint_gb, p.checkpoint_window_s,
                   p.checkpoint_rpo_min, p.recovery_rto_min,
                   p.telemetry_max_age_s) <= 0
            or p.minimum_scaling_efficiency > 1):
        return {"accepted": False, "reason": "invalid_metric"}
    efficiency = p.observed_samples_per_second / (
        p.world_size * p.single_gpu_samples_per_second)
    required_checkpoint_gbps = p.checkpoint_gb / p.checkpoint_window_s
    gates = {
        "rank_geometry": p.world_size == p.nodes * p.gpus_per_node,
        "gpu_ownership_capacity": p.admitted_gpus <= p.nodes * p.gpus_per_node,
        "gang_capacity": p.gang_required_gpus <= p.admitted_gpus,
        "checkpoint_shard_completeness": p.verified_checkpoint_shards == p.expected_checkpoint_shards,
        "scaling_efficiency": efficiency >= p.minimum_scaling_efficiency,
        "n_minus_one_fabric": p.surviving_fabric_gbps >= p.required_fabric_gbps,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "checkpoint_bandwidth": p.checkpoint_write_gbps >= required_checkpoint_gbps,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "recovery_rto": p.recovery_minutes <= p.recovery_rto_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "architecture_approved": p.architecture_approved is True,
        "slurm_control_plane_verified": p.slurm_control_plane_verified is True,
        "ray_gcs_recovery_verified": p.ray_gcs_recovery_verified is True,
        "kubernetes_gpu_admission_verified": p.kubernetes_gpu_admission_verified is True,
        "gang_scheduling_verified": p.gang_scheduling_verified is True,
        "tenant_fairshare_verified": p.tenant_fairshare_verified is True,
        "gpu_inventory_verified": p.gpu_inventory_verified is True,
        "driver_cuda_nccl_pinned": p.driver_cuda_nccl_pinned is True,
        "rank_topology_verified": p.rank_topology_verified is True,
        "fabric_n_minus_one_verified": p.fabric_n_minus_one_verified is True,
        "collective_slo_verified": p.collective_slo_verified is True,
        "data_loader_slo_verified": p.data_loader_slo_verified is True,
        "checkpoint_global_commit_verified": p.checkpoint_global_commit_verified is True,
        "checkpoint_digest_verified": p.checkpoint_digest_verified is True,
        "optimizer_rng_sampler_restored": p.optimizer_rng_sampler_restored is True,
        "rendezvous_fencing_verified": p.rendezvous_fencing_verified is True,
        "preemption_requeue_tested": p.preemption_requeue_tested is True,
        "cross_zone_restore_tested": p.cross_zone_restore_tested is True,
        "rpo_rto_evidence_verified": p.rpo_rto_evidence_verified is True,
        "gpu_telemetry_verified": p.gpu_telemetry_verified is True,
        "queue_telemetry_verified": p.queue_telemetry_verified is True,
        "security_identity_verified": p.security_identity_verified is True,
        "tenant_network_isolation_verified": p.tenant_network_isolation_verified is True,
        "finops_chargeback_verified": p.finops_chargeback_verified is True,
        "incident_drills_completed": p.incident_drills_completed is True,
        "rollback_tested": p.rollback_tested is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "scaling_efficiency": round(efficiency, 4),
            "required_checkpoint_gbps": round(required_checkpoint_gbps, 4)}

def example():
    return Plan("synthetic-distributed-training-capstone",
                4, 8, 32, 32, 32, 32, 32,
                100.0, 2400.0, 0.7, 160.0, 200.0,
                50.0, 70.0, 2400.0, 300.0, 10.0,
                18.0, 30.0, 95.0, 120.0, 10.0, 120.0,
                *([True] * 27))

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_distributed_training_capstone_policy.py`

```python
from dataclasses import replace
import pytest
from distributed_training_capstone_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_platform():
    assert evaluate(replace(example(), platform=" "))["reason"] == "missing_platform"

def test_bool_count():
    assert evaluate(replace(example(), world_size=True))["reason"] == "invalid_count"

def test_fractional_count():
    assert evaluate(replace(example(), admitted_gpus=1.5))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), surviving_fabric_gbps=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), checkpoint_rpo_min=True))["reason"] == "invalid_metric"

def test_invalid_efficiency():
    assert evaluate(replace(example(), minimum_scaling_efficiency=1.1))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 38

def test_efficiency_math():
    assert evaluate(example())["scaling_efficiency"] == 0.75

def test_checkpoint_bandwidth_math():
    assert evaluate(example())["required_checkpoint_gbps"] == 8.0

def test_nonboolean_approval():
    assert "independent_review_approved" in evaluate(
        replace(example(), independent_review_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("world_size", "rank_geometry", 16),
    ("admitted_gpus", "gpu_ownership_capacity", 33),
    ("gang_required_gpus", "gang_capacity", 33),
    ("verified_checkpoint_shards", "checkpoint_shard_completeness", 31),
    ("observed_samples_per_second", "scaling_efficiency", 1000.0),
    ("surviving_fabric_gbps", "n_minus_one_fabric", 100.0),
    ("collective_p95_ms", "collective_slo", 71.0),
    ("checkpoint_write_gbps", "checkpoint_bandwidth", 7.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("recovery_minutes", "recovery_rto", 121.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field", [
    "architecture_approved",
    "slurm_control_plane_verified",
    "ray_gcs_recovery_verified",
    "kubernetes_gpu_admission_verified",
    "gang_scheduling_verified",
    "tenant_fairshare_verified",
    "gpu_inventory_verified",
    "driver_cuda_nccl_pinned",
    "rank_topology_verified",
    "fabric_n_minus_one_verified",
    "collective_slo_verified",
    "data_loader_slo_verified",
    "checkpoint_global_commit_verified",
    "checkpoint_digest_verified",
    "optimizer_rng_sampler_restored",
    "rendezvous_fencing_verified",
    "preemption_requeue_tested",
    "cross_zone_restore_tested",
    "rpo_rto_evidence_verified",
    "gpu_telemetry_verified",
    "queue_telemetry_verified",
    "security_identity_verified",
    "tenant_network_isolation_verified",
    "finops_chargeback_verified",
    "incident_drills_completed",
    "rollback_tested",
    "independent_review_approved",
])
def test_boolean_gate(field):
    assert field in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run locally:

```bash
python -m pip install pytest
python distributed_training_capstone_policy.py
python -m pytest -q
```

**51 planned pytest cases:** 13 standalone + 11 numeric + 27 Boolean. Runtime tests, live GPU staging drills and independent acceptance remain pending.

## Ten synthetic incident exercises

1. Slurm controller and accounting outage; record gate impact, recovery, RPO/RTO and rollback.
2. Ray GCS/head recovery; record gate impact, recovery, RPO/RTO and rollback.
3. Gang admission starvation; record gate impact, recovery, RPO/RTO and rollback.
4. GPU node failure and rank fencing; record gate impact, recovery, RPO/RTO and rollback.
5. NCCL collective deadlock; record gate impact, recovery, RPO/RTO and rollback.
6. N-1 fabric degradation; record gate impact, recovery, RPO/RTO and rollback.
7. Object-store/dataloader backpressure; record gate impact, recovery, RPO/RTO and rollback.
8. Partial checkpoint commit and corrupt shard; record gate impact, recovery, RPO/RTO and rollback.
9. Preemption and cross-zone restore; record gate impact, recovery, RPO/RTO and rollback.
10. Telemetry/security incident during SLO breach; record gate impact, recovery, RPO/RTO and rollback.

## 100-point evidence review

Architecture and scheduling 20; DDP/NCCL performance 20; checkpoint/recovery 20; security/observability/FinOps 20; tests/staging/review 20. Require >=90, no critical blockers, real staging evidence and independent signoff.
