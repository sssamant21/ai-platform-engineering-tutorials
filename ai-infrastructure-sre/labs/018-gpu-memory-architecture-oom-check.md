# Chapter 018 — GPU Memory and OOM Executable Lab

**Status:** Published; runtime execution and real GPU qualification pending.

Use Python 3.11+ and `pytest`. This is a **synthetic admission model**, not an NVIDIA device test. Save the following as `gpu_memory.py`:

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Snapshot:
    total_mib: int
    free_mib: int
    weights_mib: int
    activations_mib: int
    kv_mib: int
    workspace_mib: int
    safety_mib: int
    fresh: bool
    cuda_oom: bool
    host_oom: bool
    mig_verified: bool
    approved: bool

def evaluate(s):
    if not isinstance(s, Snapshot):
        raise TypeError("Snapshot required")
    numbers = (s.total_mib, s.free_mib, s.weights_mib,
               s.activations_mib, s.kv_mib, s.workspace_mib, s.safety_mib)
    if any(type(v) is not int or v < 0 for v in numbers) or s.total_mib == 0:
        return {"accepted": False, "reason": "invalid_memory"}
    if s.free_mib > s.total_mib:
        return {"accepted": False, "reason": "free_exceeds_total"}
    required = sum(numbers[2:])
    gates = {
        "total": required <= s.total_mib,
        "free": required <= s.free_mib,
        "fresh": s.fresh is True,
        "cuda": s.cuda_oom is False,
        "host": s.host_oom is False,
        "mig": s.mig_verified is True,
        "approval": s.approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "required_mib": required, "headroom_mib": s.free_mib - required}

def example():
    return Snapshot(81920, 75000, 16000, 10000, 12000, 4000,
                    8000, True, False, False, True, True)
```

Save as `test_gpu_memory.py`:

```python
from dataclasses import replace
import pytest
from gpu_memory import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_negative():
    assert evaluate(replace(example(), kv_mib=-1))["reason"] == "invalid_memory"

def test_bool_memory():
    assert evaluate(replace(example(), total_mib=True))["reason"] == "invalid_memory"

def test_zero_total():
    assert evaluate(replace(example(), total_mib=0))["reason"] == "invalid_memory"

def test_invalid_free():
    assert evaluate(replace(example(), free_mib=90000))["reason"] == "free_exceeds_total"

def test_headroom():
    assert evaluate(example())["headroom_mib"] == 25000

def test_over_budget():
    assert "total" in evaluate(replace(example(), weights_mib=90000))["failed_gates"]

def test_over_free():
    assert "free" in evaluate(replace(example(), free_mib=10000))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("fresh", "fresh", False),
    ("cuda_oom", "cuda", True),
    ("host_oom", "host", True),
    ("mig_verified", "mig", False),
    ("approved", "approval", False),
])
def test_gates(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

def test_nonboolean_approval():
    assert "approval" in evaluate(replace(example(), approved="yes"))["failed_gates"]
```

Run `python -m pip install pytest` then `python -m pytest -q`. Expected **15 test cases**; not executed here.

### Production SRE exercise
1. Collect GPU UUID and MIG profile, `nvidia-smi` memory and PyTorch allocated/reserved memory.
2. Distinguish CUDA device OOM from host cgroup OOMKilled and driver/Xid faults.
3. Under approved staging load, vary batch size, KV-cache capacity and context length.
4. Recalculate **prospective** allocation budget from measured usage; do not double-count resident model weights.
5. Verify workload SLO, checkpoint recovery, telemetry freshness, backpressure and rollback.
6. Preserve change ticket, UTC evidence and independent reviewer signoff.

### 100-point rubric
Architecture 20, OOM diagnosis 20, memory admission 20, incidents/runbooks 20, executable evidence 20. Synthetic passing tests do not establish production acceptance.
