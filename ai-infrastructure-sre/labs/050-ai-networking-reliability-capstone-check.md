# Lab 050 — AI Networking Reliability Capstone Gate

**Python 3.11+ · pytest · offline synthetic acceptance.** Does not launch GPU workloads, modify switches or call AWS/Kubernetes APIs.

## Goal

Provide a fail-closed, transport-aware readiness gate for IB/RoCE/EFA, NCCL, N-1 GPU/network capacity, telemetry, checkpoint recovery, security and rollback. Synthetic booleans are scenario inputs, not proof of actual infrastructure.

## File: `ai_network_capstone_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    transport: str
    nodes: int
    gpus_per_node: int
    required_gpus: int
    surviving_gpus: int
    nominal_fabric_gbps: int
    surviving_fabric_gbps: int
    required_fabric_gbps: int
    collective_p95_ms: float
    collective_slo_ms: float
    measured_bandwidth_gbps: float
    minimum_bandwidth_gbps: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    transport_specific_fabric_verified: bool
    hca_nic_health_verified: bool
    gpu_health_verified: bool
    driver_cuda_nccl_compatible: bool
    gpu_nic_numa_verified: bool
    gpudirect_path_reviewed: bool
    nccl_effective_transport_verified: bool
    collective_correctness_verified: bool
    no_unapproved_fallback: bool
    congestion_counters_reviewed: bool
    kubernetes_gpu_rdma_resources_verified: bool
    tenant_isolation_verified: bool
    network_access_controls_verified: bool
    checkpoint_restore_tested: bool
    workload_slo_verified: bool
    incident_runbooks_tested: bool
    security_review_approved: bool
    change_approved: bool
    rollback_tested: bool
    independent_reviewer_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    if p.transport not in ("infiniband", "roce", "efa"):
        return {"accepted": False, "reason": "invalid_transport"}
    ints = (p.nodes, p.gpus_per_node, p.required_gpus, p.surviving_gpus,
            p.nominal_fabric_gbps, p.surviving_fabric_gbps, p.required_fabric_gbps)
    if (any(type(v) is not int or v < 0 for v in ints)
            or min(p.nodes, p.gpus_per_node, p.required_gpus,
                   p.nominal_fabric_gbps, p.required_fabric_gbps) == 0
            or p.surviving_fabric_gbps > p.nominal_fabric_gbps):
        return {"accepted": False, "reason": "invalid_capacity"}
    metrics = (p.collective_p95_ms, p.collective_slo_ms,
               p.measured_bandwidth_gbps, p.minimum_bandwidth_gbps,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.collective_slo_ms, p.minimum_bandwidth_gbps,
                   p.checkpoint_rpo_min, p.max_telemetry_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "gpu_capacity": p.required_gpus <= p.nodes * p.gpus_per_node,
        "n_minus_one_gpu": p.required_gpus <= p.surviving_gpus,
        "n_minus_one_fabric": p.surviving_fabric_gbps >= p.required_fabric_gbps,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "bandwidth_floor": p.measured_bandwidth_gbps >= p.minimum_bandwidth_gbps,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry_freshness": p.telemetry_age_s <= p.max_telemetry_age_s,
        "fabric": p.transport_specific_fabric_verified is True,
        "nic_health": p.hca_nic_health_verified is True,
        "gpu_health": p.gpu_health_verified is True,
        "compatibility": p.driver_cuda_nccl_compatible is True,
        "numa": p.gpu_nic_numa_verified is True,
        "gpudirect_review": p.gpudirect_path_reviewed is True,
        "nccl_transport": p.nccl_effective_transport_verified is True,
        "collective_correctness": p.collective_correctness_verified is True,
        "fallback": p.no_unapproved_fallback is True,
        "congestion": p.congestion_counters_reviewed is True,
        "kubernetes": p.kubernetes_gpu_rdma_resources_verified is True,
        "tenant_isolation": p.tenant_isolation_verified is True,
        "network_acl": p.network_access_controls_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "workload_slo": p.workload_slo_verified is True,
        "incident_runbooks": p.incident_runbooks_tested is True,
        "security_review": p.security_review_approved is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
        "independent_review": p.independent_reviewer_approved is True,
    }
    failed = sorted(k for k, passed in gates.items() if not passed)
    return {"accepted": not failed, "failed_gates": failed,
            "passed_gates": len(gates) - len(failed), "total_gates": len(gates)}

def example():
    return Plan(
        "synthetic-network-capstone", "roce",
        3, 8, 16, 16, 800, 600, 500,
        2.0, 5.0, 500.0, 400.0, 5.0, 30.0, 10.0, 120.0,
        *([True] * 20)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_ai_network_capstone_policy.py`

```python
from dataclasses import replace
import pytest
from ai_network_capstone_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_empty_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_invalid_transport():
    assert evaluate(replace(example(), transport="tcp"))["reason"] == "invalid_transport"

def test_bool_count():
    assert evaluate(replace(example(), nodes=True))["reason"] == "invalid_capacity"

def test_fractional_gpus():
    assert evaluate(replace(example(), required_gpus=1.5))["reason"] == "invalid_capacity"

def test_surviving_exceeds_nominal():
    assert evaluate(replace(example(), surviving_fabric_gbps=801))["reason"] == "invalid_capacity"

def test_nan_latency():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_infinite_bandwidth():
    assert evaluate(replace(example(), measured_bandwidth_gbps=float("inf")))["reason"] == "invalid_metric"

def test_nonboolean_signoff():
    assert "independent_review" in evaluate(replace(example(), independent_reviewer_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 27

@pytest.mark.parametrize("field,gate,value", [
    ("required_gpus", "gpu_capacity", 25),
    ("surviving_gpus", "n_minus_one_gpu", 15),
    ("surviving_fabric_gbps", "n_minus_one_fabric", 400),
    ("collective_p95_ms", "collective_slo", 6.0),
    ("measured_bandwidth_gbps", "bandwidth_floor", 399.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("transport_specific_fabric_verified", "fabric"),
    ("hca_nic_health_verified", "nic_health"),
    ("gpu_health_verified", "gpu_health"),
    ("driver_cuda_nccl_compatible", "compatibility"),
    ("gpu_nic_numa_verified", "numa"),
    ("gpudirect_path_reviewed", "gpudirect_review"),
    ("nccl_effective_transport_verified", "nccl_transport"),
    ("collective_correctness_verified", "collective_correctness"),
    ("no_unapproved_fallback", "fallback"),
    ("congestion_counters_reviewed", "congestion"),
    ("kubernetes_gpu_rdma_resources_verified", "kubernetes"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("network_access_controls_verified", "network_acl"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("workload_slo_verified", "workload_slo"),
    ("incident_runbooks_tested", "incident_runbooks"),
    ("security_review_approved", "security_review"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("independent_reviewer_approved", "independent_review"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run offline:

```bash
python -m pip install pytest
python ai_network_capstone_policy.py
python -m pytest -q
```

**Planned 38 pytest cases:** 11 standalone + 7 numeric + 20 boolean. Live GPU fabric, NCCL, failure injection and independent review remain pending.

## Ten synthetic failure drills

1. GPU or HCA health failure.
2. Transport mismatch or unapproved fallback.
3. NCCL collective correctness regression.
4. Driver/CUDA/NCCL incompatibility.
5. GPU/NIC NUMA mismatch.
6. Congestion and queue monitoring failure.
7. Kubernetes device resources or tenant isolation failure.
8. N-1 GPU/network capacity shortfall.
9. Checkpoint RPO/RTO or workload SLO breach.
10. Missing security review, change approval, rollback or independent signoff.

## Evidence and 100-point rubric

Produce a redacted manifest linking immutable versions, topology, transport logs, NCCL benchmark, queue metrics, N-1 failure test, checkpoint restore, security approval and change ticket. Score architecture/capacity 20; GPU/NCCL 20; Kubernetes/security 20; reliability/recovery 20; tests/evidence 20. Require >=90 and all critical gates. **Synthetic pass ≠ production acceptance.**
