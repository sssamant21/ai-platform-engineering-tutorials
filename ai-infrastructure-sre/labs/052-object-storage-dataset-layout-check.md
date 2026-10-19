# Lab 052 — Object Storage and Dataset Layout Gate

**Python 3.11+ · pytest · offline synthetic policy.** No S3/Blob calls, real uploads or cloud spend.

## Goal

Fail closed on object manifest integrity, shard size/count, concurrency, retry budget, request rate, stale telemetry, IAM/KMS, conditional publication, retention, checkpoint, SLO and rollback. Synthetic attestations must be replaced with signed real evidence before production acceptance.

## File: `object_dataset_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    dataset: str
    provider: str
    dataset_size_gib: float
    shard_size_mib: float
    max_shards: int
    readers: int
    max_reader_concurrency: int
    expected_get_requests: int
    request_budget: int
    retry_rate_pct: float
    max_retry_rate_pct: float
    manifest_age_min: float
    max_manifest_age_min: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    immutable_keys_verified: bool
    object_checksums_verified: bool
    manifest_complete_verified: bool
    atomic_pointer_publication_verified: bool
    concurrent_writer_conflicts_handled: bool
    version_recovery_verified: bool
    schema_validated: bool
    lineage_recorded: bool
    workload_identity_verified: bool
    kms_encryption_verified: bool
    tenant_isolation_verified: bool
    bounded_backoff_verified: bool
    cache_invalidation_verified: bool
    lifecycle_owner_approved: bool
    checkpoint_preservation_verified: bool
    read_slo_verified: bool
    audit_logging_verified: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.dataset, str) or not p.dataset.strip():
        return {"accepted": False, "reason": "missing_dataset"}
    if p.provider not in ("s3", "azure-blob", "compatible-object"):
        return {"accepted": False, "reason": "invalid_provider"}
    counts = (p.max_shards, p.readers, p.max_reader_concurrency,
              p.expected_get_requests, p.request_budget)
    if any(type(v) is not int or v <= 0 for v in counts):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.dataset_size_gib, p.shard_size_mib, p.retry_rate_pct,
               p.max_retry_rate_pct, p.manifest_age_min, p.max_manifest_age_min,
               p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.dataset_size_gib, p.shard_size_mib,
                   p.max_manifest_age_min, p.max_telemetry_age_s) <= 0
            or p.retry_rate_pct > 100 or p.max_retry_rate_pct > 100):
        return {"accepted": False, "reason": "invalid_metric"}
    shard_count = math.ceil(p.dataset_size_gib * 1024 / p.shard_size_mib)
    gates = {
        "shard_count": shard_count <= p.max_shards,
        "reader_concurrency": p.readers <= p.max_reader_concurrency,
        "request_budget": p.expected_get_requests <= p.request_budget,
        "retry_rate": p.retry_rate_pct <= p.max_retry_rate_pct,
        "manifest_freshness": p.manifest_age_min <= p.max_manifest_age_min,
        "telemetry_freshness": p.telemetry_age_s <= p.max_telemetry_age_s,
        "immutable_keys": p.immutable_keys_verified is True,
        "checksums": p.object_checksums_verified is True,
        "manifest": p.manifest_complete_verified is True,
        "atomic_publish": p.atomic_pointer_publication_verified is True,
        "writer_conflicts": p.concurrent_writer_conflicts_handled is True,
        "version_recovery": p.version_recovery_verified is True,
        "schema": p.schema_validated is True,
        "lineage": p.lineage_recorded is True,
        "identity": p.workload_identity_verified is True,
        "kms": p.kms_encryption_verified is True,
        "tenant_isolation": p.tenant_isolation_verified is True,
        "backoff": p.bounded_backoff_verified is True,
        "cache": p.cache_invalidation_verified is True,
        "lifecycle": p.lifecycle_owner_approved is True,
        "checkpoint": p.checkpoint_preservation_verified is True,
        "read_slo": p.read_slo_verified is True,
        "audit": p.audit_logging_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
        "independent_review": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates), "computed_shards": shard_count}

def example():
    return Plan(
        "synthetic-vision-dataset", "s3",
        1024.0, 256.0, 5000, 64, 128, 4096, 6000,
        0.5, 1.0, 5.0, 60.0, 10.0, 120.0,
        *([True] * 20)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_object_dataset_policy.py`

```python
from dataclasses import replace
import pytest
from object_dataset_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_dataset():
    assert evaluate(replace(example(), dataset=" "))["reason"] == "missing_dataset"

def test_invalid_provider():
    assert evaluate(replace(example(), provider="nvme"))["reason"] == "invalid_provider"

def test_bool_count():
    assert evaluate(replace(example(), readers=True))["reason"] == "invalid_count"

def test_fractional_count():
    assert evaluate(replace(example(), max_shards=1.5))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), shard_size_mib=float("nan")))["reason"] == "invalid_metric"

def test_infinity():
    assert evaluate(replace(example(), telemetry_age_s=float("inf")))["reason"] == "invalid_metric"

def test_invalid_retry_pct():
    assert evaluate(replace(example(), retry_rate_pct=101.0))["reason"] == "invalid_metric"

def test_nonboolean():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 26

@pytest.mark.parametrize("field,gate,value", [
    ("shard_size_mib", "shard_count", 128.0),
    ("readers", "reader_concurrency", 129),
    ("expected_get_requests", "request_budget", 6001),
    ("retry_rate_pct", "retry_rate", 1.1),
    ("manifest_age_min", "manifest_freshness", 61.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("immutable_keys_verified", "immutable_keys"),
    ("object_checksums_verified", "checksums"),
    ("manifest_complete_verified", "manifest"),
    ("atomic_pointer_publication_verified", "atomic_publish"),
    ("concurrent_writer_conflicts_handled", "writer_conflicts"),
    ("version_recovery_verified", "version_recovery"),
    ("schema_validated", "schema"),
    ("lineage_recorded", "lineage"),
    ("workload_identity_verified", "identity"),
    ("kms_encryption_verified", "kms"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("bounded_backoff_verified", "backoff"),
    ("cache_invalidation_verified", "cache"),
    ("lifecycle_owner_approved", "lifecycle"),
    ("checkpoint_preservation_verified", "checkpoint"),
    ("read_slo_verified", "read_slo"),
    ("audit_logging_verified", "audit"),
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
python object_dataset_policy.py
python -m pytest -q
```

**Planned 37 pytest cases:** 11 standalone + 6 numeric + 20 boolean. Actual execution, cloud integration and independent review remain pending.

## Ten synthetic incident drills

1. Excessive shard count.
2. Reader concurrency exceeds safe cap.
3. Request budget exceeded.
4. Retry amplification.
5. Stale release manifest.
6. Missing object checksum.
7. Unsafe concurrent release pointer update.
8. IAM/KMS or tenant isolation failure.
9. Retention destroys last recoverable checkpoint.
10. Missing change approval, rollback or independent review.

## 100-point rubric

Layout and integrity 20; performance/request cost 20; identity and governance 20; incident/recovery 20; tests/staging evidence 20. Require >=90, all critical gates and independent reviewer signoff.
