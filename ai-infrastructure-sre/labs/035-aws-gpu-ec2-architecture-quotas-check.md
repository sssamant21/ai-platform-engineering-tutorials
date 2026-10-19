# Lab 035 — AWS GPU EC2 Architecture and Quota Gate

**Python 3.11+ · pytest · offline synthetic capacity evidence.** No AWS credentials, EC2 launches, quota requests, billing calls or GPU workloads.

## Objective

Fail closed when EC2 quota headroom, instance offering, GPU sizing, regional capacity, EFA/storage fit, security, driver/CUDA, checkpoint, telemetry, N-1 or cost evidence is missing. All quota inputs are in **vCPUs in the correct EC2 quota bucket**, not GPU counts.

## File: `aws_gpu_ec2_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    account_alias: str
    region: str
    instance_type: str
    purchase_model: str
    instances: int
    vcpus_per_instance: int
    quota_vcpus: int
    consumed_quota_vcpus: int
    gpus_per_instance: int
    requested_gpu_units: int
    n_minus_one_gpu_units: int
    projected_hourly_cost_usd: float
    max_hourly_cost_usd: float
    checkpoint_age_min: float
    max_checkpoint_age_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    instance_offering_verified: bool
    physical_capacity_verified: bool
    gpu_memory_fit_verified: bool
    driver_cuda_verified: bool
    efa_network_verified_or_na: bool
    storage_throughput_verified: bool
    iam_imdsv2_kms_verified: bool
    private_network_verified: bool
    checkpoint_restore_tested: bool
    spot_interruption_plan_verified: bool
    gpu_health_verified: bool
    workload_slo_verified: bool
    cost_attribution_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if any(not isinstance(s, str) or not s.strip()
           for s in (p.account_alias, p.region, p.instance_type)):
        return {"accepted": False, "reason": "missing_identity"}
    if p.purchase_model not in ("on-demand", "spot", "capacity-reservation", "capacity-block"):
        return {"accepted": False, "reason": "invalid_purchase_model"}
    counts = (p.instances, p.vcpus_per_instance, p.quota_vcpus,
              p.consumed_quota_vcpus, p.gpus_per_instance,
              p.requested_gpu_units, p.n_minus_one_gpu_units)
    if (any(type(x) is not int or x < 0 for x in counts)
            or p.instances == 0 or p.vcpus_per_instance == 0
            or p.gpus_per_instance == 0 or p.requested_gpu_units == 0):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.projected_hourly_cost_usd, p.max_hourly_cost_usd,
               p.checkpoint_age_min, p.max_checkpoint_age_min,
               p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or p.max_hourly_cost_usd <= 0 or p.max_checkpoint_age_min <= 0
            or p.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    requested_vcpus = p.instances * p.vcpus_per_instance
    gates = {
        "quota_headroom": requested_vcpus + p.consumed_quota_vcpus <= p.quota_vcpus,
        "gpu_fit": p.requested_gpu_units <= p.instances * p.gpus_per_instance,
        "n_minus_one": p.requested_gpu_units <= p.n_minus_one_gpu_units,
        "cost": p.projected_hourly_cost_usd <= p.max_hourly_cost_usd,
        "checkpoint_freshness": p.checkpoint_age_min <= p.max_checkpoint_age_min,
        "telemetry_freshness": p.telemetry_age_s <= p.max_telemetry_age_s,
        "offering": p.instance_offering_verified is True,
        "physical_capacity": p.physical_capacity_verified is True,
        "gpu_memory": p.gpu_memory_fit_verified is True,
        "driver_cuda": p.driver_cuda_verified is True,
        "efa": p.efa_network_verified_or_na is True,
        "storage": p.storage_throughput_verified is True,
        "security": p.iam_imdsv2_kms_verified is True,
        "network": p.private_network_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "spot_recovery": p.purchase_model != "spot" or p.spot_interruption_plan_verified is True,
        "gpu_health": p.gpu_health_verified is True,
        "workload_slo": p.workload_slo_verified is True,
        "cost_attribution": p.cost_attribution_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "requested_vcpus": requested_vcpus,
            "remaining_quota_vcpus": p.quota_vcpus - p.consumed_quota_vcpus - requested_vcpus}

def example():
    return Plan(
        "synthetic-staging", "us-east-1", "g5.12xlarge", "on-demand",
        2, 48, 256, 64, 4, 4, 4,
        30.0, 50.0, 5.0, 30.0, 10.0, 120.0,
        *([True] * 15)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_aws_gpu_ec2_policy.py`

```python
from dataclasses import replace
import pytest
from aws_gpu_ec2_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_region():
    assert evaluate(replace(example(), region=" "))["reason"] == "missing_identity"

def test_invalid_purchase_model():
    assert evaluate(replace(example(), purchase_model="free"))["reason"] == "invalid_purchase_model"

def test_fractional_instance():
    assert evaluate(replace(example(), instances=0.5))["reason"] == "invalid_count"

def test_boolean_instance():
    assert evaluate(replace(example(), instances=True))["reason"] == "invalid_count"

def test_nan_cost():
    assert evaluate(replace(example(), projected_hourly_cost_usd=float("nan")))["reason"] == "invalid_metric"

def test_negative_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=-1))["reason"] == "invalid_metric"

def test_quota_exhaustion():
    assert "quota_headroom" in evaluate(replace(example(), quota_vcpus=100))["failed_gates"]

def test_gpu_fit():
    assert "gpu_fit" in evaluate(replace(example(), requested_gpu_units=9))["failed_gates"]

def test_n_minus_one():
    assert "n_minus_one" in evaluate(replace(example(), requested_gpu_units=5))["failed_gates"]

def test_cost():
    assert "cost" in evaluate(replace(example(), projected_hourly_cost_usd=51))["failed_gates"]

def test_checkpoint():
    assert "checkpoint_freshness" in evaluate(replace(example(), checkpoint_age_min=31))["failed_gates"]

def test_telemetry():
    assert "telemetry_freshness" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_spot_recovery():
    assert "spot_recovery" in evaluate(
        replace(example(), purchase_model="spot", spot_interruption_plan_verified=False)
    )["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("instance_offering_verified", "offering"),
    ("physical_capacity_verified", "physical_capacity"),
    ("gpu_memory_fit_verified", "gpu_memory"),
    ("driver_cuda_verified", "driver_cuda"),
    ("efa_network_verified_or_na", "efa"),
    ("storage_throughput_verified", "storage"),
    ("iam_imdsv2_kms_verified", "security"),
    ("private_network_verified", "network"),
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
python aws_gpu_ec2_policy.py
python -m pytest -q
```

**Planned 30 pytest cases:** 16 standalone and 14 parameterized. Tests and live AWS validation remain pending.

## Ten synthetic incident exercises

1. On-Demand quota exhausted.
2. Instance type unavailable in requested region.
3. Physical capacity unavailable despite quota.
4. GPU/VRAM insufficient for model.
5. EFA unsupported or unverified.
6. Driver/CUDA incompatibility.
7. EBS/FSx checkpoint throughput failure.
8. Spot interruption without recovery plan.
9. Stale GPU health/telemetry evidence.
10. N-1, budget, IAM or rollback control fails.

## Staging evidence checklist

Capture approved account alias/region, actual instance type/vCPUs/GPU count, quota bucket and current consumption, AZ offering and capacity procurement, IAM/IMDSv2/KMS, network/EFA, driver/CUDA, GPU UUID, checkpoint restore/RPO, N-1, projected and observed spend, SLO, rollback and change signoff.

**Limitations:** Synthetic inputs are attestations. A pass does not reserve EC2 capacity, prove quota eligibility, validate GPU hardware or authorize production use.

## Assessment

Instance architecture 20; quota/capacity 20; security/driver/EFA/storage 20; incidents/recovery/cost 20; tests/staging 20. >=90 and zero critical blockers.
