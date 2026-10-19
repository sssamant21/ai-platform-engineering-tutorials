# Lab 047 — GPU Direct RDMA and Communication Gate

**Python 3.11+ · pytest · offline synthetic policy evaluation.** No GPU allocation, cloud API calls or live fabric changes.

## Objective

Fail closed on GPU/HCA compatibility, peer-memory/DMA-BUF path, CUDA/NCCL correctness, PCIe/NUMA, RDMA device exposure, security, N-1 capacity, checkpoint, SLO, telemetry, approval and rollback. Boolean attestations are not independent proof of the underlying hardware state.

## File: `gpu_direct_rdma_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    transport: str
    memory_path: str
    hosts: int
    gpus_per_host: int
    required_gpus: int
    surviving_gpus_after_failure: int
    total_nic_gbps: int
    surviving_nic_gbps: int
    min_surviving_nic_gbps: int
    collective_p95_ms: float
    collective_slo_ms: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    gpu_hca_pcie_compatible: bool
    kernel_driver_compatible: bool
    cuda_nccl_compatible: bool
    hca_firmware_compatible: bool
    memory_registration_verified: bool
    gpu_nic_numa_verified: bool
    rdma_device_plugin_verified: bool
    gpu_device_plugin_verified: bool
    effective_transport_verified: bool
    collective_correctness_verified: bool
    fallback_detected_and_reviewed: bool
    gpu_health_verified: bool
    network_security_verified: bool
    tenant_isolation_verified: bool
    checkpoint_restore_tested: bool
    workload_slo_verified: bool
    telemetry_pipeline_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    if p.transport not in ("infiniband", "roce", "efa"):
        return {"accepted": False, "reason": "invalid_transport"}
    if p.memory_path not in ("peer-memory", "dma-buf", "provider-supported"):
        return {"accepted": False, "reason": "invalid_memory_path"}
    counts = (p.hosts, p.gpus_per_host, p.required_gpus,
              p.surviving_gpus_after_failure, p.total_nic_gbps,
              p.surviving_nic_gbps, p.min_surviving_nic_gbps)
    if (any(type(v) is not int or v < 0 for v in counts)
            or min(p.hosts, p.gpus_per_host, p.required_gpus,
                   p.total_nic_gbps, p.min_surviving_nic_gbps) == 0
            or p.surviving_nic_gbps > p.total_nic_gbps):
        return {"accepted": False, "reason": "invalid_capacity"}
    metrics = (p.collective_p95_ms, p.collective_slo_ms,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0
            for v in metrics)
            or min(p.collective_slo_ms, p.checkpoint_rpo_min,
                   p.telemetry_max_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "gpu_capacity": p.required_gpus <= p.hosts * p.gpus_per_host,
        "n_minus_one_gpu": p.required_gpus <= p.surviving_gpus_after_failure,
        "n_minus_one_network": p.surviving_nic_gbps >= p.min_surviving_nic_gbps,
        "collective_slo": p.collective_p95_ms <= p.collective_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "pcie": p.gpu_hca_pcie_compatible is True,
        "kernel_driver": p.kernel_driver_compatible is True,
        "cuda_nccl": p.cuda_nccl_compatible is True,
        "hca_firmware": p.hca_firmware_compatible is True,
        "memory_registration": p.memory_registration_verified is True,
        "gpu_numa": p.gpu_nic_numa_verified is True,
        "rdma_plugin": p.rdma_device_plugin_verified is True,
        "gpu_plugin": p.gpu_device_plugin_verified is True,
        "transport": p.effective_transport_verified is True,
        "collective_correctness": p.collective_correctness_verified is True,
        "fallback_review": p.fallback_detected_and_reviewed is True,
        "gpu_health": p.gpu_health_verified is True,
        "network_security": p.network_security_verified is True,
        "tenant_isolation": p.tenant_isolation_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "workload_slo": p.workload_slo_verified is True,
        "telemetry_pipeline": p.telemetry_pipeline_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "passed_gates": len(gates) - len(failed), "total_gates": len(gates)}

def example():
    return Plan(
        "synthetic-gpu-rdma", "infiniband", "dma-buf",
        3, 8, 16, 16, 800, 600, 500,
        2.0, 5.0, 5.0, 30.0, 10.0, 120.0,
        *([True] * 19)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_gpu_direct_rdma_policy.py`

```python
from dataclasses import replace
import pytest
from gpu_direct_rdma_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_transport():
    assert evaluate(replace(example(), transport="tcp"))["reason"] == "invalid_transport"

def test_memory_path():
    assert evaluate(replace(example(), memory_path="unknown"))["reason"] == "invalid_memory_path"

def test_bool_count():
    assert evaluate(replace(example(), hosts=True))["reason"] == "invalid_capacity"

def test_fractional_gpus():
    assert evaluate(replace(example(), required_gpus=1.5))["reason"] == "invalid_capacity"

def test_surviving_exceeds_total():
    assert evaluate(replace(example(), surviving_nic_gbps=801))["reason"] == "invalid_capacity"

def test_nan():
    assert evaluate(replace(example(), collective_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), telemetry_age_s=float("inf")))["reason"] == "invalid_metric"

def test_nonboolean_security():
    assert "network_security" in evaluate(replace(example(), network_security_verified="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 25

@pytest.mark.parametrize("field,gate,value", [
    ("required_gpus", "gpu_capacity", 25),
    ("surviving_gpus_after_failure", "n_minus_one_gpu", 8),
    ("surviving_nic_gbps", "n_minus_one_network", 400),
    ("collective_p95_ms", "collective_slo", 6.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("gpu_hca_pcie_compatible", "pcie"),
    ("kernel_driver_compatible", "kernel_driver"),
    ("cuda_nccl_compatible", "cuda_nccl"),
    ("hca_firmware_compatible", "hca_firmware"),
    ("memory_registration_verified", "memory_registration"),
    ("gpu_nic_numa_verified", "gpu_numa"),
    ("rdma_device_plugin_verified", "rdma_plugin"),
    ("gpu_device_plugin_verified", "gpu_plugin"),
    ("effective_transport_verified", "transport"),
    ("collective_correctness_verified", "collective_correctness"),
    ("fallback_detected_and_reviewed", "fallback_review"),
    ("gpu_health_verified", "gpu_health"),
    ("network_security_verified", "network_security"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("workload_slo_verified", "workload_slo"),
    ("telemetry_pipeline_verified", "telemetry_pipeline"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python gpu_direct_rdma_policy.py
python -m pytest -q
```

**Planned 37 pytest cases:** 12 standalone + 6 numeric + 19 boolean. Runtime tests and live GPUDirect staging validation remain pending.

## Ten synthetic incident drills

1. GPU–HCA PCIe mismatch.
2. Kernel or NVIDIA driver regression.
3. CUDA/NCCL mismatch.
4. Peer-memory/DMA-BUF registration failure.
5. GPU/NIC NUMA misplacement.
6. RDMA/GPU device-plugin failure.
7. Transport fallback or collective correctness failure.
8. N-1 GPU/rail loss.
9. Checkpoint RPO or workload SLO violation.
10. Missing tenant security, telemetry, approval or rollback.

## Evidence and 100-point assessment

Record GPU/HCA hardware, PCIe/NUMA topology, firmware, driver, memory registration mode, NCCL transport logs, benchmark throughput, N-1 tests, checkpoint restore, security boundaries, SLO and rollback. Score semantics 20; compatibility 20; Kubernetes/security 20; resilience 20; tests/staging 20. Require >=90 and zero critical blockers.
