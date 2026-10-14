# Chapter 033 — Databricks Log Collection, Audit Events and Cross-Cloud Telemetry Integration

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 07 — Monitoring, Observability and Incident Response  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, Platform Engineering, Cloud Security, Compliance and Incident Response  
**Prerequisites:** Chapters 019, 031–032  
**Safety:** Production inspection is read-only. Telemetry configuration and synthetic event generation require approved staging change control.

## 1. Mission

Establish an operationally reliable log and audit architecture for Databricks. Collect the right evidence, protect sensitive content, correlate events across platform and cloud boundaries, and detect missing telemetry before an incident makes investigation impossible.

### Learning objectives

- Distinguish driver/executor logs, compute events, job/task output, audit events and cloud logs.
- Understand delivery delay, retention, access and regional boundaries.
- Inventory available system tables and supported log-delivery options.
- Build an event correlation model using UTC and stable identifiers.
- Validate centralized ingestion and alerting with approved staging tests.
- Troubleshoot audit gaps, duplicate events, access denials and log cost growth.
- Define evidence handling, incident runbooks and acceptance criteria.

## 2. Reference architecture

    Users / service principals / jobs
                  |
          Databricks workspace
           /       |        \
     Job runs   Compute     Unity Catalog
           \       |        /
         Logs, events and audit
                  |
          System tables / APIs
                  |
     Cloud provider audit and metrics
       /           |           \
    CloudTrail  Azure logs   Cloud Audit Logs
                  |
        Collection / forwarding
                  |
       Normalization and redaction
                  |
        SIEM / observability store
                  |
      Dashboards, alerts and incidents
                  |
       Evidence retention and review

Databricks telemetry does not replace cloud-provider audit logging. Cloud audit logs do not capture every Databricks-level action.

## 3. Telemetry source taxonomy

| Source | Typical content | Reliability limitation |
|---|---|---|
| Job run/task metadata | State, attempts, errors, timing | May not contain full execution context |
| Driver logs | Runtime errors and application messages | Sensitive content and lifecycle limits |
| Executor logs | Worker failures, JVM errors, task details | High volume and short-lived compute |
| Cluster events | Provisioning, resizing, termination | Not a substitute for host metrics |
| Spark UI/event data | Stage, task, shuffle, executor diagnostics | Availability depends on retention/config |
| Databricks audit events | Governance and administrative activity | Privilege, coverage and ingestion latency |
| Cloud control-plane logs | IAM, resource and policy changes | Different identities and schemas |
| Cloud data-plane logs | Storage/object access where enabled | Often optional, high volume and cost |

## 4. Evidence quality requirements

Every investigation record should identify:
- UTC event time and collection time.
- Workspace/account and cloud region.
- Job ID, run ID, task key and compute ID where relevant.
- Actor or service principal (access-controlled).
- Event source and action.
- Correlation identifier or defensible join keys.
- Log delivery lag and completeness caveat.
- Redaction classification and retention policy.

Do not invent a universal correlation ID across Databricks and all cloud services. Join using available stable identifiers, time windows and documented confidence.

## 5. Centralization design decisions

Evaluate:
1. Native system table availability and retention.
2. Supported Databricks log-delivery mechanisms for the cloud/deployment.
3. Cloud-native routing and export mechanisms.
4. Ingestion pipeline availability, backpressure and dead-letter handling.
5. Schema versioning, deduplication and replay.
6. Sensitive-field filtering and access segregation.
7. Regional residency and cross-border transfer restrictions.
8. Query cost, storage tiering and legal retention.
9. Alert delivery and recovery ownership.

Avoid collecting full application payloads merely because the logging framework permits them.

## 6. Lab A — Windows PowerShell identity preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm the intended workspace and role. Never paste authentication tokens into tickets or public repositories.

## 7. Lab B — Job and compute evidence inventory

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks jobs get-run --help
databricks clusters events --help
~~~

Inspect supported CLI arguments before querying an individual run. Record whether output is permission-filtered or paginated.

### Run evidence worksheet

| Field | Finding |
|---|---|
| Workspace / region | |
| Job / run / task | |
| Compute / runtime | |
| Start/end UTC | |
| Error class | |
| Driver/executor log location | |
| Related cloud event | |
| Collection latency | |

## 8. Lab C — Databricks audit system table discovery

From an authorized SQL warehouse:

~~~sql
SHOW SCHEMAS IN system;
SHOW TABLES IN system.access;
DESCRIBE TABLE system.access.audit;
~~~

Where the audit table is available and permissioned:

~~~sql
SELECT
  event_time,
  service_name,
  action_name,
  COUNT(*) AS event_count
FROM system.access.audit
WHERE event_time >= current_timestamp() - INTERVAL 1 DAY
GROUP BY event_time, service_name, action_name
ORDER BY event_time DESC
LIMIT 100;
~~~

This query groups by exact event timestamp and is useful for inspecting raw event patterns, not as a low-cardinality dashboard. For trend charts, aggregate into explicit time buckets supported by the SQL engine.

## 9. Lab D — Time-bucketed audit event baseline

~~~sql
SELECT
  date_trunc('hour', event_time) AS event_hour,
  service_name,
  action_name,
  COUNT(*) AS events
FROM system.access.audit
WHERE event_time >= current_timestamp() - INTERVAL 7 DAYS
GROUP BY date_trunc('hour', event_time), service_name, action_name
ORDER BY event_hour DESC, events DESC;
~~~

Validate actual column names and permissions. System table availability and ingestion delay vary; zero returned rows does not necessarily mean no events occurred.

## 10. Lab E — Investigate administrative changes

For a known UTC incident window, review:
- Identity and group changes.
- Job or cluster configuration modifications.
- Storage credential and external location changes.
- Unity Catalog grants and ownership changes.
- Network and cloud IAM policy changes.
- Audit pipeline and logging configuration changes.

Use the supported audit schema and filter by a bounded time window. Limit access to sensitive identity and request fields.

## 11. Lab F — AWS audit and logging context

~~~powershell
aws sts get-caller-identity
aws configure get region
aws cloudtrail describe-trails
~~~

If authorized, inspect CloudTrail trail/event data store coverage, CloudWatch log groups, S3 data events where enabled, retention and cross-account forwarding.

CloudTrail management events and S3 object-level data events have different configuration and billing characteristics.

## 12. Lab G — Azure audit and logging context

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az monitor diagnostic-settings --help
~~~

Review Azure Activity Logs, Databricks diagnostic settings where applicable, Log Analytics destinations, storage/network diagnostics and managed identity events. Use read-only commands with explicit authorized resource scopes.

## 13. Lab H — GCP audit and logging context

~~~powershell
gcloud config get-value project
gcloud logging sinks list
~~~

Review Cloud Audit Logs, relevant Data Access logging configuration, logging sinks, destination IAM and retention. Data Access log availability and defaults differ by service; verify current provider configuration.

## 14. Lab I — Offline correlation exercise

~~~powershell
$events = @(
  [pscustomobject]@{TimeUtc="2026-10-01T10:00:00Z";Source="Databricks";Type="JOB_FAILURE";Reference="run-101"},
  [pscustomobject]@{TimeUtc="2026-10-01T09:59:45Z";Source="CloudAudit";Type="IAM_POLICY_CHANGE";Reference="change-17"},
  [pscustomobject]@{TimeUtc="2026-10-01T10:00:05Z";Source="Storage";Type="ACCESS_DENIED";Reference="bucket-demo"}
)
$events | Sort-Object TimeUtc | Format-Table -AutoSize
~~~

These are synthetic records. Temporal proximity alone is not proof that the IAM change caused the job failure. Confirm affected principal, resource and error semantics.

## 15. Lab J — Telemetry completeness calculation

~~~powershell
$expected = 500
$received = 493
$duplicate = 4
$uniqueReceived = $received - $duplicate
[pscustomobject]@{
  Expected = $expected
  Received = $received
  UniqueReceived = $uniqueReceived
  CoveragePct = [math]::Round(100 * $uniqueReceived / $expected, 2)
}
~~~

This model assumes the expected count is authoritative and duplicates are identifiable. In practice, define expected event classes and tolerate documented delivery delays before declaring loss.

## 16. Lab K — Staging end-to-end event test

**Preconditions:** approved staging workspace, test principal, synthetic resource, test alert destination, rollback owner.

1. Record collection pipeline baseline and UTC time.
2. Perform one approved benign auditable action.
3. Capture source-side evidence and event identifiers.
4. Wait for documented ingestion latency.
5. Confirm event appears in the authorized central destination.
6. Verify actor, action, resource, UTC and redaction.
7. Confirm any expected test alert reaches the correct destination.
8. Check duplicate handling and replay strategy.
9. Restore staging changes and record delivery lag.

Do not intentionally change production permissions to test audit coverage.

## 17. Retention and privacy controls

Define separate retention policies for:
- Operational job and compute logs.
- Security audit records.
- Cloud management/data-plane events.
- Incident evidence and legal holds.
- Derived metrics and aggregates.

Apply least-privilege access, encryption, documented residency, redaction and secure deletion. Avoid logging secrets, tokens, patient/customer data or raw credentials.

## 18. Incident A — Missing Databricks audit events

**Symptoms:** expected actions absent from system tables or central SIEM.

**Triage:** validate event type, permissions, source availability, ingestion delay, filters and retention.

**Recovery:** restore supported collection path; document unrecoverable gaps.

**Acceptance:** staging synthetic event visible end-to-end.

## 19. Incident B — Executor logs disappear after cluster termination

**Symptoms:** runtime investigation lacks worker-side evidence.

**Triage:** check configured log delivery, retention, ephemeral compute lifecycle and access.

**Recovery:** enable approved persistent delivery for future runs; use remaining Spark/job evidence for current incident.

**Acceptance:** subsequent staging failures retain diagnostic logs.

## 20. Incident C — Duplicate events inflate security alerts

**Symptoms:** repeated alerts for one underlying change.

**Triage:** inspect at-least-once delivery, forwarding loops, replay and event IDs.

**Recovery:** correct routing and implement source-aware deduplication.

**Acceptance:** one incident notification per qualifying event while preserving raw audit records.

## 21. Incident D — Cloud audit logs missing object-level access

**Symptoms:** IAM management events present but storage data operations absent.

**Triage:** inspect data-event logging configuration, service coverage, filters, IAM and destination.

**Recovery:** enable approved scoped data events where supported; assess cost and retention.

**Acceptance:** synthetic authorized data access produces expected event.

## 22. Incident E — Logging pipeline backlog

**Symptoms:** audit events arrive late or dashboards show stale data.

**Triage:** measure source-to-destination lag, queue depth, throttling, collector failures and dead-letter counts.

**Recovery:** restore ingestion capacity and replay supported buffered events.

**Acceptance:** lag returns within target and backlog drains without unacceptable loss.

## 23. Incident F — Sensitive data leaked into logs

**Symptoms:** tokens, credentials or regulated content appear in shared telemetry.

**Triage:** restrict access, preserve forensic evidence securely and identify exposure scope.

**Recovery:** rotate affected credentials as required, correct logging/redaction, apply approved retention and incident policy.

**Acceptance:** staging tests confirm sensitive fields are masked and access controls enforced.

## 24. Troubleshooting decision tree

    Required event not visible
              |
    Did source generate event?
      no -> source capability/config
      yes
              |
    Is source query permitted?
      no -> access/visibility
      yes
              |
    Delivery delay within expected window?
      yes -> wait and re-evaluate
      no
              |
    Forwarder/queue healthy?
      no -> ingestion/backpressure
      yes
              |
    Filter, schema or destination issue?
      yes -> routing/normalization
      no -> provider support investigation

## 25. Production incident evidence runbook

1. Declare incident and preserve UTC timeline.
2. Identify workspace, cloud, region and affected principals.
3. Retrieve authorized job, compute and audit evidence.
4. Correlate cloud IAM, network and storage events.
5. Verify telemetry completeness and delay caveats.
6. Secure and redact evidence with access controls.
7. Determine failure boundary and apply approved recovery.
8. Validate restored service and log/alert delivery.
9. Document RCA, gaps, owners and corrective actions.

## 26. Dashboards and alerting

### Collection health

- Source event volume by class.
- Ingestion lag p50/p95/p99.
- Missing expected heartbeat or synthetic events.
- Collector errors, throttling and queue depth.
- Dead-letter volume and replay success.

### Security and operations

- Privileged configuration changes.
- Unusual authorization denials.
- Storage credential and external location changes.
- Cluster/job failures correlated with infrastructure events.
- Audit coverage gaps by workspace/region.

Alert on confirmed high-impact coverage loss or security events; avoid noisy pages for expected minor delivery jitter.

## 27. Cost and capacity

Log cost drivers include ingestion volume, retention, indexing, high-cardinality fields, data-plane audit scope, cross-region transfer and query frequency.

Forecast:
- Daily events by source and class.
- Peak incident burst multiplier.
- Compression/index overhead.
- Retention and archive storage.
- Cross-region egress and security controls.
- Reprocessing and backfill capacity.

Do not cut critical audit coverage solely to meet a cost target without security and compliance approval.

## 28. Change and rollback

Before modifying log sinks, diagnostic settings, filters or SIEM parsers:
- Export approved current configuration.
- Identify affected workspaces and cloud regions.
- Test in staging with synthetic events.
- Define completeness, lag and duplicate acceptance criteria.
- Establish rollback and buffered-event replay plan.
- Verify permissions and sensitive-data masking.
- Monitor source-to-destination delivery after change.

## 29. Operational acceptance checklist

- [ ] Databricks and cloud telemetry sources inventoried.
- [ ] Driver/executor log lifecycle and retention understood.
- [ ] Audit system table availability verified.
- [ ] Cloud management and data-event coverage documented.
- [ ] UTC correlation and identifier model defined.
- [ ] Central ingestion health metrics available.
- [ ] Staging event delivered end-to-end.
- [ ] Duplicate, lag and missing-event controls tested.
- [ ] Sensitive data redaction and access verified.
- [ ] Six incident scenarios reviewed.
- [ ] Evidence handling and rollback runbooks approved.
- [ ] No unapproved production telemetry mutation performed.

## 30. Review questions

1. Why are Databricks audit logs and cloud audit logs complementary?
2. Why does a missing event not immediately prove that no action occurred?
3. How do you correlate cloud IAM changes with Databricks job failures?
4. Why is at-least-once delivery relevant to security alert deduplication?
5. What controls are needed when executor logs contain sensitive data?

## 31. Official references

- Databricks audit logs: https://docs.databricks.com/aws/en/admin/account-settings/audit-logs
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- AWS CloudTrail: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Logging: https://cloud.google.com/logging/docs
- Google Cloud Audit Logs: https://cloud.google.com/logging/docs/audit

**Next:** Chapter 034 — Databricks Alerting, SLOs and Incident Response Automation.
