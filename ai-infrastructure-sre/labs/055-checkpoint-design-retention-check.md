# Lab 055 — Checkpoint Design and Retention Gate

**Python 3.11+ · pytest · offline synthetic policy.** No distributed GPU processes, cloud API calls, checkpoint deletion or storage changes.

## Goal

Evaluate checkpoint completeness, shard integrity, publication consistency, RPO/RTO, write bandwidth, staging capacity, retention dependency safety, security and reviewer signoff. Boolean evidence fields are *attestations*, not verified live evidence.

## File: `checkpoint_retention_policy.py`

```python
from dataclasses import dataclass
import math
import json

@dataclass(frozen=True)
class Plan:
    job: str
    ranks: int
    expected_shards: int
    verified_shards: int
    checkpoint_gb: float
    write_window_seconds: float
    available_write_gbps: float
    staging_capacity_gb: float
    checkpoint_age_min: float
    rpo_min: float
    restore_minutes: float
    rto_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    complete_model_state: bool
    optimizer_state: bool
    scheduler_and_step: bool
    rng_sampler_state: bool
    dataset_manifest: bool
    world_size_compatible: bool
    all_rank_shards_present: bool
    checksums_verified: bool
    consistent_snapshot: bool
    immutable_manifest: bool
    conditional_commit: bool
    durable_copy_verified: bool
    incremental_chain_intact: bool
    retention_dependency_safe: bool
    legal_hold_checked: bool
    owner_approved: bool
    encryption_verified: bool
    least_privilege_verified: bool
    kubernetes_storage_verified: bool
    restore_tested: bool
    slo_verified: bool
    incident_drill_tested: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.job, str) or not p.job.strip():
        return {"accepted": False, "reason": "missing_job"}
    counts = (p.ranks, p.expected_shards, p.verified_shards)
    if any(type(v) is not int or v <= 0 for v in counts):
        return {"accepted": False, "reason": "invalid_count"}
    values = (p.checkpoint_gb, p.write_window_seconds, p.available_write_gbps,
              p.staging_capacity_gb, p.checkpoint_age_min, p.rpo_min,
              p.restore_minutes, p.rto_min, p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in values)
            or min(p.checkpoint_gb, p.write_window_seconds, p.rpo_min,
                   p.rto_min, p.max_telemetry_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "shard_completeness": p.verified_shards == p.expected_shards,
        "write_bandwidth": p.available_write_gbps >= p.checkpoint_gb / p.write_window_seconds,
        "staging_capacity": p.staging_capacity_gb >= p.checkpoint_gb,
        "rpo": p.checkpoint_age_min <= p.rpo_min,
        "rto": p.restore_minutes <= p.rto_min,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "complete_model_state": p.complete_model_state is True,
        "optimizer_state": p.optimizer_state is True,
        "scheduler_and_step": p.scheduler_and_step is True,
        "rng_sampler_state": p.rng_sampler_state is True,
        "dataset_manifest": p.dataset_manifest is True,
        "world_size_compatible": p.world_size_compatible is True,
        "all_rank_shards_present": p.all_rank_shards_present is True,
        "checksums_verified": p.checksums_verified is True,
        "consistent_snapshot": p.consistent_snapshot is True,
        "immutable_manifest": p.immutable_manifest is True,
        "conditional_commit": p.conditional_commit is True,
        "durable_copy_verified": p.durable_copy_verified is True,
        "incremental_chain_intact": p.incremental_chain_intact is True,
        "retention_dependency_safe": p.retention_dependency_safe is True,
        "legal_hold_checked": p.legal_hold_checked is True,
        "owner_approved": p.owner_approved is True,
        "encryption_verified": p.encryption_verified is True,
        "least_privilege_verified": p.least_privilege_verified is True,
        "kubernetes_storage_verified": p.kubernetes_storage_verified is True,
        "restore_tested": p.restore_tested is True,
        "slo_verified": p.slo_verified is True,
        "incident_drill_tested": p.incident_drill_tested is True,
        "change_approved": p.change_approved is True,
        "rollback_tested": p.rollback_tested is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, passed in gates.items() if not passed)
    return {"accepted": not failed, "failed_gates": failed, "total_gates": len(gates)}

def example():
    return Plan(
        "synthetic-training-checkpoint",
        4, 4, 4, 2400.0, 300.0, 10.0, 3000.0,
        5.0, 30.0, 60.0, 120.0, 10.0, 120.0,
        *([True] * 25)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_checkpoint_retention_policy.py`

```python
from dataclasses import replace
import pytest
from checkpoint_retention_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_empty_job():
    assert evaluate(replace(example(), job=" "))["reason"] == "missing_job"

def test_bool_rank():
    assert evaluate(replace(example(), ranks=True))["reason"] == "invalid_count"

def test_fractional_shard_count():
    assert evaluate(replace(example(), expected_shards=1.5))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), checkpoint_gb=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), restore_minutes=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), rpo_min=True))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 31

def test_nonboolean_signoff():
    assert "independent_review_approved" in evaluate(replace(example(), independent_review_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("verified_shards", "shard_completeness", 3),
    ("available_write_gbps", "write_bandwidth", 1.0),
    ("staging_capacity_gb", "staging_capacity", 100.0),
    ("checkpoint_age_min", "rpo", 31.0),
    ("restore_minutes", "rto", 121.0),
    ("telemetry_age_s", "telemetry", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("complete_model_state", "complete_model_state"),
    ("optimizer_state", "optimizer_state"),
    ("scheduler_and_step", "scheduler_and_step"),
    ("rng_sampler_state", "rng_sampler_state"),
    ("dataset_manifest", "dataset_manifest"),
    ("world_size_compatible", "world_size_compatible"),
    ("all_rank_shards_present", "all_rank_shards_present"),
    ("checksums_verified", "checksums_verified"),
    ("consistent_snapshot", "consistent_snapshot"),
    ("immutable_manifest", "immutable_manifest"),
    ("conditional_commit", "conditional_commit"),
    ("durable_copy_verified", "durable_copy_verified"),
    ("incremental_chain_intact", "incremental_chain_intact"),
    ("retention_dependency_safe", "retention_dependency_safe"),
    ("legal_hold_checked", "legal_hold_checked"),
    ("owner_approved", "owner_approved"),
    ("encryption_verified", "encryption_verified"),
    ("least_privilege_verified", "least_privilege_verified"),
    ("kubernetes_storage_verified", "kubernetes_storage_verified"),
    ("restore_tested", "restore_tested"),
    ("slo_verified", "slo_verified"),
    ("incident_drill_tested", "incident_drill_tested"),
    ("change_approved", "change_approved"),
    ("rollback_tested", "rollback_tested"),
    ("independent_review_approved", "independent_review_approved"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python checkpoint_retention_policy.py
python -m pytest -q
```

**41 planned pytest cases:** 10 standalone + 6 numeric + 25 Boolean. Tests are not claimed as executed. No live cloud/GPU/DR verification has been performed.

## Ten failure drills

1. Missing rank shard.
2. Insufficient write bandwidth.
3. Insufficient staging capacity.
4. RPO breach.
5. Restore RTO breach.
6. Stale telemetry.
7. Missing optimizer or sampler state.
8. Corrupted or partial commit manifest.
9. Unsafe incremental retention or legal-hold violation.
10. Missing owner, change, rollback or independent approval.

## 100-point review

State/consistency 20; throughput/capacity 20; security/governance 20; RPO/RTO/restore 20; test evidence/review 20. Require >=90, every critical gate green and independent staging validation for production signoff.
