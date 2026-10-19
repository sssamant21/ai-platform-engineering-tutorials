# Lab 039 — AKS GPU Node Pools and Drivers Gate

**Python 3.11+ · pytest · offline synthetic evidence.** No Azure credentials, AKS mutations, GPU allocation or cloud spend.

## Objective

Fail closed on unsafe AKS GPU user node pool configurations: quota, GPU and N-1 capacity, node bootstrap, driver ownership, device plugin, CUDA, CNI, security, autoscaler, upgrades, checkpoint, telemetry and rollback.

## File: `aks_gpu_pool_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    pool: str
    vm_sku: str
    desired_nodes: int
    max_nodes: int
    gpus_per_node: int
    requested_gpus: int
    n_minus_one_gpus: int
    family_quota_headroom_vcpus: int
    regional_quota_headroom_vcpus: int
    vcpus_per_node: int
    available_pod_ips: int
    required_pod_ips: int
    hourly_cost_usd: float
    hourly_budget_usd: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    gpu_user_pool_verified: bool
    sku_zone_verified: bool
    node_bootstrap_verified: bool
    driver_ownership_verified: bool
    driver_healthy: bool
    toolkit_cdi_verified: bool
    device_plugin_registered: bool
    cuda_canary_passed: bool
    autoscaler_verified: bool
    scheduling_verified: bool
    workload_identity_verified: bool
    tenant_security_verified: bool
    cni_network_verified: bool
    disruption_pdb_verified: bool
    checkpoint_restore_tested: bool
    gpu_health_verified: bool
    workload_slo_healthy: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if any(not isinstance(s, str) or not s.strip() for s in (p.cluster, p.pool, p.vm_sku)):
        return {"accepted": False, "reason": "missing_identity"}
    counts = (p.desired_nodes, p.max_nodes, p.gpus_per_node, p.requested_gpus,
              p.n_minus_one_gpus, p.family_quota_headroom_vcpus,
              p.regional_quota_headroom_vcpus, p.vcpus_per_node,
              p.available_pod_ips, p.required_pod_ips)
    if (any(type(x) is not int or x < 0 for x in counts)
            or p.desired_nodes == 0 or p.max_nodes < p.desired_nodes
            or min(p.gpus_per_node, p.requested_gpus, p.vcpus_per_node) == 0):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.hourly_cost_usd, p.hourly_budget_usd, p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or p.hourly_budget_usd <= 0 or p.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    vcpu_demand = p.desired_nodes * p.vcpus_per_node
    gates = {
        "gpu_capacity": p.requested_gpus <= p.desired_nodes * p.gpus_per_node,
        "n_minus_one": p.requested_gpus <= p.n_minus_one_gpus,
        "family_quota": vcpu_demand <= p.family_quota_headroom_vcpus,
        "regional_quota": vcpu_demand <= p.regional_quota_headroom_vcpus,
        "pod_ips": p.required_pod_ips <= p.available_pod_ips,
        "cost": p.hourly_cost_usd <= p.hourly_budget_usd,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "user_pool": p.gpu_user_pool_verified is True,
        "sku_zone": p.sku_zone_verified is True,
        "bootstrap": p.node_bootstrap_verified is True,
        "driver_owner": p.driver_ownership_verified is True,
        "driver": p.driver_healthy is True,
        "toolkit_cdi": p.toolkit_cdi_verified is True,
        "device_plugin": p.device_plugin_registered is True,
        "cuda": p.cuda_canary_passed is True,
        "autoscaler": p.autoscaler_verified is True,
        "scheduling": p.scheduling_verified is True,
        "identity": p.workload_identity_verified is True,
        "tenant_security": p.tenant_security_verified is True,
        "cni": p.cni_network_verified is True,
        "pdb": p.disruption_pdb_verified is True,
        "checkpoint": p.checkpoint_restore_tested is True,
        "gpu_health": p.gpu_health_verified is True,
        "workload_slo": p.workload_slo_healthy is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, value in gates.items() if not value)
    return {"accepted": not failed, "failed_gates": failed,
            "cluster": p.cluster, "pool": p.pool,
            "gpu_headroom": p.desired_nodes * p.gpus_per_node - p.requested_gpus}

def example():
    return Plan(
        "synthetic-aks", "gpu-staging", "Standard_NC24ads_A100_v4",
        3, 5, 1, 2, 2, 128, 256, 24, 40, 30,
        35.0, 100.0, 10.0, 120.0,
        *([True] * 19)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_aks_gpu_pool_policy.py`

```python
from dataclasses import replace
import pytest
from aks_gpu_pool_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_empty_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_identity"

def test_fractional_nodes():
    assert evaluate(replace(example(), desired_nodes=1.5))["reason"] == "invalid_count"

def test_boolean_gpus():
    assert evaluate(replace(example(), gpus_per_node=True))["reason"] == "invalid_count"

def test_bad_max_nodes():
    assert evaluate(replace(example(), max_nodes=2))["reason"] == "invalid_count"

def test_nan_cost():
    assert evaluate(replace(example(), hourly_cost_usd=float("nan")))["reason"] == "invalid_metric"

def test_negative_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=-1))["reason"] == "invalid_metric"

def test_gpu_capacity():
    assert "gpu_capacity" in evaluate(replace(example(), requested_gpus=4))["failed_gates"]

def test_n_minus_one():
    assert "n_minus_one" in evaluate(replace(example(), requested_gpus=3))["failed_gates"]

def test_family_quota():
    assert "family_quota" in evaluate(replace(example(), family_quota_headroom_vcpus=50))["failed_gates"]

def test_regional_quota():
    assert "regional_quota" in evaluate(replace(example(), regional_quota_headroom_vcpus=50))["failed_gates"]

def test_ip_exhaustion():
    assert "pod_ips" in evaluate(replace(example(), available_pod_ips=20))["failed_gates"]

def test_cost():
    assert "cost" in evaluate(replace(example(), hourly_cost_usd=101))["failed_gates"]

def test_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("gpu_user_pool_verified", "user_pool"),
    ("sku_zone_verified", "sku_zone"),
    ("node_bootstrap_verified", "bootstrap"),
    ("driver_ownership_verified", "driver_owner"),
    ("driver_healthy", "driver"),
    ("toolkit_cdi_verified", "toolkit_cdi"),
    ("device_plugin_registered", "device_plugin"),
    ("cuda_canary_passed", "cuda"),
    ("autoscaler_verified", "autoscaler"),
    ("scheduling_verified", "scheduling"),
    ("workload_identity_verified", "identity"),
    ("tenant_security_verified", "tenant_security"),
    ("cni_network_verified", "cni"),
    ("disruption_pdb_verified", "pdb"),
    ("checkpoint_restore_tested", "checkpoint"),
    ("gpu_health_verified", "gpu_health"),
    ("workload_slo_healthy", "workload_slo"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_required_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python aks_gpu_pool_policy.py
python -m pytest -q
```

**Planned 35 pytest cases:** 16 standalone and 19 parameterized. Runtime execution and live AKS validation remain pending.

## Ten synthetic incidents

1. Family GPU quota exhausted.
2. Total regional quota exhausted.
3. GPU SKU/zone unavailable.
4. Node bootstrap or driver ownership conflict.
5. Toolkit/CDI/device plugin failure.
6. GPU scheduling/scale-from-zero failure.
7. CNI pod IP exhaustion.
8. Identity or tenant isolation failure.
9. PDB/checkpoint/N-1 capacity failure.
10. Stale telemetry, workload SLO, budget or rollback blocker.

## Staging evidence checklist

Record approved AKS cluster/pool, VM SKU and zone, both quota headrooms, node image and driver owner, GPU UUID, device-plugin resources, CUDA canary, workload identity, Pod Security, Azure CNI, autoscaler, checkpoint restore, N-1, telemetry, SLO, cost and rollback. Use redacted evidence and change-owner signoff.

**Limitation:** Synthetic evidence cannot validate real AKS behavior, Azure capacity, CUDA correctness or security enforcement.

## 100-point assessment

AKS architecture 20; GPU runtime 20; scheduling/security/network 20; incident/recovery 20; tests/staging 20. Pass >=90 and zero critical blockers.
