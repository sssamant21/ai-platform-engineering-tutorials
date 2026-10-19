# Lab 021 — NVLink and NVSwitch Topology Acceptance

**Requirements:** Python 3.11+, pytest. **Safety:** synthetic data only; no GPU access, topology mutation, node drain or saturation testing.

## Lab objectives

Validate expected GPU identities and peer paths; distinguish vendor-certified topology expectations from observations; fail closed on missing GPUs, unsupported paths, stale metrics, degraded fabric, driver skew, missing capacity, unapproved change or absent rollback. Actual topology support must be confirmed by hardware inventory and vendor documentation.

## File: `fabric_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class FabricSnapshot:
    node: str
    sku: str
    gpu_uuids: tuple[str, ...]
    expected_uuids: tuple[str, ...]
    observed_links: frozenset[tuple[str, str]]
    required_links: frozenset[tuple[str, str]]
    supported_links: frozenset[tuple[str, str]]
    driver_compatible: bool
    fabric_manager_required: bool
    fabric_manager_ready: bool
    p2p_probe_passed: bool
    nccl_smoke_passed: bool
    telemetry_age_seconds: float
    max_telemetry_age_seconds: float
    xid_clear: bool
    pcie_aer_clear: bool
    n_minus_one_verified: bool
    checkpoint_verified: bool
    security_approved: bool
    change_approved: bool
    rollback_tested: bool
    workload_slo_healthy: bool

def pair(a, b):
    if a == b:
        raise ValueError("self link")
    return tuple(sorted((a, b)))

def evaluate(s):
    if not isinstance(s, FabricSnapshot):
        raise TypeError("FabricSnapshot required")
    if any(not isinstance(x, str) or not x.strip() for x in (s.node, s.sku)):
        return {"accepted": False, "reason": "missing_identity"}
    if (not isinstance(s.gpu_uuids, tuple) or not isinstance(s.expected_uuids, tuple)
            or not s.gpu_uuids or any(not isinstance(x, str) or not x.startswith("GPU-")
                                      for x in s.gpu_uuids + s.expected_uuids)
            or len(set(s.gpu_uuids)) != len(s.gpu_uuids)
            or len(set(s.expected_uuids)) != len(s.expected_uuids)):
        return {"accepted": False, "reason": "invalid_gpu_inventory"}
    link_sets = (s.observed_links, s.required_links, s.supported_links)
    if any(not isinstance(v, frozenset) for v in link_sets):
        return {"accepted": False, "reason": "invalid_link_policy"}
    all_uuids = set(s.expected_uuids)
    for links in link_sets:
        for edge in links:
            if (not isinstance(edge, tuple) or len(edge) != 2
                    or any(not isinstance(x, str) or x not in all_uuids for x in edge)
                    or edge[0] >= edge[1]):
                return {"accepted": False, "reason": "invalid_link"}
    ages = (s.telemetry_age_seconds, s.max_telemetry_age_seconds)
    if any(type(v) not in (int, float) or not math.isfinite(v) for v in ages):
        return {"accepted": False, "reason": "invalid_telemetry"}
    if s.telemetry_age_seconds < 0 or s.max_telemetry_age_seconds <= 0:
        return {"accepted": False, "reason": "invalid_telemetry"}
    gates = {
        "gpu_inventory": set(s.gpu_uuids) == set(s.expected_uuids),
        "vendor_supported": s.required_links <= s.supported_links,
        "required_links": s.required_links <= s.observed_links,
        "driver": s.driver_compatible is True,
        "fabric_manager": (s.fabric_manager_required is False
                           or s.fabric_manager_ready is True),
        "p2p": s.p2p_probe_passed is True,
        "nccl": s.nccl_smoke_passed is True,
        "telemetry": s.telemetry_age_seconds <= s.max_telemetry_age_seconds,
        "xid": s.xid_clear is True,
        "pcie_aer": s.pcie_aer_clear is True,
        "n_minus_one": s.n_minus_one_verified is True,
        "checkpoint": s.checkpoint_verified is True,
        "security": s.security_approved is True,
        "change": s.change_approved is True,
        "rollback": s.rollback_tested is True,
        "slo": s.workload_slo_healthy is True,
    }
    failed = sorted(name for name, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "gpu_count": len(s.gpu_uuids),
            "missing_links": sorted(s.required_links - s.observed_links)}

def example():
    a = "GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
    b = "GPU-ffffffff-bbbb-cccc-dddd-eeeeeeeeeeee"
    edge = frozenset({pair(a, b)})
    return FabricSnapshot(
        "synthetic-node-01", "synthetic-nvswitch-sku",
        (a, b), (a, b), edge, edge, edge,
        True, True, True, True, True, 10.0, 120.0,
        *([True] * 8)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

**Important:** The synthetic `supported_links` input is an **attested allowlist**, not an actual vendor matrix. The policy models required pairwise connectivity, not switch graph paths, NVLink lane counts, bandwidth, link directionality or live fabric registration.

## File: `test_fabric_policy.py`

```python
from dataclasses import replace
import pytest
from fabric_policy import evaluate, example, pair

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_identity():
    assert evaluate(replace(example(), node=" "))["reason"] == "missing_identity"

def test_empty_inventory():
    assert evaluate(replace(example(), gpu_uuids=()))["reason"] == "invalid_gpu_inventory"

def test_duplicate_gpu():
    s = example()
    assert evaluate(replace(s, gpu_uuids=(s.gpu_uuids[0],)*2))["reason"] == "invalid_gpu_inventory"

def test_missing_gpu():
    s = example()
    assert "gpu_inventory" in evaluate(replace(s, gpu_uuids=(s.gpu_uuids[0],)))["failed_gates"]

def test_link_policy_type():
    assert evaluate(replace(example(), required_links=set()))["reason"] == "invalid_link_policy"

def test_invalid_edge():
    assert evaluate(replace(example(), required_links=frozenset({("bad", "other")})))["reason"] == "invalid_link"

def test_self_link():
    with pytest.raises(ValueError):
        pair("a", "a")

def test_missing_link():
    s = example()
    result = evaluate(replace(s, observed_links=frozenset()))
    assert "required_links" in result["failed_gates"]
    assert result["missing_links"]

def test_unsupported_link():
    s = example()
    assert "vendor_supported" in evaluate(replace(s, supported_links=frozenset()))["failed_gates"]

def test_invalid_age():
    assert evaluate(replace(example(), telemetry_age_seconds=float("nan")))["reason"] == "invalid_telemetry"

def test_stale():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_seconds=121))["failed_gates"]

def test_fabric_not_required():
    s = example()
    assert evaluate(replace(s, fabric_manager_required=False, fabric_manager_ready=False))["accepted"]

def test_fabric_required_not_ready():
    assert "fabric_manager" in evaluate(replace(example(), fabric_manager_ready=False))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("driver_compatible", "driver"),
    ("p2p_probe_passed", "p2p"),
    ("nccl_smoke_passed", "nccl"),
    ("xid_clear", "xid"),
    ("pcie_aer_clear", "pcie_aer"),
    ("n_minus_one_verified", "n_minus_one"),
    ("checkpoint_verified", "checkpoint"),
    ("security_approved", "security"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("workload_slo_healthy", "slo"),
])
def test_hard_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python fabric_policy.py
python -m pytest -q
```

**Expected 27 pytest cases:** 16 standalone and 11 parameterized. These are planned cases; runtime execution must be recorded separately.

## Eight synthetic incident drills

1. Remove one required link: reject topology.
2. Change GPU inventory: reject identity mismatch.
3. Remove vendor-supported pair: reject unsupported assumption.
4. Mark Fabric Manager unhealthy on a required platform: reject readiness.
5. Disable P2P attestation: reject CUDA capability.
6. Disable NCCL smoke: reject collective acceptance.
7. Age telemetry past threshold: reject stale evidence.
8. Fail N-1, checkpoint or rollback gate: reject unsafe promotion.

## Live staging acceptance checklist

- Capture UTC, GPU UUID, PCIe bus ID, `nvidia-smi topo -m`, driver, platform model and Fabric Manager ownership.
- Verify exact vendor-certified topology and P2P support for the GPU/MIG configuration.
- Run the chapter's CUDA peer-access probe and a **version-pinned, authorized** NCCL test; record command, message size, GPU mapping and results.
- Correlate NVLink/PCIe/Xid metrics and NCCL performance with application SLO and prior baseline.
- Confirm workload checkpoint, N-1 capacity, node change approval and tested rollback before modifying any production configuration.
- Obtain security/platform owner signoff. Do not equate synthetic success with hardware acceptance.

## 100-point rubric

Architecture 20; discovery and P2P 20; placement and fabric lifecycle 20; incidents and runbooks 20; executable evidence 20. Pass >=90 with no critical safety or SLO gate failure.
