# Lab 041 — Multi-Cloud GPU Capacity and Cost Gate

**Python 3.11+ · pytest · offline synthetic evidence.** No cloud API calls, provisioning, price retrieval or cloud spend.

## Objective

Reject unsafe AWS/Azure GPU fleet plans when either applicable vCPU quota, GPU capacity, N-1 reserve, zone/physical availability, driver compatibility, checkpoint, telemetry, workload SLO, cost or security evidence fails.

## File: `multicloud_gpu_capacity_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    provider: str
    account_alias: str
    region: str
    instance_sku: str
    purchase_model: str
    instances: int
    gpus_per_instance: int
    requested_gpu_units: int
    n_minus_one_gpu_units: int
    vcpus_per_instance: int
    primary_quota_remaining_vcpus: int
    regional_quota_remaining_vcpus: int
    compute_cost_usd: float
    storage_network_cost_usd: float
    restart_waste_cost_usd: float
    budget_usd: float
    successful_jobs: int
    checkpoint_age_min: float
    max_checkpoint_age_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    sku_offering_verified: bool
    physical_capacity_verified: bool
    zone_resilience_verified: bool
    gpu_memory_fit_verified: bool
    driver_cuda_nccl_verified: bool
    storage_throughput_verified: bool
    data_residency_verified: bool
    identity_network_verified: bool
    checkpoint_restore_tested: bool
    spot_recovery_verified: bool
    workload_slo_verified: bool
    cost_attribution_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if p.provider not in ("aws", "azure"):
        return {"accepted": False, "reason": "invalid_provider"}
    if any(not isinstance(x, str) or not x.strip()
           for x in (p.account_alias, p.region, p.instance_sku)):
        return {"accepted": False, "reason": "missing_identity"}
    if p.purchase_model not in ("on-demand", "payg", "spot", "capacity-reservation"):
        return {"accepted": False, "reason": "invalid_purchase_model"}
    counts = (p.instances, p.gpus_per_instance, p.requested_gpu_units,
              p.n_minus_one_gpu_units, p.vcpus_per_instance,
              p.primary_quota_remaining_vcpus, p.regional_quota_remaining_vcpus,
              p.successful_jobs)
    if (any(type(x) is not int or x < 0 for x in counts)
            or min(p.instances, p.gpus_per_instance, p.requested_gpu_units,
                   p.vcpus_per_instance, p.successful_jobs) == 0):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.compute_cost_usd, p.storage_network_cost_usd,
               p.restart_waste_cost_usd, p.budget_usd,
               p.checkpoint_age_min, p.max_checkpoint_age_min,
               p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0
            for x in metrics)
            or min(p.budget_usd, p.max_checkpoint_age_min, p.max_telemetry_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    if ((p.provider == "aws" and p.purchase_model == "payg")
            or (p.provider == "azure" and p.purchase_model == "on-demand")):
        return {"accepted": False, "reason": "purchase_model_provider_mismatch"}
    vcpu_demand = p.instances * p.vcpus_per_instance
    total_cost = p.compute_cost_usd + p.storage_network_cost_usd + p.restart_waste_cost_usd
    gates = {
        "primary_quota": vcpu_demand <= p.primary_quota_remaining_vcpus,
        "regional_quota": p.provider == "aws" or vcpu_demand <= p.regional_quota_remaining_vcpus,
        "gpu_fit": p.requested_gpu_units <= p.instances * p.gpus_per_instance,
        "n_minus_one": p.requested_gpu_units <= p.n_minus_one_gpu_units,
        "budget": total_cost <= p.budget_usd,
        "checkpoint": p.checkpoint_age_min <= p.max_checkpoint_age_min,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "sku": p.sku_offering_verified is True,
        "physical_capacity": p.physical_capacity_verified is True,
        "zone_resilience": p.zone_resilience_verified is True,
        "gpu_memory": p.gpu_memory_fit_verified is True,
        "driver": p.driver_cuda_nccl_verified is True,
        "storage": p.storage_throughput_verified is True,
        "residency": p.data_residency_verified is True,
        "identity_network": p.identity_network_verified is True,
        "restore": p.checkpoint_restore_tested is True,
        "spot_recovery": p.purchase_model != "spot" or p.spot_recovery_verified is True,
        "slo": p.workload_slo_verified is True,
        "cost_attribution": p.cost_attribution_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "vcpu_demand": vcpu_demand,
            "total_cost_usd": round(total_cost, 2),
            "cost_per_successful_job_usd": round(total_cost / p.successful_jobs, 4)}

def example():
    return Plan(
        "aws", "synthetic-staging", "us-east-1", "gpu-example", "on-demand",
        3, 8, 16, 16, 96, 384, 512,
        100.0, 15.0, 5.0, 150.0, 10, 5.0, 30.0, 10.0, 120.0,
        *([True] * 14)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_multicloud_gpu_capacity_policy.py`

```python
from dataclasses import replace
import pytest
from multicloud_gpu_capacity_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_azure_baseline():
    assert evaluate(replace(example(), provider="azure", purchase_model="payg"))["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_invalid_provider():
    assert evaluate(replace(example(), provider="other"))["reason"] == "invalid_provider"

def test_missing_identity():
    assert evaluate(replace(example(), region=" "))["reason"] == "missing_identity"

def test_purchase_model_mismatch():
    assert evaluate(replace(example(), provider="azure"))["reason"] == "purchase_model_provider_mismatch"

def test_fractional_instances():
    assert evaluate(replace(example(), instances=1.5))["reason"] == "invalid_count"

def test_boolean_gpus():
    assert evaluate(replace(example(), gpus_per_instance=True))["reason"] == "invalid_count"

def test_zero_jobs():
    assert evaluate(replace(example(), successful_jobs=0))["reason"] == "invalid_count"

def test_nan_cost():
    assert evaluate(replace(example(), compute_cost_usd=float("nan")))["reason"] == "invalid_metric"

def test_primary_quota():
    assert "primary_quota" in evaluate(replace(example(), primary_quota_remaining_vcpus=200))["failed_gates"]

def test_azure_regional_quota():
    assert "regional_quota" in evaluate(replace(example(), provider="azure", purchase_model="payg", regional_quota_remaining_vcpus=200))["failed_gates"]

def test_gpu_fit():
    assert "gpu_fit" in evaluate(replace(example(), requested_gpu_units=25))["failed_gates"]

def test_n_minus_one():
    assert "n_minus_one" in evaluate(replace(example(), n_minus_one_gpu_units=10))["failed_gates"]

def test_budget():
    assert "budget" in evaluate(replace(example(), budget_usd=100))["failed_gates"]

def test_checkpoint():
    assert "checkpoint" in evaluate(replace(example(), checkpoint_age_min=31))["failed_gates"]

def test_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_spot():
    assert "spot_recovery" in evaluate(replace(example(), purchase_model="spot", spot_recovery_verified=False))["failed_gates"]

def test_cost_per_job():
    assert evaluate(example())["cost_per_successful_job_usd"] == 12.0

@pytest.mark.parametrize("field,gate", [
    ("sku_offering_verified", "sku"),
    ("physical_capacity_verified", "physical_capacity"),
    ("zone_resilience_verified", "zone_resilience"),
    ("gpu_memory_fit_verified", "gpu_memory"),
    ("driver_cuda_nccl_verified", "driver"),
    ("storage_throughput_verified", "storage"),
    ("data_residency_verified", "residency"),
    ("identity_network_verified", "identity_network"),
    ("checkpoint_restore_tested", "restore"),
    ("workload_slo_verified", "slo"),
    ("cost_attribution_verified", "cost_attribution"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_required_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python multicloud_gpu_capacity_policy.py
python -m pytest -q
```

**Planned 32 pytest cases:** 19 standalone and 13 parameterized. Offline test execution and live AWS/Azure verification are pending.

## Ten synthetic incident drills

1. AWS GPU vCPU quota exhausted.
2. Azure VM family quota exhausted.
3. Azure regional quota exhausted.
4. Region/AZ GPU capacity unavailable.
5. GPU or VRAM requirement exceeds chosen SKU.
6. N-1 resilience unavailable.
7. Spot interruption with no checkpoint recovery.
8. Cross-cloud residency or IAM blocks failover.
9. Total cost breaches budget.
10. Missing telemetry, SLO, approval or rollback.

## Staging acceptance checklist

Record provider/account alias, region/AZ, SKU/GPU/vCPU properties, current quota consumption, physical capacity evidence, benchmark throughput, GPU/driver/NCCL compatibility, SLO, checkpoint RPO/RTO, N-1, residency, identity, storage/network costs, pricing timestamp and owner signoff. Do not use synthetic values to authorize purchases.

**Limitations:** A green synthetic gate is not a guarantee of GPU physical capacity, current cloud pricing, security compliance, cost savings or production readiness.

## 100-point assessment

Workload sizing 20; quota/capacity 20; FinOps 20; incidents/recovery 20; tests/staging 20. Pass >=90 and zero critical blockers.
