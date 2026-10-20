# Lab 065 — Multi-Node Training Performance Gate

**Python 3.11+ · pytest · offline synthetic policy.** No GPU, network stress test or cloud API calls.

## Goal

Fail closed on invalid GPU geometry, scaling-efficiency regression, NCCL collective SLO, data-loader wait, rank skew, degraded fabric capacity, checkpoint bandwidth, GPU memory, telemetry freshness and missing operational controls. Thresholds are illustrative site policy.

## File: `multi_node_performance_policy.py`

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
    one_gpu_samples_per_second: float
    observed_samples_per_second: float
    minimum_scaling_efficiency: float
    step_p95_ms: float
    step_slo_ms: float
    collective_p95_ms: float
    collective_slo_ms: float
    loader_wait_p95_ms: float
    loader_wait_slo_ms: float
    rank_skew_p95_ms: float
    rank_skew_slo_ms: float
    required_fabric_gbps: float
    surviving_fabric_gbps: float
    checkpoint_gb: float
    checkpoint_window_s: float
    checkpoint_write_gbps: float
    gpu_memory_used_gb: float
    gpu_memory_limit_gb: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    baseline_reproducible: bool
    model_and_precision_pinned: bool
    global_batch_comparable: bool
    dataset_manifest_pinned: bool
    rank_topology_verified: bool
    gpu_nic_numa_affinity_verified: bool
    nvlink_topology_verified: bool
    fabric_capacity_verified: bool
    nccl_collectives_profiled: bool
    straggler_rank_instrumented: bool
    data_loader_profiled: bool
    object_storage_profiled: bool
    checkpoint_io_profiled: bool
    gpu_memory_headroom_verified: bool
    cpu_numa_headroom_verified: bool
    power_thermal_limits_verified: bool
    slo_dashboard_verified: bool
    telemetry_freshness_verified: bool
    regression_canary_verified: bool
    cost_per_token_reviewed: bool
    n_minus_one_tested: bool
    rollback_verified: bool
    incident_drills_completed: bool
    security_reviewed: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.job, str) or not p.job.strip():
        return {"accepted": False, "reason": "missing_job"}
    if any(type(x) is not int or x <= 0 for x in (p.nodes, p.gpus_per_node, p.world_size)):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.one_gpu_samples_per_second, p.observed_samples_per_second,
               p.minimum_scaling_efficiency, p.step_p95_ms, p.step_slo_ms,
               p.collective_p95_ms, p.collective_slo_ms,
               p.loader_wait_p95_ms, p.loader_wait_slo_ms,
               p.rank_skew_p95_ms, p.rank_skew_slo_ms,
               p.required_fabric_gbps, p.surviving_fabric_gbps,
               p.checkpoint_gb, p.checkpoint_window_s, p.checkpoint_write_gbps,
               p.gpu_memory_used_gb, p.gpu_memory_limit_gb,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or min(p.one_gpu_samples_per_second, p.minimum_scaling_efficiency,
                   p.step_slo_ms, p.collective_slo_ms, p.loader_wait_slo_ms,
                   p.rank_skew_slo_ms, p.checkpoint_gb, p.checkpoint_window_s,
                   p.gpu_memory_limit_gb, p.telemetry_max_age_s) <= 0
            or p.minimum_scaling_efficiency > 1):
        return {"accepted": False, "reason": "invalid_metric"}
    efficiency = p.observed_samples_per_second / (
        p.world_size * p.one_gpu_samples_per_second)
    gates = {
        "rank_geometry": p.world_size == p.nodes * p.gpus_per_node,
        "scaling_efficiency": efficiency >= p.minimum_scaling_efficiency,
        "step_slo": p.step_p95_ms <= p.step_slo_ms,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "loader_wait_slo": p.loader_wait_p95_ms <= p.loader_wait_slo_ms,
        "rank_skew_slo": p.rank_skew_p95_ms <= p.rank_skew_slo_ms,
        "degraded_fabric": p.surviving_fabric_gbps >= p.required_fabric_gbps,
        "checkpoint_bandwidth": p.checkpoint_write_gbps >= p.checkpoint_gb / p.checkpoint_window_s,
        "gpu_memory_headroom": p.gpu_memory_used_gb < p.gpu_memory_limit_gb,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "baseline_reproducible": p.baseline_reproducible is True,
        "model_and_precision_pinned": p.model_and_precision_pinned is True,
        "global_batch_comparable": p.global_batch_comparable is True,
        "dataset_manifest_pinned": p.dataset_manifest_pinned is True,
        "rank_topology_verified": p.rank_topology_verified is True,
        "gpu_nic_numa_affinity_verified": p.gpu_nic_numa_affinity_verified is True,
        "nvlink_topology_verified": p.nvlink_topology_verified is True,
        "fabric_capacity_verified": p.fabric_capacity_verified is True,
        "nccl_collectives_profiled": p.nccl_collectives_profiled is True,
        "straggler_rank_instrumented": p.straggler_rank_instrumented is True,
        "data_loader_profiled": p.data_loader_profiled is True,
        "object_storage_profiled": p.object_storage_profiled is True,
        "checkpoint_io_profiled": p.checkpoint_io_profiled is True,
        "gpu_memory_headroom_verified": p.gpu_memory_headroom_verified is True,
        "cpu_numa_headroom_verified": p.cpu_numa_headroom_verified is True,
        "power_thermal_limits_verified": p.power_thermal_limits_verified is True,
        "slo_dashboard_verified": p.slo_dashboard_verified is True,
        "telemetry_freshness_verified": p.telemetry_freshness_verified is True,
        "regression_canary_verified": p.regression_canary_verified is True,
        "cost_per_token_reviewed": p.cost_per_token_reviewed is True,
        "n_minus_one_tested": p.n_minus_one_tested is True,
        "rollback_verified": p.rollback_verified is True,
        "incident_drills_completed": p.incident_drills_completed is True,
        "security_reviewed": p.security_reviewed is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "scaling_efficiency": round(efficiency, 4),
            "required_checkpoint_gbps": round(p.checkpoint_gb / p.checkpoint_window_s, 4)}

def example():
    return Plan("synthetic-multi-node-training", 4, 8, 32,
                100.0, 2400.0, 0.70,
                200.0, 250.0, 50.0, 70.0,
                20.0, 30.0, 10.0, 20.0,
                160.0, 200.0, 2400.0, 300.0, 10.0,
                60.0, 80.0, 10.0, 120.0,
                *([True] * 25))

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_multi_node_performance_policy.py`

```python
from dataclasses import replace
import pytest
from multi_node_performance_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_job():
    assert evaluate(replace(example(), job=" "))["reason"] == "missing_job"

def test_bool_count():
    assert evaluate(replace(example(), nodes=True))["reason"] == "invalid_count"

def test_fractional_count():
    assert evaluate(replace(example(), world_size=1.5))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), step_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), surviving_fabric_gbps=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), step_slo_ms=True))["reason"] == "invalid_metric"

def test_efficiency_threshold():
    assert evaluate(replace(example(), minimum_scaling_efficiency=1.1))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 35

def test_efficiency_math():
    assert evaluate(example())["scaling_efficiency"] == 0.75

def test_checkpoint_math():
    assert evaluate(example())["required_checkpoint_gbps"] == 8.0

@pytest.mark.parametrize("field,gate,value", [
    ("world_size", "rank_geometry", 16),
    ("observed_samples_per_second", "scaling_efficiency", 1000.0),
    ("step_p95_ms", "step_slo", 251.0),
    ("collective_p95_ms", "collective_slo", 71.0),
    ("loader_wait_p95_ms", "loader_wait_slo", 31.0),
    ("rank_skew_p95_ms", "rank_skew_slo", 21.0),
    ("surviving_fabric_gbps", "degraded_fabric", 100.0),
    ("checkpoint_write_gbps", "checkpoint_bandwidth", 7.0),
    ("gpu_memory_used_gb", "gpu_memory_headroom", 80.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field", [
    "baseline_reproducible",
    "model_and_precision_pinned",
    "global_batch_comparable",
    "dataset_manifest_pinned",
    "rank_topology_verified",
    "gpu_nic_numa_affinity_verified",
    "nvlink_topology_verified",
    "fabric_capacity_verified",
    "nccl_collectives_profiled",
    "straggler_rank_instrumented",
    "data_loader_profiled",
    "object_storage_profiled",
    "checkpoint_io_profiled",
    "gpu_memory_headroom_verified",
    "cpu_numa_headroom_verified",
    "power_thermal_limits_verified",
    "slo_dashboard_verified",
    "telemetry_freshness_verified",
    "regression_canary_verified",
    "cost_per_token_reviewed",
    "n_minus_one_tested",
    "rollback_verified",
    "incident_drills_completed",
    "security_reviewed",
    "independent_review_approved",
])
def test_boolean_gate(field):
    assert field in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run locally:

```bash
python -m pip install pytest
python multi_node_performance_policy.py
python -m pytest -q
```

**47 planned pytest cases:** 12 standalone + 10 numeric + 25 Boolean. Runtime execution, real GPU profiling and independent review are pending.

## Ten synthetic incident exercises

1. NIC degradation and all-reduce p95 breach; evaluate gate impact, containment and rollback.
2. NCCL fallback to TCP; evaluate gate impact, containment and rollback.
3. GPU/NIC NUMA misplacement; evaluate gate impact, containment and rollback.
4. Data loader backpressure; evaluate gate impact, containment and rollback.
5. Object store throttling; evaluate gate impact, containment and rollback.
6. Checkpoint network contention; evaluate gate impact, containment and rollback.
7. Thermal throttling; evaluate gate impact, containment and rollback.
8. Rank straggler injection; evaluate gate impact, containment and rollback.
9. N-1 GPU/node capacity loss; evaluate gate impact, containment and rollback.
10. Observability outage during performance regression; evaluate gate impact, containment and rollback.

## 100-point acceptance

Baseline/scaling 20; fabric/topology 20; storage/checkpoint 20; SLO/security/FinOps 20; tests/staging/review 20. Require >=90 and no critical blockers; real multi-node evidence required for production.
