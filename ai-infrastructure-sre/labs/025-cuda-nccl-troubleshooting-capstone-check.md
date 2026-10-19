# Lab 025 — CUDA and NCCL Production Readiness Gate

**Python 3.11+ · pytest · offline synthetic evaluator.** No GPU, Kubernetes, driver, network, checkpoint or production mutations occur.

## Objectives

Evaluate a compound CUDA/NCCL incident against compatibility, memory headroom, collective consistency, GPU health, network/fabric readiness, checkpoint correctness, telemetry, coordinated recovery and SLO acceptance. Fail closed on unknown or missing evidence. Passing synthetic policy is **not** equivalent to staging hardware acceptance.

## File: `capstone_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Rank:
    rank: int
    gpu_uuid: str
    step: int
    collective: str
    healthy: bool

@dataclass(frozen=True)
class Acceptance:
    incident: str
    world_size: int
    ranks: tuple[Rank, ...]
    driver_cuda_compatible: bool
    nccl_framework_compatible: bool
    memory_headroom_gb: float
    min_memory_headroom_gb: float
    mig_topology_verified: bool
    nvlink_p2p_verified: bool
    network_healthy: bool
    no_xid: bool
    no_ecc_uncorrectable: bool
    checkpoint_verified: bool
    checkpoint_step: int
    all_ranks_quiesced: bool
    telemetry_age_s: float
    max_telemetry_age_s: float
    profiling_baseline_matched: bool
    workload_correctness_verified: bool
    slo_verified: bool
    n_minus_one_verified: bool
    security_approved: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(a):
    if not isinstance(a, Acceptance):
        raise TypeError("Acceptance required")
    if not isinstance(a.incident, str) or not a.incident.strip():
        return {"accepted": False, "reason": "missing_incident"}
    if (type(a.world_size) is not int or a.world_size < 2
            or type(a.checkpoint_step) is not int or a.checkpoint_step < 0):
        return {"accepted": False, "reason": "invalid_policy"}
    if not isinstance(a.ranks, tuple) or any(not isinstance(r, Rank) for r in a.ranks):
        return {"accepted": False, "reason": "invalid_ranks"}
    ids = [r.rank for r in a.ranks]
    if (len(ids) != len(set(ids))
            or len({r.gpu_uuid for r in a.ranks}) != len(a.ranks)
            or any(type(r.rank) is not int or not 0 <= r.rank < a.world_size
                   or not isinstance(r.gpu_uuid, str) or not r.gpu_uuid.startswith("GPU-")
                   or type(r.step) is not int or r.step < 0
                   or not isinstance(r.collective, str) or not r.collective
                   for r in a.ranks)):
        return {"accepted": False, "reason": "invalid_rank_identity"}
    numbers = (a.memory_headroom_gb, a.min_memory_headroom_gb,
               a.telemetry_age_s, a.max_telemetry_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) for v in numbers)
            or a.memory_headroom_gb < 0 or a.min_memory_headroom_gb < 0
            or a.telemetry_age_s < 0 or a.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_measurement"}
    steps = [r.step for r in a.ranks]
    gates = {
        "all_ranks": len(a.ranks) == a.world_size and set(ids) == set(range(a.world_size)),
        "collective": len({r.collective for r in a.ranks}) == 1,
        "rank_health": all(r.healthy is True for r in a.ranks),
        "step_alignment": len(set(steps)) == 1,
        "driver_cuda": a.driver_cuda_compatible is True,
        "nccl_framework": a.nccl_framework_compatible is True,
        "memory": a.memory_headroom_gb >= a.min_memory_headroom_gb,
        "mig_topology": a.mig_topology_verified is True,
        "nvlink_p2p": a.nvlink_p2p_verified is True,
        "network": a.network_healthy is True,
        "xid": a.no_xid is True,
        "ecc": a.no_ecc_uncorrectable is True,
        "checkpoint": a.checkpoint_verified is True and bool(steps)
                      and a.checkpoint_step <= min(steps),
        "quiesced": a.all_ranks_quiesced is True,
        "telemetry": a.telemetry_age_s <= a.max_telemetry_age_s,
        "profiling": a.profiling_baseline_matched is True,
        "correctness": a.workload_correctness_verified is True,
        "slo": a.slo_verified is True,
        "n_minus_one": a.n_minus_one_verified is True,
        "security": a.security_approved is True,
        "change": a.change_approved is True,
        "rollback": a.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "world_size": a.world_size, "reported_ranks": len(a.ranks),
            "checkpoint_step": a.checkpoint_step}

def example():
    ranks = (
        Rank(0, "GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee", 100, "allreduce-100", True),
        Rank(1, "GPU-ffffffff-bbbb-cccc-dddd-eeeeeeeeeeee", 100, "allreduce-100", True),
    )
    return Acceptance(
        "synthetic-capstone", 2, ranks,
        True, True, 8.0, 4.0, True, True, True, True, True,
        True, 100, True, 10.0, 120.0,
        True, True, True, True, True, True, True
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_capstone_policy.py`

```python
from dataclasses import replace
import pytest
from capstone_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_incident():
    assert evaluate(replace(example(), incident=" "))["reason"] == "missing_incident"

def test_world_size():
    assert evaluate(replace(example(), world_size=1))["reason"] == "invalid_policy"

def test_boolean_world_size():
    assert evaluate(replace(example(), world_size=True))["reason"] == "invalid_policy"

def test_checkpoint_policy():
    assert evaluate(replace(example(), checkpoint_step=-1))["reason"] == "invalid_policy"

def test_missing_rank():
    s = example()
    assert "all_ranks" in evaluate(replace(s, ranks=s.ranks[:1]))["failed_gates"]

def test_duplicate_rank():
    s = example()
    assert evaluate(replace(s, ranks=(s.ranks[0], s.ranks[0])))["reason"] == "invalid_rank_identity"

def test_collective_mismatch():
    s = example()
    r = replace(s.ranks[1], collective="broadcast-100")
    assert "collective" in evaluate(replace(s, ranks=(s.ranks[0], r)))["failed_gates"]

def test_step_skew():
    s = example()
    r = replace(s.ranks[1], step=99)
    assert "step_alignment" in evaluate(replace(s, ranks=(s.ranks[0], r)))["failed_gates"]

def test_rank_unhealthy():
    s = example()
    r = replace(s.ranks[1], healthy=False)
    assert "rank_health" in evaluate(replace(s, ranks=(s.ranks[0], r)))["failed_gates"]

def test_low_memory():
    assert "memory" in evaluate(replace(example(), memory_headroom_gb=2.0))["failed_gates"]

def test_nan_memory():
    assert evaluate(replace(example(), memory_headroom_gb=float("nan")))["reason"] == "invalid_measurement"

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_checkpoint_ahead():
    assert "checkpoint" in evaluate(replace(example(), checkpoint_step=101))["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("driver_cuda_compatible", "driver_cuda"),
    ("nccl_framework_compatible", "nccl_framework"),
    ("mig_topology_verified", "mig_topology"),
    ("nvlink_p2p_verified", "nvlink_p2p"),
    ("network_healthy", "network"),
    ("no_xid", "xid"),
    ("no_ecc_uncorrectable", "ecc"),
    ("checkpoint_verified", "checkpoint"),
    ("all_ranks_quiesced", "quiesced"),
    ("profiling_baseline_matched", "profiling"),
    ("workload_correctness_verified", "correctness"),
    ("slo_verified", "slo"),
    ("n_minus_one_verified", "n_minus_one"),
    ("security_approved", "security"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_hard_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python capstone_policy.py
python -m pytest -q
```

**Expected: 32 pytest cases** (16 standalone plus 16 parameterized). Tests have not been executed as part of publication.

## Ten synthetic game days

1. Reject driver/CUDA mismatch.
2. Reject incompatible NCCL/framework.
3. Reject GPU memory headroom shortfall.
4. Reject invalid MIG/NVLink topology.
5. Reject unhealthy network or Xid/ECC.
6. Reject missing rank or collective mismatch.
7. Reject unaligned rank steps.
8. Reject stale telemetry or profiling mismatch.
9. Reject checkpoint ahead of progress.
10. Reject missing security, approval, rollback or SLO evidence.

## Live staging evidence required

- GPU/driver/CUDA/NCCL version and immutable image inventory.
- CUDA P2P smoke and NCCL AllReduce correctness.
- Supported MIG/fabric topology and NIC path.
- All-rank progress and checkpoint integrity.
- Bounded Nsight profiling and workload SLO comparison.
- Disposable rank-failure simulation and coordinated restart.
- Security/owner approval, N-1 capacity and rollback test.
- Incident report with timeline, RCA, COA and acceptance signoff.

**Important:** Synthetic success does not authorize a production driver update, MIG change, GPU reset or network disruption.

## 100-point rubric

Architecture/compatibility 20; collective diagnosis 20; coordinated recovery 20; performance/telemetry 20; executable lab and RCA 20. Pass >=90 with no critical safety or correctness failure.
