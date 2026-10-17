# Chapter 072 — Executable Cold Start, Warm Pools and Model Loading Lab

**Status:** Published; live execution pending. **Scope:** deterministic synthetic stage timing and warm-pool capacity; no GPU, Kubernetes, network or model artifact downloads.

## 1. Prerequisites
Python 3.11+ and pytest.
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Cold-start and warm-pool simulator
Save as `cold_start.py`:
```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Stages:
    schedule_s: float
    image_s: float
    artifact_s: float
    verify_s: float
    gpu_load_s: float
    compile_s: float
    warmup_s: float
    route_s: float

def startup_seconds(stages):
    durations = [getattr(stages, field) for field in stages.__dataclass_fields__]
    if any(not isinstance(x, (int, float)) or not math.isfinite(x) or x < 0 for x in durations):
        raise ValueError("invalid stage duration")
    return sum(durations)

def ready_at(stages, began_at=0.0, verified=True, warmup_ok=True):
    if not math.isfinite(began_at) or began_at < 0:
        raise ValueError("invalid start")
    total = startup_seconds(stages)
    if not verified or not warmup_ok:
        return None
    return began_at + total

def required_replicas(peak_rps, safe_rps_per_replica, ready_replicas, failure_reserve=1):
    vals = (peak_rps, safe_rps_per_replica)
    if any(not math.isfinite(v) or v < 0 for v in vals) or safe_rps_per_replica == 0:
        raise ValueError("invalid rates")
    if type(ready_replicas) is not int or ready_replicas < 0:
        raise ValueError("invalid ready replicas")
    if type(failure_reserve) is not int or failure_reserve < 0:
        raise ValueError("invalid reserve")
    target = math.ceil(peak_rps / safe_rps_per_replica) + failure_reserve
    return {"target_ready_replicas": target, "additional_replicas": max(0, target - ready_replicas)}

def warm_pool_cost(gpus_per_replica, warm_replicas, hours, usd_per_gpu_hour):
    if any(type(v) is not int or v < 0 for v in (gpus_per_replica, warm_replicas)):
        raise ValueError("invalid counts")
    if any(not math.isfinite(v) or v < 0 for v in (hours, usd_per_gpu_hour)):
        raise ValueError("invalid billing")
    return gpus_per_replica * warm_replicas * hours * usd_per_gpu_hour

if __name__ == "__main__":
    cold = Stages(30, 20, 90, 5, 45, 60, 15, 5)
    cached = Stages(30, 5, 10, 5, 45, 15, 15, 5)
    print(json.dumps({
        "cold_start_s": startup_seconds(cold),
        "cached_start_s": startup_seconds(cached),
        "capacity": required_replicas(55, 12, 3),
        "warm_pool_cost_usd": warm_pool_cost(1, 2, 24, 3.5),
    }, indent=2))
```

**Model limitation:** Stage durations are added sequentially and do not overlap; real startup pipelines may overlap transfers, scheduling and compilation. Capacity assumes fixed safe RPS/replica, not workload-dependent batching, regional routing or autoscaler lag.

## 3. Automated tests
Save as `test_cold_start.py`:
```python
import pytest
from cold_start import Stages, startup_seconds, ready_at, required_replicas, warm_pool_cost

C = Stages(30, 20, 90, 5, 45, 60, 15, 5)

def test_total_duration():
    assert startup_seconds(C) == 270

def test_readiness_after_warmup():
    assert ready_at(C, 100) == 370

def test_checksum_failure_blocks_readiness():
    assert ready_at(C, verified=False) is None

def test_warmup_failure_blocks_readiness():
    assert ready_at(C, warmup_ok=False) is None

def test_cached_start_faster():
    cached = Stages(30, 5, 10, 5, 45, 15, 15, 5)
    assert startup_seconds(cached) < startup_seconds(C)

def test_peak_capacity_and_reserve():
    result = required_replicas(55, 12, 3, 1)
    assert result == {"target_ready_replicas": 6, "additional_replicas": 3}

def test_no_extra_capacity():
    assert required_replicas(12, 12, 2, 1)["additional_replicas"] == 0

def test_warm_pool_cost():
    assert warm_pool_cost(1, 2, 24, 3.5) == 168

@pytest.mark.parametrize("bad", [
    Stages(-1, 0, 0, 0, 0, 0, 0, 0),
    Stages(float("nan"), 0, 0, 0, 0, 0, 0, 0),
])
def test_invalid_stage(bad):
    with pytest.raises(ValueError):
        startup_seconds(bad)

def test_invalid_replica_rate():
    with pytest.raises(ValueError):
        required_replicas(10, 0, 1)

def test_invalid_cost():
    with pytest.raises(ValueError):
        warm_pool_cost(1, 2, -1, 3)
```

Run:
```bash
python cold_start.py
python -m pytest -q
```
Expected: **12 pytest cases** (10 standalone and 2 parameterized). Actual execution has not been performed.

## 4. Failure drills
1. Increase artifact transfer duration from 90 to 300 seconds; quantify time-to-readiness regression.
2. Reduce transfer time with an assumed warm local cache; compare total.
3. Inject artifact verification failure and assert readiness stays false.
4. Inject warmup failure and verify the replica is not counted as usable.
5. Increase peak traffic to 100 RPS; recalculate ready replicas and N-1 reserve.
6. Double the warm-pool replica count; quantify idle GPU-hour cost.
7. Simulate a region losing one ready replica; evaluate available safe RPS.
8. Discuss how overlapping startup phases would change the synthetic arithmetic.

## 5. Evidence and acceptance
Record Python version, pytest result, cold/warm stage durations, readiness failures, peak RPS assumption, safe RPS/replica, warm-pool count, failure reserve and GPU-hour pricing.

| Criterion | Points |
|---|---:|
| Stage timing and readiness semantics | 20 |
| Cold versus cached start comparison | 20 |
| Warm capacity and N-1 reserve | 20 |
| Failure drills and automated tests | 20 |
| Security, spend caps and runbook readiness | 20 |
| **Total** | **100** |

Recommended educational acceptance >=90/100. Production readiness requires real staging measurements, model integrity checks, service-owner approval and technical/security review.
