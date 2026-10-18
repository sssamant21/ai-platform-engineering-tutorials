# Chapter 012 — Executable GPU Discovery with nvidia-smi and NVML Lab

**Status:** Published. Tests and real hardware discovery pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Offline GPU inventory validator
Save as `gpu_inventory.py`:
```python
from dataclasses import dataclass
import json
import math
import re

UUID = re.compile(r"^GPU-[0-9a-fA-F-]{8,}$")

@dataclass(frozen=True)
class GPU:
    uuid: str
    pci_bus_id: str
    model: str
    driver_version: str
    memory_total_mib: int
    memory_used_mib: int
    utilization_percent: float | None
    temperature_c: float | None
    telemetry_age_seconds: float
    ecc_healthy: bool
    xid_clear: bool

def evaluate(gpus, expected_uuids, max_telemetry_age=120):
    if not isinstance(gpus, list) or not gpus:
        return {"accepted": False, "reason": "empty_inventory"}
    if any(not isinstance(g, GPU) for g in gpus):
        return {"accepted": False, "reason": "invalid_record"}
    if (not isinstance(expected_uuids, (set, frozenset)) or not expected_uuids
            or any(not isinstance(u, str) or not UUID.fullmatch(u) for u in expected_uuids)):
        return {"accepted": False, "reason": "invalid_expected_inventory"}
    if (type(max_telemetry_age) not in (int, float)
            or not math.isfinite(max_telemetry_age) or max_telemetry_age <= 0):
        return {"accepted": False, "reason": "invalid_age_limit"}
    ids = [g.uuid for g in gpus]
    if any(not isinstance(u, str) or not UUID.fullmatch(u) for u in ids):
        return {"accepted": False, "reason": "invalid_uuid"}
    if len(set(ids)) != len(ids):
        return {"accepted": False, "reason": "duplicate_uuid"}
    failures = {}
    for g in gpus:
        if any(not isinstance(s, str) or not s.strip()
               for s in (g.pci_bus_id, g.model, g.driver_version)):
            return {"accepted": False, "reason": "missing_metadata"}
        if (type(g.memory_total_mib) is not int or type(g.memory_used_mib) is not int
                or g.memory_total_mib <= 0 or not 0 <= g.memory_used_mib <= g.memory_total_mib):
            return {"accepted": False, "reason": "invalid_memory"}
        if (type(g.telemetry_age_seconds) not in (int, float)
                or not math.isfinite(g.telemetry_age_seconds) or g.telemetry_age_seconds < 0):
            return {"accepted": False, "reason": "invalid_telemetry_age"}
        for value in (g.utilization_percent, g.temperature_c):
            if value is not None and (type(value) not in (int, float) or not math.isfinite(value)):
                return {"accepted": False, "reason": "invalid_metric"}
        gates = {
            "telemetry_fresh": g.telemetry_age_seconds <= max_telemetry_age,
            "utilization_supported": g.utilization_percent is not None,
            "utilization_range": g.utilization_percent is not None and 0 <= g.utilization_percent <= 100,
            "temperature_supported": g.temperature_c is not None,
            "temperature_plausible": g.temperature_c is not None and -40 <= g.temperature_c <= 125,
            "ecc": g.ecc_healthy is True,
            "xid": g.xid_clear is True,
        }
        failures[g.uuid] = sorted(k for k, passed in gates.items() if not passed)
    missing = sorted(expected_uuids - set(ids))
    unexpected = sorted(set(ids) - expected_uuids)
    return {"accepted": not missing and not unexpected and all(not v for v in failures.values()),
            "missing": missing, "unexpected": unexpected,
            "failed_gates_by_gpu": failures, "gpu_count": len(gpus)}

def example():
    g = GPU("GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee", "0000:81:00.0",
            "synthetic-gpu", "synthetic-driver", 81920, 4096,
            55.0, 62.0, 12.0, True, True)
    return [g], {g.uuid}

if __name__ == "__main__":
    records, expected = example()
    print(json.dumps(evaluate(records, expected), indent=2))
```
**Limits:** This is a synthetic, deliberately conservative policy. Temperature plausibility is **not** a thermal health threshold. Requiring supported utilization/temperature counters can reject legitimate devices or MIG modes; customize policy only with approved vendor evidence. UUID format validation does not prove authenticity. No NVML or cloud APIs are called.

## 2. Automated tests
Save as `test_gpu_inventory.py`:
```python
from dataclasses import replace
import pytest
from gpu_inventory import evaluate, example

def test_baseline():
    g, expected = example()
    assert evaluate(g, expected)["accepted"]

def test_empty():
    assert evaluate([], set())["reason"] == "empty_inventory"

def test_invalid_record():
    _, expected = example()
    assert evaluate([{}], expected)["reason"] == "invalid_record"

def test_bad_expected():
    g, _ = example()
    assert evaluate(g, {"bad"})["reason"] == "invalid_expected_inventory"

def test_duplicate():
    g, expected = example()
    assert evaluate(g + g, expected)["reason"] == "duplicate_uuid"

def test_bad_uuid():
    g, expected = example()
    assert evaluate([replace(g[0], uuid="gpu0")], expected)["reason"] == "invalid_uuid"

def test_missing():
    g, expected = example()
    assert evaluate(g, expected | {"GPU-ffffffff-ffff-ffff-ffff-ffffffffffff"})["missing"]

def test_unexpected():
    g, _ = example()
    assert evaluate(g, {"GPU-ffffffff-ffff-ffff-ffff-ffffffffffff"})["unexpected"]

def test_bad_memory():
    g, expected = example()
    assert evaluate([replace(g[0], memory_used_mib=999999)], expected)["reason"] == "invalid_memory"

def test_negative_age():
    g, expected = example()
    assert evaluate([replace(g[0], telemetry_age_seconds=-1)], expected)["reason"] == "invalid_telemetry_age"

def test_stale():
    g, expected = example()
    r = evaluate([replace(g[0], telemetry_age_seconds=121)], expected)
    assert "telemetry_fresh" in r["failed_gates_by_gpu"][g[0].uuid]

@pytest.mark.parametrize("field,gate,value", [
    ("utilization_percent", "utilization_supported", None),
    ("utilization_percent", "utilization_range", 101),
    ("temperature_c", "temperature_supported", None),
    ("temperature_c", "temperature_plausible", 200),
    ("ecc_healthy", "ecc", False),
    ("xid_clear", "xid", False),
])
def test_gate(field, gate, value):
    g, expected = example()
    r = evaluate([replace(g[0], **{field: value})], expected)
    assert gate in r["failed_gates_by_gpu"][g[0].uuid]

def test_nonboolean_ecc():
    g, expected = example()
    r = evaluate([replace(g[0], ecc_healthy="yes")], expected)
    assert "ecc" in r["failed_gates_by_gpu"][g[0].uuid]

def test_nan_rejected():
    g, expected = example()
    assert evaluate([replace(g[0], utilization_percent=float("nan"))], expected)["reason"] == "invalid_metric"

def test_invalid_age_limit():
    g, expected = example()
    assert evaluate(g, expected, max_telemetry_age=0)["reason"] == "invalid_age_limit"
```
Run:
```bash
python gpu_inventory.py
python -m pytest -q
```
Expected **20 tests** (14 standalone and 6 parameterized); not executed.

## 3. Approved real-GPU extension
1. Capture sanitized `nvidia-smi -L`, UUID, PCI bus ID, driver and supported metric inventory.
2. Correlate GPU UUID with the node asset inventory and `nvidia-smi topo -m`.
3. Compare Kubernetes device-plugin resources and approved GPU/MIG configuration.
4. Collect DCGM/NVML telemetry with UTC timestamps; preserve unsupported/N/A values.
5. Test stale telemetry, missing GPU and replaced-GPU detection in staging.
6. Record runbook owner, ticket, change approval, rollback and independent validation.

## 4. 100-point rubric
NVML/identities (20), discovery (20), metric semantics (20), incidents/runbooks (20), lab/evidence (20). Pass >=90 with no critical inventory or security failure.
