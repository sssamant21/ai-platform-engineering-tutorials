# Lab 036 — EKS GPU Managed Node Groups Acceptance Gate

**Python 3.11+ · pytest · offline synthetic evidence.** Does not access EKS, AWS, GPUs or production infrastructure.

## Goal

Reject incomplete or unsafe GPU MNG plans. Validate EC2 quota headroom, GPU count, N-1 capacity, driver/device-plugin/CUDA readiness, EKS node registration, AMI/bootstrap, IAM, CNI IP capacity, autoscaler, PDB, checkpoints, telemetry, cost and rollback.

## File: `eks_gpu_mng_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    nodegroup: str
    instance_type: str
    desired_nodes: int
    max_nodes: int
    gpus_per_node: int
    requested_gpu_units: int
    n_minus_one_gpu_units: int
    quota_vcpu_headroom: int
    vcpus_per_node: int
    available_pod_ips: int
    required_pod_ips: int
    projected_hourly_cost_usd: float
    hourly_budget_usd: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    ami_bootstrap_verified: bool
    node_iam_verified: bool
    subnet_security_verified: bool
    az_offering_verified: bool
    driver_owner_verified: bool
    driver_healthy: bool
    toolkit_cdi_verified: bool
    device_plugin_registered: bool
    cuda_canary_passed: bool
    autoscaler_verified: bool
    scheduling_verified: bool
    tenant_security_verified: bool
    disruption_pdb_verified: bool
    checkpoint_restore_tested: bool
    gpu_health_verified: bool
    workload_slo_healthy: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if any(not isinstance(x, str) or not x.strip()
           for x in (p.cluster, p.nodegroup, p.instance_type)):
        return {"accepted": False, "reason": "missing_identity"}
    counts = (p.desired_nodes, p.max_nodes, p.gpus_per_node,
              p.requested_gpu_units, p.n_minus_one_gpu_units,
              p.quota_vcpu_headroom, p.vcpus_per_node,
              p.available_pod_ips, p.required_pod_ips)
    if (any(type(x) is not int or x < 0 for x in counts)
            or p.desired_nodes == 0 or p.max_nodes < p.desired_nodes
            or p.gpus_per_node == 0 or p.requested_gpu_units == 0
            or p.vcpus_per_node == 0):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.projected_hourly_cost_usd, p.hourly_budget_usd,
               p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0
            for x in metrics)
            or p.hourly_budget_usd <= 0 or p.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "gpu_capacity": p.requested_gpu_units <= p.desired_nodes * p.gpus_per_node,
        "n_minus_one": p.requested_gpu_units <= p.n_minus_one_gpu_units,
        "ec2_quota": p.desired_nodes * p.vcpus_per_node <= p.quota_vcpu_headroom,
        "pod_ips": p.required_pod_ips <= p.available_pod_ips,
        "cost": p.projected_hourly_cost_usd <= p.hourly_budget_usd,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "ami_bootstrap": p.ami_bootstrap_verified is True,
        "node_iam": p.node_iam_verified is True,
        "subnet_security": p.subnet_security_verified is True,
        "az_offering": p.az_offering_verified is True,
        "driver_owner": p.driver_owner_verified is True,
        "driver": p.driver_healthy is True,
        "toolkit_cdi": p.toolkit_cdi_verified is True,
        "device_plugin": p.device_plugin_registered is True,
        "cuda": p.cuda_canary_passed is True,
        "autoscaler": p.autoscaler_verified is True,
        "scheduling": p.scheduling_verified is True,
        "tenant_security": p.tenant_security_verified is True,
        "pdb": p.disruption_pdb_verified is True,
        "checkpoint": p.checkpoint_restore_tested is True,
        "gpu_health": p.gpu_health_verified is True,
        "workload_slo": p.workload_slo_healthy is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "cluster": p.cluster, "nodegroup": p.nodegroup,
            "gpu_headroom": p.desired_nodes * p.gpus_per_node - p.requested_gpu_units}

def example():
    return Plan(
        "synthetic-eks", "gpu-staging", "g5.12xlarge",
        3, 5, 4, 8, 8, 256, 48, 40, 30,
        35.0, 100.0, 10.0, 120.0,
        *([True] * 18)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_eks_gpu_mng_policy.py`

```python
from dataclasses import replace
import pytest
from eks_gpu_mng_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_empty_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_identity"

def test_fractional_node_count():
    assert evaluate(replace(example(), desired_nodes=2.5))["reason"] == "invalid_count"

def test_boolean_gpu_count():
    assert evaluate(replace(example(), gpus_per_node=True))["reason"] == "invalid_count"

def test_bad_max_nodes():
    assert evaluate(replace(example(), max_nodes=2))["reason"] == "invalid_count"

def test_nan_cost():
    assert evaluate(replace(example(), projected_hourly_cost_usd=float("nan")))["reason"] == "invalid_metric"

def test_negative_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=-1))["reason"] == "invalid_metric"

def test_gpu_shortage():
    assert "gpu_capacity" in evaluate(replace(example(), requested_gpu_units=13))["failed_gates"]

def test_n_minus_one():
    assert "n_minus_one" in evaluate(replace(example(), requested_gpu_units=9))["failed_gates"]

def test_ec2_quota():
    assert "ec2_quota" in evaluate(replace(example(), quota_vcpu_headroom=100))["failed_gates"]

def test_ip_exhaustion():
    assert "pod_ips" in evaluate(replace(example(), available_pod_ips=20))["failed_gates"]

def test_cost_guardrail():
    assert "cost" in evaluate(replace(example(), projected_hourly_cost_usd=101))["failed_gates"]

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("ami_bootstrap_verified", "ami_bootstrap"),
    ("node_iam_verified", "node_iam"),
    ("subnet_security_verified", "subnet_security"),
    ("az_offering_verified", "az_offering"),
    ("driver_owner_verified", "driver_owner"),
    ("driver_healthy", "driver"),
    ("toolkit_cdi_verified", "toolkit_cdi"),
    ("device_plugin_registered", "device_plugin"),
    ("cuda_canary_passed", "cuda"),
    ("autoscaler_verified", "autoscaler"),
    ("scheduling_verified", "scheduling"),
    ("tenant_security_verified", "tenant_security"),
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
python eks_gpu_mng_policy.py
python -m pytest -q
```

**Planned 33 pytest cases:** 15 standalone + 18 parameterized. No execution or live AWS validation has been performed during publication.

## Ten synthetic incidents

1. EC2 vCPU quota exhausted.
2. GPU count or N-1 headroom insufficient.
3. CNI pod IP exhaustion.
4. AMI/bootstrap or node IAM regression.
5. Driver ownership conflict.
6. Device plugin/CDI/CUDA failure.
7. Autoscaler unable to scale.
8. PDB or checkpoint recovery unverified.
9. Stale telemetry or workload SLO breach.
10. Cost or change/rollback gate failure.

## Staging acceptance

Read actual EKS cluster/node-group/launch-template/AMI versions; check AWS quotas and AZ offerings; confirm subnet IPs and node IAM. On approved disposable GPU nodes, validate node registration, NVIDIA driver/toolkit/plugin, CUDA UUID, scheduling, autoscaler behavior, N-1, PDB, checkpoint, SLO, telemetry, cost and rollback. Record reviewed UTC evidence and signoff.

**Limitation:** The gate trusts synthetic attestations; it does not validate AWS quotas, Kubernetes scheduling, driver behavior, GPU health, EKS updates or actual spend.

## 100-point rubric

EKS architecture 20; GPU runtime 20; scaling/network 20; incident recovery 20; tests/staging 20. Pass >=90 and zero critical blockers.
