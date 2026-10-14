# Chapter 053 — SLO Engineering, Alert Quality and Incident Readiness

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 11 — Compliance, Observability and Operational Assurance  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform Engineering, Data Operations, Incident Commanders  
**Prerequisites:** Chapters 041–052  
**Safety:** Production exercises are read-only. Synthetic failures, alert routing changes and paging tests require approved isolated staging. Do not intentionally interrupt production pipelines or generate customer-impacting failures.

## 1. Mission

Turn Databricks telemetry into measurable reliability commitments, high-quality alerts and practiced incident response. Establish service-level indicators (SLIs), service-level objectives (SLOs), error budgets, escalation rules, recovery exercises and evidence-based release gates.

## 2. Learning outcomes

- Define reliability boundaries for workspaces, jobs, SQL warehouses and data products.
- Specify precise SLIs with eligible populations and exclusions.
- Calculate error budgets and multiwindow burn rates.
- Build actionable alerts that distinguish symptoms from causes.
- Validate alert delivery, suppression, deduplication and monitoring health.
- Define severity, ownership, communications and escalation.
- Practice staging incident scenarios and recovery acceptance.
- Integrate SLOs with production change management.

## 3. Reliability architecture

    Business / consumer requirements
                |
         Service ownership map
                |
        SLI definitions + SLOs
                |
    Jobs / SQL / data / cloud telemetry
                |
       Windows + error-budget math
                |
        Alerts + runbook routing
                |
    On-call triage -> Incident command
                |
       Containment and recovery
                |
    Consumer validation + postmortem

An SLO is not simply a dashboard threshold: it must represent an actual service outcome.

## 4. Service boundaries

| Service | Consumer expectation | Possible SLI |
|---|---|---|
| Scheduled ingestion | Fresh data by agreed deadline | On-time successful delivery |
| Streaming pipeline | Continuous freshness | Time lag within bound |
| SQL analytics | Reliable interactive queries | Eligible requests meeting latency and success target |
| Job orchestration | Reliable scheduled execution | Successful required runs |
| Workspace/platform | Authorized access and provisioning | Successful critical operations |
| Governance | Approved principals can access data | Authorized-operation success rate |

Specify workspace, account, region, principal, dataset, job and consumer ownership for each SLO.

## 5. SLI specification template

~~~markdown
- SLI name:
- Business owner / technical owner:
- Service and affected consumers:
- Source and permissions:
- Numerator (good events):
- Denominator (eligible events):
- Exclusions and justification:
- Evaluation window and timezone:
- Freshness and telemetry lag:
- Objective and alert policy:
- Missing-data treatment:
- Recovery and escalation runbook:
~~~

Do not silently exclude failures caused by platform outages or dependency errors.

## 6. Availability and success SLOs

For event-based SLOs:

`SLI = good_eligible_events / total_eligible_events`

`Error budget = total_eligible_events × (1 − objective)`

Example: 99.9% of 100,000 eligible runs or requests permits 100 bad events in that window. For low-frequency daily jobs, deadline-based correctness and freshness may be more meaningful than percentages over a small sample.

## 7. Data freshness SLO

Define an agreed delivery deadline or a maximum accepted lag between observation time and the latest fully processed source event. Use event time only if it is trustworthy; late arrivals, backfills, clock skew and empty datasets require explicit handling. Do not confuse a recent job run timestamp with fresh business data.

## 8. SQL latency SLO

Choose a well-defined eligible query class, threshold and success condition. For example, the share of eligible production dashboard queries completing successfully under a documented threshold. P95 is useful for diagnosis but an event-based threshold often yields clearer budget accounting. Exclude ad hoc experiments only under a published policy.

## 9. Error budget burn rates

For an event-based SLO with objective `S` and observed error fraction `e`:

`burn_rate = e / (1 - S)`

For a 99.9% objective, 1% failures correspond to 10× burn. Short and long windows together reduce noisy pages; select evaluation windows according to traffic volume and on-call response needs. Avoid paging solely because a small low-volume denominator produced a high instantaneous burn.

## 10. Lab A — Windows PowerShell preflight [SAFE-READ]

~~~powershell
git --version
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
~~~

Confirm the intended account, workspace, cloud region and service principal before querying telemetry.

## 11. Lab B — SLO arithmetic without cloud access

~~~powershell
$total = 100000
$bad = 120
$objective = 0.999
$good = $total - $bad
$sli = $good / $total
$budget = $total * (1 - $objective)
$burn = ($bad / $total) / (1 - $objective)
[pscustomobject]@{
  SLI = [math]::Round($sli * 100, 3)
  BudgetEvents = [math]::Round($budget, 0)
  BadEvents = $bad
  BurnRate = [math]::Round($burn, 2)
}
~~~

Expected: SLI 99.88%, budget 100 bad events, observed 120, burn 1.2×. Budget accounting must define the eligible event set.

## 12. Lab C — System table discovery [SAFE-READ]

~~~sql
SELECT current_user();
SHOW CATALOGS;
SHOW SCHEMAS IN system;
SHOW TABLES IN system.lakeflow;
SHOW TABLES IN system.query;
~~~

Availability, retention and permissions vary by account and cloud. If a source is missing, document an alternate authorized API/metric source instead of reporting zero failures.

## 13. Lab D — Job timeline schema [SAFE-READ]

~~~sql
DESCRIBE TABLE system.lakeflow.job_run_timeline;
SELECT * FROM system.lakeflow.job_run_timeline LIMIT 10;
~~~

Review the actual schema before defining status and duration queries. Timeline rows can represent intervals rather than unique completed runs; do not count timeline segments as distinct jobs.

## 14. Lab E — Jobs API [SAFE-READ]

~~~powershell
databricks jobs list-runs --limit 25 --output json
databricks jobs list --output json
~~~

Normalize run IDs, final states, start/end times and retries. Handle API pagination and permission-filtered results. Distinguish retry attempts from the final customer-facing outcome.

## 15. Lab F — SQL history discovery [SAFE-READ]

~~~sql
SHOW TABLES IN system.query;
DESCRIBE TABLE system.query.history;
SELECT * FROM system.query.history LIMIT 10;
~~~

Restrict query text, principal information and sensitive SQL parameters. Segment by warehouse and query class before calculating latency SLOs.

## 16. Lab G — Synthetic data freshness [SAFE-READ]

~~~sql
SELECT
  COUNT(*) AS rows_seen,
  COUNT(DISTINCT event_id) AS unique_events,
  MAX(event_time) AS latest_event
FROM demo_catalog.demo_staging.synthetic_events;
~~~

Compare to the approved synthetic manifest and deadline. If the dataset is empty or the timestamp is null, mark the test inconclusive/failing according to the SLO specification.

## 17. Lab H — Audit source and monitoring freshness [SAFE-READ]

~~~sql
SELECT
  MAX(event_time) AS newest_event,
  COUNT(*) AS events_in_window
FROM system.access.audit
WHERE event_date >= date_sub(current_date(), 1);
~~~

The audit system can have ingestion lag. Missing events do not prove no activity; compare source freshness with an independently known benign staging action when authorized.

## 18. Lab I — Cloud identity verification [SAFE-READ]

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

Correlate cloud capacity, network, IAM and storage signals with Databricks incidents.

## 19. Lab J — Alert routing tabletop [STAGING]

1. Select an isolated staging alert rule and synthetic metric.
2. Record expected evaluation window, threshold, destination and owner.
3. Generate a benign test signal without impacting real jobs.
4. Verify delivery to the staging channel only.
5. Confirm the alert contains workspace, resource, SLO and runbook.
6. Acknowledge and clear the synthetic signal.
7. Record delivery time, resolution and any routing gaps.

Never page production responders without prior authorization.

## 20. Alert taxonomy

| Type | Example | Routing |
|---|---|---|
| Customer-impacting symptom | Freshness deadline missed | Page service owner |
| Critical security | Unexpected privileged change | Security incident path |
| Capacity risk | Sustained queueing with imminent SLO breach | On-call or urgent ticket |
| Operational warning | Rising retries with no SLO impact | Ticket/dashboard |
| Monitoring failure | Missing metrics or stale ingestion | Monitoring owner |
| Cost anomaly | Unexpected DBU increase | FinOps/platform owner |

Alert on actionable symptoms; retain detailed diagnostic signals in dashboards.

## 21. Multiwindow alert design

Use short-window evidence of acute impact together with a longer-window confirmation. For low-volume batch jobs, prefer explicit missed-deadline alerts over percentage burn. Record the exact denominator and suppress duplicate pages for the same incident without suppressing independent critical signals.

## 22. Alert payload contract

~~~markdown
- Alert name and severity:
- Workspace / region / environment:
- Affected job, warehouse or data product:
- Observed SLI / threshold / window:
- First observed UTC:
- Consumer impact:
- Recent change or run ID:
- Owner and escalation target:
- Dashboard / logs / trace link:
- Runbook and rollback authority:
~~~

Avoid including credentials, personal data, raw query text or restricted audit details.

## 23. Incident severity and ownership

Define severity by impact rather than technical symptom alone. Identify incident commander, technical lead, communications owner, security/data owner and executive escalation as appropriate. Explicitly state who can pause jobs, fence writers, roll back a bundle or approve infrastructure changes.

## 24. Incident response timeline

1. Detect and validate alert freshness.
2. Establish business impact and severity.
3. Assign incident commander and responders.
4. Capture UTC onset, run IDs and affected workspaces.
5. Review jobs, SQL, compute, data freshness and cloud dependencies.
6. Correlate recent Terraform, bundle and grant changes.
7. Contain unsafe writes or unauthorized changes through approved controls.
8. Recover with documented compensating actions.
9. Validate end-to-end consumer correctness and SLO.
10. Communicate restoration and record follow-up actions.

## 25. Change and error-budget policy

A depleted or rapidly burning budget triggers a reliability review of risky releases. Do not impose a universal freeze automatically: document exceptions for urgent security and reliability fixes, ownership and approval. Link budget decisions to observed impact and change risk.

## 26. Operational dashboards

Maintain service overview, error budget, job runs, freshness, SQL latency, compute capacity, audit changes, alert delivery and detector-health dashboards. Every dashboard must display source freshness and be owned by an accountable team.

## 27. SLO recording and evidence

Store definition revision, metric query revision, observation window, eligible count, good/bad count, excluded events, telemetry lag, incidents and approved exceptions. Keep raw audit and query text restricted.

## 28. Scenario A — Green jobs but stale data

**Detect:** job completion SLI passes while freshness fails.  
**Response:** inspect source delivery, checkpoints and consumer output.  
**Pass:** data freshness and correctness restored.

## 29. Scenario B — High retry rate without user impact

**Detect:** task retries increase, final jobs meet deadline.  
**Response:** investigate resource pressure as warning; do not automatically count recovered attempts as failed deliveries.  
**Pass:** retry cause understood and SLO accounting correct.

## 30. Scenario C — Query latency spikes

**Detect:** workload-specific slow-query fraction and P95 rise.  
**Response:** inspect warehouse concurrency, queueing, query plans and cloud capacity.  
**Pass:** agreed latency SLO restored.

## 31. Scenario D — Alert storm during shared cloud outage

**Detect:** many jobs page separately for one underlying dependency.  
**Response:** correlate, group and route to incident command.  
**Pass:** critical impact visible without duplicate paging.

## 32. Scenario E — Missing telemetry interpreted as healthy

**Detect:** dashboards show zero errors while ingestion is stale.  
**Response:** mark SLO evaluation unknown and alert on collection failure.  
**Pass:** freshness restored and missing period reconciled.

## 33. Scenario F — False burn due to tiny denominator

**Detect:** one failure in a small sample triggers high burn.  
**Response:** apply volume-aware thresholds and deadline-based checks.  
**Pass:** paging corresponds to meaningful consumer risk.

## 34. Scenario G — Permission regression after release

**Detect:** workload principal gets authorization failure.  
**Response:** review Unity Catalog grants and audit timeline; restore approved least privilege.  
**Pass:** positive and negative access tests succeed.

## 35. Scenario H — Capacity shortfall

**Detect:** compute provisioning failures or sustained queue delay.  
**Response:** inspect quotas, node types, autoscaling and cost controls.  
**Pass:** required jobs meet deadlines within approved limits.

## 36. Scenario I — Alert routed to wrong on-call

**Detect:** synthetic staging alert reaches incorrect destination.  
**Response:** correct labels, ownership and routing.  
**Pass:** intended channel receives and resolves the test.

## 37. Scenario J — Rollback restores job but corrupts output

**Detect:** service status green while data invariants fail.  
**Response:** fence writers, restore/replay approved data path and validate consumers.  
**Pass:** integrity and freshness restored, not just previous Git revision.

## 38. Production incident runbook

~~~markdown
# Databricks Reliability Incident
- Incident ID / severity / commander:
- First detected UTC / source freshness:
- Affected workspace, jobs, warehouses and consumers:
- SLO and error budget impact:
- Recent changes and audit timeline:
- Hypotheses and evidence:
- Safe containment actions:
- Recovery plan and authorization:
- Data integrity and freshness checks:
- Consumer acceptance and resolution UTC:
- Stakeholder updates:
- RCA, corrective actions and owners:
~~~

## 39. Reliability acceptance scorecard

| Gate | Evidence |
|---|---|
| Service ownership | Named technical/business owner |
| SLI definition | Numerator, denominator, exclusions |
| SLO target | Approved objective and window |
| Data correctness | Fixture and freshness checks |
| Telemetry health | Source freshness and completeness |
| Error budget | Calculated, reproducible and reviewed |
| Alerts | Actionable thresholds and routing |
| Incident roles | Commander and escalation contacts |
| Runbooks | Safe containment and recovery |
| Staging exercise | Alert delivery and resolution |
| Cloud correlation | IAM, network, quota and storage |
| Change linkage | Release and rollback criteria |
| Evidence | Secure retention and auditability |

## 40. GO / CONDITIONAL GO / NO-GO

**GO:** SLOs represent consumer outcomes, telemetry is healthy, alerts reach correct responders, and recovery/runbook readiness is validated.

**CONDITIONAL GO:** Only approved noncritical gaps remain with owner, expiry and compensating monitoring.

**NO-GO:** Missing critical monitoring, failed data correctness, unowned paging, uncontrolled writers, unapproved privileged change or no viable recovery procedure.

## 41. Review questions

1. Why might a batch freshness deadline be better than a percentage availability SLO?
2. How do retries affect event-based error-budget accounting?
3. Why does source freshness matter for SLO evaluation?
4. What does 10× burn mean for a 99.9% objective?
5. Why should short and long windows be combined?
6. What is the difference between an alert and an incident?
7. When should a deployment be paused because of error-budget pressure?
8. What evidence proves service recovery beyond a green job status?

## 42. Official references

- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks Jobs monitoring: https://docs.databricks.com/aws/en/jobs/monitor
- Databricks SQL query history: https://docs.databricks.com/aws/en/sql/user/queries/query-history
- Databricks audit logs: https://docs.databricks.com/aws/en/admin/account-settings/audit-logs
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Google SRE workbook alerting: https://sre.google/workbook/alerting-on-slos/
- Google SRE error budgets: https://sre.google/workbook/error-budget-policy/
- AWS CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs/monitoring-overview

**Next:** Chapter 054 — Databricks Incident Response and Root Cause Analysis.
