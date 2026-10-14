# Chapter 023 — Workflow Monitoring, Alerting and SLA Management

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 05 — Jobs, Workflows and Production Reliability  
**Clouds:** AWS, Azure, GCP  
**Audience:** SRE, Platform Engineering, Data Platform Operations  
**Prerequisites:** Chapters 021–022  
**Lab mode:** Read-only production diagnostics and synthetic staging alert tests

## 1. Mission

Create an operational monitoring system for Databricks Jobs and Lakeflow Jobs that measures user-facing reliability, predicts missed deadlines, routes actionable alerts and supports incident review. The focus is **operational reliability**, not developing data transformations.

### Learning objectives

1. Identify trustworthy workflow telemetry sources and their limitations.
2. Distinguish SLIs, SLOs, contractual SLAs and internal operational targets.
3. Calculate completion, duration, queueing, retry and deadline adherence.
4. Design missing-run, deadline-risk, failure and alert-pipeline health detectors.
5. Validate notifications and escalation using safe synthetic tests.
6. Correlate workflow performance with compute, IAM and external dependencies.
7. Produce an auditable workflow reliability report.

## 2. Monitoring architecture

    Scheduler / event source
              |
         Databricks Jobs
              |
      Runs, tasks and events
              |
      +-------+---------+
      |                 |
    Jobs API      System tables
      |           (if enabled)
      +-------+---------+
              |
     Metrics / dashboards / SIEM
              |
     SLI evaluation + alert rules
              |
      On-call / incident manager
              |
      Recovery and RCA evidence

Cloud provider metrics and external dependency logs are separate sources. No single Databricks signal guarantees end-to-end data correctness.

## 3. Workflow reliability signals

| Signal | Meaning | Source |
|---|---|---|
| Trigger occurrence | Was an expected run created? | Job schedule + run history |
| Run admission delay | Trigger to actual admission | Job run timestamps |
| Compute startup latency | Provisioning and startup time | Cluster events / run timeline |
| Task duration | Execution time for a task | Task runs / Spark UI |
| End-to-end completion | Business output ready | Job + output validation |
| Retry/repair count | Recovery attempts | Run and repair history |
| Failure category | Reason for failure | Run/task logs |
| Missing output | Silent functional failure | Independent data check |
| Alert delivery | Did the owner receive notification? | Notification/SIEM telemetry |
| Cost per successful run | Efficiency | Databricks usage + cloud costs |

## 4. SLIs, SLOs and SLAs

**SLI:** measured indicator of service behavior.  
**SLO:** internal target for an SLI over a defined measurement window.  
**SLA:** a contractual commitment that may include remedies; do not use the terms interchangeably.

Example, **illustrative only**:

    Service: demo-daily-pipeline
    Business deadline: 07:00 UTC
    SLI: validated output available by deadline
    SLO: >= 99% of eligible daily runs in rolling 30 days
    Measurement owner: platform-sre
    Exclusions: explicitly documented approved exclusions only

A green Databricks job does not prove that the business output is complete or correct.

## 5. Define denominator and measurement windows

For every SLI document:
- Eligible run definition, including schedules skipped for approved reasons.
- Timezone and daylight-saving rules.
- Deadline and measurement window.
- Treatment of canceled, retried, repaired and manually triggered runs.
- Handling of missing telemetry and delayed ingestion.
- Source of truth for business output validation.
- Error-budget owner and escalation policy.

Do not quietly remove failed or missing runs from the denominator.

## 6. Suggested dashboard panels

1. Critical workflow completion against deadline.
2. Expected versus observed scheduled runs.
3. Job success and failure trends.
4. p50/p95/p99 end-to-end duration by comparable job class.
5. Queue wait and compute startup latency.
6. Retry and repair rates.
7. Top recurring failure classes.
8. Missing or stale monitoring events.
9. Alert acknowledgement and escalation.
10. Cost per validated successful run.

Segment by workspace, environment, job criticality and owner without exposing secrets or sensitive IDs.

## 7. Lab A — Windows PowerShell CLI preflight

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

Confirm the profile and workspace host. The listing can be permission-filtered and paginated; the sample is not a complete historical inventory.

## 8. Lab B — Inspect one authorized job

~~~powershell
# Replace illustrative IDs with an approved staging job and run.
databricks jobs get --job-id 12345 --output json
databricks jobs list-runs --job-id 12345 --limit 25 --output json
databricks jobs get-run --run-id 67890 --output json
~~~

Verify installed CLI flags with help. Record schedule/timezone, job owner, run-as identity, task DAG, queueing/concurrency, retry settings, notifications and run timestamps.

### Evidence worksheet

| Field | Value |
|---|---|
| Job label / owner | |
| Workspace / cloud / region | |
| Trigger and timezone | |
| Expected start UTC | |
| Actual run start UTC | |
| Queue delay | |
| Compute startup delay | |
| Task execution time | |
| Deadline and output readiness | |
| Retry/repair attempts | |
| Notification destination | |

## 9. Lab C — Offline SLI calculations

This PowerShell lab uses **synthetic observations**, not live job data.

~~~powershell
$runs = @(
  [pscustomobject]@{ Name="run1"; DurationMin=18; DeadlineMet=$true;  Validated=$true },
  [pscustomobject]@{ Name="run2"; DurationMin=21; DeadlineMet=$true;  Validated=$true },
  [pscustomobject]@{ Name="run3"; DurationMin=47; DeadlineMet=$false; Validated=$true },
  [pscustomobject]@{ Name="run4"; DurationMin=19; DeadlineMet=$true;  Validated=$true },
  [pscustomobject]@{ Name="run5"; DurationMin=15; DeadlineMet=$true;  Validated=$false }
)
$total = $runs.Count
$onTimeValid = @($runs | Where-Object { $_.DeadlineMet -and $_.Validated }).Count
$adherence = 100.0 * $onTimeValid / $total
$sorted = @($runs | Sort-Object DurationMin)
$p95Index = [math]::Ceiling(0.95 * $total) - 1
[pscustomobject]@{
  EligibleRuns = $total
  OnTimeValidated = $onTimeValid
  DeadlineAdherencePct = [math]::Round($adherence, 2)
  P95DurationMinutes = $sorted[$p95Index].DurationMin
}
~~~

For small samples, percentiles are unstable. Use a consistent percentile definition and sufficiently large observation windows for production reporting.

## 10. Lab D — System table exploration

Where enabled and authorized, Databricks job system tables can provide historical job and run timeline information. Available tables, schemas, privileges, retention and account/region scope vary. Inspect the current schema first.

~~~sql
SHOW TABLES IN system.lakeflow;
DESCRIBE TABLE system.lakeflow.job_run_timeline;
~~~

Do not assume every workspace exposes these tables or that a single row represents an entire completed run. Timeline tables may contain multiple time slices per run and require aggregation.

### Safe exploratory sample

~~~sql
SELECT *
FROM system.lakeflow.job_run_timeline
LIMIT 10;
~~~

Before building production dashboards, validate timestamp semantics, run identifiers, event delivery lag and status values against the official schema.

## 11. Lab E — Missing-run detection design

A missing-run detector needs an independent expected-run calendar, not just observed runs.

    expected = schedule occurrences in UTC
    observed = actual job runs within tolerance
    missing = expected - matched observed

Consider:
- Cron timezone and DST.
- Paused or disabled schedules.
- Maintenance exclusions.
- Event-trigger delivery semantics.
- Run history retention and pagination.
- Late-arriving telemetry.
- Manual backfill versus original scheduled run.

A successful notification from Databricks does not replace independent missing-run detection.

## 12. Lab F — Deadline risk prediction

For a critical run, estimate:

    projected_completion =
       current_time
       + expected_remaining_queue
       + expected_compute_startup
       + expected_remaining_task_execution
       + downstream_validation_buffer

Alert if projected completion exceeds the business deadline with sufficient confidence. Use historical distributions from comparable workloads and state uncertainty explicitly. Avoid paging on noisy predictions with no operator action.

## 13. Lab G — Synthetic alert validation

Use an approved staging job or offline event fixture.

1. Select a nonproduction alert route and owner.
2. Generate a synthetic failed-run event.
3. Verify the failure is ingested and classified.
4. Confirm the correct severity, owner and runbook.
5. Verify deduplication and acknowledgement.
6. Generate a missing-run or deadline-risk synthetic event.
7. Test escalation if the primary owner does not acknowledge.
8. Confirm recovery clears the incident appropriately.
9. Restore original staging alert settings and retain sanitized evidence.

Do not intentionally fail production workflows or spam real on-call rotations.

## 14. Alert policy

| Condition | Suggested severity basis | Operator action |
|---|---|---|
| Critical job failed | Immediate user impact | Triage failed task |
| Missing scheduled run | Business deadline risk | Check trigger and backfill |
| Queue backlog | Projected deadline miss | Investigate concurrency/capacity |
| Repeated retries | Shared dependency or storm | Apply retry circuit breaker |
| Output validation failed | Data correctness incident | Contain downstream impact |
| Monitoring ingestion stale | Detection blind spot | Restore telemetry |
| Notification delivery failed | Incident response gap | Repair route |

Use SLOs, business impact and runbook actions to set actual severity; not every individual retry warrants paging.

## 15. Incident scenario A — Green job, missing output

**Triage:** validate expected partitions, row counts, freshness, downstream delivery and task side effects.

**Recovery:** reconcile data and perform controlled repair/backfill only after idempotency review.

**Acceptance:** output validated and consumers recovered.

## 16. Incident scenario B — Job missed deadline due to queueing

**Triage:** compare expected trigger, admission, concurrency, cloud quota and compute startup.

**Recovery:** address confirmed blocker with approved capacity or scheduling adjustment.

**Acceptance:** queue delay and deadline adherence return to baseline.

## 17. Incident scenario C — Retry storm hides underlying outage

**Triage:** correlate failure signatures, attempt counts, shared dependencies and incremental compute cost.

**Recovery:** throttle repeated attempts and repair underlying dependency.

**Acceptance:** retries normalize and outputs are not duplicated.

## 18. Incident scenario D — Missing-run alert does not fire

**Triage:** verify expected-run calendar, scheduler state, monitoring ingestion, alert query and delivery route.

**Recovery:** repair independent detection and validate with a synthetic missing occurrence.

**Acceptance:** notification delivered within agreed detection window.

## 19. Incident scenario E — SLO dashboard reports false success

**Triage:** examine eligibility denominator, canceled runs, repairs, late telemetry and missing output validation.

**Recovery:** correct SLI computation and backfill reports from trusted records.

**Acceptance:** independently reconciled counts and documented measurement semantics.

## 20. Incident scenario F — Alert storm during cloud outage

**Triage:** identify correlated root cause, repeated job failures, deduplication gaps and on-call load.

**Recovery:** aggregate alerts by shared incident while preserving visibility into affected critical workflows.

**Acceptance:** actionable paging and complete impact inventory.

## 21. Troubleshooting decision tree

    Workflow reliability alarm
          |
    Did expected run exist?
      no -> trigger / missing-run monitor
      yes
          |
    Did run start on time?
      no -> queue / concurrency / compute
      yes
          |
    Did tasks complete?
      no -> task / identity / dependency
      yes
          |
    Was output validated by deadline?
      no -> correctness / downstream / lateness
      yes
          |
    Was monitoring and notification healthy?
      no -> observability pipeline
      yes -> close with evidence

## 22. Production incident runbook

1. Detect user impact or deadline risk.
2. Identify affected jobs, owners, cloud, workspace and dependencies.
3. Preserve UTC timeline, run history and monitoring evidence.
4. Separate trigger, queue, compute, task and output validation.
5. Classify root cause and check shared incidents.
6. Assess idempotency before repair/rerun/backfill.
7. Apply smallest approved mitigation with rollback.
8. Validate output, downstream state and deadline recovery.
9. Communicate status and create corrective actions.
10. Review whether alerts fired early enough and reached the right owner.

## 23. Change and rollback

Before modifying SLO logic, alert rules or workflow notification settings:
- Record the existing query, filters, routing and thresholds.
- Test synthetic success, failure, missing-run and recovery cases.
- Review expected false-positive and false-negative impact.
- Stage changes with limited scope and parallel comparison.
- Roll back if coverage or routing degrades.
- Preserve prior incident evidence and historical reports.

## 24. Acceptance checklist

- [ ] Critical workflow owners and business deadlines documented.
- [ ] SLIs, SLOs and contractual SLA boundaries distinguished.
- [ ] Expected-run calendar established.
- [ ] Queue/startup/execution metrics separated.
- [ ] Output validation included in success definition.
- [ ] Retry/repair and missing-run signals monitored.
- [ ] System table scope and data freshness validated.
- [ ] Synthetic alert delivery and escalation tested.
- [ ] Six incident scenarios completed.
- [ ] Runbook and rollback reviewed.
- [ ] No production jobs or alert routes modified during lab.
- [ ] Evidence sanitized and access-controlled.

## 25. Review questions

1. Why can a green job still violate a business SLO?
2. Why is an independent expected-run calendar necessary?
3. How can an incorrect denominator inflate completion reliability?
4. What makes a deadline-risk alert actionable?
5. Why must alert delivery itself be monitored?

## 26. References

- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks CLI Jobs: https://docs.databricks.com/aws/en/dev-tools/cli/reference/jobs-commands
- Azure Databricks Jobs: https://learn.microsoft.com/azure/databricks/jobs/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 024 — Workflow Capacity, Concurrency and Performance Operations.
