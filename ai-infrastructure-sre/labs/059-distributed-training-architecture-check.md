# Lab 059 — Distributed Training Architecture Gate

**Python 3.11+ · pytest · offline synthetic readiness policy.** No GPU, cloud, Kubernetes or network mutations.

## Goal

Fail closed on invalid rank/world-size geometry, insufficient surviving network bandwidth, scaling-efficiency budget, checkpoint RPO/RTO, telemetry staleness, and unreviewed distributed training safeguards. Thresholds are illustrative site policy; Boolean evidence fields are attestations, not live validation.

## File: `distributed_training_policy.py`

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
    microbatch_per_gpu: int
    gradient_accumulation: int
    single_gpu_samples_per_second: float
    distributed_samples_per_second: float
    minimum_scaling_efficiency: float
    required_network_gbps: float
    surviving_network_gbps: float
    collective_p95_ms: float
    collective_slo_ms: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    restore_minutes: float
    restore_rto_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    topology_documented: bool
    failure_domains_mapped: bool
    gpu_nic_numa_affinity_verified: bool
    distributed_backend_compatible: bool
    rendezvous_ha_verified: bool
    rank_membership_verified: bool
    collective_timeout_defined: bool
    dataset_sharding_verified: bool
    checkpoint_global_consistency_verified: bool
    checkpoint_restore_tested: bool
    preemption_policy_verified: bool
    pod_security_verified: bool
    workload_identity_verified: bool
    network_policy_verified: bool
    tenant_quotas_verified: bool
    nccl_telemetry_verified: bool
    gpu_slo_verified: bool
    incident_runbooks_tested: bool
    rollback_tested: bool
    change_approved: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.job, str) or not p.job.strip():
        return {"accepted": False, "reason": "missing_job"}
    counts = (p.nodes, p.gpus_per_node, p.world_size,
              p.data_parallel_replicas, p.microbatch_per_gpu, p.gradient_accumulation)
    if any(type(v) is not int or v <= 0 for v in counts):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.single_gpu_samples_per_second, p.distributed_samples_per_second,
               p.minimum_scaling_efficiency, p.required_network_gbps,
               p.surviving_network_gbps, p.collective_p95_ms, p.collective_slo_ms,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.restore_minutes, p.restore_rto_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.single_gpu_samples_per_second, p.minimum_scaling_efficiency,
                   p.collective_slo_ms, p.checkpoint_rpo_min,
                   p.restore_rto_min, p.telemetry_max_age_s) <= 0
            or p.minimum_scaling_efficiency > 1):
        return {"accepted": False, "reason": "invalid_metric"}
    efficiency = p.distributed_samples_per_second / (
        p.world_size * p.single_gpu_samples_per_second)
    gates = {
        "rank_geometry": p.world_size == p.nodes * p.gpus_per_node,
        "data_parallel_geometry": p.world_size % p.data_parallel_replicas == 0,
        "scaling_efficiency": efficiency >= p.minimum_scaling_efficiency,
        "degraded_network": p.surviving_network_gbps >= p.required_network_gbps,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "restore_rto": p.restore_minutes <= p.restore_rto_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "topology_documented": p.topology_documented is True,
        "failure_domains_mapped": p.failure_domains_mapped is True,
        "gpu_nic_numa_affinity_verified": p.gpu_nic_numa_affinity_verified is True,
        "distributed_backend_compatible": p.distributed_backend_compatible is True,
        "rendezvous_ha_verified": p.rendezvous_ha_verified is True,
        "rank_membership_verified": p.rank_membership_verified is True,
        "collective_timeout_defined": p.collective_timeout_defined is True,
        "dataset_sharding_verified": p.dataset_sharding_verified is True,
        "checkpoint_global_consistency_verified": p.checkpoint_global_consistency_verified is True,
        "checkpoint_restore_tested": p.checkpoint_restore_tested is True,
        "preemption_policy_verified": p.preemption_policy_verified is True,
        "pod_security_verified": p.pod_security_verified is True,
        "workload_identity_verified": p.workload_identity_verified is True,
        "network_policy_verified": p.network_policy_verified is True,
        "tenant_quotas_verified": p.tenant_quotas_verified is True,
        "nccl_telemetry_verified": p.nccl_telemetry_verified is True,
        "gpu_slo_verified": p.gpu_slo_verified is True,
        "incident_runbooks_tested": p.incident_runbooks_tested is True,
        "rollback_tested": p.rollback_tested is True,
        "change_approved": p.change_approved is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "global_batch": p.microbatch_per_gpu * p.gradient_accumulation * p.data_parallel_replicas,
            "scaling_efficiency": round(efficiency, 4)}

def example():
    return Plan(
        "synthetic-distributed-training",
        4, 8, 32, 32, 2, 4,
        100.0, 2400.0, 0.7, 160.0, 200.0,
        15.0, 30.0, 5.0, 30.0,
        60.0, 120.0, 10.0, 120.0,
        *([True] * 21)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_distributed_training_policy.py`

```python
from dataclasses import replace
import pytest
from distributed_training_policy import evaluate, example

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

def test_infinity():
    assert evaluate(replace(example(), required_network_gbps=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), minimum_scaling_efficiency=True))["reason"] == "invalid_metric"

def test_bad_efficiency_threshold():
    assert evaluate(replace(example(), minimum_scaling_efficiency=1.1))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 29

def test_global_batch():
    assert evaluate(example())["global_batch"] == 256

def test_nonboolean_approval():
    assert "independent_review_approved" in evaluate(
        replace(example(), independent_review_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("world_size", "rank_geometry", 24),
    ("data_parallel_replicas", "data_parallel_geometry", 3),
    ("distributed_samples_per_second", "scaling_efficiency", 1000.0),
    ("surviving_network_gbps", "degraded_network", 100.0),
    ("collective_p95_ms", "collective_slo", 31.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("restore_minutes", "restore_rto", 121.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("topology_documented", "topology_documented"),
    ("failure_domains_mapped", "failure_domains_mapped"),
    ("gpu_nic_numa_affinity_verified", "gpu_nic_numa_affinity_verified"),
    ("distributed_backend_compatible", "distributed_backend_compatible"),
    ("rendezvous_ha_verified", "rendezvous_ha_verified"),
    ("rank_membership_verified", "rank_membership_verified"),
    ("collective_timeout_defined", "collective_timeout_defined"),
    ("dataset_sharding_verified", "dataset_sharding_verified"),
    ("checkpoint_global_consistency_verified", "checkpoint_global_consistency_verified"),
    ("checkpoint_restore_tested", "checkpoint_restore_tested"),
    ("preemption_policy_verified", "preemption_policy_verified"),
    ("pod_security_verified", "pod_security_verified"),
    ("workload_identity_verified", "workload_identity_verified"),
    ("network_policy_verified", "network_policy_verified"),
    ("tenant_quotas_verified", "tenant_quotas_verified"),
    ("nccl_telemetry_verified", "nccl_telemetry_verified"),
    ("gpu_slo_verified", "gpu_slo_verified"),
    ("incident_runbooks_tested", "incident_runbooks_tested"),
    ("rollback_tested", "rollback_tested"),
    ("change_approved", "change_approved"),
    ("independent_review_approved", "independent_review_approved"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run offline:

```bash
python -m pip install pytest
python distributed_training_policy.py
python -m pytest -q
```

**41 planned pytest cases:** 12 standalone + 8 numeric + 21 Boolean. Tests are not claimed as executed. No live NCCL/GPU, cluster failover or independent technical validation has been performed.

## Ten synthetic failure drills

1. World-size/rank mismatch.
2. Invalid data-parallel partitioning.
3. Scaling efficiency regression.
4. N-1 network bandwidth deficit.
5. Collective p95 SLO breach.
6. Checkpoint RPO breach.
7. Restore RTO breach.
8. Stale telemetry.
9. Rendezvous or GPU-to-NIC topology review failure.
10. Missing rollback or independent approval.

## 100-point evidence review

Architecture 20; performance/scheduling 20; dataset/checkpoint recovery 20; security/incidents 20; tests/live staging/review 20. Require >=90, all critical gates and signed live evidence for production.
