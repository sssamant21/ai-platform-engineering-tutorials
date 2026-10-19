# Lab 037 — AWS EFA and GPU Networking Acceptance Gate

**Python 3.11+ / pytest / offline synthetic evidence.** No AWS credentials, EFA devices, network changes, cloud spend or cluster mutations.

## Goal

Fail closed on unsupported EFA instances, absent interfaces/provider, insecure peer network, wrong NCCL transport, GPU incompatibility, AZ placement, checkpoint/RPO, stale telemetry, N-1, cost or rollback.

## File: `aws_efa_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class EfaPlan:
    cluster: str
    instance_type: str
    region: str
    ranks: int
    efa_interfaces: int
    gpu_units: int
    available_gpu_units: int
    n_minus_one_gpu_units: int
    nccl_p95_ms: float
    nccl_p95_slo_ms: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    hourly_cost_usd: float
    max_hourly_cost_usd: float
    efa_instance_supported: bool
    efa_interface_attached: bool
    efa_kernel_module_healthy: bool
    libfabric_provider_ready: bool
    efa_device_plugin_ready: bool
    security_group_peer_rules_verified: bool
    subnet_az_topology_verified: bool
    aws_ofi_nccl_loaded: bool
    nccl_efa_transport_verified: bool
    gpu_driver_cuda_compatible: bool
    gpu_health_verified: bool
    checkpoint_restore_tested: bool
    tenant_security_verified: bool
    training_progress_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, EfaPlan):
        raise TypeError("EfaPlan required")
    if any(not isinstance(x, str) or not x.strip()
           for x in (p.cluster, p.instance_type, p.region)):
        return {"accepted": False, "reason": "missing_identity"}
    counts = (p.ranks, p.efa_interfaces, p.gpu_units,
              p.available_gpu_units, p.n_minus_one_gpu_units)
    if any(type(x) is not int or x < 0 for x in counts) or min(p.ranks, p.efa_interfaces, p.gpu_units) < 1:
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.nccl_p95_ms, p.nccl_p95_slo_ms,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.telemetry_age_s, p.max_telemetry_age_s,
               p.hourly_cost_usd, p.max_hourly_cost_usd)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or min(p.nccl_p95_slo_ms, p.checkpoint_rpo_min,
                   p.max_telemetry_age_s, p.max_hourly_cost_usd) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    checks = {
        "gpu_capacity": p.gpu_units <= p.available_gpu_units,
        "n_minus_one": p.gpu_units <= p.n_minus_one_gpu_units,
        "nccl_latency": p.nccl_p95_ms <= p.nccl_p95_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "cost": p.hourly_cost_usd <= p.max_hourly_cost_usd,
        "instance_support": p.efa_instance_supported is True,
        "interface": p.efa_interface_attached is True,
        "kernel": p.efa_kernel_module_healthy is True,
        "libfabric": p.libfabric_provider_ready is True,
        "device_plugin": p.efa_device_plugin_ready is True,
        "peer_security": p.security_group_peer_rules_verified is True,
        "topology": p.subnet_az_topology_verified is True,
        "ofi_nccl": p.aws_ofi_nccl_loaded is True,
        "efa_transport": p.nccl_efa_transport_verified is True,
        "gpu_driver": p.gpu_driver_cuda_compatible is True,
        "gpu_health": p.gpu_health_verified is True,
        "restore": p.checkpoint_restore_tested is True,
        "tenant_security": p.tenant_security_verified is True,
        "training_progress": p.training_progress_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, value in checks.items() if not value)
    return {"accepted": not failed, "failed_gates": failed,
            "cluster": p.cluster, "instance_type": p.instance_type,
            "gpu_headroom": p.available_gpu_units - p.gpu_units}

def example():
    return EfaPlan(
        "synthetic-eks-staging", "p4d.24xlarge", "us-east-1",
        4, 4, 8, 16, 8, 2.0, 5.0, 5.0, 30.0,
        10.0, 120.0, 50.0, 100.0,
        *([True] * 16)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_aws_efa_policy.py`

```python
from dataclasses import replace
import pytest
from aws_efa_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_identity():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_identity"

def test_boolean_ranks():
    assert evaluate(replace(example(), ranks=True))["reason"] == "invalid_count"

def test_fractional_efa_interfaces():
    assert evaluate(replace(example(), efa_interfaces=0.5))["reason"] == "invalid_count"

def test_zero_gpus():
    assert evaluate(replace(example(), gpu_units=0))["reason"] == "invalid_count"

def test_nan_latency():
    assert evaluate(replace(example(), nccl_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_negative_cost():
    assert evaluate(replace(example(), hourly_cost_usd=-1))["reason"] == "invalid_metric"

def test_gpu_capacity():
    assert "gpu_capacity" in evaluate(replace(example(), gpu_units=17))["failed_gates"]

def test_n_minus_one():
    assert "n_minus_one" in evaluate(replace(example(), gpu_units=9))["failed_gates"]

def test_nccl_latency():
    assert "nccl_latency" in evaluate(replace(example(), nccl_p95_ms=6))["failed_gates"]

def test_checkpoint_rpo():
    assert "checkpoint_rpo" in evaluate(replace(example(), checkpoint_age_min=31))["failed_gates"]

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_cost_guardrail():
    assert "cost" in evaluate(replace(example(), hourly_cost_usd=101))["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("efa_instance_supported", "instance_support"),
    ("efa_interface_attached", "interface"),
    ("efa_kernel_module_healthy", "kernel"),
    ("libfabric_provider_ready", "libfabric"),
    ("efa_device_plugin_ready", "device_plugin"),
    ("security_group_peer_rules_verified", "peer_security"),
    ("subnet_az_topology_verified", "topology"),
    ("aws_ofi_nccl_loaded", "ofi_nccl"),
    ("nccl_efa_transport_verified", "efa_transport"),
    ("gpu_driver_cuda_compatible", "gpu_driver"),
    ("gpu_health_verified", "gpu_health"),
    ("checkpoint_restore_tested", "restore"),
    ("tenant_security_verified", "tenant_security"),
    ("training_progress_verified", "training_progress"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_required_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python aws_efa_policy.py
python -m pytest -q
```

**Planned 31 pytest cases:** 15 standalone and 16 parameterized. Execution and live EFA/NCCL testing are pending.

## Ten offline incident drills

1. Instance does not support EFA.
2. EFA interface missing.
3. Kernel module or libfabric provider unhealthy.
4. EKS EFA device plugin missing.
5. Peer security group rules invalid.
6. AZ/topology incompatibility.
7. AWS OFI NCCL not loaded or wrong transport selected.
8. NCCL p95 exceeds SLO.
9. GPU/N-1 capacity or checkpoint RPO insufficient.
10. Cost, security, telemetry, approval or rollback missing.

## Staging acceptance

Use read-only AWS CLI to verify actual instance EFA support, interfaces, AZ offerings and quotas. On approved disposable GPU capacity, inspect `fi_info -p efa`, kernel modules, AWS OFI NCCL logs, GPU UUID, EKS EFA device plugin and security-group peer rules. Benchmark NCCL all-reduce under controlled load, validate checkpoint restore/RPO, N-1, telemetry, cost and rollback. Record redacted evidence and owner signoff.

**Limitations:** Synthetic attestations do not prove live EFA throughput, AWS quota/capacity, actual peer connectivity or NCCL transport.

## 100-point rubric

EFA architecture 20; libfabric/security 20; NCCL performance 20; incident/recovery 20; tests/staging 20. Pass >=90 and no critical blocker.
