# Chapter 028 — Delta Lake Storage Reliability, Transaction Logs and Recovery Operations

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 06 — Storage, Data Access and Infrastructure Reliability  
**Clouds:** AWS, Azure, GCP  
**Audience:** SRE, DBRE, Platform Operations, Data Reliability  
**Prerequisites:** Chapters 026–027  
**Safety:** Production read-only. Recovery experiments require approved staging tables and synthetic data.

## 1. Mission

Operate Delta Lake tables reliably across cloud object storage. Understand the transaction log, checkpoints, optimistic concurrency, retention, time travel and restore boundaries so incidents can be diagnosed and recovered without introducing data loss.

### Learning objectives

- Explain Delta Lake transaction commits and snapshot reconstruction.
- Locate and interpret Delta table history safely.
- Distinguish metadata, object-storage and data correctness failures.
- Assess retention and VACUUM risk before recovery.
- Diagnose concurrent writes and partial operation symptoms.
- Perform controlled staging restore and verify row-level correctness.
- Create incident, backup and rollback runbooks.

## 2. Reliability architecture

    Databricks job / SQL warehouse
                |
          Delta transaction
                |
        Optimistic concurrency
                |
        _delta_log commit records
                |
         Checkpoints and snapshots
                |
        Parquet data files in cloud
           /       |        \
          S3     ADLS Gen2    GCS
                |
       Unity Catalog governance
                |
        Monitoring and recovery

A successful transaction commit makes a new logical table version visible; it does not eliminate the need for backups, independent correctness checks, retention planning or storage disaster recovery.

## 3. Delta transaction log fundamentals

Delta tables maintain transaction metadata under a table's transaction log directory. Commits describe actions such as adding/removing files, metadata changes and protocol information. Checkpoints help reconstruct snapshots efficiently.

**Operational cautions:**
- Do not manually edit, delete or rename transaction log files.
- Do not assume all historical data files remain available indefinitely.
- Never use raw cloud object operations as a substitute for supported Delta table management.
- Understand that catalog metadata and physical cloud objects have different lifecycles.
- Verify supported Delta protocol/features before moving data between runtimes.

## 4. ACID and concurrent writers

| Property | SRE interpretation |
|---|---|
| Atomicity | A committed Delta transaction is logically applied as a unit |
| Consistency | Transaction rules do not replace business data validation |
| Isolation | Concurrent writes follow supported conflict detection semantics |
| Durability | Committed data depends on the durability and accessibility of cloud storage |

Delta Lake commonly uses optimistic concurrency control. Conflicting operations may fail and require application-level re-evaluation. Blindly retrying a non-idempotent append can duplicate data.

## 5. Table reliability failure domains

| Symptom | Possible cause | First evidence |
|---|---|---|
| Concurrent modification error | Competing writers or metadata change | Job overlap, table history |
| Missing version/time travel | Retention or unavailable files/logs | History, retention settings |
| Storage 403 | Cloud IAM, KMS, UC or ACL | Run-as, storage credential, audit |
| Table path missing | Object lifecycle/deletion | Cloud audit, storage inventory |
| Slow snapshot loading | Log/checkpoint, metadata or storage latency | Driver/task logs, cloud metrics |
| Incorrect row counts | Source duplication, partial external effects | Business validation, history |
| Restore rejected | Unsupported table feature or retention | Runtime, protocol, object availability |

## 6. Retention and recovery boundaries

Three distinct concepts matter:

1. **Transaction log/history retention:** affects how long historical metadata remains available.
2. **Deleted data file retention:** affects whether files needed for older snapshots remain available.
3. **External backup/replication retention:** determines independent disaster recovery options.

VACUUM can remove data files no longer referenced by retained table versions. Shortening retention without analyzing readers, streaming consumers and recovery objectives can permanently eliminate time-travel options. Do not disable retention safety checks to expedite cleanup.

## 7. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list-runs --limit 25 --output json
~~~

Verify workspace host and identity. Read-only SQL exploration requires an authorized SQL warehouse or notebook session.

## 8. Lab B — Inspect a Delta table safely

Run against an approved table, replacing the illustrative identifiers:

~~~sql
DESCRIBE DETAIL demo_catalog.demo_schema.demo_delta_table;
DESCRIBE HISTORY demo_catalog.demo_schema.demo_delta_table;
SELECT COUNT(*) AS row_count
FROM demo_catalog.demo_schema.demo_delta_table;
~~~

Record table format, location (sensitive), version history, operation timestamps, size, file count, relevant table properties and current row count. Do not assume the count alone proves correctness.

### Evidence worksheet

| Field | Finding |
|---|---|
| Catalog/schema/table | |
| Table owner | |
| Cloud and region | |
| Current version | |
| Latest operation UTC | |
| Table size / file count | |
| Retention policy | |
| Active readers/writers | |
| Business validation | |
| RPO/RTO | |

## 9. Lab C — Inspect transaction history

~~~sql
DESCRIBE HISTORY demo_catalog.demo_schema.demo_delta_table LIMIT 20;
SHOW TBLPROPERTIES demo_catalog.demo_schema.demo_delta_table;
~~~

Correlate the history with jobs, change tickets, concurrent writers and cloud audit events. History output can contain operation parameters and user identifiers; sanitize evidence.

## 10. Lab D — Synthetic staging table creation

**Only in an approved staging catalog/schema** with synthetic values:

~~~sql
CREATE TABLE IF NOT EXISTS demo_catalog.demo_staging.delta_recovery_lab (
  event_id BIGINT,
  status STRING,
  batch_id STRING
) USING DELTA;

INSERT INTO demo_catalog.demo_staging.delta_recovery_lab
VALUES
  (1, 'READY', 'batch-001'),
  (2, 'READY', 'batch-001');

DESCRIBE HISTORY demo_catalog.demo_staging.delta_recovery_lab;
~~~

Use a uniquely named disposable table in a real exercise; avoid collisions with existing objects. This lab intentionally changes staging data.

## 11. Lab E — Create and inspect a synthetic change

~~~sql
UPDATE demo_catalog.demo_staging.delta_recovery_lab
SET status = 'PROCESSED'
WHERE event_id = 1;

SELECT event_id, status, batch_id
FROM demo_catalog.demo_staging.delta_recovery_lab
ORDER BY event_id;

DESCRIBE HISTORY demo_catalog.demo_staging.delta_recovery_lab;
~~~

Record the version before and after the update. Do not infer the version number from examples; use actual history.

## 12. Lab F — Staging time travel validation

Use the version observed **before** the update:

~~~sql
SELECT event_id, status, batch_id
FROM demo_catalog.demo_staging.delta_recovery_lab VERSION AS OF 1
ORDER BY event_id;
~~~

Version 1 is illustrative and may not be the correct version in your table. Replace it with the verified version from history. Time travel requires the referenced metadata and data files to remain available.

## 13. Lab G — Controlled staging restore

**Approval gate:** verify that the target version exists, that all required files are retained, and that a restore will not overwrite concurrent changes.

~~~sql
-- Replace 1 with a verified staging target version.
RESTORE TABLE demo_catalog.demo_staging.delta_recovery_lab
TO VERSION AS OF 1;
~~~

RESTORE is a state-changing operation that creates a new table version. It does not rewind history in place. Coordinate with streaming consumers and downstream jobs because restored data may be observed as new changes. Do not run this against production as part of the tutorial.

### Post-restore checks

~~~sql
DESCRIBE HISTORY demo_catalog.demo_staging.delta_recovery_lab;
SELECT event_id, status, batch_id
FROM demo_catalog.demo_staging.delta_recovery_lab
ORDER BY event_id;
~~~

Compare expected rows, unique keys and downstream state.

## 14. Lab H — Retention risk review

Read-only inspection:

~~~sql
SHOW TBLPROPERTIES demo_catalog.demo_schema.demo_delta_table;
DESCRIBE DETAIL demo_catalog.demo_schema.demo_delta_table;
~~~

Determine:
- Time travel requirements and recovery objectives.
- Streaming reader lag and long-running jobs.
- Historical versions still needed for compliance or backfill.
- Planned VACUUM and log retention behavior.
- Availability of independent backups and cloud recovery.
- Table features affecting restore or interoperability.

Do not execute VACUUM or shorten retention in production during this lab.

## 15. Lab I — Offline incident decision matrix

~~~powershell
$cases = @(
  [pscustomobject]@{ Failure="CONCURRENT_WRITE"; HistoryAvailable=$true; DataFilesAvailable=$true; Action="REVIEW_CONFLICT_AND_IDEMPOTENCY" },
  [pscustomobject]@{ Failure="BAD_UPDATE"; HistoryAvailable=$true; DataFilesAvailable=$true; Action="ASSESS_CONTROLLED_RESTORE" },
  [pscustomobject]@{ Failure="FILES_REMOVED"; HistoryAvailable=$true; DataFilesAvailable=$false; Action="BACKUP_RECOVERY_ASSESSMENT" },
  [pscustomobject]@{ Failure="STORAGE_403"; HistoryAvailable=$true; DataFilesAvailable=$true; Action="IAM_NETWORK_TRIAGE" }
)
$cases | Format-Table -AutoSize
~~~

This exercise performs no Databricks operations. A proposed action is not authorization to execute it.

## 16. Incident A — Conflicting concurrent writes

**Triage:** identify overlapping jobs, transaction types, history, isolation/conflict messages and partition scope.

**Recovery:** coordinate writers, correct scheduling or transaction strategy, then safely replay idempotent work.

**Acceptance:** correct row counts and no duplicate batch effects.

## 17. Incident B — Incorrect production update

**Triage:** preserve current version, identify affected rows and downstream consumers, check history and retained files.

**Recovery:** evaluate targeted correction versus approved RESTORE; avoid overwriting valid concurrent writes.

**Acceptance:** reconciled data and validated consumers.

## 18. Incident C — VACUUM removed needed historical files

**Triage:** inspect retention changes, VACUUM history, missing objects and backup availability.

**Recovery:** use supported independent backup/recovery sources where available; do not promise time travel can recover removed files.

**Acceptance:** restored data integrity and documented remaining historical gaps.

## 19. Incident D — Delta log path or objects deleted

**Triage:** contain further changes, preserve object inventory and audit logs, determine scope of deletion.

**Recovery:** use cloud-specific backup/versioning and supported Databricks recovery procedures; avoid manual reconstruction without vendor guidance.

**Acceptance:** snapshot consistency, query success and validated table history.

## 20. Incident E — Cloud IAM or KMS regression

**Triage:** correlate job failures with UC grants, storage credential identity, cloud IAM, KMS and private network changes.

**Recovery:** restore least-privilege access through approved change.

**Acceptance:** authorized operations succeed and negative access tests remain denied.

## 21. Incident F — Restore triggers downstream duplicates

**Triage:** inspect streaming/change data consumers, checkpoint state, restore commit and batch identifiers.

**Recovery:** pause affected consumers if approved, reconcile changes and resume with validated semantics.

**Acceptance:** no missing or duplicate downstream records.

## 22. Troubleshooting decision tree

    Delta table failure
            |
    Catalog/table metadata accessible?
       no -> UC grants / metastore / object metadata
       yes
            |
    Cloud object access healthy?
       no -> IAM / KMS / network / object lifecycle
       yes
            |
    Current snapshot readable?
       no -> log / checkpoint / data file integrity
       yes
            |
    Concurrent writer conflict?
       yes -> transaction coordination / safe retry
       no
            |
    Business data correct?
       no -> version comparison / restore vs correction
       yes -> monitor and close

## 23. Production recovery runbook

1. Declare severity and business/data correctness impact.
2. Preserve table identifier, current version, history and UTC timeline.
3. Identify active writers, readers and downstream consumers.
4. Determine whether failure is governance, cloud storage, transaction conflict or bad data.
5. Assess retention, required historical files and independent backups.
6. Select least destructive approved recovery method.
7. Coordinate writers and streaming consumers before changes.
8. Validate row counts, key uniqueness, freshness and downstream state.
9. Monitor subsequent commits, latency and cloud errors.
10. Document RCA, recovery gap, RPO/RTO and preventive controls.

## 24. Monitoring and SLIs

Track:
- Delta commit failure/conflict rates by critical workload.
- Time since last validated successful commit.
- Snapshot/read latency and cloud storage errors.
- Data freshness, completeness and duplicate indicators.
- VACUUM activity and retention configuration drift.
- Table file count, growth and storage cost.
- Backup availability and restore rehearsal success.
- Recovery time against RTO and data loss against RPO.

Define thresholds using workload baselines; avoid universal fixed values.

## 25. Change and rollback controls

Before changing table properties, retention, optimization, clustering, schema or access:
- Capture approved current configuration and latest validated version.
- Inventory readers, writers and streaming consumers.
- Review protocol and feature compatibility.
- Confirm backup/recovery options and cost.
- Rehearse changes in staging.
- Define rollback or compensating correction.
- Verify both table state and downstream business output.

RESTORE creates a new transaction; it does not necessarily undo side effects outside the table.

## 26. Operational acceptance checklist

- [ ] Delta transaction log and checkpoint roles explained.
- [ ] Managed/external table storage ownership documented.
- [ ] Table history and metadata inspection completed.
- [ ] Concurrent writer and idempotency risks assessed.
- [ ] Retention and VACUUM recovery limits reviewed.
- [ ] Synthetic staging update/time travel tested.
- [ ] Approved staging restore and integrity checks completed.
- [ ] Streaming/downstream implications evaluated.
- [ ] Six incident scenarios reviewed.
- [ ] Backup and RPO/RTO controls documented.
- [ ] No production RESTORE, VACUUM or object mutation performed.

## 27. Review questions

1. Why is a Delta transaction log essential to snapshot reconstruction?
2. Why can a successful commit still violate business data correctness?
3. When does RESTORE create risk for downstream streaming consumers?
4. Why might time travel fail even when historical metadata exists?
5. Why must cloud object deletion never be used as an ad hoc rollback?

## 28. Official references

- Delta Lake overview: https://docs.databricks.com/aws/en/delta/
- Delta history and time travel: https://docs.databricks.com/aws/en/delta/history
- Delta RESTORE: https://docs.databricks.com/aws/en/sql/language-manual/delta-restore
- Delta VACUUM: https://docs.databricks.com/aws/en/delta/vacuum
- Delta concurrency: https://docs.databricks.com/aws/en/optimizations/isolation-level
- Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/

**Next:** Chapter 029 — Storage Performance, Data Lifecycle and Cost Optimization Operations.
