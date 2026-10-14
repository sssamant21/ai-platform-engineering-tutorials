# Chapter 030 — Storage Reliability, Security and Recovery Acceptance Capstone

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 06 — Storage, Data Access and Infrastructure Reliability (capstone)  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Cloud Platform, Security, Data Reliability, FinOps  
**Prerequisites:** Chapters 026–029  
**Safety:** Production is read-only. All mutation and fault-injection tests require approved disposable staging resources.

## 1. Mission

Validate a production-ready Databricks storage architecture by testing governance, cloud identity, private connectivity, Delta Lake transaction safety, retention, performance, cost visibility and recoverability. This capstone measures end-to-end **authorized, correct, timely and recoverable** data access rather than just cloud storage availability.

### Acceptance deliverables

1. Storage dependency inventory and trust-boundary diagram.
2. Unity Catalog storage credential and external location assessment.
3. Cloud IAM, encryption, private endpoint and audit review.
4. Delta table transaction, retention and recovery assessment.
5. Storage performance, growth and cost baseline.
6. Six staged incident simulations or approved tabletops.
7. Tested operational runbook and rollback strategy.
8. Signed GO / CONDITIONAL GO / NO-GO assessment.

## 2. Reference architecture

    Databricks workspace / jobs / SQL
                    |
             Unity Catalog grants
                    |
            External location
                    |
            Storage credential
                    |
        Cloud identity and encryption
                    |
           DNS / private networking
                    |
         S3 / ADLS Gen2 / GCS
                    |
         Delta log and data files
                    |
       Validated business output
                    |
        Monitoring / backup / DR
                    |
            Incident recovery

### Trust boundaries

- Workspace authentication and job run-as identity.
- Unity Catalog object privileges.
- Storage credential cloud trust.
- Cloud bucket/container policy, ACL and encryption.
- Network, private endpoint, DNS and TLS.
- Delta transaction and data integrity.
- Backup/replication and retention.
- Monitoring, incident response and change management.

A success at one layer does not establish success at another.

## 3. Roles and responsibilities

| Role | Responsibility |
|---|---|
| SRE lead | Reliability acceptance, incident coordination |
| Data owner | Correctness, freshness and business impact |
| Platform engineer | Workspace, compute and Unity Catalog |
| Cloud engineer | IAM, endpoints, storage, KMS and quotas |
| Security | Least privilege, audit and policy review |
| DBRE/data reliability | Delta transaction and recovery validation |
| FinOps | Storage cost, growth and unit economics |
| Change approver | Test scope, rollback and production readiness |

## 4. Safety and preflight

- Identify the authorized staging workspace, cloud account/project and region.
- Use only synthetic data and a uniquely named disposable table.
- Confirm cost cap, maintenance window, owners and rollback.
- Avoid production RESTORE, VACUUM, OPTIMIZE, policy mutation or fault injection.
- Do not delete raw Delta transaction log files or objects.
- Protect credentials, cloud resource IDs and audit logs in restricted evidence.
- Never relax IAM or public networking merely to pass a connectivity test.

## 5. Lab A — Windows PowerShell environment verification

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks catalogs list --output json
databricks storage-credentials list --output json
databricks external-locations list --output json
~~~

Check the actual workspace host and run-as identity. CLI output may be paginated or permission-filtered. Validate flags with the installed version.

## 6. Lab B — Cloud context checks

Run **only** the commands relevant to the approved cloud:

~~~powershell
# AWS
aws sts get-caller-identity
aws configure get region

# Azure
az account show --query "{subscription:id,tenant:tenantId}" -o json

# GCP
gcloud config get-value project
~~~

The shell identity is not necessarily the identity that Unity Catalog uses to access storage. These commands establish context, not end-to-end authorization.

## 7. Lab C — Storage dependency inventory

| Object or control | Owner | Scope | Dependencies | Evidence |
|---|---|---|---|---|
| Metastore | | Region/account | Workspaces | |
| Catalog and schema | | Governance | Jobs/tables | |
| Storage credential | | Cloud identity | Locations | |
| External location | | Storage prefix | Tables/volumes | |
| Cloud IAM and encryption | | Bucket/container/key | Credential | |
| Private network path | | Region/subnet | Compute | |
| Delta table | | Catalog/path | Readers/writers | |
| Backup/DR | | Recovery boundary | Storage | |
| Monitoring | | Alert scope | On-call | |

Document blast radius before any credential, network or retention change.

## 8. Lab D — Unity Catalog authorization assessment

Use authorized staging objects:

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_staging;
SHOW GRANTS ON EXTERNAL LOCATION demo_landing;
~~~

Confirm object names and current SQL support. Visibility of grants can be limited by privileges. Compare effective access with approved policy; do not assume missing visibility means no grants.

### Positive and negative access tests

1. Verify an authorized staging principal can read the synthetic dataset.
2. Verify an unauthorized staging principal cannot access it, **only if approved**.
3. Check cloud IAM/ACL and key permissions independently.
4. Preserve sanitized audit events and timestamp correlation.
5. Restore any changed staging grants.

## 9. Lab E — Private connectivity validation

From an approved test host with a network path comparable to Databricks compute:

~~~powershell
Resolve-DnsName storage-endpoint.example.invalid
Test-NetConnection storage-endpoint.example.invalid -Port 443
~~~

The endpoint is intentionally fictitious. Replace with the approved actual storage endpoint. DNS and TCP success do not prove data-plane authorization. Confirm that private endpoint and firewall policies remain enforced.

## 10. Lab F — Delta table baseline

~~~sql
DESCRIBE DETAIL demo_catalog.demo_schema.demo_delta_table;
DESCRIBE HISTORY demo_catalog.demo_schema.demo_delta_table LIMIT 20;
SHOW TBLPROPERTIES demo_catalog.demo_schema.demo_delta_table;
~~~

Capture current version, table type, file count, size, location sensitivity, transaction history, retention settings, and active consumers.

## 11. Lab G — Synthetic Delta recovery exercise

**Approved staging only.** Choose a unique disposable table name; below is illustrative.

~~~sql
CREATE TABLE IF NOT EXISTS demo_catalog.demo_staging.storage_capstone_lab (
  event_id BIGINT,
  status STRING,
  batch_id STRING
) USING DELTA;

INSERT INTO demo_catalog.demo_staging.storage_capstone_lab
VALUES (101, 'READY', 'batch-001'),
       (102, 'READY', 'batch-001');

DESCRIBE HISTORY demo_catalog.demo_staging.storage_capstone_lab;
~~~

Record the verified version after insertion.

~~~sql
UPDATE demo_catalog.demo_staging.storage_capstone_lab
SET status = 'PROCESSED'
WHERE event_id = 101;

SELECT * FROM demo_catalog.demo_staging.storage_capstone_lab
ORDER BY event_id;

DESCRIBE HISTORY demo_catalog.demo_staging.storage_capstone_lab;
~~~

### Recovery approval gate

Before RESTORE:
- Confirm the target version and required files exist.
- Confirm no concurrent writer is active.
- Confirm streaming/downstream effects are understood.
- Confirm recovery owner and change approval.
- Confirm a rollback/compensating correction plan.

~~~sql
-- Replace the illustrative version with the observed pre-update version.
-- Execute only on the approved disposable staging table.
RESTORE TABLE demo_catalog.demo_staging.storage_capstone_lab
TO VERSION AS OF 1;
~~~

RESTORE creates a new transaction and can affect downstream consumers. Validate current rows, uniqueness, history and expected output after the exercise. Do not assume version 1 is always correct.

## 12. Lab H — Retention and VACUUM risk review

Read-only:

~~~sql
SHOW TBLPROPERTIES demo_catalog.demo_schema.demo_delta_table;
DESCRIBE HISTORY demo_catalog.demo_schema.demo_delta_table LIMIT 20;
~~~

Assess:
- Time-travel requirements.
- Deleted data-file and log retention.
- Long-running and streaming readers.
- Backup/replication availability.
- Cloud object lifecycle and versioning.
- Recovery RPO/RTO.
- Impact of optimization and VACUUM.

Do not run VACUUM or shorten retention in production. Cloud lifecycle policies must not remove active Delta files or transaction logs.

## 13. Lab I — Storage performance and cost baseline

| Measure | Current | Target | Source |
|---|---|---|---|
| Table logical bytes | | | Table metrics |
| Physical object bytes | | | Cloud inventory |
| Active Delta files | | | DESCRIBE DETAIL |
| Read latency p95 | | | Workload/cloud metrics |
| Write/commit failures | | | Job/Delta telemetry |
| Object request count | | | Cloud metrics |
| Throttling rate | | | Cloud metrics |
| Monthly storage spend | | | Billing export |
| Cost per validated job | | | Job + billing |
| Restore rehearsal time | | | Staging evidence |

Logical table bytes may differ from physical cloud bytes due to versions, rewritten files, replication and backups.

## 14. Lab J — Offline storage efficiency model

~~~powershell
$logicalGB = 400.0
$physicalGB = 630.0
$activeFiles = 24000
$meanActiveFileMiB = ($logicalGB * 1024) / $activeFiles
[pscustomobject]@{
  LogicalGB = $logicalGB
  PhysicalGB = $physicalGB
  PhysicalToLogicalRatio = [math]::Round($physicalGB / $logicalGB, 2)
  MeanActiveFileMiB = [math]::Round($meanActiveFileMiB, 2)
}
~~~

This is a synthetic diagnostic example. The ratio does not alone prove waste; explain retention, object versions and protection requirements before proposing deletion.

## 15. Lab K — Cost and growth forecast

~~~powershell
$currentPhysicalTB = 4.0
$netGrowthTBPerMonth = 0.45
$months = 6
$projectedTB = $currentPhysicalTB + ($netGrowthTBPerMonth * $months)
[pscustomobject]@{
  CurrentTB = $currentPhysicalTB
  Months = $months
  ProjectedTB = [math]::Round($projectedTB, 2)
}
~~~

Model expected, high-growth and backfill scenarios. Include retention, optimization rewrite amplification, replication, egress, requests and backup costs. Use current cloud billing data for pricing.

## 16. Incident simulation A — Cloud IAM denial

**Inject/tabletop:** authorized staging workload receives storage 403 after an identity change.

**Investigate:** Unity Catalog grants, credential mapping, IAM trust, bucket/container policy and encryption key permissions.

**Recover:** restore approved least-privilege mapping.

**Pass:** intended access works; unrelated access remains denied.

## 17. Incident simulation B — Private endpoint/DNS outage

**Inject/tabletop:** staging compute cannot resolve or reach approved storage endpoint.

**Investigate:** DNS, private endpoint, routes, firewall and region.

**Recover:** restore approved private connectivity without enabling public exposure.

**Pass:** connectivity and authorized data access both succeed.

## 18. Incident simulation C — Delta concurrent write conflict

**Inject:** controlled conflicting synthetic writes or tabletop conflict evidence.

**Investigate:** overlapping jobs, transaction history, commit failure and idempotency.

**Recover:** coordinate writers and safely retry only repeatable work.

**Pass:** valid final snapshot, no duplicate batch effects.

## 19. Incident simulation D — Incorrect Delta update

**Inject:** modify one synthetic row in staging.

**Investigate:** history, affected data, downstream consumers and retained versions.

**Recover:** approved targeted correction or staging RESTORE.

**Pass:** correct rows, version history and consumer consistency.

## 20. Incident simulation E — Retention/lifecycle data loss

**Tabletop only:** lifecycle or VACUUM removes a required historical object.

**Investigate:** retention policy, cloud object inventory, backup and versioning.

**Recover:** use approved backup/cloud recovery; do not promise time travel after file removal.

**Pass:** documented RPO/RTO outcome and policy correction.

## 21. Incident simulation F — Storage throttling and cost spike

**Inject/tabletop:** high concurrent requests increase latency, retries and cost.

**Investigate:** cloud metrics, file count, workload concurrency and retry behavior.

**Recover:** bounded retries, pacing and staging-validated layout changes.

**Pass:** workload SLO and unit cost return toward baseline.

## 22. Troubleshooting decision tree

    Storage reliability failure
                |
    Identity and Unity Catalog grants valid?
      no -> governance and cloud IAM
      yes
                |
    DNS / TLS / private network healthy?
      no -> network path
      yes
                |
    Objects and Delta log available?
      no -> lifecycle / deletion / backup
      yes
                |
    Delta commits and snapshots consistent?
      no -> transaction / concurrency / recovery
      yes
                |
    Performance and cost within baseline?
      no -> layout / throttling / capacity
      yes -> validate business output and close

## 23. Production incident runbook

1. Declare severity using data correctness, business deadline and consumer impact.
2. Identify affected workspace, cloud, region, table and storage prefix.
3. Preserve UTC timeline, sanitized errors and recent changes.
4. Separate UC, IAM, encryption, network, storage and Delta failure domains.
5. Assess current data integrity and partial external side effects.
6. Confirm retention and independent recovery options.
7. Select least-destructive approved mitigation with rollback.
8. Validate authorized/unauthorized access and Delta snapshot consistency.
9. Confirm downstream output, performance and cost recovery.
10. Document RCA, remaining risk, owner and preventive actions.

## 24. Monitoring and SLO acceptance

Track:
- Authorized access success rate.
- Unauthorized access attempts and privilege drift.
- Storage latency, throttling and service errors.
- Delta commit failure and snapshot availability.
- Data freshness, completeness and duplicates.
- Cloud object growth and cost per validated job.
- Backup success, restore rehearsal time and data loss.
- Audit log ingestion and alert delivery.

Define thresholds using business criticality and measured baselines.

## 25. Change, rollback and cleanup

Before modifying storage credentials, IAM, DNS, table properties, retention or optimization:
- Record approved state and dependent consumers.
- Validate in staging with synthetic data.
- Establish cost and risk guardrails.
- Define rollback, compensating data correction and escalation.
- Revalidate both cloud and Unity Catalog permissions.
- Do not delete raw Delta log files.
- Remove only approved disposable staging resources after evidence capture.

Rollback of configuration does not reverse already deleted objects or external data side effects.

## 26. Acceptance scorecard

| Control | Status | Evidence / owner |
|---|---|---|
| Storage dependency inventory | Not assessed | |
| Credential and external location ownership | Not assessed | |
| Least-privilege and negative authorization tests | Not assessed | |
| Cloud encryption and IAM review | Not assessed | |
| Private endpoint and DNS verification | Not assessed | |
| Delta history and transaction integrity | Not assessed | |
| Time travel and restore rehearsal | Not assessed | |
| Retention/VACUUM/lifecycle safety | Not assessed | |
| Performance and cloud cost baseline | Not assessed | |
| Capacity and growth forecast | Not assessed | |
| Backup RPO/RTO and recovery evidence | Not assessed | |
| Audit/alert delivery and runbooks | Not assessed | |
| Six incident simulations | Not assessed | |
| Rollback and change approval | Not assessed | |

## 27. Go / no-go criteria

**GO:** All critical controls verified; staging recovery and alert validation pass; no unresolved critical security, integrity or recovery risks.

**CONDITIONAL GO:** Only documented noncritical gaps, with explicit risk acceptance, owner, mitigation and due date.

**NO-GO:** Unauthorized storage access, unvalidated Delta restore, missing critical backup, destructive lifecycle policy, unresolved data corruption, absent critical alert coverage or unapproved changes.

## 28. Final evidence template

    Assessment UTC:
    Cloud / region / workspace:
    Metastore / catalog / table:
    Storage credential / external location:
    IAM and encryption review:
    Network / private DNS review:
    Delta version / retention baseline:
    Authorized and denied access evidence:
    Synthetic recovery evidence:
    Backup RPO / RTO:
    Performance and cost baseline:
    Incident simulations A-F:
    Risks / owners / deadlines:
    Decision: GO / CONDITIONAL GO / NO-GO
    Approvals:

## 29. Review questions

1. Why must a storage reliability assessment test governance, cloud IAM and network separately?
2. Why does a Delta RESTORE require downstream consumer validation?
3. What can prevent time travel even if table history exists?
4. Why might physical object storage exceed logical Delta table size?
5. What evidence supports a production readiness decision?

## 30. Official references

- Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Storage credentials: https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/storage-credentials
- External locations: https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/external-locations
- Delta Lake: https://docs.databricks.com/aws/en/delta/
- Delta history: https://docs.databricks.com/aws/en/delta/history
- Delta VACUUM: https://docs.databricks.com/aws/en/delta/vacuum
- AWS S3: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- Azure Data Lake Storage: https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-introduction
- Google Cloud Storage: https://cloud.google.com/storage/docs

**Part 06 complete. Next:** Chapter 031 — Databricks Monitoring and Observability Architecture.
