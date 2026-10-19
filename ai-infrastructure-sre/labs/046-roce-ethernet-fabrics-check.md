# Lab 046 — RoCE and Ethernet Fabrics Gate

**Python 3.11+ · pytest · offline synthetic evidence.** No live switch configuration, packet injection, GPU provisioning or cloud spend.

## Objective

Fail closed when RoCEv2 NIC/fabric readiness, QoS mapping, ECN/PFC design, N-1 bandwidth, NCCL, security, checkpoint, telemetry or change governance is missing. This gate evaluates synthetic metrics and declared attestations; it does not prove live switch configuration.

## File: `roce_fabric_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    fabric: str
    hosts: int
    host_nic_gbps: int
    uplinks: int
    uplink_gbps: int
    failed_uplinks_budget: int
    max_degraded_oversubscription: float
    collective_p95_ms: float
    collective_slo_ms: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    nic_driver_verified: bool
    roce_v2_path_verified: bool
    mtu_consistent: bool
    qos_priority_mapping_verified: bool
    ecn_marking_verified: bool
    congestion_control_verified: bool
    pfc_scope_reviewed: bool
    pause_storm_absent: bool
    switch_queue_telemetry_verified: bool
    ecpm_path_diversity_verified: bool
    gpu_hca_numa_verified: bool
    rdma_device_plugin_verified: bool
    nccl_transport_verified: bool
    tenant_isolation_verified: bool
    network_acl_verified: bool
    checkpoint_restore_tested: bool
    workload_slo_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    if p.fabric != "roce-v2":
        return {"accepted": False, "reason": "invalid_fabric"}
    counts = (p.hosts, p.host_nic_gbps, p.uplinks, p.uplink_gbps, p.failed_uplinks_budget)
    if (any(type(x) is not int or x < 0 for x in counts)
            or min(p.hosts, p.host_nic_gbps, p.uplinks, p.uplink_gbps) == 0
            or p.failed_uplinks_budget >= p.uplinks):
        return {"accepted": False, "reason": "invalid_topology"}
    metrics = (p.max_degraded_oversubscription, p.collective_p95_ms,
               p.collective_slo_ms, p.telemetry_age_s, p.telemetry_max_age_s,
               p.checkpoint_age_min, p.checkpoint_rpo_min)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or min(p.max_degraded_oversubscription, p.collective_slo_ms,
                   p.telemetry_max_age_s, p.checkpoint_rpo_min) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    degraded_gbps = (p.uplinks - p.failed_uplinks_budget) * p.uplink_gbps
    degraded_ratio = p.hosts * p.host_nic_gbps / degraded_gbps
    gates = {
        "n_minus_one": degraded_ratio <= p.max_degraded_oversubscription,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "telemetry": p.telemetry_age_s <= p.telemetry_max_age_s,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "nic_driver": p.nic_driver_verified is True,
        "roce_path": p.roce_v2_path_verified is True,
        "mtu": p.mtu_consistent is True,
        "qos": p.qos_priority_mapping_verified is True,
        "ecn": p.ecn_marking_verified is True,
        "congestion_control": p.congestion_control_verified is True,
        "pfc_scope": p.pfc_scope_reviewed is True,
        "pause_storm": p.pause_storm_absent is True,
        "queue_telemetry": p.switch_queue_telemetry_verified is True,
        "ecmp": p.ecpm_path_diversity_verified is True,
        "gpu_numa": p.gpu_hca_numa_verified is True,
        "rdma_plugin": p.rdma_device_plugin_verified is True,
        "nccl_transport": p.nccl_transport_verified is True,
        "tenant_isolation": p.tenant_isolation_verified is True,
        "network_acl": p.network_acl_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "workload_slo": p.workload_slo_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, v in gates.items() if not v)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates), "degraded_oversubscription": round(degraded_ratio, 4),
            "surviving_uplink_gbps": degraded_gbps}

def example():
    return Plan(
        "synthetic-roce-cluster", "roce-v2",
        8, 200, 5, 400, 1, 1.2,
        2.0, 5.0, 10.0, 120.0, 5.0, 30.0,
        *([True] * 19)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_roce_fabric_policy.py`

```python
from dataclasses import replace
import pytest
from roce_fabric_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_wrong_fabric():
    assert evaluate(replace(example(), fabric="efa"))["reason"] == "invalid_fabric"

def test_bool_host_count():
    assert evaluate(replace(example(), hosts=True))["reason"] == "invalid_topology"

def test_fractional_uplinks():
    assert evaluate(replace(example(), uplinks=1.5))["reason"] == "invalid_topology"

def test_no_surviving_uplink():
    assert evaluate(replace(example(), failed_uplinks_budget=5))["reason"] == "invalid_topology"

def test_nan_latency():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_inf_ratio():
    assert evaluate(replace(example(), max_degraded_oversubscription=float("inf")))["reason"] == "invalid_metric"

def test_nonboolean_change():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 23

@pytest.mark.parametrize("field,gate,value", [
    ("hosts", "n_minus_one", 12),
    ("collective_p95_ms", "collective_slo", 6),
    ("telemetry_age_s", "telemetry", 121),
    ("checkpoint_age_min", "checkpoint_rpo", 31),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("nic_driver_verified", "nic_driver"),
    ("roce_v2_path_verified", "roce_path"),
    ("mtu_consistent", "mtu"),
    ("qos_priority_mapping_verified", "qos"),
    ("ecn_marking_verified", "ecn"),
    ("congestion_control_verified", "congestion_control"),
    ("pfc_scope_reviewed", "pfc_scope"),
    ("pause_storm_absent", "pause_storm"),
    ("switch_queue_telemetry_verified", "queue_telemetry"),
    ("ecpm_path_diversity_verified", "ecmp"),
    ("gpu_hca_numa_verified", "gpu_numa"),
    ("rdma_device_plugin_verified", "rdma_plugin"),
    ("nccl_transport_verified", "nccl_transport"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("network_acl_verified", "network_acl"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("workload_slo_verified", "workload_slo"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_required_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python roce_fabric_policy.py
python -m pytest -q
```

**Planned 34 pytest cases:** 11 standalone, 4 numeric and 19 boolean. Live fabric validation remains pending.

## Ten synthetic incident drills

1. RoCE path or NIC driver missing.
2. MTU or QoS mapping drift.
3. ECN marking or congestion-control regression.
4. PFC pause storm.
5. Switch queue telemetry missing.
6. ECMP path diversity failure.
7. GPU/NIC affinity or RDMA plugin failure.
8. NCCL transport fallback.
9. N-1 uplink loss or collective SLO violation.
10. Checkpoint, tenant security, telemetry or rollback failure.

## Evidence and 100-point acceptance

Collect switch/NIC firmware and queue policy, DSCP/PCP mapping, ECN/PFC counters, fabric topology and N-1 calculations, RDMA plugin allocatable, NCCL benchmark, tenant ACLs, checkpoint restore, incident timeline, security review and approved rollback. Score architecture 20; QoS/congestion 20; GPU/Kubernetes/security 20; incidents/recovery 20; tests/staging 20. Pass >=90 and no critical blockers. Synthetic attestations alone do not certify production.
