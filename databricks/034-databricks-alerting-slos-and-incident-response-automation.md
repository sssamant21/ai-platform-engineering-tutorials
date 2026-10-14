# Chapter 034 — Databricks Alerting, SLOs and Incident Response Automation

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 07 — Observability  
**Clouds:** AWS, Azure, GCP  
**Prerequisites:** Chapters 031–033  
**Audience:** SRE, Platform Engineering, DBRE, Security Operations  
**Execution policy:** Production investigation is read-only. Synthetic failures, alert configuration and automation tests are staging-only and require approval.

## 1. Objectives

Design a measurable and actionable Databricks reliability program, from service-level indicators through alert routing, escalation, incident response and guarded automation.

By the end, operators can:

- Define service boundaries, critical user journeys and measurable SLIs.
- Calculate SLO compliance and error-budget consumption.
- Distinguish availability, timeliness, freshness and correctness.
- Design multi-window burn-rate alerts and symptom-based paging.
- Route incidents to owners with runbooks, evidence and escalation.
- Automate low-risk containment and recovery with approval gates.
- Correlate Databricks events with AWS, Azure and GCP telemetry.
- Prove alert delivery, deduplication, recovery and auditability in staging.

## 2. Reference architecture

    Business service / data consumer
                  |
       Success + timeliness + freshness
                  |
           SLI computation
                  |
        SLO and error budget
                  |
       Alert evaluation engine
                  |
       Deduplication / grouping
                  |
     On-call routing and escalation
                  |
     Incident command / runbooks
                  |
      Human-approved automation
                  |
       Recovery and validation
                  |
     Post-incident improvement

Telemetry inputs:

    Databricks Jobs + SQL + compute + audit
                    |
         System tables / APIs / logs
                    |
      AWS CloudWatch / Azure Monitor /
           Google Cloud Monitoring
                    |
       Metrics store and event pipeline

An alert is only useful if it corresponds to user impact or a credible near-term risk and has a responder who can act.

## 3. Define the service before the metric

A Databricks workspace is not itself a complete business service. A critical service may consist of scheduled jobs, SQL warehouses, Unity Catalog objects, cloud storage, cloud networking and downstream consumers.

For each service record:

| Attribute | Example |
|---|---|
| Service | demo-hourly-reporting |
| Owner | data-platform-oncall |
| Tier | Critical |
| User promise | Validated dataset by minute 20 |
| Dependency | Jobs, compute, Delta, cloud storage |
| Recovery target | Approved business RTO |
| Source of truth | Job outcomes plus data validation |
| Exclusions | Explicitly approved maintenance only |

Avoid reporting job success as proof of data correctness.

## 4. SLIs and measurement boundaries

| SLI | Definition | Failure mode |
|---|---|---|
| Run success | Successful eligible runs / eligible runs | Terminal failure |
| Deadline compliance | Validated runs on time / eligible runs | Late result |
| Freshness | Age since last validated dataset update | Stale data |
| Correctness | Validated outputs / evaluated outputs | Bad or duplicate records |
| Availability | Successful authorized requests / eligible requests | Failed user request |
| Recovery time | Impact start to validated recovery | Prolonged incident |
| Alert delivery | Delivered qualifying alerts / injected qualifying cases | Silent monitoring failure |

Eligibility rules must address skipped runs, cancellations, retries, manual reruns, partial task success and maintenance. Keep the denominator stable and documented.

## 5. SLO and error-budget mathematics

For an event-based SLO:

    SLO compliance = good eligible events / total eligible events

    Allowed bad events = eligible events × (1 - target fraction)

    Budget used = observed bad events / allowed bad events

For example, at 99.5% success over 10,000 eligible events:

- Allowed failures = 50.
- If 20 fail, 40% of the event budget is consumed.
- If 60 fail, the budget is exceeded by 20%.

Event-count budgets do not express downtime minutes. For time-based SLOs, define observation periods and downtime rules separately.

## 6. Lab A — Windows PowerShell SLO calculator

~~~powershell
$eligible = 10000
$good = 9980
$target = 0.995
$bad = $eligible - $good
$allowedBad = $eligible * (1 - $target)

[pscustomobject]@{
  Eligible = $eligible
  Good = $good
  CompliancePct = [math]::Round(100 * $good / $eligible, 3)
  AllowedBad = $allowedBad
  ObservedBad = $bad
  BudgetConsumedPct = [math]::Round(100 * $bad / $allowedBad, 1)
}
~~~

These are illustrative values, not a universal target. For small denominators, avoid misleading fractional allowed-event interpretations; document the counting policy.

## 7. Error-budget burn rates

Burn rate compares observed bad-event fraction with the allowed bad-event fraction:

    burn rate = observed error fraction / (1 - SLO target)

A burn rate of 1 consumes budget at its steady-state allowed pace. A rate of 10 consumes it ten times faster.

### Alerting approach

- Fast burn: short window combined with a longer confirming window.
- Slow burn: sustained degradation over a longer observation period.
- Immediate page: confirmed correctness/security incident or hard business deadline risk.
- Ticket: nonurgent drift, trend or low-impact recoverable error.

Window sizes and thresholds depend on the SLO period, event frequency and business impact. Low-volume scheduled jobs often need deadline-based alerts rather than statistical burn-rate paging.

## 8. Lab B — Offline burn-rate model

~~~powershell
$target = 0.999
$windowEligible = 1000
$windowFailed = 8
$errorFraction = $windowFailed / $windowEligible
$burnRate = $errorFraction / (1 - $target)

[pscustomobject]@{
  TargetPct = 100 * $target
  WindowErrors = $windowFailed
  ErrorFractionPct = [math]::Round(100 * $errorFraction, 2)
  BurnRate = [math]::Round($burnRate, 2)
}
~~~

This is a synthetic illustration. In production, account for sparse events, ingestion delay and alert evaluation windows.

## 9. Telemetry sources and caveats

| Signal | Primary source | Caveat |
|---|---|---|
| Run lifecycle | Databricks Jobs API | Permissions, pagination, timing |
| Task failures | Run details, logs | Retries can mask intermediate failures |
| SQL latency | Query history/warehouse metrics | Availability and retention vary |
| Compute health | Cluster metrics and Spark UI | Not a business outcome |
| Audit actions | System audit tables and cloud audit | Delayed, scoped and access-controlled |
| Billing usage | System billing tables | Usage units are not necessarily dollars |
| Storage/network | Cloud-native metrics | Cloud dependency is not identical to job SLI |

System table schemas and availability differ by cloud, account, region and permissions. Discover before querying.

## 10. Lab C — Databricks CLI preflight and job inventory

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks jobs get-run --help
~~~

Confirm workspace and identity before inspecting run details. Do not export credentials into shared logs.

## 11. Lab D — Discover SQL telemetry

~~~sql
SHOW SCHEMAS IN system;
SHOW TABLES IN system.lakeflow;
SHOW TABLES IN system.access;
SHOW TABLES IN system.billing;
~~~

Inspect schemas before selecting columns:

~~~sql
-- Run only if present and authorized.
DESCRIBE TABLE system.lakeflow.job_run_timeline;
DESCRIBE TABLE system.lakeflow.job_task_run_timeline;
~~~

Availability and schema can change; some accounts may not expose these tables. Use Jobs API or approved operational telemetry as an alternative. Do not assume a missing row is a successful run.

## 12. Lab E — Build a reliability event ledger

Use an authorized staging workflow to produce a normalized event dataset with:

| Field | Meaning |
|---|---|
| event_time_utc | Timestamp of observed event |
| workspace_id | Workspace boundary |
| job_id / run_id | Stable execution identifiers |
| service_name | Business service mapping |
| eligible | Included in SLO denominator |
| successful | Terminal and validated outcome |
| deadline_met | Finished and validated on time |
| error_category | Compute, IAM, data, network, platform |
| source | API, system table or collector |
| ingestion_time_utc | Delay and completeness evaluation |

Deduplicate by stable source and run identifiers. Define how repeated attempts and repair runs map to the eligible run.

## 13. Lab F — SQL SLO query on synthetic data

~~~sql
WITH events AS (
  SELECT * FROM VALUES
    ('demo-hourly', true, true, true),
    ('demo-hourly', true, true, false),
    ('demo-hourly', true, false, false),
    ('demo-hourly', false, false, false)
  AS t(service, eligible, successful, deadline_met)
)
SELECT
  service,
  COUNT_IF(eligible) AS eligible_runs,
  COUNT_IF(eligible AND successful) AS successful_runs,
  COUNT_IF(eligible AND deadline_met) AS on_time_runs,
  ROUND(
    100.0 * COUNT_IF(eligible AND successful) /
    NULLIF(COUNT_IF(eligible), 0), 2
  ) AS success_pct
FROM events
GROUP BY service;
~~~

The fourth row is excluded. This is an illustrative SLI calculation, not a replacement for authoritative production run semantics.

## 14. Alert severity and routing

| Severity | Example | Response |
|---|---|---|
| SEV1 | Corrupt output used by critical consumers | Immediate incident command |
| SEV2 | Critical pipeline missing delivery deadline | Page service owner |
| SEV3 | Persistent noncritical failure or slow burn | Business-hours escalation |
| Informational | Expected retry or brief transient spike | Dashboard/ticket |

Severity depends on impact, not merely the error message. Ensure fallback owners and escalation timeouts are documented.

## 15. Alert payload contract

Every actionable alert should include:
- Service, environment, workspace and cloud region.
- SLI/SLO symptom and measured window.
- Impacted job, run, task or dataset.
- UTC incident start and last observation.
- Runbook and dashboard links.
- Owning team and escalation policy.
- Correlated recent changes.
- Data freshness/correctness status.
- Whether automated mitigation has run.

Exclude secrets and sensitive data from notifications.

## 16. Deduplication, inhibition and maintenance

Group repeated events by service, incident class and affected dependency. Suppress derivative alerts when a verified parent outage already explains them. Time-bound maintenance silences require owner, reason and expiration.

Never suppress confirmed data corruption or security incidents merely because a scheduled maintenance window exists.

## 17. Automation maturity levels

| Level | Behavior | Guardrail |
|---|---|---|
| 0 | Manual investigation | Runbook and owner |
| 1 | Automated evidence gathering | Read-only permissions |
| 2 | Suggested remediation | Human approval |
| 3 | Approved bounded retry/restart | Idempotency and rate limits |
| 4 | Autonomous corrective action | Formal risk acceptance and proven rollback |

Prefer automated diagnostics before automated mutation.

## 18. Safe recovery orchestration

    Alert qualified
           |
    Validate current state
           |
    Is impact still present?
       no -> resolve
       yes
           |
    Check recent mitigation attempts
           |
    Idempotent action permitted?
       no -> page owner
       yes
           |
    Approval / policy gate
           |
    Execute bounded mitigation
           |
    Validate output and SLO
       pass -> close with evidence
       fail -> rollback/escalate

Required controls: least privilege, rate limits, concurrency locks, audit trail, dry-run where possible, timeout, kill switch and rollback owner.

## 19. Lab G — Read-only incident evidence collector

~~~powershell
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
$stamp = Get-Date -AsUTC -Format "yyyyMMddTHHmmssZ"
$folder = Join-Path $PWD "incident-evidence-$stamp"
New-Item -ItemType Directory -Path $folder | Out-Null

databricks jobs list --output json |
  Out-File (Join-Path $folder "jobs.json") -Encoding utf8

databricks jobs list-runs --limit 25 --output json |
  Out-File (Join-Path $folder "recent-runs.json") -Encoding utf8

Write-Host "Review and redact evidence before sharing: $folder"
~~~

The collector intentionally does not capture secrets, driver log bodies or unrestricted audit records. Treat all output as potentially sensitive and apply approved access controls.

## 20. Lab H — Cloud alert integration inventory

### AWS

~~~powershell
aws sts get-caller-identity
aws configure get region
aws cloudwatch describe-alarms --max-records 25
~~~

Review CloudWatch alarms, notification routing, cloud quotas, EC2/EBS/network and S3 metrics where enabled.

### Azure

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az monitor metrics alert list --resource-group demo-staging-rg -o json
~~~

Review Azure Monitor alert rules, action groups, Activity Logs, storage metrics and private networking.

### GCP

~~~powershell
gcloud config get-value project
gcloud alpha monitoring policies --help
~~~

Use approved Cloud Monitoring console/API tooling to inspect alerting policies and notification channels. The alpha command is a discovery aid only; availability depends on installed SDK components.

Do not assume cloud-native alerts automatically represent Databricks job SLOs; connect them to service impact.

## 21. Lab I — Controlled staging alert simulation

Preconditions: approved staging job, synthetic input, isolated test notification route, cost cap and rollback owner.

1. Record a healthy baseline and SLO event.
2. Define a test-only alert for a failed eligible run.
3. Trigger one safe synthetic failure in staging.
4. Verify source event, collector, SLI ledger and alert evaluation.
5. Confirm one actionable notification with runbook and owner.
6. Verify escalation and acknowledgement using the test channel.
7. Correct the synthetic failure and rerun if safe and idempotent.
8. Validate correct output, recovery event and alert resolution.
9. Remove test-only configuration and preserve sanitized evidence.

Do not inject failures or disable monitoring in production.

## 22. Lab J — Alert delivery SLI

~~~powershell
$qualifyingIncidents = 25
$deliveredWithinTarget = 24
$duplicatePages = 2
[pscustomobject]@{
  Qualifying = $qualifyingIncidents
  OnTimeDelivery = $deliveredWithinTarget
  DeliveryPct = [math]::Round(
    100 * $deliveredWithinTarget / $qualifyingIncidents, 2)
  DuplicatePages = $duplicatePages
}
~~~

Measure notification delay, acknowledgement delay, deduplication quality and runbook usability separately.

## 23. Incident A — Silent job failures

**Symptoms:** repeated failures without notification.

**Investigate:** source events, collection gaps, SLO eligibility filters, alert rule and notification route.

**Recover:** repair monitoring path and inspect missed impact.

**Acceptance:** staging synthetic failure produces one timely actionable page.

## 24. Incident B — Alert storm during shared storage outage

**Symptoms:** many jobs page simultaneously with the same storage error.

**Investigate:** common cloud dependency, workspace/region and recent change.

**Recover:** group child alerts under parent incident; restore dependency.

**Acceptance:** incident remains visible while redundant paging is reduced.

## 25. Incident C — Retry automation amplifies outage

**Symptoms:** repeated failed reruns increase compute cost and dependency load.

**Investigate:** retry bounds, idempotency, concurrent automation, original error class.

**Recover:** stop retry loop, contain impact, apply human-approved correction.

**Acceptance:** no duplicate output, bounded spend and documented guardrail.

## 26. Incident D — SLO dashboard shows false green

**Symptoms:** no failures reported despite missed deadlines.

**Investigate:** eligibility denominator, late ingestion, missing events, success-only filtering and incorrect handling of retries.

**Recover:** repair event ledger and backfill only with defensible evidence.

**Acceptance:** independent sample agrees with SLO report.

## 27. Incident E — Wrong-team escalation

**Symptoms:** critical alerts route to unstaffed or unrelated responder.

**Investigate:** ownership metadata, notification policies, fallback and schedule changes.

**Recover:** correct routing and test acknowledgement chain.

**Acceptance:** simulated page reaches primary and backup responders.

## 28. Incident F — Automated restart masks data corruption

**Symptoms:** run status returns green but dataset contains invalid output.

**Investigate:** data validation, commit semantics, idempotency and downstream consumption.

**Recover:** stop unsafe automation, contain affected data and use approved correction/restore.

**Acceptance:** validated correctness, freshness and durable prevention.

## 29. Incident command runbook

1. Declare severity and identify affected business SLOs.
2. Record UTC start, workspace, region, job/run IDs and responders.
3. Confirm alert evidence and telemetry completeness.
4. Correlate compute, Delta, IAM, networking and cloud provider signals.
5. Contain damage and disable unsafe automation if necessary.
6. Choose least-risk mitigation with change approval and rollback.
7. Validate job status, output correctness, freshness and cost.
8. Confirm alert resolution and absence of duplicate side effects.
9. Communicate status and remaining risk.
10. Record RCA, error-budget impact, detection gap and follow-up owners.

## 30. Production rollout strategy

### Phase 1 — Instrument

Define service catalog, owners, SLI event schema and data quality checks.

### Phase 2 — Observe

Collect baseline data, identify sparse workloads and quantify ingestion lag.

### Phase 3 — Alert

Deploy symptom-based alerts, routing, escalation and maintenance controls.

### Phase 4 — Rehearse

Run approved staging failure and alert acceptance exercises.

### Phase 5 — Automate

Introduce read-only enrichment first; add bounded remediation only after idempotency, rollback and approval are demonstrated.

## 31. Acceptance scorecard

| Control | Status | Evidence |
|---|---|---|
| Critical services and owners mapped | Not assessed | |
| Eligibility and SLI definitions approved | Not assessed | |
| SLO targets and error budgets agreed | Not assessed | |
| Source telemetry and delay documented | Not assessed | |
| Alert rules tested in staging | Not assessed | |
| Escalation and backup routing validated | Not assessed | |
| Deduplication and inhibition tested | Not assessed | |
| Read-only evidence automation tested | Not assessed | |
| Mutating automation safety reviewed | Not assessed | |
| Data correctness included in recovery | Not assessed | |
| AWS/Azure/GCP dependency signals mapped | Not assessed | |
| Six incident scenarios reviewed | Not assessed | |

**GO:** critical SLOs measurable, alerts actionable and delivered, ownership clear, recovery safe.

**CONDITIONAL GO:** only documented noncritical gaps with owners and deadlines.

**NO-GO:** silent critical failures, missing ownership, unsafe unbounded automation, or unvalidated data correctness.

## 32. Troubleshooting quick reference

| Symptom | First checks | Avoid |
|---|---|---|
| No page | Event source, pipeline, rule, route | Assuming no incident |
| Excess pages | Grouping, retry storms, dependency | Blanket suppression |
| False green | Eligibility, missing runs, freshness | Success-only denominator |
| Automation loop | Limits, locks, idempotency | Unlimited reruns |
| Slow response | Escalation and runbook | Paging without owner |
| Green job, bad data | Validation and downstream impact | Closing on run state alone |

## 33. Review questions

1. How does a deadline SLO differ from a run-success SLO?
2. Why are statistical burn-rate alerts difficult for low-volume jobs?
3. What prevents a remediation bot from repeatedly restarting a broken workflow?
4. Why must recovery validate data correctness and not only terminal run status?
5. How do you prove that a missing page is a monitoring defect rather than a healthy service?

## 34. Official references

- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks audit logs: https://docs.databricks.com/aws/en/admin/account-settings/audit-logs
- Google SRE workbook — alerting on SLOs: https://sre.google/workbook/alerting-on-slos/
- AWS CloudWatch alarms: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html
- Azure Monitor alerts: https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-overview
- Google Cloud Monitoring alerting: https://cloud.google.com/monitoring/alerts

**Next:** Chapter 035 — Monitoring Acceptance Lab (Part 07 capstone).
