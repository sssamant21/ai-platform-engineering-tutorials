# Lab 056 — Checkpoint Recovery and Consistency Gate

**Python 3.11+ · pytest · offline synthetic recovery policy.** No cloud API calls, GPU jobs, checkpoint deletion or filesystem mutations.

## Goal

Reject incomplete, corrupt, stale, incompatible, insecure or unrecoverable distributed checkpoints. The Boolean inputs are *synthetic attestations*, not independently verified operational evidence.

## File: `checkpoint_recovery_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    job: str
    checkpoint_id: str
    checkpoint_gb: float
    transfer_budget_seconds: float
    surviving_read_gbps: float
    verified_shards: int
    expected_shards: int
    checkpoint_age_min: float
    rpo_min: float
    restore_minutes: float
    rto_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    manifest_signature_verified: bool
    all_shards_hashed: bool
    model_state_complete: bool
    optimizer_state_complete: bool
    scheduler_step_verified: bool
    rng_sampler_verified: bool
    dataset_release_pinned: bool
    incremental_chain_intact: bool
    object_versions_pinned: bool
    consistent_snapshot_verified: bool
    framework_compatible: bool
    resharding_supported: bool
    rank_rendezvous_tested: bool
    replay_correctness_verified: bool
    durable_replica_verified: bool
    restore_identity_scoped: bool
    kms_access_verified: bool
    tenant_isolation_verified: bool
    legal_hold_checked: bool
    checkpoint_retention_safe: bool
    checkpoint_restore_tested: bool
    telemetry_source_verified: bool
    incident_runbook_tested: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if any(not isinstance(v, str) or not v.strip() for v in (p.job, p.checkpoint_id)):
        return {"accepted": False, "reason": "missing_identity"}
    if (type(p.verified_shards) is not int or type(p.expected_shards) is not int
            or p.verified_shards < 0 or p.expected_shards <= 0):
        return {"accepted": False, "reason": "invalid_shard_count"}
    values = (p.checkpoint_gb, p.transfer_budget_seconds, p.surviving_read_gbps,
              p.checkpoint_age_min, p.rpo_min, p.restore_minutes, p.rto_min,
              p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in values)
            or min(p.checkpoint_gb, p.transfer_budget_seconds, p.rpo_min,
                   p.rto_min, p.max_telemetry_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "shard_completeness": p.verified_shards == p.expected_shards,
        "restore_bandwidth": p.surviving_read_gbps >= p.checkpoint_gb / p.transfer_budget_seconds,
        "rpo": p.checkpoint_age_min <= p.rpo_min,
        "rto": p.restore_minutes <= p.rto_min,
        "telemetry_freshness": p.telemetry_age_s <= p.max_telemetry_age_s,
        "manifest_signature_verified": p.manifest_signature_verified is True,
        "all_shards_hashed": p.all_shards_hashed is True,
        "model_state_complete": p.model_state_complete is True,
        "optimizer_state_complete": p.optimizer_state_complete is True,
        "scheduler_step_verified": p.scheduler_step_verified is True,
        "rng_sampler_verified": p.rng_sampler_verified is True,
        "dataset_release_pinned": p.dataset_release_pinned is True,
        "incremental_chain_intact": p.incremental_chain_intact is True,
        "object_versions_pinned": p.object_versions_pinned is True,
        "consistent_snapshot_verified": p.consistent_snapshot_verified is True,
        "framework_compatible": p.framework_compatible is True,
        "resharding_supported": p.resharding_supported is True,
        "rank_rendezvous_tested": p.rank_rendezvous_tested is True,
        "replay_correctness_verified": p.replay_correctness_verified is True,
        "durable_replica_verified": p.durable_replica_verified is True,
        "restore_identity_scoped": p.restore_identity_scoped is True,
        "kms_access_verified": p.kms_access_verified is True,
        "tenant_isolation_verified": p.tenant_isolation_verified is True,
        "legal_hold_checked": p.legal_hold_checked is True,
        "checkpoint_retention_safe": p.checkpoint_retention_safe is True,
        "checkpoint_restore_tested": p.checkpoint_restore_tested is True,
        "telemetry_source_verified": p.telemetry_source_verified is True,
        "incident_runbook_tested": p.incident_runbook_tested is True,
        "change_approved": p.change_approved is True,
        "rollback_tested": p.rollback_tested is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(name for name, passed in gates.items() if not passed)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "minimum_read_gbps": round(p.checkpoint_gb / p.transfer_budget_seconds, 4)}

def example():
    return Plan(
        "synthetic-training", "ckpt-000100",
        2400.0, 600.0, 5.0, 16, 16,
        5.0, 30.0, 75.0, 120.0, 10.0, 120.0,
        *([True] * 26)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_checkpoint_recovery_policy.py`

```python
from dataclasses import replace
import pytest
from checkpoint_recovery_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_job():
    assert evaluate(replace(example(), job=" "))["reason"] == "missing_identity"

def test_missing_checkpoint_id():
    assert evaluate(replace(example(), checkpoint_id=""))["reason"] == "missing_identity"

def test_bool_shards():
    assert evaluate(replace(example(), verified_shards=True))["reason"] == "invalid_shard_count"

def test_fractional_shards():
    assert evaluate(replace(example(), expected_shards=1.5))["reason"] == "invalid_shard_count"

def test_nan():
    assert evaluate(replace(example(), checkpoint_gb=float("nan")))["reason"] == "invalid_metric"

def test_infinity():
    assert evaluate(replace(example(), restore_minutes=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), rpo_min=True))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 31

def test_nonboolean_approval():
    assert "independent_review_approved" in evaluate(
        replace(example(), independent_review_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("verified_shards", "shard_completeness", 15),
    ("surviving_read_gbps", "restore_bandwidth", 3.0),
    ("checkpoint_age_min", "rpo", 31.0),
    ("restore_minutes", "rto", 121.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("manifest_signature_verified", "manifest_signature_verified"),
    ("all_shards_hashed", "all_shards_hashed"),
    ("model_state_complete", "model_state_complete"),
    ("optimizer_state_complete", "optimizer_state_complete"),
    ("scheduler_step_verified", "scheduler_step_verified"),
    ("rng_sampler_verified", "rng_sampler_verified"),
    ("dataset_release_pinned", "dataset_release_pinned"),
    ("incremental_chain_intact", "incremental_chain_intact"),
    ("object_versions_pinned", "object_versions_pinned"),
    ("consistent_snapshot_verified", "consistent_snapshot_verified"),
    ("framework_compatible", "framework_compatible"),
    ("resharding_supported", "resharding_supported"),
    ("rank_rendezvous_tested", "rank_rendezvous_tested"),
    ("replay_correctness_verified", "replay_correctness_verified"),
    ("durable_replica_verified", "durable_replica_verified"),
    ("restore_identity_scoped", "restore_identity_scoped"),
    ("kms_access_verified", "kms_access_verified"),
    ("tenant_isolation_verified", "tenant_isolation_verified"),
    ("legal_hold_checked", "legal_hold_checked"),
    ("checkpoint_retention_safe", "checkpoint_retention_safe"),
    ("checkpoint_restore_tested", "checkpoint_restore_tested"),
    ("telemetry_source_verified", "telemetry_source_verified"),
    ("incident_runbook_tested", "incident_runbook_tested"),
    ("change_approved", "change_approved"),
    ("rollback_tested", "rollback_tested"),
    ("independent_review_approved", "independent_review_approved"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run offline:

```bash
python -m pip install pytest
python checkpoint_recovery_policy.py
python -m pytest -q
```

**42 planned pytest cases:** 11 standalone + 5 numeric + 26 Boolean. Do not claim tests passed until executed. Live distributed restore, corruption injection, cross-region DR and independent review remain pending.

## Ten synthetic recovery drills

1. Missing rank shard.
2. Checksum/manifest integrity failure.
3. Broken base/delta chain.
4. RPO breach.
5. RTO breach.
6. Insufficient surviving transfer bandwidth.
7. Framework/world-size incompatibility.
8. Sampler replay mismatch.
9. KMS or isolated replica unavailable.
10. Missing rollback, change approval or independent review.

## 100-point rubric

Consistency 20; recovery capacity 20; security/governance 20; RPO/RTO/correctness 20; evidence/review 20. Require >=90 and all critical gates; synthetic pass does not certify production.
