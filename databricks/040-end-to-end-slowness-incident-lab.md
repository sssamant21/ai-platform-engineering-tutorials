# Chapter 040 — End-to-End Slowness Incident Lab

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 08 — Performance and Troubleshooting (capstone)  
**Clouds:** AWS, Azure, Google Cloud  
**Prerequisites:** Chapters 031–039  
**Audience:** SRE, DBRE, Platform Engineering, Cloud Networking and Incident Command  
**Execution policy:** Production investigation is read-only. Failure injection, load testing and tuning are limited to approved staging workloads with cost and rollback controls.

## 1. Mission

Investigate a realistic end-to-end Databricks slowness incident. Trace the failure from business impact through workflow scheduling, compute startup, Spark stages, executor metrics, cloud network/storage telemetry and data correctness. Select an evidence-backed remediation, validate it in staging and produce an operationally complete RCA.

### Completion criteria

- Establish a measurable business SLO and incident scope.
- Capture healthy versus degraded end-to-end timings.
- Identify the dominant phase and critical path.
- Distinguish CPU, memory, skew, shuffle, DNS, networking and storage waits.
- Correlate Databricks and cloud telemetry in UTC.
- Validate a safe recovery with correctness and cost guardrails.
- Produce RCA, runbook, acceptance decision and prevention actions.

## 2. Incident scenario

A synthetic reporting workflow named demo-hourly-refresh normally delivers a validated Delta dataset within 20 minutes of its scheduled start. During a staged exercise, completion time increases to 42 minutes. The downstream consumer receives stale data.

Illustrative incident record:

| Attribute | Value |
|---|---|
| Service | demo-hourly-refresh |
| Environment | demo-staging |
| Deadline | 20 minutes after schedule |
| Baseline end-to-end | 14 minutes |
| Incident end-to-end | 42 minutes |
| Data correctness | Must be independently checked |
| Cloud | AWS, Azure or GCP |
| Severity | Exercise SEV2, not a real incident |
| Recovery goal | Meet agreed deadline and validate output |

Do not treat these synthetic values as universal production SLO targets.

## 3. Reference architecture

    Scheduled workflow
            |
       Jobs admission
            |
      Compute provision
            |
       Spark driver
            |
      Stages and tasks
       /     |      \
    CPU/GC  Shuffle  Object storage
       \     |      /
         Delta commit
            |
      Quality validation
            |
      Consumer freshness
            |
       SLO + alerting

Supporting evidence comes from Jobs API, Spark UI, compute metrics, cloud metrics, audit events and Delta history.

## 4. Lab safety and setup

Before starting, document:

- Authorized staging workspace and cloud account/project/subscription.
- Job owner, on-call test route and escalation backup.
- Disposable synthetic dataset and test catalog/schema.
- Expected row counts, uniqueness and freshness invariants.
- Approved test duration, concurrency and cost cap.
- Baseline compute size, runtime and autoscaling policy.
- Recovery/rollback owner and change record.
- Evidence storage with access control and redaction.

Never induce failures or change cluster, IAM, network or storage settings in production for this exercise.

## 5. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Verify the actual host and principal. Do not rely on the profile name alone.

## 6. Lab B — Inventory and run discovery

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks jobs get-run --help
databricks clusters events --help
~~~

Use installed CLI help to confirm exact flags for the selected run and cluster. Inventory can be paginated or permission-filtered.

## 7. Lab C — Capture a baseline

Select three healthy runs of comparable input size and compute settings.

| Run | Queue sec | Startup sec | Planning sec | Execution sec | Commit sec | Total sec | Valid output |
|---|---|---|---|---|---|---|---|
| Baseline 1 | | | | | | | |
| Baseline 2 | | | | | | | |
| Baseline 3 | | | | | | | |

Capture job/run IDs, UTC timestamps, worker count, node family, runtime, input size, cache state and concurrent workloads.

## 8. Lab D — Incident timeline

Create a unified UTC record:

| UTC timestamp | Source | Event | Impact | Evidence |
|---|---|---|---|---|
| | Scheduler | Scheduled start | | |
| | Jobs | Queue begins/ends | | |
| | Compute | Provisioning | | |
| | Spark | Stage begins | | |
| | Cloud | Storage/network change | | |
| | Delta | Commit | | |
| | Validation | Output check | | |
| | SLO | Deadline breach | | |

Distinguish event time from ingestion time. Time correlation is evidence, not proof of causation.

## 9. Lab E — Synthetic phase analysis

~~~powershell
$baseline = [ordered]@{Queue=30;Startup=150;Planning=30;Execution=580;Commit=50}
$incident = [ordered]@{Queue=60;Startup=240;Planning=60;Execution=2050;Commit=110}

$rows = foreach ($phase in $baseline.Keys) {
  [pscustomobject]@{
    Phase=$phase
    BaselineSeconds=$baseline[$phase]
    IncidentSeconds=$incident[$phase]
    AddedSeconds=$incident[$phase]-$baseline[$phase]
  }
}
$rows | Format-Table -AutoSize
[pscustomobject]@{
  BaselineTotalSeconds=($baseline.Values | Measure-Object -Sum).Sum
  IncidentTotalSeconds=($incident.Values | Measure-Object -Sum).Sum
}
~~~

The synthetic totals are 840 seconds (14 minutes) and 2520 seconds (42 minutes). Spark execution accounts for most of the added time; investigate execution before making provisioning the primary remediation.

## 10. Lab F — Spark UI investigation

For the degraded run:

1. Open Jobs and find the critical-path Spark job.
2. Identify the longest stage and any stage retries.
3. Compare task median, p95 and maximum duration.
4. Inspect task input records/bytes and shuffle read/write.
5. Review GC, spill and executor losses.
6. Inspect SQL/DataFrame physical plan when available.
7. Compare with the baseline run under similar input.

Evidence worksheet:

| Metric | Healthy | Incident | Finding |
|---|---|---|---|
| Longest stage sec | | | |
| Task median sec | | | |
| Task maximum sec | | | |
| Shuffle read GB | | | |
| Disk spill GB | | | |
| GC percentage | | | |
| Executor losses | | | |

## 11. Lab G — Synthetic task skew analysis

~~~powershell
$taskSeconds = @(12,13,13,14,15,15,16,180)
$sorted = @($taskSeconds | Sort-Object)
$median = ($sorted[3]+$sorted[4])/2
$max = ($sorted | Measure-Object -Maximum).Maximum
[pscustomobject]@{
  MedianSeconds=$median
  MaximumSeconds=$max
  MaxMedianRatio=[math]::Round($max/$median,2)
}
~~~

A long tail suggests a straggler. Confirm with input and shuffle distribution, executor events and retries before declaring skew.

## 12. Lab H — Query plan analysis

Use a known authorized staging table:

~~~sql
EXPLAIN FORMATTED
SELECT category, COUNT(*) AS total
FROM demo_catalog.demo_staging.events
WHERE event_date >= DATE '2026-10-01'
GROUP BY category;
~~~

Look for scans, filters, exchanges, aggregates and adaptive execution. EXPLAIN describes a plan; it does not measure actual elapsed time.

## 13. Lab I — Delta and storage metadata

~~~sql
DESCRIBE DETAIL demo_catalog.demo_staging.events;
DESCRIBE HISTORY demo_catalog.demo_staging.events;
~~~

Confirm table existence and permissions. Record file count, table size, commit timing and concurrent writers. Do not delete Delta transaction logs.

## 14. Lab J — System telemetry discovery

~~~sql
SHOW SCHEMAS IN system;
SHOW TABLES IN system.access;
SHOW TABLES IN system.compute;
SHOW TABLES IN system.lakeflow;
~~~

Where available:

~~~sql
DESCRIBE TABLE system.access.audit;
DESCRIBE TABLE system.lakeflow.job_run_timeline;
~~~

System tables and columns vary by cloud, region and account. Missing data is a coverage gap, not evidence of a healthy platform.

## 15. Lab K — AWS correlation

~~~powershell
aws sts get-caller-identity
aws configure get region
aws cloudwatch describe-alarms --max-records 25
aws service-quotas list-service-quotas --service-code ec2 --max-results 20
~~~

Correlate EC2 capacity and CPU, storage/network telemetry, S3 request latency and IAM changes. AWS CLI identity may differ from the Databricks compute role.

## 16. Lab L — Azure correlation

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus -o table
az monitor metrics alert list --resource-group demo-staging-rg -o json
~~~

Inspect Azure Monitor VM/network/storage metrics, regional quotas and Activity Logs for the actual workspace resources.

## 17. Lab M — GCP correlation

~~~powershell
gcloud config get-value project
gcloud compute regions describe us-central1 --format=json
gcloud logging sinks list
~~~

Review Cloud Monitoring, Compute Engine quotas, VPC telemetry and Cloud Storage metrics. Confirm project and region first.

## 18. Lab N — DNS and connectivity checks

~~~powershell
Resolve-DnsName example.com
Test-NetConnection example.com -Port 443
curl.exe -I --connect-timeout 5 --max-time 15 https://example.com
~~~

These commands test the local workstation, not Databricks compute. Use an approved compute-side diagnostic method to validate workspace connectivity; never disable TLS validation.

## 19. Determine the bottleneck

| Observation | Candidate cause | Disproving evidence |
|---|---|---|
| Long queue | Concurrency limit | Queue unchanged from baseline |
| Long startup | Cloud provisioning | Startup stable |
| All workers CPU-bound | Compute saturation | Low task CPU time |
| Few slow tasks | Skew/straggler | Balanced task distribution |
| High GC/OOM | Memory pressure | Stable memory and low GC |
| High shuffle fetch wait | Network/executor loss | No shuffle or network wait |
| Slow scans, low CPU | Storage/network | Normal storage latency |
| Slow commit | Delta transaction contention | No overlapping writes |
| Output late but job fast | Downstream validation/consumer | Consumer timing normal |

Select one primary hypothesis and list the evidence required to reject it.

## 20. Lab O — Error-budget impact

~~~powershell
$eligible = 1000
$good = 991
$target = 0.995
$bad = $eligible - $good
$allowedBad = $eligible*(1-$target)
[pscustomobject]@{
  CompliancePct=[math]::Round(100*$good/$eligible,2)
  AllowedBad=$allowedBad
  ObservedBad=$bad
  BudgetUsedPct=[math]::Round(100*$bad/$allowedBad,1)
}
~~~

Synthetic outcome: 99.1% compliance and 180% event-budget consumption. Do not mix event-based and time-based budgets.

## 21. Controlled remediation experiment

Choose **one** approved staging intervention:

- Fix a demonstrably skewed partition or join.
- Reduce unnecessary shuffle or small-file overhead.
- Adjust approved compute capacity for confirmed CPU pressure.
- Change scheduling or workload isolation for proven contention.
- Correct cloud networking/storage issue through change control.

Procedure:

1. Preserve baseline and incident evidence.
2. Document hypothesis and expected metric change.
3. Apply one staging-only change.
4. Run at least three comparable trials.
5. Compare p50/p95 duration and critical-path stages.
6. Validate output count, uniqueness and data quality.
7. Compare cost per successful validated run.
8. Roll back if performance, correctness or cost regresses.
9. Document whether evidence supports the hypothesis.

## 22. Incident simulation A — CPU saturation

**Inject/tabletop:** concurrent staging workload saturates workers.

**Investigate:** per-worker CPU, runnable tasks, stage time and concurrency.

**Mitigate:** approved scheduling/isolation or capacity change.

**Pass:** deadline compliance improves without runaway cost.

## 23. Incident simulation B — Executor OOM

**Inject/tabletop:** synthetic memory-heavy stage fails.

**Investigate:** first OOM, executor logs, GC, spill and skew.

**Mitigate:** stop retry amplification and stage memory-aware correction.

**Pass:** no executor churn and correct output.

## 24. Incident simulation C — Skewed join

**Inject/tabletop:** one key dominates input.

**Investigate:** task duration and input/shuffle distribution.

**Mitigate:** approved skew-aware query/layout adjustment.

**Pass:** reduced tail latency, identical result.

## 25. Incident simulation D — Storage latency

**Tabletop:** object-store request latency rises.

**Investigate:** Spark scan wait, cloud request metrics, network path.

**Mitigate:** resolve storage/request pattern bottleneck, not blind CPU scale-out.

**Pass:** storage latency and SLO normalize.

## 26. Incident simulation E — Provisioning quota

**Tabletop:** regional quota blocks additional workers.

**Investigate:** cluster events, requested instance family and provider quotas.

**Mitigate:** approved quota or alternate allowed capacity.

**Pass:** predictable startup and controlled spend.

## 27. Incident simulation F — DNS/private endpoint regression

**Tabletop:** storage endpoint resolves incorrectly.

**Investigate:** DNS, private zone, route, firewall and TLS.

**Mitigate:** approved rollback of specific networking change.

**Pass:** correct compute-side connectivity and authorized access.

## 28. Incident simulation G — Concurrent Delta writers

**Tabletop:** overlapping jobs conflict at commit.

**Investigate:** Delta history, run overlap and transaction errors.

**Mitigate:** coordinate writers and idempotent retry.

**Pass:** no duplicate or missing data.

## 29. Incident simulation H — Silent correctness failure

**Inject:** staging workflow finishes but output violates an invariant.

**Investigate:** data quality gate, SLI eligibility and downstream freshness.

**Mitigate:** correct synthetic output and validation logic.

**Pass:** service remains unhealthy until correctness and freshness are proven.

## 30. Read-only evidence collection

~~~powershell
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
$stamp = (Get-Date).ToUniversalTime().ToString("yyyyMMddTHHmmssZ")
$folder = Join-Path $PWD "slowness-incident-$stamp"
New-Item -ItemType Directory -Path $folder | Out-Null

databricks jobs list --output json |
  Out-File (Join-Path $folder "jobs.json") -Encoding utf8

databricks jobs list-runs --limit 25 --output json |
  Out-File (Join-Path $folder "runs.json") -Encoding utf8

Write-Host "Review and redact before sharing: $folder"
~~~

Never commit raw operational evidence, tokens or sensitive SQL text to a public repository.

## 31. Incident command and communication

- Declare severity from business impact.
- Identify incident commander, investigator, communications lead and approver.
- State affected workflows, consumer impact and UTC start.
- Share only verified facts and clearly labeled hypotheses.
- Record mitigation decision, rollback owner and next update.
- Validate recovery before closing.
- Track residual risks and follow-up owners.

## 32. Root cause analysis template

    Incident title:
    Incident ID:
    Start / detection / recovery UTC:
    Cloud / region / workspace:
    Affected service and SLO:
    Customer / consumer impact:
    Baseline vs degraded timing:
    Critical-path phase:
    Primary root cause:
    Contributing factors:
    Evidence supporting cause:
    Hypotheses rejected:
    Detection gap:
    Containment and mitigation:
    Rollback and correctness validation:
    Cost impact:
    Prevention actions and owners:
    Acceptance decision:

Avoid attributing root cause to CPU, Spark or cloud infrastructure without corroborating evidence.

## 33. Production incident runbook

1. Confirm impact, deadline, affected consumers and incident severity.
2. Capture UTC timeline and compare last healthy runs.
3. Split queue, startup, planning, execution and commit time.
4. Inspect Spark jobs, stages, tasks and executors.
5. Inspect query plan and Delta metadata.
6. Correlate AWS/Azure/GCP compute, network and storage metrics.
7. Form a falsifiable root-cause hypothesis.
8. Contain unsafe retries and select approved mitigation.
9. Validate output correctness, freshness, p95 and cost.
10. Publish RCA, alerts and capacity/runbook improvements.

## 34. Acceptance scorecard

| Requirement | Result | Evidence |
|---|---|---|
| SLO and business impact defined | Not assessed | |
| Healthy baseline captured | Not assessed | |
| Slow run phase decomposition | Not assessed | |
| Spark critical path identified | Not assessed | |
| Task skew/GC/shuffle assessed | Not assessed | |
| Query plan and Delta history checked | Not assessed | |
| Cloud metrics correlated | Not assessed | |
| DNS/network/storage assessed | Not assessed | |
| Hypothesis and disproof criteria documented | Not assessed | |
| Staging remediation benchmarked | Not assessed | |
| Correctness and freshness verified | Not assessed | |
| Cost and rollback validated | Not assessed | |
| Eight incident scenarios reviewed | Not assessed | |
| RCA and prevention actions completed | Not assessed | |

## 35. GO / CONDITIONAL GO / NO-GO

**GO:** Root cause supported by evidence; recovery meets SLO; data correctness/freshness and cost validated; monitoring and rollback proven.

**CONDITIONAL GO:** Only noncritical documented gaps remain with approved owner, mitigation and deadline.

**NO-GO:** Unknown critical-path bottleneck, unsafe retries, missing correctness checks, unverified cloud dependency, or unresolved critical SLO breach.

## 36. Review questions

1. Why should execution time be separated from queue and startup time?
2. What evidence distinguishes skew from executor resource contention?
3. Why can adding workers fail to resolve storage latency?
4. What prevents an automated retry from duplicating Delta output?
5. Which evidence is required to close an incident after a job turns green?

## 37. Official references

- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks compute metrics: https://docs.databricks.com/aws/en/compute/cluster-metrics
- Databricks Delta Lake: https://docs.databricks.com/aws/en/delta/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Apache Spark monitoring: https://spark.apache.org/docs/latest/monitoring.html
- Apache Spark SQL tuning: https://spark.apache.org/docs/latest/sql-performance-tuning.html
- AWS CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs
- Google SRE workbook: https://sre.google/workbook/alerting-on-slos/

**Part 08 complete. Next:** Chapter 041 — Availability Architecture and Failure Domains (Part 09 — Reliability and Recovery).
