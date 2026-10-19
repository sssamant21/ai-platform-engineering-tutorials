# Lab 043 — AI Cluster Network Architecture Gate

**Python 3.11+ · pytest · offline synthetic evidence.** No cloud API calls, packet injection, cluster mutation or spend.

## Objective

Validate network topology, N-1 uplink bandwidth, oversubscription, GPU/NIC affinity, fabric driver and collective readiness, isolation, checkpointing, latency, telemetry and approved rollback. The gate consumes attestations and synthetic metrics; it does **not** independently inspect live switches or GPUs.

## File: `ai_cluster_network_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class FabricPlan:
    cluster: str
    fabric: str
    hosts: int
    host_nic_gbps: int
    uplinks: int
    uplink_gbps: int
    failed_uplinks_budget: int
    max_oversubscription: float
    collective_p95_ms: float
    collective_p95_slo_ms: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    topology_verified: bool
    gpu_nic_numa_verified: bool
    nic_firmware_driver_verified: bool
    fabric_provider_verified: bool
    collective_transport_verified: bool
    rank_placement_verified: bool
    network_security_verified: bool
    tenant_isolation_verified: bool
    cni_ip_headroom_verified: bool
    congestion_controls_verified: bool
    gpu_health_verified: bool
    storage_bandwidth_verified: bool
    checkpoint_restore_tested: bool
    workload_slo_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, FabricPlan):
        raise TypeError("FabricPlan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    if p.fabric not in ("ethernet", "efa", "infiniband", "roce"):
        return {"accepted": False, "reason": "invalid_fabric"}
    counts = (p.hosts, p.host_nic_gbps, p.uplinks, p.uplink_gbps,
              p.failed_uplinks_budget)
    if (any(type(v) is not int or v < 0 for v in counts)
            or min(p.hosts, p.host_nic_gbps, p.uplinks, p.uplink_gbps) == 0
            or p.failed_uplinks_budget >= p.uplinks):
        return {"accepted": False, "reason": "invalid_topology"}
    metrics = (p.max_oversubscription, p.collective_p95_ms,
               p.collective_p95_slo_ms, p.telemetry_age_s,
               p.telemetry_max_age_s, p.checkpoint_age_min, p.checkpoint_rpo_min)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0
            for v in metrics)
            or min(p.max_oversubscription, p.collective_p95_slo_ms,
                   p.telemetry_max_age_s, p.checkpoint_rpo_min) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    host_bw = p.hosts * p.host_nic_gbps
    uplink_bw = p.uplinks * p.uplink_gbps
    n_minus_one_bw = (p.uplinks - p.failed_uplinks_budget) * p.uplink_gbps
    ratio = host_bw / uplink_bw
    degraded_ratio = host_bw / n_minus_one_bw
    gates = {
        "normal_oversubscription": ratio <= p.max_oversubscription,
        "degraded_oversubscription": degraded_ratio <= p.max_oversubscription,
        "collective_latency": p.collective_p95_ms <= p.collective_p95_slo_ms,
        "telemetry": p.telemetry_age_s <= p.telemetry_max_age_s,
        "checkpoint": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "topology": p.topology_verified is True,
        "gpu_nic_numa": p.gpu_nic_numa_verified is True,
        "nic_driver": p.nic_firmware_driver_verified is True,
        "fabric_provider": p.fabric_provider_verified is True,
        "collective_transport": p.collective_transport_verified is True,
        "rank_placement": p.rank_placement_verified is True,
        "network_security": p.network_security_verified is True,
        "tenant_isolation": p.tenant_isolation_verified is True,
        "cni_ip_headroom": p.cni_ip_headroom_verified is True,
        "congestion_controls": p.congestion_controls_verified is True,
        "gpu_health": p.gpu_health_verified is True,
        "storage_bandwidth": p.storage_bandwidth_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "workload_slo": p.workload_slo_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(name for name, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "normal_oversubscription": round(ratio, 4),
            "degraded_oversubscription": round(degraded_ratio, 4),
            "usable_n_minus_one_gbps": n_minus_one_bw}

def example():
    return FabricPlan(
        "synthetic-ai-fabric", "ethernet",
        8, 200, 5, 400, 1, 1.2,
        2.0, 5.0, 10.0, 120.0, 5.0, 30.0,
        *([True] * 16)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_ai_cluster_network_policy.py`

```python
from dataclasses import replace
import pytest
from ai_cluster_network_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_bad_fabric():
    assert evaluate(replace(example(), fabric="unknown"))["reason"] == "invalid_fabric"

def test_bool_hosts():
    assert evaluate(replace(example(), hosts=True))["reason"] == "invalid_topology"

def test_fractional_uplinks():
    assert evaluate(replace(example(), uplinks=1.5))["reason"] == "invalid_topology"

def test_no_uplink_after_failure():
    assert evaluate(replace(example(), failed_uplinks_budget=5))["reason"] == "invalid_topology"

def test_nan_latency():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_negative_checkpoint():
    assert evaluate(replace(example(), checkpoint_age_min=-1))["reason"] == "invalid_metric"

def test_normal_oversubscription():
    assert "normal_oversubscription" in evaluate(replace(example(), hosts=15))["failed_gates"]

def test_degraded_oversubscription():
    assert "degraded_oversubscription" in evaluate(replace(example(), max_oversubscription=0.9))["failed_gates"]

def test_collective_latency():
    assert "collective_latency" in evaluate(replace(example(), collective_p95_ms=6))["failed_gates"]

def test_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_checkpoint():
    assert "checkpoint" in evaluate(replace(example(), checkpoint_age_min=31))["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("topology_verified", "topology"),
    ("gpu_nic_numa_verified", "gpu_nic_numa"),
    ("nic_firmware_driver_verified", "nic_driver"),
    ("fabric_provider_verified", "fabric_provider"),
    ("collective_transport_verified", "collective_transport"),
    ("rank_placement_verified", "rank_placement"),
    ("network_security_verified", "network_security"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("cni_ip_headroom_verified", "cni_ip_headroom"),
    ("congestion_controls_verified", "congestion_controls"),
    ("gpu_health_verified", "gpu_health"),
    ("storage_bandwidth_verified", "storage_bandwidth"),
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
python ai_cluster_network_policy.py
python -m pytest -q
```

**Planned 31 pytest cases:** 15 standalone and 16 parameterized. Live fabric validation is pending.

## Ten synthetic incident drills

1. Normal fabric oversubscription.
2. Uplink failure violates N-1 budget.
3. NCCL p95 exceeds SLO.
4. NIC firmware/driver or NUMA mismatch.
5. EFA/InfiniBand/RoCE provider unavailable.
6. Rank placement or transport mismatch.
7. Network security/tenant isolation failure.
8. CNI IP exhaustion or congestion-control regression.
9. Storage checkpoint RPO violation.
10. Stale telemetry, change approval or rollback.

## Staging evidence checklist

Collect approved topology diagram, NIC/switch inventory, uplink capacities, N-1 measurements, GPU/NIC/NUMA mapping, EFA/InfiniBand/RoCE version and transport logs where applicable, NCCL baseline, packet/drop counters, security controls, checkpoint recovery, SLO and rollback evidence. Use synthetic data for fault injection until staging is approved.

**Limitation:** Boolean attestations do not independently verify network state or prove production SLO.

## 100-point assessment

Architecture 20; transport/topology 20; security/resilience 20; incidents/recovery 20; tests/staging 20. Pass >=90 and zero critical blockers.
