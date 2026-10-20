# Lab 064 — Training Failure Recovery and Checkpoints Gate

**Python 3.11+ · pytest · offline synthetic recovery policy.** No GPU, storage mutations, scheduler calls or cloud DR operations.

## Goal

Fail closed on missing distributed checkpoint shards, incomplete global commit, unsafe restore geometry, checkpoint bandwidth, preemption grace, RPO/RTO, last-known-good, telemetry and security attestations. Numeric thresholds are illustrative and flags require independent evidence verification.

## File: `training_recovery_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    job: str
    expected_shards: int
    verified_shards: int
    world_size: int
    restore_world_size: int
    checkpoint_gb: float
    write_window_seconds: float
    surviving_write_gbps: float
    checkpoint_age_min: float
    rpo_min: float
    detection_min: float
    fencing_min: float
    provision_min: float
    transfer_min: float
    verify_min: float
    load_min: float
    resume_min: float
    rto_min: float
    preemption_grace_min: float
    checkpoint_commit_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    rank_failure_detection_verified: bool
    rendezvous_fencing_verified: bool
    world_size_recovery_supported: bool
    dataset_manifest_pinned: bool
    sampler_cursor_restored: bool
    optimizer_state_restored: bool
    scheduler_state_restored: bool
    rng_state_restored: bool
    mixed_precision_scaler_restored: bool
    all_rank_shards_verified: bool
    global_commit_manifest_verified: bool
    checkpoint_digest_verified: bool
    durable_independent_copy_verified: bool
    delta_chain_retention_verified: bool
    last_known_good_protected: bool
    storage_kms_access_verified: bool
    workload_identity_verified: bool
    checkpoint_encryption_verified: bool
    preemption_grace_verified: bool
    gpu_node_replacement_tested: bool
    cross_zone_restore_tested: bool
    rollback_runbook_verified: bool
    telemetry_freshness_verified: bool
    incident_drills_completed: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.job, str) or not p.job.strip():
        return {"accepted": False, "reason": "missing_job"}
    counts = (p.expected_shards, p.verified_shards, p.world_size, p.restore_world_size)
    if any(type(x) is not int or x <= 0 for x in counts):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.checkpoint_gb, p.write_window_seconds, p.surviving_write_gbps,
               p.checkpoint_age_min, p.rpo_min, p.detection_min, p.fencing_min,
               p.provision_min, p.transfer_min, p.verify_min, p.load_min,
               p.resume_min, p.rto_min, p.preemption_grace_min,
               p.checkpoint_commit_min, p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or min(p.checkpoint_gb, p.write_window_seconds, p.rpo_min,
                   p.rto_min, p.telemetry_max_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    required_write = p.checkpoint_gb / p.write_window_seconds
    measured_rto = sum((p.detection_min, p.fencing_min, p.provision_min,
                        p.transfer_min, p.verify_min, p.load_min, p.resume_min))
    gates = {
        "shard_completeness": p.verified_shards == p.expected_shards,
        "world_size_compatibility": p.restore_world_size == p.world_size or p.world_size_recovery_supported is True,
        "checkpoint_write_bandwidth": p.surviving_write_gbps >= required_write,
        "checkpoint_rpo": p.checkpoint_age_min <= p.rpo_min,
        "end_to_end_rto": measured_rto <= p.rto_min,
        "preemption_grace": p.checkpoint_commit_min <= p.preemption_grace_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "rank_failure_detection_verified": p.rank_failure_detection_verified is True,
        "rendezvous_fencing_verified": p.rendezvous_fencing_verified is True,
        "world_size_recovery_supported": p.world_size_recovery_supported is True,
        "dataset_manifest_pinned": p.dataset_manifest_pinned is True,
        "sampler_cursor_restored": p.sampler_cursor_restored is True,
        "optimizer_state_restored": p.optimizer_state_restored is True,
        "scheduler_state_restored": p.scheduler_state_restored is True,
        "rng_state_restored": p.rng_state_restored is True,
        "mixed_precision_scaler_restored": p.mixed_precision_scaler_restored is True,
        "all_rank_shards_verified": p.all_rank_shards_verified is True,
        "global_commit_manifest_verified": p.global_commit_manifest_verified is True,
        "checkpoint_digest_verified": p.checkpoint_digest_verified is True,
        "durable_independent_copy_verified": p.durable_independent_copy_verified is True,
        "delta_chain_retention_verified": p.delta_chain_retention_verified is True,
        "last_known_good_protected": p.last_known_good_protected is True,
        "storage_kms_access_verified": p.storage_kms_access_verified is True,
        "workload_identity_verified": p.workload_identity_verified is True,
        "checkpoint_encryption_verified": p.checkpoint_encryption_verified is True,
        "preemption_grace_verified": p.preemption_grace_verified is True,
        "gpu_node_replacement_tested": p.gpu_node_replacement_tested is True,
        "cross_zone_restore_tested": p.cross_zone_restore_tested is True,
        "rollback_runbook_verified": p.rollback_runbook_verified is True,
        "telemetry_freshness_verified": p.telemetry_freshness_verified is True,
        "incident_drills_completed": p.incident_drills_completed is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "required_write_gbps": round(required_write, 4),
            "measured_rto_min": round(measured_rto, 4)}

def example():
    return Plan("synthetic-recovery", 16, 16, 16, 16,
                2400.0, 300.0, 10.0, 18.0, 30.0,
                5.0, 5.0, 15.0, 30.0, 10.0, 20.0, 10.0,
                120.0, 10.0, 5.0, 10.0, 120.0,
                *([True] * 25))

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_training_recovery_policy.py`

```python
from dataclasses import replace
import pytest
from training_recovery_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_job():
    assert evaluate(replace(example(), job=" "))["reason"] == "missing_job"

def test_bool_count():
    assert evaluate(replace(example(), expected_shards=True))["reason"] == "invalid_count"

def test_fractional_count():
    assert evaluate(replace(example(), world_size=2.5))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), checkpoint_gb=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), transfer_min=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), rto_min=True))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 32

def test_rto_math():
    assert evaluate(example())["measured_rto_min"] == 95.0

def test_nonboolean_review():
    assert "independent_review_approved" in evaluate(
        replace(example(), independent_review_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("verified_shards", "shard_completeness", 15),
    ("restore_world_size", "world_size_compatibility", 8),
    ("surviving_write_gbps", "checkpoint_write_bandwidth", 7.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("transfer_min", "end_to_end_rto", 70.0),
    ("checkpoint_commit_min", "preemption_grace", 11.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    p = replace(example(), **{field: value})
    if gate == "world_size_compatibility":
        p = replace(p, world_size_recovery_supported=False)
    assert gate in evaluate(p)["failed_gates"]

@pytest.mark.parametrize("field", [
    "rank_failure_detection_verified",
    "rendezvous_fencing_verified",
    "world_size_recovery_supported",
    "dataset_manifest_pinned",
    "sampler_cursor_restored",
    "optimizer_state_restored",
    "scheduler_state_restored",
    "rng_state_restored",
    "mixed_precision_scaler_restored",
    "all_rank_shards_verified",
    "global_commit_manifest_verified",
    "checkpoint_digest_verified",
    "durable_independent_copy_verified",
    "delta_chain_retention_verified",
    "last_known_good_protected",
    "storage_kms_access_verified",
    "workload_identity_verified",
    "checkpoint_encryption_verified",
    "preemption_grace_verified",
    "gpu_node_replacement_tested",
    "cross_zone_restore_tested",
    "rollback_runbook_verified",
    "telemetry_freshness_verified",
    "incident_drills_completed",
    "independent_review_approved",
])
def test_boolean_gate(field):
    assert field in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run locally:

```bash
python -m pip install pytest
python training_recovery_policy.py
python -m pytest -q
```

**43 planned pytest cases:** 11 standalone + 7 numeric + 25 Boolean. Runtime execution and live recovery validation are pending.

## Ten synthetic recovery exercises

1. Rank loss mid-step; record gate failure and expected containment.
2. All-reduce deadlock; record gate failure and expected containment.
3. Stale rendezvous fencing; record gate failure and expected containment.
4. Node failure and replacement; record gate failure and expected containment.
5. Preemption during checkpoint; record gate failure and expected containment.
6. Partial checkpoint commit; record gate failure and expected containment.
7. Corrupt checkpoint shard; record gate failure and expected containment.
8. Broken incremental chain; record gate failure and expected containment.
9. Cross-zone restore with KMS drift; record gate failure and expected containment.
10. Monitoring blackout during RPO breach; record gate failure and expected containment.

## 100-point acceptance

Detection/fencing 20; checkpoint integrity 20; RPO/RTO and DR 20; security/observability 20; staging tests and independent review 20. Offline passing alone does not certify production.
