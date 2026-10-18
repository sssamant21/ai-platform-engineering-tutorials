# Chapter 095 — Integrated Incident Response and Disaster Recovery Acceptance Lab

**Status:** Published. Offline tests, live integrated staging drills and independent acceptance review pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Deterministic acceptance harness
Save as `dr_acceptance.py`:
```python
from dataclasses import dataclass
import json

WEIGHTS = {
    "incident_command": 10, "fault_diagnosis": 10, "n_minus_one": 15,
    "model_integrity": 10, "data_recovery": 15, "security_fencing": 15,
    "slo_rto": 15, "rca_evidence": 10,
}

@dataclass(frozen=True)
class Acceptance:
    scores: dict[str, int]
    security_verified: bool
    tenant_isolation_verified: bool
    signed_model_verified: bool
    primary_writers_fenced: bool
    tool_workers_fenced: bool
    data_integrity_verified: bool
    residency_approved: bool
    rpo_verified: bool
    rto_verified: bool
    inference_slo_verified: bool
    owner_approved: bool

def evaluate(a):
    if not isinstance(a, Acceptance):
        raise TypeError("Acceptance required")
    if not isinstance(a.scores, dict) or set(a.scores) != set(WEIGHTS):
        raise ValueError("exact rubric domains required")
    for name, score in a.scores.items():
        if type(score) is not int or not 0 <= score <= WEIGHTS[name]:
            raise ValueError("invalid rubric score: " + name)
    critical = {
        "security_verified": a.security_verified,
        "tenant_isolation_verified": a.tenant_isolation_verified,
        "signed_model_verified": a.signed_model_verified,
        "primary_writers_fenced": a.primary_writers_fenced,
        "tool_workers_fenced": a.tool_workers_fenced,
        "data_integrity_verified": a.data_integrity_verified,
        "residency_approved": a.residency_approved,
        "rpo_verified": a.rpo_verified,
        "rto_verified": a.rto_verified,
        "inference_slo_verified": a.inference_slo_verified,
        "owner_approved": a.owner_approved,
    }
    failed = sorted(k for k, v in critical.items() if type(v) is not bool or not v)
    score = sum(a.scores.values())
    return {"score": score, "max_score": 100, "failed_critical_gates": failed,
            "accepted": score >= 90 and not failed}

def example():
    return Acceptance(dict(WEIGHTS), True, True, True, True, True,
                      True, True, True, True, True, True)

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

**Limitations:** Rubric scores and gates are asserted by an operator. The script cannot verify actual SLOs, restore points, fencing, signatures, data integrity or reviewer authorization. Production acceptance requires independent evidence from staging systems.

## 2. Automated tests
Save as `test_dr_acceptance.py`:
```python
from dataclasses import replace
import pytest
from dr_acceptance import WEIGHTS, Acceptance, evaluate, example

def test_full_score():
    r = evaluate(example())
    assert r["score"] == 100 and r["accepted"]

def test_passing_boundary():
    a = example()
    s = dict(a.scores)
    s["incident_command"] = 0
    assert evaluate(replace(a, scores=s))["accepted"]

def test_below_threshold():
    a = example()
    s = dict(a.scores)
    s["incident_command"] = 0
    s["fault_diagnosis"] = 0
    assert not evaluate(replace(a, scores=s))["accepted"]

@pytest.mark.parametrize("field", [
    "security_verified", "tenant_isolation_verified", "signed_model_verified",
    "primary_writers_fenced", "tool_workers_fenced", "data_integrity_verified",
    "residency_approved", "rpo_verified", "rto_verified",
    "inference_slo_verified", "owner_approved",
])
def test_critical_gate_blocks(field):
    a = replace(example(), **{field: False})
    r = evaluate(a)
    assert not r["accepted"] and field in r["failed_critical_gates"]

def test_missing_domain():
    a = example()
    s = dict(a.scores)
    s.pop("incident_command")
    with pytest.raises(ValueError):
        evaluate(replace(a, scores=s))

def test_extra_domain():
    a = example()
    s = dict(a.scores)
    s["invented"] = 1
    with pytest.raises(ValueError):
        evaluate(replace(a, scores=s))

def test_overweight_score():
    a = example()
    s = dict(a.scores)
    s["n_minus_one"] = 16
    with pytest.raises(ValueError):
        evaluate(replace(a, scores=s))

def test_negative_score():
    a = example()
    s = dict(a.scores)
    s["incident_command"] = -1
    with pytest.raises(ValueError):
        evaluate(replace(a, scores=s))

def test_invalid_gate_type():
    r = evaluate(replace(example(), security_verified="yes"))
    assert not r["accepted"]

def test_invalid_acceptance_type():
    with pytest.raises(TypeError):
        evaluate({})
```

Run:
```bash
python dr_acceptance.py
python -m pytest -q
```
Expected: **20 pytest cases** (11 critical-gate parameterizations plus 9 standalone tests). Execution has not been performed.

## 3. Integrated staging drills
1. Baseline end-to-end inference, stream completion, TTFT, RAG grounding and tool correctness.
2. Simulate bounded GPU/zone loss; verify N-1 throughput and queue health.
3. Interrupt deployment reconciliation while retaining existing serving routes.
4. Deny model registry access; verify immutable signed artifact fallback.
5. Inject tokenizer mismatch and assert deployment promotion is blocked.
6. Introduce vector-index replication lag and enforce RPO gates.
7. Simulate agent timeout; prove idempotency and writer fencing.
8. Practice approved DR region promotion with IAM/KMS/residency checks.
9. Restore primary and complete safe failback with budget reconciliation.
10. Score actual evidence, document gaps and request independent acceptance review.

## 4. Evidence checklist
- Incident ticket and UTC event timeline.
- Model/tokenizer/runtime/prompt digests and approval IDs.
- RPO checkpoint and actual RTO measurements per dependency.
- GPU N-1 capacity and traffic/error-budget graphs.
- Primary/tool writer fencing evidence.
- Tenant ACL, residency, identity and artifact-signature checks.
- Recovery synthetic probes and quality/SLO measurements.
- RCA/COA owners, deadlines and retest evidence.

**Educational pass:** >=90/100 with every critical gate passed. This lab does not issue a production readiness certification.
