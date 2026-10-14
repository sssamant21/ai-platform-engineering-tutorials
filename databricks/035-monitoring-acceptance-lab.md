# Chapter 035 — Monitoring Acceptance Lab

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 07 — Observability (capstone)  
**Clouds:** AWS, Azure, GCP  
**Prerequisites:** Chapters 031–034  
**Audience:** SRE, DBRE, Platform Engineering, Cloud Security, FinOps  
**Change policy:** Production checks are read-only. Inject failures and modify alert rules only in approved staging resources.

## 1. Mission and exit criteria

Prove that the Databricks platform can detect, explain, escalate and verify recovery from operational failures. This is a hands-on acceptance test, not simply a checklist of dashboards.

Successful completion requires:

1. Inventory of critical services, owners, dependencies and telemetry sources.
2. Reliable job, compute, Spark, audit, cloud and data-freshness signals.
3. SLI/SLO definitions with explicit event eligibility.
4. Tested alert evaluation, notification, escalation and deduplication.
5. Evidence of cross-cloud incident correlation and source completeness.
6. Six staged simulations/tabletops with recovery validation.
7. Runbooks, retention, access controls and cost review.
8. A documented GO, CONDITIONAL GO or NO-GO decision.

## 2. Reference architecture

    Critical workflow / SQL workload
                 |
       Validated output + deadline
                 |
     Jobs API / Spark UI / compute
                 |
       System tables / audit logs
                 |
      AWS / Azure / GCP telemetry
                 |
       Normalization / UTC joins
                 |
       SLIs / SLOs / dashboards
                 |
      Alert routing and escalation
                 |
      Incident response and recovery
                 |
       Correctness and SLO check
                 |
        Evidence / acceptance

**Key principle:** Healthy infrastructure does not prove valid data; a successful job does not prove deadline compliance.

## 3. Lab scope and safety

Use a fictional service such as demo-hourly-refresh and a disposable synthetic Delta table in an approved staging workspace.

Before testing, record:
- Workspace and cloud region.
- Staging catalog/schema and service owner.
- On-call test destination and backup contact.
- Test cost cap and expected duration.
- Synthetic dataset and expected row-level invariants.
- Approved changes, rollback owner and change ticket.
- Whether system tables and cloud logs are available.
- Evidence retention and access restrictions.

Do not intentionally fail production workloads, alter production IAM, disable production monitoring or expose tokens in logs.

## 4. Acceptance control matrix

| Control | Evidence required | Pass criteria |
|---|---|---|
| Service inventory | Ownership and dependency mapping | All critical services covered |
| Job telemetry | Run/task state and timing | Failed runs visible |
| Compute telemetry | Startup, CPU, memory, events | Bottlenecks diagnosable |
| Spark telemetry | Stages, tasks, skew, spill | Slow stage identifiable |
| Audit coverage | Databricks and cloud events | Critical changes observable |
| Data reliability | Freshness/correctness checks | Bad output cannot appear healthy |
| SLO ledger | Eligibility and run outcomes | Reconciles with source runs |
| Alerts | Test event and delivered page | Correct owner and context |
| Escalation | Acknowledgement and fallback | Routing verified |
| Recovery | Output and dependency checks | Correctness and SLO restored |
| Governance | Access, redaction and retention | Security review passed |
| Cost | Collection/retention estimate | Approved budget |

## 5. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Validate the actual host and principal. A profile name alone is not evidence that you are in staging.

## 6. Lab B — Read-only inventory

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks jobs get-run --help
databricks clusters events --help
~~~

Confirm installed CLI syntax and pagination. Capture UTC observation time and note access-limited inventory.

### Inventory worksheet

| Service | Job ID | Owner | Compute | Criticality | Deadline | Dashboard |
|---|---|---|---|---|---|---|
| demo-hourly-refresh | | | | | | |

## 7. Lab C — System table discovery

Run from an authorized SQL warehouse:

~~~sql
SHOW SCHEMAS IN system;
SHOW TABLES IN system.access;
SHOW TABLES IN system.billing;
SHOW TABLES IN system.compute;
SHOW TABLES IN system.lakeflow;
~~~

Where present:

~~~sql
DESCRIBE TABLE system.access.audit;
DESCRIBE TABLE system.billing.usage;
DESCRIBE TABLE system.lakeflow.job_run_timeline;
~~~

Availability and column schemas vary by account, region, permissions and feature support. Missing tables require an alternate supported source; do not fabricate telemetry.

## 8. Lab D — Audit signal baseline

~~~sql
-- Run only if system.access.audit is available and authorized.
SELECT
  date_trunc('hour', event_time) AS event_hour,
  service_name,
  action_name,
  COUNT(*) AS event_count
FROM system.access.audit
WHERE event_time >= current_timestamp() - INTERVAL 1 DAY
GROUP BY date_trunc('hour', event_time), service_name, action_name
ORDER BY event_hour DESC;
~~~

Compare event delivery time with source event time. Do not interpret a zero count as proof that no actions occurred.

## 9. Lab E — Cloud monitoring inventory

Use only the relevant cloud commands:

~~~powershell
# AWS
aws sts get-caller-identity
aws configure get region
aws cloudwatch describe-alarms --max-records 25

# Azure
az account show --query "{subscription:id,tenant:tenantId}" -o json
az monitor metrics alert list --resource-group demo-staging-rg -o json

# GCP
gcloud config get-value project
gcloud logging sinks list
~~~

Inspect authorized cloud console/API metrics for compute capacity, disk, network, storage latency, throttling, IAM and quotas. Cloud-native alarms are not substitutes for workflow deadline and correctness SLIs.

## 10. Lab F — Synthetic workload and validation contract

In approved staging SQL:

~~~sql
CREATE TABLE IF NOT EXISTS demo_catalog.demo_staging.monitoring_acceptance (
  event_id BIGINT,
  batch_id STRING,
  status STRING
) USING DELTA;

INSERT INTO demo_catalog.demo_staging.monitoring_acceptance
VALUES (101, 'batch-001', 'READY'),
       (102, 'batch-001', 'READY');

SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT event_id) AS distinct_events,
  COUNT_IF(status IS NULL) AS null_statuses
FROM demo_catalog.demo_staging.monitoring_acceptance;
~~~

Use a uniquely named disposable table and ensure repeated lab execution cannot silently duplicate test rows. Clean up only after approval and evidence capture.

Expected invariant for a fresh test table: two distinct events, no null status. If the table already exists, reset only the approved disposable staging object before using these expectations.

## 11. Lab G — Job timeline and Spark UI baseline

For one completed staging run, record:

| Metric | Baseline | Source |
|---|---|---|
| Queue duration | | Jobs timeline |
| Startup duration | | Compute events |
| Execution duration | | Run details |
| Stage p95 task time | | Spark UI |
| Failed/retried tasks | | Spark UI |
| Executor GC and spill | | Spark UI/metrics |
| Data freshness | | Validation table |
| Cloud storage errors | | Cloud monitoring |

A job's elapsed time must be decomposed before selecting a remediation.

## 12. Lab H — SLO ledger acceptance

Define an eligible scheduled run and a validated good run. Count retries consistently.

~~~sql
WITH events AS (
  SELECT * FROM VALUES
    ('demo-hourly-refresh', true, true, true),
    ('demo-hourly-refresh', true, true, false),
    ('demo-hourly-refresh', true, false, false),
    ('demo-hourly-refresh', false, false, false)
  AS t(service, eligible, valid_output, deadline_met)
)
SELECT
  service,
  COUNT_IF(eligible) AS eligible_runs,
  COUNT_IF(eligible AND valid_output) AS valid_runs,
  COUNT_IF(eligible AND valid_output AND deadline_met) AS good_runs,
  ROUND(
    100.0 * COUNT_IF(eligible AND valid_output AND deadline_met) /
    NULLIF(COUNT_IF(eligible), 0), 2
  ) AS on_time_valid_pct
FROM events
GROUP BY service;
~~~

Expected: three eligible runs and one on-time valid run. Confirm denominator and semantics before adapting to real Jobs data.

## 13. Lab I — Offline error-budget exercise

~~~powershell
$eligible = 1000
$good = 986
$target = 0.99
$allowedBad = $eligible * (1 - $target)
$observedBad = $eligible - $good

[pscustomobject]@{
  SloPct = $target * 100
  ActualPct = [math]::Round(100 * $good / $eligible, 2)
  AllowedBad = $allowedBad
  ObservedBad = $observedBad
  BudgetUsedPct = [math]::Round(100 * $observedBad / $allowedBad, 1)
}
~~~

Synthetic result: 98.6% compliance and 140% budget consumption. Do not mix event-based budgets with downtime-minute budgets.

## 14. Lab J — Cross-source correlation

Build a bounded UTC timeline:

| UTC | Source | Event | Workspace/job/region | Interpretation |
|---|---|---|---|---|
| | Databricks | Run failure | | |
| | Spark UI | Executor loss | | |
| | Cloud metrics | Capacity or storage issue | | |
| | Audit | Relevant configuration change | | |
| | Validation | Stale or invalid output | | |

Correlate identifiers, actor, affected resource and time. Temporal proximity alone does not prove causation.

## 15. Lab K — Staging alert delivery test

1. Record healthy telemetry and test alert configuration.
2. Trigger one approved benign synthetic failure.
3. Confirm failed run appears in source metadata.
4. Confirm SLI ledger marks it bad or late as appropriate.
5. Confirm the alert rule evaluates and fires.
6. Verify notification contains service, run, impact, UTC, owner and runbook.
7. Verify primary and fallback routing in the test channel.
8. Recover the synthetic workload safely.
9. Confirm correct output and alert resolution.
10. Restore test settings and preserve sanitized evidence.

Do not use a production paging route without explicit coordination.

## 16. Lab L — Alert completeness and duplication

~~~powershell
$qualifying = 20
$delivered = 19
$duplicatePages = 2

[pscustomobject]@{
  QualifyingIncidents = $qualifying
  DeliveredWithinTarget = $delivered
  DeliveryPct = [math]::Round(100 * $delivered / $qualifying, 2)
  DuplicatePages = $duplicatePages
}
~~~

Measure collection latency, alert evaluation latency, notification latency and acknowledgement separately.

## 17. Lab M — Evidence capture automation

~~~powershell
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
$stamp = (Get-Date).ToUniversalTime().ToString("yyyyMMddTHHmmssZ")
$folder = Join-Path $PWD "databricks-monitoring-acceptance-$stamp"
New-Item -ItemType Directory -Path $folder | Out-Null

databricks jobs list --output json |
  Out-File (Join-Path $folder "jobs.json") -Encoding utf8

databricks jobs list-runs --limit 25 --output json |
  Out-File (Join-Path $folder "runs.json") -Encoding utf8

Write-Host "Evidence folder: $folder"
Write-Host "Review and redact all files before sharing."
~~~

The script is intentionally read-only. Store evidence in an approved secured location; do not commit raw operational logs to public GitHub.

## 18. Simulation A — Critical job fails without an alert

**Inject:** staging job failure.

**Diagnose:** source events, ingestion, eligibility, alert rule and routing.

**Recover:** repair test telemetry/rule, rerun safely.

**Pass:** one actionable alert, validated output and documented detection delay.

## 19. Simulation B — Compute startup exceeds deadline

**Inject/tabletop:** provisioning delay due to capacity or quota.

**Diagnose:** queue, cluster events, cloud capacity and initialization.

**Recover:** approved compute/scheduling change.

**Pass:** startup signal and deadline risk visible; no false Spark bottleneck attribution.

## 20. Simulation C — Spark skew and tail latency

**Inject/tabletop:** synthetic workload with one slow task.

**Diagnose:** Spark stage/task distribution, shuffle and spill.

**Recover:** staging-tested layout or workload correction.

**Pass:** task-level evidence supports cause and SLO improvement.

## 21. Simulation D — IAM or storage failure

**Tabletop or controlled staging denial only.**

**Diagnose:** Unity Catalog, cloud IAM, network, storage metrics and audit trail.

**Recover:** restore approved least privilege; never open public access as workaround.

**Pass:** authorized workload succeeds, unauthorized access remains blocked.

## 22. Simulation E — Audit delivery gap

**Inject/tabletop:** delayed or missing synthetic audit event.

**Diagnose:** source coverage, ingestion delay, filters, sinks and permissions.

**Recover:** restore supported collection and replay where available.

**Pass:** event delivery within documented target, with no false no-activity claim.

## 23. Simulation F — Green job with invalid data

**Inject:** approved staging job returns success but validation invariant fails.

**Diagnose:** data quality, run metadata, SLI definition and downstream impact.

**Recover:** correct synthetic output and validation gates.

**Pass:** service remains unhealthy until correctness and freshness pass.

## 24. Incident response decision tree

    SLO breach or test failure
              |
      Source telemetry present?
       no -> coverage / ingestion
       yes
              |
      Run/task failed or late?
       yes -> compute / Spark / dependency
       no
              |
      Output correct and fresh?
       no -> data reliability incident
       yes
              |
      Alert fired and routed?
       no -> alert / escalation defect
       yes
              |
      Recovery validated?
       no -> continue incident
       yes -> document and close

## 25. SRE incident runbook

1. Declare severity based on business impact and affected SLO.
2. Record UTC start, workspace, job/run and affected dataset.
3. Preserve source evidence and note collection delays.
4. Correlate Databricks compute, Spark, audit and cloud telemetry.
5. Identify the failed dependency and recent changes.
6. Apply least-risk approved mitigation with rollback.
7. Validate run completion, data correctness, freshness and cost.
8. Confirm alert resolution and notification delivery.
9. Communicate remaining risk and recovery status.
10. Document RCA, error-budget impact and monitoring gaps.

## 26. Telemetry security and retention review

Validate:
- Least privilege for audit and billing system tables.
- Redaction of tokens, credentials, sensitive SQL and regulated data.
- Encryption and residency for central log destinations.
- Log retention matched to incident and compliance needs.
- Ingestion delay, sampling and replay caveats.
- Collector and alert pipeline availability.
- Storage/ingestion cost and cardinality guardrails.
- Access-controlled evidence and incident records.

## 27. Failure-mode coverage

| Failure class | Primary detector | Corroborating evidence |
|---|---|---|
| Job terminal failure | Run outcome SLI | Task errors |
| Deadline miss | Deadline SLI | Queue/startup timeline |
| Executor OOM | Runtime/compute metrics | Executor logs |
| Data skew | Task tail latency | Shuffle/input distribution |
| Storage throttling | Cloud storage metrics | Spark task errors |
| IAM denial | Error and audit events | Cloud IAM audit |
| Stale data | Freshness SLI | Table update history |
| Invalid output | Correctness validation | Sample/invariant checks |
| Audit gap | Synthetic heartbeat | Collector lag and sink health |
| Alert routing defect | Test delivery SLI | Notification/acknowledgement logs |

## 28. Acceptance scorecard

| Requirement | Result | Evidence / owner |
|---|---|---|
| Critical service inventory | Not assessed | |
| SLI definitions and exclusions | Not assessed | |
| SLO and error budget calculation | Not assessed | |
| Job and task telemetry | Not assessed | |
| Compute and Spark diagnostics | Not assessed | |
| Audit and cloud log coverage | Not assessed | |
| UTC correlation and retention | Not assessed | |
| Alert evaluation and delivery | Not assessed | |
| Primary/fallback escalation | Not assessed | |
| Deduplication and alert noise | Not assessed | |
| Data correctness/freshness gating | Not assessed | |
| Six simulations/tabletops | Not assessed | |
| Recovery and rollback runbooks | Not assessed | |
| Cost and security approval | Not assessed | |

## 29. GO / CONDITIONAL GO / NO-GO

**GO:** All critical service signals are measurable; staging failures are detected and routed; recovery is validated for output correctness and freshness; no unresolved critical coverage gaps.

**CONDITIONAL GO:** Only documented noncritical gaps with owner, mitigation, accepted risk and due date.

**NO-GO:** Silent critical failures, unowned alerts, missing correctness checks, absent essential audit evidence, unsafe remediation automation or unverified recovery.

## 30. Final evidence record

    Assessment date/time UTC:
    Workspace / cloud / region:
    Service and business owner:
    Jobs / compute / dataset:
    Telemetry sources and limitations:
    SLI definitions / SLO targets:
    Error budget result:
    Alert test / delivery / escalation:
    Spark and cloud correlation:
    Simulation A-F outcomes:
    Data correctness and freshness evidence:
    Recovery time and rollback:
    Security and cost review:
    Open risks / owners / deadlines:
    Decision: GO / CONDITIONAL GO / NO-GO
    Approvers:

## 31. Review questions

1. Why must the monitoring capstone test invalid output even when a job succeeds?
2. How do you demonstrate that a critical incident is not silently missed?
3. What evidence separates slow provisioning from slow Spark execution?
4. Why must audit telemetry be assessed alongside cloud-provider logs?
5. Which conditions should force a production NO-GO decision?

## 32. Official references

- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks compute metrics: https://docs.databricks.com/aws/en/compute/cluster-metrics
- Databricks audit logs: https://docs.databricks.com/aws/en/admin/account-settings/audit-logs
- Apache Spark monitoring: https://spark.apache.org/docs/latest/monitoring.html
- Google SRE workbook: https://sre.google/workbook/alerting-on-slos/
- AWS CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs

**Part 07 complete. Next:** Chapter 036 — Compute Saturation and Resource Contention (Part 08 — Performance and Troubleshooting).
