# Chapter 080 — Executable Model Artifact Supply Chain, SBOM and Provenance Lab

**Status:** Published; live execution and independent security review pending. Offline synthetic integrity gates only.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic artifact integrity gate
Save as `artifact_gate.py`:
```python
from dataclasses import dataclass
from hashlib import sha256
import json
import re

HEX64 = re.compile(r"^[0-9a-f]{64}$")

@dataclass(frozen=True)
class Release:
    model_sha256: str
    tokenizer_sha256: str
    image_digest: str
    source_approved: bool
    license_approved: bool
    sbom_present: bool
    provenance_verified: bool
    signature_verified: bool
    critical_findings: int
    revoked: bool

def digest(data):
    if not isinstance(data, bytes):
        raise TypeError("artifact must be bytes")
    return sha256(data).hexdigest()

def verify_release(release, model_bytes, tokenizer_bytes):
    if not isinstance(release, Release):
        raise TypeError("release required")
    if type(release.critical_findings) is not int or release.critical_findings < 0:
        raise ValueError("invalid findings")
    if not HEX64.fullmatch(release.model_sha256) or not HEX64.fullmatch(release.tokenizer_sha256):
        raise ValueError("invalid SHA-256")
    if not release.image_digest.startswith("sha256:") or not HEX64.fullmatch(release.image_digest[7:]):
        raise ValueError("invalid image digest")
    gates = {
        "model_digest": digest(model_bytes) == release.model_sha256,
        "tokenizer_digest": digest(tokenizer_bytes) == release.tokenizer_sha256,
        "source": release.source_approved is True,
        "license": release.license_approved is True,
        "sbom": release.sbom_present is True,
        "provenance": release.provenance_verified is True,
        "signature": release.signature_verified is True,
        "critical_vulnerabilities": release.critical_findings == 0,
        "not_revoked": release.revoked is False,
    }
    return {"approved": all(gates.values()), "gates": gates,
            "failed_gates": [k for k, value in gates.items() if not value]}

def sample():
    model = b"synthetic-model-shard-v1"
    tokenizer = b"synthetic-tokenizer-v1"
    release = Release(
        model_sha256=digest(model),
        tokenizer_sha256=digest(tokenizer),
        image_digest="sha256:" + digest(b"synthetic-container-image"),
        source_approved=True, license_approved=True, sbom_present=True,
        provenance_verified=True, signature_verified=True,
        critical_findings=0, revoked=False)
    return release, model, tokenizer

if __name__ == "__main__":
    release, model, tokenizer = sample()
    print(json.dumps(verify_release(release, model, tokenizer), indent=2))
```

**Important:** The booleans `signature_verified`, `provenance_verified` and `sbom_present` are **asserted inputs**, not actual cryptographic or scanner checks. A malicious caller could set them to `True`. A production admission controller must verify signatures, trusted signer identity, provenance subjects, policy, actual SBOM contents and registry digests independently.

## 3. Automated tests
Save as `test_artifact_gate.py`:
```python
from dataclasses import replace
import pytest
from artifact_gate import digest, sample, verify_release

def test_valid_release():
    r, model, tok = sample()
    assert verify_release(r, model, tok)["approved"]

def test_model_tampering():
    r, model, tok = sample()
    assert not verify_release(r, model + b"tampered", tok)["gates"]["model_digest"]

def test_tokenizer_tampering():
    r, model, tok = sample()
    assert not verify_release(r, model, tok + b"tampered")["gates"]["tokenizer_digest"]

@pytest.mark.parametrize("field", [
    "source_approved", "license_approved", "sbom_present",
    "provenance_verified", "signature_verified",
])
def test_required_gate_missing(field):
    r, model, tok = sample()
    assert not verify_release(replace(r, **{field: False}), model, tok)["approved"]

def test_critical_finding():
    r, model, tok = sample()
    assert not verify_release(replace(r, critical_findings=1), model, tok)["approved"]

def test_revoked_release():
    r, model, tok = sample()
    assert not verify_release(replace(r, revoked=True), model, tok)["approved"]

def test_invalid_model_digest():
    r, model, tok = sample()
    with pytest.raises(ValueError):
        verify_release(replace(r, model_sha256="not-a-digest"), model, tok)

def test_invalid_image_digest():
    r, model, tok = sample()
    with pytest.raises(ValueError):
        verify_release(replace(r, image_digest="latest"), model, tok)

def test_negative_findings():
    r, model, tok = sample()
    with pytest.raises(ValueError):
        verify_release(replace(r, critical_findings=-1), model, tok)

def test_digest_type():
    with pytest.raises(TypeError):
        digest("not bytes")
```

Run:
```bash
python artifact_gate.py
python -m pytest -q
```
Expected: **14 pytest cases** (9 standalone plus 5 parameterized). Execution has not yet been performed.

## 4. Optional real-tool staging validation
Use a disposable approved registry, real image digest and trusted identity configuration. The following are examples requiring tool installation and security-owner approval:
```bash
# Create an SPDX SBOM for a pinned staging image:
syft registry.example.invalid/team/llm-server@sha256:REPLACE_DIGEST -o spdx-json > sbom.spdx.json

# Scan the SBOM and fail on high severity:
grype sbom:sbom.spdx.json --fail-on high

# Verify trusted signer identity and immutable image digest:
cosign verify --certificate-identity-regexp 'YOUR_TRUSTED_IDENTITY' \
  --certificate-oidc-issuer 'YOUR_TRUSTED_ISSUER' \
  registry.example.invalid/team/llm-server@sha256:REPLACE_DIGEST
```
Record actual tool versions, trusted roots, signed provenance policy, model shard checksums and approvals. Do not report the offline simulation as a real signature verification.

## 5. Failure drills
1. Modify one model byte and verify the digest gate fails.
2. Modify tokenizer bytes and verify paired-release integrity fails.
3. Disable signature/provenance assertions and verify fail-closed policy.
4. Mark release revoked and confirm denial.
5. Add a critical vulnerability and confirm denial.
6. Supply malformed model or image digest and confirm validation error.
7. Document why a valid checksum cannot independently establish source trust.
8. Design a production admission test with real signatures, verified identities and SBOM policy.

## 6. Acceptance rubric
| Domain | Points |
|---|---:|
| Model/tokenizer/container inventory | 20 |
| Cryptographic digest and provenance design | 20 |
| SBOM and vulnerability gates | 20 |
| Registry admission, promotion and revocation | 20 |
| Automated tests, runbooks and evidence | 20 |
| **Total** | **100** |

Educational acceptance >=90/100 with no critical integrity failure. Real production release requires independent signature/provenance verification and security review.
