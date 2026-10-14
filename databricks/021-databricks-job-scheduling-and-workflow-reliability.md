# Chapter 021 — Databricks Job Scheduling and Workflow Reliability

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 05 — Jobs, Workflows and Production Reliability  
**Clouds:** AWS, Azure, GCP  
**Prerequisites:** Chapters 001–020  
**Lab safety:** Read-only production assessment; changes and fault injection only in approved staging.

## 1. Learning objectives

- Inventory Lakeflow Jobs definitions, schedules, task dependencies, run-as identities and compute.
- Explain scheduled triggers, file/event triggers, run admission, queueing and concurrency.
- Diagnose task failure, compute startup delays, retries, timeouts and missed deadlines.
- Design SLIs/SLOs and actionable alerts.
- Safely decide between repair, rerun and backfill after partial execution.
- Produce incident timelines, rollback procedures and operational acceptance evidence.

## 2. Reference architecture

    Cron / event / manual trigger
               |
         Databricks Jobs
               |
       Queue and concurrency
               |
         Task dependency DAG
          /           \
      Task A         Task B
          \           /
           Downstream task
                  |
        Jobs compute/serverless
                  |
      Outputs, events, run history
                  |
      SRE dashboards and alerts

Scheduling success is not workload success. Separate trigger delay, queue wait, compute startup, task execution and downstream output validation.

## 3. Reliability failure domains

| Layer | Symptom | Evidence |
|---|---|---|
| Trigger | Expected run absent | Schedule, timezone, pause state, source event |
| Admission | Run queued | Concurrency limits, queue state |
| Compute | Startup timeout | Cluster events, policy, IAM, quotas |
| Task | Execution failure | Task logs, Spark UI, data dependency |
| DAG | Downstream skipped | Dependency conditions and task states |
| External system | Transient errors | Storage/network/API telemetry |
| Notifications | Failure not reported | Notification destination and routing |

## 4. Scheduling governance

For each critical job record its owner, run-as identity, trigger, timezone, business deadline, concurrency configuration, compute mode, retries, timeout, task graph, upstream dependencies and alert destination.

Daylight-saving changes, paused schedules, late upstream data and overlapping runs can cause deadline misses even if Databricks itself is healthy. Validate schedules against business timezone requirements.

## 5. Concurrency and queueing

Concurrency protects shared infrastructure but may create backlogs. Investigate the job-level maximum concurrent runs, overlapping prior runs, workspace/compute capacity, autoscaling, cloud quotas and downstream service limits.

Never increase concurrency without confirming task idempotency, storage throughput, quotas and downstream capacity.

## 6. Retry, timeout and idempotency

| Failure | Recommended handling |
|---|---|
| Temporary network error | Bounded retry with backoff |
| Capacity shortage | Limited retry plus capacity investigation |
| Expired credential | Repair identity; retry alone is ineffective |
| Permission denied | Restore narrowly approved grant |
| Deterministic schema/data error | Fix root cause before rerun |
| Partial external write | Reconcile side effects before retry |

Define explicit maximum attempts, timeout budgets and an overall business completion deadline. Use checkpoints, stable batch IDs and deduplication where appropriate.

## 7. Lab A — Windows PowerShell CLI preflight

Use an approved staging profile. These commands are read-only:

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks jobs get --help
databricks jobs get-run --help
databricks jobs get-run-output --help
~~~

Confirm the workspace host and effective identity. Job listing and run history may be permission-filtered or paginated; a sample is not exhaustive. Do not export credentials.

## 8. Lab B — Inspect one authorized job

After checking installed CLI flags:

~~~powershell
databricks jobs get --job-id 12345 --output json
databricks jobs list-runs --job-id 12345 --limit 25 --output json
~~~

Replace the example job ID with an approved staging job. Review:
1. Trigger type, cron expression, timezone and pause state.
2. Task dependencies and conditional branches.
3. Job concurrency and queueing configuration.
4. Retry, timeout and notification policy.
5. Job owner, run-as identity and workspace permissions.
6. Task compute, runtime and policy.
7. External dependencies and output locations.

## 9. Lab C — Run-state investigation

~~~powershell
databricks jobs get-run --run-id 67890 --output json
~~~

Use an authorized run ID. Collect UTC trigger/start/end times, queueing, task attempts, lifecycle/result state and termination details.

For a synthetic duration calculation:

~~~powershell
$startUtc = [datetimeoffset]::Parse("2026-10-01T04:00:00Z")
$endUtc = [datetimeoffset]::Parse("2026-10-01T04:17:30Z")
[pscustomobject]@{
  DurationMinutes = [math]::Round(($endUtc - $startUtc).TotalMinutes,2)
}
~~~

Separate end-to-end wall time from queue delay, compute provisioning and actual task execution.

## 10. Lab D — Workflow SLIs and SLOs

| SLI | Definition |
|---|---|
| Job completion success | Successful eligible runs / total eligible runs |
| Deadline adherence | Runs completed before deadline / eligible scheduled runs |
| Queue latency | Time between trigger and admission |
| End-to-end latency | Trigger to validated output completion |
| Retry rate | Runs needing retries / eligible runs |
| Recovery time | Incident detection to validated recovery |

**Illustrative SLO:** at least 99% of eligible daily runs of demo-daily-pipeline finish before 07:00 UTC over 30 days. Select actual targets with workload owners and document exclusions.

Alert on missing expected runs, projected deadline misses, repeated failures, queue backlog and notification delivery failure. Include severity, owner, runbook and escalation.

## 11. Lab E — Controlled staging failure exercise

1. Select an approved disposable job using synthetic input and a defined cost cap.
2. Capture current configuration and last successful run.
3. Introduce a synthetic task failure or conduct a tabletop trigger outage.
4. Observe task states, retries and alert delivery.
5. Record partial output and external side effects.
6. Repair the synthetic cause.
7. Validate idempotency before repair or rerun.
8. Reconcile output, confirm downstream state and restore original staging settings.
9. Retain sanitized evidence in restricted storage.

Never disable production schedules, credentials or network paths for a lab.

## 12. Incident A — Scheduled job did not run

**Triage:** verify trigger configuration, timezone, paused state, event-source delivery, permissions and job visibility.

**Recovery:** restore approved trigger and determine whether controlled backfill is necessary.

**Acceptance:** synthetic trigger works and missed business output is reconciled.

## 13. Incident B — Queue backlog

**Triage:** inspect concurrency, overlapping runs, capacity, quotas, cluster startup and upstream delays.

**Recovery:** remove verified blockers, prioritize critical workloads and use approved capacity changes.

**Acceptance:** queue drains without overloading dependent services.

## 14. Incident C — Retry storm

**Triage:** inspect repeated error signatures, retry count, backoff, shared dependency and duplicate side effects.

**Recovery:** throttle or pause the approved staging workflow, repair root cause and restore bounded retries.

**Acceptance:** attempts and costs normalize, with no duplicate output.

## 15. Incident D — Partial DAG failure

**Triage:** identify successful tasks, failed dependencies, committed data and external calls.

**Recovery:** repair only safe failed tasks where supported, or reconcile and rerun the minimum required scope.

**Acceptance:** final outputs consistent, downstream consumers healthy.

## 16. Incident E — Run-as identity denied

**Triage:** separate authentication, workspace assignment, job/compute permissions, Unity Catalog grants and cloud IAM.

**Recovery:** restore the minimum approved permission, never blanket admin access.

**Acceptance:** intended job succeeds and unauthorized access remains denied.

## 17. Incident F — Notification failure

**Triage:** inspect notification configuration, webhook/email delivery, suppression and alert ownership.

**Recovery:** repair routing and add independent missing-completion detection.

**Acceptance:** synthetic failure pages the correct on-call route.

## 18. Troubleshooting decision tree

    Expected run missing?
      -> schedule / timezone / pause / event source
    Run queued?
      -> concurrency / quota / compute capacity
    Compute not starting?
      -> policy / IAM / cloud / network
    Task failing?
      -> logs / Spark / data / storage / dependency
    Run successful but deadline missed?
      -> queue + startup + execution breakdown
    Run green but output incorrect?
      -> validation / idempotency / downstream state

## 19. Production recovery runbook

1. Detect user impact, missed deadline or missing run.
2. Scope workspace, job, run, tasks, upstream and downstream systems.
3. Preserve UTC timeline, sanitized errors and recent changes.
4. Classify trigger, admission, compute, execution, identity or dependency failure.
5. Assess partial commits and rerun safety.
6. Choose the smallest approved mitigation with rollback.
7. Repair, rerun or backfill only validated scope.
8. Verify output correctness, downstream state, latency and cost.
9. Communicate recovery and create corrective actions.

## 20. Change and rollback

Before changing triggers, concurrency, retries, timeouts or compute:
- Record current settings and dependent jobs.
- Test in staging and define rollback conditions.
- Review side effects from already-started runs.
- Obtain change approval and schedule a controlled rollout.
- Validate the first execution and next scheduled occurrence.
- Confirm no missed or duplicate output.

Reverting the job definition does not reverse committed external side effects.

## 21. Acceptance checklist

- [ ] Critical job inventory and owners verified.
- [ ] Schedules, timezones and event triggers reviewed.
- [ ] DAG and dependency paths documented.
- [ ] Concurrency, queueing, retry and timeout settings justified.
- [ ] Run-as identity and cloud/data permissions reviewed.
- [ ] SLIs, SLOs and missing-run alerts defined.
- [ ] Staging failure and recovery test completed.
- [ ] Repair/rerun/backfill safety documented.
- [ ] Six incident scenarios reviewed.
- [ ] Notification delivery validated.
- [ ] Rollback plan approved.
- [ ] No production changes or secrets exposed.

## 22. Review questions

1. Why can a job be successful but miss its business deadline?
2. What makes a retry storm dangerous?
3. When is repair safer than a complete rerun?
4. Why is cloud storage authorization separate from run-as authentication?
5. What evidence proves end-to-end recovery beyond a green run status?

## 23. Official references

- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Jobs CLI: https://docs.databricks.com/aws/en/dev-tools/cli/reference/jobs-commands
- Databricks authentication: https://docs.databricks.com/aws/en/dev-tools/auth/
- Azure Databricks Jobs: https://learn.microsoft.com/azure/databricks/jobs/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 022 — Workflow Failure Handling, Retries and Recovery Automation.
