# Chapter 007 — Executable Infrastructure Security and Trust Boundary Lab

**Status:** Published. Offline pytest execution, real admission-controller validation and security review pending.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Fail-closed workload admission validator
Save as `security_admission.py`:
```python
from dataclasses import dataclass
import json
import re

_DIGEST = re.compile(r"^sha256:[0-9a-f]{64}$")

@dataclass(frozen=True)
class Workload:
    name: str
    environment: str
    owner: str
    tenant: str
    data_classification: str
    image_digest: str
    model_digest: str
    workload_identity: bool
    least_privilege_rbac: bool
    network_default_deny: bool
    approved_egress_only: bool
    tenant_isolation_verified: bool
    residency_approved: bool
    secrets_externalized: bool
    artifact_signatures_verified: bool
    privileged_container: bool
    host_path_mount: bool
    audit_logging_enabled: bool
    incident_owner_assigned: bool

def evaluate(w):
    if not isinstance(w, Workload):
        raise TypeError("Workload required")
    if any(type(v) is not str or not v.strip() for v in (w.name, w.owner, w.tenant)):
        return {"accepted": False, "reason": "missing_ownership"}
    if w.environment not in {"dev", "staging", "prod"}:
        return {"accepted": False, "reason": "invalid_environment"}
    if w.data_classification not in {"public", "internal", "confidential", "restricted"}:
        return {"accepted": False, "reason": "invalid_data_classification"}
    if not isinstance(w.image_digest, str) or not isinstance(w.model_digest, str):
        return {"accepted": False, "reason": "invalid_digest"}
    if not _DIGEST.fullmatch(w.image_digest) or not _DIGEST.fullmatch(w.model_digest):
        return {"accepted": False, "reason": "invalid_digest"}
    gates = {
        "identity": w.workload_identity is True,
        "rbac": w.least_privilege_rbac is True,
        "network_default_deny": w.network_default_deny is True,
        "egress": w.approved_egress_only is True,
        "tenant_isolation": w.tenant_isolation_verified is True,
        "residency": w.residency_approved is True,
        "secrets": w.secrets_externalized is True,
        "signatures": w.artifact_signatures_verified is True,
        "not_privileged": w.privileged_container is False,
        "no_host_path": w.host_path_mount is False,
        "audit": w.audit_logging_enabled is True,
        "incident_owner": w.incident_owner_assigned is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "workload": w.name, "environment": w.environment}

def example():
    return Workload(
        "inference-staging", "staging", "llmops-sre", "tenant-a", "confidential",
        "sha256:" + "a" * 64, "sha256:" + "b" * 64,
        True, True, True, True, True, True, True, True,
        False, False, True, True,
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```
**Scope limitation:** All booleans are synthetic declarations. A valid digest format is not proof that an artifact was signed or verified. This script does not inspect Kubernetes, IAM, cloud network enforcement, GPU memory isolation or real artifact signatures.

## 3. Automated pytest suite
Save as `test_security_admission.py`:
```python
from dataclasses import replace
import pytest
from security_admission import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_missing_owner():
    assert evaluate(replace(example(), owner=" "))["reason"] == "missing_ownership"

def test_missing_tenant():
    assert evaluate(replace(example(), tenant=""))["reason"] == "missing_ownership"

def test_invalid_environment():
    assert evaluate(replace(example(), environment="unknown"))["reason"] == "invalid_environment"

def test_invalid_classification():
    assert evaluate(replace(example(), data_classification="secret"))["reason"] == "invalid_data_classification"

def test_bad_image_digest():
    assert evaluate(replace(example(), image_digest="latest"))["reason"] == "invalid_digest"

def test_bad_model_digest():
    assert evaluate(replace(example(), model_digest="sha256:123"))["reason"] == "invalid_digest"

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

@pytest.mark.parametrize("field,gate", [
    ("workload_identity", "identity"),
    ("least_privilege_rbac", "rbac"),
    ("network_default_deny", "network_default_deny"),
    ("approved_egress_only", "egress"),
    ("tenant_isolation_verified", "tenant_isolation"),
    ("residency_approved", "residency"),
    ("secrets_externalized", "secrets"),
    ("artifact_signatures_verified", "signatures"),
    ("audit_logging_enabled", "audit"),
    ("incident_owner_assigned", "incident_owner"),
])
def test_required_controls(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("privileged_container", "not_privileged"),
    ("host_path_mount", "no_host_path"),
])
def test_prohibited_controls(field, gate):
    assert gate in evaluate(replace(example(), **{field: True}))["failed_gates"]

def test_nonboolean_identity():
    assert "identity" in evaluate(replace(example(), workload_identity="yes"))["failed_gates"]

def test_nonboolean_privilege():
    assert "not_privileged" in evaluate(replace(example(), privileged_container=0))["failed_gates"]

def test_restricted_classification():
    assert evaluate(replace(example(), data_classification="restricted"))["accepted"]
```
Run:
```bash
python security_admission.py
python -m pytest -q
```
Expected: **23 test cases** (11 standalone + 12 parameterized). Test execution has not been performed.

## 4. Approved staging validation
1. Collect sanitized Kubernetes RBAC, service-account, Pod Security and NetworkPolicy configurations.
2. Validate cloud workload identity and tenant-scoped storage access with test identities.
3. Confirm registry signature verification and immutable image/model digest pinning.
4. Attempt an unauthorized cross-namespace and cross-tenant access using approved synthetic fixtures; expect denial.
5. Test that application pods cannot run privileged, mount host paths or use unrestricted egress.
6. Verify GPU operator exceptions are narrowly scoped and audited.
7. Confirm secrets are absent from application logs, traces and incident tickets.
8. Exercise security incident paging, containment, rollback and evidence retention with security approval.

## 5. 100-point rubric
Threat model (20), identity/isolation (20), artifact/data protection (20), incident response (20), lab and evidence (20). Pass >=90 with no critical bypass.
