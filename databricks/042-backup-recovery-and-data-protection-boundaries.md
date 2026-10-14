# Chapter 042 — Backup, Recovery and Data Protection Boundaries

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 09 — Reliability and Recovery  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform, Security and Data Recovery Engineering  
**Prerequisites:** Chapters 026–030 and 041  
**Safety:** All restoration exercises use approved disposable staging data. Do not run RESTORE, VACUUM, DROP, credential changes or cloud failover on production without explicit change approval, backup verification and rollback.

## 1. Mission

Define what must be protected in a Databricks platform, what the platform itself does and does not back up, and how to prove that data and configurations can be restored to a business-approved recovery point.

### Outcomes

- Differentiate Delta time travel, Delta RESTORE, object versioning, snapshots, replication and independent backups.
- Inventory storage, metadata, jobs, code, permissions, identity and networking dependencies.
- Establish measurable RPO, RTO, retention and recovery coverage.
- Build cross-cloud protection strategies for S3, ADLS Gen2 and GCS.
- Practice safe, non-destructive restore tests with validation and evidence.
- Recognize retention, key, deletion and corruption hazards.

## 2. Protection architecture

    Data sources / ingestion
             |
     Databricks jobs and SQL
             |
      Unity Catalog objects
             |
       Delta table storage
       /       |        \
    Data   Transaction   Metadata
    files      log      dependencies
       \       |        /
      Cloud object storage
             |
    Versioning / replication /
      independent backups
             |
    Isolated recovery target
             |
     Validation + consumers

Also protect workspace configuration, IaC, repositories, job definitions, schedules, permissions and secrets-management references. Backing up data files alone does not reconstruct the service.

## 3. The protection boundary

| Asset | Protection concern | Recovery evidence |
|---|---|---|
| Delta data files | Deletion, corruption, retention | Restored readable snapshot |
| Delta transaction log | Consistent table state | Transaction-consistent restore |
| Unity Catalog objects | Catalog/schema/permissions | Object inventory and grants |
| Managed tables | Managed storage/metadata semantics | Supported restore procedure |
| External tables | External storage ownership | Storage and metadata mapping |
| Volumes | File-level recovery | File integrity test |
| Jobs/workflows | Definitions, triggers, dependencies | IaC/export and test run |
| Notebooks/code | Source-of-truth and versions | Repository checkout |
| Compute policies | Drift and access control | IaC and policy comparison |
| Secrets and keys | Decryption and authentication | Approved recovery mechanism |
| Network/DNS | Connectivity dependencies | Rebuilt endpoint tests |
| Audit evidence | Retention/compliance | Immutable or protected archive |

Recovery ownership must be explicit for each row.

## 4. What Delta time travel is

Delta Lake can query earlier table versions while required transaction-log and data files remain available.

Time travel is **not** an independent backup. It may fail after required files are removed by retention operations or if the underlying storage is deleted or corrupted.

## 5. What Delta RESTORE does

RESTORE can move a Delta table to an earlier version or timestamp when the required files still exist. It is a table-changing operation that creates a new transaction and can affect downstream consumers and streams. It does not replace an isolated backup and is not automatically safe for production.

Use a disposable staging table for demonstrations.

## 6. Retention and VACUUM boundaries

Consider:
- Delta log retention and deleted-file retention.
- VACUUM lifecycle and data-file availability.
- Long-running readers and streaming consumers.
- Regulatory/legal retention requirements.
- Cloud object lifecycle policies.
- Replication and backup retention independent of Delta.

Never shorten retention or bypass VACUUM safety checks to free storage during an incident without an approved risk assessment.

## 7. Backups versus replication

| Mechanism | Helps with | Does not guarantee |
|---|---|---|
| Delta time travel | Recent logical rollback | Survival of source storage loss |
| Delta RESTORE | Supported table-version recovery | Independent backup |
| Object versioning | Recovery of prior object versions | Application-consistent table snapshot |
| Cross-region replication | Regional copy availability | Protection from replicated corruption/deletion |
| Snapshot/copy | Point-in-time preservation if consistent | Metadata/identity reconstruction |
| Independent immutable backup | Isolation from source failure | Immediate RTO without tested restore |
| IaC/repository | Configuration reconstruction | Recovery of stored data and secrets |

Design for logical corruption, accidental deletion, regional outage and compromised credentials separately.

## 8. Recovery point and recovery time

**RPO:** maximum tolerated loss/staleness of recoverable data.

**RTO:** maximum tolerated time to restore the consumer-visible service.

For each protected dataset, record last validated commit, last protected consistent point, observed restore time and downstream replay needs.

## 9. Lab A — Windows PowerShell environment preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks clusters list --output json
~~~

Confirm actual workspace host, cloud and principal. CLI inventory may be paginated or permission-filtered.

## 10. Lab B — Recovery coverage inventory

| Asset | Cloud/region | Owner | Protection | Retention | Last restore test | RPO/RTO |
|---|---|---|---|---|---|---|
| demo Delta table | | | | | | |
| UC metadata | | | | | | |
| Job definitions | | | | | | |
| Secrets/key references | | | | | | |
| IaC and networking | | | | | | |
| Audit archive | | | | | | |

Mark unknown protection as **unverified**, not protected.

## 11. Lab C — Read-only Delta inventory

~~~sql
DESCRIBE DETAIL demo_catalog.demo_staging.events;
DESCRIBE HISTORY demo_catalog.demo_staging.events;
~~~

Use an authorized existing staging Delta table. Record location, current version, table properties and last successful commit. Do not expose sensitive storage URLs in shared evidence.

## 12. Lab D — Read-only time-travel test

First verify a known retained version from history, then use:

~~~sql
SELECT COUNT(*) AS row_count
FROM demo_catalog.demo_staging.events VERSION AS OF 0;
~~~

Version 0 is illustrative and may no longer be readable. A failed historical read may indicate expired files, permissions or other errors; inspect the exact failure before concluding.

## 13. Lab E — Disposable recovery demonstration

Run only in an approved disposable schema:

~~~sql
CREATE TABLE IF NOT EXISTS demo_catalog.demo_staging.recovery_lab
USING DELTA AS
SELECT id, CONCAT('row-', CAST(id AS STRING)) AS label
FROM range(100);

SELECT COUNT(*) AS expected_100
FROM demo_catalog.demo_staging.recovery_lab;

DESCRIBE HISTORY demo_catalog.demo_staging.recovery_lab;
~~~

If the table already exists, do not assume its contents are new. For repeatability, use a unique disposable table name under an approved naming convention.

## 14. Lab F — Non-destructive logical recovery

Read an earlier retained version without changing the source:

~~~sql
SELECT COUNT(*) AS historical_rows
FROM demo_catalog.demo_staging.recovery_lab VERSION AS OF 0;
~~~

If version 0 is unavailable or does not represent the expected snapshot, select an appropriate retained version from DESCRIBE HISTORY. Prefer a new isolated target when testing actual restoration; do not issue RESTORE against the source table.

## 15. Lab G — Isolated copy validation

Where supported and authorized, create a disposable recovery copy:

~~~sql
CREATE TABLE demo_catalog.demo_staging.recovery_lab_copy
DEEP CLONE demo_catalog.demo_staging.recovery_lab;
~~~

A deep clone copies table data at the selected snapshot but is not automatically an independent disaster-recovery backup; assess storage location, permissions, retention and regional failure domain. Do not overwrite an existing recovery target.

~~~sql
SELECT COUNT(*) AS source_count
FROM demo_catalog.demo_staging.recovery_lab;

SELECT COUNT(*) AS recovered_count
FROM demo_catalog.demo_staging.recovery_lab_copy;
~~~

Validate more than row counts: uniqueness, aggregates, checksums where appropriate, schema, partitioning, grants and consumer compatibility.

## 16. Lab H — Synthetic RPO calculation

~~~powershell
$failure = [datetimeoffset]"2026-10-14T12:00:00Z"
$lastConsistentBackup = [datetimeoffset]"2026-10-14T11:48:00Z"
$restored = [datetimeoffset]"2026-10-14T12:43:00Z"
[pscustomobject]@{
  PotentialRecoveryGapMinutes = ($failure-$lastConsistentBackup).TotalMinutes
  ObservedRecoveryMinutes = ($restored-$failure).TotalMinutes
}
~~~

The result is a synthetic 12-minute recovery gap and 43-minute restoration. Actual RPO must be verified against the restored data and business recovery reference.

## 17. AWS S3 protection

Evaluate:
- Bucket versioning and lifecycle.
- S3 Object Lock where required and appropriately configured.
- Cross-region replication and replication monitoring.
- IAM, bucket policies, KMS key availability and separation of duties.
- Independent account/region backup where justified.
- Delta-consistent recovery and transaction log preservation.
- Restore costs, request volume and time.

S3 replication can replicate unwanted mutations or deletions depending on configuration; test the actual behavior.

## 18. Lab I — AWS read-only protection inventory

~~~powershell
aws sts get-caller-identity
aws configure get region
aws s3api get-bucket-versioning --bucket demo-staging-backup
aws s3api get-bucket-replication --bucket demo-staging-backup
aws s3api get-object-lock-configuration --bucket demo-staging-backup
~~~

Replace bucket with an authorized resource. Missing configurations may return an error; record it accurately rather than treating all errors as proof of no protection.

## 19. Azure ADLS Gen2 protection

Evaluate:
- Storage account redundancy and geo-replication behavior.
- Blob versioning, soft delete and point-in-time restore compatibility.
- Hierarchical namespace and feature limitations.
- Storage lifecycle, immutability and retention.
- RBAC, ACL, managed identity and Key Vault recovery.
- Private endpoint/DNS recovery.
- Data and Delta transaction consistency.

Feature combinations vary by storage account type and hierarchical namespace; validate current support before relying on a mechanism.

## 20. Lab J — Azure read-only protection inventory

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az storage account show --name demostagingstorage --resource-group demo-staging-rg --query "{name:name,sku:sku.name,kind:kind,allowBlobPublicAccess:allowBlobPublicAccess}" -o json
az storage account blob-service-properties show --account-name demostagingstorage --resource-group demo-staging-rg -o json
~~~

Replace names and confirm permissions. Redact account identifiers and endpoint details before sharing.

## 21. GCP Cloud Storage protection

Evaluate:
- Bucket location and redundancy characteristics.
- Object Versioning and soft delete policies.
- Retention policies and Bucket Lock where required.
- Cross-region data-copy strategy and recovery consistency.
- Service account permissions and KMS keys.
- Object lifecycle and cost.
- Delta table metadata and object generation consistency.

A dual-region bucket is not equivalent to a complete second-region Databricks environment.

## 22. Lab K — GCP read-only protection inventory

~~~powershell
gcloud config get-value project
gcloud storage buckets describe gs://demo-staging-backup
gcloud kms keyrings list --location=us-central1
~~~

Use authorized resources only. Verify the actual CLI version and required KMS permissions.

## 23. Workspace configuration recovery

Protect source-controlled definitions of:
- Jobs, schedules, tasks and dependencies.
- Cluster policies and permitted compute.
- SQL warehouse configuration where supported.
- Repositories and deployment pipelines.
- Service principals, groups and grants as code.
- Unity Catalog definitions and storage references.
- Alerting and observability configuration.
- Network endpoints, routing and DNS.
- Secret references and key identifiers, not secret values.

A workspace export alone does not guarantee restoration of managed metadata, external systems or access policies.

## 24. Unity Catalog metadata boundaries

Inventory:
- Metastore/workspace assignment.
- Catalog and schema definitions.
- Managed and external tables.
- External locations and storage credentials.
- Volumes and permissions.
- Ownership and grants.
- Lineage/audit requirements.

Catalog metadata recovery and regional availability are product-specific. Validate supported backup/DR procedures; do not attempt direct edits to managed metadata stores.

## 25. Secrets and encryption keys

Data recovery may fail even when all objects exist if KMS keys, credentials or service principals are unavailable.

- Maintain key ownership and rotation procedures.
- Document backup/recovery for secret manager configuration.
- Verify least-privilege access in the recovery region.
- Test decryption using authorized synthetic data.
- Keep recovery credentials out of repository artifacts.
- Plan for revoked/expired identities and key deletion safeguards.

## 26. Job and pipeline recovery

Restore order:
1. Cloud account/project/subscription and quotas.
2. Network, DNS, private endpoints and identity.
3. Storage and encryption keys.
4. Unity Catalog metadata and permissions.
5. Workspace settings and compute policies.
6. Jobs, schedules, code and libraries.
7. Data snapshot and required replay.
8. Data-quality and consumer acceptance.
9. Controlled activation of production schedules.

Prevent dual-active writers and duplicate replay effects.

## 27. Scenario A — Accidental Delta DELETE

**Evidence:** transaction history, affected rows, retained versions.

**Recovery:** read historical data or restore an isolated copy from an approved recovery point.

**Acceptance:** data correctness, lineage and no unintended downstream side effects.

## 28. Scenario B — VACUUM removed required files

**Evidence:** missing file errors, retention settings, cloud object protection.

**Recovery:** independent object-level or application-consistent backup if available.

**Acceptance:** complete transaction-consistent table and verified reads.

## 29. Scenario C — Storage bucket/container deletion

**Evidence:** cloud audit, lifecycle and deletion events.

**Recovery:** protected isolated backup or supported cloud recovery mechanism.

**Acceptance:** restored data, Delta logs, access and consumer functionality.

## 30. Scenario D — Replicated corruption

**Evidence:** corruption timestamp, replication status, affected objects.

**Recovery:** select a known-good immutable/isolated recovery point, not the latest replica blindly.

**Acceptance:** validated clean data and replay reconciliation.

## 31. Scenario E — Workspace configuration loss

**Evidence:** IaC drift, audit events and missing jobs/policies.

**Recovery:** version-controlled rebuild with approved credentials and governance.

**Acceptance:** functional jobs, permissions and monitoring.

## 32. Scenario F — Lost key or identity access

**Evidence:** KMS/secret access failures and audit.

**Recovery:** approved key/identity recovery procedure.

**Acceptance:** decryptability and least-privilege access.

## 33. Scenario G — Regional storage disruption

**Evidence:** provider incident and request metrics.

**Recovery:** tested alternate-region data and compute activation, subject to replication consistency.

**Acceptance:** agreed RTO/RPO and no split-brain.

## 34. Scenario H — Backup exists but restore is too slow

**Evidence:** restore throughput, data size, object counts and dependency delays.

**Recovery:** redesign backup granularity, warm standby or recovery workflow.

**Acceptance:** measured end-to-end RTO, not merely backup job success.

## 35. Production recovery runbook

1. Declare impact, severity and recovery objective.
2. Freeze unsafe writers/retries and protect evidence.
3. Identify last validated consistent recovery point.
4. Verify source corruption/deletion and blast radius.
5. Validate backup independence, retention and key access.
6. Restore into an isolated authorized target.
7. Verify Delta transaction consistency and metadata.
8. Validate row counts, invariants, schema and access.
9. Reconcile incremental data and downstream consumers.
10. Promote only after approval; record actual RTO/RPO and RCA.

## 36. Restore validation matrix

| Test | Expected evidence |
|---|---|
| Backup discovery | Recoverable snapshot identified |
| Object integrity | Expected files and checksums |
| Delta consistency | Readable transaction snapshot |
| Schema | Matches approved baseline |
| Data quality | Counts, uniqueness, business checks |
| Security | Authorized access succeeds; denied access fails |
| Replay | No duplicates or missing events |
| Consumer | Freshness and correctness restored |
| Performance | Recovery time within objective |
| Cost | Restore and steady-state cost reviewed |

## 37. Backup monitoring and alerts

Monitor:
- Backup completion and age.
- Restore test success and duration.
- Replication lag and failures.
- Storage deletion/retention policy changes.
- KMS key state and access failures.
- Delta retention/VACUUM policy drift.
- Configuration export/IaC drift.
- Dataset freshness and correctness.
- Recovery-point coverage and RTO trend.

Alert on protection gaps that threaten an approved RPO, not only on backup-job failure.

## 38. Acceptance checklist

- [ ] Critical data and configuration assets inventoried.
- [ ] Protection ownership defined.
- [ ] RTO/RPO and retention approved.
- [ ] Delta time travel limitations documented.
- [ ] Independent backup and replication behavior verified.
- [ ] Unity Catalog metadata dependencies recorded.
- [ ] Key/identity recovery tested.
- [ ] Isolated staging restore performed.
- [ ] Transaction consistency and correctness verified.
- [ ] Recovery cost and duration measured.
- [ ] Eight failure scenarios reviewed.
- [ ] Rollback and escalation documented.

## 39. GO / CONDITIONAL GO / NO-GO

**GO:** Critical assets have tested recovery paths, validated consistency, available keys and measured RTO/RPO.

**CONDITIONAL GO:** Noncritical gaps have approved mitigation, owner and deadline.

**NO-GO:** No verified recovery point, missing encryption keys, incomplete transaction logs, uncontrolled dual writers or untested restoration of critical assets.

## 40. Review questions

1. Why is Delta time travel not an independent backup?
2. How can cross-region replication copy a destructive change?
3. Why must the Delta transaction log be protected with data files?
4. What is the difference between a successful backup and a validated restore?
5. Why can workspace recovery fail even when all table files are present?

## 41. Official references

- Databricks Delta Lake: https://docs.databricks.com/aws/en/delta/
- Databricks Delta RESTORE: https://docs.databricks.com/aws/en/delta/history
- Databricks disaster recovery: https://docs.databricks.com/aws/en/security/disaster-recovery
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- AWS S3 versioning: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
- AWS S3 replication: https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html
- Azure Storage data protection: https://learn.microsoft.com/azure/storage/blobs/data-protection-overview
- Azure Storage redundancy: https://learn.microsoft.com/azure/storage/common/storage-redundancy
- Google Cloud Storage Object Versioning: https://cloud.google.com/storage/docs/object-versioning
- Google Cloud Storage soft delete: https://cloud.google.com/storage/docs/soft-delete
- Google SRE workbook: https://sre.google/workbook/

**Next:** Chapter 043 — Workspace and Infrastructure Disaster Recovery.
