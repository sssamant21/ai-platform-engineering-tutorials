# Chapter 052 — Object Storage and Dataset Layout

**AI Infrastructure Engineering & SRE · Part 07 — AI Storage and Data Pipelines · Chapter 052/100**

> **Publication status:** The examples are reference designs. Offline validation is separate from cloud deployment, actual object-store load testing, security review and production acceptance.

## Objectives and prerequisites

Design reproducible, secure, cost-aware object-storage datasets for distributed AI training and inference. Cover S3/Azure Blob semantics, prefixes and partitioning, shard sizing, immutable manifests, atomic publication, checksums, versioning, concurrency, request throttling, encryption, identity, lifecycle, lineage, checkpoint artifacts and SRE recovery. Prerequisite: Chapter 051 plus Kubernetes/cloud identity fundamentals.

## 1. Enterprise architecture

```text
source / ingest / classification
           |
    raw versioned object zone
           |
 schema + checksum + quality gate
           |
 curated immutable shard objects
           |
 immutable manifest + publish pointer
           |
     training data loaders
       |          |
    cache      shared filesystem
       |          |
  GPU workers / model checkpoints
           |
 durable versioned artifacts + DR
           |
  metrics, audit, cost, SLO, lineage
```

**Never train against a mutable wildcard prefix when reproducibility matters.** Pin the dataset release to a manifest listing exact object keys, sizes, checksums and version identifiers where available. The release pointer should be published only after all referenced objects pass verification.

## 2. Object storage versus filesystem

Object stores provide key-based APIs, not POSIX directory and rename semantics. Modern S3 offers strong read-after-write and list consistency for successful operations, but that does not make a multi-object dataset update atomic. Azure Blob has its own consistency, blob types, conditional writes and lease capabilities. A safe cross-provider pattern is **immutable data objects plus a versioned manifest plus an atomic/conditional pointer update** appropriate to the provider. Document eventual downstream cache or metadata propagation separately.

## 3. Key and prefix layout

```text
s3://approved-ai-data/
  raw/source=events/ingest_date=2026-10-19/...
  curated/dataset=vision/v=2026-10-19T2100Z/
      shards/part-000001.parquet
      shards/part-000002.parquet
      manifest.json
  releases/dataset=vision/current.json
  checkpoints/job=<job-id>/run=<run-id>/step=<step>/...
  models/model=<model-id>/version=<version>/...
```

Avoid embedding secrets, PII or mutable user names in object keys. Partition by common query and training access paths, not by an unbounded number of tiny prefixes. Object key distribution, request limits and performance guidance are provider-specific and change over time.

## 4. Sharding and request economics

Tiny objects cause metadata/API overhead; excessively large objects hurt parallelism, retry granularity and tail latency. Select shard size from benchmarked reader concurrency, format, compression, sample access pattern, network and memory budgets.

```text
shard_count = ceil(dataset_bytes / target_shard_bytes)
effective_read_GBps = total_bytes_read / wall_clock_seconds
request_cost = GET_count * per_GET_price + LIST_count * per_LIST_price
```

**Synthetic example:** 1 TiB / 256 MiB = 4096 shards. At 64 parallel readers, that averages 64 shards per reader for a full epoch, before skew, retries and shuffle. This is an example, not a universal 256 MiB recommendation.

## 5. Dataset manifest and integrity

```json
{
  "schema_version": 1,
  "dataset_id": "vision-v1",
  "release": "2026-10-19T2100Z",
  "format": "parquet",
  "objects": [
    {
      "key": "curated/dataset=vision/v=2026-10-19T2100Z/shards/part-000001.parquet",
      "size_bytes": 268435456,
      "sha256": "<computed-sha256>",
      "version_id": "<provider-version-id-if-available>"
    }
  ],
  "publisher": "<workload-identity>",
  "lineage_ref": "<approved-lineage-record>"
}
```

Compute SHA-256 on actual content; do not assume object ETag is a SHA-256 or even a simple MD5, especially for multipart uploads or provider-specific implementations. Use conditional writes for release-pointer updates and retain the prior known-good pointer for rollback. Protect against a manifest that references unverified or missing objects.

## 6. Publication protocol

1. Allocate a unique release ID and immutable key namespace.
2. Upload shards with bounded concurrency, retries and checksums.
3. Verify expected count, byte size, hash and schema against a source manifest.
4. Write the final immutable manifest and validate its references.
5. Publish a small release pointer with a provider-supported conditional operation.
6. Record object versions, publisher identity, timestamp and lineage.
7. Validate a sample read by a production-equivalent workload identity.
8. Keep the prior release until consumers and rollback windows expire.

A release pointer must not be overwritten by concurrent publishers without conflict detection. Failed uploads must not be silently included in a manifest.

## 7. AWS S3 and Azure Blob controls

**S3:** use IAM roles/workload identity, bucket policy, KMS, versioning, replication where needed, access logs/CloudTrail, private network access and lifecycle rules. S3 request throttling or `503 Slow Down` calls for bounded exponential backoff with jitter, adaptive concurrency and prefix/load analysis.

**Azure Blob:** use managed identities/Entra authorization, RBAC, encryption, private endpoints, versioning/snapshots where applicable, immutability policies and storage analytics. Handle 429/503 and service-specific retry guidance.

Neither lifecycle rules nor replication alone prove that a logically corrupt dataset can be recovered.

## 8. Kubernetes and GPU data loaders

Avoid embedding static cloud keys in pod specs. Use EKS Pod Identity/IRSA or Azure Workload Identity when supported. Bound download concurrency to prevent object API storms, prefetch to local NVMe or parallel filesystems where beneficial, validate cached object checksums and monitor GPU idle time. Ensure dataset version is included in training run metadata and checkpoints.

## 9. SLOs and telemetry

Track manifest completeness, checksum failures, publish latency, GET/LIST request rate, 4xx/5xx/429/503, retry amplification, p95 first-byte/read latency, effective GB/s, cache hit ratio, loader wait, cost per epoch, checkpoint age, telemetry freshness and release rollback time. Segment by dataset release and job; protect tenant identifiers in logs.

## 10. Security, governance and FinOps

Apply dataset classification, least privilege, encryption in transit/at rest, object ownership, key rotation, retention/legal hold, approved deletion workflow, cross-region transfer budget and lineage/audit records. Lifecycle cleanup must be owner-approved and must never remove the last recoverable dataset release or checkpoint. Treat training data poisoning and accidental mutable overwrite as integrity incidents.

## 11. Twelve troubleshooting scenarios

1. Thousands of tiny objects overwhelm request throughput.
2. Hot prefix or request surge causes throttling.
3. Multipart upload completes with incorrect checksum expectations.
4. Concurrent publishers overwrite the release pointer.
5. Manifest references missing or corrupt shards.
6. Object key naming leaks sensitive identifiers.
7. Worker identity lacks KMS decrypt permission.
8. Cache serves stale dataset release.
9. Cross-region reads inflate latency and egress.
10. Lifecycle expiration removes rollback objects.
11. Kubernetes data-loader concurrency saturates the network.
12. Telemetry gaps hide 503 retries and GPU starvation.

## 12. Ten incident simulations

**INC-052-01:** missing shard after publication. **02:** checksum mismatch. **03:** stale release pointer. **04:** S3/Blob API throttling. **05:** object credential revocation. **06:** KMS access failure. **07:** cache poisoning. **08:** retention misconfiguration. **09:** regional outage. **10:** monitoring outage during dataset rollback.

For each: capture UTC timeline, dataset release, object version references, impacted jobs, integrity/SLO blast radius, containment, rollback, approval and RCA/COA.

## 13. Eight operational runbooks

**RB-052-A — Ingest:** verify schema, checksums and immutable object keys.

**RB-052-B — Publish:** validate manifest, conditional pointer update and concurrent writer control.

**RB-052-C — Throttling:** inspect 429/503, request concurrency, backoff and prefix distribution.

**RB-052-D — Corruption:** quarantine release, compare checksums and restore known-good version.

**RB-052-E — Identity:** inspect workload role, KMS access and audit logs.

**RB-052-F — Cost:** identify tiny-object/request amplification and cross-region egress.

**RB-052-G — Recovery:** restore immutable manifest and verify all referenced objects.

**RB-052-H — Governance:** approve lifecycle change, protect legal hold and document rollback.

## 14. Eight hands-on labs

1. Design raw/curated/checkpoint/model key namespaces.
2. Generate synthetic shards and SHA-256 checksums.
3. Build and validate an immutable manifest offline.
4. Calculate shard counts, request budgets and reader concurrency.
5. Simulate incomplete publication and concurrent pointer conflict.
6. Simulate API throttling, stale cache and checksum mismatch.
7. Run companion fail-closed Python policy tests.
8. Produce an SRE evidence packet and 100-point scorecard.

**Companion:** [Lab 052 — Object Storage and Dataset Layout Gate](../labs/052-object-storage-dataset-layout-check.md).

## 15. 100-point enterprise assessment

| Domain | Points |
|---|---:|
| Layout, shard strategy and dataset correctness | 20 |
| Request performance, caching and cost | 20 |
| Identity, encryption, tenancy and governance | 20 |
| SLO, incident response, retention and recovery | 20 |
| Automated tests, staging evidence and independent review | 20 |
| **Total** | **100** |

Pass >=90 with no missing-shard, integrity, access-control, retention, rollback or restore blocker. Offline synthetic pass does not prove live cloud acceptance.

## References

- [Amazon S3 performance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
- [Amazon S3 consistency model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Azure Blob Storage introduction](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-introduction)
- [Azure Blob concurrency](https://learn.microsoft.com/azure/storage/blobs/concurrency-manage)
- [Kubernetes workload identity](https://kubernetes.io/docs/concepts/security/service-accounts/)

**Next: Chapter 053 — FSx for Lustre and Parallel Filesystems.**
