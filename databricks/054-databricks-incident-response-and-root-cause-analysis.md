# Chapter 054 — Databricks Incident Response and Root Cause Analysis

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 11 — Compliance, Observability and Operational Assurance  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Data Platform, Security, Incident Commanders  
**Prerequisites:** Chapters 041–053  
**Safety:** Production investigation is read-only until an incident commander authorizes a change. Perform fault injection, access changes, replay, rollback and destructive exercises only in isolated staging. Protect audit logs, query text, secrets and patient/customer data.

## 1. Mission

Build a consistent, evidence-driven process for detecting, triaging, mitigating, recovering from and reviewing Databricks incidents. Distinguish trigger, contributing conditions, root causes and impact; establish an auditable timeline and corrective actions that measurably prevent recurrence.

## 2. Outcomes

After this chapter, operators can:
- Classify production incidents by customer and data impact.
- Coordinate incident command, communications and decision-making.
- Correlate Databricks jobs, compute, SQL, Unity Catalog, cloud telemetry and recent deployments.
- Diagnose permission, capacity, networking, performance, pipeline and data-integrity failures.
- Preserve a reliable UTC timeline and restricted evidence.
- Choose containment, rollback, replay or restore with explicit approval.
- Write a blameless, technically rigorous RCA.
- Validate corrective actions with measurable acceptance criteria.

## 3. Incident lifecycle

    Signal / customer report
              |
     Validate + declare severity
              |
    Commander / owners / comms
              |
    Stabilize and preserve evidence
              |
    Correlate changes and symptoms
              |
    Contain -> recover -> verify
              |
     Resolve / monitor / handoff
              |
     RCA -> actions -> exercises

The first priority is customer safety, service restoration and data integrity—not assigning blame or producing an early speculative root cause.

## 4. Severity model

| Severity | Example | Coordination |
|---|---|---|
| SEV1 | Widespread critical data outage, security exposure or data corruption | Incident command, immediate escalation |
| SEV2 | Major production pipeline or consumer degradation | On-call and affected owners |
| SEV3 | Limited impact with workaround | Team incident workflow |
| SEV4 | Low-impact defect without material SLO breach | Standard ticket |

Adapt labels to organizational policy. Severity reflects actual or credible business impact, not merely an error message.

## 5. Incident roles

| Role | Accountability |
|---|---|
| Incident commander | Severity, priorities, decisions, escalation and timeline |
| Technical lead | Investigation, mitigation and validation |
| Communications lead | Stakeholder updates and status |
| Data owner | Data correctness, replay, consumer acceptance |
| Security lead | Identity, privilege, audit and containment |
| Cloud/platform lead | IAM, network, storage, quotas and Terraform |
| Scribe | UTC chronology, evidence references and action log |

For small incidents one person may hold multiple roles, but decision authority must be explicit.

## 6. First 15 minutes

1. Confirm alert validity and monitoring freshness.
2. Establish affected workspace, region, jobs, warehouses and consumers.
3. Declare severity, commander and communication channel.
4. Record first symptom, UTC timestamps and latest known-good state.
5. Check data-integrity risk and whether writers need authorized fencing.
6. Identify active change windows and recent deployments.
7. Preserve logs, run IDs, job IDs, deployment SHA and audit references.
8. Assign investigation branches with clear owners and deadlines.
9. State next stakeholder update time.
10. Avoid unreviewed concurrent fixes.

## 7. Impact assessment

Document affected tenants, catalogs, datasets, SLAs, downstream consumers, failed runs, delayed events, unauthorized access, cost impact and recovery objectives. Differentiate observed impact from suspected exposure. A job that reports SUCCESS may still violate data correctness or freshness.

## 8. Evidence sources

| Layer | Sources | Common clues |
|---|---|---|
| Databricks jobs | Runs, tasks, retries, output, timeline | Failed dependency, timeout, retry storm |
| Compute/Spark | Cluster events, driver/executor logs, Spark UI | OOM, skew, shuffle, startup failure |
| SQL | Query history, warehouse metrics | Queueing, slow query, timeout |
| Unity Catalog | Grants, ownership, audit events | Privilege regression, external location |
| Cloud | IAM, network flow, object storage, quotas | Denied requests, DNS, capacity, KMS |
| Deployments | GitLab, Terraform, Bundles | Change correlation |
| Data | Delta history, counts, freshness, checkpoints | Corruption, duplicates, stalled ingestion |
| Monitoring | SLO dashboards and collection health | Impact and blind spots |

## 9. Timeline and causality

Use UTC and distinguish event time, ingestion time and detection time. Record evidence quality. A deployment immediately preceding failure is a hypothesis, not automatically the cause. Establish the causal mechanism through reproducible signals, comparison and counterfactual checks.

## 10. Lab A — Identity and environment preflight [SAFE-READ]

~~~powershell
git rev-parse HEAD
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm intended workspace and principal. Never paste tokens or secret-bearing output into an incident chat.

## 11. Lab B — Job triage [SAFE-READ]

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
~~~

For an approved run ID, inspect the run:

~~~powershell
databricks jobs get-run REPLACE_WITH_RUN_ID --output json
~~~

Capture state, task dependencies, retry counts, start/end timestamps and cluster association. API responses can be permission-filtered or paginated.

## 12. Lab C — Compute inventory [SAFE-READ]

~~~powershell
databricks clusters list --output json
databricks cluster-policies list --output json
databricks warehouses list --output json
~~~

For a known cluster ID:

~~~powershell
databricks clusters get REPLACE_WITH_CLUSTER_ID --output json
~~~

Check startup failures, policy, node family, autoscaling, runtime and recent changes. Driver/executor logs require separate authorized collection.

## 13. Lab D — System table discovery [SAFE-READ]

~~~sql
SELECT current_user();
SHOW SCHEMAS IN system;
SHOW TABLES IN system.lakeflow;
SHOW TABLES IN system.access;
SHOW TABLES IN system.query;
~~~

System-table coverage, schemas, retention and latency vary. Do not infer absence of incidents from missing rows.

## 14. Lab E — Job timeline investigation [SAFE-READ]

~~~sql
DESCRIBE TABLE system.lakeflow.job_run_timeline;
SELECT *
FROM system.lakeflow.job_run_timeline
LIMIT 10;
~~~

Review the live schema and filter by approved workspace/job and UTC window. Timeline records may be segmented; avoid double-counting runs.

## 15. Lab F — Audit correlation [SAFE-READ]

~~~sql
SELECT event_time, workspace_id, service_name, action_name
FROM system.access.audit
WHERE event_date >= date_sub(current_date(), 2)
ORDER BY event_time DESC
LIMIT 100;
~~~

Compare with incident onset, change tickets and GitLab deployment records. Audit events can be delayed. Limit access to sensitive identity and request fields.

## 16. Lab G — Query history [SAFE-READ]

~~~sql
SHOW TABLES IN system.query;
DESCRIBE TABLE system.query.history;
SELECT *
FROM system.query.history
LIMIT 10;
~~~

Identify warehouse, queue time, execution duration, error class and query pattern using the account's documented schema. Avoid exporting sensitive query text.

## 17. Lab H — Data correctness and freshness [SAFE-READ]

For an authorized synthetic staging table:

~~~sql
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT event_id) AS distinct_events,
  SUM(CASE WHEN event_id IS NULL THEN 1 ELSE 0 END) AS null_ids,
  MAX(event_time) AS newest_event
FROM demo_catalog.demo_staging.synthetic_events;
~~~

Compare against a deterministic fixture manifest and expected delivery deadline. Never use a full scan of a large production table without assessing cost and load.

## 18. Lab I — Delta transaction history [SAFE-READ]

~~~sql
DESCRIBE HISTORY demo_catalog.demo_staging.synthetic_events;
DESCRIBE DETAIL demo_catalog.demo_staging.synthetic_events;
~~~

Inspect operation timestamps, write modes, operation metrics and table properties. Retention and time-travel availability depend on configuration. History alone is not a verified restore plan.

## 19. Lab J — Cloud context and dependency checks [SAFE-READ]

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

Correlate storage, KMS, network, identity and quota events with Databricks errors.

## 20. Incident hypotheses

Create a hypothesis table with symptom, candidate mechanism, supporting evidence, contradicting evidence, next safe test and owner. Prefer tests that distinguish competing explanations. Stop broad log gathering when a narrower causal test is available.

## 21. Failure domain: jobs and orchestration

Investigate run-as identity, task dependencies, schedule overlap, retries, queue delays, library installation, parameter changes, concurrency limits and failed external calls. Differentiate a job scheduler issue from a task's own application exception.

## 22. Failure domain: compute and Spark

Check cluster startup, autoscaling, node availability, driver/executor memory, skew, shuffle spill, disk pressure, runtime compatibility and Spark event logs. Correlate changes with actual resource telemetry; increasing worker size without evidence can increase cost without resolving skew.

## 23. Failure domain: SQL performance

Separate compilation, queueing, execution and result retrieval. Check query shape, warehouse concurrency, cache effects, data pruning, statistics, join strategies and storage latency. Compare equivalent workload populations, not unrelated P95 values.

## 24. Failure domain: governance and identity

Inspect Unity Catalog privileges, service principal permissions, workspace assignment, external location/storage credential access, cloud IAM and token/federation expiry. Avoid granting broad admin rights to resolve a narrow workload failure.

## 25. Failure domain: networking and storage

Investigate DNS, private endpoints, routing, firewalls/security groups, NAT, object-store request failures, KMS and regional dependencies. Distinguish Databricks service errors from underlying cloud access failures.

## 26. Failure domain: data correctness

Check schema evolution, nulls, duplicates, watermarks, source completeness, idempotency, checkpoint ownership, concurrent writers and Delta transaction history. Fencing and replay require data-owner authorization and a verified recovery point.

## 27. Containment versus recovery

**Containment** prevents additional damage: stop unsafe writes, isolate exposed credentials or halt promotion. **Recovery** restores service and data correctness. Rollback of Git or Terraform does not automatically undo mutated tables or external side effects.

## 28. Rollback decision tree

    Impact ongoing?
       |
       +-- Security/data integrity risk -> authorized containment
       |
       +-- Configuration regression -> reviewed compensating change
       |
       +-- Data mutation -> approved restore/replay/compensation
       |
       +-- Cloud dependency -> failover or capacity action if preapproved
       |
       +-- Verify jobs, data, consumers, access and SLOs
       |
       +-- Monitor and close

Never use `terraform force-unlock`, unreviewed table restore or broad grant changes as a reflexive response.

## 29. Staging exercise — Failure simulation

Use an isolated synthetic job and fixture:
1. Record known-good run ID, dataset checksum/row count and SLO.
2. Introduce a harmless, explicitly approved staging-only failure (for example, invalid synthetic input).
3. Observe run state, alert and logs.
4. Create a UTC event timeline.
5. Restore fixture/configuration through the approved staging procedure.
6. Re-run and verify count, uniqueness and freshness.
7. Write a one-page incident summary and identify detection gaps.

Do not deliberately fail production jobs.

## 30. Scenario A — Job fails after bundle release

**Detect:** failures begin after a known revision.  
**Investigate:** task parameters, run-as, libraries and changed dependencies.  
**Recover:** approved rollback or corrected deployment.  
**Accept:** successful run and consumer invariants.

## 31. Scenario B — Cluster cannot start

**Detect:** provisioning errors and queue delay.  
**Investigate:** cloud quota, capacity, policy, IAM and networking.  
**Recover:** approved alternate capacity or rollback.  
**Accept:** workload completes within deadline and cost guardrails.

## 32. Scenario C — SQL latency regression

**Detect:** P95/P99 rise in one workload class.  
**Investigate:** queueing, query plan, concurrency and storage.  
**Recover:** tested query/warehouse change.  
**Accept:** SLO and resource baseline restored.

## 33. Scenario D — Unity Catalog access denied

**Detect:** authorized workload principal fails after grant change.  
**Investigate:** audit timeline, ownership and effective grants.  
**Recover:** minimum approved privilege restoration.  
**Accept:** positive and negative access tests.

## 34. Scenario E — Streaming pipeline stalls

**Detect:** freshness lag grows while infrastructure appears healthy.  
**Investigate:** source offsets, checkpoint, upstream availability and backpressure.  
**Recover:** approved single-writer restart or replay.  
**Accept:** no duplicates and freshness target met.

## 35. Scenario F — Duplicate output after retry

**Detect:** uniqueness invariant fails.  
**Investigate:** retry semantics, idempotency and concurrent writers.  
**Recover:** fence writers, approved correction/replay.  
**Accept:** deterministic output reconciles.

## 36. Scenario G — Storage or KMS access failure

**Detect:** cloud access denied or storage timeout.  
**Investigate:** IAM, credential rotation, endpoint, KMS and audit records.  
**Recover:** least-privilege controlled fix.  
**Accept:** read/write synthetic tests and audit review.

## 37. Scenario H — Monitoring blind spot

**Detect:** dashboards show zero while source timestamps are stale.  
**Investigate:** collector failures, system-table lag and permissions.  
**Recover:** restore collection and use alternate telemetry.  
**Accept:** historical gap assessed and alert delivery tested.

## 38. Scenario I — Terraform change creates workspace outage

**Detect:** availability degradation after approved apply.  
**Investigate:** exact plan, changed resources, dependency chain and cloud logs.  
**Recover:** reviewed compensating plan; preserve state integrity.  
**Accept:** workspace operations and dependent jobs validated.

## 39. Scenario J — Rollback restores service but not data

**Detect:** job succeeds but counts/freshness remain incorrect.  
**Investigate:** Delta history, checkpoints and external side effects.  
**Recover:** data-owner-approved restore or replay.  
**Accept:** downstream consumers confirm correctness.

## 40. RCA methodology

Use a blameless structure:
- **Impact:** who and what was affected, with timestamps and measurable duration.
- **Detection:** first signal, alert latency and blind spots.
- **Timeline:** facts in UTC with source references.
- **Trigger:** event that immediately preceded failure.
- **Causal mechanism:** how the system failed.
- **Contributing conditions:** architecture, testing, monitoring or process gaps.
- **What worked / what did not:** response effectiveness.
- **Recovery:** actions, approvals and consumer verification.
- **Corrective actions:** prevention, detection, mitigation and owner/deadline.
- **Open questions:** uncertainty, assumptions and follow-up evidence.

Do not substitute “human error” for a technical and organizational causal explanation.

## 41. Five whys and causal graph

Five whys can reveal weak controls but can oversimplify distributed failures. Map interacting causes: deployment policy, cloud quota, job retry semantics, alert delay and absent staging coverage. Use evidence to distinguish necessary conditions from correlations.

## 42. RCA template

~~~markdown
# Databricks Production Incident RCA

## Incident metadata
- ID / severity:
- Start, detection, mitigation and resolution (UTC):
- Commander / technical lead / data owner:
- Workspaces, regions, datasets and consumers:

## Executive summary
- What happened:
- Business impact and SLO breach:

## Detection
- First signal and alert delay:
- Missing or misleading signals:

## Timeline
| UTC | Event | Evidence | Owner |
|---|---|---|---|

## Root cause
- Trigger:
- Verified causal mechanism:
- Contributing conditions:
- Evidence and confidence:
- Alternative hypotheses ruled out:

## Recovery
- Containment:
- Authorized changes:
- Data integrity and consumer acceptance:
- Residual risks:

## Corrective actions
| Action | Type | Owner | Due | Verification |
|---|---|---|---|---|

## Follow-up
- Exercise / alert / runbook changes:
- Closure approver:
~~~

## 43. Corrective action quality

Prefer measurable controls over generic reminders. Examples:
- Add a protected deployment gate verifying target workspace identity.
- Add a synthetic data uniqueness/freshness assertion.
- Alert on missing telemetry and delayed source events.
- Add a staging quota/capacity fallback exercise.
- Introduce single-writer fencing and idempotent replay.
- Automate detection of unapproved Unity Catalog grants.

Each action needs a named owner, deadline, test and evidence of completion.

## 44. Communications templates

**Initial:** “We are investigating a Databricks issue affecting [service/consumers] since [UTC]. Current impact is [verified facts]. Next update by [UTC].”

**Update:** “Impact [changed/unchanged]. We identified [verified finding] and are executing [approved mitigation]. Data integrity [status]. Next update [UTC].”

**Resolved:** “Service was restored at [UTC], verified by [jobs, data checks, consumer tests]. Monitoring continues until [UTC]. RCA follow-up [owner/date].”

Never announce resolution based solely on a green infrastructure dashboard.

## 45. Evidence package

    incident-YYYYMMDD-ID/
      01-severity-and-impact.md
      02-timeline.md
      03-jobs-and-compute.md
      04-sql-and-data-checks.md
      05-audit-and-identity.md
      06-cloud-dependencies.md
      07-deployment-and-terraform.md
      08-containment-and-recovery.md
      09-consumer-acceptance.md
      10-root-cause-analysis.md
      11-corrective-actions.md
      12-closure.md

Keep raw audit data, query text, Terraform plans and credentials in restricted evidence storage.

## 46. Operational acceptance scorecard

| Gate | Required evidence |
|---|---|
| Incident scope | Affected consumers and UTC onset |
| Severity and roles | Commander, leads and communications |
| Telemetry health | Freshness and known gaps |
| Timeline | Verifiable events and source references |
| Hypotheses | Supported and ruled-out alternatives |
| Containment | Authorized, documented and safe |
| Recovery | Tested against SLO and data integrity |
| Security | Effective privileges validated |
| Consumer sign-off | Freshness and correctness accepted |
| RCA | Causal mechanism and contributors |
| Actions | Owners, dates and verification |
| Closure | Approver and residual-risk record |

## 47. GO / CONDITIONAL GO / NO-GO

**GO:** Service and data correctness are restored, consumers are validated, monitoring is healthy and follow-up ownership is documented.

**CONDITIONAL GO:** Only explicitly accepted noncritical residual risks remain with owner, expiry and compensating controls.

**NO-GO:** Unverified data integrity, unauthorized privileges, uncontrolled writers, missing critical telemetry or incomplete containment of active harm.

## 48. Review questions

1. Why does temporal correlation with a deployment not prove causation?
2. How do you distinguish mitigation from recovery?
3. Why must a Databricks incident include data correctness validation?
4. What makes an RCA blameless yet accountable?
5. Why can Terraform rollback be insufficient?
6. What is the difference between an incident trigger and contributing conditions?
7. How should delayed audit telemetry be represented in the timeline?
8. What evidence is necessary before announcing resolution?

## 49. Official references

- Databricks Jobs monitoring: https://docs.databricks.com/aws/en/jobs/monitor
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks audit logs: https://docs.databricks.com/aws/en/admin/account-settings/audit-logs
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Delta Lake history: https://docs.databricks.com/aws/en/delta/history
- Unity Catalog privileges: https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/
- Google SRE incident management: https://sre.google/workbook/incident-response/
- Google SRE postmortem culture: https://sre.google/sre-book/postmortem-culture/
- AWS CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs/monitoring-overview

**Next:** Chapter 055 — Databricks Disaster Recovery, Backup and Restore Validation.
