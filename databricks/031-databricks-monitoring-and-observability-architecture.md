# Chapter 031 — Databricks Monitoring and Observability Architecture

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 07 — Monitoring, Observability and Incident Response  
**Clouds:** AWS, Azure, GCP  
**Audience:** SRE, DBRE, Platform Engineering, Security and FinOps  
**Prerequisites:** Chapters 001–030  
**Safety:** Production labs are read-only. Alert simulations and telemetry changes require approved staging resources.

## 1. Mission

Build an end-to-end observability architecture for Databricks that connects **user-visible reliability**, workspace control operations, compute health, job execution, governance events, cloud infrastructure, and cost.

The objective is not merely to collect metrics: operators must be able to answer **what failed, who is affected, when it started, which dependency changed, and whether recovery succeeded**.

### Learning objectives

- Separate control-plane, compute-plane, workload, data and cloud signals.
- Identify authoritative telemetry sources and their blind spots.
- Inspect job failures and run timelines with Databricks CLI.
- Query accessible Databricks system tables safely.
- Correlate cloud metrics, audit logs and workflow outcomes.
- Define service-level indicators and actionable alerts.
- Design dashboards, evidence retention and escalation paths.
- Validate monitoring through synthetic staging incidents.

## 2. Observability reference architecture

    Users / APIs / scheduled workflows
                    |
           Reliability SLIs and SLOs
                    |
      Databricks workspace and jobs
            /           \
      Job runs        SQL warehouses
            \           /
       Compute and Spark metrics
                    |
     System tables / audit / events
                    |
    Cloud infrastructure telemetry
        /          |           \
    CloudWatch  Azure Monitor  Cloud Monitoring
                    |
        Collection / normalization
                    |
     Dashboards / alerts / incidents
                    |
         Runbooks and postmortems

Do not assume all system tables, metrics, APIs, or event sources are enabled in every account, region, or workspace.

## 3. Five observability layers

| Layer | Primary questions | Evidence |
|---|---|---|
| Service experience | Are jobs completing on time? | Success, latency, freshness |
| Databricks control plane | Are workspaces, jobs and APIs operating? | API errors, run states, service status |
| Compute and runtime | Are clusters healthy and adequately sized? | CPU, memory, Spark events, executor failures |
| Data and storage | Is data available, fresh and correct? | Delta history, object errors, quality checks |
| Cloud dependencies | Are network, IAM, storage and quotas healthy? | Provider logs, network metrics, audit events |

A healthy cluster does not prove a healthy pipeline; a successful job does not prove correct data.

## 4. Core telemetry sources

### Databricks-native

- Jobs API and run/task states.
- Job run details, task attempts and error messages.
- Cluster and compute events.
- Spark UI, task/stage metrics and executor logs where accessible.
- SQL warehouse query history and performance telemetry.
- Account/workspace audit events.
- Databricks system tables, subject to availability and permissions.
- Delta table history and data validation metrics.

### Cloud-native

- AWS: CloudWatch, CloudTrail, VPC/network telemetry and S3 metrics.
- Azure: Azure Monitor, Log Analytics, Activity Logs and storage/network diagnostics.
- GCP: Cloud Monitoring, Cloud Logging, Cloud Audit Logs and network/storage telemetry.

Cloud services have different data retention, sampling, latency and pricing models. Document these explicitly.

## 5. Signal design: metrics, logs, traces and events

| Signal | Best use | Limitation |
|---|---|---|
| Metrics | Trends, SLOs and alert thresholds | Often lack request-level context |
| Logs | Errors, stack traces and audit | High cardinality and sensitive content |
| Traces | Cross-service request paths | Not universally available end-to-end |
| Events | State transitions and changes | May be delayed or incomplete |
| Data quality | Correctness and freshness | Requires workload-specific checks |

Correlate with UTC timestamps, workspace ID, job ID, run ID, task key, cluster/warehouse ID, cloud region and change ticket where permitted.

## 6. Golden signals for Databricks SRE

- **Availability:** successful eligible runs / eligible runs.
- **Latency:** end-to-end completion time, queue delay and startup time.
- **Errors:** failed tasks, retries, API failures, authorization errors.
- **Saturation:** compute utilization, concurrency, quotas, throttling.
- **Data correctness:** completeness, duplicate rate, schema expectations.
- **Freshness:** time since last validated successful dataset update.
- **Cost:** spend per validated workflow and unexpected growth.

Avoid treating an automatic retry that eventually succeeds as a fully healthy experience if it breaches deadlines or consumes excessive resources.

## 7. Service-level indicators and objectives

### Example definitions

| SLI | Numerator / denominator or measure | Example objective |
|---|---|---|
| Critical job success | Successful eligible scheduled runs / eligible runs | Set from business requirement |
| Deadline compliance | Runs completed by deadline / eligible runs | Set from delivery commitment |
| Data freshness | Age of last validated update | Per dataset |
| Recovery time | Incident detection to validated restoration | Per service tier |
| Alert delivery | Delivered actionable alerts / injected qualifying incidents | Measured in staging |

The table deliberately avoids universal thresholds. Define exclusions for planned maintenance and user-cancelled runs before calculating SLOs.

## 8. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm the workspace and identity. Do not include secrets or tokens in incident evidence.

## 9. Lab B — Job and run inventory

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks jobs get --help
databricks jobs get-run --help
~~~

Inspect actual CLI syntax with --help. Output may be permission-filtered or paginated; record scope and collection time.

### Inventory worksheet

| Job | Criticality | Schedule | Owner | Deadline | Alert destination |
|---|---|---|---|---|---|
| demo-daily-ingest | Tier 1 | Daily | | | |
| demo-hourly-refresh | Tier 2 | Hourly | | | |

Use generic examples and do not copy production identifiers into public tutorials.

## 10. Lab C — Investigate a failed run

1. Identify a previously failed run in an authorized workspace.
2. Capture run ID, job ID, start/end UTC and lifecycle/result states.
3. Inspect task-level attempts, compute startup and retry history.
4. Review sanitized error details and linked logs.
5. Correlate with cloud metrics and deployment/change history.
6. Classify failure: code/data, compute, permissions, network, storage or service.
7. Record evidence and a recovery decision.

~~~powershell
databricks jobs get-run --help
# After confirming syntax, inspect the approved run ID.
~~~

A failed task that is retried successfully can still produce deadline and cost violations.

## 11. Lab D — Discover available system tables

Run from an authorized SQL warehouse:

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN system;
~~~

Where access is granted and the tables are available:

~~~sql
SHOW TABLES IN system.billing;
SHOW TABLES IN system.access;
SHOW TABLES IN system.compute;
~~~

System schemas, columns and availability vary. Check actual schema before authoring production queries.

## 12. Lab E — Query billable usage safely

Where available and authorized:

~~~sql
DESCRIBE TABLE system.billing.usage;

SELECT
  usage_date,
  sku_name,
  SUM(usage_quantity) AS total_usage
FROM system.billing.usage
WHERE usage_date >= current_date() - INTERVAL 7 DAYS
GROUP BY usage_date, sku_name
ORDER BY usage_date DESC, sku_name;
~~~

Usage quantity units differ by SKU; do not interpret a sum across incompatible units as a currency total. Apply current price data separately for monetary cost estimates.

## 13. Lab F — Audit event discovery

~~~sql
SHOW TABLES IN system.access;
~~~

If the audit table is available:

~~~sql
DESCRIBE TABLE system.access.audit;

SELECT event_time, service_name, action_name
FROM system.access.audit
WHERE event_time >= current_timestamp() - INTERVAL 1 DAY
ORDER BY event_time DESC
LIMIT 100;
~~~

Audit tables may be restricted, delayed, or unavailable. Do not assume this captures every cloud-side action; correlate with provider audit logs.

## 14. Lab G — Cloud-native read-only context

~~~powershell
# AWS
aws sts get-caller-identity
aws configure get region

# Azure
az account show --query "{subscription:id,tenant:tenantId}" -o json

# GCP
gcloud config get-value project
~~~

Use only the commands for your deployment. Human CLI identity is not necessarily the Databricks workload identity.

## 15. Lab H — Cloud metrics correlation worksheet

| UTC interval | Databricks signal | Cloud signal | Hypothesis | Confidence |
|---|---|---|---|---|
| | Job startup delay | VM capacity/quota | Capacity unavailable | |
| | Storage read errors | Object storage errors | Storage or IAM | |
| | SQL slowdown | CPU/network/storage | Shared bottleneck | |
| | Auth failures | Cloud audit events | Policy regression | |

Correlation is not proof of causation. Compare unaffected control workloads and recent changes.

## 16. Lab I — Offline SLO calculation

~~~powershell
$eligibleRuns = 1000
$successfulRuns = 987
$deadlineCompliant = 961

[pscustomobject]@{
  SuccessPct = [math]::Round(100 * $successfulRuns / $eligibleRuns, 2)
  DeadlinePct = [math]::Round(100 * $deadlineCompliant / $eligibleRuns, 2)
  FailedRuns = $eligibleRuns - $successfulRuns
  MissedDeadlines = $eligibleRuns - $deadlineCompliant
}
~~~

These are synthetic counts. In a real implementation, define run eligibility, retries, overlapping schedules, partial successes and observation windows.

## 17. Dashboard design

### Executive reliability panel

- Critical job success and deadline compliance.
- Data freshness by dataset tier.
- Active incidents and unresolved SLO risks.
- Cost per validated workload.
- Recovery drill outcomes.

### On-call triage panel

- Failed and retrying jobs by owner.
- Startup/queue/processing duration.
- Cluster health and recent events.
- SQL warehouse latency and concurrency.
- Storage, IAM and network error trends.
- Deployment and configuration changes.

### Platform capacity panel

- Compute utilization and autoscaling.
- Quota and provisioning failures.
- Storage throughput/throttling.
- Job concurrency and queueing.
- Usage and cost trends by workload.

## 18. Alert design principles

Every actionable alert must contain:
- Service or workload and environment.
- Measured symptom and threshold/window.
- Business impact or risk.
- Runbook link and owner.
- Relevant workspace/job/run identifiers.
- Dashboard/evidence links.
- Escalation and suppression policy.

Prefer multi-signal alerts for critical incidents. Avoid paging on transient low-impact failures that self-heal within the SLO.

## 19. Lab J — Staging alert acceptance

**Prerequisites:** approved staging job, synthetic data, on-call test channel and rollback.

1. Define a measurable failure condition.
2. Execute a controlled staging job failure or approved tabletop.
3. Confirm the expected metric/event appears.
4. Confirm alert delivery to the test channel.
5. Verify alert context, ownership and runbook.
6. Resolve the test condition and verify recovery signal.
7. Record alert delay and any missed or duplicate notifications.
8. Restore staging configuration.

Do not inject failures into production to prove monitoring.

## 20. Incident A — Job failures without paging

**Triage:** confirm run events, monitoring collection, filtering, alert thresholds and notification routing.

**Recovery:** repair the missing telemetry/alert link and backfill evidence where supported.

**Acceptance:** controlled staging failure produces one actionable notification.

## 21. Incident B — Cluster saturation hidden by average CPU

**Triage:** inspect per-node CPU, memory, task skew, executor failures, queueing and autoscaling events.

**Recovery:** correct workload skew or capacity configuration through approved change.

**Acceptance:** tail latency and deadline compliance improve.

## 22. Incident C — Cloud storage errors misclassified as Spark bugs

**Triage:** correlate task errors with S3/ADLS/GCS metrics, cloud IAM and network changes.

**Recovery:** restore the failed cloud dependency rather than repeatedly increasing Spark retries.

**Acceptance:** storage errors normalize and output is correct.

## 23. Incident D — System table data appears missing

**Triage:** check permissions, regional availability, ingestion latency, retention and actual table schema.

**Recovery:** correct access/configuration where supported and use alternate authoritative telemetry.

**Acceptance:** coverage documented; no false zero-error conclusion.

## 24. Incident E — High-cost retry storm

**Triage:** inspect job attempts, concurrency, failure type, cluster startup and usage trends.

**Recovery:** contain repeated failures, apply bounded retries and fix root cause.

**Acceptance:** deadline compliance and cost per successful job recover.

## 25. Incident F — Audit event gap during security investigation

**Triage:** verify Databricks audit availability, cloud audit collection, log routing, filters and retention.

**Recovery:** restore collection and document unavailable historical coverage.

**Acceptance:** approved synthetic event visible end-to-end with acceptable latency.

## 26. Troubleshooting decision tree

    Databricks incident or SLO breach
                  |
       Is user impact measurable?
          no -> improve SLI coverage
          yes
                  |
       Are run/task errors visible?
          no -> collection/permissions
          yes
                  |
       Compute or scheduling issue?
          yes -> capacity / quotas / runtime
          no
                  |
       Data/storage/network/auth issue?
          yes -> correlate cloud and UC
          no
                  |
       Data correctness or freshness issue?
          yes -> pipeline and Delta investigation
          no -> investigate external dependencies

## 27. Incident response runbook

1. Declare severity and impacted SLO.
2. Establish UTC start time, scope and affected workloads.
3. Collect job/run/task IDs and current lifecycle states.
4. Correlate compute, system tables, cloud logs and recent changes.
5. Preserve sanitized evidence before mitigation.
6. Choose lowest-risk recovery with rollback.
7. Validate job completion **and** data correctness/freshness.
8. Confirm alert clearance and SLO recovery.
9. Communicate status, remaining risk and follow-up actions.
10. Record RCA, detection gaps and preventive work.

## 28. Telemetry governance

- Use least-privilege access to audit and billing tables.
- Mask secrets, tokens, sensitive query text and identifiers in exported evidence.
- Define metric and log retention based on incident and compliance needs.
- Document ingestion delay and sampling.
- Control cardinality and dashboard query cost.
- Avoid placing confidential data in alert notifications.
- Validate time synchronization and UTC normalization.

## 29. Operational acceptance checklist

- [ ] Five observability layers documented.
- [ ] Critical jobs and owners inventoried.
- [ ] Job run and task failure evidence retrievable.
- [ ] System table availability and access verified.
- [ ] Cloud metric and audit sources mapped.
- [ ] Success, latency, freshness and cost SLIs defined.
- [ ] Dashboard includes end-to-end service impact.
- [ ] Alerts include runbooks and escalation owners.
- [ ] Staging alert delivery and recovery validated.
- [ ] Six incident scenarios reviewed.
- [ ] Evidence retention and access controls approved.
- [ ] No production fault injection performed.

## 30. Review questions

1. Why is job success insufficient as a data reliability SLI?
2. How do you distinguish control-plane and compute-plane failures?
3. Why might system tables fail to provide a complete real-time incident picture?
4. What evidence distinguishes storage throttling from Spark compute saturation?
5. What makes an alert actionable rather than noisy?

## 31. Official references

- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks compute monitoring: https://docs.databricks.com/aws/en/compute/cluster-metrics
- Databricks audit logs: https://docs.databricks.com/aws/en/admin/account-settings/audit-logs
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- AWS CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs

**Next:** Chapter 032 — Databricks Compute Metrics, Spark Observability and Performance Diagnostics.
