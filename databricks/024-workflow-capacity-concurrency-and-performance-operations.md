# Chapter 024 — Workflow Capacity, Concurrency and Performance Operations

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 05 — Jobs, Workflows and Production Reliability  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, Platform Engineering, Capacity Planning, FinOps  
**Prerequisites:** Chapters 021–023  
**Lab safety:** Read-only production assessment; approved staging experiments with synthetic workloads and cost caps

## 1. Mission

Build a repeatable operational method to size, monitor and recover Databricks Jobs capacity. Capacity is not merely the number of available cores: it is the interaction of trigger arrival rates, concurrency limits, compute provisioning, task execution, cloud quotas, shared dependencies and business deadlines.

### Learning objectives

- Model job admission, concurrency, queueing and end-to-end latency.
- Identify cloud capacity, compute policy and dependency bottlenecks.
- Distinguish job queue wait, compute startup and actual task execution.
- Establish capacity baselines and forecast demand.
- Design controlled scaling and workload isolation.
- Execute safe staging load experiments and incident recovery.
- Connect capacity changes to cost and workflow SLOs.

## 2. Capacity reference architecture

    Schedules / event arrivals
               |
       Jobs admission and queue
               |
     Concurrency / policy controls
               |
        Task dependency graph
               |
      +--------+---------+
      |                  |
    Jobs compute      Serverless
      |                  |
      +--------+---------+
               |
     Cloud quotas / networking
               |
    Storage / databases / APIs
               |
    Completed validated output
               |
      SLO / metrics / FinOps

A high concurrency limit does not guarantee immediate compute availability or faster completion. Bottlenecks can move to storage, networking, upstream systems or workspace/cloud limits.

## 3. Capacity dimensions

| Dimension | Measure | Operational concern |
|---|---|---|
| Arrival rate | Scheduled/event runs per interval | Peak trigger bursts |
| Active concurrency | Simultaneous admitted runs/tasks | Resource contention |
| Queue delay | Trigger-to-admission time | Deadline risk |
| Compute startup | Provisioning/initialization time | Quota, image, policy, pool |
| Execution time | Task wall time and CPU/I/O profile | Data volume and skew |
| Resource supply | Available cores, memory, workers | Quotas and autoscaling |
| Dependency capacity | Storage/API/database throughput | Throttling and saturation |
| Recovery capacity | Backlog drained per hour | Incident catch-up |
| Cost | Compute spend per validated run | Efficiency and budget |

## 4. Queueing and concurrency model

For a stable system, average arrival rate must be lower than sustainable service capacity. A simple first approximation uses Little's Law:

    L = lambda * W

Where L is average work in the system, lambda is average arrival rate, and W is average time in the system. Apply only when the observation window and system assumptions are appropriate.

Example: 12 job runs/hour with average 20-minute end-to-end time implies roughly four concurrent runs in the system on average. Peak arrivals, DAG parallelism, startup time and shared resource contention can make the required headroom much larger.

Do not use a single average to size production peaks.

## 5. Identify the bottleneck

### Trigger or admission

Symptoms: run created late, queue delay, max concurrent runs reached.

### Compute provisioning

Symptoms: long cluster startup, instance shortages, cloud quota failures, pool depletion or policy rejection.

### Execution

Symptoms: slow Spark stages, skew, spill, GC pressure, disk/network saturation, inefficient task parallelism.

### External dependency

Symptoms: storage throttling, database rate limits, network timeouts, API errors, delayed source files.

### Downstream validation

Symptoms: job finished but data contract, freshness or downstream publication missed deadline.

## 6. Cloud-specific capacity considerations

| Cloud | Common constraints to verify |
|---|---|
| AWS | EC2 vCPU/service quotas, instance availability, VPC IPs, EBS/S3 performance, instance profiles/IAM |
| Azure | VM family quotas, regional capacity, subnet IPs, managed identity, ADLS throttling, networking |
| GCP | Compute Engine quotas and availability, subnet capacity, service accounts, Cloud Storage throughput |

Capacity symptoms must be verified using the cloud's actual quota and telemetry. Cloud quota increases, instance family changes and subnet expansion require separate approvals and may have cost implications.

## 7. Workload isolation

Use environment-specific job definitions, compute policies, permissions and resource controls. Consider:
- Separate critical deadline-driven jobs from exploratory or backfill work.
- Reserve appropriate capacity only where supported and justified.
- Avoid one job monopolizing shared pools or external services.
- Limit concurrent backfills during peak production windows.
- Use workload-aware alerts instead of a single global threshold.
- Keep run-as identities and storage permissions least privileged.

## 8. Lab A — Windows PowerShell CLI preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks instance-pools list --help
~~~

Confirm the authorized workspace host and identity. Listings can be permission-filtered and paginated.

## 9. Lab B — Inspect a critical job

~~~powershell
# Replace example IDs with authorized values.
databricks jobs get --job-id 12345 --output json
databricks jobs list-runs --job-id 12345 --limit 25 --output json
databricks jobs get-run --run-id 67890 --output json
~~~

Verify flags against installed CLI help. Collect schedule, concurrency, queueing, task graph, compute mode, worker settings, autoscaling, task retries, runtime and run-as identity.

### Assessment worksheet

| Field | Observed | Target/baseline |
|---|---|---|
| Business deadline | | |
| Peak arrivals/hour | | |
| Max concurrent runs | | |
| p95 queue wait | | |
| p95 compute startup | | |
| p95 task execution | | |
| p95 end-to-end duration | | |
| Retry rate | | |
| Peak resource utilization | | |
| Cost per successful run | | |

## 10. Lab C — Offline workload sizing with PowerShell

Use **synthetic** values to estimate average in-system concurrency:

~~~powershell
$jobsPerHour = 18
$avgMinutes = 25
$lambdaPerMinute = $jobsPerHour / 60.0
$averageConcurrent = $lambdaPerMinute * $avgMinutes

[pscustomobject]@{
  JobsPerHour = $jobsPerHour
  AverageMinutes = $avgMinutes
  EstimatedAverageInSystem = [math]::Round($averageConcurrent, 2)
}
~~~

This is not a direct recommendation for the Databricks max-concurrent-runs setting. The calculation includes whatever waiting and execution stages are represented in the duration measurement. Validate peak concurrency and resource requirements from real workload telemetry.

## 11. Lab D — Estimate deadline headroom

~~~powershell
$deadlineUtc = [datetimeoffset]::Parse("2026-10-01T07:00:00Z")
$nowUtc = [datetimeoffset]::Parse("2026-10-01T06:10:00Z")
$queueMin = 8
$startupMin = 7
$executionMin = 28
$validationMin = 5
$predictedMin = $queueMin + $startupMin + $executionMin + $validationMin
$availableMin = ($deadlineUtc - $nowUtc).TotalMinutes

[pscustomobject]@{
  AvailableMinutes = $availableMin
  PredictedMinutes = $predictedMin
  HeadroomMinutes = $availableMin - $predictedMin
}
~~~

Replace synthetic estimates with observed distributions. Treat a negative headroom value as a risk signal requiring investigation, not automatic permission to scale production.

## 12. Lab E — System table exploration

Where enabled and authorized, Databricks system tables may expose job run timeline and compute/billing usage. Inspect schemas and access first:

~~~sql
SHOW TABLES IN system.lakeflow;
DESCRIBE TABLE system.lakeflow.job_run_timeline;
SELECT * FROM system.lakeflow.job_run_timeline LIMIT 10;
~~~

Timeline tables may have multiple slices per run; aggregate correctly and verify run identity and timestamp semantics. Availability, schema and retention vary. Do not infer compute CPU utilization directly from billing usage.

## 13. Lab F — Staging load and backlog experiment

**Preconditions:** approved disposable job, synthetic input, fixed cost budget, owner, rollback and cloud quota checks.

1. Establish baseline for run duration, queue wait, startup time and cost.
2. Select a safe number of synthetic arrivals within the approved budget.
3. Observe queueing and active concurrency.
4. Record task and dependency performance.
5. Compare results with the baseline and business deadline.
6. Reduce synthetic load or stop when cost/latency guardrails are exceeded.
7. Validate that no output was duplicated or lost.
8. Restore staging configuration and archive sanitized findings.

Never load-test shared production storage or services without explicit approval.

## 14. Capacity forecasting

Forecast using:
- Historical arrival rate by hour/day/week.
- Expected data growth and task execution trend.
- Peak season and batch windows.
- Planned product or data source changes.
- Cloud instance/region availability.
- Failure recovery and backfill requirements.
- Budget constraints and reservations where applicable.

Maintain at least baseline, expected growth and stress scenarios. Forecast **time to deadline miss**, not only resource exhaustion.

## 15. Incident A — Queue backlog after missed trigger

**Triage:** examine queued runs, overlapping jobs, concurrency and expected trigger calendar.

**Recovery:** prioritize critical work, use approved catch-up windows and prevent uncontrolled backfills.

**Acceptance:** backlog drains and business output reconciles.

## 16. Incident B — Cloud instance capacity shortage

**Triage:** check cloud quota, regional instance availability, policies and compute events.

**Recovery:** approved instance-family/region/capacity adjustment where supported; avoid speculative changes.

**Acceptance:** compute starts reliably and cost remains within agreed guardrails.

## 17. Incident C — Autoscaling does not improve duration

**Triage:** inspect task parallelism, skew, I/O, executor utilization, shuffle, startup overhead and external bottlenecks.

**Recovery:** rightsize or tune workload with owners rather than simply increasing maximum workers.

**Acceptance:** improved end-to-end duration at justified cost.

## 18. Incident D — Shared storage throttling

**Triage:** correlate concurrent jobs, object storage request limits, network throughput and error rates.

**Recovery:** throttle concurrency/backfills, optimize access patterns and coordinate storage capacity changes.

**Acceptance:** dependency errors normalize and outputs remain correct.

## 19. Incident E — Critical jobs displaced by backfill

**Triage:** compare scheduling, shared compute, quotas, queueing and external service contention.

**Recovery:** isolate backfill capacity or reschedule within approved windows.

**Acceptance:** critical workflow deadlines met without data loss.

## 20. Incident F — Cost spike after capacity increase

**Triage:** compare run counts, retries, worker utilization, startup time and cost per validated output.

**Recovery:** roll back ineffective sizing changes, correct retry storms and optimize compute selection.

**Acceptance:** SLO retained while unit cost returns toward baseline.

## 21. Troubleshooting decision tree

    Deadline miss / high latency
             |
       Queue delay high?
       yes -> concurrency, triggers, overlapping runs
       no
             |
       Startup delay high?
       yes -> quota, pool, policy, provisioning
       no
             |
       Execution time high?
       yes -> CPU, memory, Spark, skew, I/O
       no
             |
       External dependency slow?
       yes -> storage/API/network throttling
       no
             |
       Output validation late?
       yes -> downstream readiness and data checks

## 22. Production capacity incident runbook

1. Detect deadline risk, queue growth or elevated run duration.
2. Scope critical jobs, workspace, cloud region and shared dependencies.
3. Capture UTC timeline, recent changes and capacity baselines.
4. Decompose latency into trigger, queue, startup, execution and validation.
5. Identify bottleneck and confirm cloud quotas/dependency saturation.
6. Choose smallest approved mitigation with cost and rollback.
7. Validate critical output, backlog drain, cost and downstream impact.
8. Communicate expected recovery and residual backlog.
9. Record forecast changes and preventive actions.

## 23. Change management and rollback

Before changing concurrency, worker sizes, pools, scheduling or isolation:
- Export current job/compute settings securely.
- Review dependency and cloud quota constraints.
- Test in staging with synthetic load.
- Define p95 latency, error rate and cost rollback triggers.
- Roll out in limited scope.
- Verify both SLO improvement and cost per successful run.
- Restore prior approved settings if outcomes degrade.

Reducing concurrency may increase queue time; scaling up may increase cost or downstream load. Every change requires end-to-end validation.

## 24. Operational acceptance checklist

- [ ] Critical jobs and business deadlines inventoried.
- [ ] Queue, startup, execution and validation latency measured separately.
- [ ] Peak arrival and concurrency baseline established.
- [ ] Cloud quotas and subnet/IP constraints reviewed.
- [ ] External dependencies and shared capacity mapped.
- [ ] Workload isolation strategy documented.
- [ ] Synthetic staging load test completed.
- [ ] Capacity forecast includes growth and recovery scenarios.
- [ ] Six incident scenarios reviewed.
- [ ] Cost per validated successful run tracked.
- [ ] Rollback triggers and owners defined.
- [ ] No production load tests or unapproved changes performed.

## 25. Review questions

1. Why is average concurrency insufficient for peak capacity planning?
2. What distinguishes queueing delay from compute startup delay?
3. When does increasing workers fail to improve runtime?
4. Why can aggressive backfills threaten production deadlines?
5. How do you demonstrate that a capacity change improved reliability without waste?

## 26. References

- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks CLI Jobs: https://docs.databricks.com/aws/en/dev-tools/cli/reference/jobs-commands
- Azure Databricks: https://learn.microsoft.com/azure/databricks/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 025 — Production Workflow Reliability and Recovery Capstone.
