# Chapter 085 — Executable Security, Governance and FinOps Acceptance Lab

**Status:** Published; offline execution and independent review pending. All signals are synthetic assertions, not real control integrations.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Cross-domain acceptance engine
Save as `acceptance_lab.py`:
```python
from dataclasses import dataclass, replace
import json

@dataclass(frozen=True)
class Evidence:
    threat_model_approved: bool = True
    verified_identity: bool = True
    tenant_isolated: bool = True
    retrieval_acl_enforced: bool = True
    tool_policy_enforced: bool = True
    injection_drill_passed: bool = True
    workload_identity_verified: bool = True
    network_egress_enforced: bool = True
    secret_rotation_tested: bool = True
    artifact_digest_verified: bool = True
    signature_verified: bool = True
    sbom_approved: bool = True
    telemetry_minimized: bool = True
    retention_reviewed: bool = True
    quota_enforced: bool = True
    budget_reservation_atomic: bool = True
    gpu_slo_guardrails_passed: bool = True
    policy_change_approved: bool = True
    audit_integrity_checked: bool = True
    incident_runbooks_tested: bool = True

WEIGHTS = {
    "threat_model_approved": 5,
    "verified_identity": 5,
    "tenant_isolated": 5,
    "retrieval_acl_enforced": 5,
    "tool_policy_enforced": 5,
    "injection_drill_passed": 5,
    "workload_identity_verified": 5,
    "network_egress_enforced": 5,
    "secret_rotation_tested": 5,
    "artifact_digest_verified": 5,
    "signature_verified": 3,
    "sbom_approved": 2,
    "telemetry_minimized": 5,
    "retention_reviewed": 5,
    "quota_enforced": 5,
    "budget_reservation_atomic": 5,
    "gpu_slo_guardrails_passed": 5,
    "policy_change_approved": 5,
    "audit_integrity_checked": 5,
    "incident_runbooks_tested": 10,
}
# 20 evidence fields, 100 total points.
CRITICAL = {
    "verified_identity", "tenant_isolated", "retrieval_acl_enforced",
    "tool_policy_enforced", "network_egress_enforced",
    "artifact_digest_verified", "signature_verified",
    "telemetry_minimized", "quota_enforced", "budget_reservation_atomic",
    "policy_change_approved",
}

def evaluate(evidence):
    if not isinstance(evidence, Evidence):
        raise TypeError("Evidence required")
    values = vars(evidence)
    if any(type(v) is not bool for v in values.values()):
        raise ValueError("all evidence must be boolean")
    score = sum(weight for key, weight in WEIGHTS.items() if values[key])
    failed = sorted(key for key, value in values.items() if not value)
    critical_failed = sorted(set(failed) & CRITICAL)
    return {
        "score": score,
        "max_score": sum(WEIGHTS.values()),
        "accepted": score >= 90 and not critical_failed,
        "failed": failed,
        "critical_failed": critical_failed,
    }

if __name__ == "__main__":
    print(json.dumps(evaluate(Evidence()), indent=2))
```

**Important:** The booleans are assertions supplied by the lab operator; setting them to `True` does not prove a real security control. A production acceptance system must consume independently verified test evidence, signed artifacts, live telemetry, identity and billing systems.

## 3. Automated tests
Save as `test_acceptance_lab.py`:
```python
from dataclasses import replace
import pytest
from acceptance_lab import Evidence, WEIGHTS, evaluate

def test_perfect_score():
    r = evaluate(Evidence())
    assert r["accepted"] and r["score"] == 100

def test_weights_sum():
    assert sum(WEIGHTS.values()) == 100

def test_noncritical_failure_still_passes_above_threshold():
    r = evaluate(replace(Evidence(), threat_model_approved=False))
    assert r["accepted"] and r["score"] == 95

def test_multiple_noncritical_failures_under_threshold():
    r = evaluate(replace(Evidence(), threat_model_approved=False, secret_rotation_tested=False, retention_reviewed=False))
    assert not r["accepted"] and r["score"] == 85

@pytest.mark.parametrize("field", [
    "verified_identity", "tenant_isolated", "retrieval_acl_enforced",
    "tool_policy_enforced", "network_egress_enforced",
    "artifact_digest_verified", "signature_verified",
    "telemetry_minimized", "quota_enforced", "budget_reservation_atomic",
    "policy_change_approved",
])
def test_critical_gate_blocks(field):
    r = evaluate(replace(Evidence(), **{field: False}))
    assert not r["accepted"]
    assert field in r["critical_failed"]

def test_invalid_evidence_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_score_floor():
    values = {field: False for field in WEIGHTS}
    r = evaluate(Evidence(**values))
    assert r["score"] == 0 and not r["accepted"]

def test_no_critical_on_perfect_score():
    assert not evaluate(Evidence())["critical_failed"]

def test_failed_list_sorted():
    r = evaluate(replace(Evidence(), incident_runbooks_tested=False, sbom_approved=False))
    assert r["failed"] == sorted(r["failed"])
```

Run:
```bash
python acceptance_lab.py
python -m pytest -q
```
Expected: **19 pytest cases** (8 standalone and 11 parameterized). Execution has not been performed.

## 4. Integration acceptance checklist
In staging, replace every synthetic assertion with actual evidence:
1. Real OIDC/JWT verifier and tenant-scoped RAG/tool calls.
2. Prompt-injection and data-leakage drills with synthetic tenants.
3. Workload identity token audience and secret rotation.
4. CNI-enforced NetworkPolicy allow/deny connectivity probes.
5. Real signed artifact provenance, model digest and SBOM validation.
6. Telemetry canary export checks and retention/deletion lifecycle.
7. Distributed atomic gateway budget reservations and retry storm.
8. GPU p95/TTFT/goodput, cost reconciliation and N-1 failover.
9. Signed policy change, audit storage and rollback.
10. Incident command exercise with owners, evidence and action items.

## 5. 100-point capstone rubric
| Domain | Points |
|---|---:|
| Threat model, identity and tenant isolation | 15 |
| Prompt injection, tool and retrieval security | 15 |
| Secrets, workload identity and network isolation | 15 |
| Artifact provenance and SBOM | 10 |
| Privacy, telemetry and retention | 10 |
| Gateway budgets and GPU FinOps | 15 |
| Policy, change control and audit | 10 |
| Incident runbooks, SLO and evidence | 10 |
| **Total** | **100** |

**Acceptance:** >=90/100 and no critical failures. Offline success does not authorize a production release; require independent security/privacy/FinOps sign-off and verified staging evidence.
