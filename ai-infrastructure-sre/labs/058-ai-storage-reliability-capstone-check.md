# Lab 058 — AI Storage Reliability Capstone Gate

**Python 3.11+ · pytest · offline synthetic policy.** No cloud API calls, storage mutations, GPU jobs, billing access or fault injection.

## Goal

Integrate fail-closed checks across dataset integrity, Lustre/CSI, loader correctness, degraded-mode throughput/IOPS, checkpoint write/recovery, security, retention, telemetry, cost and change governance. Numeric thresholds are synthetic site budgets; Boolean flags represent attestations only.

## File: `storage_capstone_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    assessment: str
    workers: int
    worker_read_gbps: float
    cache_miss_fraction: float
    burst_factor: float
    avg_read_kib: float
    surviving_read_gbps: float
    surviving_iops: float
    checkpoint_gb: float
    checkpoint_window_seconds: float
    surviving_write_gbps: float
    available_capacity_gb: float
    required_capacity_gb: float
    gpu_wait_p95_ms: float
    gpu_wait_slo_ms: float
    checkpoint_age_min: float
    rpo_min: float
    restore_minutes: float
    rto_min: float
    cost_per_epoch_usd: float
    cost_budget_usd: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    object_manifest_verified: bool
    dataset_checksum_verified: bool
    dataset_lineage_pinned: bool
    parallel_filesystem_healthy: bool
    metadata_slo_verified: bool
    data_loader_sharding_verified: bool
    cache_integrity_verified: bool
    gpu_data_wait_slo_verified: bool
    checkpoint_shards_complete: bool
    checkpoint_commit_verified: bool
    checkpoint_restore_tested: bool
    checkpoint_retention_safe: bool
    n_minus_one_failure_domains_verified: bool
    storage_network_isolation_verified: bool
    workload_identity_verified: bool
    kms_encryption_verified: bool
    tenant_isolation_verified: bool
    legal_hold_checked: bool
    capacity_forecast_reviewed: bool
    cost_attribution_verified: bool
    observability_pipeline_verified: bool
    incident_drills_completed: bool
    runbooks_approved: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.assessment, str) or not p.assessment.strip():
        return {"accepted": False, "reason": "missing_assessment"}
    if type(p.workers) is not int or p.workers <= 0:
        return {"accepted": False, "reason": "invalid_workers"}
    metrics = (p.worker_read_gbps, p.cache_miss_fraction, p.burst_factor,
               p.avg_read_kib, p.surviving_read_gbps, p.surviving_iops,
               p.checkpoint_gb, p.checkpoint_window_seconds, p.surviving_write_gbps,
               p.available_capacity_gb, p.required_capacity_gb,
               p.gpu_wait_p95_ms, p.gpu_wait_slo_ms, p.checkpoint_age_min,
               p.rpo_min, p.restore_minutes, p.rto_min,
               p.cost_per_epoch_usd, p.cost_budget_usd,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.worker_read_gbps, p.burst_factor, p.avg_read_kib,
                   p.checkpoint_gb, p.checkpoint_window_seconds,
                   p.required_capacity_gb, p.gpu_wait_slo_ms, p.rpo_min,
                   p.rto_min, p.cost_budget_usd, p.telemetry_max_age_s) <= 0
            or p.cache_miss_fraction > 1):
        return {"accepted": False, "reason": "invalid_metric"}
    read_gbps = p.workers * p.worker_read_gbps * p.cache_miss_fraction * p.burst_factor
    iops = read_gbps * 1e9 / (p.avg_read_kib * 1024)
    write_gbps = p.checkpoint_gb / p.checkpoint_window_seconds
    gates = {
        "degraded_read": p.surviving_read_gbps >= read_gbps,
        "degraded_iops": p.surviving_iops >= iops,
        "degraded_write": p.surviving_write_gbps >= write_gbps,
        "capacity": p.available_capacity_gb >= p.required_capacity_gb,
        "gpu_wait_slo": p.gpu_wait_p95_ms <= p.gpu_wait_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.rpo_min,
        "restore_rto": p.restore_minutes <= p.rto_min,
        "cost_budget": p.cost_per_epoch_usd <= p.cost_budget_usd,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "object_manifest_verified": p.object_manifest_verified is True,
        "dataset_checksum_verified": p.dataset_checksum_verified is True,
        "dataset_lineage_pinned": p.dataset_lineage_pinned is True,
        "parallel_filesystem_healthy": p.parallel_filesystem_healthy is True,
        "metadata_slo_verified": p.metadata_slo_verified is True,
        "data_loader_sharding_verified": p.data_loader_sharding_verified is True,
        "cache_integrity_verified": p.cache_integrity_verified is True,
        "gpu_data_wait_slo_verified": p.gpu_data_wait_slo_verified is True,
        "checkpoint_shards_complete": p.checkpoint_shards_complete is True,
        "checkpoint_commit_verified": p.checkpoint_commit_verified is True,
        "checkpoint_restore_tested": p.checkpoint_restore_tested is True,
        "checkpoint_retention_safe": p.checkpoint_retention_safe is True,
        "n_minus_one_failure_domains_verified": p.n_minus_one_failure_domains_verified is True,
        "storage_network_isolation_verified": p.storage_network_isolation_verified is True,
        "workload_identity_verified": p.workload_identity_verified is True,
        "kms_encryption_verified": p.kms_encryption_verified is True,
        "tenant_isolation_verified": p.tenant_isolation_verified is True,
        "legal_hold_checked": p.legal_hold_checked is True,
        "capacity_forecast_reviewed": p.capacity_forecast_reviewed is True,
        "cost_attribution_verified": p.cost_attribution_verified is True,
        "observability_pipeline_verified": p.observability_pipeline_verified is True,
        "incident_drills_completed": p.incident_drills_completed is True,
        "runbooks_approved": p.runbooks_approved is True,
        "change_approved": p.change_approved is True,
        "rollback_tested": p.rollback_tested is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "required_read_gbps": round(read_gbps, 4),
            "required_iops": round(iops, 2),
            "required_checkpoint_write_gbps": round(write_gbps, 4)}

def example():
    return Plan(
        "synthetic-storage-capstone", 32,
        0.25, 0.4, 1.25, 256.0, 5.0, 20000.0,
        2400.0, 300.0, 10.0, 10000.0, 8000.0,
        10.0, 30.0, 5.0, 30.0, 75.0, 120.0,
        600.0, 800.0, 10.0, 120.0,
        *([True] * 26)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_storage_capstone_policy.py`

```python
from dataclasses import replace
import pytest
from storage_capstone_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_assessment():
    assert evaluate(replace(example(), assessment=" "))["reason"] == "missing_assessment"

def test_bool_workers():
    assert evaluate(replace(example(), workers=True))["reason"] == "invalid_workers"

def test_fractional_workers():
    assert evaluate(replace(example(), workers=1.5))["reason"] == "invalid_workers"

def test_nan():
    assert evaluate(replace(example(), surviving_iops=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), restore_minutes=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), cost_budget_usd=True))["reason"] == "invalid_metric"

def test_bad_cache_fraction():
    assert evaluate(replace(example(), cache_miss_fraction=1.1))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 35

def test_nonboolean_review():
    assert "independent_review_approved" in evaluate(
        replace(example(), independent_review_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("surviving_read_gbps", "degraded_read", 1.0),
    ("surviving_iops", "degraded_iops", 1000.0),
    ("surviving_write_gbps", "degraded_write", 7.0),
    ("available_capacity_gb", "capacity", 7000.0),
    ("gpu_wait_p95_ms", "gpu_wait_slo", 31.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("restore_minutes", "restore_rto", 121.0),
    ("cost_per_epoch_usd", "cost_budget", 801.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("object_manifest_verified", "object_manifest_verified"),
    ("dataset_checksum_verified", "dataset_checksum_verified"),
    ("dataset_lineage_pinned", "dataset_lineage_pinned"),
    ("parallel_filesystem_healthy", "parallel_filesystem_healthy"),
    ("metadata_slo_verified", "metadata_slo_verified"),
    ("data_loader_sharding_verified", "data_loader_sharding_verified"),
    ("cache_integrity_verified", "cache_integrity_verified"),
    ("gpu_data_wait_slo_verified", "gpu_data_wait_slo_verified"),
    ("checkpoint_shards_complete", "checkpoint_shards_complete"),
    ("checkpoint_commit_verified", "checkpoint_commit_verified"),
    ("checkpoint_restore_tested", "checkpoint_restore_tested"),
    ("checkpoint_retention_safe", "checkpoint_retention_safe"),
    ("n_minus_one_failure_domains_verified", "n_minus_one_failure_domains_verified"),
    ("storage_network_isolation_verified", "storage_network_isolation_verified"),
    ("workload_identity_verified", "workload_identity_verified"),
    ("kms_encryption_verified", "kms_encryption_verified"),
    ("tenant_isolation_verified", "tenant_isolation_verified"),
    ("legal_hold_checked", "legal_hold_checked"),
    ("capacity_forecast_reviewed", "capacity_forecast_reviewed"),
    ("cost_attribution_verified", "cost_attribution_verified"),
    ("observability_pipeline_verified", "observability_pipeline_verified"),
    ("incident_drills_completed", "incident_drills_completed"),
    ("runbooks_approved", "runbooks_approved"),
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
python storage_capstone_policy.py
python -m pytest -q
```

**46 planned pytest cases:** 11 standalone + 9 numeric + 26 Boolean. Tests have not been executed. The gate cannot independently prove that the attested evidence is real.

## Ten synthetic failure drills

1. Object manifest/checksum mismatch.
2. Lustre/CSI or metadata SLO failure.
3. Rank/worker sharding inconsistency.
4. N-1 degraded read or IOPS deficit.
5. Checkpoint write bandwidth deficit.
6. Partial checkpoint or unsafe retention.
7. Restore RTO and checkpoint RPO breach.
8. KMS, tenant isolation or legal-hold violation.
9. Cost-per-epoch regression.
10. Stale telemetry or missing independent signoff.

## 100-point scorecard

Dataset/filesystem 20; throughput/IOPS/N-1 20; checkpoint/DR 20; security/incident operations 20; FinOps/tests/review 20. Require >=90, all critical gates and live staging evidence for operational acceptance.
