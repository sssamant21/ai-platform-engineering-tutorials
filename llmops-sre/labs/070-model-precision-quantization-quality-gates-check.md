# Chapter 070 — Executable Quantization Quality-Gate Lab

**Status:** Published; execution/reviews pending. Synthetic evaluation only; no model or external API.

## Setup
Python 3.11+; `python -m pip install pytest`.

Save as `quality_gate.py`:
```python
from dataclasses import dataclass
import math

@dataclass(frozen=True)
class Candidate:
    name: str
    task_correct: int
    task_total: int
    json_valid: int
    json_total: int
    critical_safety_failures: int
    ttft_p95_ms: float
    gpu_memory_gib: float

def rate(ok, total):
    if type(ok) is not int or type(total) is not int or total <= 0 or not 0 <= ok <= total:
        raise ValueError("invalid counts")
    return ok / total

def evaluate(baseline, candidate, max_accuracy_drop=0.01, min_json_validity=0.995, max_ttft_ms=2000):
    if not 0 <= max_accuracy_drop <= 1 or not 0 <= min_json_validity <= 1:
        raise ValueError("invalid thresholds")
    for c in (baseline, candidate):
        if type(c.critical_safety_failures) is not int or c.critical_safety_failures < 0:
            raise ValueError("invalid safety failures")
        if not math.isfinite(c.ttft_p95_ms) or c.ttft_p95_ms < 0:
            raise ValueError("invalid latency")
        if not math.isfinite(c.gpu_memory_gib) or c.gpu_memory_gib <= 0:
            raise ValueError("invalid memory")
    base = rate(baseline.task_correct, baseline.task_total)
    accuracy = rate(candidate.task_correct, candidate.task_total)
    validity = rate(candidate.json_valid, candidate.json_total)
    gates = {
        "accuracy": accuracy >= base - max_accuracy_drop,
        "json": validity >= min_json_validity,
        "safety": candidate.critical_safety_failures == 0,
        "latency": candidate.ttft_p95_ms <= max_ttft_ms,
    }
    return {"approved": all(gates.values()), "gates": gates,
            "accuracy_delta": accuracy - base, "json_validity": validity,
            "memory_saved_gib": baseline.gpu_memory_gib - candidate.gpu_memory_gib}

if __name__ == "__main__":
    b = Candidate("bf16", 980, 1000, 1000, 1000, 0, 1400, 24)
    c = Candidate("int8", 977, 1000, 999, 1000, 0, 1300, 16)
    print(evaluate(b, c))
```

Save as `test_quality_gate.py`:
```python
import pytest
from quality_gate import Candidate, evaluate, rate
B = Candidate("bf16", 980, 1000, 1000, 1000, 0, 1400, 24)
C = Candidate("int8", 977, 1000, 999, 1000, 0, 1300, 16)

def test_approved(): assert evaluate(B, C)["approved"]
def test_memory_saved(): assert evaluate(B, C)["memory_saved_gib"] == 8
def test_accuracy_regression(): assert not evaluate(B, Candidate("bad", 900, 1000, 1000, 1000, 0, 1300, 12))["approved"]
def test_safety_block(): assert not evaluate(B, Candidate("unsafe", 990, 1000, 1000, 1000, 1, 1300, 12))["approved"]
def test_json_block(): assert not evaluate(B, Candidate("json", 990, 1000, 980, 1000, 0, 1300, 12))["approved"]
def test_latency_block(): assert not evaluate(B, Candidate("slow", 990, 1000, 1000, 1000, 0, 2500, 12))["approved"]
@pytest.mark.parametrize("ok,total", [(1,0),(-1,10),(11,10),(1.0,10)])
def test_bad_counts(ok,total):
    with pytest.raises(ValueError): rate(ok,total)
def test_invalid_memory():
    with pytest.raises(ValueError):
        evaluate(B, Candidate("bad", 980,1000,1000,1000,0,1200,0))
```

Run `python quality_gate.py && python -m pytest -q` (PowerShell: separate commands). **Expected: 11 collected cases**; actual execution not yet performed.

## Drills and acceptance
Change accuracy, JSON validity, critical safety failures and latency independently; confirm each fails closed. Compare calibration and held-out evaluation datasets, inspect subgroup regressions and model artifact signatures in staging. Review rubric: reproducibility 20, accuracy 20, safety 20, serving 20, release controls 20 (100 total). A passing synthetic gate does **not** certify a real model.
