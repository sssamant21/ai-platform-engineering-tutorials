# Chapter 088 — Executable GPU XID, ECC, Driver and CUDA Incident Lab

**Status:** Published; offline execution, live GPU checks and independent review pending. Synthetic signals only.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic GPU incident evaluator
Save as `gpu_incident.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class GpuSignals:
    xid_events: int = 0
    ecc_correctable_delta: int = 0
    ecc_uncorrectable_delta: int = 0
    cuda_oom: bool = False
    driver_compatible: bool = True
    device_allocatable: bool = True
    thermal_throttle: bool = False

def classify(s):
    counters = (s.xid_events, s.ecc_correctable_delta, s.ecc_uncorrectable_delta)
    if any(type(x) is not int or x < 0 for x in counters):
        return "invalid_signal"
    if s.ecc_uncorrectable_delta > 0:
        return "hardware_fault_escalation"
    if s.xid_events > 0:
        return "xid_investigation"
    if not s.driver_compatible:
        return "driver_runtime_mismatch"
    if not s.device_allocatable:
        return "device_plugin_or_scheduling"
    if s.cuda_oom:
        return "cuda_memory_pressure"
    if s.thermal_throttle:
        return "thermal_or_power_throttle"
    if s.ecc_correctable_delta > 0:
        return "ecc_trend_review"
    return "healthy"

@dataclass(frozen=True)
class Recovery:
    healthy_gpu_capacity: int
    required_gpu_capacity: int
    disruption_budget_ok: bool
    fault_cleared: bool
    synthetic_inference_passed: bool
    slo_verified: bool
    owner_approved: bool

def can_drain(r):
    if any(type(x) is not int or x < 0 for x in (r.healthy_gpu_capacity, r.required_gpu_capacity)):
        return False
    return r.healthy_gpu_capacity >= r.required_gpu_capacity and r.disruption_budget_ok is True and r.owner_approved is True

def can_return_to_service(r):
    return all((r.fault_cleared is True, r.synthetic_inference_passed is True,
                r.slo_verified is True, r.owner_approved is True))

if __name__ == "__main__":
    signals = GpuSignals(xid_events=1)
    recovery = Recovery(4, 3, True, True, True, True, True)
    print(json.dumps({"classification": classify(signals),
                      "can_drain": can_drain(recovery),
                      "can_return": can_return_to_service(recovery)}, indent=2))
```

**Limitations:** Xid counts are not vendor-specific severity determinations. ECC behavior varies by GPU and firmware. `can_drain` models only capacity, disruption budget and approval, not Kubernetes eviction, placement, topology, cold-start, stateful data or real hardware health.

## 3. Automated tests
Save as `test_gpu_incident.py`:
```python
from dataclasses import replace
import pytest
from gpu_incident import GpuSignals, Recovery, classify, can_drain, can_return_to_service

@pytest.mark.parametrize("signals,expected", [
    (GpuSignals(), "healthy"),
    (GpuSignals(ecc_uncorrectable_delta=1), "hardware_fault_escalation"),
    (GpuSignals(xid_events=1), "xid_investigation"),
    (GpuSignals(driver_compatible=False), "driver_runtime_mismatch"),
    (GpuSignals(device_allocatable=False), "device_plugin_or_scheduling"),
    (GpuSignals(cuda_oom=True), "cuda_memory_pressure"),
    (GpuSignals(thermal_throttle=True), "thermal_or_power_throttle"),
    (GpuSignals(ecc_correctable_delta=10), "ecc_trend_review"),
    (GpuSignals(xid_events=-1), "invalid_signal"),
])
def test_classification(signals, expected):
    assert classify(signals) == expected

def good_recovery():
    return Recovery(4, 3, True, True, True, True, True)

def test_safe_drain():
    assert can_drain(good_recovery())

def test_insufficient_capacity_blocks_drain():
    assert not can_drain(replace(good_recovery(), healthy_gpu_capacity=2))

def test_pdb_blocks_drain():
    assert not can_drain(replace(good_recovery(), disruption_budget_ok=False))

def test_unapproved_drain_blocked():
    assert not can_drain(replace(good_recovery(), owner_approved=False))

def test_return_to_service():
    assert can_return_to_service(good_recovery())

def test_fault_persists_blocks_return():
    assert not can_return_to_service(replace(good_recovery(), fault_cleared=False))

def test_failed_inference_blocks_return():
    assert not can_return_to_service(replace(good_recovery(), synthetic_inference_passed=False))

def test_slo_failure_blocks_return():
    assert not can_return_to_service(replace(good_recovery(), slo_verified=False))

def test_invalid_capacity_blocks_drain():
    assert not can_drain(replace(good_recovery(), healthy_gpu_capacity=-1))
```

Run:
```bash
python gpu_incident.py
python -m pytest -q
```
Expected: **17 pytest cases** (9 parameterized and 8 standalone). Execution has not been performed.

## 4. SRE failure drills
1. Trigger a synthetic Xid event and route to GPU/platform on-call.
2. Differentiate correctable ECC trend from uncorrectable fault.
3. Simulate CUDA OOM and reduce token/concurrency admission.
4. Simulate missing device allocatable capacity.
5. Simulate driver/CUDA incompatibility and plan node-image rollback.
6. Block node drain when N-1 GPU capacity is inadequate.
7. Block return-to-service until fault clearance, inference and SLO checks pass.
8. Draft a vendor escalation package with sanitized logs, GPU UUID, driver version and timeline.

## 5. 100-point acceptance
| Area | Points |
|---|---:|
| Fault classification | 20 |
| CUDA/driver and device plugin diagnosis | 20 |
| Safe drain/capacity controls | 20 |
| Recovery validation | 20 |
| Tests, runbooks and evidence | 20 |
| **Total** | **100** |

Educational target >=90/100 with no unsafe drain or premature return-to-service. Live GPU hardware validation and vendor guidance remain mandatory for production.
