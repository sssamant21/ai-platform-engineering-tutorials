# Lab 042 — AWS and Azure GPU Operations Capstone Gate

**Python 3.11+ · pytest · offline synthetic evidence.** No cloud API calls, GPU launches or spend.

## Objective

Evaluate a cross-cloud GPU operational readiness record. Require AWS accelerated vCPU quota, Azure family and regional quotas, workload GPU/N-1 capacity, SKU/zone/physical availability, driver/CUDA/NCCL, security, checkpoint RPO/RTO, SLO, telemetry, budget and approval. Reject invalid inputs and fail closed on all critical controls.

## File: `gpu_operations_capstone.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Evidence:
    workload: str
    aws_region: str
    azure_region: str
    aws_requested_vcpus: int
    aws_quota_remaining_vcpus: int
    azure_requested_vcpus: int
    azure_family_quota_remaining_vcpus: int
    azure_regional_quota_remaining_vcpus: int
    available_gpu_units: int
    required_gpu_units: int
    n_minus_one_gpu_units: int
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    restore_rto_min: float
    restore_rto_slo_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    estimated_total_cost_usd: float
    approved_budget_usd: float
    aws_sku_az_verified: bool
    azure_sku_az_verified: bool
    aws_physical_capacity_verified: bool
    azure_physical_capacity_verified: bool
    driver_ownership_verified: bool
    gpu_device_plugin_verified: bool
    cuda_canary_verified: bool
    nccl_network_verified_or_not_required: bool
    gpu_health_verified: bool
    identity_least_privilege_verified: bool
    tenant_isolation_verified: bool
    image_provenance_verified: bool
    encrypted_private_storage_verified: bool
    checkpoint_checksum_verified: bool
    checkpoint_restore_tested: bool
    autoscaling_verified: bool
    upgrade_rollback_verified: bool
    incident_drills_verified: bool
    workload_slo_verified: bool
    cost_attribution_verified: bool
    audit_evidence_verified: bool
    change_approved: bool
    independent_review_approved: bool

def evaluate(e):
    if not isinstance(e, Evidence):
        raise TypeError("Evidence required")
    if any(not isinstance(v, str) or not v.strip()
           for v in (e.workload, e.aws_region, e.azure_region)):
        return {"accepted": False, "reason": "missing_identity"}
    ints = (e.aws_requested_vcpus, e.aws_quota_remaining_vcpus,
            e.azure_requested_vcpus, e.azure_family_quota_remaining_vcpus,
            e.azure_regional_quota_remaining_vcpus, e.available_gpu_units,
            e.required_gpu_units, e.n_minus_one_gpu_units)
    if (any(type(v) is not int or v < 0 for v in ints)
            or min(e.aws_requested_vcpus, e.azure_requested_vcpus,
                   e.available_gpu_units, e.required_gpu_units) == 0):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (e.checkpoint_age_min, e.checkpoint_rpo_min,
               e.restore_rto_min, e.restore_rto_slo_min,
               e.telemetry_age_s, e.telemetry_max_age_s,
               e.estimated_total_cost_usd, e.approved_budget_usd)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0
            for v in metrics)
            or min(e.checkpoint_rpo_min, e.restore_rto_slo_min,
                   e.telemetry_max_age_s, e.approved_budget_usd) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "aws_quota": e.aws_requested_vcpus <= e.aws_quota_remaining_vcpus,
        "azure_family_quota": e.azure_requested_vcpus <= e.azure_family_quota_remaining_vcpus,
        "azure_regional_quota": e.azure_requested_vcpus <= e.azure_regional_quota_remaining_vcpus,
        "gpu_capacity": e.required_gpu_units <= e.available_gpu_units,
        "n_minus_one": e.required_gpu_units <= e.n_minus_one_gpu_units,
        "checkpoint_rpo": e.checkpoint_age_min <= e.checkpoint_rpo_min,
        "restore_rto": e.restore_rto_min <= e.restore_rto_slo_min,
        "telemetry": e.telemetry_age_s <= e.telemetry_max_age_s,
        "budget": e.estimated_total_cost_usd <= e.approved_budget_usd,
        "aws_sku_az": e.aws_sku_az_verified is True,
        "azure_sku_az": e.azure_sku_az_verified is True,
        "aws_physical_capacity": e.aws_physical_capacity_verified is True,
        "azure_physical_capacity": e.azure_physical_capacity_verified is True,
        "driver_owner": e.driver_ownership_verified is True,
        "device_plugin": e.gpu_device_plugin_verified is True,
        "cuda": e.cuda_canary_verified is True,
        "nccl_network": e.nccl_network_verified_or_not_required is True,
        "gpu_health": e.gpu_health_verified is True,
        "identity": e.identity_least_privilege_verified is True,
        "tenant_isolation": e.tenant_isolation_verified is True,
        "provenance": e.image_provenance_verified is True,
        "storage": e.encrypted_private_storage_verified is True,
        "checkpoint_checksum": e.checkpoint_checksum_verified is True,
        "checkpoint_restore": e.checkpoint_restore_tested is True,
        "autoscaling": e.autoscaling_verified is True,
        "upgrade_rollback": e.upgrade_rollback_verified is True,
        "incident_drills": e.incident_drills_verified is True,
        "slo": e.workload_slo_verified is True,
        "cost_attribution": e.cost_attribution_verified is True,
        "audit": e.audit_evidence_verified is True,
        "change": e.change_approved is True,
        "independent_review": e.independent_review_approved is True,
    }
    failed = sorted(name for name, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "passed_gates": len(gates) - len(failed), "total_gates": len(gates)}

def example():
    return Evidence(
        "synthetic-training", "us-east-1", "eastus",
        192, 384, 192, 384, 512, 16, 8, 8,
        5.0, 30.0, 20.0, 60.0, 10.0, 120.0, 400.0, 500.0,
        *([True] * 23)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_gpu_operations_capstone.py`

```python
from dataclasses import replace
import pytest
from gpu_operations_capstone import Evidence, evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_empty_workload():
    assert evaluate(replace(example(), workload=" "))["reason"] == "missing_identity"

def test_bool_as_int():
    assert evaluate(replace(example(), aws_requested_vcpus=True))["reason"] == "invalid_count"

def test_fractional_gpu():
    assert evaluate(replace(example(), available_gpu_units=1.5))["reason"] == "invalid_count"

def test_negative_count():
    assert evaluate(replace(example(), azure_requested_vcpus=-1))["reason"] == "invalid_count"

def test_nan_cost():
    assert evaluate(replace(example(), estimated_total_cost_usd=float("nan")))["reason"] == "invalid_metric"

def test_infinite_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=float("inf")))["reason"] == "invalid_metric"

def test_bool_as_metric():
    assert evaluate(replace(example(), checkpoint_age_min=True))["reason"] == "invalid_metric"

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_all_gates_count():
    assert evaluate(example())["total_gates"] == 32

@pytest.mark.parametrize("field,gate,value", [
    ("aws_quota_remaining_vcpus", "aws_quota", 100),
    ("azure_family_quota_remaining_vcpus", "azure_family_quota", 100),
    ("azure_regional_quota_remaining_vcpus", "azure_regional_quota", 100),
    ("available_gpu_units", "gpu_capacity", 4),
    ("n_minus_one_gpu_units", "n_minus_one", 4),
    ("checkpoint_age_min", "checkpoint_rpo", 31),
    ("restore_rto_min", "restore_rto", 61),
    ("telemetry_age_s", "telemetry", 121),
    ("estimated_total_cost_usd", "budget", 501),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("aws_sku_az_verified", "aws_sku_az"),
    ("azure_sku_az_verified", "azure_sku_az"),
    ("aws_physical_capacity_verified", "aws_physical_capacity"),
    ("azure_physical_capacity_verified", "azure_physical_capacity"),
    ("driver_ownership_verified", "driver_owner"),
    ("gpu_device_plugin_verified", "device_plugin"),
    ("cuda_canary_verified", "cuda"),
    ("nccl_network_verified_or_not_required", "nccl_network"),
    ("gpu_health_verified", "gpu_health"),
    ("identity_least_privilege_verified", "identity"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("image_provenance_verified", "provenance"),
    ("encrypted_private_storage_verified", "storage"),
    ("checkpoint_checksum_verified", "checkpoint_checksum"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("autoscaling_verified", "autoscaling"),
    ("upgrade_rollback_verified", "upgrade_rollback"),
    ("incident_drills_verified", "incident_drills"),
    ("workload_slo_verified", "slo"),
    ("cost_attribution_verified", "cost_attribution"),
    ("audit_evidence_verified", "audit"),
    ("change_approved", "change"),
    ("independent_review_approved", "independent_review"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run offline:

```bash
python -m pip install pytest
python gpu_operations_capstone.py
python -m pytest -q
```

**Planned: 43 pytest cases** (11 standalone + 9 numeric + 23 boolean). This lab validates *synthetic attestations*, not the underlying live cloud evidence.

## Ten incident drills

1. AWS accelerated GPU quota exhaustion.
2. Azure family quota exhaustion.
3. Azure regional quota exhaustion.
4. GPU SKU/AZ or physical capacity shortage.
5. NVIDIA driver or device-plugin failure.
6. CUDA/NCCL/EFA/RDMA regression.
7. GPU tenant identity or storage exposure.
8. Checkpoint restore/RPO/RTO failure.
9. Autoscaler or SLO regression during zone loss.
10. Budget breach or audit/approval failure.

## Staging evidence and scorecard

Collect redacted AWS/Azure quota records, GPU SKU and AZ restrictions, successful disposable staging launch, GPU UUID/driver, CUDA/NCCL, network tests, workload identity and private storage, checkpoint checksum/restore, GPU SLO, FinOps allocation, ten incident timelines and change/security/SRE approvals.

**100-point rubric:** capacity 20; runtime/network 20; security/storage 20; reliability/FinOps 20; test evidence and approval 20. Require >=90 with zero critical failures. Live staging and independent approval remain pending until explicitly executed and evidenced.
