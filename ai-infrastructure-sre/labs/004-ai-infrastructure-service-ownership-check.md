# Chapter 004 — Executable AI Infrastructure Ownership and Escalation Lab

**Publication:** Complete. **Offline test execution, live paging tests and review:** Pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Fail-closed service ownership evaluator
Save as `service_ownership.py`:
```python
from dataclasses import dataclass
import json

VALID_TIERS = {"tier0", "tier1", "tier2", "tier3"}
VALID_SEVERITIES = {"sev1", "sev2", "sev3", "sev4"}

@dataclass(frozen=True)
class Service:
    service_id: str
    environment: str
    primary_owner: str
    backup_owner: str
    support_tier: str
    severity: str
    runbook_url: str
    escalation_route: str
    slo_defined: bool
    recovery_targets_defined: bool
    change_approver_assigned: bool
    security_owner_assigned: bool
    oncall_coverage_verified: bool
    alert_delivery_tested: bool
    tenant_boundary_verified: bool

def evaluate(s):
    if not isinstance(s, Service):
        raise TypeError("Service required")
    strings = (s.service_id, s.primary_owner, s.backup_owner,
               s.runbook_url, s.escalation_route)
    if any(type(v) is not str or not v.strip() for v in strings):
        return {"accepted": False, "reason": "missing_required_metadata"}
    if s.primary_owner == s.backup_owner:
        return {"accepted": False, "reason": "missing_independent_backup"}
    if s.environment not in {"dev", "staging", "prod"}:
        return {"accepted": False, "reason": "invalid_environment"}
    if s.support_tier not in VALID_TIERS or s.severity not in VALID_SEVERITIES:
        return {"accepted": False, "reason": "invalid_classification"}
    gates = {
        "slo": s.slo_defined is True,
        "recovery": s.recovery_targets_defined is True,
        "change_approver": s.change_approver_assigned is True,
        "security_owner": s.security_owner_assigned is True,
        "oncall": s.oncall_coverage_verified is True,
        "alert_delivery": s.alert_delivery_tested is True,
        "tenant_boundary": s.tenant_boundary_verified is True,
    }
    failed = sorted(k for k, passed in gates.items() if not passed)
    return {"accepted": not failed, "failed_gates": failed,
            "service_id": s.service_id, "severity": s.severity,
            "support_tier": s.support_tier}

def example():
    return Service(
        "gpu-platform", "staging", "gpu-platform-sre", "cloud-platform-sre",
        "tier2", "sev2", "https://example.invalid/runbook/gpu-platform",
        "pager-gpu-platform", True, True, True, True, True, True, True,
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
All ownership and readiness flags are **synthetic assertions**; the script does not contact a paging system, inspect production service catalogs or authorize changes.

## 2. Pytest suite
Save as `test_service_ownership.py`:
```python
from dataclasses import replace
import pytest
from service_ownership import evaluate, example

def test_reference_service():
    assert evaluate(example())["accepted"]

def test_missing_primary():
    assert evaluate(replace(example(), primary_owner=""))["reason"] == "missing_required_metadata"

def test_missing_backup():
    assert evaluate(replace(example(), backup_owner=""))["reason"] == "missing_required_metadata"

def test_same_backup():
    assert evaluate(replace(example(), backup_owner="gpu-platform-sre"))["reason"] == "missing_independent_backup"

def test_missing_runbook():
    assert evaluate(replace(example(), runbook_url=" "))["reason"] == "missing_required_metadata"

def test_missing_route():
    assert evaluate(replace(example(), escalation_route=""))["reason"] == "missing_required_metadata"

def test_bad_environment():
    assert evaluate(replace(example(), environment="unknown"))["reason"] == "invalid_environment"

def test_bad_tier():
    assert evaluate(replace(example(), support_tier="tier9"))["reason"] == "invalid_classification"

def test_bad_severity():
    assert evaluate(replace(example(), severity="sev0"))["reason"] == "invalid_classification"

@pytest.mark.parametrize("field,gate", [
    ("slo_defined", "slo"),
    ("recovery_targets_defined", "recovery"),
    ("change_approver_assigned", "change_approver"),
    ("security_owner_assigned", "security_owner"),
    ("oncall_coverage_verified", "oncall"),
    ("alert_delivery_tested", "alert_delivery"),
    ("tenant_boundary_verified", "tenant_boundary"),
])
def test_critical_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

def test_nonboolean_gate():
    assert "oncall" in evaluate(replace(example(), oncall_coverage_verified="yes"))["failed_gates"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_sev1_classification():
    r = evaluate(replace(example(), severity="sev1"))
    assert r["accepted"] and r["severity"] == "sev1"

def test_prod_classification():
    assert evaluate(replace(example(), environment="prod"))["accepted"]
```
Run:
```bash
python service_ownership.py
python -m pytest -q
```
Expected: **20 pytest cases** (13 standalone + 7 parameterized). Execution has not been performed.

## 3. Approved staging exercises
1. Export a sanitized service catalog with owners, backup owners, tiers, runbooks and SLOs.
2. Compare declared owners with actual on-call schedules and escalation policies.
3. Trigger a synthetic test page with explicit approval; verify primary and backup acknowledgement.
4. Simulate GPU device-plugin failure, NCCL hang, inference TTFT breach and checkpoint recovery incident.
5. Verify a single incident commander, named technical workstream owners and customer communications.
6. Validate security stop conditions, change approver and rollback owner.
7. Preserve evidence, UTC timelines, RCA/COA and retest outcomes.

## 4. 100-point rubric
Service catalog (20), tier/severity contracts (20), RACI/escalation/security (20), SLO/runbook/recovery (20), lab and simulations (20). Pass >=90/100 with no critical ownership or safety gap.
