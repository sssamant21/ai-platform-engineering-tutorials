# Lab 054 — High Throughput Dataset Loading Gate

**Python 3.11+ · pytest · offline synthetic policy.** No GPU, cloud, storage or Kubernetes access required.

## Goal

Fail closed on throughput, cache misses, decode CPU, prefetch memory, GPU batch wait, checkpoint RPO, telemetry and distributed correctness. Inputs are synthetic; Boolean attestations must be replaced by reviewed evidence in a real environment.

## File: `dataset_loading_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    job: str
    ranks: int
    workers_per_rank: int
    samples_per_second_per_rank: float
    bytes_per_sample_mb: float
    cache_miss_fraction: float
    burst_factor: float
    available_read_gbps: float
    cpu_seconds_per_sample: float
    available_decode_cores: float
    prefetch_batches_per_worker: int
    batch_memory_mb: float
    available_prefetch_memory_mb: float
    batch_wait_p95_ms: float
    batch_wait_slo_ms: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    immutable_manifest_verified: bool
    checksum_verified: bool
    rank_sharding_verified: bool
    worker_sharding_verified: bool
    sampler_epoch_verified: bool
    reproducible_restart_verified: bool
    object_retry_backoff_verified: bool
    cache_integrity_verified: bool
    cache_not_sole_copy: bool
    filesystem_metadata_reviewed: bool
    gpu_cpu_numa_reviewed: bool
    kubernetes_resource_limits_verified: bool
    workload_identity_verified: bool
    tenant_isolation_verified: bool
    checkpoint_restore_tested: bool
    workload_slo_verified: bool
    incident_runbook_tested: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.job, str) or not p.job.strip():
        return {"accepted": False, "reason": "missing_job"}
    ints = (p.ranks, p.workers_per_rank, p.prefetch_batches_per_worker)
    if any(type(v) is not int or v <= 0 for v in ints):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.samples_per_second_per_rank, p.bytes_per_sample_mb,
               p.cache_miss_fraction, p.burst_factor, p.available_read_gbps,
               p.cpu_seconds_per_sample, p.available_decode_cores,
               p.batch_memory_mb, p.available_prefetch_memory_mb,
               p.batch_wait_p95_ms, p.batch_wait_slo_ms,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.samples_per_second_per_rank, p.bytes_per_sample_mb,
                   p.burst_factor, p.available_read_gbps, p.available_decode_cores,
                   p.batch_memory_mb, p.available_prefetch_memory_mb,
                   p.batch_wait_slo_ms, p.checkpoint_rpo_min,
                   p.telemetry_max_age_s) <= 0
            or p.cache_miss_fraction > 1):
        return {"accepted": False, "reason": "invalid_metric"}
    samples = p.ranks * p.samples_per_second_per_rank
    required_read = samples * p.bytes_per_sample_mb / 1000 * p.cache_miss_fraction * p.burst_factor
    required_cores = samples * p.cpu_seconds_per_sample
    required_memory = p.ranks * p.workers_per_rank * p.prefetch_batches_per_worker * p.batch_memory_mb
    gates = {
        "read_capacity": p.available_read_gbps >= required_read,
        "decode_cpu": p.available_decode_cores >= required_cores,
        "prefetch_memory": p.available_prefetch_memory_mb >= required_memory,
        "batch_wait_slo": p.batch_wait_p95_ms <= p.batch_wait_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "manifest": p.immutable_manifest_verified is True,
        "checksum": p.checksum_verified is True,
        "rank_sharding": p.rank_sharding_verified is True,
        "worker_sharding": p.worker_sharding_verified is True,
        "sampler_epoch": p.sampler_epoch_verified is True,
        "reproducible_restart": p.reproducible_restart_verified is True,
        "backoff": p.object_retry_backoff_verified is True,
        "cache_integrity": p.cache_integrity_verified is True,
        "durable_copy": p.cache_not_sole_copy is True,
        "metadata": p.filesystem_metadata_reviewed is True,
        "numa": p.gpu_cpu_numa_reviewed is True,
        "kubernetes": p.kubernetes_resource_limits_verified is True,
        "identity": p.workload_identity_verified is True,
        "tenant_isolation": p.tenant_isolation_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "workload_slo": p.workload_slo_verified is True,
        "runbook": p.incident_runbook_tested is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
        "independent_review": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "required_read_gbps": round(required_read, 4),
            "required_decode_cores": round(required_cores, 4),
            "required_prefetch_memory_mb": round(required_memory, 4)}

def example():
    return Plan(
        "synthetic-training-job", 16, 4, 500.0, 0.5, 0.25, 1.3,
        2.0, 0.002, 24.0, 2, 8.0, 1200.0,
        10.0, 30.0, 5.0, 30.0, 10.0, 120.0,
        *([True] * 20)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_dataset_loading_policy.py`

```python
from dataclasses import replace
import pytest
from dataset_loading_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_job():
    assert evaluate(replace(example(), job=" "))["reason"] == "missing_job"

def test_bool_rank():
    assert evaluate(replace(example(), ranks=True))["reason"] == "invalid_count"

def test_fractional_workers():
    assert evaluate(replace(example(), workers_per_rank=1.5))["reason"] == "invalid_count"

def test_zero_prefetch():
    assert evaluate(replace(example(), prefetch_batches_per_worker=0))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), batch_wait_p95_ms=float("nan")))["reason"] == "invalid_metric"

def test_infinity():
    assert evaluate(replace(example(), available_read_gbps=float("inf")))["reason"] == "invalid_metric"

def test_invalid_cache_miss():
    assert evaluate(replace(example(), cache_miss_fraction=1.1))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), burst_factor=True))["reason"] == "invalid_metric"

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 26

@pytest.mark.parametrize("field,gate,value", [
    ("available_read_gbps", "read_capacity", 0.1),
    ("available_decode_cores", "decode_cpu", 10.0),
    ("available_prefetch_memory_mb", "prefetch_memory", 100.0),
    ("batch_wait_p95_ms", "batch_wait_slo", 31.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("immutable_manifest_verified", "manifest"),
    ("checksum_verified", "checksum"),
    ("rank_sharding_verified", "rank_sharding"),
    ("worker_sharding_verified", "worker_sharding"),
    ("sampler_epoch_verified", "sampler_epoch"),
    ("reproducible_restart_verified", "reproducible_restart"),
    ("object_retry_backoff_verified", "backoff"),
    ("cache_integrity_verified", "cache_integrity"),
    ("cache_not_sole_copy", "durable_copy"),
    ("filesystem_metadata_reviewed", "metadata"),
    ("gpu_cpu_numa_reviewed", "numa"),
    ("kubernetes_resource_limits_verified", "kubernetes"),
    ("workload_identity_verified", "identity"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("workload_slo_verified", "workload_slo"),
    ("incident_runbook_tested", "runbook"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("independent_review_approved", "independent_review"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run offline:

```bash
python -m pip install pytest
python dataset_loading_policy.py
python -m pytest -q
```

**Planned 38 pytest cases:** 12 standalone + 6 numeric + 20 boolean. Actual execution and production-equivalent GPU/storage benchmarks remain pending.

## Ten synthetic incident drills

1. Source-read bandwidth shortfall.
2. CPU decode saturation.
3. Prefetch memory exhaustion.
4. Batch wait SLO breach.
5. Checkpoint RPO breach.
6. Stale telemetry.
7. Dataset checksum or manifest mismatch.
8. Duplicate distributed samples.
9. Cache poisoning or tenant isolation failure.
10. Missing rollback, change approval or independent review.

## 100-point evidence rubric

Dataset/sharding correctness 20; GPU/CPU/storage performance 20; Kubernetes/security 20; incidents/DR 20; tests/staging/approvals 20. Require >=90, all critical gates and reviewed real evidence for production.
