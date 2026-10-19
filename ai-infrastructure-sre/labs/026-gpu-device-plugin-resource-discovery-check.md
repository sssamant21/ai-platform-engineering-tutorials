# Lab 026 — GPU Device Plugin Resource Discovery Gate

**Python 3.11+ · pytest · offline synthetic model.** No Kubernetes credentials, GPU access, privileged operations, driver changes, device allocations or cluster mutations.

## Goal

Fail closed if physical GPU inventory, Kubernetes extended resource supply, device-plugin registration, MIG strategy, runtime/CDI, CUDA smoke, health, security or rollback evidence is missing. A passing result does **not** prove that a live pod can initialize CUDA.

## File: `device_plugin_policy.py`

```python
from dataclasses import dataclass
import json
import math
import re

RESOURCE = re.compile(r"^nvidia\.com/(?:gpu|mig-[a-zA-Z0-9.-]+)$")

@dataclass(frozen=True)
class NodeInventory:
    node: str
    gpu_uuids: tuple[str, ...]
    expected_gpu_uuids: tuple[str, ...]
    resource_key: str
    advertised: int
    allocatable: int
    requested: int
    mig_strategy: str
    mig_profiles_verified: bool
    plugin_registered: bool
    plugin_ready: bool
    driver_compatible: bool
    toolkit_cdi_compatible: bool
    cuda_smoke_passed: bool
    gpu_health_verified: bool
    telemetry_age_s: float
    max_telemetry_age_s: float
    node_ready: bool
    tenant_security_approved: bool
    n_minus_one_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(n):
    if not isinstance(n, NodeInventory):
        raise TypeError("NodeInventory required")
    if not isinstance(n.node, str) or not n.node.strip():
        return {"accepted": False, "reason": "missing_node"}
    if (not isinstance(n.gpu_uuids, tuple)
            or not isinstance(n.expected_gpu_uuids, tuple)
            or not n.expected_gpu_uuids
            or any(not isinstance(u, str) or not u.startswith("GPU-")
                   for u in n.gpu_uuids + n.expected_gpu_uuids)
            or len(set(n.gpu_uuids)) != len(n.gpu_uuids)
            or len(set(n.expected_gpu_uuids)) != len(n.expected_gpu_uuids)):
        return {"accepted": False, "reason": "invalid_gpu_inventory"}
    if not isinstance(n.resource_key, str) or not RESOURCE.fullmatch(n.resource_key):
        return {"accepted": False, "reason": "invalid_resource_key"}
    if (any(type(v) is not int or v < 0 for v in (n.advertised, n.allocatable, n.requested))
            or n.allocatable > n.advertised):
        return {"accepted": False, "reason": "invalid_resource_count"}
    if n.mig_strategy not in ("none", "single", "mixed"):
        return {"accepted": False, "reason": "invalid_mig_strategy"}
    if (n.mig_strategy == "none" and n.resource_key != "nvidia.com/gpu"
            or n.mig_strategy == "mixed" and n.resource_key == "nvidia.com/gpu"
               and not n.mig_profiles_verified):
        return {"accepted": False, "reason": "invalid_mig_resource"}
    if (type(n.telemetry_age_s) not in (int, float)
            or type(n.max_telemetry_age_s) not in (int, float)
            or not math.isfinite(n.telemetry_age_s)
            or not math.isfinite(n.max_telemetry_age_s)
            or n.telemetry_age_s < 0 or n.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_telemetry"}
    gates = {
        "inventory": set(n.gpu_uuids) == set(n.expected_gpu_uuids),
        "resource_supply": n.requested <= n.allocatable and n.allocatable > 0,
        "mig_profiles": n.mig_strategy == "none" or n.mig_profiles_verified is True,
        "plugin_registered": n.plugin_registered is True,
        "plugin_ready": n.plugin_ready is True,
        "driver": n.driver_compatible is True,
        "runtime_cdi": n.toolkit_cdi_compatible is True,
        "cuda_smoke": n.cuda_smoke_passed is True,
        "gpu_health": n.gpu_health_verified is True,
        "telemetry": n.telemetry_age_s <= n.max_telemetry_age_s,
        "node_ready": n.node_ready is True,
        "security": n.tenant_security_approved is True,
        "n_minus_one": n.n_minus_one_verified is True,
        "change": n.change_approved is True,
        "rollback": n.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "physical_gpu_count": len(n.gpu_uuids),
            "resource_key": n.resource_key,
            "available_for_new_requests": n.allocatable - n.requested}

def example():
    return NodeInventory(
        "synthetic-gpu-node",
        ("GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
         "GPU-ffffffff-bbbb-cccc-dddd-eeeeeeeeeeee"),
        ("GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
         "GPU-ffffffff-bbbb-cccc-dddd-eeeeeeeeeeee"),
        "nvidia.com/gpu", 2, 2, 1, "none",
        *([True] * 7), 10.0, 120.0, *([True] * 5)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_device_plugin_policy.py`

```python
from dataclasses import replace
import pytest
from device_plugin_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_node():
    assert evaluate(replace(example(), node=" "))["reason"] == "missing_node"

def test_missing_inventory():
    assert "inventory" in evaluate(replace(example(), gpu_uuids=()))["failed_gates"]

def test_duplicate_uuid():
    s = example()
    assert evaluate(replace(s, gpu_uuids=(s.gpu_uuids[0],)*2))["reason"] == "invalid_gpu_inventory"

def test_bad_resource_key():
    assert evaluate(replace(example(), resource_key="gpu"))["reason"] == "invalid_resource_key"

def test_negative_count():
    assert evaluate(replace(example(), requested=-1))["reason"] == "invalid_resource_count"

def test_allocatable_exceeds_advertised():
    assert evaluate(replace(example(), allocatable=3))["reason"] == "invalid_resource_count"

def test_insufficient_resources():
    assert "resource_supply" in evaluate(replace(example(), requested=3))["failed_gates"]

def test_bad_mig_strategy():
    assert evaluate(replace(example(), mig_strategy="invalid"))["reason"] == "invalid_mig_strategy"

def test_invalid_mig_resource():
    assert evaluate(replace(example(), resource_key="nvidia.com/mig-1g.10gb"))["reason"] == "invalid_mig_resource"

def test_mig_profile_not_verified():
    assert "mig_profiles" in evaluate(replace(example(), mig_strategy="mixed", mig_profiles_verified=False))["failed_gates"]

def test_mig_resource():
    s = replace(example(), mig_strategy="mixed", resource_key="nvidia.com/mig-1g.10gb")
    assert evaluate(s)["accepted"]

def test_nan_telemetry():
    assert evaluate(replace(example(), telemetry_age_s=float("nan")))["reason"] == "invalid_telemetry"

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nonboolean_change():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("plugin_registered", "plugin_registered"),
    ("plugin_ready", "plugin_ready"),
    ("driver_compatible", "driver"),
    ("toolkit_cdi_compatible", "runtime_cdi"),
    ("cuda_smoke_passed", "cuda_smoke"),
    ("gpu_health_verified", "gpu_health"),
    ("node_ready", "node_ready"),
    ("tenant_security_approved", "security"),
    ("n_minus_one_verified", "n_minus_one"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_hard_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python device_plugin_policy.py
python -m pytest -q
```

**Expected: 27 pytest cases** (16 standalone and 11 parameterized). The examples are designed for offline execution, not production deployment.

## Ten synthetic incident drills

1. Host inventory differs from expected UUIDs.
2. Plugin registration fails and resource count drops.
3. GPU pod requests exceed node allocatable.
4. Wrong MIG resource key or strategy.
5. NVIDIA runtime/CDI compatibility fails.
6. CUDA canary fails despite correct scheduling.
7. GPU health/Xid is not verified.
8. Telemetry is stale.
9. Security approval or N-1 capacity is missing.
10. Change approval or rollback test fails.

## Live staging acceptance (separate approval required)

1. Verify GPU node ownership, node pool, driver/toolkit, GPU Operator/plugin versions and MIG strategy.
2. Compare physical GPU/MIG UUIDs against node capacity and allocatable resource keys.
3. Deploy a pinned, reviewed GPU canary in an approved namespace using an appropriate extended resource.
4. Inspect scheduler placement, plugin logs, kubelet registration and runtime/CDI device injection.
5. Verify `nvidia-smi`, CUDA device initialization, expected GPU UUID and application correctness inside the pod.
6. Correlate DCGM/Xid/ECC, resource supply, workload SLO and telemetry freshness.
7. Verify N-1 capacity, safe node drain, rollback and security review before production promotion.

**Limitation:** Synthetic acceptance cannot attest real GPU resource allocation, MIG partition safety, GPU Operator reconciliation, or CUDA functionality.

## 100-point rubric

Device-plugin architecture 20; extended resources/MIG 20; runtime/scheduling/security 20; incident runbooks 20; test and staging evidence 20. Pass >=90 with no critical correctness, security or resource-inventory blocker.
