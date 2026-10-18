# Chapter 092 — Model Artifact, Registry and Configuration Recovery

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 092 of 100**  
**Publication:** Tutorial and executable offline lab published. Live artifact restoration, registry failover and independent security/operational review pending.

## Learning objectives
Restore production LLM inference when model registries, artifact stores, signed model manifests, tokenizer revisions, prompt templates or runtime configuration become unavailable, corrupted, incompatible or compromised. Preserve model provenance, authorization, tenant isolation, recovery point/time objectives and auditability.

## 1. Architecture and recovery boundaries
```text
Git / CI approval -----> signed release manifest
                              |
                  model registry / catalog
                              |
                 immutable artifact storage
                    /                \
             primary region       DR replica
                    \                /
              checksum + signature gates
                              |
         model server / tokenizer / adapters / prompts
                              |
                  canary inference validation
                              |
                traffic promotion / rollback
```
A model release is a **dependency graph**, not only a weights file: weights, tokenizer, config, chat template, adapter, quantization profile, safety policies, runtime image, retrieval embedding model and route mapping must be compatible and independently versioned.

## 2. Recovery taxonomy
| Failure | Detection | Recovery approach |
|---|---|---|
| registry API outage | lookup errors, control-plane alerts | use approved immutable artifact manifest and verified replica/cache |
| artifact store unavailable | model load 5xx, object-store errors | fail over to approved replica with same digest |
| truncated/corrupted weights | SHA-256 mismatch, model load error | quarantine and restore digest-verified copy |
| missing tokenizer/config | load exception, semantic regression | restore compatible release bundle |
| poisoned/unauthorized artifact | signature or provenance failure | block deployment, escalate security |
| wrong adapter/base model | output quality/shape regression | restore pinned compatibility matrix |
| prompt/routing drift | unexpected behavior or model identity | GitOps revert to approved manifest |
| deleted release metadata | registry entry missing | rebuild from signed immutable manifests, not guessed tags |
| cross-region stale artifact | digest/revision mismatch | verify replication and explicit RPO |
| compromised credential | unexpected registry writes | revoke and rotate, audit access and validate immutable store |

## 3. RTO/RPO design
Define RPO for mutable release metadata, prompt configurations, registry catalog and artifact replication independently. Immutable weights may be reconstructable from an approved source but restoration still has a measured RTO. RTO includes incident detection, artifact discovery, verification, transfer, model initialization, GPU warmup, synthetic inference, safety evaluation and route promotion.

## 4. Immutable manifest example
```yaml
release_id: llm-inference-2026-10-18-r7
model:
  uri: s3://approved-artifacts/models/example/model.safetensors
  sha256: REPLACE_WITH_VERIFIED_SHA256
tokenizer:
  uri: s3://approved-artifacts/tokenizers/example/tokenizer.json
  sha256: REPLACE_WITH_VERIFIED_SHA256
runtime:
  image: registry.example.invalid/inference@sha256:REPLACE_WITH_VERIFIED_IMAGE_DIGEST
configuration:
  prompt_revision: prompt-r17
  routing_revision: route-r9
  policy_revision: policy-r6
security:
  signed_release_required: true
  tenant_isolation_required: true
```
The manifest is **illustrative**, not an executable deployable release. Object URIs, digests and signature references must be resolved from a trusted registry and approved before use. A checksum alone detects byte changes but does not establish trusted provenance; verify signatures and signing identity separately.

## 5. Safe read-only diagnostics
```bash
# Approved Kubernetes context and namespace only
kubectl config current-context
kubectl get deploy,sts,pods -n llm-staging
kubectl describe pod <model-pod> -n llm-staging
kubectl get events -n llm-staging --sort-by=.lastTimestamp
kubectl logs -n llm-staging <model-pod> --since=30m

# Verify a local staged artifact, never an untrusted arbitrary URL
sha256sum ./staged/model.safetensors
sha256sum ./staged/tokenizer.json
```
Do not print credentials, signed download URLs, raw prompts, secrets or patient/customer data into shared incident channels.

## 6. Recovery workflow
1. Declare severity and freeze artifact/config promotions.
2. Determine blast radius: existing warmed replicas versus new starts, zones/regions, tenants and models.
3. Preserve sanitized evidence: registry events, object version IDs, manifest digest, release identity, UTC timeline.
4. Identify last known-good **signed** manifest; verify approval and current security policy.
5. Restore from versioned immutable storage or verified DR replica; do not trust mutable `latest` tags.
6. Verify bytes against pinned checksums **and** signatures/provenance, including runtime image and adapters.
7. Verify tokenizer, context length, chat template, quantization and embedding compatibility.
8. Validate secret scopes, artifact read permissions, tenant ACLs and supply-chain policy.
9. Stage model load on isolated capacity; run synthetic health, output format, safety and quality tests.
10. Canary approved traffic, observe TTFT, throughput, 5xx, memory and token spend; roll back if gates fail.
11. Reconcile registry metadata, replication status, backups and audit trail.
12. Close incident after sustained SLO recovery, independent review and RCA/COA.

## 7. Production runbooks

### RB-092-A — Registry unavailable
Check whether warmed inference remains healthy; freeze new promotions, use signed cached release manifests where authorized, restore registry API, verify metadata consistency and resume deployments only after reconciliation.

### RB-092-B — Artifact corruption
Quarantine mismatched objects; retrieve immutable version or verified replica, compare SHA-256 and signing identity, test load on staging GPUs and audit write permissions.

### RB-092-C — Tokenizer/config incompatibility
Compare release dependency graph and model runtime versions; restore compatible bundle, run golden prompts, structured-output tests and safety checks before canary.

### RB-092-D — Malicious or unauthorized artifact
Isolate affected release, revoke compromised credentials, engage security response, preserve forensic chain of custody, validate signer and build provenance; do not promote unverified fallback.

### RB-092-E — Cross-region artifact recovery
Confirm residency/legal constraints, replication completeness, region IAM, key availability and network throughput; measure actual restoration time and validate signed artifact equivalence.

### RB-092-F — Release metadata reconstruction
Rebuild catalog entries from approved signed manifests and object versions; ensure IDs/digests match deployment history and no tenant-specific config is silently lost.

## 8. Troubleshooting
| Symptom | Check | Guardrail |
|---|---|---|
| model download 403 | workload identity, bucket policy, key access | no broad IAM grants |
| model download 404 | URI, version ID, replication | no mutable-tag guessing |
| digest mismatch | transfer truncation, corruption, overwrite | quarantine, never override |
| signature invalid | trust root, signer, expiry, policy | block deployment |
| CUDA OOM after restore | quantization, GPU SKU, KV cache, batch | restore tested profile |
| quality drift | tokenizer, prompt, adapter, embedding | pinned golden evaluation |
| prolonged cold start | transfer bandwidth, cache, initialization | validate RTO realistically |
| stale registry metadata | eventual consistency, lost update | reconcile from immutable evidence |

## 9. Staging game-day scenarios
1. Registry API returns 503 while existing serving remains healthy.
2. Model artifact checksum mismatch.
3. Missing tokenizer file.
4. Incorrect adapter/base-model pair.
5. Revoked object-store access.
6. Invalid signing identity.
7. DR region artifact replica lags.
8. Model cache cold start exceeds RTO.
9. Restored model produces quality regression.
10. Credential compromise and artifact quarantine.

## 10. Acceptance rubric (100 points)
| Domain | Points |
|---|---:|
| Release graph and artifact inventory | 20 |
| Integrity, provenance and security | 20 |
| Registry/DR restore and RTO/RPO | 20 |
| Model compatibility and inference SLO validation | 20 |
| Automated tests, runbooks, evidence and RCA/COA | 20 |
| **Total** | **100** |

Educational pass >=90/100 with no critical provenance, privacy, tenant isolation or unsigned artifact bypass. Offline tests do not certify production recovery.

## 11. Companion lab
[Chapter 092 — Executable Artifact and Registry Recovery Lab](../labs/092-model-artifact-registry-configuration-recovery-check.md) uses standard-library SHA-256 checks, immutable manifest validation, compatibility and recovery gates with pytest.

## References
- [SLSA Supply-chain Levels for Software Artifacts](https://slsa.dev/)
- [Sigstore Cosign](https://docs.sigstore.dev/cosign/)
- [Kubernetes Image Digests](https://kubernetes.io/docs/concepts/containers/images/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [Google SRE Workbook — Disaster Recovery](https://sre.google/workbook/disaster-recovery/)

**Next:** Chapter 093 — Backup, DR, RTO/RPO and Region Failover.
