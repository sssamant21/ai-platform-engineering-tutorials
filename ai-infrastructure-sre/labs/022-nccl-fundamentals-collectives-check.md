# Lab 022 — NCCL Collective Contract and Reliability Gate

**Python 3.11+ · pytest.** This is an offline **synthetic** collective contract check. It does not allocate GPU memory, launch NCCL or test RDMA.

## Design

Each rank reports its intended sequence of collectives. The validator checks world size, unique ranks, ordered operations, operation counts/dtypes/root constraints, freshness, readiness, checkpoint and SLO gates. In production, real collective behavior, rank failures, network partitions and timeouts require framework-specific distributed tests.

## File: `collective_policy.py`

```python
from dataclasses import dataclass
import json
import math

OPS = frozenset({"allreduce", "broadcast", "reduce", "allgather", "reducescatter"})
DTYPES = frozenset({"float16", "float32", "bfloat16", "int32"})

@dataclass(frozen=True)
class Collective:
    kind: str
    count: int
    dtype: str
    root: int | None = None

@dataclass(frozen=True)
class RankReport:
    rank: int
    gpu_uuid: str
    operations: tuple[Collective, ...]
    ready: bool

@dataclass(frozen=True)
class Job:
    world_size: int
    ranks: tuple[RankReport, ...]
    cuda_nccl_compatible: bool
    topology_verified: bool
    network_verified: bool
    checkpoint_verified: bool
    telemetry_age_s: float
    max_telemetry_age_s: float
    no_xid: bool
    change_approved: bool
    rollback_tested: bool
    workload_slo_healthy: bool

def evaluate(job):
    if not isinstance(job, Job):
        raise TypeError("Job required")
    if type(job.world_size) is not int or job.world_size < 2:
        return {"accepted": False, "reason": "invalid_world_size"}
    if not isinstance(job.ranks, tuple) or any(not isinstance(r, RankReport) for r in job.ranks):
        return {"accepted": False, "reason": "invalid_ranks"}
    ids = [r.rank for r in job.ranks]
    if (any(type(r.rank) is not int or r.rank < 0 or r.rank >= job.world_size
            or not isinstance(r.gpu_uuid, str) or not r.gpu_uuid.startswith("GPU-")
            for r in job.ranks)
            or len(ids) != len(set(ids))
            or len({r.gpu_uuid for r in job.ranks}) != len(job.ranks)):
        return {"accepted": False, "reason": "invalid_rank_identity"}
    if (any(not isinstance(r.operations, tuple) or not r.operations for r in job.ranks)
            or any(not isinstance(o, Collective)
                   or o.kind not in OPS or type(o.count) is not int or o.count <= 0
                   or o.dtype not in DTYPES
                   or (o.kind in {"broadcast", "reduce"} and
                       (type(o.root) is not int or not 0 <= o.root < job.world_size))
                   or (o.kind not in {"broadcast", "reduce"} and o.root is not None)
                   or (o.kind == "reducescatter" and o.count % job.world_size != 0)
                   for r in job.ranks for o in r.operations)):
        return {"accepted": False, "reason": "invalid_collective"}
    ages = (job.telemetry_age_s, job.max_telemetry_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) for v in ages)
            or job.telemetry_age_s < 0 or job.max_telemetry_age_s <= 0):
        return {"accepted": False, "reason": "invalid_telemetry"}
    complete = len(job.ranks) == job.world_size and set(ids) == set(range(job.world_size))
    ordered = sorted(job.ranks, key=lambda r: r.rank)
    reference = ordered[0].operations if ordered else ()
    gates = {
        "all_ranks": complete,
        "matching_sequence": all(r.operations == reference for r in ordered),
        "rank_readiness": all(r.ready is True for r in ordered),
        "compatibility": job.cuda_nccl_compatible is True,
        "topology": job.topology_verified is True,
        "network": job.network_verified is True,
        "checkpoint": job.checkpoint_verified is True,
        "telemetry": job.telemetry_age_s <= job.max_telemetry_age_s,
        "xid": job.no_xid is True,
        "approval": job.change_approved is True,
        "rollback": job.rollback_tested is True,
        "slo": job.workload_slo_healthy is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "world_size": job.world_size, "reported_ranks": len(job.ranks)}

def example():
    op = (Collective("allreduce", 1024, "float32"),)
    ranks = (
        RankReport(0, "GPU-aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee", op, True),
        RankReport(1, "GPU-ffffffff-bbbb-cccc-dddd-eeeeeeeeeeee", op, True),
    )
    return Job(2, ranks, True, True, True, True, 10.0, 120.0,
               True, True, True, True)

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_collective_policy.py`

```python
from dataclasses import replace
import pytest
from collective_policy import Collective, RankReport, evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_world_size():
    assert evaluate(replace(example(), world_size=1))["reason"] == "invalid_world_size"

def test_bool_world_size():
    assert evaluate(replace(example(), world_size=True))["reason"] == "invalid_world_size"

def test_missing_rank():
    s = example()
    assert "all_ranks" in evaluate(replace(s, ranks=s.ranks[:1]))["failed_gates"]

def test_duplicate_rank():
    s = example()
    assert evaluate(replace(s, ranks=(s.ranks[0], s.ranks[0])))["reason"] == "invalid_rank_identity"

def test_duplicate_gpu():
    s = example()
    other = replace(s.ranks[1], gpu_uuid=s.ranks[0].gpu_uuid)
    assert evaluate(replace(s, ranks=(s.ranks[0], other)))["reason"] == "invalid_rank_identity"

def test_rank_out_of_range():
    s = example()
    assert evaluate(replace(s, ranks=(s.ranks[0], replace(s.ranks[1], rank=2))))["reason"] == "invalid_rank_identity"

def test_mismatched_sequence():
    s = example()
    other = replace(s.ranks[1], operations=(Collective("allgather", 1024, "float32"),))
    assert "matching_sequence" in evaluate(replace(s, ranks=(s.ranks[0], other)))["failed_gates"]

def test_bad_count():
    s = example()
    r = replace(s.ranks[0], operations=(Collective("allreduce", 0, "float32"),))
    assert evaluate(replace(s, ranks=(r, s.ranks[1])))["reason"] == "invalid_collective"

def test_bad_root():
    s = example()
    r = replace(s.ranks[0], operations=(Collective("broadcast", 10, "float32", root=3),))
    assert evaluate(replace(s, ranks=(r, s.ranks[1])))["reason"] == "invalid_collective"

def test_reducescatter_divisibility():
    s = example()
    r = replace(s.ranks[0], operations=(Collective("reducescatter", 3, "float32"),))
    assert evaluate(replace(s, ranks=(r, s.ranks[1])))["reason"] == "invalid_collective"

def test_invalid_dtype():
    s = example()
    r = replace(s.ranks[0], operations=(Collective("allreduce", 10, "unknown"),))
    assert evaluate(replace(s, ranks=(r, s.ranks[1])))["reason"] == "invalid_collective"

def test_nan_age():
    assert evaluate(replace(example(), telemetry_age_s=float("nan")))["reason"] == "invalid_telemetry"

def test_stale():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_readiness():
    s = example()
    other = replace(s.ranks[1], ready=False)
    assert "rank_readiness" in evaluate(replace(s, ranks=(s.ranks[0], other)))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("cuda_nccl_compatible", "compatibility"),
    ("topology_verified", "topology"),
    ("network_verified", "network"),
    ("checkpoint_verified", "checkpoint"),
    ("no_xid", "xid"),
    ("change_approved", "approval"),
    ("rollback_tested", "rollback"),
    ("workload_slo_healthy", "slo"),
])
def test_hard_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_approval():
    assert "approval" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python collective_policy.py
python -m pytest -q
```

**Expected: 25 test cases** (17 standalone and 8 parameterized). Execution has not been performed as part of publication.

## Synthetic incident drills
1. Omit rank 1 and verify the job fails closed.
2. Change the collective order on one rank.
3. Change a reduction datatype or count.
4. Supply a root outside world size.
5. Mark a GPU rank not ready.
6. Fail NCCL compatibility or topology evidence.
7. Age telemetry beyond its permitted threshold.
8. Fail checkpoint, approval or rollback evidence.

## Optional live GPU staging
1. Confirm GPU UUIDs, driver, CUDA/NCCL versions, node topology and NIC.
2. Compile and run Chapter 022's single-node two-GPU AllReduce smoke.
3. Run an approved `nccl-tests` version with bounded message sizes and log exact options.
4. Check collective numerical correctness, completion, throughput, GPU/NIC errors and SLO.
5. If approved for multi-node, test the actual launcher/network path, checkpoint and coordinated recovery.
6. Attach evidence, security review and owner signoff.

## 100-point assessment
Collective semantics 20; rank correctness 20; topology/transport 20; incident runbooks 20; tests and evidence 20. Pass >=90 with no critical correctness, security or recovery blocker.
