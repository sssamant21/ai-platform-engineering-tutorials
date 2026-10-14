# Chapter 020 — Security, Identity and Governance Operations Capstone

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 04 — Security, Identity and Governance (final capstone)  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, Platform Engineering, IAM, Security Operations, Data Governance  
**Prerequisites:** Chapters 016–019  
**Execution mode:** Read-only production assessment; approved staging and tabletop tests only

## 1. Mission

Prove that the Databricks platform has controlled identities, least-privilege authorization, auditable data access, recoverable credential dependencies and actionable security monitoring. This capstone evaluates operational readiness rather than building application features.

**Required deliverables**
1. Identity, service principal and credential ownership inventory.
2. Workspace, compute, Unity Catalog and cloud access boundary map.
3. Read-only permission and governance review.
4. Audit coverage, freshness and retention assessment.
5. Six documented security and availability incident exercises.
6. Change/rollback runbook and signed acceptance scorecard.

## 2. Safety contract

- Confirm workspace, account, cloud, region and authorized assessment scope.
- Do not change production IAM, grants, secret scopes, storage credentials or audit settings.
- Never print, retrieve or store secrets or tokens in evidence files.
- Use only approved synthetic data and staging resources for negative access tests.
- Keep raw identity names, storage URLs, audit records and resource IDs in restricted storage.
- Use change tickets and dual approval for break-glass or credential revocation exercises.
- Verify CLI syntax and privileges in the installed version; visibility may be incomplete.

## 3. End-to-end security architecture

~~~text
                  Enterprise IdP / CI issuer
                            |
                  Users / groups / SPs
                            |
                 Databricks account identity
                            |
                   Workspace assignment
                            |
            +---------------+----------------+
            |               |                |
        Workspace ACL    Compute/Jobs      SQL access
            |               |                |
            +---------------+----------------+
                            |
                    Unity Catalog grants
                            |
                Storage credential / location
                            |
                  Cloud IAM and storage ACL
                            |
                Private network / DNS / TLS
                            |
                       Data access

        Audit: IdP + Databricks + cloud + CI/CD
                            |
                   SIEM / alerts / evidence
~~~

A successful sign-in does not prove resource authorization. Unity Catalog grants do not automatically repair cloud IAM or network failures.

## 4. Roles and approval matrix

| Activity | Operator | Approver | Evidence |
|---|---|---|---|
| Read-only inventory | SRE | Workspace owner | Assessment record |
| IAM/group change | IAM admin | Security/owner | Approved change |
| UC grants | Governance admin | Data owner | Grant diff |
| Cloud storage IAM | Cloud engineer | Cloud/security owner | Policy diff |
| Secret rotation | Credential owner | Change owner | Rotation validation |
| Emergency access | Authorized responder | Dual approval | Audit trail |
| Audit pipeline change | Security engineering | Compliance owner | Pipeline validation |

## 5. Lab A — Workspace identity preflight

On Windows PowerShell:

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Replace the profile with the approved workspace and verify effective identity and host. Never export the credential configuration to a public repository.

### Inventory commands

~~~powershell
databricks users list --help
databricks groups list --help
databricks service-principals list --help
databricks permissions get --help

databricks users list --output json
databricks groups list --output json
databricks service-principals list --output json
~~~

Identity listings can be paginated or restricted. Record coverage and limitations. Avoid sharing raw user lists outside approved evidence storage.

## 6. Lab B — Non-human identity and credential audit

For each approved service principal, capture:
- Owner and backup owner.
- Business purpose and environment.
- Workspace assignment and group membership.
- Authentication method: OAuth M2M, federation or approved legacy flow.
- Secret store and credential expiration policy.
- Resource permissions and Unity Catalog grants.
- Rotation and emergency revocation contacts.
- Recent authentication failures and audit visibility.

### Metadata-only secrets review

~~~powershell
databricks secrets list-scopes --output json
databricks secrets list-secrets --help
databricks secrets list-acls --help
~~~

Do not use secret retrieval commands. Document which external secret manager, if any, is authoritative.

## 7. Lab C — Workspace and compute access

~~~powershell
databricks clusters list --output json
databricks jobs list --output json
databricks warehouses list --output json
databricks permissions get --help
~~~

For a selected authorized resource, inspect the relevant permissions with installed CLI help and a known object ID. Check:
1. Ownership and group-based grants.
2. Service principal permissions.
3. Unnecessary administrative privileges.
4. Environment separation.
5. Audit events for recent changes.

Do not assume a workspace administrator automatically owns every data securable.

## 8. Lab D — Unity Catalog inventory

~~~powershell
databricks metastores list --output json
databricks catalogs list --output json
databricks storage-credentials list --output json
databricks external-locations list --output json
~~~

Metadata may reveal sensitive cloud storage paths and identities; restrict exports.

### Read-only SQL grant inspection

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_schema;
~~~

Replace identifiers with approved catalog/schema names. Verify inherited grants and actual effective principal.

## 9. Lab E — Cloud storage authorization mapping

| Cloud | Storage boundary | Validate |
|---|---|---|
| AWS | S3 bucket policy, IAM role/trust, network endpoint | Correct role, region, bucket policy and network |
| Azure | ADLS Gen2, managed identity/Entra principal, RBAC/ACL | Identity, filesystem ACL, firewall and DNS |
| GCP | Cloud Storage, service account/IAM, networking | Identity, bucket policy and connectivity |

**Read-only cloud context checks**

~~~powershell
# AWS (if applicable)
aws sts get-caller-identity

# Azure (if applicable)
az account show --query "{subscription:id,tenant:tenantId}" -o json

# GCP (if applicable)
gcloud config get-value project
~~~

Run only the relevant section with authorized credentials. These commands establish context; they do not prove data-path access.

## 10. Lab F — Audit coverage and freshness

Where enabled and authorized:

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

~~~sql
SELECT
  MAX(event_time) AS newest_event_time,
  CURRENT_TIMESTAMP() AS observation_time
FROM system.access.audit
WHERE event_time >= current_timestamp() - INTERVAL 2 DAYS;
~~~

Check table scope, permissions, retention, schema and ingestion delays. An absence of rows does not prove the absence of activity.

### Audit coverage checklist

- Databricks account and workspace events.
- Service principal and administrative changes.
- Unity Catalog permissions and storage credential changes.
- IdP sign-in, provisioning and federation events.
- Cloud IAM, object storage and network events.
- CI/CD identity and configuration changes.
- SIEM ingestion, retention, integrity and alerting.

## 11. Lab G — Safe staging authorization tests

Use two approved synthetic principals and a disposable staging object.

| Test | Expected |
|---|---|
| Assigned principal authenticates | Success |
| Assigned principal accesses permitted workspace resource | Success |
| Principal without catalog privileges | Denied |
| Principal without table privilege | Denied |
| Cloud identity without storage permission | Denied |
| Audit records relevant operation where supported | Evidence present |

Use non-destructive queries and access checks only. Negative tests must not target unauthorized production resources.

## 12. Scenario 1 — Automation authentication outage

**Inject/tabletop:** Critical scheduled job fails after a credential expiration or federation policy change.

**Triage:** effective identity, workspace host, credential metadata, issuer/audience/subject, secret store, recent changes and audit events.

**Recovery:** rotate through approved store or repair narrowly scoped federation trust.

**Rollback:** restore known-good consumer configuration using a valid credential; never reuse revoked credentials.

**Acceptance:** critical automation resumes and unauthorized identities remain denied.

## 13. Scenario 2 — Unexpected administrator privilege

**Inject/tabletop:** A service principal receives excessive administrative access.

**Triage:** grant source, group membership, actor, approval, effective permissions and usage during exposure.

**Recovery:** security-approved containment and least-privilege restoration.

**Rollback:** only restore a previous grant if it is still authorized; security containment overrides convenience.

**Acceptance:** broad access removed and dependent workloads continue.

## 14. Scenario 3 — Unity Catalog access denied

**Inject/staging:** Approved service principal receives a USE CATALOG or object privilege error.

**Triage:** workspace assignment, compute access, catalog/schema/object grants and inherited privileges.

**Recovery:** restore only required approved privileges.

**Acceptance:** authorized synthetic query succeeds and negative test remains denied.

## 15. Scenario 4 — Cloud storage 403

**Inject/tabletop:** Unity Catalog grants appear correct but storage denies access.

**Triage:** storage credential, cloud identity/trust, bucket/container IAM/ACL, endpoint, firewall and DNS.

**Recovery:** repair exact cloud authorization or connectivity boundary through change control.

**Acceptance:** synthetic storage access succeeds without broadening unrelated permissions.

## 16. Scenario 5 — Audit pipeline failure

**Inject/tabletop:** SIEM stops receiving an expected class of events.

**Triage:** source activity, system table availability, schema changes, transport, sink filters, permissions and retention.

**Recovery:** repair pipeline, backfill available history with deduplication, document unrecoverable gaps.

**Acceptance:** synthetic event appears end-to-end; alert and freshness checks pass.

## 17. Scenario 6 — Suspected secret compromise

**Inject/tabletop:** A service principal secret is exposed in a build artifact.

**Response:** activate security incident process, restrict evidence, revoke/rotate, inspect audit activity, update consumers, remediate exposure and review preventive controls.

**Acceptance:** old credential cannot authenticate, replacement workload functions and incident evidence is preserved.

## 18. Troubleshooting decision tree

~~~text
Security / data access failure
          |
    Network and TLS reachable?
    no -> DNS, proxy, private endpoints
    yes
          |
    Authentication successful?
    no -> OAuth, federation, secret expiry
    yes
          |
    Workspace/resource authorized?
    no -> assignment, ACL, compute policy
    yes
          |
    Unity Catalog grants valid?
    no -> catalog/schema/object privileges
    yes
          |
    Cloud IAM/storage access valid?
    no -> role trust, ACL, firewall, endpoint
    yes -> workload/data issue
~~~

HTTP 401/403 are clues, not complete root-cause determinations. Correlate the full error with audit and cloud telemetry.

## 19. Change and rollback runbook

1. **Request:** identify principal, resource, permission, duration and justification.
2. **Approve:** validate requester, owner, security/data classification.
3. **Baseline:** capture current grants, identity mapping and dependent workloads.
4. **Stage:** test authorized and denied paths on synthetic resources.
5. **Apply:** use least-privilege change with controlled blast radius.
6. **Observe:** verify logs, critical jobs and unexpected access.
7. **Rollback:** restore prior *authorized* state or replace compromised credential safely.
8. **Document:** record actor, UTC time, approvals, evidence and remaining risk.
9. **Expire:** remove temporary access and validate revocation.

## 20. Monitoring and operational controls

Minimum alerts:
- Critical service principal authentication failures.
- Credential expiration within policy-defined window.
- Unexpected administrative grants or role assignments.
- Storage credential/external location changes.
- Critical Unity Catalog access denials.
- Emergency access activation.
- Audit source missing events or unacceptable lag.
- Cloud IAM trust or storage policy drift.

Each alert must include ownership, escalation, runbook, severity and test evidence. Do not put tokens or personal data in alert payloads.

## 21. Evidence and sign-off package

~~~text
Assessment: Databricks Security, Identity and Governance
Assessment UTC:
Cloud / region / workspace:
Lead assessor:
Security reviewer:
Data governance owner:
Cloud IAM owner:
Change reference:

Identity inventory coverage:
Service principal ownership coverage:
Credential lifecycle controls:
Workspace/compute permission findings:
Unity Catalog grant findings:
Cloud storage access boundary:
Audit source coverage and freshness:
Detection tests:
Scenario 1 — auth outage:
Scenario 2 — privilege escalation:
Scenario 3 — UC denial:
Scenario 4 — cloud storage 403:
Scenario 5 — audit gap:
Scenario 6 — credential compromise:
Outstanding findings:
Remediation owners and due dates:
Decision: GO / CONDITIONAL GO / NO-GO
Approvals:
~~~

## 22. Acceptance scorecard

| Control | Status | Evidence/owner |
|---|---|---|
| Identity provisioning and workspace assignment | Not assessed | |
| Service principal inventory and ownership | Not assessed | |
| OAuth/federation and secret lifecycle | Not assessed | |
| Workspace and compute least privilege | Not assessed | |
| Unity Catalog effective grants | Not assessed | |
| Cloud IAM/storage authorization mapping | Not assessed | |
| Audit coverage and retention | Not assessed | |
| Alerting and escalation | Not assessed | |
| Six incident exercises | Not assessed | |
| Change/rollback and break-glass process | Not assessed | |
| Evidence privacy and integrity | Not assessed | |
| Open critical security findings | Not assessed | |

**GO:** critical controls verified, exercises passed, no unresolved critical findings.  
**CONDITIONAL GO:** only documented noncritical gaps with accepted risk, owner and due date.  
**NO-GO:** critical access exposure, unrecoverable identity path, missing required audit evidence or untested emergency recovery.

## 23. Cleanup

- Remove only disposable staging identities/resources created under the approved change.
- Revoke temporary staging credentials and permissions.
- Confirm negative access tests still deny unapproved principals.
- Archive sanitized assessment summary and restrict raw evidence.
- Verify no production security configuration was changed during the capstone.

## 24. Review questions

1. Why must Databricks, Unity Catalog and cloud IAM permissions be evaluated separately?
2. What is the safest response when an automation credential is exposed?
3. Why can audit data be incomplete despite successful SQL queries?
4. How does an access change rollback differ after credential compromise?
5. What evidence is needed for a defensible security readiness decision?

## 25. References

- Databricks identity administration: https://docs.databricks.com/aws/en/admin/users-groups/
- Databricks authentication: https://docs.databricks.com/aws/en/dev-tools/auth/
- Databricks access control: https://docs.databricks.com/aws/en/security/auth/access-control/
- Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Unity Catalog privileges: https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Azure Databricks security: https://learn.microsoft.com/azure/databricks/security/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Part 04 complete. Next:** Chapter 021 — Databricks Job Scheduling and Workflow Reliability.
