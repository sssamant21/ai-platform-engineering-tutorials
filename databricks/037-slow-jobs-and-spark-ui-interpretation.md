# Chapter 037 — Slow Jobs and Spark UI Interpretation

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 08 — Performance and Troubleshooting  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform and Performance Engineering  
**Prerequisites:** Chapters 031–036  
**Safety:** Production diagnosis is read-only. Query changes, load generation and tuning experiments require an approved staging workspace.

## 1. Mission

Diagnose slow Databricks jobs systematically, using workflow timelines, Spark UI, SQL query profiles, executor statistics and cloud telemetry. Distinguish startup delay, task skew, expensive joins, shuffle, spill, network and object storage waits. Recommend only changes supported by before-and-after evidence.

### Learning outcomes

- Reconstruct an end-to-end slow-run timeline.
- Identify critical-path Spark jobs, stages and tasks.
- Interpret SQL physical plans, scans, exchanges and joins.
- Diagnose task skew, stragglers, GC, spill and shuffle.
- Differentiate compute-bound from I/O-bound execution.
- Correlate AWS, Azure and GCP metrics.
- Benchmark approved staging optimizations and verify data correctness.

## 2. Execution architecture

    Scheduled workflow / SQL request
                    |
      Queue and cluster provisioning
                    |
            Driver planning
                    |
        Spark job and stage DAG
                    |
         Task execution waves
          /       |        \
       CPU/GC   Shuffle   Storage reads
          \       |        /
            Output and commit
                    |
         Validated result + SLO

End-to-end elapsed time is not identical to Spark task execution time. Do not optimize a scan when the dominant delay is provisioning.

## 3. Triage by elapsed-time phase

| Phase | Primary evidence | Typical cause |
|---|---|---|
| Waiting/queue | Jobs run timeline | Concurrency/admission |
| Provisioning | Cluster events | Quota, cloud capacity, init |
| Planning | Driver logs, SQL plan | Metadata, complex query |
| Execution | Spark UI Jobs/Stages | CPU, skew, shuffle, I/O |
| Commit | Delta history, cloud metrics | Transaction contention, storage |
| Validation | Data quality checks | Downstream correctness/freshness |

Capture UTC boundaries and compare with the last known healthy run.

## 4. Spark UI navigation order

1. **Jobs:** find the job consuming most wall-clock time.
2. **Stages:** identify the longest stage and stage retries.
3. **Tasks:** inspect median, p95, maximum and input distribution.
4. **Executors:** inspect lost executors, task distribution, GC and spill.
5. **SQL/DataFrame:** inspect the physical plan and expensive operators when available.
6. **Storage/Environment:** inspect cache, configuration and runtime.
7. **Cloud metrics:** correlate network, disk, storage and quota events.

Spark UI details and retention depend on runtime, compute mode and permissions.

## 5. Reading stage metrics

| Metric | Interpretation | Caution |
|---|---|---|
| Task duration | Per-task elapsed time | Includes waits |
| Executor CPU time | Actual CPU work | Compare with wall time |
| GC time | JVM collection overhead | Driver vs executor differs |
| Input bytes/records | Scan work | Does not prove correctness |
| Shuffle read/write | Network/data exchange | Can be expected for joins |
| Fetch wait | Remote shuffle wait | May reflect network/executor loss |
| Memory/disk spill | Execution pressure | Not automatically a failure |
| Failed tasks | Retry/reliability risk | Identify first root error |
| Scheduler delay | Dispatch overhead | Many tiny tasks can dominate |

Investigate distributions rather than cluster-wide averages.

## 6. Task skew and stragglers

A skewed stage often shows a handful of tasks running much longer than peers with disproportionate input or shuffle bytes. Other causes of stragglers include unhealthy hosts, GC, retries and slow remote storage.

### Evidence checklist

- Is max task duration far above median?
- Are input bytes or records concentrated?
- Is shuffle read concentrated?
- Did the slow task run on a node with cloud resource errors?
- Are other tasks idle waiting for one partition?
- Did the data distribution change since the baseline?

Do not assume every long-tail task is a skewed join key.

## 7. Joins and physical plans

Common operators to recognize:
- FileScan / BatchScan: input reading and pruning.
- Filter and Project: filtering and column selection.
- Exchange: repartition/shuffle boundary.
- SortMergeJoin: sort and shuffle characteristics.
- BroadcastHashJoin: broadcast-side memory and transfer.
- HashAggregate: aggregation and possible spill.
- AdaptiveSparkPlan: runtime plan adaptation when enabled.

A broadcast join is not always optimal. Large broadcast inputs can pressure driver/executor memory; query optimization requires representative data and correct statistics.

## 8. SQL profile versus Spark UI

Use SQL query profile when available to identify operator-level time, bytes, pruning, join behavior and scan cost. Use Spark UI for task/stage/executor diagnostics. Some SQL warehouse execution modes expose different profiling surfaces than classic Spark clusters.

Correlate by query/job/run ID and UTC, not by matching approximate duration alone.

## 9. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm workspace host and principal before collecting any evidence.

## 10. Lab B — Jobs and compute inventory

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks jobs get-run --help
databricks clusters events --help
~~~

Use installed CLI help to inspect supported arguments. Output may be paginated or permission-filtered.

## 11. Lab C — Slow run comparison

Choose one healthy and one slow run of the same workload and comparable input.

| Evidence | Healthy | Slow | Delta |
|---|---|---|---|
| Input records/bytes | | | |
| Queue seconds | | | |
| Startup seconds | | | |
| Planning seconds | | | |
| Spark execution seconds | | | |
| Commit seconds | | | |
| Output validation | | | |
| Compute cost | | | |

Control for cache state, cluster size, runtime, data growth, concurrency and region.

## 12. Lab D — Spark stage investigation

For the slowest stage:

| Stage ID | Tasks | Median sec | Max sec | Shuffle GB | Spill GB | Failures |
|---|---|---|---|---|---|---|
| | | | | | | |

Record task input size distribution, executor host, retry count and GC. Screenshots or exported evidence must be sanitized.

## 13. Lab E — Offline task-tail analysis

~~~powershell
$taskSeconds = @(11,12,11,13,12,14,13,132)
$sorted = @($taskSeconds | Sort-Object)
$median = ($sorted[3] + $sorted[4]) / 2
$maximum = ($sorted | Measure-Object -Maximum).Maximum
[pscustomobject]@{
  Tasks = $taskSeconds.Count
  MedianSeconds = $median
  MaximumSeconds = $maximum
  MaxMedianRatio = [math]::Round($maximum / $median, 2)
}
~~~

This synthetic example shows a severe straggler. Confirm skew with input/shuffle distributions before changing partitioning.

## 14. Lab F — Read-only SQL plan inspection

Run against an approved staging table:

~~~sql
EXPLAIN FORMATTED
SELECT category, COUNT(*) AS total
FROM demo_catalog.demo_staging.events
WHERE event_date >= DATE '2026-10-01'
GROUP BY category;
~~~

The table is illustrative; use an existing authorized synthetic table. Inspect scans, filters, exchanges and aggregation. EXPLAIN does not measure runtime latency.

## 15. Lab G — Synthetic data and query benchmark

Create only in an approved disposable staging schema:

~~~sql
CREATE TABLE IF NOT EXISTS demo_catalog.demo_staging.spark_ui_lab
USING DELTA
AS SELECT
  id,
  CASE WHEN id % 10 = 0 THEN 'hot' ELSE 'normal' END AS category,
  CAST(id % 30 AS INT) AS bucket_id
FROM range(100000);

SELECT category, COUNT(*) AS total
FROM demo_catalog.demo_staging.spark_ui_lab
GROUP BY category;
~~~

If rerunning, confirm table contents rather than assuming CREATE TABLE changed an existing table. Compare Spark UI stage/task metrics. The small dataset may not produce measurable shuffle or spill; do not force excessive load to manufacture a bottleneck.

## 16. Lab H — Compare query plans

~~~sql
EXPLAIN FORMATTED
SELECT bucket_id, COUNT(*) AS total
FROM demo_catalog.demo_staging.spark_ui_lab
GROUP BY bucket_id;

EXPLAIN FORMATTED
SELECT category, COUNT(*) AS total
FROM demo_catalog.demo_staging.spark_ui_lab
GROUP BY category;
~~~

Review physical plan operators, adaptive behavior and partitioning. Do not infer that one plan is faster solely from its operator count.

## 17. Lab I — Shuffle and spill worksheet

| Stage | Shuffle read | Shuffle write | Fetch wait | Memory spill | Disk spill |
|---|---|---|---|---|---|
| | | | | | |

Correlate elevated spill with GC, local disk latency and executor losses. Spilling alone can be normal.

## 18. Lab J — Offline wall-time decomposition

~~~powershell
$queue = 75
$startup = 180
$planning = 35
$execution = 510
$commit = 40
$total = $queue + $startup + $planning + $execution + $commit

[pscustomobject]@{
  TotalSeconds = $total
  ExecutionPct = [math]::Round(100*$execution/$total, 1)
  NonExecutionSeconds = $total-$execution
}
~~~

This synthetic model demonstrates why a Spark-only optimization may have limited impact on total run time.

## 19. Lab K — Cloud correlation

~~~powershell
# AWS
aws sts get-caller-identity
aws configure get region

# Azure
az account show --query "{subscription:id,tenant:tenantId}" -o json

# GCP
gcloud config get-value project
~~~

Review only relevant provider telemetry:

| Databricks symptom | AWS | Azure | GCP |
|---|---|---|---|
| Slow provisioning | EC2 quotas/capacity | VM allocation/quota | Compute Engine quota/capacity |
| Disk spill latency | EBS/instance storage | Managed/temp disk | Persistent/local disk |
| Slow reads | S3 request metrics | ADLS/Blob metrics | Cloud Storage metrics |
| Network delays | VPC/network metrics | VNet/NSG/network | VPC/network metrics |
| Executor loss | EC2 events | VM events | GCE instance events |

Cloud console identity and Databricks runtime identity can differ.

## 20. Lab L — Controlled optimization experiment

Preconditions: synthetic staging data, fixed compute, cost cap and correctness baseline.

1. Capture three baseline runs with documented cache conditions.
2. Change one approved variable (query plan, layout, concurrency or compute).
3. Repeat three runs under comparable input and load.
4. Compare p50/p95 total time and stage/task distributions.
5. Verify row counts, uniqueness and business invariants.
6. Compare compute and storage request costs.
7. Restore the original configuration if the result regresses.

Do not simultaneously resize compute, rewrite data and change SQL; that prevents useful attribution.

## 21. Incident A — Long-tail stage caused by key skew

**Symptoms:** most tasks finish quickly while one task dominates stage completion.

**Triage:** inspect task input, shuffle and key distribution.

**Mitigation:** approved query/data-layout changes, adaptive execution or skew-specific approach.

**Acceptance:** lower tail latency with identical validated output.

## 22. Incident B — Shuffle fetch failures

**Symptoms:** failed/retried stages, fetch exceptions, executor loss.

**Triage:** network, executor health, local disk, cloud events and shuffle metrics.

**Mitigation:** restore failed infrastructure or adjust workload through staging-tested change.

**Acceptance:** no repeated fetch failures and acceptable deadline compliance.

## 23. Incident C — Executor GC and memory pressure

**Symptoms:** high GC, spills, OOM and unstable task durations.

**Triage:** per-executor heap/off-heap, task concurrency, joins and skew.

**Mitigation:** approved memory or workload adjustment; contain retries.

**Acceptance:** stable executors and correct results at acceptable cost.

## 24. Incident D — Slow file scan from small files

**Symptoms:** high file-open/task overhead and slow planning relative to data volume.

**Triage:** Delta file count, file sizes, partition pruning and storage request rates.

**Mitigation:** staging-tested file layout and ingestion tuning.

**Acceptance:** faster end-to-end scan without new correctness or cost issues.

## 25. Incident E — Slow query due to cloud object storage

**Symptoms:** low executor CPU but elevated read wait and storage latency.

**Triage:** provider request metrics, throttling, networking, IAM and data locality.

**Mitigation:** fix cloud dependency; avoid indiscriminate CPU scale-out.

**Acceptance:** read latency and job SLO normalize.

## 26. Incident F — Slow startup mistaken for Spark regression

**Symptoms:** total job duration increases while Spark stages are unchanged.

**Triage:** queueing, cluster events, init scripts, library install, quotas and instance availability.

**Mitigation:** correct provisioning or scheduling issue.

**Acceptance:** startup time returns to baseline.

## 27. Incident G — Broadcast join memory regression

**Symptoms:** query slows or executors fail after plan change.

**Triage:** physical plan, broadcast-side size, memory pressure and stats.

**Mitigation:** test alternate join strategy or improve statistics in staging.

**Acceptance:** stable memory and equivalent query output.

## 28. Incident H — Adaptive plan changes unexpectedly

**Symptoms:** plan and task distribution differ after data/runtime change.

**Triage:** adaptive plan, updated statistics, data skew and runtime configuration.

**Mitigation:** validate plan-specific change in staging; avoid blanket disabling of adaptive optimization.

**Acceptance:** repeatable performance and unchanged correctness.

## 29. Troubleshooting decision tree

    Job exceeded expected duration
                 |
       Was it queued or provisioning?
         yes -> scheduler/cloud quota
         no
                 |
       Is planning the critical path?
         yes -> metadata/query plan
         no
                 |
       Is one stage dominant?
         no -> multiple dependencies
         yes
                 |
       Few tasks dominate?
         yes -> skew/straggler/host
         no
                 |
       GC, spill or shuffle high?
         yes -> memory/network/layout
         no
                 |
       I/O wait or storage errors?
         yes -> cloud storage/network
         no -> CPU/parallelism/plan review

## 30. Production slow-job runbook

1. Confirm business deadline, scope and impact.
2. Capture run IDs, UTC timeline and last healthy run.
3. Compare input size, compute, runtime and concurrent load.
4. Separate queue/startup/planning/execution/commit time.
5. Inspect Spark UI job, stage, task and executor metrics.
6. Review SQL physical plan and query profile if available.
7. Correlate cloud quotas, disk, network and storage telemetry.
8. Form an evidence-backed hypothesis and stage one change.
9. Verify correctness, latency, retry rate and cost.
10. Document RCA, rollback and preventive monitoring.

## 31. Dashboard and alerting

Monitor:
- End-to-end job p50/p95 and deadline compliance.
- Queue, startup, planning and execution durations.
- Slowest stage duration and task max-to-median ratio.
- Executor losses, task failures, GC and spill.
- Shuffle bytes and fetch failures.
- Cloud storage latency/throttling and network errors.
- Cost per validated successful run.

Page for sustained service impact or credible missed deadlines; avoid alerting on an isolated slow task without business impact.

## 32. Acceptance checklist

- [ ] Healthy and slow runs compared.
- [ ] End-to-end phases decomposed.
- [ ] Spark Jobs, Stages, Tasks and Executors inspected.
- [ ] Task skew and straggler evidence distinguished.
- [ ] SQL physical plan reviewed.
- [ ] Shuffle, spill and GC interpreted.
- [ ] Cloud infrastructure metrics correlated.
- [ ] Controlled staging benchmark completed.
- [ ] Output correctness and cost validated.
- [ ] Eight incident scenarios reviewed.
- [ ] Production runbook and rollback documented.
- [ ] No unapproved production load test performed.

## 33. Review questions

1. Why can a Spark stage with high shuffle volume still be healthy?
2. What proves that a straggler is caused by data skew?
3. Why must queue/startup time be separated from execution?
4. How can SQL query profile and Spark UI complement each other?
5. What evidence is required before recommending larger workers?

## 34. Official references

- Apache Spark monitoring: https://spark.apache.org/docs/latest/monitoring.html
- Apache Spark SQL performance tuning: https://spark.apache.org/docs/latest/sql-performance-tuning.html
- Databricks compute monitoring: https://docs.databricks.com/aws/en/compute/cluster-metrics
- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks Delta Lake: https://docs.databricks.com/aws/en/delta/
- AWS CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs

**Next:** Chapter 038 — Network, DNS and Storage Bottlenecks.
