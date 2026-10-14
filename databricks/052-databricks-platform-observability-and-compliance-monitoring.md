# Chapter 052 — Databricks Platform Observability and Compliance Monitoring

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 11 — Compliance, Observability and Operational Assurance  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform Engineering, Security and Data Operations  
**Prerequisites:** Chapters 041–051  
**Safety:** Production exercises are read-only. Create alerts, change monitors, alter grants or simulate failures only in an authorized isolated staging environment. Audit and billing telemetry can be sensitive and delayed.

## 1. Mission

Build a reliable observability and compliance monitoring practice across Databricks workspaces, jobs, compute, SQL warehouses, Unity Catalog, infrastructure, audit events and cloud dependencies. Convert signals into actionable alerts, SLOs, incident runbooks and auditable evidence.

## 2. Learning outcomes

- Identify Databricks control-plane, data-plane and cloud observability sources.
- Query system tables for billing, jobs, audit and compute where enabled and authorized.
- Define service-level indicators (SLIs), objectives (SLOs) and error budgets.
- Distinguish metrics, logs, traces, audit records and data-quality signals.
- Build safe read-only monitoring queries and synthetic acceptance tests.
- Correlate job failures with cloud capacity, access changes and deployment events.
- Design actionable alerting and compliance dashboards.
- Manage monitoring permissions, retention, freshness and evidence.

## 3. Reference architecture

    Databricks system tables / APIs / SQL query history
                   |
    Cloud metrics / logs / IAM / networking / storage
                   |
        Jobs, compute, pipeline and UC signals
                   |
          Central collection and normalization
                   |
       Dashboards + SLO evaluation + audit review
                   |
        Alerts -> on-call -> incident runbook
                   |
        Evidence archive and corrective actions

A green control-plane API does not prove a data pipeline is correct. Monitor business-facing freshness and data invariants as well as platform availability.

## 4. Telemetry sources

| Source | Examples | Caveat |
|---|---|---|
| System tables | Audit, billing usage, job timeline, compute and query history | Availability, schema, lag and retention vary |
| Jobs API | Run state, task failures, retries, queue time | Permission-filtered and paginated |
| Clusters/compute API | Cluster configuration and status | Not a complete OS-level metrics feed |
| SQL warehouses | Query history, latency, concurrency | Depends on permissions and available history |
| Unity Catalog | Grants, ownership, lineage and access controls | Requires governed privileges |
| Cloud monitoring | Network, VM, storage, KMS, quotas | Account/region integration needed |
| Synthetic tests | Data correctness, availability, freshness | Must use isolated deterministic fixtures |
| GitLab/Terraform | Changes, deployments, drift and approvals | Not direct proof of runtime health |

## 5. Define service boundaries

Document the monitored workspace, region, cloud account, identity, critical jobs, warehouses, catalogs, upstream/downstream systems, customer-facing dependencies, support owner and runbook. Distinguish a platform SLO from a data-product SLO.

## 6. SLIs and SLOs

| SLI | Calculation | Example operational purpose |
|---|---|---|
| Job success rate | Successful required runs / eligible runs | Pipeline reliability |
| Data freshness lag | Observation time minus latest accepted event time | Consumer freshness |
| Query latency | P50/P95/P99 for a defined workload | Interactive experience |
| Queue time | Time from submission to execution | Capacity pressure |
| Retry rate | Retried tasks / eligible tasks | Instability |
| Authorization failures | Denied requests per principal/resource | Permission regression |
| Compute startup | Provisioning time per approved workload | Capacity readiness |
| Cost anomaly | Current usage vs approved budget/baseline | FinOps guardrail |

Set targets using service requirements and measured history, not invented universal thresholds. Define numerator, denominator, exclusion rules, evaluation window and owner for each SLO.

## 7. Monitoring data trust

System tables are not necessarily real-time. Check regional availability, workspace enablement, required privileges, event delay and retention. Some tables can be unavailable in a particular account or cloud. Record `MAX(event_time)` or corresponding timestamp and ingestion delay before using results for incident decisions. Never treat missing rows as proof of no incidents.

## 8. Lab A — Windows PowerShell preflight [SAFE-READ]

~~~powershell
git --version
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
~~~

Confirm workspace host, account and principal. Use approved profiles; do not put tokens in command history.

## 9. Lab B — Discover system catalogs [SAFE-READ]

In an authorized Databricks SQL warehouse:

~~~sql
SELECT current_user();
SHOW CATALOGS;
SHOW SCHEMAS IN system;
SHOW TABLES IN system.billing;
SHOW TABLES IN system.access;
~~~

Catalog/schema/table availability depends on account features and privileges. If access is denied, request governed read access; do not grant yourself broader privileges.

## 10. Lab C — Billing usage baseline [SAFE-READ]

~~~sql
SELECT
  usage_date,
  sku_name,
  SUM(usage_quantity) AS usage_units
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 7)
GROUP BY usage_date, sku_name
ORDER BY usage_date DESC, sku_name;
~~~

Usage units differ by SKU and are not equivalent to dollars. For cost, use approved price references, applicable contracts, usage metadata and currency rules; do not sum different units as if they were a single currency.

## 11. Lab D — Billing attribution [SAFE-READ]

~~~sql
SELECT
  workspace_id,
  sku_name,
  SUM(usage_quantity) AS usage_units
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 7)
GROUP BY workspace_id, sku_name
ORDER BY usage_units DESC;
~~~

Validate tags and attribution completeness before allocating cost. Billing events can arrive late.

## 12. Lab E — Audit event exploration [SAFE-READ]

~~~sql
SELECT
  event_time,
  workspace_id,
  service_name,
  action_name
FROM system.access.audit
WHERE event_date >= date_sub(current_date(), 1)
ORDER BY event_time DESC
LIMIT 100;
~~~

Restrict access to user identities, IP addresses, request parameters and other sensitive audit fields. Confirm column names in the account's current schema.

## 13. Lab F — Privileged-action review [SAFE-READ]

~~~sql
SELECT
  event_time,
  service_name,
  action_name,
  COUNT(*) OVER (PARTITION BY service_name, action_name) AS matching_events
FROM system.access.audit
WHERE event_date >= date_sub(current_date(), 7)
  AND (
    lower(action_name) LIKE '%permission%'
    OR lower(action_name) LIKE '%grant%'
    OR lower(action_name) LIKE '%policy%'
  )
ORDER BY event_time DESC
LIMIT 100;
~~~

This is a heuristic discovery query, not a complete privileged-action detector. Build an approved, versioned action-name mapping from observed audit schema and product documentation; correlate with change tickets.

## 14. Lab G — Jobs timeline discovery [SAFE-READ]

~~~sql
SHOW TABLES IN system.lakeflow;
DESCRIBE TABLE system.lakeflow.job_run_timeline;
SELECT *
FROM system.lakeflow.job_run_timeline
LIMIT 10;
~~~

Do not assume table availability or the exact state column names across deployments. Inspect schema first. Scope production reporting to permitted workspaces and avoid exposing sensitive job metadata.

## 15. Lab H — Jobs API recent runs [SAFE-READ]

~~~powershell
databricks jobs list-runs --limit 25 --output json
databricks jobs list --output json
~~~

Collect run ID, job ID, status, duration, retry context and timestamps through approved APIs. Handle pagination and permissions. Correlate with GitLab deployment and audit event timelines.

## 16. Lab I — SQL query history discovery [SAFE-READ]

~~~sql
SHOW TABLES IN system.query;
DESCRIBE TABLE system.query.history;
SELECT *
FROM system.query.history
LIMIT 10;
~~~

Query text and identity can be sensitive. Limit access, avoid broad exports and confirm retention, scope and table availability.

## 17. Lab J — Synthetic freshness check [SAFE-READ]

~~~sql
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT event_id) AS unique_events,
  SUM(CASE WHEN event_id IS NULL THEN 1 ELSE 0 END) AS null_ids,
  MAX(event_time) AS latest_event_time
FROM demo_catalog.demo_staging.synthetic_events;
~~~

Compare against a deterministic fixture manifest and declared freshness SLA. The example requires an existing approved synthetic staging table.

## 18. Compute and cluster observability

Track cluster creation errors, startup time, autoscaling decisions, node loss, executor failures, out-of-memory events, CPU/memory pressure, Spark spill, disk and network saturation. Correlate cloud VM capacity and quotas with Databricks run failures. Avoid assuming a running cluster means a healthy workload.

## 19. SQL warehouse observability

Monitor warehouse availability, query queue time, P95/P99 latency, concurrency, timeouts, failed queries, autoscaling and credit/DBU consumption. Segment by workload and warehouse rather than combining incompatible latency populations.

## 20. Unity Catalog security observability

Watch unexpected ownership/grant changes, privileged actions, denied requests, external locations, storage credentials and service-principal use. Verify effective authorization using approved principals. An audit event alone is not proof of a successful privilege escalation.

## 21. Data quality and correctness

Define invariants for row count, uniqueness, nullability, schema, accepted event time, checkpoint continuity and downstream consumption. Use immutable expected fixtures and guard against duplicate processing after retries. A successful job run can still produce incorrect data.

## 22. Cloud-specific monitoring

**AWS:** CloudWatch, CloudTrail, VPC Flow Logs, S3/KMS, IAM, EC2 capacity and service quotas.

**Azure:** Azure Monitor, Activity Logs, Network Watcher, Private Link, ADLS, Key Vault, managed identity and quota telemetry.

**GCP:** Cloud Monitoring, Cloud Logging/Audit Logs, VPC Flow Logs, GCS/KMS, IAM and regional quotas.

Correlate cloud resource IDs, Databricks workspace IDs and deployment timestamps.

## 23. Dashboard design

Create views for:
- Executive service health and error budget.
- Jobs: success, duration, queue, retry and failures.
- Compute: startup, resource pressure, scale and capacity.
- SQL: P95/P99, concurrency, queue and failures.
- Data products: freshness, completeness and duplicates.
- Security: access denials, privileged actions and drift.
- Cost: SKU usage, spend attribution and anomalies.
- Detector health: ingestion lag, query errors and alert delivery.

Every dashboard needs a defined owner, timezone, refresh interval and source freshness indicator.

## 24. Alert design

An alert should contain environment, workspace, affected resource, severity, observed value, threshold, evaluation window, impact, runbook, owner and correlation link. Deduplicate noisy events and separate symptom alerts from underlying causes. Alert on monitoring pipeline failure independently.

## 25. Error budget and burn rate

For a reliability SLO, compute budget from the eligible event population and objective. Use both short and long windows for meaningful burn alerts, and document exclusion rules. Do not equate all failed task attempts with failed user-facing jobs if retries recovered within the SLO.

## 26. Monitoring deployment and permissions

Deploy monitors through an approved change-controlled process. Keep monitoring read-only where feasible, separate alert administration from observation, restrict system-table grants, and validate that queries do not unintentionally scan sensitive unrestricted datasets.

## 27. Staging alert acceptance exercise

In an isolated staging environment:
1. Define a benign synthetic metric or controlled test failure.
2. Verify alert threshold and evaluation window.
3. Trigger only the synthetic condition.
4. Confirm delivery to the designated staging channel.
5. Confirm runbook and ownership metadata.
6. Clear the condition and verify resolution.
7. Archive redacted evidence.

Do not trigger production paging or deliberately fail real customer pipelines.

## 28. Correlation workflow

For a reported failed job:
1. Establish UTC failure onset and run ID.
2. Check job/task error and retries.
3. Inspect recent approved GitLab/bundle/Terraform deployments.
4. Check relevant Databricks audit actions and grant changes.
5. Check compute provisioning and cloud quota/network/storage.
6. Validate data invariants and downstream freshness.
7. Determine impact, remediation owner and incident severity.
8. Preserve restricted evidence.

## 29. Scenario A — System table missing

**Detect:** table not found or insufficient privileges.  
**Response:** check regional feature support, enablement, schema and authorized access.  
**Pass:** supported source available or documented alternate telemetry; never silently report zero incidents.

## 30. Scenario B — Audit events delayed

**Detect:** latest event time lags actual activity.  
**Response:** measure freshness and corroborate with cloud/workspace APIs.  
**Pass:** alert thresholds account for ingestion delay and freshness is visible.

## 31. Scenario C — Job succeeds but data is stale

**Detect:** job success status green while freshness SLI fails.  
**Response:** inspect upstream ingestion, checkpoints and consumer lag.  
**Pass:** accepted data freshness restored.

## 32. Scenario D — SQL latency increases

**Detect:** P95/P99 and queue time rise for a defined workload.  
**Response:** inspect concurrency, warehouse sizing, query patterns and cloud pressure.  
**Pass:** workload-specific latency returns within approved SLO.

## 33. Scenario E — Compute provisioning fails

**Detect:** cluster start errors, quota or unavailable instance types.  
**Response:** verify region, quotas, node policies, IAM and networking.  
**Pass:** authorized workload starts and meets capacity baseline.

## 34. Scenario F — Unexpected privileged audit action

**Detect:** grant/policy action without matching change ticket.  
**Response:** investigate actor, resource, effective permissions and impact; escalate as warranted.  
**Pass:** least privilege restored or change formally approved.

## 35. Scenario G — Cost anomaly after deployment

**Detect:** usage deviates from approved baseline.  
**Response:** inspect SKU, cluster uptime, retries, schedules and attribution.  
**Pass:** explained approved spend or corrective change validated.

## 36. Scenario H — Alert storm

**Detect:** duplicate notifications across many jobs.  
**Response:** correlate shared cause, deduplicate and apply controlled grouping.  
**Pass:** actionable alert volume without hiding critical incidents.

## 37. Scenario I — Monitoring principal loses access

**Detect:** collection query fails with permission error.  
**Response:** verify intended grants and recent identity changes; restore only approved access.  
**Pass:** collector healthy with least privilege.

## 38. Scenario J — Dashboard shows zero due to failed ingestion

**Detect:** empty data and stale freshness watermark.  
**Response:** mark dashboard unavailable, alert on collection failure, use alternate evidence.  
**Pass:** recovered ingestion and reconciled historical gaps.

## 39. Production observability incident runbook

1. Confirm incident scope and severity.
2. Validate telemetry source health and freshness.
3. Identify affected workspace, jobs, warehouse and data products.
4. Establish UTC onset and deployment/audit timeline.
5. Check run failures, capacity, security and data correctness.
6. Correlate AWS/Azure/GCP dependencies.
7. Escalate security/data integrity incidents immediately.
8. Contain unsafe writers or changes only with authorization.
9. Apply reviewed recovery and validate consumer SLO.
10. Document evidence, gaps, owner and corrective actions.

## 40. Compliance review runbook

Review monitoring coverage, system-table permissions, audit retention, identity changes, grant changes, policy drift, encryption/network controls, cost anomalies, detector failures and open exceptions. Link each finding to an owner and time-bound remediation ticket. Never infer compliance solely from a green dashboard.

## 41. Evidence package

    observability-review-YYYYMMDD/
      01-scope-and-service-map.md
      02-source-availability-and-lag.md
      03-sli-slo-definitions.md
      04-jobs-and-compute-summary.md
      05-sql-and-data-freshness.md
      06-security-audit-summary.md
      07-cloud-correlation.md
      08-alert-delivery-test.md
      09-cost-and-capacity.md
      10-open-findings-and-exceptions.md
      11-runbook-and-recovery.md
      12-final-acceptance.md

Store raw audit records, sensitive query text and privileged inventory only in approved restricted storage.

## 42. Acceptance scorecard

| Gate | Required evidence |
|---|---|
| Source coverage | Known enabled system tables/APIs and cloud signals |
| Source freshness | Latest timestamps and lag documented |
| Identity | Read-only monitoring access validated |
| Jobs | Success, duration, queue and retry SLIs |
| Compute | Startup, resource pressure and quota coverage |
| SQL | Query failure and latency baselines |
| Data | Freshness, uniqueness and correctness |
| Security | Grant/audit monitoring and response |
| Cost | Usage baseline and attribution |
| Alerting | Benign staging notification test |
| Detector health | Failure alert and freshness indicator |
| Runbooks | Owners, escalation and recovery |
| Evidence | Retention and restricted access |
| SLOs | Approved objectives and evaluation windows |

## 43. GO / CONDITIONAL GO / NO-GO

**GO:** Mandatory signals are available and fresh, SLOs are defined, critical alert paths work, security and data correctness checks pass and runbooks are owned.

**CONDITIONAL GO:** Only explicitly permitted noncritical gaps remain with accountable owner, time-bound exception and compensating monitoring.

**NO-GO:** Unknown monitoring blind spot for critical services, stale/missing audit coverage without fallback, failed data correctness, unapproved privileged changes or missing critical paging.

## 44. Review questions

1. Why is a successful Databricks API call not proof of end-to-end pipeline health?
2. How do system-table ingestion delays affect incident response?
3. What is the difference between a job success SLI and a data freshness SLI?
4. Why must billing usage units be interpreted by SKU?
5. How should audit data be protected?
6. Why is monitoring pipeline health a separate alert?
7. How do you correlate a job failure with a recent infrastructure deployment?
8. Which evidence is required to approve production observability readiness?

## 45. Official references

- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- System billing usage: https://docs.databricks.com/aws/en/admin/system-tables/billing
- Audit logs: https://docs.databricks.com/aws/en/admin/account-settings/audit-logs
- Jobs monitoring: https://docs.databricks.com/aws/en/jobs/monitor
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Unity Catalog privileges: https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/
- Databricks SQL query history: https://docs.databricks.com/aws/en/sql/user/queries/query-history
- AWS CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs/monitoring-overview

**Next:** Chapter 053 — SLO Engineering, Alert Quality and Incident Readiness.
