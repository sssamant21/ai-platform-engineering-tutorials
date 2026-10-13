# Chapter 019 — Audit Logging, Security Monitoring and Compliance Operations

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 04 — Security, Identity and Governance  
**Audience:** SRE, Platform Security, IAM, SOC, Compliance Engineering  
**Clouds:** AWS, Azure, GCP  
**Lab mode:** Read-only collection, synthetic detections and approved staging exercises  
**Prerequisites:** Chapters 016–018

## 1. Mission

Build reliable, auditable Databricks security operations. An audit event is useful only if it can be collected, protected, correlated, retained, investigated and converted into an actionable response.

### Outcomes

- Identify account, workspace, identity, Unity Catalog and cloud audit sources.
- Define a security event ingestion architecture and integrity controls.
- Query supported audit system tables without assuming universal event coverage.
- Detect authentication failures, privilege drift, secret incidents and unusual administrative activity.
- Monitor audit pipeline health and compliance evidence completeness.
- Practice security incident response, rollback and audit acceptance.

## 2. Audit architecture

~~~text
 Databricks account/workspace activity
       |              |
       |       Unity Catalog events
       |              |
       +------ Databricks audit system tables
                         |
              Governed query / export
                         |
               Security analytics / SIEM
                         |
            Detection, triage, response
                         |
             Restricted evidence store

 Cloud IAM / identity provider / storage logs
                         |
                  Cloud log service
                         |
             SIEM correlation and alerting
~~~

Some events originate in Databricks, while others originate in the cloud provider or identity provider. A single Databricks source does **not** establish complete end-to-end coverage.

## 3. Security event source inventory

| Layer | Evidence | Questions answered |
|---|---|---|
| Databricks account | Account administration and identity events | Who changed account-level access? |
| Databricks workspace | Workspace operations and permissions | Who changed or accessed resources? |
| Unity Catalog | Data governance and authorization events | Which principal attempted an operation? |
| Identity provider | Sign-in, MFA, SCIM, federation | Was identity provisioning or authentication healthy? |
| Cloud IAM | Role assumption, policy and identity changes | Was cloud access granted or denied? |
| Cloud storage | Data access and storage policy logs | Did cloud storage reject or allow access? |
| Network | DNS, firewall, private endpoint telemetry | Did traffic reach the intended endpoint? |
| CI/CD | Deployment and automation identity changes | Did a release change security posture? |

### Coverage matrix

For each source document:
- Owner and access approval.
- Ingestion method and scope.
- Supported event classes.
- Event timestamp and ingestion timestamp.
- Expected delivery delay.
- Retention and archive policy.
- Access controls and encryption.
- Gap detection and incident escalation.
- Data classification and privacy obligations.

## 4. Databricks audit system table considerations

Where enabled and authorized, Databricks provides an audit system table, commonly queried as:

~~~sql
SELECT *
FROM system.access.audit
LIMIT 10;
~~~

Before writing production detections, inspect the current schema, available event types, retention, regional/account scope and privilege requirements. Fields and event coverage can change. Do not assume all cloud, IdP or storage events appear in this table.

### Inspect recent event volume

~~~sql
SELECT
  date(event_time) AS event_date,
  service_name,
  action_name,
  COUNT(*) AS event_count
FROM system.access.audit
WHERE event_time >= current_timestamp() - INTERVAL 7 DAYS
GROUP BY date(event_time), service_name, action_name
ORDER BY event_date DESC, event_count DESC;
~~~

This is an exploratory query, not a complete compliance attestation. Interpret service/action values based on events actually observed.

## 5. Lab A — PowerShell authentication and access preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Replace the example profile with the authorized workspace and verify host and identity. Do not print tokens or export authentication configuration to shared locations.

### Inventory read-only resources

~~~powershell
databricks clusters list --output json
databricks jobs list --output json
databricks warehouses list --output json
~~~

Use results to select **authorized** resources for event correlation. API visibility and pagination can limit completeness.

## 6. Lab B — Audit freshness and ingestion lag

Execute in a SQL editor with permission to read the audit table:

~~~sql
SELECT
  MAX(event_time) AS newest_event_time,
  CURRENT_TIMESTAMP() AS query_time,
  TIMESTAMPDIFF(MINUTE, MAX(event_time), CURRENT_TIMESTAMP()) AS newest_event_age_minutes
FROM system.access.audit
WHERE event_time >= current_timestamp() - INTERVAL 2 DAYS;
~~~

**Caveat:** event age is a proxy, not a guaranteed ingestion-delay measurement. Low event volume, table publication delay, clock issues and filtering can all affect results.

### Detect unusually quiet sources

~~~sql
SELECT
  service_name,
  COUNT(*) AS events_last_24h
FROM system.access.audit
WHERE event_time >= current_timestamp() - INTERVAL 24 HOURS
GROUP BY service_name
ORDER BY events_last_24h DESC;
~~~

Compare with historical baselines and known activity. Absence of an event may indicate no activity, missing coverage or delayed delivery.

## 7. Lab C — Privileged operation review

~~~sql
SELECT
  event_time,
  service_name,
  action_name,
  user_identity
FROM system.access.audit
WHERE event_time >= current_timestamp() - INTERVAL 24 HOURS
ORDER BY event_time DESC
LIMIT 100;
~~~

Review the actual event schema before production use. Identity fields can contain sensitive information and must be handled under access controls.

### Triage worksheet

| Item | Record |
|---|---|
| Event time UTC | |
| Observed action | |
| Identity / principal (restricted) | |
| Source service | |
| Resource class | |
| Approved change reference | |
| Expected or unexpected | |
| Impact assessment | |
| Escalation owner | |

Never assume every administrative event is malicious. Correlate change approvals, maintenance windows and related activity.

## 8. Lab D — Synthetic detection engineering

Use **synthetic events** in an approved staging SIEM/test index. Do not intentionally brute-force authentication or generate unauthorized access attempts.

### Detection candidates

1. Repeated authentication failures for a critical service principal.
2. New privileged grant outside an approved change window.
3. Break-glass administrative activity.
4. Secret or federation trust configuration changes.
5. Unexpected service principal workspace assignment.
6. External location or storage credential modification.
7. Audit source stops producing expected events.
8. Significant audit ingestion lag or retention gap.

For each detection define:
- Signal source and event field mapping.
- Threshold or correlation logic.
- Known benign patterns.
- Severity based on impact.
- Runbook and escalation.
- Evidence handling and privacy.
- Test case and expected outcome.

## 9. Lab E — Cloud audit correlation

### AWS

Potential sources: CloudTrail, CloudWatch Logs, VPC Flow Logs, S3 data events where enabled, and identity provider events.

~~~powershell
aws sts get-caller-identity
aws configure get region
aws cloudtrail describe-trails
~~~

### Azure

Potential sources: Azure Activity Log, Entra sign-in/audit logs, Azure Monitor, storage diagnostic logs and network logs.

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az monitor activity-log list --max-events 10 --output json
~~~

### GCP

Potential sources: Cloud Audit Logs, Cloud Logging, IAM policy history and VPC Flow Logs where enabled.

~~~powershell
gcloud config get-value project
gcloud logging read 'severity>=ERROR' --limit=10 --format=json
~~~

All commands require authorized cloud credentials. Some logging types are disabled by default or have additional cost. **Do not** assume an empty result means there was no security event.

## 10. Correlation model

~~~text
Incident UTC timeline
  |
  +-- Databricks audit event
  +-- Identity provider sign-in / federation
  +-- CI/CD deployment or change request
  +-- Cloud IAM / role assumption
  +-- Cloud storage access or denial
  +-- Network / endpoint event
  +-- Job / compute / query failure
~~~

Normalize timestamps to UTC and preserve original event IDs. Account for delivery delay and clock skew. Avoid joining unrelated events solely because timestamps are close.

## 11. Compliance operations

Compliance requirements depend on contractual, regulatory and organizational obligations. SREs should implement the approved controls, not declare a platform compliant based only on log availability.

### Evidence categories

- Identity and privileged access reviews.
- Change approval and audit trail.
- Secret rotation and access revocation.
- Catalog/storage governance reviews.
- Audit retention and integrity.
- Incident response exercises.
- Security monitoring coverage and alert tests.
- Backup, restore and evidence retrieval procedures.

### Evidence quality requirements

Each evidence item needs an owner, UTC collection time, scope, source, integrity/provenance, access classification, retention rule and reviewer. Protect raw logs from unauthorized modification and deletion.

## 12. Incident scenario A — Sudden authentication failures

**Symptom:** A critical automation identity receives repeated authentication errors.

**Triage:** correlate Databricks audit events, identity provider events, credential expiry, federation trust and recent deployments.

**Recovery:** follow approved credential or trust remediation; do not broaden permissions.

**Validation:** intended workload authenticates and unauthorized operations remain denied.

## 13. Incident scenario B — Unexpected admin grant

**Symptom:** An administrative permission change occurs outside the change window.

**Triage:** identify actor, target, effective grant, approval, related changes and usage during the exposure period.

**Recovery:** security-approved containment and restoration of least privilege.

**Validation:** effective grants and audit trail match desired state.

## 14. Incident scenario C — Audit pipeline gap

**Symptom:** Expected audit events stop appearing.

**Triage:** distinguish low activity, source publication delay, ingestion failure, permission changes, SIEM processing and retention filters.

**Recovery:** repair the affected ingestion path and backfill supported historical records without silently overwriting evidence.

**Validation:** controlled synthetic event appears end-to-end and gap duration is documented.

## 15. Incident scenario D — Storage credential modification

**Symptom:** A storage credential or external location change coincides with data access failures.

**Triage:** compare Databricks governance audit, cloud IAM policy changes, network events and approved change records.

**Recovery:** restore approved credential/trust/path configuration using change control.

**Validation:** designated synthetic data operation succeeds; unrelated access remains denied.

## 16. Incident scenario E — Suspicious service principal activity

**Symptom:** A non-human identity performs unexpected actions or operates from an unusual context.

**Triage:** determine credential method, owner, workload, event sequence, recent pipeline changes and potential compromise.

**Recovery:** invoke security incident response; revoke/rotate credentials where indicated, contain access and preserve evidence.

**Validation:** legitimate automation restored with scoped permissions and suspicious access blocked.

## 17. Troubleshooting matrix

| Symptom | Possible cause | First validation |
|---|---|---|
| No audit rows | No events, no permission, disabled scope, delay | Table access and coverage |
| SIEM missing events | Pipeline failure, filter, schema change | Ingestion checkpoints |
| High ingestion lag | Backlog or publication delay | Source and sink timestamps |
| Duplicate events | Retry or replay | Event IDs and deduplication |
| Incorrect actor | Identity mapping ambiguity | Effective principal and source event |
| Cloud 403 without UC denial | Cloud IAM/storage boundary | Cloud access logs |
| Unexpected grant | Change or compromise | Approval and audit correlation |
| Retention gap | Misconfigured lifecycle | Retention policy and archive |

## 18. Alerting design

### Example SLOs (set real thresholds from policy and baselines)

- Audit source availability.
- Maximum acceptable event freshness.
- Percentage of critical event types mapped to a detection.
- Time to acknowledge critical security alerts.
- Time to retrieve required evidence.
- Time to revoke a compromised automation credential.

### Alert hygiene

- Use clear owners and escalation paths.
- Deduplicate repeat events.
- Suppress only documented maintenance patterns.
- Keep security-critical alerts independent of noisy infrastructure alerts.
- Test with synthetic events and retain test evidence.
- Protect alert payloads from exposing secrets or personal data.

## 19. Audit pipeline recovery runbook

1. **Detect:** identify missing source, service and first gap time.
2. **Scope:** affected account, workspace, region and event classes.
3. **Preserve:** checkpoints, logs, configurations and error messages.
4. **Classify:** source delay, permissions, schema drift, transport, sink or retention.
5. **Mitigate:** restore supported collection path with approved change.
6. **Backfill:** replay available historical records with idempotent deduplication.
7. **Validate:** confirm synthetic event, freshness and event counts.
8. **Document:** residual gap, compliance impact and owner.
9. **Prevent:** add pipeline health alerts and schema compatibility checks.

## 20. Change and rollback controls

Before changing audit ingestion, retention, SIEM mappings or privileges:
- Capture current configuration and event volume baseline.
- Review data classification and legal retention requirements.
- Test the new pipeline or detection against synthetic events.
- Maintain parallel validation where practical.
- Define rollback without deleting or mutating existing evidence.
- Confirm recovery of ingestion freshness and event integrity.

## 21. Operational acceptance checklist

- [ ] Audit sources inventoried across Databricks, IdP and cloud.
- [ ] Audit table access and scope verified.
- [ ] Event freshness baseline established.
- [ ] Privileged activity review demonstrated.
- [ ] Synthetic detection tests completed.
- [ ] Cloud audit correlation documented.
- [ ] Retention, encryption and evidence access controls reviewed.
- [ ] Five incident scenarios completed.
- [ ] Audit gap detection and backfill runbook tested.
- [ ] Detection owners and escalation paths defined.
- [ ] No secrets or sensitive raw logs published.
- [ ] No production audit configuration changed during lab.

## 22. Review questions

1. Why is a Databricks audit table alone insufficient for cloud storage access investigations?
2. What can cause the newest audit event to appear old even if ingestion is healthy?
3. How would you distinguish an expected administrative grant from privilege escalation?
4. What safeguards are needed when backfilling missing audit records?
5. Why must compliance evidence include provenance and retention information?

## 23. References

- Databricks audit logs: https://docs.databricks.com/aws/en/admin/account-settings/audit-logs
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks audit system table: https://docs.databricks.com/aws/en/admin/system-tables/audit-logs
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- AWS CloudTrail: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/
- Google Cloud Audit Logs: https://cloud.google.com/logging/docs/audit

**Next:** Chapter 020 — Security, Identity and Governance Operations Capstone.
