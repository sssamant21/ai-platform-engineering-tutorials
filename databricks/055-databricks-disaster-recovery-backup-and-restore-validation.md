# Chapter 055 — Databricks Disaster Recovery, Backup and Restore Validation

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 12 — Disaster Recovery and Production Resilience  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform Engineering, Data Engineering, Security and DR Coordinators  
**Prerequisites:** Chapters 041–054  
**Safety:** Production discovery is read-only. Restoration, failover, deletion, table rollback, checkpoint manipulation, permission changes and infrastructure deployment require approved change authorization. Perform active drills on isolated staging fixtures. Never assume Delta time travel is a backup.

## 1. Mission

Design and validate disaster recovery for Databricks workspaces, infrastructure, data, pipelines and governance. Translate business recovery objectives into tested recovery procedures, independent backup protections, controlled failover and verifiable consumer acceptance.

## 2. Learning outcomes

- Define recovery point objective (RPO), recovery time objective (RTO) and maximum tolerable downtime.
- Inventory critical Databricks dependencies and failure domains.
- Distinguish table history, snapshots, backups, replication and source replay.
- Design cloud-aware, region-aware recovery architectures.
- Protect Delta data, checkpoints, configuration, identities and encryption keys.
- Recover jobs, bundles, Terraform-managed infrastructure and Unity Catalog configuration.
- Run safe staged recovery drills with measured timings.
- Prove restored data correctness, access controls and service-level readiness.

## 3. Recovery architecture

    Source systems / object storage / IAM / KMS
                      |
       Databricks workspace + Unity Catalog
                      |
          Pipelines / SQL / Delta tables
                      |
    Immutable configuration + protected backup
                      |
        Alternate account / region / workspace
                      |
       Restore -> validate -> controlled cutover
                      |
          Consumer acceptance + monitoring

A regional DR plan must address independent failures of cloud storage, KMS, identity, networking, metadata, orchestration and external consumers.

## 4. Define recovery objectives

**RPO:** Maximum acceptable loss of committed business data, measured from the last recoverable point to the failure.

**RTO:** Maximum acceptable elapsed time from declared disruption until the required service is restored and validated.

**MTD/MTPD:** Business-defined maximum tolerable disruption.

Set separate objectives for each critical data product and workload. A workspace becoming accessible does not mean its data product has met RTO.

## 5. Recovery tiers

| Tier | Example | Design implication |
|---|---|---|
| Critical | Time-sensitive customer/clinical ingestion | Low RPO/RTO, tested replication and failover |
| High | Core reporting data product | Recoverable snapshots plus rapid reprocessing |
| Standard | Internal analytics | Restore from source/backup within agreed window |
| Development | Disposable test environment | Recreate from code and synthetic fixtures |

Values must be approved by business owners rather than inferred from technology.

## 6. Failure-domain matrix

| Failure | Affected elements | Required recovery strategy |
|---|---|---|
| Accidental Delta mutation | Table contents, downstream outputs | Restore/version or independent backup, validate |
| Corrupted checkpoint | Stream state, exactly-once assumptions | Safe replay with writer fencing |
| Workspace misconfiguration | Jobs, policies, secrets, access | Versioned IaC and governed export |
| Cloud region outage | Compute, storage endpoint, networking | Alternate region plus accessible data/keys |
| Identity or KMS outage | Access to storage and services | Tested break-glass and key dependency plan |
| Storage deletion/ransomware | Data and backup access | Independent protected immutable backup |
| Metastore/governance issue | Catalog/privileges/metadata | Documented UC recovery and privilege reconstruction |
| Upstream source outage | Data arrival and freshness | Buffering, replay, alternative ingestion |

## 7. Recovery inventory

Record workspace URLs/IDs, region, account, Unity Catalog metastore, catalogs/schemas, external locations, storage credentials, jobs, bundles, compute policies, service principals, cloud roles, networking, storage paths, encryption keys, secrets references, Terraform state, CI/CD dependencies, monitoring and downstream consumers. Store sensitive details in restricted approved repositories.

## 8. Recovery data classification

Identify authoritative sources, derived tables, checkpoints, intermediate outputs and externally published data. Distinguish what can be recomputed from source from what requires independent backup. Record retention, legal constraints and key ownership.

## 9. Delta Lake recovery boundaries

Delta transaction history and time travel can support recovery while required data and log files remain available. VACUUM, retention settings, deleted storage objects, external system side effects and compromised storage can eliminate recovery options. An independent, access-isolated backup strategy is necessary for stronger durability and ransomware scenarios.

## 10. Cloud storage protections

**AWS:** S3 versioning, Object Lock where applicable, replication, cross-account backup isolation, KMS access and restore permissions.

**Azure:** ADLS/Blob versioning and soft delete where supported, immutability policies, geo-redundancy and Key Vault/key availability.

**GCP:** Cloud Storage object versioning, retention policies, bucket replication, cross-project isolation and Cloud KMS.

Verify product-specific behavior and policy constraints for the actual storage account/bucket and region. Replication alone may propagate deletions or corruption.

## 11. Configuration and infrastructure backup

Protect Git revisions, Terraform code, provider locks, backend configuration, module versions, CI/CD pipeline definitions, bundle definitions, policy definitions, permission matrices and runbooks. Terraform state is sensitive and must be protected separately with backend recovery/versioning. Git alone does not capture all manually configured or external state.

## 12. Unity Catalog recovery planning

Inventory metastore assignment, catalogs, schemas, grants, owners, external locations, storage credentials and managed/external table behavior. Understand which metadata is recreated through IaC versus restored through supported provider mechanisms. Do not assume a region outage can be solved by attaching the same metastore everywhere; validate supported architecture and regional constraints.

## 13. Backup validation principles

A backup is usable only if its integrity, accessibility, encryption keys, permissions, recovery procedure and target environment are verified. Test restoration periodically. Track backup creation time, last successful restore test, immutable retention, failure alerts and recovery-point age.

## 14. Lab A — Windows PowerShell identity preflight [SAFE-READ]

~~~powershell
git rev-parse HEAD
terraform version
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm workspace, principal, cloud account and region against the approved staging exercise.

## 15. Lab B — Workspace inventory [SAFE-READ]

~~~powershell
databricks jobs list --output json
databricks clusters list --output json
databricks cluster-policies list --output json
databricks warehouses list --output json
~~~

Capture redacted resource IDs, ownership and required dependencies. Results can be paginated or permission-filtered.

## 16. Lab C — Bundle and IaC reproducibility [SAFE-READ]

~~~powershell
databricks bundle validate --target staging
databricks bundle summary --target staging
terraform -chdir=infrastructure/environments/staging fmt -check -recursive
terraform -chdir=infrastructure/environments/staging init -backend=false
terraform -chdir=infrastructure/environments/staging validate
~~~

These checks establish static readiness, not proof that a restored workspace works.

## 17. Lab D — Delta table history [SAFE-READ]

On an approved synthetic staging table:

~~~sql
DESCRIBE HISTORY demo_catalog.demo_staging.synthetic_events;
DESCRIBE DETAIL demo_catalog.demo_staging.synthetic_events;
~~~

Record table location, format, operation history and retention assumptions. Verify that required historical files still exist before promising a particular restore point.

## 18. Lab E — Historical data comparison [SAFE-READ]

For an existing recoverable staging version identified in the previous lab:

~~~sql
SELECT COUNT(*) AS current_rows
FROM demo_catalog.demo_staging.synthetic_events;

SELECT COUNT(*) AS historical_rows
FROM demo_catalog.demo_staging.synthetic_events VERSION AS OF 0;
~~~

Version 0 is illustrative; it may be unavailable or unsuitable. Select an actually verified version and compare deterministic checksums, uniqueness and business invariants. Do not run broad scans against large production tables without approval.

## 19. Lab F — Safe staging restore rehearsal [MUTATING-STAGING]

Only with an explicitly authorized disposable synthetic table and a verified source version:

~~~sql
CREATE TABLE demo_catalog.demo_staging.synthetic_events_dr_copy
DEEP CLONE demo_catalog.demo_staging.synthetic_events;
~~~

Verify permissions, ownership, clone semantics, storage cost and copied data. A deep clone is not automatically an independent immutable backup or cross-region disaster recovery solution. Use a separately approved storage isolation plan when required.

## 20. Lab G — Recovery correctness validation [SAFE-READ]

~~~sql
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT event_id) AS distinct_ids,
  SUM(CASE WHEN event_id IS NULL THEN 1 ELSE 0 END) AS null_ids,
  MAX(event_time) AS latest_event_time
FROM demo_catalog.demo_staging.synthetic_events_dr_copy;
~~~

Compare to a manifest recorded before the drill. Include schema, expected values, permissions and downstream read acceptance.

## 21. Lab H — Cloud identity and backup context [SAFE-READ]

**AWS**

~~~powershell
aws sts get-caller-identity
aws configure get region
~~~

**Azure**

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
~~~

**GCP**

~~~powershell
gcloud config get-value project
gcloud auth list
~~~

Confirm source and recovery accounts/regions are intentionally distinct. Never use broad list permissions as a substitute for tested recovery permissions.

## 22. Lab I — Recovery timing calculation

~~~powershell
$failure = [datetimeoffset]"2026-10-14T12:00:00Z"
$lastRecoverable = [datetimeoffset]"2026-10-14T11:52:00Z"
$validatedRestore = [datetimeoffset]"2026-10-14T12:38:00Z"
[pscustomobject]@{
  RPO_Minutes = ($failure - $lastRecoverable).TotalMinutes
  RTO_Minutes = ($validatedRestore - $failure).TotalMinutes
}
~~~

Expected: 8-minute realized RPO and 38-minute realized RTO. In real drills, record UTC timestamps from evidence rather than assuming estimates.

## 23. Lab J — Tabletop regional outage [NO MUTATION]

1. Select a critical synthetic staging data product.
2. Document source region, destination region and required services.
3. Identify latest independently recoverable data point.
4. Identify key, identity, networking, storage and metadata dependencies.
5. Specify writer fencing and failover authority.
6. Define restoration steps and checkpoint/replay plan.
7. Define data integrity and consumer validation.
8. Record projected versus measured RPO/RTO and blockers.
9. Document failback strategy and split-brain prevention.
10. Obtain DR owner sign-off.

## 24. Pipeline replay and checkpoints

A checkpoint captures processing state, not necessarily all required source data. Verify source retention and idempotency before replay. Never delete or replace production checkpoints to “restart from zero” without assessing duplicate writes, lost events and downstream effects. Ensure only one authorized writer owns each output during cutover.

## 25. Disaster recovery sequencing

1. Declare incident and authorize recovery.
2. Preserve evidence and freeze unsafe writes.
3. Verify alternate cloud account, region, identity and storage access.
4. Confirm recovery-point age and key availability.
5. Restore network, identity and infrastructure dependencies.
6. Reconstruct workspace/governance configuration.
7. Restore or reattach validated data using supported procedures.
8. Restore jobs, compute policies, schedules and monitoring.
9. Run synthetic and business data acceptance.
10. Resume controlled writers and consumer traffic.
11. Observe SLOs, document actual RPO/RTO and plan failback.

## 26. Cross-region failover controls

Avoid simultaneous active writers unless the application and data architecture explicitly support them. Define leader ownership, fencing, routing changes, DNS/endpoint TTL, storage replication lag, secret availability, downstream destination consistency and rollback to the original region. Regional recovery must be rehearsed, not assumed.

## 27. Data restore decision matrix

| Failure | Potential recovery | Key validation |
|---|---|---|
| Incorrect table update | Delta historical restore if files retained | Rows, schema, downstream side effects |
| Deleted storage files | Independent backup or cloud object recovery | Object completeness, key access |
| Corrupted stream checkpoint | Approved replay from retained source | Idempotency and single writer |
| Workspace loss | Terraform + Bundles + governed metadata | Permissions, jobs and policies |
| Region outage | Alternate region with accessible data and keys | End-to-end SLO and consumer access |
| Ransomware | Isolated immutable recovery point | Clean environment and security validation |

## 28. Scenario A — Accidental table overwrite

**Detect:** data count or checksum diverges after a write.  
**Respond:** stop affected writers, identify valid Delta version and downstream effects.  
**Accept:** approved restore/replay and consumer correctness.

## 29. Scenario B — VACUUM removed needed history

**Detect:** time travel fails for target version.  
**Respond:** use verified independent backup or authoritative source replay.  
**Accept:** data reconstructed within business RPO/RTO, gaps documented.

## 30. Scenario C — Checkpoint corruption

**Detect:** streaming restart fails or duplicates appear.  
**Respond:** fence writers, inspect source retention and approved replay.  
**Accept:** exactly intended output, freshness and single-writer ownership.

## 31. Scenario D — Workspace configuration loss

**Detect:** jobs/policies/grants missing or altered.  
**Respond:** recover from versioned IaC, bundles and governance inventory.  
**Accept:** workload identity, schedules and permissions verified.

## 32. Scenario E — Region-wide compute outage

**Detect:** cluster provisioning unavailable across the region.  
**Respond:** initiate approved alternate-region recovery.  
**Accept:** data available, consumers routed, RPO/RTO measured.

## 33. Scenario F — Backup encrypted with unavailable key

**Detect:** backup exists but restore access denied.  
**Respond:** activate approved KMS/key recovery and security escalation.  
**Accept:** decryption and end-to-end restore test succeed.

## 34. Scenario G — Replication propagated deletion

**Detect:** source and replica both missing objects.  
**Respond:** restore from isolated immutable backup or retained versions.  
**Accept:** independent recovery point verified.

## 35. Scenario H — Terraform state unavailable

**Detect:** backend state missing, corrupted or inaccessible.  
**Respond:** restore state from protected backend history under change control; reconcile live resources.  
**Accept:** clean reviewed plan without unintended replacement.

## 36. Scenario I — Unity Catalog grants not restored

**Detect:** restored data exists but workload principals fail.  
**Respond:** reconstruct approved minimum grants and storage credential access.  
**Accept:** positive and negative authorization tests.

## 37. Scenario J — Failback causes double writes

**Detect:** two regions process the same source concurrently.  
**Respond:** fence one writer, reconcile checkpoints and outputs.  
**Accept:** deterministic data integrity and documented active owner.

## 38. Backup operational runbook

- Confirm policy, owner, recovery tier and target.
- Check last successful backup and recoverability watermark.
- Validate independent access and immutable retention.
- Verify encryption keys and cross-account/cross-region permissions.
- Monitor backup errors, replication lag and storage capacity.
- Run scheduled isolated restore validation.
- Record checksum/invariant results, RPO and RTO.
- Escalate missed backup objectives immediately.

## 39. Disaster declaration runbook

~~~markdown
# Databricks Disaster Declaration
- Incident ID / severity / commander:
- Business services and consumers:
- Failure onset UTC:
- Primary workspace / account / region:
- Approved alternate environment:
- Last validated recovery point UTC:
- Target RPO / RTO:
- Estimated data loss and service impact:
- Writer fencing owner:
- Cloud identity / KMS / storage readiness:
- Restore sequence and owners:
- Consumer acceptance checks:
- Actual RPO / RTO:
- Failback authority and criteria:
~~~

## 40. DR drill evidence package

    dr-drill-YYYYMMDD/
      01-scope-and-rpo-rto.md
      02-asset-dependency-inventory.md
      03-backup-and-replication-health.md
      04-identity-network-and-key-validation.md
      05-recovery-point-selection.md
      06-restoration-timeline.md
      07-data-integrity-validation.md
      08-workload-and-consumer-acceptance.md
      09-security-and-governance-checks.md
      10-measured-rpo-rto.md
      11-failback-plan.md
      12-lessons-and-action-items.md

Never store tokens, raw Terraform state or sensitive source data in public Git.

## 41. Acceptance scorecard

| Gate | Required evidence |
|---|---|
| Business tier | Owner-approved RPO/RTO |
| Dependency inventory | Workspace, data, cloud and identity |
| Independent backup | Recoverable point and access isolation |
| Storage/key readiness | Verified in recovery environment |
| IaC and bundle recovery | Immutable revisions available |
| Unity Catalog | Approved metadata and privileges |
| Writer control | Fencing and replay plan |
| Data integrity | Counts, uniqueness, schema and checksum |
| Freshness | Consumer deadline validated |
| Monitoring | Alerts and telemetry in recovery region |
| Actual RPO/RTO | Measured from drill timestamps |
| Failback | Controlled, tested and owned |
| Evidence | Restricted archive and sign-off |

## 42. GO / CONDITIONAL GO / NO-GO

**GO:** Recoverable backup, independent key access, tested recovery procedure, validated consumer data and measured RPO/RTO meet approved targets.

**CONDITIONAL GO:** Noncritical deviations have signed time-bound exceptions and compensating controls.

**NO-GO:** No verified recovery point, missing keys, uncontrolled dual writers, failed data integrity, untested critical restore path or unacceptable RPO/RTO.

## 43. Review questions

1. Why is Delta time travel not a replacement for an independent backup?
2. How are realized RPO and RTO measured?
3. Why can cross-region replication propagate corruption?
4. What happens when a backup exists but its KMS key is unavailable?
5. Why must checkpoints and source retention be evaluated together?
6. How does writer fencing prevent split-brain output?
7. Why does a restored workspace not necessarily mean service recovery?
8. What evidence is required to close a DR drill?

## 44. Official references

- Databricks Delta history: https://docs.databricks.com/aws/en/delta/history
- Databricks Delta clone: https://docs.databricks.com/aws/en/delta/clone
- Databricks Terraform: https://docs.databricks.com/aws/en/dev-tools/terraform/
- Databricks Bundles: https://docs.databricks.com/aws/en/dev-tools/bundles/
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- AWS S3 versioning: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
- AWS S3 Object Lock: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html
- Azure Blob data protection: https://learn.microsoft.com/azure/storage/blobs/data-protection-overview
- Google Cloud Storage object versioning: https://cloud.google.com/storage/docs/object-versioning
- Terraform state: https://developer.hashicorp.com/terraform/language/state

**Next:** Chapter 056 — Databricks Cross-Region Failover, Failback and Resilience Testing.
