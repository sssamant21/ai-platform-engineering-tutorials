# Lab 051 — AI Data Storage Architecture Gate

**Python 3.11+ · pytest · offline synthetic policy.** No object-store requests, CSI operations, GPU provisioning or cloud spend.

## Objective

Fail closed on insufficient read/write throughput, capacity, cache, checkpoint RPO, telemetry, dataset integrity, identity, isolation, lifecycle governance and rollback. Attestations are synthetic; real staging validation remains mandatory.

## File: `ai_storage_architecture_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    durable_tier: str
    workers: int
    per_worker_read_gbps: float
    burst_factor: float
    provisioned_read_gbps: float
    provisioned_write_gbps: float
    required_checkpoint_write_gbps: float
    dataset_size_gb: float
    available_capacity_gb: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    immutable_manifest_verified: bool
    dataset_checksum_verified: bool
    data_lineage_verified: bool
    cache_not_sole_durable_copy: bool
    storage_csi_verified: bool
    access_modes_verified: bool
    zone_failure_capacity_verified: bool
    object_retry_backoff_verified: bool
    metadata_hotspot_reviewed: bool
    workload_identity_verified: bool
    encryption_verified: bool
    tenant_isolation_verified: bool
    retention_owner_approved: bool
    checkpoint_restore_tested: bool
    storage_slo_verified: bool
    cost_budget_approved: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    if p.durable_tier not in ("s3", "azure-blob", "compatible-object"):
        return {"accepted": False, "reason": "invalid_durable_tier"}
    if type(p.workers) is not int or p.workers <= 0:
        return {"accepted": False, "reason": "invalid_workers"}
    metrics = (p.per_worker_read_gbps, p.burst_factor, p.provisioned_read_gbps,
               p.provisioned_write_gbps, p.required_checkpoint_write_gbps,
               p.dataset_size_gb, p.available_capacity_gb, p.checkpoint_age_min,
               p.checkpoint_rpo_min, p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or min(p.per_worker_read_gbps, p.burst_factor, p.provisioned_read_gbps,
                   p.provisioned_write_gbps, p.required_checkpoint_write_gbps,
                   p.dataset_size_gb, p.available_capacity_gb,
                   p.checkpoint_rpo_min, p.telemetry_max_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    demand = p.workers * p.per_worker_read_gbps * p.burst_factor
    gates = {
        "read_capacity": p.provisioned_read_gbps >= demand,
        "write_capacity": p.provisioned_write_gbps >= p.required_checkpoint_write_gbps,
        "dataset_capacity": p.available_capacity_gb >= p.dataset_size_gb,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "manifest": p.immutable_manifest_verified is True,
        "checksum": p.dataset_checksum_verified is True,
        "lineage": p.data_lineage_verified is True,
        "durable_copy": p.cache_not_sole_durable_copy is True,
        "csi": p.storage_csi_verified is True,
        "access_modes": p.access_modes_verified is True,
        "zone_resilience": p.zone_failure_capacity_verified is True,
        "retry_backoff": p.object_retry_backoff_verified is True,
        "metadata": p.metadata_hotspot_reviewed is True,
        "identity": p.workload_identity_verified is True,
        "encryption": p.encryption_verified is True,
        "tenant_isolation": p.tenant_isolation_verified is True,
        "retention_approval": p.retention_owner_approved is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "storage_slo": p.storage_slo_verified is True,
        "cost_budget": p.cost_budget_approved is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
        "independent_review": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates), "required_read_gbps": round(demand, 4)}

def example():
    return Plan(
        "synthetic-ai-storage", "s3", 16,
        0.25, 1.25, 6.0, 3.0, 2.0,
        2000.0, 4000.0, 5.0, 30.0, 10.0, 120.0,
        *([True] * 19)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_ai_storage_architecture_policy.py`

```python
from dataclasses import replace
import pytest
from ai_storage_architecture_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_empty_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_bad_tier():
    assert evaluate(replace(example(), durable_tier="nvme"))["reason"] == "invalid_durable_tier"

def test_bool_workers():
    assert evaluate(replace(example(), workers=True))["reason"] == "invalid_workers"

def test_fractional_workers():
    assert evaluate(replace(example(), workers=1.5))["reason"] == "invalid_workers"

def test_nan():
    assert evaluate(replace(example(), provisioned_read_gbps=float("nan")))["reason"] == "invalid_metric"

def test_infinite():
    assert evaluate(replace(example(), checkpoint_age_min=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), per_worker_read_gbps=True))["reason"] == "invalid_metric"

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 24

@pytest.mark.parametrize("field,gate,value", [
    ("workers", "read_capacity", 32),
    ("provisioned_write_gbps", "write_capacity", 1.0),
    ("available_capacity_gb", "dataset_capacity", 1000.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("immutable_manifest_verified", "manifest"),
    ("dataset_checksum_verified", "checksum"),
    ("data_lineage_verified", "lineage"),
    ("cache_not_sole_durable_copy", "durable_copy"),
    ("storage_csi_verified", "csi"),
    ("access_modes_verified", "access_modes"),
    ("zone_failure_capacity_verified", "zone_resilience"),
    ("object_retry_backoff_verified", "retry_backoff"),
    ("metadata_hotspot_reviewed", "metadata"),
    ("workload_identity_verified", "identity"),
    ("encryption_verified", "encryption"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("retention_owner_approved", "retention_approval"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("storage_slo_verified", "storage_slo"),
    ("cost_budget_approved", "cost_budget"),
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
python ai_storage_architecture_policy.py
python -m pytest -q
```

**Planned 35 pytest cases:** 11 standalone + 5 numeric + 19 boolean. Actual execution and live storage validation remain pending.

## Ten synthetic failure drills

1. Read demand exceeds provisioned throughput.
2. Checkpoint write throughput below target.
3. Dataset exceeds usable capacity.
4. Checkpoint RPO violation.
5. Stale telemetry.
6. Manifest/checksum or lineage failure.
7. Cache is the only durable copy.
8. CSI/access mode or AZ-resilience issue.
9. Tenant identity, encryption or retention policy failure.
10. Missing change, rollback or independent review.

## 100-point evidence rubric

Architecture and data correctness 20; performance and capacity 20; Kubernetes/security 20; reliability/DR 20; testing and approvals 20. Require >=90, all critical gates, verified restore and independent review.
