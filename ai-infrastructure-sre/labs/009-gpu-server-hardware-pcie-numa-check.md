# Chapter 009 — Executable GPU Server PCIe and NUMA Acceptance Lab

**Status:** Published. Test execution and real hardware validation pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Topology policy evaluator
Save as `gpu_topology.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class GPU:
    gpu_id: str
    numa_node: int
    pcie_generation: int
    pcie_width: int
    expected_generation: int
    expected_width: int
    nic_numa_node: int
    driver_verified: bool
    thermal_healthy: bool
    ecc_healthy: bool

def evaluate(gpus):
    if not isinstance(gpus, list) or not gpus:
        return {"accepted": False, "reason": "missing_inventory"}
    if any(not isinstance(g, GPU) for g in gpus):
        return {"accepted": False, "reason": "invalid_gpu"}
    ids = [g.gpu_id for g in gpus]
    if any(not isinstance(i, str) or not i.strip() for i in ids) or len(ids) != len(set(ids)):
        return {"accepted": False, "reason": "invalid_gpu_ids"}
    failures = {}
    for g in gpus:
        values = (g.numa_node, g.nic_numa_node, g.pcie_generation,
                  g.pcie_width, g.expected_generation, g.expected_width)
        if any(type(v) is not int for v in values):
            return {"accepted": False, "reason": "invalid_numeric_fields"}
        if (g.numa_node < 0 or g.nic_numa_node < 0
                or g.pcie_generation <= 0 or g.expected_generation <= 0
                or g.pcie_width <= 0 or g.expected_width <= 0):
            return {"accepted": False, "reason": "invalid_numeric_fields"}
        gates = {
            "pcie_generation": g.pcie_generation >= g.expected_generation,
            "pcie_width": g.pcie_width >= g.expected_width,
            "nic_numa_affinity": g.numa_node == g.nic_numa_node,
            "driver": g.driver_verified is True,
            "thermal": g.thermal_healthy is True,
            "ecc": g.ecc_healthy is True,
        }
        failures[g.gpu_id] = sorted(k for k, passed in gates.items() if not passed)
    return {"accepted": all(not v for v in failures.values()),
            "failed_gates_by_gpu": failures, "gpu_count": len(gpus)}

def example():
    return [GPU("gpu0", 0, 4, 16, 4, 16, 0, True, True, True),
            GPU("gpu1", 1, 4, 16, 4, 16, 1, True, True, True)]

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
**Scope:** Values are **synthetic**. PCIe link generation/width must be measured under appropriate load; topology alone does not prove performance. The simple same-NUMA rule is a policy preference, not a universal requirement (multi-NIC, multi-root or fabric-aware systems may be optimal with other placements). No cloud or GPU APIs are invoked.

## 2. Pytest regression suite
Save as `test_gpu_topology.py`:
```python
from dataclasses import replace
import pytest
from gpu_topology import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_empty_inventory():
    assert evaluate([])["reason"] == "missing_inventory"

def test_invalid_gpu():
    assert evaluate([{}])["reason"] == "invalid_gpu"

def test_duplicate_ids():
    a, b = example()
    assert evaluate([a, replace(b, gpu_id=a.gpu_id)])["reason"] == "invalid_gpu_ids"

def test_empty_id():
    a, b = example()
    assert evaluate([replace(a, gpu_id=""), b])["reason"] == "invalid_gpu_ids"

def test_wrong_numa_type():
    a, b = example()
    assert evaluate([replace(a, numa_node=True), b])["reason"] == "invalid_numeric_fields"

def test_negative_numa():
    a, b = example()
    assert evaluate([replace(a, numa_node=-1), b])["reason"] == "invalid_numeric_fields"

def test_pcie_gen_regression():
    a, b = example()
    r = evaluate([replace(a, pcie_generation=3), b])
    assert "pcie_generation" in r["failed_gates_by_gpu"]["gpu0"]

def test_pcie_width_regression():
    a, b = example()
    r = evaluate([replace(a, pcie_width=8), b])
    assert "pcie_width" in r["failed_gates_by_gpu"]["gpu0"]

def test_nic_affinity():
    a, b = example()
    r = evaluate([replace(a, nic_numa_node=1), b])
    assert "nic_numa_affinity" in r["failed_gates_by_gpu"]["gpu0"]

@pytest.mark.parametrize("field,gate", [
    ("driver_verified", "driver"),
    ("thermal_healthy", "thermal"),
    ("ecc_healthy", "ecc"),
])
def test_health_gates(field, gate):
    a, b = example()
    r = evaluate([replace(a, **{field: False}), b])
    assert gate in r["failed_gates_by_gpu"]["gpu0"]

def test_nonboolean_health():
    a, b = example()
    r = evaluate([replace(a, driver_verified="yes"), b])
    assert "driver" in r["failed_gates_by_gpu"]["gpu0"]
```
Run:
```bash
python gpu_topology.py
python -m pytest -q
```
Expected: **14 pytest cases** (11 standalone + 3 parameterized); execution not performed.

## 3. Authorized staging extension
1. Record hardware model, CPU sockets, NUMA, PCIe root complexes and GPU/NIC bus IDs.
2. Capture `lscpu`, `numactl --hardware`, `lspci -tv`, `nvidia-smi topo -m` and GPU health with approved permissions.
3. Compare negotiated PCIe width/gen **under load** to documented SKU capabilities.
4. Benchmark H2D/D2H transfers, supported peer transfers and NCCL collectives with multiple runs and full configuration metadata.
5. Compare local versus remote NUMA CPU/memory placement, controlling for I/O and thermal conditions.
6. Record Xid/ECC, thermal/power, CPU utilization, NIC counters and user-facing inference/training SLOs.
7. Document findings, owner, remediation, rollback, ticket, UTC timeline and independent signoff.

## 4. Assessment
Hardware/PCIe (20), NUMA/affinity (20), discovery (20), troubleshooting/runbooks (20), executable lab/evidence (20). Pass >=90 without critical safety failures.
