# Lab 031 — GPU Node Autoscaling and Karpenter Acceptance Gate

**Python 3.11+ · pytest · offline synthetic evidence model.** No AWS credentials, EC2 provisioning, cluster mutation, GPU usage or billing API calls.

## Goal

Fail closed when Karpenter configuration, GPU supply, cloud quota, AZ resilience, readiness, telemetry, checkpoint, disruption, SLO, cost guardrail, security or rollback evidence is missing.

## File: `gpu_autoscaling_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class ScalePlan:
    cluster: str
    nodepool: str
    gpu_instance_family: str
    capacity_type: str
    requested_gpu_units: int
    allocatable_gpu_units: int
    n_minus_one_gpu_units: int
    cloud_quota_gpu_units: int
    max_hourly_cost_usd: float
    projected_hourly_cost_usd: float
    pending_p95_s: float
    pending_slo_s: float
    karpenter_version_verified: bool
    ec2nodeclass_verified: bool
    iam_verified: bool
    subnet_ip_verified: bool
    az_offerings_verified: bool
    driver_ready: bool
    toolkit_ready: bool
    device_plugin_ready: bool
    cuda_canary_passed: bool
    gpu_health_verified: bool
    spot_checkpoint_verified: bool
    disruption_budget_verified: bool
    tenant_security_verified: bool
    workload_checkpoint_verified: bool
    telemetry_age_s: float
    max_telemetry_age_s: float
    change_approved: bool
    rollback_tested: bool
    workload_slo_verified: bool

def evaluate(p):
    if not isinstance(p, ScalePlan):
        raise TypeError("ScalePlan required")
    if any(not isinstance(x, str) or not x.strip()
           for x in (p.cluster, p.nodepool, p.gpu_instance_family)):
        return {"accepted": False, "reason": "missing_identity"}
    if p.capacity_type not in ("on-demand", "spot"):
        return {"accepted": False, "reason": "invalid_capacity_type"}
    counts = (p.requested_gpu_units, p.allocatable_gpu_units,
              p.n_minus_one_gpu_units, p.cloud_quota_gpu_units)
    if any(type(x) is not int or x < 0 for x in counts) or p.requested_gpu_units == 0:
        return {"accepted": False, "reason": "invalid_gpu_count"}
    floats = (p.max_hourly_cost_usd, p.projected_hourly_cost_usd,
              p.pending_p95_s, p.pending_slo_s,
              p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) for x in floats)
            or any(x < 0 for x in floats)
            or p.max_hourly_cost_usd <= 0 or p.pending_slo_s <= 0
            or p.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "capacity": p.requested_gpu_units <= p.allocatable_gpu_units,
        "n_minus_one": p.requested_gpu_units <= p.n_minus_one_gpu_units,
        "cloud_quota": p.requested_gpu_units <= p.cloud_quota_gpu_units,
        "cost": p.projected_hourly_cost_usd <= p.max_hourly_cost_usd,
        "pending_slo": p.pending_p95_s <= p.pending_slo_s,
        "karpenter_version": p.karpenter_version_verified is True,
        "nodeclass": p.ec2nodeclass_verified is True,
        "iam": p.iam_verified is True,
        "subnet_ips": p.subnet_ip_verified is True,
        "az_offerings": p.az_offerings_verified is True,
        "driver": p.driver_ready is True,
        "toolkit": p.toolkit_ready is True,
        "device_plugin": p.device_plugin_ready is True,
        "cuda_canary": p.cuda_canary_passed is True,
        "gpu_health": p.gpu_health_verified is True,
        "spot_checkpoint": p.capacity_type != "spot" or p.spot_checkpoint_verified is True,
        "disruption_budget": p.disruption_budget_verified is True,
        "tenant_security": p.tenant_security_verified is True,
        "checkpoint": p.workload_checkpoint_verified is True,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
        "workload_slo": p.workload_slo_verified is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "cluster": p.cluster, "nodepool": p.nodepool,
            "capacity_type": p.capacity_type,
            "gpu_headroom": p.allocatable_gpu_units - p.requested_gpu_units}

def example():
    return ScalePlan(
        "synthetic-eks-staging", "gpu-staging", "g5", "on-demand",
        8, 12, 8, 16, 100.0, 80.0, 120.0, 300.0,
        *([True] * 15), 10.0, 120.0, True, True, True
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_gpu_autoscaling_policy.py`

```python
from dataclasses import replace
import pytest
from gpu_autoscaling_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_identity"

def test_capacity_type():
    assert evaluate(replace(example(), capacity_type="reserved"))["reason"] == "invalid_capacity_type"

def test_fractional_gpu():
    assert evaluate(replace(example(), requested_gpu_units=1.5))["reason"] == "invalid_gpu_count"

def test_boolean_gpu():
    assert evaluate(replace(example(), requested_gpu_units=True))["reason"] == "invalid_gpu_count"

def test_nan_cost():
    assert evaluate(replace(example(), projected_hourly_cost_usd=float("nan")))["reason"] == "invalid_metric"

def test_negative_latency():
    assert evaluate(replace(example(), pending_p95_s=-1))["reason"] == "invalid_metric"

def test_capacity():
    assert "capacity" in evaluate(replace(example(), requested_gpu_units=13))["failed_gates"]

def test_n_minus_one():
    assert "n_minus_one" in evaluate(replace(example(), requested_gpu_units=9))["failed_gates"]

def test_quota():
    assert "cloud_quota" in evaluate(replace(example(), cloud_quota_gpu_units=7))["failed_gates"]

def test_cost():
    assert "cost" in evaluate(replace(example(), projected_hourly_cost_usd=101))["failed_gates"]

def test_pending_slo():
    assert "pending_slo" in evaluate(replace(example(), pending_p95_s=301))["failed_gates"]

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_spot_checkpoint():
    assert "spot_checkpoint" in evaluate(replace(example(), capacity_type="spot", spot_checkpoint_verified=False))["failed_gates"]

def test_nonboolean_change():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("karpenter_version_verified", "karpenter_version"),
    ("ec2nodeclass_verified", "nodeclass"),
    ("iam_verified", "iam"),
    ("subnet_ip_verified", "subnet_ips"),
    ("az_offerings_verified", "az_offerings"),
    ("driver_ready", "driver"),
    ("toolkit_ready", "toolkit"),
    ("device_plugin_ready", "device_plugin"),
    ("cuda_canary_passed", "cuda_canary"),
    ("gpu_health_verified", "gpu_health"),
    ("disruption_budget_verified", "disruption_budget"),
    ("tenant_security_verified", "tenant_security"),
    ("workload_checkpoint_verified", "checkpoint"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("workload_slo_verified", "workload_slo"),
])
def test_required_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python gpu_autoscaling_policy.py
python -m pytest -q
```

**Expected: 32 pytest cases** (16 standalone and 16 parameterized). Tests have not been executed as part of publication.

## Ten synthetic incident drills

1. GPU cloud quota exhausted.
2. Insufficient N-1 GPU capacity.
3. Projected hourly cost exceeds budget.
4. GPU pending p95 exceeds SLO.
5. EC2NodeClass configuration unverified.
6. IAM or subnet IP availability unverified.
7. Driver/device plugin not ready after launch.
8. Spot interruption without verified checkpoint.
9. Disruption budget or tenant security unverified.
10. Telemetry, change approval or rollback absent.

## Staging acceptance

Verify installed Karpenter version, actual NodePool/EC2NodeClass schema, IAM, subnets, GPU instance offerings and quotas. Render and review manifests; run a disposable GPU canary on an approved staging EKS cluster. Measure Pending-to-CUDA-ready, GPU UUID, DCGM/Xid/ECC, checkpoint and N-1 behavior. Exercise controlled disruption and rollback. Reconcile projected spend with cloud billing and workload SLO. No synthetic pass substitutes for live evidence.

## 100-point rubric

Autoscaling 20; Karpenter/AWS capacity 20; GPU readiness/security 20; incident/FinOps 20; tests/staging 20. Pass >=90 with no critical checkpoint, GPU readiness, tenant isolation or cost blocker.
