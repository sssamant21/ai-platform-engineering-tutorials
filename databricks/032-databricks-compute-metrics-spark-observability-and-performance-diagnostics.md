# Chapter 032 — Databricks Compute Metrics, Spark Observability and Performance Diagnostics

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 07 — Monitoring, Observability and Incident Response  
**Clouds:** AWS, Azure, GCP  
**Audience:** SRE, DBRE, Platform Engineering, Performance Engineering  
**Prerequisites:** Chapters 011–015 and 031  
**Safety:** Production exercises are read-only; workload and configuration changes require approved staging.

## 1. Mission

Diagnose Databricks compute performance using evidence from job timelines, cluster metrics, Spark UI, executor/task behavior and cloud infrastructure. Separate provisioning delay, resource contention, data skew, shuffle overhead and downstream storage/network issues before changing compute size.

### Outcomes

- Establish a repeatable compute performance baseline.
- Read Spark job, stage, task and executor metrics.
- Detect CPU, memory, garbage collection and spill pressure.
- Distinguish skew, shuffle and insufficient parallelism.
- Analyze startup, autoscaling, quotas and capacity failures.
- Correlate Spark behavior with AWS, Azure and GCP signals.
- Validate improvements using staging workloads and cost/SLO guardrails.

## 2. Architecture

    Workflow / SQL request
             |
       Job queue and startup
             |
      Databricks compute
       /             \
    Driver          Executors
       \             /
         Spark stages
              |
       Tasks / shuffle / spill
              |
      Storage and network
       /       |       \
      S3     ADLS      GCS
              |
     Metrics / logs / events
              |
    Dashboards and incident RCA

A slow job can originate in startup, scheduling, compute, shuffle, data access, or downstream systems. Measure each phase separately.

## 3. Compute lifecycle phases

| Phase | Typical bottleneck | Evidence |
|---|---|---|
| Queueing | Concurrency and limits | Run timeline, queue time |
| Provisioning | Cloud capacity, policy, quotas | Cluster events, cloud logs |
| Initialization | Libraries, init scripts, networking | Driver logs, startup events |
| Execution | CPU, memory, skew, shuffle | Spark UI, executor metrics |
| Commit/output | Storage, transactions, downstream | Delta history, cloud metrics |
| Termination | Cleanup and resource lifecycle | Cluster events, billing |

Do not attribute provisioning delay to Spark executor performance.

## 4. Key metrics and interpretations

| Metric | Meaning | Common caveat |
|---|---|---|
| CPU utilization | Compute saturation indicator | Average hides hot executors |
| Memory used | Memory pressure indicator | JVM heap differs from system memory |
| JVM GC time | Collection overhead | Correlate with task duration |
| Shuffle read/write | Data exchange volume | High volume can be expected |
| Spill memory/disk | Execution memory pressure | May indicate skew or layout |
| Task duration p95/p99 | Tail execution time | Long tail may be stragglers |
| Executor lost | Compute/runtime instability | Root cause can be cloud/network |
| Input bytes and records | Scan work | Not a correctness measure |
| Scheduler delay | Task dispatch overhead | Can increase with excessive tasks |
| Cluster startup time | Provisioning and init delay | Distinct from processing time |

Use workload-specific baselines; no single threshold is appropriate for every Spark application.

## 5. Spark UI navigation

Review in this order:
1. Jobs tab: which job consumed elapsed time?
2. Stages: which stage dominates and why?
3. Tasks: duration distribution, locality, skew and retries.
4. SQL tab where applicable: physical plan, scans, joins and exchanges.
5. Executors: task distribution, failures, GC and memory/spill.
6. Environment and logs: configuration, driver/executor errors.
7. Correlate with cloud CPU, memory, disk, network and storage metrics.

A single expensive stage is not necessarily defective; compare against expected data volume and the previous healthy baseline.

## 6. Data skew diagnostics

Indicators include:
- One or a few tasks much slower than peers.
- Extreme task input-size or shuffle-read imbalance.
- Long-tail completion while most executors idle.
- Large spills on a subset of executors.

Potential causes: uneven keys, hot partitions, joins, group-by patterns, insufficient partitioning or changed data distribution.

Mitigation requires workload-specific validation: adaptive query execution, query rewrites, data layout or skew-aware strategies. Increasing cluster size alone may not solve a single hot task.

## 7. Shuffle and spill diagnostics

Shuffle involves data exchange between stages. High shuffle can stress network, memory and local disk. Disk spill is not automatically a failure; persistent spill combined with degraded latency and executor loss deserves investigation.

Compare:
- Shuffle read/write volume.
- Remote read wait and fetch failures.
- Task memory/disk spill.
- Executor GC and out-of-memory symptoms.
- Local disk capacity and I/O pressure.
- Network throughput and storage service errors.

## 8. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm host, identity and environment before collecting evidence.

## 9. Lab B — Job and compute inventory

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks clusters get --help
databricks clusters events --help
~~~

CLI syntax and permissions vary. Review installed help and account capabilities before executing additional commands.

### Inventory

| Job | Run ID | Compute ID | Node type | Workers | Duration | Owner |
|---|---|---|---|---|---|---|
| demo-batch | | | | | | |

## 10. Lab C — Run timeline breakdown

For an authorized completed run, record:

| Phase | Start UTC | End UTC | Duration |
|---|---|---|---|
| Queued | | | |
| Provisioning | | | |
| Initialization | | | |
| Spark execution | | | |
| Output commit | | | |

Do not assume the Jobs API exposes every phase directly. Derive phase boundaries only from actual run/task timelines, compute events and logs; label estimates.

## 11. Lab D — Spark UI evidence worksheet

| Stage | Tasks | Median task duration | Max task duration | Shuffle read | Spill | Failed tasks |
|---|---|---|---|---|---|---|
| | | | | | | |

Capture the slowest stage and task distribution. Redact sensitive SQL, paths and application identifiers.

## 12. Lab E — Offline skew calculation

~~~powershell
$taskDurationsSec = @(12, 13, 12, 14, 11, 12, 13, 135)
$sorted = $taskDurationsSec | Sort-Object
$median = ($sorted[3] + $sorted[4]) / 2
$max = ($sorted | Measure-Object -Maximum).Maximum
[pscustomobject]@{
  MedianSeconds = $median
  MaxSeconds = $max
  MaxToMedianRatio = [math]::Round($max / $median, 2)
}
~~~

Synthetic output indicates a severe long-tail task. The ratio is a triage signal, not proof of key skew; correlate task input and shuffle metrics.

## 13. Lab F — Offline execution efficiency

~~~powershell
$queuedSec = 80
$startupSec = 190
$executionSec = 420
$commitSec = 35
$totalSec = $queuedSec + $startupSec + $executionSec + $commitSec
[pscustomobject]@{
  TotalSeconds = $totalSec
  ExecutionPct = [math]::Round(100 * $executionSec / $totalSec, 2)
  NonExecutionSeconds = $totalSec - $executionSec
}
~~~

Use this decomposition to identify whether optimizing Spark execution can meaningfully improve end-to-end latency.

## 14. Lab G — Read-only system table discovery

From an authorized SQL warehouse:

~~~sql
SHOW SCHEMAS IN system;
SHOW TABLES IN system.compute;
SHOW TABLES IN system.billing;
~~~

Where supported:

~~~sql
DESCRIBE TABLE system.billing.usage;
~~~

Availability, freshness and permissions differ. Do not assume system tables expose all per-executor performance metrics; Spark UI and compute telemetry remain essential.

## 15. Lab H — Cloud infrastructure correlation

~~~powershell
# AWS
aws sts get-caller-identity
aws configure get region

# Azure
az account show --query "{subscription:id,tenant:tenantId}" -o json

# GCP
gcloud config get-value project
~~~

Then inspect authorized cloud-native telemetry:
- AWS: EC2 capacity, EBS I/O, network, CloudWatch, quotas and spot interruptions where relevant.
- Azure: VM/VMSS metrics, disk and network latency, Azure Monitor, quota and allocation errors.
- GCP: VM metrics, persistent disk/network, Cloud Monitoring, quota and preemption where relevant.

The cloud console identity is not necessarily the Databricks runtime identity.

## 16. Lab I — Staging benchmark design

**Prerequisites:** representative synthetic workload, approved staging compute, cost cap, unchanged data and documented configuration.

1. Run baseline three or more times with controlled cache/warm-up conditions.
2. Capture end-to-end, startup, stage and task timing.
3. Capture executor CPU, memory, GC, shuffle and spill.
4. Change one variable: compute size, autoscaling, job concurrency or approved workload setting.
5. Repeat with comparable conditions.
6. Validate output correctness and cost per successful run.
7. Roll back if SLO, correctness or spend regresses.

Do not claim causality from one run or change several variables simultaneously.

## 17. Incident A — Executor out-of-memory

**Symptoms:** executor loss, OOM errors, task retries and unstable duration.

**Triage:** inspect JVM heap, off-heap memory, task skew, shuffle/spill and driver versus executor logs.

**Recovery:** address skew/workload memory requirements or approved compute configuration.

**Acceptance:** no repeated OOM, correct output and acceptable cost.

## 18. Incident B — Long-tail tasks from skew

**Symptoms:** most tasks complete quickly while a few run much longer.

**Triage:** compare per-task input, shuffle read, keys and partition distribution.

**Recovery:** staging-tested skew mitigation or query/layout change.

**Acceptance:** lower tail latency without correctness regression.

## 19. Incident C — Shuffle fetch failures

**Symptoms:** failed stages, remote fetch errors and retries.

**Triage:** network, executor loss, local disk, shuffle service behavior and cloud events.

**Recovery:** restore failed infrastructure or adjust workload through approved change.

**Acceptance:** stages complete without repeated fetch errors.

## 20. Incident D — Slow cluster startup

**Symptoms:** job deadline misses before Spark execution begins.

**Triage:** queue time, cloud quotas, instance availability, policies, init scripts, library installation and private connectivity.

**Recovery:** correct provisioning dependency, use suitable compute strategy and validate cost.

**Acceptance:** startup SLO met and run output correct.

## 21. Incident E — Autoscaling oscillation

**Symptoms:** frequent worker changes, unstable duration and unexpected cost.

**Triage:** workload burst pattern, min/max workers, task backlog, autoscaling events and provisioning delays.

**Recovery:** tune approved capacity bounds and workload scheduling.

**Acceptance:** stable completion time and predictable resource spend.

## 22. Incident F — Storage latency mistaken for CPU saturation

**Symptoms:** high job duration with idle CPU or blocked tasks.

**Triage:** Spark I/O wait, cloud storage latency, object throttling, network and Delta metadata.

**Recovery:** address storage/network bottleneck rather than adding idle compute.

**Acceptance:** I/O latency and end-to-end duration normalize.

## 23. Troubleshooting decision tree

    Slow or failing Databricks workload
                    |
          Delay before execution?
           yes -> queue / startup / quotas
           no
                    |
          Executor/task failures?
           yes -> OOM / loss / fetch / runtime
           no
                    |
          Long-tail task imbalance?
           yes -> skew / partition distribution
           no
                    |
          Shuffle/spill pressure high?
           yes -> memory / disk / network / plan
           no
                    |
          Storage wait or external dependency?
           yes -> object store / IAM / network
           no -> deeper plan and data analysis

## 24. Production performance runbook

1. Identify affected job, run, workspace and business deadline.
2. Capture last healthy baseline and UTC change timeline.
3. Decompose queue, startup, execution and commit time.
4. Review Spark UI jobs, stages, tasks and executors.
5. Check cloud capacity, storage, network and quotas.
6. Form one evidence-backed bottleneck hypothesis.
7. Rehearse mitigation in staging with representative data.
8. Apply approved change with rollback and cost guardrails.
9. Validate correctness, p95 duration, failure rate and cost.
10. Record RCA, capacity implications and monitoring improvements.

## 25. Dashboard design

### Workload reliability

- Job completion p50/p95/p99.
- Queue/startup/execution/commit breakdown.
- Task failure and retry rate.
- Deadline compliance and data freshness.

### Compute health

- Driver/executor CPU and memory.
- JVM GC time and executor loss.
- Shuffle read/write, fetch failures and spill.
- Autoscaling events and provisioning failures.

### Cloud dependency

- Instance allocation failures and quotas.
- Disk latency and capacity.
- Network throughput/errors.
- Storage request latency and throttling.
- Cost per successful validated run.

## 26. Alert strategy

Page for user-impacting failures, sustained deadline risk or repeated executor instability. Route low-impact transient resource spikes to dashboards or tickets unless correlated with SLO degradation.

Every page should identify workload, run ID, symptom, time window, owner and runbook. Test notification delivery in staging.

## 27. Change and rollback controls

Before modifying worker size, node family, runtime, autoscaling or Spark configuration:
- Record baseline and configuration.
- Validate quota and regional availability.
- Identify critical job schedules.
- Set cost cap and rollback threshold.
- Benchmark with equivalent input.
- Verify output correctness.
- Confirm post-change telemetry and alerting.

A faster job that produces incorrect output or doubles cost without benefit is not an accepted optimization.

## 28. Operational acceptance checklist

- [ ] Compute lifecycle phases mapped.
- [ ] Job/run/compute inventory collected.
- [ ] Spark UI stage/task/executor evidence reviewed.
- [ ] CPU, memory, GC, shuffle and spill baselines recorded.
- [ ] Startup and provisioning latency separated from execution.
- [ ] Skew and long-tail task diagnostics demonstrated.
- [ ] Cloud metrics and quota dependencies correlated.
- [ ] Controlled staging benchmark completed.
- [ ] Output correctness and unit cost validated.
- [ ] Six incident scenarios reviewed.
- [ ] Alert and rollback ownership documented.
- [ ] No unapproved production tuning performed.

## 29. Review questions

1. Why does low average CPU not rule out a compute performance issue?
2. How can a single skewed task dominate a large cluster's runtime?
3. Why must queue/startup time be separated from Spark execution?
4. What distinguishes executor OOM from storage-induced task failure?
5. What evidence is required to claim an optimization improved reliability?

## 30. Official references

- Databricks compute monitoring: https://docs.databricks.com/aws/en/compute/cluster-metrics
- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Apache Spark monitoring: https://spark.apache.org/docs/latest/monitoring.html
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- AWS CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs

**Next:** Chapter 033 — Databricks Log Collection, Audit Events and Cross-Cloud Telemetry Integration.
