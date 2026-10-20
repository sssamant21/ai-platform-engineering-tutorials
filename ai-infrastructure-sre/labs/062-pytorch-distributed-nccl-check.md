# Lab 062 — PyTorch Distributed and NCCL Gate

**Python 3.11+ · pytest · offline synthetic distributed training readiness.** No live CUDA, NCCL, Kubernetes or cloud calls.

## Goal

Reject invalid rank/GPU geometry, insufficient N-1 fabric bandwidth, collective SLO violations, scaling-efficiency regressions, checkpoint RPO/RTO breaches, telemetry staleness and missing distributed training controls. Thresholds are synthetic site policies and Boolean attestations do not independently prove operational readiness.

## File: `pytorch_nccl_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    job: str
    nodes: int
    gpus_per_node: int
    world_size: int
    data_parallel_replicas: int
    microbatch_per_rank: int
    gradient_accumulation: int
    one_gpu_samples_per_second: float
    observed_samples_per_second: float
    min_scaling_efficiency: float
    required_fabric_gbps: float
    surviving_fabric_gbps: float
    collective_p95_ms: float
    collective_slo_ms: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    restore_minutes: float
    restore_rto_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    driver_cuda_pytorch_compatible: bool
    nccl_version_pinned: bool
    gpu_nic_topology_verified: bool
    process_group_backend_approved: bool
    rank_membership_verified: bool
    collective_ordering_reviewed: bool
    rendezvous_fencing_verified: bool
    collective_timeout_configured: bool
    network_fabric_verified: bool
    rdma_efa_roce_config_verified: bool
    distributed_sampler_verified: bool
    global_batch_semantics_verified: bool
    ddp_fsdp_strategy_reviewed: bool
    checkpoint_manifest_verified: bool
    optimizer_rng_sampler_restored: bool
    checkpoint_restore_tested: bool
    mixed_precision_policy_verified: bool
    gpu_memory_headroom_verified: bool
    pod_security_verified: bool
    workload_identity_verified: bool
    network_policy_verified: bool
    nccl_observability_verified: bool
    incident_drills_completed: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.job, str) or not p.job.strip():
        return {"accepted": False, "reason": "missing_job"}
    counts = (p.nodes, p.gpus_per_node, p.world_size, p.data_parallel_replicas,
              p.microbatch_per_rank, p.gradient_accumulation)
    if any(type(v) is not int or v <= 0 for v in counts):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.one_gpu_samples_per_second, p.observed_samples_per_second,
               p.min_scaling_efficiency, p.required_fabric_gbps,
               p.surviving_fabric_gbps, p.collective_p95_ms,
               p.collective_slo_ms, p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.restore_minutes, p.restore_rto_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.one_gpu_samples_per_second, p.min_scaling_efficiency,
                   p.collective_slo_ms, p.checkpoint_rpo_min,
                   p.restore_rto_min, p.telemetry_max_age_s) <= 0
            or p.min_scaling_efficiency > 1):
        return {"accepted": False, "reason": "invalid_metric"}
    efficiency = p.observed_samples_per_second / (
        p.world_size * p.one_gpu_samples_per_second)
    gates = {
        "rank_geometry": p.world_size == p.nodes * p.gpus_per_node,
        "data_parallel_geometry": p.world_size % p.data_parallel_replicas == 0,
        "scaling_efficiency": efficiency >= p.min_scaling_efficiency,
        "degraded_fabric": p.surviving_fabric_gbps >= p.required_fabric_gbps,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "restore_rto": p.restore_minutes <= p.restore_rto_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "driver_cuda_pytorch_compatible": p.driver_cuda_pytorch_compatible is True,
        "nccl_version_pinned": p.nccl_version_pinned is True,
        "gpu_nic_topology_verified": p.gpu_nic_topology_verified is True,
        "process_group_backend_approved": p.process_group_backend_approved is True,
        "rank_membership_verified": p.rank_membership_verified is True,
        "collective_ordering_reviewed": p.collective_ordering_reviewed is True,
        "rendezvous_fencing_verified": p.rendezvous_fencing_verified is True,
        "collective_timeout_configured": p.collective_timeout_configured is True,
        "network_fabric_verified": p.network_fabric_verified is True,
        "rdma_efa_roce_config_verified": p.rdma_efa_roce_config_verified is True,
        "distributed_sampler_verified": p.distributed_sampler_verified is True,
        "global_batch_semantics_verified": p.global_batch_semantics_verified is True,
        "ddp_fsdp_strategy_reviewed": p.ddp_fsdp_strategy_reviewed is True,
        "checkpoint_manifest_verified": p.checkpoint_manifest_verified is True,
        "optimizer_rng_sampler_restored": p.optimizer_rng_sampler_restored is True,
        "checkpoint_restore_tested": p.checkpoint_restore_tested is True,
        "mixed_precision_policy_verified": p.mixed_precision_policy_verified is True,
        "gpu_memory_headroom_verified": p.gpu_memory_headroom_verified is True,
        "pod_security_verified": p.pod_security_verified is True,
        "workload_identity_verified": p.workload_identity_verified is True,
        "network_policy_verified": p.network_policy_verified is True,
        "nccl_observability_verified": p.nccl_observability_verified is True,
        "incident_drills_completed": p.incident_drills_completed is True,
        "change_approved": p.change_approved is True,
        "rollback_tested": p.rollback_tested is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "global_batch": p.microbatch_per_rank * p.gradient_accumulation * p.data_parallel_replicas,
            "scaling_efficiency": round(efficiency, 4)}

def example():
    return Plan(
        "synthetic-ddp-nccl", 2, 8, 16, 16, 2, 8,
        100.0, 1200.0, 0.70, 160.0, 200.0,
        15.0, 30.0, 5.0, 30.0,
        60.0, 120.0, 10.0, 120.0,
        *([True] * 26)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_pytorch_nccl_policy.py`

```python
from dataclasses import replace
import pytest
from pytorch_nccl_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_job():
    assert evaluate(replace(example(), job=" "))["reason"] == "missing_job"

def test_bool_count():
    assert evaluate(replace(example(), world_size=True))["reason"] == "invalid_count"

def test_fractional_count():
    assert evaluate(replace(example(), nodes=1.5))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), surviving_fabric_gbps=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), min_scaling_efficiency=True))["reason"] == "invalid_metric"

def test_bad_efficiency_threshold():
    assert evaluate(replace(example(), min_scaling_efficiency=1.1))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 34

def test_global_batch():
    assert evaluate(example())["global_batch"] == 256

def test_nonboolean_review():
    assert "independent_review_approved" in evaluate(
        replace(example(), independent_review_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("world_size", "rank_geometry", 8),
    ("data_parallel_replicas", "data_parallel_geometry", 3),
    ("observed_samples_per_second", "scaling_efficiency", 500.0),
    ("surviving_fabric_gbps", "degraded_fabric", 100.0),
    ("collective_p95_ms", "collective_slo", 31.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("restore_minutes", "restore_rto", 121.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("driver_cuda_pytorch_compatible", "driver_cuda_pytorch_compatible"),
    ("nccl_version_pinned", "nccl_version_pinned"),
    ("gpu_nic_topology_verified", "gpu_nic_topology_verified"),
    ("process_group_backend_approved", "process_group_backend_approved"),
    ("rank_membership_verified", "rank_membership_verified"),
    ("collective_ordering_reviewed", "collective_ordering_reviewed"),
    ("rendezvous_fencing_verified", "rendezvous_fencing_verified"),
    ("collective_timeout_configured", "collective_timeout_configured"),
    ("network_fabric_verified", "network_fabric_verified"),
    ("rdma_efa_roce_config_verified", "rdma_efa_roce_config_verified"),
    ("distributed_sampler_verified", "distributed_sampler_verified"),
    ("global_batch_semantics_verified", "global_batch_semantics_verified"),
    ("ddp_fsdp_strategy_reviewed", "ddp_fsdp_strategy_reviewed"),
    ("checkpoint_manifest_verified", "checkpoint_manifest_verified"),
    ("optimizer_rng_sampler_restored", "optimizer_rng_sampler_restored"),
    ("checkpoint_restore_tested", "checkpoint_restore_tested"),
    ("mixed_precision_policy_verified", "mixed_precision_policy_verified"),
    ("gpu_memory_headroom_verified", "gpu_memory_headroom_verified"),
    ("pod_security_verified", "pod_security_verified"),
    ("workload_identity_verified", "workload_identity_verified"),
    ("network_policy_verified", "network_policy_verified"),
    ("nccl_observability_verified", "nccl_observability_verified"),
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
python pytorch_nccl_policy.py
python -m pytest -q
```

**46 planned pytest cases:** 12 standalone + 8 numeric + 26 Boolean. Runtime execution, live NCCL diagnostics and independent review are pending.

## Ten synthetic incident drills

1. Rank/world-size mismatch.
2. Data-parallel partitioning mismatch.
3. Scaling-efficiency regression.
4. N-1 fabric bandwidth deficit.
5. Collective p95 SLO breach.
6. Checkpoint RPO breach.
7. Restore RTO breach.
8. Stale telemetry.
9. GPU/NIC topology or rendezvous fencing failure.
10. Missing security, rollback or independent approval.

## 100-point acceptance

Process groups 20; NCCL/performance 20; checkpoint/data correctness 20; security/incident response 20; tests/staging/review 20. Require >=90, every critical gate green and signed staging evidence for production.
