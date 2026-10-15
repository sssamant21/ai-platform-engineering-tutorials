# Lab 6.2 — Model Artifacts, Registries and Storage Integration Check

**Canonical:** [Chapter 6.2](../02-model-artifacts-registries-storage/02-model-artifacts-registries-storage.md)  
**Mode:** **[SAFE-READ]** by default. Uploads, permission changes, cache deletion and deployment changes require a dedicated **[LAB-MUTATE]** nonproduction workspace.

## Objectives

Verify immutable model versioning, artifact manifests, checksum validation, cloud workload identity, storage access, cache safety and cold-start evidence. Diagnose 15 storage-related failures and document safe image+model recovery.

## 0. Safety preflight

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,pvc -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Confirm TrueFoundry workspace, namespace, model owner, approved storage prefix and least-privilege read identity. Do not expose model bytes, tokens or customer data.

## 1. Artifact identity and provenance

Record model ID, immutable version, registry metadata, image digest, framework version, input/output schema, manifest location, per-file SHA-256 and known-good rollback version.

| Field | Observed | Evidence |
|---|---|---|
| Model ID and version | | |
| Image digest | | |
| Model manifest and checksum | | |
| Object version/generation | | |
| Model runtime compatibility | | |
| Approved publisher | | |
| Workload read identity | | |

Check whether any model reference is mutable (such as `latest`, `production` or an unversioned object key). Document the drift risk.

## 2. Cloud metadata checks [SAFE-READ]

Choose **only the provider actually in use**. These are workstation identity checks unless executed with the actual approved workload identity.

### AWS S3

```bash
aws sts get-caller-identity
aws s3api head-object --bucket <approved-bucket> --key <approved-model-key>
```

If KMS encryption is used, confirm scoped decrypt permissions through approved IAM/audit evidence. A 403 is not necessarily a missing object.

### Azure Blob

```bash
az account show --query '{subscription:id,tenant:tenantId}' -o json
az storage blob show --account-name <storage-account> --container-name <container> --name <blob-path> --auth-mode login
```

Confirm Blob Data Reader scope, workload federation, private endpoint and storage firewall.

### Google Cloud Storage

```bash
gcloud auth list
gcloud storage objects describe gs://<approved-bucket>/<approved-object>
```

Confirm workload identity and object metadata. Do not treat a privileged local identity as proof that the Pod can download.

## 3. Pod storage and model-load evidence [SAFE-READ]

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> logs <pod> --previous --tail=150
kubectl -n <namespace> get pvc
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Capture sanitized download start/end, artifact size, checksum verification, load/warmup completion, cache location, cache key and readiness time. `--previous` may be unavailable.

## 4. Optional checksum exercise [LAB-MUTATE]

Use a **small synthetic artifact**, never proprietary weights. Create it in an approved nonproduction storage prefix, compute SHA-256, download into an isolated disposable environment, recompute SHA-256 and compare. Simulate a changed byte **only in a separate local copy**, proving the integrity check fails. Do not mutate shared model objects or overwrite immutable versions.

For a local synthetic file:

```bash
sha256sum <synthetic-file>
```

On Windows PowerShell:

```powershell
Get-FileHash -Algorithm SHA256 <synthetic-file>
```

Record actual computed hashes and expected failure result.

## 5. Cold-start and cache worksheet

| Signal | Cold start | Warm start | Interpretation |
|---|---|---|---|
| Artifact size | | | |
| Transfer duration | | | |
| Checksum verification | | | |
| Model load duration | | | |
| Warmup duration | | | |
| Ready transition | | | |
| Cache hit/miss | | | |
| Storage/network errors | | | |

Avoid deleting a production cache to manufacture a cold start. For capacity planning, include concurrency and storage egress cost.

## 6. Failure scenarios (tabletop)

| ID | Symptom | First failed transition |
|---|---|---|
| A1 | Object 403 | Identity → storage |
| A2 | Object 404 | Model reference → object |
| A3 | KMS decrypt denied | Identity → KMS |
| A4 | Private endpoint DNS fails | Pod DNS → storage |
| A5 | Download timeout | Network → artifact transfer |
| A6 | SHA-256 mismatch | Bytes → integrity |
| A7 | Tokenizer/config mismatch | Artifact set → loader |
| A8 | PVC Pending | Claim → provisioner |
| A9 | CSI mount failure | Volume → Pod |
| A10 | Ephemeral storage exhausted | Cache → node |
| A11 | Repeated cold starts | Cache → model load |
| A12 | Stale cached model | Version key → runtime |
| A13 | Host/GPU OOM on load | Model allocation → memory |
| A14 | Startup probe timeout | Load/warmup → readiness |
| A15 | Rollback loads mutable latest | Release → model reference |

For each scenario record lowest proven healthy layer, first failed transition, evidence, owner, minimal approved fix, and compatible image+model rollback.

## 7. Acceptance scorecard

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G1 Immutable version contract | | |
| G2 Manifest/checksum verified | | |
| G3 Workload identity and scope | | |
| G4 Storage network/KMS/mount | | |
| G5 Cache concurrency/invalidation | | |
| G6 Cold-start measurements | | |
| G7 Loaded version and golden request | | |
| G8 Rollback and cleanup | | |

## 8. Recovery evidence

```text
TrueFoundry workspace, deployment revision:
Image digest and model ID/version:
Registry and object version:
Artifact manifest and verified checksums:
Cloud provider, storage location and identity:
KMS/private network/mount requirements:
Cache key, volume type and lifecycle:
Cold-start transfer/load/warmup timings:
Actual loaded model version:
Golden inference result:
A1–A15 scenario classification:
Lowest Proven Healthy Layer:
First Failed Transition:
Remediation owner/approval:
Image+model rollback:
Post-recovery readiness/SLO:
Cleanup and unexecuted steps:
```

Delete only lab-owned synthetic files and disposable deployments after owner approval. Do not remove shared caches, PVCs, bucket objects or registry versions.

**Next:** Chapter 6.3 — CPU and GPU Inference Runtimes and Scheduling.
