# Chapter 008 — Executable Foundations Architecture Review Acceptance Lab

**Status:** Published. Offline pytest execution, live GPU/cluster checks, independent review and production approval pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Deterministic architecture review evaluator
Save as `architecture_review.py`:
```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Review:
    zone_safe_rps: dict[str, float]
    peak_rps: float
    reserve_fraction: float
    slo_target: float
    eligible_requests: int
    bad_requests: int
    service_owner: str
    backup_owner: str
    workload_contract_defined: bool
    gpu_compatibility_verified: bool
    dependency_graph_reviewed: bool
    network_storage_tested: bool
    workload_identity_verified: bool
    tenant_isolation_verified: bool
    artifact_signatures_verified: bool
    checkpoint_restore_verified: bool
    recovery_targets_defined: bool
    telemetry_verified: bool
    paging_verified: bool
    rollback_approved: bool
    budget_approved: bool

def evaluate(r):
    if not isinstance(r, Review):
        raise TypeError("Review required")
    if not isinstance(r.zone_safe_rps, dict) or len(r.zone_safe_rps) < 3:
        return {"accepted": False, "reason": "insufficient_zones"}
    if any(type(v) not in (int, float) or not math.isfinite(v) or v <= 0
           for v in r.zone_safe_rps.values()):
        return {"accepted": False, "reason": "invalid_zone_capacity"}
    if any(type(v) not in (int, float) or not math.isfinite(v)
           for v in (r.peak_rps, r.reserve_fraction, r.slo_target)):
        return {"accepted": False, "reason": "invalid_numeric_input"}
    if not (r.peak_rps >= 0 and 0 <= r.reserve_fraction < 1
            and 0 < r.slo_target < 1):
        return {"accepted": False, "reason": "invalid_numeric_input"}
    if (type(r.eligible_requests) is not int or type(r.bad_requests) is not int
            or r.eligible_requests <= 0 or not 0 <= r.bad_requests <= r.eligible_requests):
        return {"accepted": False, "reason": "invalid_request_counts"}
    if (not isinstance(r.service_owner, str) or not r.service_owner.strip()
            or not isinstance(r.backup_owner, str) or not r.backup_owner.strip()
            or r.service_owner == r.backup_owner):
        return {"accepted": False, "reason": "invalid_ownership"}
    n1 = sum(r.zone_safe_rps.values()) - max(r.zone_safe_rps.values())
    ceiling = n1 * (1 - r.reserve_fraction)
    allowed_bad = math.floor(r.eligible_requests * (1 - r.slo_target) + 1e-9)
    gates = {
        "n_minus_one": r.peak_rps <= ceiling,
        "error_budget": r.bad_requests <= allowed_bad,
        "workload_contract": r.workload_contract_defined is True,
        "gpu_compatibility": r.gpu_compatibility_verified is True,
        "dependency_graph": r.dependency_graph_reviewed is True,
        "network_storage": r.network_storage_tested is True,
        "workload_identity": r.workload_identity_verified is True,
        "tenant_isolation": r.tenant_isolation_verified is True,
        "artifact_signatures": r.artifact_signatures_verified is True,
        "checkpoint_restore": r.checkpoint_restore_verified is True,
        "recovery_targets": r.recovery_targets_defined is True,
        "telemetry": r.telemetry_verified is True,
        "paging": r.paging_verified is True,
        "rollback": r.rollback_approved is True,
        "budget": r.budget_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "n_minus_one_safe_rps": round(n1, 3),
            "admission_ceiling_rps": round(ceiling, 3),
            "allowed_bad_requests": allowed_bad,
            "remaining_error_budget": allowed_bad - r.bad_requests}

def example():
    return Review(
        {"zone-a": 100.0, "zone-b": 100.0, "zone-c": 100.0},
        150.0, 0.20, 0.999, 1_000_000, 700,
        "gpu-platform-sre", "cloud-platform-sre",
        True, True, True, True, True, True, True, True,
        True, True, True, True, True,
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
**Important:** Every policy flag is a synthetic assertion. The tool cannot authenticate owners, verify signatures, test real checkpoint restores, observe GPUs, check provider quotas or approve production deployment.

## 2. Pytest suite
Save as `test_architecture_review.py`:
```python
from dataclasses import replace
import pytest
from architecture_review import evaluate, example

def test_reference():
    r = evaluate(example())
    assert r["accepted"] and r["allowed_bad_requests"] == 1000

def test_n1_ceiling():
    assert evaluate(example())["admission_ceiling_rps"] == 160

def test_n1_overload():
    assert "n_minus_one" in evaluate(replace(example(), peak_rps=170))["failed_gates"]

def test_error_budget_exhausted():
    assert "error_budget" in evaluate(replace(example(), bad_requests=1001))["failed_gates"]

def test_missing_owner():
    assert evaluate(replace(example(), service_owner=""))["reason"] == "invalid_ownership"

def test_same_backup_owner():
    assert evaluate(replace(example(), backup_owner="gpu-platform-sre"))["reason"] == "invalid_ownership"

def test_insufficient_zones():
    assert evaluate(replace(example(), zone_safe_rps={"a": 100, "b": 100}))["reason"] == "insufficient_zones"

def test_invalid_capacity():
    assert evaluate(replace(example(), zone_safe_rps={"a": -1, "b": 100, "c": 100}))["reason"] == "invalid_zone_capacity"

def test_invalid_target():
    assert evaluate(replace(example(), slo_target=1))["reason"] == "invalid_numeric_input"

def test_invalid_requests():
    assert evaluate(replace(example(), bad_requests=-1))["reason"] == "invalid_request_counts"

def test_invalid_type():
    with pytest.raises(TypeError):
        evaluate({})

@pytest.mark.parametrize("field,gate", [
    ("workload_contract_defined", "workload_contract"),
    ("gpu_compatibility_verified", "gpu_compatibility"),
    ("dependency_graph_reviewed", "dependency_graph"),
    ("network_storage_tested", "network_storage"),
    ("workload_identity_verified", "workload_identity"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("artifact_signatures_verified", "artifact_signatures"),
    ("checkpoint_restore_verified", "checkpoint_restore"),
    ("recovery_targets_defined", "recovery_targets"),
    ("telemetry_verified", "telemetry"),
    ("paging_verified", "paging"),
    ("rollback_approved", "rollback"),
    ("budget_approved", "budget"),
])
def test_fail_closed_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_security():
    assert "tenant_isolation" in evaluate(replace(example(), tenant_isolation_verified="yes"))["failed_gates"]
```
Run:
```bash
python architecture_review.py
python -m pytest -q
```
Expected: **25 pytest cases** (12 standalone + 13 parameterized). Execution has not been performed.

## 3. Staging evidence checklist
1. Approved architecture diagram, workload contract and ADR.
2. Hardware and compatibility inventory: GPU, driver, CUDA, NCCL, Kubernetes, GPU Operator.
3. Per-zone measured safe capacity, N-1 synthetic zone-loss result and queue/stream SLO.
4. Dependency graph with owner, failure domain and shared service risks.
5. Workload identity, RBAC, network segmentation, signed artifact and tenant isolation tests.
6. Checkpoint integrity/restore evidence including optimizer and data cursor.
7. GPU/OTel/Prometheus dashboards, missing-data alerts and on-call page test.
8. Change, rollback, incident simulations, UTC timeline, cost and independent approvals.

## 4. Final 100-point rubric
Architecture and ADR (20), GPU/cloud topology (20), SLO/DR/recovery (20), security/ownership/incident response (20), lab/evidence/FinOps (20). Pass >=90 with no critical hard-gate failure.
