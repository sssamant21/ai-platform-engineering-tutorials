# Lab 024 — GPU Profiling and Bottleneck Acceptance

**Python 3.11+ · pytest · offline synthetic policy.** This lab does not query a GPU, run a profiler, allocate VRAM or infer bottleneck root cause from invented data.

## Objectives

Validate a profiling experiment's identity, measurement completeness, repeatability, profiler overhead, SLO, GPU health, telemetry freshness, privacy approval and rollback. The policy **does not** classify real kernels; that requires live profiler evidence and expert interpretation.

## File: `profiling_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Profile:
    workload: str
    gpu_uuid: str
    image_digest: str
    baseline_latency_ms: float
    candidate_latency_ms: float
    max_latency_regression_pct: float
    baseline_throughput: float
    candidate_throughput: float
    min_throughput_ratio: float
    profiler_overhead_pct: float
    max_profiler_overhead_pct: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    repetitions: int
    min_repetitions: int
    workload_shape_matched: bool
    hardware_matched: bool
    cuda_driver_compatible: bool
    gpu_health_verified: bool
    counters_supported: bool
    trace_privacy_approved: bool
    experiment_approved: bool
    rollback_tested: bool
    correctness_verified: bool

def evaluate(p):
    if not isinstance(p, Profile):
        raise TypeError("Profile required")
    if (not isinstance(p.workload, str) or not p.workload.strip()
            or not isinstance(p.gpu_uuid, str) or not p.gpu_uuid.startswith("GPU-")
            or not isinstance(p.image_digest, str) or not p.image_digest.startswith("sha256:")):
        return {"accepted": False, "reason": "missing_identity"}
    fields = (
        p.baseline_latency_ms, p.candidate_latency_ms, p.max_latency_regression_pct,
        p.baseline_throughput, p.candidate_throughput, p.min_throughput_ratio,
        p.profiler_overhead_pct, p.max_profiler_overhead_pct,
        p.telemetry_age_s, p.max_telemetry_age_s
    )
    if any(type(x) not in (int, float) or not math.isfinite(x) for x in fields):
        return {"accepted": False, "reason": "invalid_measurement"}
    if (p.baseline_latency_ms <= 0 or p.candidate_latency_ms <= 0
            or p.baseline_throughput <= 0 or p.candidate_throughput <= 0
            or p.max_latency_regression_pct < 0
            or not 0 < p.min_throughput_ratio <= 1
            or p.profiler_overhead_pct < 0 or p.max_profiler_overhead_pct < 0
            or p.telemetry_age_s < 0 or p.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_measurement"}
    if (type(p.repetitions) is not int or type(p.min_repetitions) is not int
            or p.repetitions < 1 or p.min_repetitions < 1):
        return {"accepted": False, "reason": "invalid_repetitions"}
    latency_regression = (p.candidate_latency_ms / p.baseline_latency_ms - 1) * 100
    throughput_ratio = p.candidate_throughput / p.baseline_throughput
    gates = {
        "latency_slo": latency_regression <= p.max_latency_regression_pct,
        "throughput_slo": throughput_ratio >= p.min_throughput_ratio,
        "profiler_overhead": p.profiler_overhead_pct <= p.max_profiler_overhead_pct,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "repetitions": p.repetitions >= p.min_repetitions,
        "workload_shape": p.workload_shape_matched is True,
        "hardware": p.hardware_matched is True,
        "compatibility": p.cuda_driver_compatible is True,
        "gpu_health": p.gpu_health_verified is True,
        "counters": p.counters_supported is True,
        "privacy": p.trace_privacy_approved is True,
        "approval": p.experiment_approved is True,
        "rollback": p.rollback_tested is True,
        "correctness": p.correctness_verified is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {
        "accepted": not failed,
        "failed_gates": failed,
        "latency_regression_pct": round(latency_regression, 3),
        "throughput_ratio": round(throughput_ratio, 4),
    }

def example():
    return Profile(
        "synthetic-inference", "GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
        "sha256:synthetic-verified-image",
        100.0, 95.0, 5.0, 1000.0, 1050.0, 0.95,
        2.0, 5.0, 10.0, 120.0, 5, 3,
        *([True] * 9)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_profiling_policy.py`

```python
from dataclasses import replace
import pytest
from profiling_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_workload():
    assert evaluate(replace(example(), workload=" "))["reason"] == "missing_identity"

def test_missing_uuid():
    assert evaluate(replace(example(), gpu_uuid="bad"))["reason"] == "missing_identity"

def test_image_identity():
    assert evaluate(replace(example(), image_digest="latest"))["reason"] == "missing_identity"

def test_nan():
    assert evaluate(replace(example(), candidate_latency_ms=float("nan")))["reason"] == "invalid_measurement"

def test_negative_throughput():
    assert evaluate(replace(example(), candidate_throughput=-1))["reason"] == "invalid_measurement"

def test_bool_latency():
    assert evaluate(replace(example(), baseline_latency_ms=True))["reason"] == "invalid_measurement"

def test_invalid_ratio():
    assert evaluate(replace(example(), min_throughput_ratio=2))["reason"] == "invalid_measurement"

def test_invalid_repetitions():
    assert evaluate(replace(example(), repetitions=True))["reason"] == "invalid_repetitions"

def test_latency_regression():
    assert "latency_slo" in evaluate(replace(example(), candidate_latency_ms=120))["failed_gates"]

def test_throughput_regression():
    assert "throughput_slo" in evaluate(replace(example(), candidate_throughput=800))["failed_gates"]

def test_profiler_overhead():
    assert "profiler_overhead" in evaluate(replace(example(), profiler_overhead_pct=10))["failed_gates"]

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_insufficient_repetitions():
    assert "repetitions" in evaluate(replace(example(), repetitions=1))["failed_gates"]

def test_output_metrics():
    r = evaluate(example())
    assert r["latency_regression_pct"] == -5.0
    assert r["throughput_ratio"] == 1.05

@pytest.mark.parametrize("field,gate", [
    ("workload_shape_matched", "workload_shape"),
    ("hardware_matched", "hardware"),
    ("cuda_driver_compatible", "compatibility"),
    ("gpu_health_verified", "gpu_health"),
    ("counters_supported", "counters"),
    ("trace_privacy_approved", "privacy"),
    ("experiment_approved", "approval"),
    ("rollback_tested", "rollback"),
    ("correctness_verified", "correctness"),
])
def test_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_approval():
    assert "approval" in evaluate(replace(example(), experiment_approved="yes"))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python profiling_policy.py
python -m pytest -q
```

**Expected 26 pytest cases:** 17 standalone and 9 parameterized. Runtime execution is a separate validation step.

## Eight offline incident drills

1. Latency p95/p99 regression exceeds acceptance threshold.
2. Throughput falls despite healthy GPU utilization.
3. Profiler overhead exceeds allowed budget.
4. Telemetry is stale or unsupported.
5. Workload shape differs between baseline and candidate.
6. GPU hardware/driver baseline changes.
7. Trace privacy or experiment approval is missing.
8. Correctness or rollback evidence fails.

## Staging acceptance

1. Obtain profiling and privacy approval; verify workload and GPU UUID.
2. Record immutable image, GPU/driver/CUDA/framework/NCCL, MIG profile, model and batch shape.
3. Warm up and measure unprofiled p50/p95/p99, throughput, memory and GPU health.
4. Collect bounded Nsight Systems trace; locate CPU launch, kernel, copy, sync and NCCL intervals.
5. Collect scoped Nsight Compute counters where supported and compare against roofline/occupancy hypotheses.
6. Run repeated, comparable canary tests; quantify trace overhead and uncertainty.
7. Confirm correctness, SLO, telemetry, rollback and independent review before production rollout.

**Limitation:** Synthetic policy acceptance does not prove a bottleneck, a valid GPU counter, profiler reproducibility or actual performance improvement.

## 100-point rubric

Profiling fundamentals 20; metric interpretation 20; controlled experiments 20; incidents/runbooks 20; executable evidence 20. Pass >=90 with no critical security, correctness or SLO failure.
