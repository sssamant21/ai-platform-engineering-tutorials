# Lab 053 — FSx for Lustre and Parallel Filesystems Gate

**Python 3.11+ · pytest · offline synthetic policy.** No filesystem mutations, cloud provisioning or GPU benchmarks.

## Goal

Fail closed on read/write throughput, N-1 headroom, storage/metadata capacity, checkpoint RPO, telemetry, CSI, Lustre client compatibility, data repository integrity, tenant security and rollback. Attestations are synthetic, not live operational proof.

## File: `lustre_reliability_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    filesystem: str
    workers: int
    per_worker_read_gbps: float
    burst_factor: float
    surviving_read_gbps: float
    checkpoint_size_gb: float
    checkpoint_target_seconds: float
    surviving_write_gbps: float
    available_capacity_gb: float
    required_capacity_gb: float
    metadata_p95_ms: float
    metadata_slo_ms: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    lustre_client_compatible: bool
    mdt_health_verified: bool
    ost_health_verified: bool
    stripe_layout_reviewed: bool
    network_path_verified: bool
    kubernetes_csi_verified: bool
    access_modes_verified: bool
    dataset_manifest_verified: bool
    repository_import_verified: bool
    repository_export_verified: bool
    checkpoint_restore_tested: bool
    backup_retention_verified: bool
    tenant_isolation_verified: bool
    encryption_verified: bool
    workload_identity_verified: bool
    storage_slo_verified: bool
    incident_runbook_tested: bool
    cost_budget_approved: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    if p.filesystem not in ("fsx-lustre", "lustre", "beegfs", "azure-managed-lustre"):
        return {"accepted": False, "reason": "invalid_filesystem"}
    if type(p.workers) is not int or p.workers <= 0:
        return {"accepted": False, "reason": "invalid_workers"}
    metrics = (p.per_worker_read_gbps, p.burst_factor, p.surviving_read_gbps,
               p.checkpoint_size_gb, p.checkpoint_target_seconds,
               p.surviving_write_gbps, p.available_capacity_gb,
               p.required_capacity_gb, p.metadata_p95_ms, p.metadata_slo_ms,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.per_worker_read_gbps, p.burst_factor,
                   p.checkpoint_size_gb, p.checkpoint_target_seconds,
                   p.required_capacity_gb, p.metadata_slo_ms,
                   p.checkpoint_rpo_min, p.telemetry_max_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    read_demand = p.workers * p.per_worker_read_gbps * p.burst_factor
    checkpoint_write_demand = p.checkpoint_size_gb / p.checkpoint_target_seconds
    gates = {
        "n_minus_one_read": p.surviving_read_gbps >= read_demand,
        "n_minus_one_write": p.surviving_write_gbps >= checkpoint_write_demand,
        "capacity": p.available_capacity_gb >= p.required_capacity_gb,
        "metadata_slo": p.metadata_p95_ms <= p.metadata_slo_ms,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "client": p.lustre_client_compatible is True,
        "mdt": p.mdt_health_verified is True,
        "ost": p.ost_health_verified is True,
        "stripe": p.stripe_layout_reviewed is True,
        "network": p.network_path_verified is True,
        "csi": p.kubernetes_csi_verified is True,
        "access_modes": p.access_modes_verified is True,
        "manifest": p.dataset_manifest_verified is True,
        "import": p.repository_import_verified is True,
        "export": p.repository_export_verified is True,
        "restore": p.checkpoint_restore_tested is True,
        "retention": p.backup_retention_verified is True,
        "tenant": p.tenant_isolation_verified is True,
        "encryption": p.encryption_verified is True,
        "identity": p.workload_identity_verified is True,
        "slo": p.storage_slo_verified is True,
        "runbook": p.incident_runbook_tested is True,
        "cost": p.cost_budget_approved is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
        "review": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "required_read_gbps": round(read_demand, 4),
            "required_checkpoint_write_gbps": round(checkpoint_write_demand, 4)}

def example():
    return Plan(
        "synthetic-lustre-cluster", "fsx-lustre", 32,
        0.2, 1.25, 10.0, 2400.0, 300.0, 10.0,
        10000.0, 8000.0, 2.0, 5.0,
        5.0, 30.0, 10.0, 120.0,
        *([True] * 21)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_lustre_reliability_policy.py`

```python
from dataclasses import replace
import pytest
from lustre_reliability_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_bad_filesystem():
    assert evaluate(replace(example(), filesystem="nfs"))["reason"] == "invalid_filesystem"

def test_bool_workers():
    assert evaluate(replace(example(), workers=True))["reason"] == "invalid_workers"

def test_fractional_workers():
    assert evaluate(replace(example(), workers=1.5))["reason"] == "invalid_workers"

def test_nan():
    assert evaluate(replace(example(), surviving_read_gbps=float("nan")))["reason"] == "invalid_metric"

def test_infinity():
    assert evaluate(replace(example(), checkpoint_age_min=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), per_worker_read_gbps=True))["reason"] == "invalid_metric"

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

def test_gate_count():
    assert evaluate(example())["total_gates"] == 27

@pytest.mark.parametrize("field,gate,value", [
    ("workers", "n_minus_one_read", 64),
    ("surviving_write_gbps", "n_minus_one_write", 7.0),
    ("available_capacity_gb", "capacity", 7000.0),
    ("metadata_p95_ms", "metadata_slo", 6.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("lustre_client_compatible", "client"),
    ("mdt_health_verified", "mdt"),
    ("ost_health_verified", "ost"),
    ("stripe_layout_reviewed", "stripe"),
    ("network_path_verified", "network"),
    ("kubernetes_csi_verified", "csi"),
    ("access_modes_verified", "access_modes"),
    ("dataset_manifest_verified", "manifest"),
    ("repository_import_verified", "import"),
    ("repository_export_verified", "export"),
    ("checkpoint_restore_tested", "restore"),
    ("backup_retention_verified", "retention"),
    ("tenant_isolation_verified", "tenant"),
    ("encryption_verified", "encryption"),
    ("workload_identity_verified", "identity"),
    ("storage_slo_verified", "slo"),
    ("incident_runbook_tested", "runbook"),
    ("cost_budget_approved", "cost"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
    ("independent_review_approved", "review"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run offline:

```bash
python -m pip install pytest
python lustre_reliability_policy.py
python -m pytest -q
```

**Planned 38 pytest cases:** 11 standalone + 6 numeric + 21 boolean. Actual execution, cloud benchmarking and independent review remain pending.

## Ten synthetic incident drills

1. N-1 read throughput shortfall.
2. N-1 checkpoint write shortfall.
3. Capacity exhaustion.
4. Metadata SLO violation.
5. Checkpoint RPO breach.
6. Telemetry staleness.
7. Client/MDT/OST compatibility failure.
8. S3 import/export or dataset manifest mismatch.
9. Tenant identity, encryption or retention failure.
10. Missing change, rollback or independent signoff.

## 100-point rubric

Lustre architecture 20; performance/capacity 20; Kubernetes/security 20; recovery/SLO 20; testing/approvals 20. Require >=90, all critical gates and live staging evidence before production acceptance.
