# Lab 049 — NCCL Network Performance Troubleshooting Gate

**Python 3.11+ · pytest · offline synthetic acceptance.** This lab never invokes GPUs, cloud APIs, NCCL binaries or network fault injection.

## Goal

Fail closed on invalid counts and metrics, missing rank/transport correctness, GPU/NIC compatibility, provider configuration, N-1 capacity, checkpoint, SLO, security, change approval, telemetry and rollback. Synthetic boolean attestations must not be confused with real measured evidence.

## File: `nccl_network_policy.py`

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
    ranks: int
    surviving_gpus: int
    nominal_fabric_gbps: int
    surviving_fabric_gbps: int
    required_fabric_gbps: int
    collective_p95_ms: float
    collective_slo_ms: float
    bandwidth_gbps: float
    min_bandwidth_gbps: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    rank_rendezvous_verified: bool
    collective_order_verified: bool
    collective_correctness_verified: bool
    gpu_driver_cuda_nccl_verified: bool
    gpu_health_verified: bool
    nic_hca_healthy: bool
    gpu_nic_locality_verified: bool
    provider_plugin_verified: bool
    effective_transport_verified: bool
    no_unapproved_socket_fallback: bool
    network_path_verified: bool
    congestion_reviewed: bool
    kubernetes_device_allocation_verified: bool
    tenant_security_verified: bool
    checkpoint_restore_tested: bool
    workload_slo_verified: bool
    incident_runbook_tested: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    if p.transport not in ("infiniband", "roce", "efa", "tcp"):
        return {"accepted": False, "reason": "invalid_transport"}
    ints = (p.nodes, p.gpus_per_node, p.ranks, p.surviving_gpus,
            p.nominal_fabric_gbps, p.surviving_fabric_gbps, p.required_fabric_gbps)
    if (any(type(x) is not int or x < 0 for x in ints)
            or min(p.nodes, p.gpus_per_node, p.ranks, p.nominal_fabric_gbps,
                   p.required_fabric_gbps) == 0
            or p.surviving_fabric_gbps > p.nominal_fabric_gbps):
        return {"accepted": False, "reason": "invalid_capacity"}
    metrics = (p.collective_p95_ms, p.collective_slo_ms,
               p.bandwidth_gbps, p.min_bandwidth_gbps,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or min(p.collective_slo_ms, p.min_bandwidth_gbps,
                   p.checkpoint_rpo_min, p.max_telemetry_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "rank_capacity": p.ranks <= p.nodes * p.gpus_per_node,
        "n_minus_one_gpu": p.ranks <= p.surviving_gpus,
        "n_minus_one_fabric": p.surviving_fabric_gbps >= p.required_fabric_gbps,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "bandwidth_floor": p.bandwidth_gbps >= p.min_bandwidth_gbps,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry_freshness": p.telemetry_age_s <= p.max_telemetry_age_s,
        "rendezvous": p.rank_rendezvous_verified is True,
        "collective_order": p.collective_order_verified is True,
        "collective_correctness": p.collective_correctness_verified is True,
        "gpu_stack": p.gpu_driver_cuda_nccl_verified is True,
        "gpu_health": p.gpu_health_verified is True,
        "nic_health": p.nic_hca_healthy is True,
        "gpu_nic_locality": p.gpu_nic_locality_verified is True,
        "provider": p.provider_plugin_verified is True,
        "transport": p.effective_transport_verified is True,
        "fallback": p.no_unapproved_socket_fallback is True,
        "network_path": p.network_path_verified is True,
        "congestion": p.congestion_reviewed is True,
        "kubernetes": p.kubernetes_device_allocation_verified is True,
        "tenant_security": p.tenant_security_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "workload_slo": p.workload_slo_verified is True,
        "incident_runbook": p.incident_runbook_tested is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates), "passed_gates": len(gates) - len(failed)}

def example():
    return Plan(
        "synthetic-nccl-cluster", "roce",
        3, 8, 16, 16, 800, 600, 500,
        2.0, 5.0, 500.0, 400.0, 5.0, 30.0, 10.0, 120.0,
        *([True] * 19)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_nccl_network_policy.py`

```python
from dataclasses import replace
import pytest
from nccl_network_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_empty_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_invalid_transport():
    assert evaluate(replace(example(), transport="unknown"))["reason"] == "invalid_transport"

def test_bool_count():
    assert evaluate(replace(example(), nodes=True))["reason"] == "invalid_capacity"

def test_fractional_rank_count():
    assert evaluate(replace(example(), ranks=1.5))["reason"] == "invalid_capacity"

def test_invalid_surviving_capacity():
    assert evaluate(replace(example(), surviving_fabric_gbps=801))["reason"] == "invalid_capacity"

def test_nan_latency():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_infinite_bandwidth():
    assert evaluate(replace(example(), bandwidth_gbps=float("inf")))["reason"] == "invalid_metric"

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 26

@pytest.mark.parametrize("field,gate,value", [
    ("ranks", "rank_capacity", 25),
    ("surviving_gpus", "n_minus_one_gpu", 15),
    ("surviving_fabric_gbps", "n_minus_one_fabric", 400),
    ("collective_p95_ms", "collective_slo", 6.0),
    ("bandwidth_gbps", "bandwidth_floor", 399.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("rank_rendezvous_verified", "rendezvous"),
    ("collective_order_verified", "collective_order"),
    ("collective_correctness_verified", "collective_correctness"),
    ("gpu_driver_cuda_nccl_verified", "gpu_stack"),
    ("gpu_health_verified", "gpu_health"),
    ("nic_hca_healthy", "nic_health"),
    ("gpu_nic_locality_verified", "gpu_nic_locality"),
    ("provider_plugin_verified", "provider"),
    ("effective_transport_verified", "transport"),
    ("no_unapproved_socket_fallback", "fallback"),
    ("network_path_verified", "network_path"),
    ("congestion_reviewed", "congestion"),
    ("kubernetes_device_allocation_verified", "kubernetes"),
    ("tenant_security_verified", "tenant_security"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("workload_slo_verified", "workload_slo"),
    ("incident_runbook_tested", "incident_runbook"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run in an isolated Python environment:

```bash
python -m pip install pytest
python nccl_network_policy.py
python -m pytest -q
```

**Planned 37 pytest cases:** 11 standalone + 7 numeric + 19 boolean. Runtime execution and live NCCL staging evidence remain pending.

## Ten offline incident drills

1. Rank rendezvous failure.
2. Collective ordering or correctness failure.
3. GPU Xid, driver or CUDA/NCCL regression.
4. HCA or GPU/NIC locality issue.
5. Provider plugin failure.
6. Socket fallback.
7. Congestion or broken network path.
8. N-1 GPU/fabric shortfall.
9. Checkpoint, SLO or telemetry breach.
10. Missing tenant security, incident runbook or rollback.

## Evidence and 100-point assessment

Record rank mapping, job image digest, GPU UUIDs, driver/CUDA/NCCL/plugin versions, transport logs, `nccl-tests` results, link counters, p95/p99, N-1 drills, checkpoint restore, incident timeline, security approval and rollback. Score collectives/baselines 20; transport diagnosis 20; Kubernetes/security 20; resilience/incident response 20; tests/evidence 20. Require >=90 and zero critical blockers.
