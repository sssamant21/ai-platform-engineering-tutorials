# Chapter 025 — Production Workflow Reliability and Recovery Capstone

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 05 — Jobs, Workflows and Production Reliability (capstone)  
**Clouds:** AWS, Azure, GCP  
**Audience:** SRE, Platform Engineering, Operations, FinOps  
**Prerequisites:** Chapters 021–024  
**Safety:** Read-only production review; approved staging exercises with synthetic data

## 1. Mission

Demonstrate that critical Databricks Jobs are scheduled correctly, monitored independently, protected against unsafe retries, sized for peak demand, recoverable after partial failure, and supported by tested runbooks.

This is an infrastructure and operational reliability project, not a data transformation development tutorial.

### Deliverables

1. Critical workflow inventory and dependency diagram.
2. SLI/SLO specification and deadline monitoring design.
3. Retry, repair, rerun and backfill safety matrix.
4. Capacity and cloud quota assessment.
5. Staging fault-injection evidence and six incident scenarios.
6. Recovery automation design with approval gates.
7. Final operational readiness scorecard and sign-off.

## 2. Reference architecture

    Business schedule / file event / API trigger
                       |
                Databricks Jobs
                       |
            Admission / concurrency
                       |
                Workflow task DAG
                       |
         Compute / serverless execution
                       |
        Cloud storage / database / APIs
                       |
              Validated business output
                       |
          +------------+------------+
          |                         |
    Jobs telemetry             Output freshness
          |                         |
          +------------+------------+
                       |
            SLOs / alerts / incident
                       |
         Controlled retry/repair/backfill
                       |
           Evidence / RCA / prevention

A successful job run is insufficient if the business output is missing, late, duplicated or incorrect.

## 3. Safety and access controls

- Confirm cloud, region, workspace and staging profile before commands.
- Do not modify production jobs, permissions, alerts, secrets or compute during the lab.
- Use synthetic input, disposable staging jobs and a documented cost cap.
- Protect job IDs, resource paths, identity details and logs in restricted evidence storage.
- Require change approval for state-changing recovery operations.
- Never automatically rerun a job with unknown external side effects.
- Maintain an independent alerting path for automation failure.

## 4. Assessment roles

| Role | Responsibility |
|---|---|
| SRE lead | Incident and acceptance owner |
| Job owner | Task behavior, output validation, business deadline |
| Platform engineer | Jobs, compute, policies, quotas |
| Data governance | Unity Catalog and data permissions |
| Cloud engineer | IAM, storage, network, capacity |
| FinOps | Cost baseline and scaling economics |
| Change approver | Controlled staging and production changes |

## 5. Lab A — PowerShell workspace preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
~~~

Verify installed CLI syntax and actual workspace host. Results can be paginated or permission-filtered.

## 6. Lab B — Inventory a critical job

~~~powershell
# Replace example IDs with approved staging values.
databricks jobs get --job-id 12345 --output json
databricks jobs list-runs --job-id 12345 --limit 25 --output json
databricks jobs get-run --run-id 67890 --output json
~~~

Collect:
- Job owner, run-as identity and escalation.
- Schedule/trigger and timezone.
- Task DAG and dependency conditions.
- Concurrency, queueing, retries and timeouts.
- Compute mode, policies and autoscaling.
- Business deadline and validated output.
- Upstream/downstream service dependencies.
- Notification and missing-run alert route.

## 7. Workflow inventory template

| Field | Value |
|---|---|
| Workflow | demo-daily-pipeline |
| Environment | demo-staging |
| Cloud/region | |
| Business owner | |
| On-call owner | |
| Trigger/timezone | |
| Expected completion UTC | |
| Run-as identity | |
| Critical dependencies | |
| Idempotency guarantee | |
| Recovery action | |
| Change approval | |

## 8. Lab C — Reliability baseline

Measure for a defined observation window:
- Eligible expected runs.
- Observed runs, including failures and cancellations.
- Validated outputs completed before deadline.
- Queue latency and compute startup latency.
- Task duration, retries, repairs and recovery time.
- Cloud capacity errors and external throttling.
- Cost per validated successful run.

### Example SLO (illustrative)

    Workflow: demo-daily-pipeline
    Window: rolling 30 days
    Target: >= 99% of eligible scheduled outputs validated by 07:00 UTC
    Owner: platform-sre
    Alert: missing run, predicted deadline miss, invalid output
    Exclusions: only documented and approved exclusions

Do not exclude missing or failed runs merely because no successful run record exists.

## 9. Lab D — Synthetic reliability score

~~~powershell
$runs = @(
  [pscustomobject]@{ Run="r1"; Eligible=$true; OnTime=$true; Valid=$true },
  [pscustomobject]@{ Run="r2"; Eligible=$true; OnTime=$true; Valid=$true },
  [pscustomobject]@{ Run="r3"; Eligible=$true; OnTime=$false; Valid=$true },
  [pscustomobject]@{ Run="r4"; Eligible=$true; OnTime=$true; Valid=$false },
  [pscustomobject]@{ Run="r5"; Eligible=$true; OnTime=$true; Valid=$true }
)
$eligible = @($runs | Where-Object Eligible).Count
$good = @($runs | Where-Object { $_.Eligible -and $_.OnTime -and $_.Valid }).Count
[pscustomobject]@{
  Eligible = $eligible
  ValidatedOnTime = $good
  ReliabilityPct = [math]::Round(100.0 * $good / $eligible,2)
}
~~~

Expected result: three of five eligible runs validated on time, or 60%. This tiny synthetic sample is a teaching example, not a production SLO assessment.

## 10. Lab E — System table discovery

Where enabled and authorized:

~~~sql
SHOW TABLES IN system.lakeflow;
DESCRIBE TABLE system.lakeflow.job_run_timeline;
SELECT * FROM system.lakeflow.job_run_timeline LIMIT 10;
~~~

Confirm schema, privileges, retention, region/account scope and timeline row granularity before aggregating. One timeline row may not equal one complete run.

## 11. Lab F — Queueing and capacity worksheet

| Dimension | Baseline | Peak | Action |
|---|---|---|---|
| Runs/hour | | | |
| Active concurrent runs | | | |
| Queue wait p95 | | | |
| Startup p95 | | | |
| Execution p95 | | | |
| Output validation delay | | | |
| Cloud quota headroom | | | |
| Dependency saturation | | | |
| Cost per successful run | | | |

### Offline queueing illustration

~~~powershell
$arrivalsPerHour = 12
$meanEndToEndMinutes = 20
$averageRunsInSystem = ($arrivalsPerHour / 60.0) * $meanEndToEndMinutes
[pscustomobject]@{
  ArrivalsPerHour = $arrivalsPerHour
  EstimatedAverageRunsInSystem = [math]::Round($averageRunsInSystem,2)
}
~~~

This Little's Law illustration assumes a stable measurement window and is not a direct concurrency setting recommendation. Validate peaks, task parallelism and external capacity separately.

## 12. Lab G — Cloud capacity validation

| AWS | Azure | GCP |
|---|---|---|
| EC2 quotas, instance availability, VPC IPs, IAM, S3 throttling | VM-family quotas, subnet IPs, identity, ADLS limits | Compute Engine quotas, subnet capacity, service account, Cloud Storage |

Read-only context checks, where relevant:

~~~powershell
aws sts get-caller-identity
az account show --query "{subscription:id,tenant:tenantId}" -o json
gcloud config get-value project
~~~

Run only the command for the approved cloud account/project. These checks confirm context, not capacity availability.

## 13. Lab H — Controlled staging recovery

**Preconditions:** approved change, synthetic dataset, disposable job, cost cap, rollback owner and evidence location.

1. Record baseline configuration and latest validated successful output.
2. Trigger a synthetic task failure or tabletop dependency outage.
3. Confirm failure telemetry and on-call alert delivery.
4. Inspect retries, queue state and partial side effects.
5. Stop if the retry budget or cost guardrail is exceeded.
6. Repair the synthetic cause.
7. Confirm idempotency and choose the smallest safe repair/rerun.
8. Validate output count, uniqueness, freshness and downstream state.
9. Restore staging configuration and close the change.

Do not disable production credentials, storage, networking or monitoring.

## 14. Recovery decision matrix

| Condition | Preferred path | Required validation |
|---|---|---|
| Transient idempotent task | Bounded retry | Output consistency |
| Failed downstream task after upstream success | Supported targeted repair | Dependency state |
| Missing scheduled interval | Controlled backfill | Coverage/no overlap |
| Expired credential | Approved rotation/repair | Authentication and least privilege |
| Cloud quota shortage | Capacity/queue mitigation | SLO and cost |
| Unknown partial external write | Stop and reconcile | Exact side effects |
| Shared platform outage | Incident response, throttling | Recovery of all affected jobs |

## 15. Recovery automation guardrails

Design automation to be **read-only by default**. Any mutating action must have:
- Explicit approved scope and identity.
- Error-class allowlist.
- Maximum attempts and cooldown.
- Deadline and cost budget.
- Idempotency and partial-commit gate.
- Per-job deduplication key.
- Shared-outage circuit breaker.
- Independent alert and kill switch.
- Complete audit trail and human escalation.

Do not allow an automation to bypass security approvals or rerun a non-idempotent task blindly.

## 16. Incident simulation A — Missing scheduled run

**Inject:** expected occurrence absent from history.

**Investigate:** timezone, pause state, trigger source, run-history completeness and permission visibility.

**Recover:** restore approved trigger and reconcile missing interval.

**Pass:** expected-run detector alerts and missing output is safely backfilled.

## 17. Incident simulation B — Queue backlog

**Inject:** synthetic arrivals exceed staging concurrency.

**Investigate:** queue wait, overlapping runs, compute capacity and shared dependencies.

**Recover:** controlled backlog drain and approved capacity change.

**Pass:** critical deadline restored without downstream saturation.

## 18. Incident simulation C — Retry storm

**Inject:** synthetic transient failure repeats beyond normal attempts.

**Investigate:** failure classification, retry policy, dependency outage and cost.

**Recover:** enforce bounded attempts and circuit breaker.

**Pass:** no uncontrolled repeated execution or duplicate output.

## 19. Incident simulation D — Partial write

**Inject/tabletop:** task fails after a synthetic partial commit.

**Investigate:** batch ID, checkpoint, committed records and downstream consumers.

**Recover:** reconcile and perform targeted repair only if safe.

**Pass:** no missing or duplicate records after recovery.

## 20. Incident simulation E — Identity or catalog regression

**Inject/tabletop:** run-as principal loses required permission.

**Investigate:** workspace access, Unity Catalog grants, cloud IAM and recent changes.

**Recover:** restore minimum approved access.

**Pass:** authorized job succeeds while unauthorized access remains denied.

## 21. Incident simulation F — Monitoring blind spot

**Inject:** synthetic failed run is not delivered to expected alert route.

**Investigate:** event source, ingestion delay, alert query, suppression, delivery and ownership.

**Recover:** restore notification and independent missing-run alert.

**Pass:** synthetic failure pages the correct route and is acknowledged.

## 22. Incident response runbook

1. Declare severity based on deadline, data correctness and affected consumers.
2. Identify workspace, job, run, tasks, cloud and upstream dependencies.
3. Capture UTC timeline and preserve sanitized logs/configuration.
4. Decompose trigger, queue, startup, execution and output validation.
5. Determine failure class and whether a shared outage exists.
6. Reconcile partial side effects before any rerun.
7. Choose the smallest approved mitigation with rollback.
8. Validate business output, downstream state, security and cost.
9. Communicate impact, recovery and remaining risk.
10. Create corrective actions with owners and deadlines.

## 23. Change management and rollback

Before modifying schedules, retry policy, concurrency, compute, alerts or automation:
- Capture current approved settings.
- Review blast radius, cloud quota, dependency capacity and cost.
- Test in staging.
- Define rollback trigger and approver.
- Roll out with bounded scope.
- Verify the next expected scheduled run and output.
- Remember that reverting configuration cannot undo committed external side effects.

## 24. Acceptance scorecard

| Control | Status | Evidence / owner |
|---|---|---|
| Critical workflow inventory | Not assessed | |
| Owner, run-as identity and dependencies | Not assessed | |
| Schedule and timezone correctness | Not assessed | |
| Deadline-based SLO and denominator | Not assessed | |
| Independent missing-run monitoring | Not assessed | |
| Queue and startup telemetry | Not assessed | |
| Retry/timeout/idempotency controls | Not assessed | |
| Safe repair/rerun/backfill runbook | Not assessed | |
| Cloud capacity and quota assessment | Not assessed | |
| Cost per validated successful run | Not assessed | |
| Synthetic alert delivery and escalation | Not assessed | |
| Six incident simulations | Not assessed | |
| Change/rollback and evidence controls | Not assessed | |

## 25. Readiness decision

**GO:** all critical controls verified, staging recovery accepted, alerts reach owners and no unresolved critical risks.

**CONDITIONAL GO:** only explicitly accepted noncritical gaps, each with an owner, deadline and mitigation.

**NO-GO:** unsafe reruns, unknown partial writes, missing critical alert coverage, unverified security boundaries, unrecoverable deadline misses or unresolved critical findings.

## 26. Final evidence template

    Assessment UTC:
    Cloud / region / workspace:
    Critical workflow:
    SRE lead:
    Business owner:
    Change approval:
    Baseline measurement window:
    Deadline and SLO:
    Trigger/queue/startup/execution evidence:
    Output validation:
    Retry/repair/backfill findings:
    Cloud capacity and quota findings:
    Alert delivery test:
    Incident simulations A–F:
    Remaining risks:
    Decision: GO / CONDITIONAL GO / NO-GO
    Approvals:

## 27. Cleanup

Remove only approved disposable staging resources. Restore original schedules, alert routes and concurrency. Revoke temporary staging access, verify output integrity and retain sanitized evidence in restricted storage.

## 28. Review questions

1. Why must output validation be part of workflow completion SLOs?
2. When is a targeted repair preferable to a full rerun?
3. How do cloud quotas and external dependencies affect concurrency planning?
4. What safeguards prevent recovery automation from amplifying an incident?
5. What evidence is required for a production go/no-go decision?

## 29. Official references

- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks Jobs CLI: https://docs.databricks.com/aws/en/dev-tools/cli/reference/jobs-commands
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Azure Databricks Jobs: https://learn.microsoft.com/azure/databricks/jobs/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Part 05 complete. Next:** Chapter 026 — Databricks Storage Architecture and Cloud Object Storage Integration.
