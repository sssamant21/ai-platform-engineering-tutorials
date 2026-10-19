# Lab 044 — RDMA and InfiniBand Fundamentals Gate

**Python 3.11+ · pytest · offline synthetic evidence.** No fabric mutation, GPU launch or live packet injection.

## Goal

Fail closed on missing HCA/driver, native IB subnet-manager and partition readiness, RDMA verbs resources, GPU–HCA locality, GPUDirect compatibility when required, NCCL transport, fabric N-1 bandwidth, latency, checkpoint RPO, security, telemetry, change approval and rollback. EFA and RoCE do **not** require a native InfiniBand subnet manager.

## File: `rdma_infiniband_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class RDMAPlan:
    cluster: str
    fabric: str
    hosts: int
    host_nic_gbps: int
    uplinks: int
    uplink_gbps: int
    failed_uplinks_budget: int
    max_degraded_oversubscription: float
    collective_p95_ms: float
    collective_p95_slo_ms: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    gpudirect_required: bool
    hca_detected: bool
    hca_driver_healthy: bool
    port_active: bool
    subnet_manager_verified: bool
    partition_membership_verified: bool
    verbs_pd_mr_qp_cq_verified: bool
    path_resolution_verified: bool
    gpu_hca_numa_verified: bool
    gpudirect_compatible: bool
    nccl_transport_verified: bool
    multi_rail_failover_verified: bool
    network_security_verified: bool
    tenant_isolation_verified: bool
    checkpoint_restore_tested: bool
    workload_slo_verified: bool
    audit_telemetry_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, RDMAPlan):
        raise TypeError("RDMAPlan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    if p.fabric not in ("infiniband", "roce", "efa"):
        return {"accepted": False, "reason": "invalid_fabric"}
    if type(p.gpudirect_required) is not bool:
        return {"accepted": False, "reason": "invalid_gpudirect_flag"}
    counts = (p.hosts, p.host_nic_gbps, p.uplinks, p.uplink_gbps,
              p.failed_uplinks_budget)
    if (any(type(v) is not int or v < 0 for v in counts)
            or min(p.hosts, p.host_nic_gbps, p.uplinks, p.uplink_gbps) == 0
            or p.failed_uplinks_budget >= p.uplinks):
        return {"accepted": False, "reason": "invalid_topology"}
    metrics = (p.max_degraded_oversubscription, p.collective_p95_ms,
               p.collective_p95_slo_ms, p.checkpoint_age_min,
               p.checkpoint_rpo_min, p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0
            for v in metrics)
            or min(p.max_degraded_oversubscription, p.collective_p95_slo_ms,
                   p.checkpoint_rpo_min, p.telemetry_max_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    degraded_bw = (p.uplinks - p.failed_uplinks_budget) * p.uplink_gbps
    degraded_ratio = p.hosts * p.host_nic_gbps / degraded_bw
    native_ib = p.fabric == "infiniband"
    gates = {
        "n_minus_one_bandwidth": degraded_ratio <= p.max_degraded_oversubscription,
        "collective_latency": p.collective_p95_ms <= p.collective_p95_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "hca_detected": p.hca_detected is True,
        "hca_driver": p.hca_driver_healthy is True,
        "port_active": p.port_active is True,
        "subnet_manager": not native_ib or p.subnet_manager_verified is True,
        "ib_partition": not native_ib or p.partition_membership_verified is True,
        "verbs_resources": p.verbs_pd_mr_qp_cq_verified is True,
        "path_resolution": p.path_resolution_verified is True,
        "gpu_hca_numa": p.gpu_hca_numa_verified is True,
        "gpudirect": not p.gpudirect_required or p.gpudirect_compatible is True,
        "nccl_transport": p.nccl_transport_verified is True,
        "multi_rail": p.multi_rail_failover_verified is True,
        "network_security": p.network_security_verified is True,
        "tenant_isolation": p.tenant_isolation_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "workload_slo": p.workload_slo_verified is True,
        "audit": p.audit_telemetry_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(name for name, passed in gates.items() if not passed)
    return {"accepted": not failed, "failed_gates": failed,
            "degraded_oversubscription": round(degraded_ratio, 4),
            "n_minus_one_gbps": degraded_bw, "total_gates": len(gates)}

def example():
    return RDMAPlan(
        "synthetic-ib-cluster", "infiniband",
        8, 200, 5, 400, 1, 1.2,
        2.0, 5.0, 5.0, 30.0, 10.0, 120.0,
        True, *([True] * 18)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_rdma_infiniband_policy.py`

```python
from dataclasses import replace
import pytest
from rdma_infiniband_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_bad_fabric():
    assert evaluate(replace(example(), fabric="tcp"))["reason"] == "invalid_fabric"

def test_bool_as_host_count():
    assert evaluate(replace(example(), hosts=True))["reason"] == "invalid_topology"

def test_no_surviving_uplink():
    assert evaluate(replace(example(), failed_uplinks_budget=5))["reason"] == "invalid_topology"

def test_nan_latency():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_negative_checkpoint():
    assert evaluate(replace(example(), checkpoint_age_min=-1))["reason"] == "invalid_metric"

def test_invalid_gpudirect_flag():
    assert evaluate(replace(example(), gpudirect_required="yes"))["reason"] == "invalid_gpudirect_flag"

def test_n_minus_one():
    assert "n_minus_one_bandwidth" in evaluate(replace(example(), hosts=12))["failed_gates"]

def test_latency():
    assert "collective_latency" in evaluate(replace(example(), collective_p95_ms=6))["failed_gates"]

def test_checkpoint():
    assert "checkpoint_rpo" in evaluate(replace(example(), checkpoint_age_min=31))["failed_gates"]

def test_telemetry():
    assert "telemetry_freshness" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_roce_does_not_require_ib_sm():
    p = replace(example(), fabric="roce", subnet_manager_verified=False, partition_membership_verified=False)
    assert evaluate(p)["accepted"]

def test_efa_does_not_require_ib_sm():
    p = replace(example(), fabric="efa", subnet_manager_verified=False, partition_membership_verified=False)
    assert evaluate(p)["accepted"]

def test_optional_gpudirect():
    p = replace(example(), gpudirect_required=False, gpudirect_compatible=False)
    assert evaluate(p)["accepted"]

def test_nonboolean_change():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 22

@pytest.mark.parametrize("field,gate", [
    ("hca_detected", "hca_detected"),
    ("hca_driver_healthy", "hca_driver"),
    ("port_active", "port_active"),
    ("subnet_manager_verified", "subnet_manager"),
    ("partition_membership_verified", "ib_partition"),
    ("verbs_pd_mr_qp_cq_verified", "verbs_resources"),
    ("path_resolution_verified", "path_resolution"),
    ("gpu_hca_numa_verified", "gpu_hca_numa"),
    ("gpudirect_compatible", "gpudirect"),
    ("nccl_transport_verified", "nccl_transport"),
    ("multi_rail_failover_verified", "multi_rail"),
    ("network_security_verified", "network_security"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("workload_slo_verified", "workload_slo"),
    ("audit_telemetry_verified", "audit"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_critical_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run in a disposable Python environment:

```bash
python -m pip install pytest
python rdma_infiniband_policy.py
python -m pytest -q
```

**Planned: 36 pytest cases** (18 standalone + 18 parameterized). Test execution and live RDMA/IB staging remain pending.

## Ten synthetic incident drills

1. HCA driver failure.
2. Port Down or wrong native IB SM.
3. P_Key mismatch.
4. QP/CQ or registered-memory exhaustion.
5. Path resolution failure.
6. GPU–HCA NUMA regression.
7. Unsupported GPUDirect path.
8. NCCL transport fallback.
9. Rail loss breaches N-1.
10. Stale telemetry, checkpoint failure or missing rollback.

## Evidence and assessment

Capture HCA/firmware, SM/partition and port state for native IB, verbs device inventory, PCIe/NUMA map, GPUDirect compatibility evidence, NCCL baseline, link counters, multi-rail failure test, checkpoint RPO, SLO and security review. All evidence must be redacted and approved.

**100 points:** RDMA fundamentals 20; IB topology 20; GPU/Kubernetes/security 20; incident recovery 20; tests/staging 20. Pass >=90 and zero critical blockers. Synthetic attestations are not production authorization.
