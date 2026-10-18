# Chapter 077 — Executable Authentication, Authorization and Tenant Isolation Lab

**Status:** Published; execution and independent security review pending. This lab uses synthetic identities; it is **not a JWT, OAuth or mTLS implementation**.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic policy and admission engine
Save as `tenant_policy.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class Principal:
    subject: str
    verified: bool
    memberships: frozenset[str]
    scopes: frozenset[str]

@dataclass(frozen=True)
class Request:
    tenant: str
    resource_tenant: str
    action: str
    input_tokens: int
    output_tokens: int
    concurrent: int

ALLOWED_SCOPES = {"infer": "inference:invoke", "retrieve": "retrieval:read"}
MAX_INPUT = 8192
MAX_OUTPUT = 1024
MAX_CONCURRENT = 8

def decide(principal, request):
    if type(principal.verified) is not bool or not principal.verified or not principal.subject:
        return {"allow": False, "reason": "unauthenticated"}
    if not request.tenant or request.tenant not in principal.memberships:
        return {"allow": False, "reason": "tenant_not_member"}
    if request.resource_tenant != request.tenant:
        return {"allow": False, "reason": "resource_tenant_mismatch"}
    scope = ALLOWED_SCOPES.get(request.action)
    if scope is None or scope not in principal.scopes:
        return {"allow": False, "reason": "scope_denied"}
    counts = (request.input_tokens, request.output_tokens, request.concurrent)
    if any(type(x) is not int or x < 0 for x in counts):
        return {"allow": False, "reason": "invalid_usage"}
    if request.input_tokens > MAX_INPUT or request.output_tokens > MAX_OUTPUT or request.concurrent >= MAX_CONCURRENT:
        return {"allow": False, "reason": "quota_exceeded"}
    return {"allow": True, "reason": "policy_allow"}

def cache_namespace(principal, request):
    decision = decide(principal, request)
    if not decision["allow"]:
        return None
    return "tenant:" + request.tenant

if __name__ == "__main__":
    p = Principal("user-001", True, frozenset({"alpha"}), frozenset({"inference:invoke"}))
    r = Request("alpha", "alpha", "infer", 1000, 200, 1)
    print(json.dumps({"decision": decide(p, r), "cache_namespace": cache_namespace(p, r)}, indent=2))
```

**Security boundary:** `verified=True` is an **input assertion** supplied by a hypothetical trusted gateway, not cryptographic proof. A production system must validate token signatures, issuer, audience, expiration and key rotation using a vetted library before constructing `Principal`. Likewise, `cache_namespace` is a teaching aid, not a secure cache-key design.

## 3. Automated tests
Save as `test_tenant_policy.py`:
```python
import pytest
from tenant_policy import Principal, Request, decide, cache_namespace

P = Principal("user-001", True, frozenset({"alpha"}), frozenset({"inference:invoke", "retrieval:read"}))
R = Request("alpha", "alpha", "infer", 1000, 200, 1)

def test_valid_inference():
    assert decide(P, R)["allow"]

def test_unverified_identity():
    p = Principal("user-001", False, P.memberships, P.scopes)
    assert decide(p, R)["reason"] == "unauthenticated"

def test_missing_membership():
    assert decide(P, Request("beta", "beta", "infer", 100, 10, 1))["reason"] == "tenant_not_member"

def test_resource_scope():
    assert decide(P, Request("alpha", "beta", "infer", 100, 10, 1))["reason"] == "resource_tenant_mismatch"

def test_missing_action_scope():
    p = Principal("user-001", True, P.memberships, frozenset())
    assert decide(p, R)["reason"] == "scope_denied"

def test_retrieval_scope():
    assert decide(P, Request("alpha", "alpha", "retrieve", 100, 10, 1))["allow"]

def test_unknown_action():
    assert decide(P, Request("alpha", "alpha", "delete", 100, 10, 1))["reason"] == "scope_denied"

def test_cache_namespace():
    assert cache_namespace(P, R) == "tenant:alpha"

def test_denied_cache_namespace():
    assert cache_namespace(P, Request("alpha", "beta", "infer", 100, 10, 1)) is None

@pytest.mark.parametrize("input_tokens,output_tokens,concurrent", [
    (8193, 100, 1),
    (100, 1025, 1),
    (100, 100, 8),
    (-1, 100, 1),
    (100, -1, 1),
    (100, 100, -1),
])
def test_quota_and_usage_denial(input_tokens, output_tokens, concurrent):
    req = Request("alpha", "alpha", "infer", input_tokens, output_tokens, concurrent)
    assert not decide(P, req)["allow"]

def test_boundary_quota_allowed():
    assert decide(P, Request("alpha", "alpha", "infer", 8192, 1024, 7))["allow"]
```

Run:
```bash
python tenant_policy.py
python -m pytest -q
```
Expected: **16 pytest cases** (10 standalone and 6 parameterized). Actual execution has not been performed.

## 4. Failure drills
1. Attempt inference with an unverified principal and confirm fail-closed response.
2. Spoof the requested tenant; ensure membership, not a header, determines authorization.
3. Attempt cross-tenant retrieval by changing `resource_tenant`.
4. Remove required scope and confirm authorization denial.
5. Exceed token and concurrency quotas separately.
6. Confirm denied requests cannot obtain a cache namespace.
7. Plan real staging tests for JWT issuer/audience/signature failures and JWKS rotation using a vetted identity stack.
8. Verify audit records contain only sanitized subject reference, tenant class, action, decision and policy version—not raw tokens or prompts.

## 5. Evidence and 100-point acceptance
| Domain | Points |
|---|---:|
| Identity verification architecture | 20 |
| Tenant membership and resource scope | 20 |
| Inference/retrieval authorization | 20 |
| Quotas and cache isolation | 20 |
| Regression tests, audit and incident readiness | 20 |
| **Total** | **100** |

Educational acceptance >=90/100 with no critical boundary failures. Production approval requires real gateway/JWT verification, retrieval ACL enforcement, cache isolation, load testing and independent security review.
