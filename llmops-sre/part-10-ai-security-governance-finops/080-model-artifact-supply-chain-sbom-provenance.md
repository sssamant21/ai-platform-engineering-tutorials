# Chapter 080 — Model Artifact Supply Chain, SBOM and Provenance

**Part 10 — AI Security, Governance and FinOps · Chapter 080 of 100**  
**Status:** Chapter and executable companion lab published. Live deployment validation and independent technical/security review pending.

## Learning objectives
Protect the end-to-end LLM model and inference artifact supply chain: acquisition, build, dependencies, model weights, tokenizer, adapters, container image, registry, signing, promotion, deployment admission, audit and rollback. Build an SBOM/provenance evidence bundle and enforce deterministic integrity gates.

## 1. Enterprise artifact supply chain
```text
approved upstream source + model license
             |
      pinned revision/digest
             |
  controlled download + malware/format review
             |
  isolated build / model packaging
             |
      SBOM + vulnerability scan
             |
  provenance attestation + signing
             |
  immutable artifact registry
             |
  promotion (dev -> stage -> prod)
             |
  admission verification + policy
             |
     inference runtime / GPU nodes
             |
  monitoring, revocation, rollback
```
Trust boundaries: external repositories, CI runner, dependency mirrors, signing key infrastructure, artifact registry, deployment controller and runtime. Treat model files, tokenizer definitions, chat templates, adapters, Python wheels, CUDA libraries and custom kernels as independent attack surfaces.

## 2. Threat model
| Threat | Asset | Prevention/detection |
|---|---|---|
| Model weight substitution | weights/adapters | pinned cryptographic digest, signed manifest |
| Unsafe model serialization | loader/runtime | prefer safe formats, restrict deserialization and sandbox conversion |
| Malicious tokenizer/chat template | prompt assembly | pin and review tokenizer and templates |
| Dependency compromise | Python/CUDA/container | lockfile, hashes, SBOM, scanning |
| CI runner compromise | provenance/signing | ephemeral hardened runners, isolated credentials |
| Registry tag overwrite | release artifact | deploy immutable digest, registry retention controls |
| Forged provenance | attestations | verify issuer, identity, signature and subject digest |
| License/data policy violation | model usage | license and approved-source review |
| Vulnerable model-serving image | inference runtime | patch, rescan and approved exception expiry |
| Rollback to revoked artifact | deployment | denylist/revocation and admission gate |

## 3. Artifact inventory and manifest
Every release should inventory:
- model identifier, upstream source, immutable revision and license;
- model weights and shard digests, adapter digests and tokenizer/chat-template versions;
- serving image digest, Python lockfile, CUDA/driver compatibility and runtime configuration;
- SBOM (e.g. SPDX or CycloneDX), vulnerability findings and exception approvals;
- signed provenance linking builder identity, build inputs, outputs and policy version;
- promotion approvals, evaluation evidence, deployment digest and rollback target.

**SBOM scope matters:** a container SBOM does not automatically enumerate model weights, training data, tokenizer assets or external downloaded artifacts. Maintain separate inventories or manifests for these.

## 4. Provenance and verification
SLSA-style provenance captures who built what, from which immutable inputs, using which trusted process. Sign the manifest or attestation with organization-approved key management or keyless identity; verify **signature, trusted issuer/identity, expected artifact subject digest, environment and policy** before deployment. A hash alone proves consistency with a reference hash, not trust in the reference source.

Illustrative signed release record:
```json
{
  "release_id": "llm-inference-staging-080",
  "model_revision": "approved-immutable-revision",
  "model_sha256": "<64-hex-digest>",
  "image_digest": "sha256:<64-hex-digest>",
  "tokenizer_sha256": "<64-hex-digest>",
  "sbom_format": "SPDX",
  "provenance_verified": true,
  "signature_verified": true,
  "license_approved": true
}
```
This is a conceptual record, not a substitute for verifying a real signed attestation.

## 5. CI/CD release gates
1. Restrict artifact sources and pin immutable revisions.
2. Download into an isolated staging area; verify expected checksums from an independently trusted channel.
3. Inventory model shards, tokenizer, adapters, code, container and dependency versions.
4. Generate SBOM and scan vulnerabilities/secrets/malware using approved tooling.
5. Record approved licenses, model risk classification and security exceptions with expiry.
6. Produce signed provenance and store attestations in a controlled registry.
7. Promote **the same digest** across environments, never rebuild untracked mutable tags.
8. Verify signatures, trusted builder identity, SBOM policy and digest at deployment admission.
9. Exercise revocation, quarantine and rollback to a known-good approved digest.
10. Preserve sanitized evidence and review approvals; no secrets in build logs.

## 6. Example production commands (review before use)
```bash
# Examples only: install and configure approved tooling separately.
syft registry.example.invalid/team/llm-server@sha256:REPLACE_WITH_REAL_DIGEST -o spdx-json > server.spdx.json
grype sbom:server.spdx.json --fail-on high
# Verify with organization-configured trust identity and attestation policy:
cosign verify --certificate-identity-regexp 'YOUR_APPROVED_BUILDER_IDENTITY' \
  --certificate-oidc-issuer 'YOUR_TRUSTED_OIDC_ISSUER' \
  registry.example.invalid/team/llm-server@sha256:REPLACE_WITH_REAL_DIGEST
```
These are **templates** requiring real digests, trusted identities and tool version checks; they have not been executed. Do not weaken scanning gates to accommodate an urgent deployment without formal risk acceptance.

## 7. Kubernetes deployment controls
- Deploy containers by immutable image digest, not `latest`.
- Verify artifact signatures and provenance with an approved admission controller/policy engine.
- Use restricted service accounts, read-only mounts, non-root processes and least-privilege artifact-store access.
- Validate model weights and tokenizer digests before serving; fail closed on mismatch.
- Cache only verified artifacts and prevent cross-tenant model registry access.
- Monitor unexpected model digest changes, signature failures, revoked artifact use and SBOM exception expiry.

## 8. Troubleshooting and incidents
| Symptom | Investigation | Containment |
|---|---|---|
| Signature verification fails | signer identity, issuer, trust roots, digest | block rollout |
| Model digest mismatch | cache, download, registry and manifest | quarantine artifact |
| SBOM incomplete | scanner scope, downloaded weights, dynamic dependencies | regenerate inventory |
| Critical CVE detected | package version, exploitability, runtime exposure | patch or approved exception |
| Tokenizer mismatch | tokenizer digest, chat template, revision | restore paired release |
| Provenance builder unexpected | CI identity, branch, attestation subject | deny promotion |
| Registry artifact revoked | deployed digest, active replicas | coordinated rollback |

## 9. Production runbooks

### RB-080-A — Artifact integrity alert
Stop rollout, record expected/observed digests, quarantine suspect object, verify independently trusted source and review access logs. Restore only an approved signed artifact.

### RB-080-B — Compromised signing identity
Revoke affected credentials/trust grants, inventory impacted attestations, rotate signing configuration, reverify releases and coordinate security incident response.

### RB-080-C — Critical dependency vulnerability
Identify affected deployed digests using SBOM inventory, assess exposure, patch and rebuild with new provenance; roll out via approved canary and rollback controls.

### RB-080-D — Model/adapter mismatch
Disable candidate, compare model/tokenizer/adapter manifests, run compatibility and safety evaluations and restore approved immutable release.

## 10. Acceptance scorecard (100 points)
| Domain | Points |
|---|---:|
| Source, model and dependency inventory | 20 |
| Digests, signatures and provenance | 20 |
| SBOM, vulnerability/license policies | 20 |
| Admission, promotion and rollback | 20 |
| Automated tests, incident runbooks and evidence | 20 |
| **Total** | **100** |

Critical blockers: unverified signature, unexpected digest, unapproved source/license, known compromised builder or revoked artifact. Documentation publication is not deployment approval.

## 11. Companion lab
[Chapter 080 — Executable Model Artifact Supply Chain, SBOM and Provenance Lab](../labs/080-model-artifact-supply-chain-sbom-provenance-check.md). Offline lab demonstrates hash/inventory gates with synthetic files. **It does not implement cryptographic signature verification or actual SBOM scanning.**

## References
- [SLSA Framework](https://slsa.dev/)
- [Sigstore Cosign](https://docs.sigstore.dev/cosign/)
- [SPDX](https://spdx.dev/)
- [CycloneDX](https://cyclonedx.org/)
- [NIST Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)
- [Kubernetes Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)

**Next:** Chapter 081 — Privacy-Safe Telemetry and Data Retention.
