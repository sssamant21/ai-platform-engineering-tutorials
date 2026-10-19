# Lab 038 — Azure GPU VM Architecture and Quotas Gate

**Python 3.11+ · pytest · offline synthetic evidence.** Does not connect to Azure, consume GPU quota, provision VMs or incur cloud charges.

## Objective

Validate both Azure regional and GPU VM-family vCPU quota headroom, GPU/VRAM fit, SKU/zone availability, physical capacity, secure identity/network, driver/CUDA, RDMA if required, storage/checkpoints, N-1, telemetry, SLO, cost and rollback. Reject malformed or missing evidence.

## File: `azure_gpu_vm_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    subscription_alias: str
    region: str
    vm_sku: str
    purchase_model: str
    instances: int
    vcpus_per_vm: int
    gpus_per_vm: int
    requested_gpus: int
    n_minus_one_gpus: int
    family_quota_vcpus: int
    family_consumed_vcpus: int
    regional_quota_vcpus: int
    regional_consumed_vcpus: int
    projected_hourly_cost_usd: float
    hourly_budget_usd: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    sku_subscription_available: bool
    zone_offering_verified: bool
    physical_capacity_verified: bool
    gpu_memory_fit_verified: bool
    driver_cuda_verified: bool
    rdma_verified_or_not_required: bool
    storage_throughput_verified: bool
    managed_identity_rbac_verified: bool
    private_network_encryption_verified: bool
    checkpoint_restore_tested: bool
    spot_recovery_verified: bool
    gpu_health_verified: bool
    workload_slo_verified: bool
    cost_attribution_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if any(not isinstance(s, str) or not s.strip()
           for s in (p.subscription_alias, p.region, p.vm_sku)):
        return {"accepted": False, "reason": "missing_identity"}
    if p.purchase_model not in ("payg", "spot", "capacity-reservation"):
        return {"accepted": False, "reason": "invalid_purchase_model"}
    counts = (p.instances, p.vcpus_per_vm, p.gpus_per_vm, p.requested_gpus,
              p.n_minus_one_gpus, p.family_quota_vcpus, p.family_consumed_vcpus,
              p.regional_quota_vcpus, p.regional_consumed_vcpus)
    if (any(type(x) is not int or x < 0 for x in counts)
            or min(p.instances, p.vcpus_per_vm, p.gpus_per_vm, p.requested_gpus) == 0):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.projected_hourly_cost_usd, p.hourly_budget_usd,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0
            for x in metrics)
            or min(p.hourly_budget_usd, p.checkpoint_rpo_min,
                   p.max_telemetry_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    requested_vcpus = p.instances * p.vcpus_per_vm
    gates = {
        "family_quota": requested_vcpus + p.family_consumed_vcpus <= p.family_quota_vcpus,
        "regional_quota": requested_vcpus + p.regional_consumed_vcpus <= p.regional_quota_vcpus,
        "gpu_fit": p.requested_gpus <= p.instances * p.gpus_per_vm,
        "n_minus_one": p.requested_gpus <= p.n_minus_one_gpus,
        "cost": p.projected_hourly_cost_usd <= p.hourly_budget_usd,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "sku": p.sku_subscription_available is True,
        "zone": p.zone_offering_verified is True,
        "physical_capacity": p.physical_capacity_verified is True,
        "gpu_memory": p.gpu_memory_fit_verified is True,
        "driver_cuda": p.driver_cuda_verified is True,
        "rdma": p.rdma_verified_or_not_required is True,
        "storage": p.storage_throughput_verified is True,
        "identity": p.managed_identity_rbac_verified is True,
        "network": p.private_network_encryption_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "spot_recovery": p.purchase_model != "spot" or p.spot_recovery_verified is True,
        "gpu_health": p.gpu_health_verified is True,
        "workload_slo": p.workload_slo_verified is True,
        "cost_attribution": p.cost_attribution_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "requested_vcpus": requested_vcpus,
            "family_quota_remaining": p.family_quota_vcpus - p.family_consumed_vcpus - requested_vcpus,
            "regional_quota_remaining": p.regional_quota_vcpus - p.regional_consumed_vcpus - requested_vcpus}

def example():
    return Plan(
        "synthetic-azure-subscription", "eastus", "Standard_NC24ads_A100_v4", "payg",
        2, 24, 1, 2, 2, 128, 24, 256, 48,
        25.0, 100.0, 5.0, 30.0, 10.0, 120.0,
        *([True] * 16)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_azure_gpu_vm_policy.py`

```python
from dataclasses import replace
import pytest
from azure_gpu_vm_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_region():
    assert evaluate(replace(example(), region=" "))["reason"] == "missing_identity"

def test_bad_purchase_model():
    assert evaluate(replace(example(), purchase_model="free"))["reason"] == "invalid_purchase_model"

def test_fractional_vm():
    assert evaluate(replace(example(), instances=1.5))["reason"] == "invalid_count"

def test_boolean_gpu():
    assert evaluate(replace(example(), gpus_per_vm=True))["reason"] == "invalid_count"

def test_nan_cost():
    assert evaluate(replace(example(), projected_hourly_cost_usd=float("nan")))["reason"] == "invalid_metric"

def test_negative_checkpoint():
    assert evaluate(replace(example(), checkpoint_age_min=-1))["reason"] == "invalid_metric"

def test_family_quota():
    assert "family_quota" in evaluate(replace(example(), family_quota_vcpus=60))["failed_gates"]

def test_regional_quota():
    assert "regional_quota" in evaluate(replace(example(), regional_quota_vcpus=60))["failed_gates"]

def test_gpu_fit():
    assert "gpu_fit" in evaluate(replace(example(), requested_gpus=3))["failed_gates"]

def test_n_minus_one():
    assert "n_minus_one" in evaluate(replace(example(), n_minus_one_gpus=1))["failed_gates"]

def test_cost():
    assert "cost" in evaluate(replace(example(), projected_hourly_cost_usd=101))["failed_gates"]

def test_checkpoint():
    assert "checkpoint_rpo" in evaluate(replace(example(), checkpoint_age_min=31))["failed_gates"]

def test_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_spot():
    assert "spot_recovery" in evaluate(
        replace(example(), purchase_model="spot", spot_recovery_verified=False)
    )["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("sku_subscription_available", "sku"),
    ("zone_offering_verified", "zone"),
    ("physical_capacity_verified", "physical_capacity"),
    ("gpu_memory_fit_verified", "gpu_memory"),
    ("driver_cuda_verified", "driver_cuda"),
    ("rdma_verified_or_not_required", "rdma"),
    ("storage_throughput_verified", "storage"),
    ("managed_identity_rbac_verified", "identity"),
    ("private_network_encryption_verified", "network"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("gpu_health_verified", "gpu_health"),
    ("workload_slo_verified", "workload_slo"),
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
python azure_gpu_vm_policy.py
python -m pytest -q
```

**Planned 32 pytest cases:** 17 standalone + 15 parameterized. Tests and live Azure validation remain pending.

## Ten synthetic incident drills

1. GPU family quota exceeded.
2. Total regional quota exceeded.
3. SKU unavailable for subscription.
4. AZ or physical capacity shortage.
5. GPU/VRAM sizing inadequate.
6. Driver/CUDA mismatch.
7. RDMA/InfiniBand not validated.
8. Checkpoint restore or Spot eviction readiness missing.
9. Telemetry, workload SLO or GPU health unverified.
10. Identity, network, budget, approval or rollback blocker.

## Staging acceptance checklist

Read actual subscription/region/SKU and both applicable quota limits, SKU restrictions and zone offerings. On approved disposable capacity, verify NVIDIA driver, GPU UUID, CUDA/NCCL and RDMA when required. Test managed identity, private network, durable checkpoint restore, RPO/RTO, N-1, SLO, telemetry, cost and rollback. Save redacted evidence with owner signoff.

**Limitation:** Synthetic evidence does not establish live Azure capacity, quota eligibility, real GPU performance, storage durability or production readiness.

## 100-point assessment

GPU architecture 20; quota/capacity 20; runtime/network/security 20; incident/recovery/cost 20; tests/staging 20. Pass >=90 and zero critical blockers.
