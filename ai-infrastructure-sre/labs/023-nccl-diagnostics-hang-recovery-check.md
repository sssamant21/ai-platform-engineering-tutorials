# Lab 023 — NCCL Hang Triage and Recovery Policy

**Python 3.11+ · pytest · offline synthetic model.** No GPU, network, Kubernetes, or checkpoint mutations occur.

## Goal and safety model

Validate that a distributed job has complete rank evidence, compatible collective sequence, bounded progress age, known failure classification, a verified checkpoint and explicit approval for recovery. **A passing result is only a synthetic policy gate**, not a claim that NCCL can be resumed safely. Real recovery must follow the exact framework's communicator and checkpoint contracts.

## File: `nccl_hang_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Rank:
    rank: int
    gpu_uuid: str
    last_step: int
    heartbeat_age_s: float
    last_collective: str
    process_alive: bool
    cuda_healthy: bool

@dataclass(frozen=True)
class Incident:
    job: str
    world_size: int
    ranks: tuple[Rank, ...]
    max_heartbeat_age_s: float
    max_step_skew: int
    network_healthy: bool
    no_xid: bool
    checkpoint_verified: bool
    checkpoint_step: int
    telemetry_fresh: bool
    root_cause_classified: bool
    all_ranks_quiesced: bool
    restart_approved: bool
    rollback_tested: bool
    workload_slo_verified: bool

def evaluate(i):
    if not isinstance(i, Incident):
        raise TypeError("Incident required")
    if not isinstance(i.job, str) or not i.job.strip():
        return {"accepted": False, "reason": "missing_job"}
    if (type(i.world_size) is not int or i.world_size < 2
            or type(i.max_step_skew) is not int or i.max_step_skew < 0
            or type(i.checkpoint_step) is not int or i.checkpoint_step < 0):
        return {"accepted": False, "reason": "invalid_policy"}
    if (type(i.max_heartbeat_age_s) not in (int, float)
            or not math.isfinite(i.max_heartbeat_age_s)
            or i.max_heartbeat_age_s <= 0):
        return {"accepted": False, "reason": "invalid_heartbeat_limit"}
    if not isinstance(i.ranks, tuple) or any(not isinstance(r, Rank) for r in i.ranks):
        return {"accepted": False, "reason": "invalid_ranks"}
    ids = [r.rank for r in i.ranks]
    if (len(set(ids)) != len(ids)
            or len({r.gpu_uuid for r in i.ranks}) != len(i.ranks)
            or any(type(r.rank) is not int or not 0 <= r.rank < i.world_size
                   or not isinstance(r.gpu_uuid, str) or not r.gpu_uuid.startswith("GPU-")
                   or type(r.last_step) is not int or r.last_step < 0
                   or not isinstance(r.last_collective, str) or not r.last_collective
                   or type(r.heartbeat_age_s) not in (int, float)
                   or not math.isfinite(r.heartbeat_age_s)
                   or r.heartbeat_age_s < 0
                   for r in i.ranks)):
        return {"accepted": False, "reason": "invalid_rank_evidence"}
    steps = [r.last_step for r in i.ranks]
    gates = {
        "all_ranks": len(i.ranks) == i.world_size and set(ids) == set(range(i.world_size)),
        "heartbeat": all(r.heartbeat_age_s <= i.max_heartbeat_age_s for r in i.ranks),
        "step_skew": bool(steps) and max(steps) - min(steps) <= i.max_step_skew,
        "collective_sequence": len({r.last_collective for r in i.ranks}) == 1,
        "process_alive": all(r.process_alive is True for r in i.ranks),
        "cuda_healthy": all(r.cuda_healthy is True for r in i.ranks),
        "network": i.network_healthy is True,
        "xid": i.no_xid is True,
        "checkpoint": i.checkpoint_verified is True and bool(steps)
                      and i.checkpoint_step <= min(steps),
        "telemetry": i.telemetry_fresh is True,
        "classified": i.root_cause_classified is True,
        "quiesced": i.all_ranks_quiesced is True,
        "approval": i.restart_approved is True,
        "rollback": i.rollback_tested is True,
        "slo": i.workload_slo_verified is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "rank_count": len(i.ranks), "checkpoint_step": i.checkpoint_step,
            "observed_step_range": [min(steps), max(steps)] if steps else []}

def example():
    ranks = (
        Rank(0, "GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee", 100, 5.0, "allreduce-100", True, True),
        Rank(1, "GPU-ffffffff-bbbb-cccc-dddd-eeeeeeeeeeee", 100, 6.0, "allreduce-100", True, True),
    )
    return Incident("staging-training", 2, ranks, 120.0, 0,
                    True, True, True, 100, True, True, True, True, True, True)

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_nccl_hang_policy.py`

```python
from dataclasses import replace
import pytest
from nccl_hang_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_job():
    assert evaluate(replace(example(), job=" "))["reason"] == "missing_job"

def test_invalid_world_size():
    assert evaluate(replace(example(), world_size=1))["reason"] == "invalid_policy"

def test_boolean_world_size():
    assert evaluate(replace(example(), world_size=True))["reason"] == "invalid_policy"

def test_negative_checkpoint():
    assert evaluate(replace(example(), checkpoint_step=-1))["reason"] == "invalid_policy"

def test_invalid_heartbeat_limit():
    assert evaluate(replace(example(), max_heartbeat_age_s=float("nan")))["reason"] == "invalid_heartbeat_limit"

def test_missing_rank():
    s = example()
    assert "all_ranks" in evaluate(replace(s, ranks=s.ranks[:1]))["failed_gates"]

def test_duplicate_rank():
    s = example()
    assert evaluate(replace(s, ranks=(s.ranks[0], s.ranks[0])))["reason"] == "invalid_rank_evidence"

def test_invalid_gpu_uuid():
    s = example()
    r = replace(s.ranks[1], gpu_uuid="bad")
    assert evaluate(replace(s, ranks=(s.ranks[0], r)))["reason"] == "invalid_rank_evidence"

def test_stale_heartbeat():
    s = example()
    r = replace(s.ranks[1], heartbeat_age_s=200.0)
    assert "heartbeat" in evaluate(replace(s, ranks=(s.ranks[0], r)))["failed_gates"]

def test_step_skew():
    s = example()
    r = replace(s.ranks[1], last_step=98)
    assert "step_skew" in evaluate(replace(s, ranks=(s.ranks[0], r)))["failed_gates"]

def test_collective_mismatch():
    s = example()
    r = replace(s.ranks[1], last_collective="broadcast-100")
    assert "collective_sequence" in evaluate(replace(s, ranks=(s.ranks[0], r)))["failed_gates"]

def test_dead_rank():
    s = example()
    r = replace(s.ranks[1], process_alive=False)
    assert "process_alive" in evaluate(replace(s, ranks=(s.ranks[0], r)))["failed_gates"]

def test_cuda_fault():
    s = example()
    r = replace(s.ranks[1], cuda_healthy=False)
    assert "cuda_healthy" in evaluate(replace(s, ranks=(s.ranks[0], r)))["failed_gates"]

def test_checkpoint_ahead():
    assert "checkpoint" in evaluate(replace(example(), checkpoint_step=101))["failed_gates"]

def test_nonboolean_approval():
    assert "approval" in evaluate(replace(example(), restart_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("network_healthy", "network"),
    ("no_xid", "xid"),
    ("checkpoint_verified", "checkpoint"),
    ("telemetry_fresh", "telemetry"),
    ("root_cause_classified", "classified"),
    ("all_ranks_quiesced", "quiesced"),
    ("restart_approved", "approval"),
    ("rollback_tested", "rollback"),
    ("workload_slo_verified", "slo"),
])
def test_hard_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python nccl_hang_policy.py
python -m pytest -q
```

**Expected 26 pytest cases:** 17 standalone plus 9 parameterized. Execution is pending.

## Eight offline incident drills

1. Remove rank 1 and confirm incomplete world-size gate.
2. Make one heartbeat stale and confirm progress rejection.
3. Diverge rank collective sequence.
4. Skew global steps beyond configured tolerance.
5. Mark a rank's CUDA context unhealthy.
6. Mark network or Xid state unhealthy.
7. Set checkpoint ahead of the observed rank step.
8. Remove coordinated quiesce/approval/rollback evidence.

## Optional live staging validation

1. Inventory GPU UUIDs, driver, CUDA, NCCL, NIC, topology, node and pod identity.
2. Run an approved, bounded `nccl-tests` AllReduce smoke and record numeric correctness.
3. Exercise a **disposable** distributed training job with known-good checkpoint and explicit owner approval.
4. Simulate loss of one disposable rank and verify watchdog, all-rank shutdown, checkpoint restoration and progress.
5. Compare baseline throughput, checkpoint age, RTO, error rate and application SLO.
6. Preserve logs with secret redaction and document root-cause classification.

**Important limitation:** this validator models *evidence completeness and restart authorization*, not whether a live communicator is safe to resume. Rank health and “quiesced” are synthetic attestations; real operations require independently verified signals and framework-specific restart logic.

## 100-point assessment

Failure classification 20; all-rank diagnostics 20; coordinated recovery 20; incident runbooks 20; executable evidence 20. Pass >=90 with no critical correctness, security, checkpoint or SLO blocker.
