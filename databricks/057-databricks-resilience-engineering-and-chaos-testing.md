# Chapter 057 — Databricks Resilience Engineering and Chaos Testing

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 12 — Disaster Recovery and Production Resilience  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform Engineering, Data Engineering, Security and Incident Commanders  
**Prerequisites:** Chapters 041–056  
**Safety:** All fault injection is confined to isolated, approved staging resources and synthetic datasets. Production work in this chapter is read-only. Never interrupt shared networks, revoke real credentials, delete checkpoints, corrupt customer data, terminate production compute, or trigger real paging. Require explicit authorization, abort authority, blast-radius controls and a verified rollback before every experiment.

## 1. Mission

Validate that Databricks workloads tolerate realistic infrastructure, identity, storage, capacity and orchestration failures without unacceptable data loss or consumer impact. Build a disciplined chaos engineering process based on hypotheses, observable steady state, controlled fault injection, measurable recovery and actionable corrective work.

## 2. Learning objectives

- Define a falsifiable resilience hypothesis and steady-state indicators.
- Select fault scenarios using actual incident history and failure modes.
- Set staging blast-radius, duration and abort limits.
- Instrument jobs, Spark, SQL warehouses, Delta pipelines and cloud dependencies.
- Test retry, backoff, idempotency, single-writer control and recovery.
- Distinguish availability recovery from data correctness recovery.
- Measure detection time, mitigation time, RPO and RTO.
- Produce evidence and convert failed hypotheses into owned corrective actions.

## 3. Resilience experiment lifecycle

    Incident history + service dependency map
                    |
           Select failure hypothesis
                    |
        Define steady state + SLO limits
                    |
      Approve staging scope + abort criteria
                    |
          Capture baseline + backups
                    |
          Inject one controlled fault
                    |
          Observe + abort if necessary
                    |
      Recover + validate consumer outcomes
                    |
       Evidence + RCA + corrective actions
                    |
              Re-test and close

Chaos testing is not random destruction. A good experiment produces interpretable evidence and is safe to stop.

## 4. Reliability dimensions

| Dimension | Example resilience question |
|---|---|
| Compute | Does a task recover when an executor becomes unavailable? |
| Capacity | Does the pipeline queue safely when compute capacity is constrained? |
| Storage | Are transient object-store errors retried without duplicate output? |
| Identity | Are expired or denied credentials detected and escalated promptly? |
| Networking | Are dependency timeouts bounded with safe backoff? |
| Orchestration | Can a failed task retry without replaying committed side effects? |
| Data | Are uniqueness, completeness and freshness preserved? |
| Governance | Are access controls maintained through recovery? |
| Monitoring | Does a failure generate a timely, actionable alert? |
| Disaster recovery | Can an isolated alternate region recover with a single writer? |

## 5. Experiment selection and risk

Prefer a recent, frequent, high-impact failure with an observable hypothesis. Start with read-only table-top and application-level staging faults before infrastructure disruption. Reject scenarios that affect shared production resources or cannot be stopped and reversed safely.

## 6. Experiment charter

~~~markdown
# Databricks Resilience Experiment
- Experiment ID / ticket:
- Service, owner and cloud/region:
- Staging workspace and synthetic data:
- Incident/failure mode motivating test:
- Hypothesis:
- Baseline SLIs and expected steady state:
- Fault mechanism and exact scope:
- Duration and maximum blast radius:
- Prerequisites and backup/recovery point:
- Explicit abort triggers:
- Incident commander / executor / observer:
- Recovery steps and authorization:
- Data correctness and consumer checks:
- Expected versus observed behavior:
- Corrective actions and re-test date:
~~~

## 7. Steady-state indicators

Record job success and duration, queue time, query P95/P99, freshness, duplicate/null counts, checkpoint progress, cluster provisioning, retry rate, audit access denials and alert delivery. Establish baseline over an appropriate workload cadence; a single green run is not sufficient for daily jobs.

## 8. Blast-radius design

Use separate staging workspace/account or subscription/project, isolated storage and catalogs, synthetic source events, restricted service principals, bounded job concurrency, spend limits, short experiment duration and a named abort operator. Prevent synthetic alerts from reaching production paging.

## 9. Fault injection taxonomy

**Application-level:** controlled exception, invalid synthetic payload, timeout from a test dependency.

**Orchestration-level:** paused synthetic schedule, bounded retry, intentionally failed staging task.

**Resource-level:** constrained synthetic cluster resources or controlled quota simulation.

**Data-level:** duplicate or out-of-order synthetic events, schema mismatch, missing fixture.

**Dependency-level:** test endpoint latency or denied access on disposable staging identity.

**Recovery-level:** restore synthetic table copy, resume paused staging workflow, replay synthetic source with idempotency.

Avoid live cloud network or KMS disruption unless a separately reviewed, isolated exercise explicitly permits it.

## 10. Lab A — Preflight [SAFE-READ]

~~~powershell
git rev-parse HEAD
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Verify workspace, principal, region and isolated staging ownership against the experiment charter.

## 11. Lab B — Inventory workloads [SAFE-READ]

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks cluster-policies list --output json
~~~

Inventory only authorized resources. API output can be paginated or permission-filtered.

## 12. Lab C — Static deployment readiness [SAFE-READ]

~~~powershell
databricks bundle validate --target staging
databricks bundle summary --target staging
terraform -chdir=infrastructure/environments/staging fmt -check -recursive
terraform -chdir=infrastructure/environments/staging init -backend=false
terraform -chdir=infrastructure/environments/staging validate
~~~

Static validation does not establish runtime resilience or prove the intended staging identity.

## 13. Lab D — Synthetic baseline [SAFE-READ]

~~~sql
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT event_id) AS distinct_events,
  SUM(CASE WHEN event_id IS NULL THEN 1 ELSE 0 END) AS null_ids,
  MAX(event_time) AS latest_event_time
FROM demo_catalog.demo_staging.synthetic_events;
~~~

Record a fixture manifest, schema, expected count and freshness deadline. The table must already exist in an authorized staging catalog.

## 14. Lab E — Job run baseline [SAFE-READ]

~~~powershell
databricks jobs list-runs --limit 25 --output json
~~~

Capture run IDs, task state, retry counts, duration and completion time. Distinguish individual attempts from final job outcomes. If the CLI version differs, consult its installed help rather than assuming flags.

## 15. Lab F — Delta history and recovery point [SAFE-READ]

~~~sql
DESCRIBE HISTORY demo_catalog.demo_staging.synthetic_events;
DESCRIBE DETAIL demo_catalog.demo_staging.synthetic_events;
~~~

Confirm historical data files and retention support the planned staging recovery. Delta time travel is not an independent backup.

## 16. Lab G — Deterministic application fault [MUTATING-STAGING]

Use an isolated staging notebook/task with synthetic inputs. Example Python task body:

~~~python
import os
import sys

mode = os.environ.get("CHAOS_TEST_MODE", "baseline")
if mode == "inject_failure":
    raise RuntimeError("STAGING_ONLY_SYNTHETIC_FAULT")

print("Synthetic staging task completed")
sys.exit(0)
~~~

Run the failure mode only in the approved isolated staging job with explicit parameters and a short timeout. Observe retry behavior, job outcome and alert delivery. Restore the baseline mode immediately after the test. Do not put fault toggles in production runtime paths.

## 17. Lab H — Idempotency exercise [STAGING]

1. Create a deterministic synthetic input batch with stable event IDs.
2. Process it through a staging-only pipeline.
3. Re-submit the same batch through an approved replay procedure.
4. Verify output uniqueness, completeness and expected version.
5. Confirm that a retry does not duplicate external side effects.
6. Document idempotency key, transaction boundary and recovery limitations.

A successful Delta MERGE alone does not prove that external notifications or downstream writes are idempotent.

## 18. Lab I — Detection and recovery timing

~~~powershell
$injection = [datetimeoffset]"2026-10-14T12:00:00Z"
$detected = [datetimeoffset]"2026-10-14T12:02:00Z"
$recovered = [datetimeoffset]"2026-10-14T12:18:00Z"
[pscustomobject]@{
  DetectionMinutes = ($detected - $injection).TotalMinutes
  RecoveryMinutes = ($recovered - $injection).TotalMinutes
}
~~~

Expected: detection 2 minutes, recovery 18 minutes. Actual experiments require timestamps from logs and consumer validation.

## 19. Lab J — Cloud identity and isolation [SAFE-READ]

**AWS**

~~~powershell
aws sts get-caller-identity
aws configure get region
~~~

**Azure**

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
~~~

**GCP**

~~~powershell
gcloud config get-value project
gcloud auth list
~~~

Verify the test cloud account, region and service identity are isolated from production before any authorized mutation.

## 20. Safe staging experiment: task failure and retry

**Hypothesis:** one transient synthetic task exception is retried within bounded limits, produces one accepted output and alerts only if the final SLO is violated.

**Procedure:** capture baseline; inject one controlled staging exception; observe retries and task state; restore baseline; validate output count, uniqueness and consumer deadline.

**Abort:** repeated unbounded retries, unexpected spend, wrong workspace or real customer dependency.

**Pass:** bounded recovery, correct output and actionable telemetry.

## 21. Safe staging experiment: source delay

**Hypothesis:** temporary synthetic source unavailability produces a freshness warning without corrupting output.

**Procedure:** pause only the isolated synthetic feeder for a preapproved interval; observe freshness and job state; resume; verify catch-up and ordering.

**Abort:** consumer isolation fails or the synthetic backlog exceeds the approved limit.

**Pass:** correct freshness alert, bounded catch-up and no lost/duplicate events.

## 22. Safe staging experiment: malformed event

**Hypothesis:** invalid synthetic records are rejected or quarantined without poisoning valid events.

**Procedure:** submit an invalid staging fixture through the test input path; inspect validation and dead-letter handling; verify valid events continue.

**Pass:** invalid event traceability, valid output correctness and documented recovery.

## 23. Safe staging experiment: access denial

**Hypothesis:** a disposable staging principal denied a test resource generates a clear authorization failure and no privilege escalation.

**Procedure:** use a deliberately unprivileged pre-created test principal; attempt an authorized negative-access test. Do not modify production grants.

**Pass:** denied access, actionable audit evidence and unchanged authorized principal access.

## 24. Safe staging experiment: capacity pressure

**Hypothesis:** bounded synthetic workload pressure produces queueing/backpressure rather than data corruption or uncontrolled scaling.

**Procedure:** use a preapproved low-capacity staging configuration and capped job concurrency; observe startup, queue, retry and budget.

**Pass:** workload completes or fails safely within time and cost limits.

## 25. Recovery and abort protocol

Before injection, document exact stop control, rollback operator, last-known-good configuration, fixture restore point, writer ownership and acceptance tests. Abort immediately for wrong target, unexpected external calls, unauthorized privileges, data-integrity failure outside synthetic scope, monitoring loss or runaway resource consumption.

## 26. Observability during experiments

Capture job/task events, Spark driver/executor logs, SQL history, Delta history, cloud metrics, audit events, deployment SHA and synthetic output manifest. Monitor source freshness and collector health. Missing telemetry is an experiment failure, not proof of resilience.

## 27. SLO evaluation

Define the eligible population, success condition, observation window and error budget before injecting faults. For scheduled jobs, measure deadline completion and data correctness. For SQL, measure eligible request success and latency. Do not mask errors with retries if the business deadline is still breached.

## 28. Scenario A — Executor loss

**Detect:** task retry and executor removal in staging.  
**Investigate:** memory, shuffle, resource availability and retry policy.  
**Pass:** bounded recovery with correct output.

## 29. Scenario B — Driver process failure

**Detect:** staging job fails or restarts after driver termination simulation.  
**Investigate:** checkpoint durability, task restart and side effects.  
**Pass:** safe recovery or explicit fail-fast escalation.

## 30. Scenario C — Object-store transient timeout

**Detect:** controlled test dependency returns timeout.  
**Investigate:** retry/backoff and idempotency.  
**Pass:** bounded retries and no duplicate writes.

## 31. Scenario D — Cluster capacity shortfall

**Detect:** staging job queues or fails to provision.  
**Investigate:** quotas, node types and concurrency limits.  
**Pass:** clear alert and documented fallback within cost limits.

## 32. Scenario E — Expired staging credential

**Detect:** disposable test principal fails authentication.  
**Investigate:** rotation, federation, token lifetime and audit trail.  
**Pass:** no secret exposure and authorized recovery.

## 33. Scenario F — Malformed synthetic event

**Detect:** schema validation or quality check rejects input.  
**Investigate:** quarantine, schema evolution and downstream contracts.  
**Pass:** bad record isolated and valid data accepted.

## 34. Scenario G — Duplicate source replay

**Detect:** output uniqueness invariant fails or remains stable.  
**Investigate:** checkpoint, MERGE keys and external side effects.  
**Pass:** exactly intended accepted output.

## 35. Scenario H — Alert delivery failure

**Detect:** synthetic staging alert is missing.  
**Investigate:** monitoring lag, routing and collector permissions.  
**Pass:** verified delivery and resolution.

## 36. Scenario I — Conflicting writer activation

**Detect:** staging writer ownership check identifies simultaneous writers.  
**Investigate:** leases, schedule overlap and failover sequencing.  
**Pass:** single-writer fencing before processing.

## 37. Scenario J — Recovery configuration succeeds but data is wrong

**Detect:** staging job green but fixture counts or freshness fail.  
**Investigate:** replay gaps, duplicates and schema changes.  
**Pass:** consumer correctness, not merely process availability.

## 38. Chaos experiment runbook

~~~markdown
# Databricks Staging Resilience Exercise
- Experiment ID / ticket:
- Hypothesis / expected steady state:
- Staging workspace, cloud and region:
- Approved synthetic resources:
- Owner, observer and abort authority:
- Baseline metrics and fixture manifest:
- Fault and exact injection time UTC:
- Expected failure duration:
- Abort thresholds and stop command:
- Actual observed behavior:
- Recovery actions and timestamps:
- Data correctness / access / freshness checks:
- SLO and error-budget impact:
- Result: pass / fail / inconclusive:
- Follow-up action owner and due date:
~~~

## 39. Failure analysis and corrective actions

A failed hypothesis should produce a specific technical control: retry cap, circuit breaker, idempotent sink, source retention adjustment, queue alert, run-as privilege fix, backup verification, writer lease or staging test. Assign an owner, due date and reproducible re-test.

## 40. Evidence package

    resilience-test-YYYYMMDD/
      01-charter-and-approval.md
      02-isolation-and-target-proof.md
      03-baseline-and-slo.md
      04-fixture-and-recovery-point.md
      05-injection-timeline.md
      06-jobs-compute-and-spark.md
      07-cloud-and-audit.md
      08-alerting-and-detection.md
      09-data-integrity-and-freshness.md
      10-recovery-timing.md
      11-findings-and-actions.md
      12-acceptance-signoff.md

Restrict logs, identities, query text, raw state and credentials. Publish only redacted findings.

## 41. Resilience acceptance scorecard

| Gate | Required evidence |
|---|---|
| Authorization | Approved staging charter |
| Isolation | Workspace, account and synthetic resources verified |
| Hypothesis | Falsifiable steady-state claim |
| Baseline | SLO and fixture captured |
| Abort | Named authority and tested stop method |
| Recovery | Last-known-good and restore procedure |
| Observability | Job, data, cloud and alert coverage |
| Fault scope | Bounded duration and blast radius |
| Idempotency | No unintended duplicate effects |
| Data integrity | Counts, schema, uniqueness and freshness |
| Detection | Measured alert delay |
| Recovery | Measured time to consumer acceptance |
| Corrective actions | Owner, due date and re-test |
| Closure | Evidence and sign-off |

## 42. GO / CONDITIONAL GO / NO-GO

**GO:** The experiment remains isolated, produces valid evidence, demonstrates expected recovery and meets approved data/SLO acceptance.

**CONDITIONAL GO:** Only explicitly accepted noncritical gaps remain with owner, expiry and compensating controls.

**NO-GO:** Unverified isolation, uncontrolled blast radius, data-integrity failure, dual writers, missing abort control, unavailable recovery path or unexpected production impact.

## 43. Review questions

1. Why must a chaos experiment begin with a hypothesis?
2. How is a resilience test different from a disaster recovery test?
3. Why should a staging task failure be preferred to terminating shared cloud resources?
4. What makes retry behavior safe for Delta and external sinks?
5. Why is a green job status insufficient after replay?
6. How should missing telemetry affect experiment results?
7. What signals require immediate abort?
8. How do you turn a failed experiment into a measurable reliability improvement?

## 44. Official references

- Databricks Jobs monitoring: https://docs.databricks.com/aws/en/jobs/monitor
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks Delta history: https://docs.databricks.com/aws/en/delta/history
- Databricks Bundles: https://docs.databricks.com/aws/en/dev-tools/bundles/
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks compute policies: https://docs.databricks.com/aws/en/compute/policies
- AWS Fault Injection Service: https://docs.aws.amazon.com/fis/latest/userguide/what-is.html
- Azure Chaos Studio: https://learn.microsoft.com/azure/chaos-studio/chaos-studio-overview
- Google Cloud reliability guidance: https://cloud.google.com/architecture/framework/reliability
- Google SRE testing for reliability: https://sre.google/sre-book/testing-reliability/

**Next:** Chapter 058 — Databricks Capacity Planning, Performance Engineering and FinOps.
