# Chapter 029 — Storage Performance, Data Lifecycle and Cost Optimization Operations

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 06 — Storage, Data Access and Infrastructure Reliability  
**Clouds:** AWS, Azure, GCP  
**Prerequisites:** Chapters 026–028  
**Audience:** SRE, DBRE, Cloud Platform, FinOps  
**Safety:** Production inspection is read-only; optimization experiments require approved staging resources.

## 1. Mission

Operate storage as a reliability and cost-critical component of Databricks. Establish performance baselines, identify small-file and metadata overhead, understand lifecycle and retention constraints, and evaluate optimization using **cost per validated workload**, not simply lower object counts.

### Learning outcomes

- Diagnose latency from compute through object storage.
- Distinguish request, transfer, metadata and transaction-log bottlenecks.
- Measure Delta file count, table growth and operation history.
- Assess OPTIMIZE, auto compaction and predictive optimization applicability.
- Evaluate lifecycle and VACUUM without compromising recoverability.
- Forecast storage and request cost on AWS, Azure and GCP.
- Execute controlled staging performance and cost acceptance tests.

## 2. Architecture and performance path

    Databricks job / SQL warehouse
                 |
          Query or Spark plan
                 |
      Task scheduling / executors
                 |
       Delta metadata / snapshots
                 |
         Object read and writes
                 |
        DNS / TLS / network path
                 |
      S3 / ADLS Gen2 / Cloud Storage
                 |
       Requests + bytes + retention
                 |
          Performance and cost

A slow job is not automatically a slow storage service. Compare compute saturation, query planning, file layout, cloud service metrics and network telemetry.

## 3. Storage bottleneck taxonomy

| Symptom | Candidate cause | Evidence |
|---|---|---|
| High file-open overhead | Excess small files | File count, scan task duration |
| Slow listing/metadata | Metadata or log overhead | Planning time, Delta history |
| High read latency | Network/storage throughput | Cloud metrics, task I/O |
| High write latency | Commit conflicts or storage throttling | Transaction errors, provider logs |
| Intermittent failures | Throttling, IAM, DNS, TLS | Correlated errors and changes |
| Growing cost | Data volume, versions, requests | Billing and storage inventory |
| Slow restore | Retention and object availability | History, backup, lifecycle |

## 4. Delta file layout

Small files increase object operations and metadata/planning overhead. Large files can reduce parallelism for some workloads. File sizing is workload-dependent.

Relevant controls include:
- Optimized writes and auto compaction where supported.
- OPTIMIZE and appropriate layout strategies.
- Predictive optimization for eligible managed tables.
- Partitioning choices and clustering where supported.
- Table statistics and query access patterns.

Do not blindly apply legacy partitioning rules or assume one file size is ideal for every table.

## 5. Managed versus external table responsibilities

Managed tables may benefit from platform-managed optimization and lifecycle features, subject to eligibility and configuration. External tables may require explicit coordination with storage owners and independent lifecycle policies.

Before any cleanup, identify:
- Table type and ownership.
- Data path and cloud account.
- Readers, writers, streams and downstream dependencies.
- Transaction log and deleted-file retention.
- Time-travel, backup, RPO and compliance requirements.
- Cloud lifecycle and object versioning policies.

Cloud object lifecycle rules can break Delta tables if they remove active data or log objects. Never apply generic object expiration to a live Delta table path without a validated design.

## 6. Cloud cost dimensions

| Cost component | AWS | Azure | GCP |
|---|---|---|---|
| Stored bytes | S3 storage classes | Blob access tiers | GCS storage classes |
| Requests | PUT/GET/LIST and other requests | Read/write/list transactions | Operation classes |
| Retrieval | Storage-class dependent | Tier dependent | Storage-class dependent |
| Network | Inter-region/egress | Inter-region/egress | Inter-region/egress |
| Protection | Versioning, replication, KMS | Versioning, redundancy, encryption | Versioning, replication, KMS |
| Databricks compute | Job/SQL compute | Job/SQL compute | Job/SQL compute |

Provider pricing varies by region, class and current terms. Use current billing exports and calculators rather than fixed tutorial prices.

## 7. Lab A — Workspace and identity preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Verify workspace host, environment and authorization. Do not record credentials in shared evidence.

## 8. Lab B — Read-only Delta storage inspection

Use an approved table:

~~~sql
DESCRIBE DETAIL demo_catalog.demo_schema.demo_delta_table;
DESCRIBE HISTORY demo_catalog.demo_schema.demo_delta_table LIMIT 20;
SHOW TBLPROPERTIES demo_catalog.demo_schema.demo_delta_table;
~~~

Record table size, file count, latest operations, relevant table features, retention settings and location sensitivity.

### Baseline worksheet

| Metric | Baseline | Target |
|---|---|---|
| Total table bytes | | |
| Active file count | | |
| Mean active file bytes | | |
| Query p95 duration | | |
| Read bytes / request count | | |
| Write/commit failure rate | | |
| Storage throttling | | |
| Monthly cost | | |

The average active file size is only an indicator, not a sufficient optimization decision.

## 9. Lab C — Offline small-file calculation

~~~powershell
$tableBytes = 250GB
$activeFiles = 20000
$meanFileMB = ($tableBytes / $activeFiles) / 1MB
[pscustomobject]@{
  TableGB = [math]::Round($tableBytes / 1GB, 2)
  ActiveFiles = $activeFiles
  MeanFileMB = [math]::Round($meanFileMB, 2)
}
~~~

Synthetic example: approximately 12.8 MiB per file. Review query patterns, partitioning and actual task metrics before recommending compaction.

## 10. Lab D — Inspect optimization commands without changing production

~~~sql
-- Read-only history review:
DESCRIBE HISTORY demo_catalog.demo_schema.demo_delta_table;
~~~

Potential state-changing operations include OPTIMIZE, VACUUM, ALTER TABLE and changes to clustering/partitioning. Execute them **only** on approved disposable staging tables after checking feature support, concurrency, compute cost and rollback options.

## 11. Lab E — Controlled staging optimization

Preconditions: synthetic Delta table, approved cost cap, baseline workload, no external readers, and an identified rollback owner.

1. Generate a small synthetic dataset using an approved staging notebook or SQL warehouse.
2. Create a deliberately fragmented **disposable** Delta table.
3. Capture file count, size, query time, task metrics and storage requests.
4. Run a supported OPTIMIZE operation on that staging table.
5. Re-run identical queries with comparable compute and cache conditions.
6. Compare p50/p95 duration, file count, bytes scanned and total cost.
7. Inspect Delta history and validate record counts and key uniqueness.
8. Retain evidence and clean up only the approved disposable resources.

An OPTIMIZE operation rewrites files and consumes compute; it does not guarantee performance gains for every workload.

## 12. Lab F — Retention and lifecycle audit

~~~sql
SHOW TBLPROPERTIES demo_catalog.demo_schema.demo_delta_table;
DESCRIBE HISTORY demo_catalog.demo_schema.demo_delta_table LIMIT 20;
~~~

Review:
- Deleted-file retention and log retention.
- Streaming reader lag and long-running transactions.
- Time-travel and restore objectives.
- Independent backup and object versioning.
- Cloud lifecycle rules affecting the table prefix.
- Planned VACUUM frequency and maintenance windows.

Do not shorten retention, bypass retention checks or run VACUUM in production as part of this exercise.

## 13. Lab G — AWS S3 inventory and metrics context

~~~powershell
aws sts get-caller-identity
aws configure get region
aws s3api get-bucket-location --bucket demo-staging-data
~~~

Use authorized resources only. For production capacity review, correlate S3 storage metrics, request metrics where enabled, billing exports, network egress and CloudTrail evidence. Bucket location checks do not measure performance.

## 14. Lab H — Azure ADLS Gen2 inventory context

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az storage account show --name demostagingstore --resource-group demo-staging-rg --query "{name:name,location:location,kind:kind}" -o json
~~~

Review Azure Monitor storage latency, transaction counts, throttling, access tier, redundancy, network egress and cost exports. Verify hierarchical namespace and private endpoint configuration where relevant.

## 15. Lab I — GCP Cloud Storage inventory context

~~~powershell
gcloud config get-value project
gcloud storage buckets describe gs://demo-staging-data --format=json
~~~

Review Cloud Monitoring storage metrics, request costs, retrieval/early deletion charges where applicable, object versions and cross-region egress.

## 16. Lab J — Offline cost comparison

The following is a **synthetic model**, not a provider quote:

~~~powershell
$before = [pscustomobject]@{
  StorageMonthly = 900
  RequestsMonthly = 240
  ComputeMonthly = 600
}
$after = [pscustomobject]@{
  StorageMonthly = 920
  RequestsMonthly = 140
  ComputeMonthly = 580
}
$beforeTotal = $before.StorageMonthly + $before.RequestsMonthly + $before.ComputeMonthly
$afterTotal = $after.StorageMonthly + $after.RequestsMonthly + $after.ComputeMonthly
[pscustomobject]@{
  BeforeUSD = $beforeTotal
  AfterUSD = $afterTotal
  MonthlyDeltaUSD = $afterTotal - $beforeTotal
  MonthlySavingsUSD = $beforeTotal - $afterTotal
}
~~~

Include the one-time cost of optimization and ongoing maintenance when calculating payback. Savings without stable SLOs and correct output are not an acceptable optimization.

## 17. Forecasting storage growth

Track logical data growth separately from physical storage growth. Physical bytes may include historical versions, rewritten files, retained deleted files and object versions.

Forecast:
- Daily net new logical data.
- Delta rewrite/compaction amplification.
- Retention and VACUUM effects.
- Object versioning and replication.
- Backfill and seasonal peaks.
- Backup copies and disaster recovery regions.
- Query/request growth and network egress.

Build baseline, expected and stress scenarios, with explicit uncertainty.

## 18. Incident A — Small-file explosion

**Symptoms:** planning time and object requests grow despite stable logical data volume.

**Triage:** inspect file count, ingestion frequency, write layout and recent changes.

**Recovery:** approved write-path tuning and staging-tested compaction.

**Acceptance:** improved end-to-end performance and validated data at acceptable cost.

## 19. Incident B — OPTIMIZE causes unexpected compute spend

**Symptoms:** maintenance cost spikes and concurrent workloads slow.

**Triage:** inspect optimization schedule, table size, compute, concurrency and eligibility for automated optimization.

**Recovery:** pause unnecessary maintenance, reschedule and adjust scope.

**Acceptance:** workload SLO restored and cost model updated.

## 20. Incident C — Cloud lifecycle deletes active Delta files

**Symptoms:** file-not-found errors and table snapshot failures after lifecycle change.

**Triage:** preserve audit evidence, identify deleted objects and active table references.

**Recovery:** stop destructive lifecycle policy, assess cloud versioning/backups and use supported recovery guidance.

**Acceptance:** table integrity restored; policy excludes unsafe table paths.

## 21. Incident D — Storage request throttling

**Symptoms:** intermittent timeouts/429/5xx and elevated job duration.

**Triage:** correlate request rates, concurrency, object layout, network and provider metrics.

**Recovery:** bounded retries, workload pacing and approved architecture changes.

**Acceptance:** throttling and retry storm normalize without duplicate output.

## 22. Incident E — Unexpected storage bill

**Symptoms:** billed bytes or operations grow faster than logical table data.

**Triage:** inspect object versions, retention, replicas, deleted-file retention, backfills and cross-region egress.

**Recovery:** correct approved lifecycle and workload design without sacrificing RPO/RTO.

**Acceptance:** explained cost variance and validated retention.

## 23. Incident F — Performance degrades after file compaction

**Symptoms:** fewer files but slower selective queries or reduced parallelism.

**Triage:** compare query plans, data skipping, clustering, scan parallelism and cache conditions.

**Recovery:** adjust layout using controlled experiments; do not assume reversing file count alone fixes performance.

**Acceptance:** target query classes improve and other workloads do not regress.

## 24. Troubleshooting decision tree

    Storage-related latency/cost anomaly
                |
      Cloud service errors elevated?
        yes -> throttling / IAM / network
        no
                |
      Metadata or file-open overhead high?
        yes -> file layout / Delta log / planning
        no
                |
      Bytes scanned unexpectedly high?
        yes -> pruning / layout / query pattern
        no
                |
      Retained physical bytes growing?
        yes -> versions / VACUUM / lifecycle
        no
                |
      Compute or downstream bottleneck?
        yes -> task profile / shared dependency
        no -> deeper workload investigation

## 25. Production runbook

1. Confirm impacted jobs, tables, deadlines and cloud regions.
2. Capture UTC timeline, baselines and recent configuration changes.
3. Separate compute, metadata, object storage, network and dependency time.
4. Check cloud throttling, error rates and object request costs.
5. Inspect Delta file count, history and retention.
6. Select least-risk change and rehearse in staging.
7. Apply only approved production changes with rollback and cost cap.
8. Validate data correctness, performance and billing.
9. Document RCA and preventive monitoring.

## 26. Monitoring and SLOs

Recommended signals:
- p95 end-to-end query/job latency by workload class.
- Delta active file count and bytes.
- Cloud object request count and error rate.
- Storage read/write latency and throughput.
- Logical versus physical storage growth.
- Retained historical data age.
- Cost per validated job or query.
- VACUUM/optimization execution and failure.
- Backup and recovery rehearsal success.

Set alert thresholds using baseline and business criticality.

## 27. Change and rollback controls

Before optimization or lifecycle changes:
- Identify managed/external ownership and active readers.
- Capture current configuration and data version.
- Verify restore and backup options.
- Review streaming and time-travel retention.
- Define SLO and cost rollback thresholds.
- Test with representative staging data.
- Document that file deletion may be irreversible without backup.

## 28. Acceptance checklist

- [ ] Storage performance path documented.
- [ ] File count, table size and workload latency baselined.
- [ ] Cloud request, storage and network costs inventoried.
- [ ] Retention and VACUUM risk reviewed.
- [ ] Cloud lifecycle policies checked for Delta safety.
- [ ] Staging optimization benchmark completed.
- [ ] Output correctness verified before and after optimization.
- [ ] Cost model includes one-time and recurring expenses.
- [ ] Growth forecast includes retention and recovery scenarios.
- [ ] Six incident scenarios reviewed.
- [ ] No unapproved production optimization or deletion performed.

## 29. Review questions

1. Why can fewer Delta files sometimes reduce query performance?
2. Why is logical table size different from physical cloud storage consumption?
3. How can object lifecycle policies undermine Delta recovery?
4. What should be included in the payback calculation for OPTIMIZE?
5. Why must storage cost changes be evaluated against business SLOs?

## 30. Official references

- Databricks Delta Lake: https://docs.databricks.com/aws/en/delta/
- Databricks OPTIMIZE: https://docs.databricks.com/aws/en/sql/language-manual/delta-optimize
- Databricks VACUUM: https://docs.databricks.com/aws/en/delta/vacuum
- Databricks predictive optimization: https://docs.databricks.com/aws/en/optimizations/predictive-optimization
- AWS S3: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- Azure Storage monitoring: https://learn.microsoft.com/azure/storage/common/monitor-storage
- Google Cloud Storage: https://cloud.google.com/storage/docs

**Next:** Chapter 030 — Storage Reliability, Security and Recovery Acceptance Capstone.
