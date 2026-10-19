# Lab 048 — Network Congestion and Packet Loss Gate

**Python 3.11+ · pytest · offline only.** No network mutation, GPU provisioning or packet injection.

## Goal

Validate normal/N-1 capacity, queue drops, physical errors, p95 collective latency, telemetry freshness, checkpoint RPO, congestion-control design, fabric security, operator ownership and rollback. Distinguish counters from attestations: these synthetic values are **not live measurements**.

## File: `network_congestion_policy.py`

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
    failed_uplinks: int
    max_degraded_ratio: float
    collective_p95_ms: float
    collective_slo_ms: float
    queue_drops: int
    max_queue_drops: int
    crc_errors: int
    max_crc_errors: int
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    physical_links_verified: bool
    ecpm_path_diversity_verified: bool
    mtu_consistent: bool
    qos_mapping_verified: bool
    congestion_policy_verified: bool
    pfc_pause_storm_absent: bool
    gpu_health_verified: bool
    nccl_transport_verified: bool
    cni_ip_headroom_verified: bool
    tenant_network_isolation_verified: bool
    checkpoint_restore_tested: bool
    workload_slo_verified: bool
    incident_runbook_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    if p.fabric not in ("roce", "infiniband", "efa", "tcp"):
        return {"accepted": False, "reason": "invalid_fabric"}
    ints = (p.hosts, p.host_nic_gbps, p.uplinks, p.uplink_gbps, p.failed_uplinks,
            p.queue_drops, p.max_queue_drops, p.crc_errors, p.max_crc_errors)
    if (any(type(v) is not int or v < 0 for v in ints)
            or min(p.hosts, p.host_nic_gbps, p.uplinks, p.uplink_gbps) == 0
            or p.failed_uplinks >= p.uplinks):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.max_degraded_ratio, p.collective_p95_ms, p.collective_slo_ms,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.max_degraded_ratio, p.collective_slo_ms,
                   p.checkpoint_rpo_min, p.telemetry_max_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    degraded_capacity = (p.uplinks - p.failed_uplinks) * p.uplink_gbps
    degraded_ratio = p.hosts * p.host_nic_gbps / degraded_capacity
    gates = {
        "n_minus_one": degraded_ratio <= p.max_degraded_ratio,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "queue_drops": p.queue_drops <= p.max_queue_drops,
        "crc_errors": p.crc_errors <= p.max_crc_errors,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry": p.telemetry_age_s <= p.telemetry_max_age_s,
        "physical_links": p.physical_links_verified is True,
        "ecmp": p.ecpm_path_diversity_verified is True,
        "mtu": p.mtu_consistent is True,
        "qos": p.qos_mapping_verified is True,
        "congestion_policy": p.congestion_policy_verified is True,
        "pause_storm": p.pfc_pause_storm_absent is True,
        "gpu_health": p.gpu_health_verified is True,
        "nccl_transport": p.nccl_transport_verified is True,
        "cni": p.cni_ip_headroom_verified is True,
        "tenant_isolation": p.tenant_network_isolation_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "workload_slo": p.workload_slo_verified is True,
        "incident_runbook": p.incident_runbook_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "degraded_oversubscription": round(degraded_ratio, 4)}

def example():
    return Plan(
        "synthetic-ai-fabric", "roce", 8, 200, 5, 400, 1, 1.2,
        2.0, 5.0, 0, 0, 0, 0, 5.0, 30.0, 10.0, 120.0,
        *([True] * 15)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_network_congestion_policy.py`

```python
from dataclasses import replace
import pytest
from network_congestion_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_empty_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_bad_fabric():
    assert evaluate(replace(example(), fabric="invalid"))["reason"] == "invalid_fabric"

def test_bool_count():
    assert evaluate(replace(example(), hosts=True))["reason"] == "invalid_count"

def test_fractional_count():
    assert evaluate(replace(example(), uplinks=1.5))["reason"] == "invalid_count"

def test_no_uplinks():
    assert evaluate(replace(example(), failed_uplinks=5))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_infinite():
    assert evaluate(replace(example(), max_degraded_ratio=float("inf")))["reason"] == "invalid_metric"

def test_nonboolean():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 21

@pytest.mark.parametrize("field,gate,value", [
    ("hosts", "n_minus_one", 12),
    ("collective_p95_ms", "collective_slo", 6.0),
    ("queue_drops", "queue_drops", 1),
    ("crc_errors", "crc_errors", 1),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("telemetry_age_s", "telemetry", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("physical_links_verified", "physical_links"),
    ("ecpm_path_diversity_verified", "ecmp"),
    ("mtu_consistent", "mtu"),
    ("qos_mapping_verified", "qos"),
    ("congestion_policy_verified", "congestion_policy"),
    ("pfc_pause_storm_absent", "pause_storm"),
    ("gpu_health_verified", "gpu_health"),
    ("nccl_transport_verified", "nccl_transport"),
    ("cni_ip_headroom_verified", "cni"),
    ("tenant_network_isolation_verified", "tenant_isolation"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("workload_slo_verified", "workload_slo"),
    ("incident_runbook_verified", "incident_runbook"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run offline:

```bash
python -m pip install pytest
python network_congestion_policy.py
python -m pytest -q
```

**Planned 32 pytest cases:** 11 standalone + 6 numeric + 15 boolean. Test execution and live fabric acceptance are pending.

## Ten incident exercises

1. Link/uplink loss violates N-1.
2. NCCL p95 breaches SLO.
3. Queue drops during collective incast.
4. CRC/FEC physical link errors.
5. Checkpoint RPO violation.
6. Telemetry staleness.
7. QoS/ECN/PFC design drift.
8. GPU/NCCL or CNI failure.
9. Tenant isolation breach.
10. Missing change approval or rollback.

## 100-point rubric

Topology/capacity 20; congestion diagnosis 20; GPU/Kubernetes/security 20; incidents/DR 20; tests/staging 20. Require >=90, zero critical blockers and independently reviewed evidence.
