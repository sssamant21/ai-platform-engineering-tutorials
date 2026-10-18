# Chapter 084 — Executable Policy-as-Code, Change Management and Audit Lab

**Status:** Published; execution and independent security review pending. All identities, policy and audit records are synthetic.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Offline policy evaluator and audit chain
Save as `policy_audit.py`:
```python
from dataclasses import dataclass
from datetime import datetime, timezone
from hashlib import sha256
import json

@dataclass(frozen=True)
class DecisionInput:
    verified: bool
    tenant: str
    resource_tenant: str
    action: str
    scopes: frozenset[str]
    model_approved: bool
    budget_available: bool
    approval_until: datetime | None

SCOPES = {"infer": "inference:invoke", "retrieve": "retrieval:read", "export": "data:export"}

def decide(inp, now):
    if not isinstance(now, datetime) or now.tzinfo is None:
        raise ValueError("timezone-aware now required")
    if inp.verified is not True:
        return {"allow": False, "reason": "unauthenticated"}
    if not inp.tenant or inp.tenant != inp.resource_tenant:
        return {"allow": False, "reason": "tenant_denied"}
    required = SCOPES.get(inp.action)
    if required is None or required not in inp.scopes:
        return {"allow": False, "reason": "scope_denied"}
    if inp.model_approved is not True or inp.budget_available is not True:
        return {"allow": False, "reason": "release_or_budget_denied"}
    if inp.action == "export":
        expiry = inp.approval_until
        if not isinstance(expiry, datetime) or expiry.tzinfo is None or now >= expiry:
            return {"allow": False, "reason": "approval_expired"}
    return {"allow": True, "reason": "policy_allow"}

def append_audit(chain, record):
    if not isinstance(record, dict) or not {"decision_id", "action", "reason"} <= record.keys():
        raise ValueError("sanitized audit record required")
    if any(key in record for key in ("prompt", "token", "secret", "authorization")):
        raise ValueError("sensitive audit field")
    previous = chain[-1]["hash"] if chain else "0" * 64
    payload = json.dumps(record, sort_keys=True, separators=(",", ":"))
    value = sha256((previous + payload).encode()).hexdigest()
    chain.append({"previous": previous, "record": dict(record), "hash": value})
    return value

def verify_chain(chain):
    previous = "0" * 64
    for entry in chain:
        if entry["previous"] != previous:
            return False
        payload = json.dumps(entry["record"], sort_keys=True, separators=(",", ":"))
        expected = sha256((previous + payload).encode()).hexdigest()
        if expected != entry["hash"]:
            return False
        previous = expected
    return True

def sample():
    now = datetime(2026, 1, 1, tzinfo=timezone.utc)
    inp = DecisionInput(True, "alpha", "alpha", "infer",
                        frozenset({"inference:invoke"}), True, True, None)
    return inp, now

if __name__ == "__main__":
    inp, now = sample()
    result = decide(inp, now)
    chain = []
    append_audit(chain, {"decision_id": "demo-1", "action": inp.action, "reason": result["reason"]})
    print(json.dumps({"decision": result, "audit_valid": verify_chain(chain)}, indent=2))
```

**Limitations:** `verified`, `model_approved` and `budget_available` are synthetic assertions. A real PEP must obtain them from trusted identity, artifact and ledger systems. Hash chaining does not provide signatures, external anchoring, completeness, immutability or protection against an attacker rewriting the entire chain.

## 3. Automated tests
Save as `test_policy_audit.py`:
```python
from dataclasses import replace
from datetime import timedelta
import pytest
from policy_audit import DecisionInput, decide, append_audit, verify_chain, sample

def test_authorized_inference():
    inp, now = sample()
    assert decide(inp, now)["allow"]

def test_unverified_denied():
    inp, now = sample()
    assert not decide(replace(inp, verified=False), now)["allow"]

def test_cross_tenant_denied():
    inp, now = sample()
    assert decide(replace(inp, resource_tenant="beta"), now)["reason"] == "tenant_denied"

def test_scope_denied():
    inp, now = sample()
    assert not decide(replace(inp, scopes=frozenset()), now)["allow"]

def test_model_not_approved():
    inp, now = sample()
    assert not decide(replace(inp, model_approved=False), now)["allow"]

def test_budget_unavailable():
    inp, now = sample()
    assert not decide(replace(inp, budget_available=False), now)["allow"]

def test_export_without_approval():
    inp, now = sample()
    inp = replace(inp, action="export", scopes=frozenset({"data:export"}))
    assert decide(inp, now)["reason"] == "approval_expired"

def test_export_with_approval():
    inp, now = sample()
    inp = replace(inp, action="export", scopes=frozenset({"data:export"}), approval_until=now + timedelta(minutes=5))
    assert decide(inp, now)["allow"]

def test_expired_approval():
    inp, now = sample()
    inp = replace(inp, action="export", scopes=frozenset({"data:export"}), approval_until=now)
    assert not decide(inp, now)["allow"]

def test_unknown_action():
    inp, now = sample()
    assert not decide(replace(inp, action="admin"), now)["allow"]

def test_valid_audit_chain():
    chain = []
    append_audit(chain, {"decision_id": "1", "action": "infer", "reason": "policy_allow"})
    append_audit(chain, {"decision_id": "2", "action": "export", "reason": "approval_expired"})
    assert verify_chain(chain)

def test_audit_tamper_detected():
    chain = []
    append_audit(chain, {"decision_id": "1", "action": "infer", "reason": "policy_allow"})
    chain[0]["record"]["reason"] = "changed"
    assert not verify_chain(chain)

@pytest.mark.parametrize("key", ["prompt", "token", "secret", "authorization"])
def test_sensitive_audit_field_denied(key):
    chain = []
    with pytest.raises(ValueError):
        append_audit(chain, {"decision_id": "1", "action": "infer", "reason": "ok", key: "synthetic"})
```

Run:
```bash
python policy_audit.py
python -m pytest -q
```
Expected: **16 pytest cases** (12 standalone and 4 parameterized). Execution has not been performed.

## 4. Failure drills
1. Attempt cross-tenant inference and verify default deny.
2. Remove action scope or model approval and verify denial.
3. Attempt privileged export without time-bounded approval.
4. Expire an approval and verify fail-closed behavior.
5. Append two sanitized audit events and verify chain integrity.
6. Mutate an earlier audit record and verify detection.
7. Attempt to include a raw prompt or token field and verify rejection.
8. Plan staging integration tests for real OPA policy bundles, signed artifacts, policy rollback and immutable audit storage.

## 5. 100-point rubric
| Area | Points |
|---|---:|
| Policy scope and trust boundaries | 20 |
| Default deny and expiring approvals | 20 |
| Versioned change and rollback plan | 20 |
| Audit sanitization and integrity | 20 |
| Automated tests, runbooks and evidence | 20 |
| **Total** | **100** |

Educational target >=90/100 with zero critical policy bypass. Production approval requires independent policy evaluation, signed bundle verification, live canary tests and secure audit retention.
