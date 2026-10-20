# Lab 057 — Storage IOPS, Throughput and Cost Gate

**Python 3.11+ · pytest · offline synthetic policy.** No cloud charges, destructive I/O, GPU provisioning or actual benchmarking.

## Goal

Fail closed on insufficient degraded-mode read IOPS/throughput, checkpoint write throughput, storage capacity, tail latency, GPU data-wait, RPO, stale telemetry, cost-per-epoch budget and critical governance evidence. Thresholds are example site budgets and must be replaced with approved workload-specific values.

## File: `storage_finops_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    workload: str
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
    metadata_p95_ms: float
    metadata_slo_ms: float
    gpu_wait_p95_ms: float
    gpu_wait_slo_ms: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    cost_per_epoch_usd: float
    cost_budget_usd: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    read_workload_profile_verified: bool
    random_io_profile_verified: bool
    metadata_profile_verified: bool
    object_request_budget_verified: bool
    cache_integrity_verified: bool
    dataset_manifest_pinned: bool
    checkpoint_durable_copy_verified: bool
    checkpoint_restore_tested: bool
    n_minus_one_failure_domain_verified: bool
    read_write_contention_tested: bool
    gpu_data_wait_slo_verified: bool
    storage_latency_slo_verified: bool
    tenant_isolation_verified: bool
    workload_identity_verified: bool
    encryption_verified: bool
    retention_owner_approved: bool
    legal_hold_checked: bool
    cost_attribution_verified: bool
    cross_zone_egress_reviewed: bool
    benchmark_reproducible: bool
    incident_runbook_tested: bool
    telemetry_source_verified: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.workload, str) or not p.workload.strip():
        return {"accepted": False, "reason": "missing_workload"}
    if type(p.workers) is not int or p.workers <= 0:
        return {"accepted": False, "reason": "invalid_workers"}
    metrics = (p.worker_read_gbps, p.cache_miss_fraction, p.burst_factor,
               p.avg_read_kib, p.surviving_read_gbps, p.surviving_iops,
               p.checkpoint_gb, p.checkpoint_window_seconds, p.surviving_write_gbps,
               p.available_capacity_gb, p.required_capacity_gb,
               p.metadata_p95_ms, p.metadata_slo_ms, p.gpu_wait_p95_ms,
               p.gpu_wait_slo_ms, p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.cost_per_epoch_usd, p.cost_budget_usd,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.worker_read_gbps, p.burst_factor, p.avg_read_kib,
                   p.checkpoint_gb, p.checkpoint_window_seconds,
                   p.required_capacity_gb, p.metadata_slo_ms, p.gpu_wait_slo_ms,
                   p.checkpoint_rpo_min, p.cost_budget_usd, p.telemetry_max_age_s) <= 0
            or p.cache_miss_fraction > 1):
        return {"accepted": False, "reason": "invalid_metric"}
    read_gbps = p.workers * p.worker_read_gbps * p.cache_miss_fraction * p.burst_factor
    required_iops = read_gbps * 1e9 / (p.avg_read_kib * 1024)
    write_gbps = p.checkpoint_gb / p.checkpoint_window_seconds
    gates = {
        "degraded_read": p.surviving_read_gbps >= read_gbps,
        "degraded_iops": p.surviving_iops >= required_iops,
        "degraded_write": p.surviving_write_gbps >= write_gbps,
        "capacity": p.available_capacity_gb >= p.required_capacity_gb,
        "metadata_slo": p.metadata_p95_ms <= p.metadata_slo_ms,
        "gpu_wait_slo": p.gpu_wait_p95_ms <= p.gpu_wait_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "cost_budget": p.cost_per_epoch_usd <= p.cost_budget_usd,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "read_workload_profile_verified": p.read_workload_profile_verified is True,
        "random_io_profile_verified": p.random_io_profile_verified is True,
        "metadata_profile_verified": p.metadata_profile_verified is True,
        "object_request_budget_verified": p.object_request_budget_verified is True,
        "cache_integrity_verified": p.cache_integrity_verified is True,
        "dataset_manifest_pinned": p.dataset_manifest_pinned is True,
        "checkpoint_durable_copy_verified": p.checkpoint_durable_copy_verified is True,
        "checkpoint_restore_tested": p.checkpoint_restore_tested is True,
        "n_minus_one_failure_domain_verified": p.n_minus_one_failure_domain_verified is True,
        "read_write_contention_tested": p.read_write_contention_tested is True,
        "gpu_data_wait_slo_verified": p.gpu_data_wait_slo_verified is True,
        "storage_latency_slo_verified": p.storage_latency_slo_verified is True,
        "tenant_isolation_verified": p.tenant_isolation_verified is True,
        "workload_identity_verified": p.workload_identity_verified is True,
        "encryption_verified": p.encryption_verified is True,
        "retention_owner_approved": p.retention_owner_approved is True,
        "legal_hold_checked": p.legal_hold_checked is True,
        "cost_attribution_verified": p.cost_attribution_verified is True,
        "cross_zone_egress_reviewed": p.cross_zone_egress_reviewed is True,
        "benchmark_reproducible": p.benchmark_reproducible is True,
        "incident_runbook_tested": p.incident_runbook_tested is True,
        "telemetry_source_verified": p.telemetry_source_verified is True,
        "change_approved": p.change_approved is True,
        "rollback_tested": p.rollback_tested is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(name for name, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "required_read_gbps": round(read_gbps, 4),
            "required_iops": round(required_iops, 2),
            "required_checkpoint_write_gbps": round(write_gbps, 4)}

def example():
    return Plan(
        "synthetic-ai-training", 32,
        0.25, 0.4, 1.25, 256.0,
        5.0, 20000.0,
        2400.0, 300.0, 10.0,
        10000.0, 8000.0,
        2.0, 5.0, 10.0, 30.0,
        5.0, 30.0, 600.0, 800.0,
        10.0, 120.0,
        *([True] * 25)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_storage_finops_policy.py`

```python
from dataclasses import replace
import pytest
from storage_finops_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_workload():
    assert evaluate(replace(example(), workload=" "))["reason"] == "missing_workload"

def test_bool_workers():
    assert evaluate(replace(example(), workers=True))["reason"] == "invalid_workers"

def test_fractional_workers():
    assert evaluate(replace(example(), workers=1.5))["reason"] == "invalid_workers"

def test_nan():
    assert evaluate(replace(example(), surviving_iops=float("nan")))["reason"] == "invalid_metric"

def test_infinite():
    assert evaluate(replace(example(), checkpoint_age_min=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), cost_budget_usd=True))["reason"] == "invalid_metric"

def test_invalid_cache_fraction():
    assert evaluate(replace(example(), cache_miss_fraction=1.1))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 34

def test_nonboolean_approval():
    assert "change_approved" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("surviving_read_gbps", "degraded_read", 1.0),
    ("surviving_iops", "degraded_iops", 1000.0),
    ("surviving_write_gbps", "degraded_write", 7.0),
    ("available_capacity_gb", "capacity", 7000.0),
    ("metadata_p95_ms", "metadata_slo", 6.0),
    ("gpu_wait_p95_ms", "gpu_wait_slo", 31.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("cost_per_epoch_usd", "cost_budget", 801.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("read_workload_profile_verified", "read_workload_profile_verified"),
    ("random_io_profile_verified", "random_io_profile_verified"),
    ("metadata_profile_verified", "metadata_profile_verified"),
    ("object_request_budget_verified", "object_request_budget_verified"),
    ("cache_integrity_verified", "cache_integrity_verified"),
    ("dataset_manifest_pinned", "dataset_manifest_pinned"),
    ("checkpoint_durable_copy_verified", "checkpoint_durable_copy_verified"),
    ("checkpoint_restore_tested", "checkpoint_restore_tested"),
    ("n_minus_one_failure_domain_verified", "n_minus_one_failure_domain_verified"),
    ("read_write_contention_tested", "read_write_contention_tested"),
    ("gpu_data_wait_slo_verified", "gpu_data_wait_slo_verified"),
    ("storage_latency_slo_verified", "storage_latency_slo_verified"),
    ("tenant_isolation_verified", "tenant_isolation_verified"),
    ("workload_identity_verified", "workload_identity_verified"),
    ("encryption_verified", "encryption_verified"),
    ("retention_owner_approved", "retention_owner_approved"),
    ("legal_hold_checked", "legal_hold_checked"),
    ("cost_attribution_verified", "cost_attribution_verified"),
    ("cross_zone_egress_reviewed", "cross_zone_egress_reviewed"),
    ("benchmark_reproducible", "benchmark_reproducible"),
    ("incident_runbook_tested", "incident_runbook_tested"),
    ("telemetry_source_verified", "telemetry_source_verified"),
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
python storage_finops_policy.py
python -m pytest -q
```

**45 planned pytest cases:** 11 standalone + 9 numeric + 25 Boolean. Actual execution, live benchmark and billing validation remain pending.

## Ten synthetic incident drills

1. Degraded read throughput shortfall.
2. Degraded IOPS shortfall.
3. Checkpoint write bandwidth shortage.
4. Capacity exhaustion.
5. Metadata latency SLO breach.
6. GPU data-wait SLO breach.
7. Checkpoint RPO breach.
8. Cost per epoch exceeds budget.
9. Stale telemetry.
10. Missing identity, retention, rollback or independent review.

## 100-point acceptance

Performance 20; capacity/checkpoints 20; cost 20; governance/incidents 20; tests and independent evidence 20. Pass >=90 with all critical gates and real staging measurements before production.
