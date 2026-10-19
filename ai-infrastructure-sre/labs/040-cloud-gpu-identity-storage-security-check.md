# Lab 040 — Cloud GPU Identity, Storage and Security Gate

**Python 3.11+ · pytest · offline synthetic evidence.** No cloud credentials, production secrets, GPU access or infrastructure mutations.

## Goal

Fail closed on missing AWS/Azure workload federation, overprivileged nodes, public storage, encryption/key readiness, unsigned images, GPU tenant isolation, checkpoint integrity/restore, stale audit, telemetry, incident readiness and change/rollback.

## File: `cloud_gpu_security_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class SecurityPlan:
    platform: str
    workload: str
    data_classification: str
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    restore_rto_min: float
    restore_rto_slo_min: float
    audit_age_s: float
    max_audit_age_s: float
    telemetry_age_s: float
    max_telemetry_age_s: float
    workload_identity_verified: bool
    federation_trust_scoped: bool
    node_role_least_privilege: bool
    object_access_scoped: bool
    public_access_blocked: bool
    encryption_at_rest_verified: bool
    tls_in_transit_verified: bool
    kms_key_available: bool
    key_access_least_privilege: bool
    private_endpoint_verified: bool
    egress_policy_verified: bool
    gpu_tenant_isolation_verified: bool
    privileged_runtime_boundary_verified: bool
    signed_image_verified: bool
    immutable_digest_verified: bool
    checkpoint_checksum_verified: bool
    checkpoint_restore_tested: bool
    retention_versioning_verified: bool
    audit_coverage_verified: bool
    secrets_rotation_verified: bool
    incident_runbook_verified: bool
    change_approved: bool
    rollback_tested: bool

def evaluate(p):
    if not isinstance(p, SecurityPlan):
        raise TypeError("SecurityPlan required")
    if p.platform not in ("aws", "azure"):
        return {"accepted": False, "reason": "invalid_platform"}
    if (not isinstance(p.workload, str) or not p.workload.strip()
            or p.data_classification not in ("internal", "confidential", "restricted")):
        return {"accepted": False, "reason": "invalid_identity"}
    metrics = (p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.restore_rto_min, p.restore_rto_slo_min,
               p.audit_age_s, p.max_audit_age_s,
               p.telemetry_age_s, p.max_telemetry_age_s)
    if (any(type(x) not in (int, float) or not math.isfinite(x) or x < 0 for x in metrics)
            or min(p.checkpoint_rpo_min, p.restore_rto_slo_min,
                   p.max_audit_age_s, p.max_telemetry_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "restore_rto": p.restore_rto_min <= p.restore_rto_slo_min,
        "audit_freshness": p.audit_age_s <= p.max_audit_age_s,
        "telemetry": p.telemetry_age_s <= p.max_telemetry_age_s,
        "workload_identity": p.workload_identity_verified is True,
        "federation": p.federation_trust_scoped is True,
        "node_least_privilege": p.node_role_least_privilege is True,
        "object_access": p.object_access_scoped is True,
        "public_access": p.public_access_blocked is True,
        "encryption": p.encryption_at_rest_verified is True,
        "tls": p.tls_in_transit_verified is True,
        "kms": p.kms_key_available is True,
        "key_least_privilege": p.key_access_least_privilege is True,
        "private_endpoint": p.private_endpoint_verified is True,
        "egress": p.egress_policy_verified is True,
        "gpu_isolation": p.gpu_tenant_isolation_verified is True,
        "privileged_boundary": p.privileged_runtime_boundary_verified is True,
        "signed_image": p.signed_image_verified is True,
        "digest": p.immutable_digest_verified is True,
        "checkpoint_checksum": p.checkpoint_checksum_verified is True,
        "checkpoint_restore": p.checkpoint_restore_tested is True,
        "retention": p.retention_versioning_verified is True,
        "audit_coverage": p.audit_coverage_verified is True,
        "secret_rotation": p.secrets_rotation_verified is True,
        "incident_runbook": p.incident_runbook_verified is True,
        "change": p.change_approved is True,
        "rollback": p.rollback_tested is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "platform": p.platform, "workload": p.workload,
            "classification": p.data_classification}

def example():
    return SecurityPlan(
        "aws", "synthetic-training", "confidential",
        5.0, 30.0, 20.0, 60.0, 10.0, 120.0, 10.0, 120.0,
        *([True] * 23)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_cloud_gpu_security_policy.py`

```python
from dataclasses import replace
import pytest
from cloud_gpu_security_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_wrong_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_azure_baseline():
    assert evaluate(replace(example(), platform="azure"))["accepted"]

def test_bad_platform():
    assert evaluate(replace(example(), platform="gcp"))["reason"] == "invalid_platform"

def test_missing_workload():
    assert evaluate(replace(example(), workload=" "))["reason"] == "invalid_identity"

def test_bad_classification():
    assert evaluate(replace(example(), data_classification="public"))["reason"] == "invalid_identity"

def test_nan_rto():
    assert evaluate(replace(example(), restore_rto_min=float("nan")))["reason"] == "invalid_metric"

def test_negative_audit():
    assert evaluate(replace(example(), audit_age_s=-1))["reason"] == "invalid_metric"

def test_bool_as_metric():
    assert evaluate(replace(example(), checkpoint_age_min=True))["reason"] == "invalid_metric"

def test_checkpoint_rpo():
    assert "checkpoint_rpo" in evaluate(replace(example(), checkpoint_age_min=31))["failed_gates"]

def test_restore_rto():
    assert "restore_rto" in evaluate(replace(example(), restore_rto_min=61))["failed_gates"]

def test_stale_audit():
    assert "audit_freshness" in evaluate(replace(example(), audit_age_s=121))["failed_gates"]

def test_stale_telemetry():
    assert "telemetry" in evaluate(replace(example(), telemetry_age_s=121))["failed_gates"]

def test_nonboolean_approval():
    assert "change" in evaluate(replace(example(), change_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("workload_identity_verified", "workload_identity"),
    ("federation_trust_scoped", "federation"),
    ("node_role_least_privilege", "node_least_privilege"),
    ("object_access_scoped", "object_access"),
    ("public_access_blocked", "public_access"),
    ("encryption_at_rest_verified", "encryption"),
    ("tls_in_transit_verified", "tls"),
    ("kms_key_available", "kms"),
    ("key_access_least_privilege", "key_least_privilege"),
    ("private_endpoint_verified", "private_endpoint"),
    ("egress_policy_verified", "egress"),
    ("gpu_tenant_isolation_verified", "gpu_isolation"),
    ("privileged_runtime_boundary_verified", "privileged_boundary"),
    ("signed_image_verified", "signed_image"),
    ("immutable_digest_verified", "digest"),
    ("checkpoint_checksum_verified", "checkpoint_checksum"),
    ("checkpoint_restore_tested", "checkpoint_restore"),
    ("retention_versioning_verified", "retention"),
    ("audit_coverage_verified", "audit_coverage"),
    ("secrets_rotation_verified", "secret_rotation"),
    ("incident_runbook_verified", "incident_runbook"),
    ("change_approved", "change"),
    ("rollback_tested", "rollback"),
])
def test_required_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run:

```bash
python -m pip install pytest
python cloud_gpu_security_policy.py
python -m pytest -q
```

**Planned 37 pytest cases:** 14 standalone and 23 parameterized. Execution and live AWS/Azure policy enforcement validation remain pending.

## Ten synthetic security incidents

1. Workload identity federation misconfigured.
2. Node role exposes overbroad storage permissions.
3. Public object access accidentally enabled.
4. KMS/Key Vault key disabled.
5. Private endpoint or egress isolation broken.
6. GPU tenant isolation or privileged boundary violated.
7. Unsigned container or mutable image tag.
8. Checkpoint checksum mismatch or restore failure.
9. Audit log or telemetry pipeline stale.
10. Secret rotation, incident runbook, approval or rollback missing.

## Staging evidence

Capture redacted IAM/Entra trust and RBAC, node identity boundaries, S3/Blob encryption/public access, KMS/Key Vault availability, network endpoint/egress tests, GPU pod security, image signatures, checkpoint manifest/hash and restore RPO/RTO, audit retention, incident exercise and approvals. Test only on authorized synthetic data.

**Limitations:** Boolean attestations are not independent security verification. Production authorization, storage encryption, key policy, network isolation and data recovery require live controlled testing and reviewer signoff.

## 100-point assessment

Identity 20; storage/keys 20; GPU isolation/network/supply chain 20; incident/audit 20; tests/staging 20. Pass >=90 and zero critical blockers.
