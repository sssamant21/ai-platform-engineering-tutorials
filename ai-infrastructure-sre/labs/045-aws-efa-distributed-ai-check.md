# Lab 045 — AWS EFA for Distributed AI Gate

**Python 3.11+ · pytest · offline synthetic evidence.** No AWS credentials, EC2 launches, Kubernetes mutations or cloud spend.

## Objective

Fail closed on missing EFA SKU/ENI, driver/libfabric/plugin readiness, same-AZ placement, security-group peer rules, GPU/EFA device plugin, quota/capacity, N-1, NCCL correctness, checkpoint recovery, telemetry, SLO, cost and rollback.

## File: `aws_efa_distributed_policy.py`

```python
from dataclasses import dataclass, replace
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    region: str
    instance_type: str
    ranks: int
    gpus_per_node: int
    nodes: int
    requested_gpus: int
    n_minus_one_gpus: int
    vcpus_per_node: int
    quota_headroom_vcpus: int
    efa_interfaces_required: int
    efa_interfaces_available: int
    collective_p95_ms: float
    collective_slo_ms: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    estimated_cost_usd: float
    approved_budget_usd: float
    efa_sku_supported: bool
    efa_eni_attached: bool
    physical_capacity_verified: bool
    same_az_placement_verified: bool
    efa_driver_healthy: bool
    libfabric_efa_provider_verified: bool
    aws_ofi_nccl_verified: bool
    gpu_driver_cuda_verified: bool
    efa_device_plugin_verified: bool
    gpu_device_plugin_verified: bool
    security_group_peer_rules_verified: bool
    gpu_nic_topology_verified: bool
    nccl_transport_and_correctness_verified: bool
    checkpoint_restore_tested: bool
    workload_slo_verified: bool
    identity_tenant_security_verified: bool
    gpu_health_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if any(not isinstance(v, str) or not v.strip()
           for v in (p.cluster, p.region, p.instance_type)):
        return {"accepted": False, "reason": "missing_identity"}
    ints = (p.ranks, p.gpus_per_node, p.nodes, p.requested_gpus,
            p.n_minus_one_gpus, p.vcpus_per_node, p.quota_headroom_vcpus,
            p.efa_interfaces_required, p.efa_interfaces_available)
    if (any(type(v) is not int or v < 0 for v in ints)
            or min(p.ranks, p.gpus_per_node, p.nodes, p.requested_gpus,
                   p.vcpus_per_node, p.efa_interfaces_required) == 0):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.collective_p95_ms, p.collective_slo_ms, p.checkpoint_age_min,
               p.checkpoint_rpo_min, p.telemetry_age_s, p.max_telemetry_age_s,
               p.estimated_cost_usd, p.approved_budget_usd)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0
            for v in metrics)
            or min(p.collective_slo_ms, p.checkpoint_rpo_min,
                   p.max_telemetry_age_s, p.approved_budget_usd) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "gpu_capacity": p.requested_gpus <= p.nodes * p.gpus_per_node,
        "rank_capacity": p.ranks <= p.nodes * p.gpus_per_node,
        "n_minus_one": p.requested_gpus <= p.n_minus_one_gpus,
        "quota": p.nodes * p.vcpus_per_node <= p.quota_headroom_vcpus,
        "efa_interfaces": p.efa_interfaces_required <= p.efa_interfaces_available,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "budget": p.estimated_cost_usd <= p.approved_budget_usd,
        "efa_sku": p.efa_sku_supported is True,
        "efa_eni": p.efa_eni_attached is True,
        "physical_capacity": p.physical_capacity_verified is True,
        "same_az": p.same_az_placement_verified is True,
        "efa_driver": p.efa_driver_healthy is True,
        "libfabric": p.libfabric_efa_provider_verified is True,
        "aws_ofi_nccl": p.aws_ofi_nccl_verified is True,
        "gpu_driver": p.gpu_driver_cuda_verified is True,
        "efa_plugin": p.efa_device_plugin_verified is True,
        "gpu_plugin": p.gpu_device_plugin_verified is True,
        "peer_security": p.security_group_peer_rules_verified is True,
        "gpu_nic_topology": p.gpu_nic_topology_verified is True,
        "nccl_correctness": p.nccl_transport_and_correctness_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "workload_slo": p.workload_slo_verified is True,
        "tenant_security": p.identity_tenant_security_verified is True,
        "gpu_health": p.gpu_health_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates), "passed_gates": len(gates)-len(failed)}

def example():
    return Plan(
        "synthetic-efa-cluster", "us-east-1", "efa-gpu-example",
        8, 8, 2, 8, 8, 96, 256, 2, 4,
        2.0, 5.0, 5.0, 30.0, 10.0, 120.0, 100.0, 150.0,
        *([True] * 19)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_aws_efa_distributed_policy.py`

```python
from dataclasses import replace
import pytest
from aws_efa_distributed_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_identity"

def test_bool_as_int():
    assert evaluate(replace(example(), ranks=True))["reason"] == "invalid_count"

def test_fractional_node_count():
    assert evaluate(replace(example(), nodes=1.5))["reason"] == "invalid_count"

def test_nan_latency():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_infinite_cost():
    assert evaluate(replace(example(), estimated_cost_usd=float("inf")))["reason"] == "invalid_metric"

def test_bad_budget():
    assert evaluate(replace(example(), approved_budget_usd=0))["reason"] == "invalid_metric"

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 28

@pytest.mark.parametrize("field,gate,value", [
    ("requested_gpus", "gpu_capacity", 17),
    ("ranks", "rank_capacity", 17),
    ("n_minus_one_gpus", "n_minus_one", 7),
    ("quota_headroom_vcpus", "quota", 100),
    ("efa_interfaces_available", "efa_interfaces", 1),
    ("collective_p95_ms", "collective_slo", 6),
    ("checkpoint_age_min", "checkpoint_rpo", 31),
    ("telemetry_age_s", "telemetry", 121),
    ("estimated_cost_usd", "budget", 151),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("efa_sku_supported", "efa_sku"),
    ("efa_eni_attached", "efa_eni"),
    ("physical_capacity_verified", "physical_capacity"),
    ("same_az_placement_verified", "same_az"),
    ("efa_driver_healthy", "efa_driver"),
    ("libfabric_efa_provider_verified", "libfabric"),
    ("aws_ofi_nccl_verified", "aws_ofi_nccl"),
    ("gpu_driver_cuda_verified", "gpu_driver"),
    ("efa_device_plugin_verified", "efa_plugin"),
    ("gpu_device_plugin_verified", "gpu_plugin"),
    ("security_group_peer_rules_verified", "peer_security"),
    ("gpu_nic_topology_verified", "gpu_nic_topology"),
    ("nccl_transport_and_correctness_verified", "nccl_correctness"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("workload_slo_verified", "workload_slo"),
    ("identity_tenant_security_verified", "tenant_security"),
    ("gpu_health_verified", "gpu_health"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python aws_efa_distributed_policy.py
python -m pytest -q
```

**Planned 38 pytest cases:** 10 standalone + 9 numeric + 19 boolean. Execution and live EFA/NCCL testing remain pending.

## Ten offline incident drills

1. EFA SKU or ENI unsupported.
2. Driver or libfabric provider missing.
3. AWS OFI NCCL plugin mismatch.
4. EKS EFA device plugin unavailable.
5. GPU driver/device plugin regression.
6. Security-group peer rules broken.
7. Cross-AZ rank placement.
8. Quota/physical capacity shortage.
9. NCCL collective p95 or checkpoint RPO breach.
10. Missing telemetry, approval or rollback.

## Evidence and acceptance

Capture EFA SKU/AZ and quota, physical capacity, EFA ENI, driver, libfabric, plugin versions, GPU/NIC topology, EKS allocatable, peer SG rules, NCCL collective correctness and throughput, GPU health, checkpoint restore, cost, SLO, incident timeline and change approval. Do not use synthetic attestations as live evidence.

**100-point rubric:** architecture/capacity 20; transport/GPU 20; EKS/security 20; SLO/recovery/FinOps 20; tests/staging 20. Pass >=90 and zero critical blockers.
