# Chapter 092 — Executable Model Artifact, Registry and Configuration Recovery Lab

**Status:** Published. Offline tests, real registry recovery and independent security review pending.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Offline artifact verification and recovery gates
Save as `artifact_recovery.py`:
```python
from dataclasses import dataclass
from hashlib import sha256
import json
import re

SHA256_RE = re.compile(r"^[0-9a-f]{64}$")

@dataclass(frozen=True)
class Artifact:
    name: str
    content: bytes
    expected_sha256: str
    signed: bool
    approved_signer: bool

def verify_artifact(a):
    if not isinstance(a, Artifact):
        raise TypeError("Artifact required")
    if not isinstance(a.name, str) or not a.name:
        return False
    if not isinstance(a.content, bytes):
        return False
    if not isinstance(a.expected_sha256, str) or not SHA256_RE.fullmatch(a.expected_sha256):
        return False
    if type(a.signed) is not bool or type(a.approved_signer) is not bool:
        return False
    return (sha256(a.content).hexdigest() == a.expected_sha256
            and a.signed and a.approved_signer)

@dataclass(frozen=True)
class Release:
    model: Artifact
    tokenizer: Artifact
    model_revision: str
    tokenizer_compatible_with: str
    prompt_revision: str
    policy_approved: bool
    tenant_isolation_verified: bool
    inference_probe_passed: bool
    quality_gate_passed: bool
    rto_seconds: int
    rto_target_seconds: int
    owner_approved: bool

def can_restore(r):
    if not isinstance(r, Release):
        raise TypeError("Release required")
    if type(r.rto_seconds) is not int or type(r.rto_target_seconds) is not int:
        return False
    if r.rto_seconds < 0 or r.rto_target_seconds <= 0:
        return False
    if not all(isinstance(x, str) and x for x in
               (r.model_revision, r.tokenizer_compatible_with, r.prompt_revision)):
        return False
    checks = (r.policy_approved, r.tenant_isolation_verified,
              r.inference_probe_passed, r.quality_gate_passed, r.owner_approved)
    return (verify_artifact(r.model) and verify_artifact(r.tokenizer)
            and r.model_revision == r.tokenizer_compatible_with
            and all(type(x) is bool and x for x in checks)
            and r.rto_seconds <= r.rto_target_seconds)

def artifact(name, content=b"synthetic-test-data"):
    return Artifact(name, content, sha256(content).hexdigest(), True, True)

def example():
    return Release(
        artifact("model"), artifact("tokenizer"), "model-r7", "model-r7",
        "prompt-r17", True, True, True, True, 90, 120, True
    )

if __name__ == "__main__":
    print(json.dumps({"model_sha256": sha256(example().model.content).hexdigest(),
                      "recovery_allowed": can_restore(example())}, indent=2))
```

**Security limitation:** `signed` and `approved_signer` are operator-supplied booleans, **not cryptographic signature verification**. The lab checks hashes and asserted approvals only. Production must validate signatures against trusted identities, attestations and policy.

## 3. Automated tests
Save as `test_artifact_recovery.py`:
```python
from dataclasses import replace
from hashlib import sha256
import pytest
from artifact_recovery import Artifact, Release, artifact, verify_artifact, can_restore, example

def test_valid_artifact():
    assert verify_artifact(artifact("model"))

def test_corrupted_content():
    a = artifact("model")
    assert not verify_artifact(replace(a, content=b"corrupt"))

def test_wrong_digest():
    assert not verify_artifact(replace(artifact("model"), expected_sha256="0" * 64))

def test_invalid_digest_format():
    assert not verify_artifact(replace(artifact("model"), expected_sha256="latest"))

def test_unsigned():
    assert not verify_artifact(replace(artifact("model"), signed=False))

def test_unapproved_signer():
    assert not verify_artifact(replace(artifact("model"), approved_signer=False))

def test_compatible_release():
    assert can_restore(example())

def test_incompatible_tokenizer():
    assert not can_restore(replace(example(), tokenizer_compatible_with="model-r6"))

def test_corrupt_tokenizer():
    a = example()
    assert not can_restore(replace(a, tokenizer=replace(a.tokenizer, content=b"bad")))

@pytest.mark.parametrize("field", [
    "policy_approved", "tenant_isolation_verified", "inference_probe_passed",
    "quality_gate_passed", "owner_approved",
])
def test_failed_gate(field):
    assert not can_restore(replace(example(), **{field: False}))

def test_rto_exceeded():
    assert not can_restore(replace(example(), rto_seconds=121))

def test_rto_boundary():
    assert can_restore(replace(example(), rto_seconds=120))

def test_invalid_rto():
    assert not can_restore(replace(example(), rto_seconds=-1))

def test_missing_prompt_revision():
    assert not can_restore(replace(example(), prompt_revision=""))

def test_invalid_release_type():
    with pytest.raises(TypeError):
        can_restore({})
```

Run:
```bash
python artifact_recovery.py
python -m pytest -q
```

Expected: **19 pytest cases** (5 parameterized and 14 standalone). Execution has not been performed.

## 4. Staging extension
1. Create a synthetic signed model manifest using your organization's trusted signing workflow.
2. Verify artifact digests and actual signatures, signer identity and build provenance.
3. Simulate registry outage without interrupting already warmed inference.
4. Restore model and tokenizer from an immutable object version/approved DR replica.
5. Test model initialization, inference quality, safety and tenant isolation.
6. Measure transfer, warmup, recovery and total RTO.
7. Exercise invalid signatures, revoked IAM and stale region replica as blocked restores.
8. Capture UTC timeline, signed release IDs, approval, cost and RCA/COA.

## 5. 100-point assessment
| Dimension | Points |
|---|---:|
| Artifact inventory and compatibility | 20 |
| Hash integrity and trusted provenance | 20 |
| Recovery safety gates | 20 |
| RTO and inference validation | 20 |
| Tests, runbooks and audit evidence | 20 |
| **Total** | **100** |

Educational pass >=90/100 and no critical unsigned artifact, privacy or tenant-isolation bypass. Live recovery and independent security review remain pending.
