# 6.2 — Model Artifacts, Registries and Storage Integration

**Part:** 06 — Model Serving · **Chapter:** 2 of 7  
**Audience:** TrueFoundry platform engineers, MLOps, cloud security, storage engineers and SREs  
**Safety:** Read-only inspection is **[SAFE-READ]**. Storage policies, credentials, model aliases, deployments and cache deletion require **[CHANGE-CONTROL]**; use nonproduction for experiments.

## 1. Learning objectives

Design a traceable model-artifact supply chain; compare model registries, object stores and volume mounts; enforce integrity and least-privilege access; investigate download latency, denied access, stale caches and incompatible artifacts; and validate a safe model-version rollback.

## 2. Architecture and ownership

```text
Training pipeline → model validation → immutable artifact + manifest
                                   ↓
                      Model registry / metadata catalog
                                   ↓ version ID, checksum, signature
                      S3 / Azure Blob / GCS / approved OCI registry
                                   ↓ private network, workload identity
TrueFoundry deployment specification → Kubernetes Pod
                                   ↓
                    init / runtime loader / cache / PVC
                                   ↓
                      framework deserialize + allocate
                                   ↓
                         warmup → Ready → inference
```

A registry often manages versions, lifecycle stages and metadata while an object store holds the actual bytes. Some systems combine both functions. TrueFoundry integration details and supported artifact types depend on the installed version: inspect the real model reference and deployment workflow instead of inventing a universal CLI or schema.

## 3. Model artifact contract

Pin **both** container image and model artifact. An immutable container image can still fetch a mutable model alias at startup.

| Field | Example | Why it matters |
|---|---|---|
| Model ID | `fraud-classifier` | Logical ownership |
| Immutable version | `2026-10-15-build-042` | Reproducibility |
| Artifact URI | `s3://approved-bucket/models/fraud/versions/042/` | Storage locator |
| Manifest | `manifest.json` | Enumerates required files |
| Checksums | SHA-256 per file | Detect corruption or replacement |
| Framework | PyTorch/ONNX/etc. version | Loader compatibility |
| Tokenizer/config | Pinned alongside weights | Prevent silent mismatches |
| Model schema | Input/output contract | Golden inference validation |
| Provenance | Training run, commit, dataset lineage | Audit and rollback |
| License/approval | Registry metadata | Usage restrictions |
| Identity | Scoped workload role | Least-privilege access |

A storage object version ID or content digest is stronger evidence than an unversioned object key. If bucket versioning is enabled, record the **actual** object version IDs. For multipart uploads, S3 ETags are not reliable SHA-256 content hashes.

## 4. Artifact layout and manifest

Example structure:

```text
models/fraud-classifier/versions/042/
  manifest.json
  config.json
  weights.onnx
  labels.json
  checksums.sha256
```

Illustrative manifest:

```json
{
  "model_id": "fraud-classifier",
  "version": "042",
  "format": "onnx",
  "runtime": "onnxruntime",
  "input_schema_version": "v3",
  "files": [
    {"path": "weights.onnx", "sha256": "<verified-sha256>"},
    {"path": "config.json", "sha256": "<verified-sha256>"}
  ]
}
```

The placeholder hashes must be replaced with real verified digests before use. Validate every file's checksum after download and before loading. Where supported, use signed manifests or attestation and a registry approval workflow. Never enable unsafe deserialization or arbitrary model repository code from untrusted sources.

## 5. Storage integration patterns

| Pattern | Strength | Operational tradeoff |
|---|---|---|
| Download from object storage on startup | Simple immutable version selection | Startup time, network dependency, repeated egress |
| Init container download into shared volume | Separate fetch/verify from serving | Init failures block Pod; shared volume size |
| Runtime lazy download | Lower initial footprint | Cold-start latency and first-request failures |
| Read-only PVC / pre-staged artifacts | Stable local access, controlled provisioning | CSI/storage topology, lifecycle and stale-version risk |
| Node-local cache | Faster repeated model loads | Cache invalidation, node affinity and disk pressure |
| OCI model artifact | Registry-centric provenance/versioning | Runtime and registry support vary |

**Choice criteria:** artifact size, cold-start SLO, storage consistency, update cadence, locality, compliance, node capacity, GPU utilization and rollback speed. Avoid embedding multi-GB weights into every application image unless deliberately chosen for portability and release atomicity.

## 6. AWS S3 integration

Prefer IAM roles for service accounts or the environment's supported workload identity rather than long-lived static keys. The identity that pulls the container image may differ from the identity that downloads the model.

Required permissions are usually narrowly scoped to model prefixes and metadata operations; include KMS decrypt permissions when using customer-managed encryption keys. A successful `ListBucket` does not prove `GetObject` permission and vice versa.

**Read-only, authorized checks:**

```bash
aws sts get-caller-identity
aws s3api head-object --bucket <approved-bucket> --key <approved-model-key>
aws s3api get-object-attributes --bucket <approved-bucket> --key <approved-model-key> --object-attributes ObjectSize Checksum
```

Object checksums may be absent or algorithm-specific. A `HeadObject` response is not by itself a complete cryptographic verification. Do not download confidential model bytes to an unmanaged workstation.

## 7. Azure Blob Storage integration

Use managed identity / workload identity federation where supported, with appropriately scoped Blob Data Reader permissions. Confirm tenant, identity audience, storage account firewall/private endpoint, DNS, container name, blob version and encryption policy.

```bash
az account show --query '{subscription:id,tenant:tenantId}' -o json
az storage blob show --account-name <storage-account> --container-name <container> --name <blob-path> --auth-mode login --query '{name:name,properties:properties.contentLength}' -o json
```

The local Azure CLI identity may differ from the Kubernetes workload identity; do not conclude workload access from an administrator's successful CLI command.

## 8. Google Cloud Storage integration

Prefer Workload Identity Federation for GKE or an approved workload identity mechanism; scope permissions to the required bucket/object prefix using supported IAM conditions where appropriate. Verify project, bucket location, service account identity and private egress.

```bash
gcloud auth list
gcloud storage objects describe gs://<approved-bucket>/<approved-model-object>
```

As with AWS/Azure, workstation identity tests are not substitutes for validating the serving Pod's actual identity.

## 9. Kubernetes storage, cache and startup

A model can be staged on an `emptyDir`, PVC or runtime cache. `emptyDir` is tied to Pod lifetime and can consume node ephemeral storage (or memory if memory-backed); PVC durability and sharing depend on access modes and CSI implementation.

Check:

```bash
kubectl -n <namespace> get pvc
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get resourcequota,limitrange
```

Use cache keys that incorporate **model ID + immutable version + digest**. Never treat a directory's existence as proof that a download completed successfully. Download to a temporary path, verify checksums, then atomically publish the ready artifact where filesystem semantics permit.

### Cache and concurrency risks

- Multiple replicas downloading the same large model may saturate storage/network and delay readiness.
- Concurrent writers to a shared cache need locking or atomic publish.
- Stale cache may silently serve a previous model if keys are mutable.
- Eviction during active use can corrupt assumptions or cause reload storms.
- GPU VRAM caching is separate from filesystem model caching.

## 10. Capacity and cold-start budget

Estimate:

```text
cold_start ≈ image_pull + artifact_transfer + verification
           + framework_initialization + model_allocation + warmup
```

Stages may overlap or be cached, so this is a planning decomposition rather than an exact formula. A 10 GiB model over an effective 200 MiB/s transfer path has a **best-case transfer floor** around 51 seconds before protocol overhead, contention, checksum verification and model initialization. Real measurements matter more than nominal bandwidth.

Track model bytes, observed transfer throughput, P50/P95 load time, failed downloads, cache hit rate, node disk usage, storage throttling and egress cost.

## 11. Security and compliance controls

1. Require approved model provenance, license and integrity validation.
2. Separate training publisher from inference reader identities.
3. Use least-privilege read access and avoid broad bucket/container grants.
4. Encrypt at rest and in transit; verify private networking requirements.
5. Rotate identities/keys without exposing values in logs.
6. Pin immutable versions and retain last-known-good artifact.
7. Sanitize artifact paths, tokens and model payloads in incident evidence.
8. Define retention, legal/compliance obligations and deletion procedures.
9. Protect against malicious model deserialization and remote-code execution.
10. Audit downloads, promotions, cache evictions and rollback decisions.

## 12. Troubleshooting commands [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,pvc -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=200
kubectl -n <namespace> logs <pod> --previous --tail=200
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get endpointslices -o wide
```

Inspect init-container states, volume mount events, image pull, application logs, startupProbe failures and readiness. Avoid printing environment variables or secret values.

## 13. Failure matrix

| Symptom | Evidence | First failed transition |
|---|---|---|
| 403 object download | Workload identity, cloud audit | Identity → object store |
| 404 model version | Exact bucket/key/version | Model reference → artifact |
| KMS decrypt denied | Encryption policy, audit | Identity → KMS |
| DNS timeout to private endpoint | Pod DNS/network path | Pod → storage endpoint |
| Slow download | Transfer metrics, storage throttling | Storage/network → loader |
| Corrupt weights | SHA-256 mismatch | Bytes → integrity check |
| Model config/tokenizer mismatch | Manifest and loader logs | Artifact set → runtime |
| PVC Pending | PVC events, StorageClass | Claim → provisioning |
| Volume mount failure | CSI/kubelet events | Storage → Pod |
| Node ephemeral storage full | Node pressure, events | Cache → node storage |
| Repeated cold starts | Cache hit ratio, Pod churn | Cache lifecycle → load |
| Old model served | Runtime model ID vs pinned manifest | Cache/reference → runtime |
| Model loads but OOM | Host/VRAM metrics | Allocation → runtime memory |
| Ready never becomes true | Startup/warmup logs | Model load → readiness |
| Rollback loads wrong model | Mutable alias or missing version | Release → artifact contract |

## 14. Recovery runbook

**A. Identify:** TrueFoundry deployment revision, image digest, immutable model ID/version, checksum, storage provider, Pod/node and first error time.

**B. Prove each boundary:** reference exists → workload identity authorized → network reachable → bytes transferred → checksum verified → loader compatible → model warmed → Pod Ready → golden inference request passes.

**C. Contain:** halt rollout, preserve evidence and protect known-good artifacts. Do not widen cloud IAM permissions or disable integrity checks as a shortcut.

**D. Recover:** restore an approved object version, fix scoped identity/network or return to the known-good **image + model** combination via the authoritative TrueFoundry/GitOps workflow. Do not blindly delete shared caches or PVCs.

**E. Verify:** checksum, actual loaded model version, readiness, endpoint traffic, golden prediction, cold-start duration, error rate and latency over a defined observation window.

## 15. Acceptance gates

| Gate | Evidence |
|---|---|
| G1 | Immutable model and image references recorded |
| G2 | Manifest, provenance and checksums verified |
| G3 | Cloud workload identity scoped and tested |
| G4 | Storage networking, KMS and mount requirements validated |
| G5 | Cache and concurrency safety documented |
| G6 | Cold-start and artifact-transfer metrics captured |
| G7 | Golden inference uses the intended loaded version |
| G8 | Known-good image+model rollback validated or tabletop documented |

## 16. Evidence template

```text
Date/time / TrueFoundry workspace / cluster:
Deployment revision and image digest:
Model ID / immutable version / checksum:
Manifest and approved artifact URI:
Storage provider, region and object version:
Workload identity (not credential values):
KMS and private network requirements:
Volume/cache type, key and capacity:
Transfer time / checksum result / load time:
Pod readiness / golden request:
Lowest Proven Healthy Layer:
First Failed Transition:
Approved remediation / owner:
Known-good model+image rollback:
Post-recovery validation and cleanup:
```

**Hands-on lab:** [6.2 — Model Artifact Storage and Integrity Check](../labs/02-model-artifacts-registries-storage-check.md)  
**Next:** Chapter 6.3 — CPU and GPU Inference Runtimes and Scheduling.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [AWS S3 Object Checksums](https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity.html)
- [Azure Blob Storage Security](https://learn.microsoft.com/en-us/azure/storage/blobs/security-recommendations)
- [Google Cloud Storage IAM](https://cloud.google.com/storage/docs/access-control/iam)
