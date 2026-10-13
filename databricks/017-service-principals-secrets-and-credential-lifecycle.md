# Chapter 017 — Service Principals, Secrets and Credential Lifecycle

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 04 — Security, Identity and Governance  
**Audience:** SRE, Platform Engineering, IAM, Security Operations  
**Clouds:** AWS, Azure, Google Cloud  
**Lab mode:** Read-only inspection, offline exercises, approved staging tests  
**Prerequisite:** Chapter 016

## 1. Mission

Operate Databricks non-human identities securely through their full lifecycle: provisioning, assignment, authentication, authorization, monitoring, rotation, incident response and decommissioning.

This chapter focuses on **platform operations**, not application development.

### Learning objectives

1. Distinguish Databricks service principals from cloud IAM identities.
2. Select appropriate OAuth or workload identity federation patterns.
3. Inventory identity ownership and workspace assignments.
4. Review secret scopes and external secret manager dependencies without revealing values.
5. Implement a tested credential rotation and rollback procedure.
6. Detect and recover from authentication, authorization and credential compromise incidents.
7. Define auditable SRE acceptance criteria.

## 2. Reference architecture

~~~text
  GitLab CI/CD / Scheduled automation / Platform tools
                         |
             Identity provider or CI OIDC
                         |
          +--------------+--------------+
          |                             |
  Workload identity federation    OAuth client credentials
  (where supported)               (secret-backed)
          |                             |
          +--------------+--------------+
                         |
               Databricks OAuth
                         |
             Databricks service principal
                         |
             Account/workspace assignment
                         |
             Workspace resource permissions
                         |
                 Unity Catalog grants
                         |
          Cloud IAM / storage / network checks
                         |
             Audit logs and SRE alerts
~~~

**Important:** Databricks identity, cloud IAM role/service account/managed identity, and external identity provider principals are distinct security objects. Explicitly document the trust relationship and effective principal for each operation.

## 3. Identity and credential comparison

| Mechanism | Typical use | Operational concern |
|---|---|---|
| Databricks OAuth machine-to-machine | Service principal automation | Client secret lifecycle and scope |
| Workload identity federation | Supported CI/cloud workloads | Issuer, audience, subject and trust policy drift |
| OAuth user-to-machine | Interactive user tools | User session, MFA and consent |
| Personal access token | Legacy or constrained use cases | Long-lived bearer token exposure |
| Cloud IAM role/managed identity/service account | Cloud storage and resource access | Separate cloud authorization boundary |

Prefer short-lived, centrally governed authentication where supported. Do not create long-lived PATs merely for convenience.

## 4. Ownership and lifecycle controls

Every automation identity should have:
- A named team owner and backup owner.
- Documented purpose and allowed environments.
- Account and workspace assignment.
- Authentication method and credential store.
- Minimum resource and data permissions.
- Credential expiry/rotation plan.
- Audit log coverage and alert routing.
- Emergency disable/revocation runbook.
- Decommissioning criteria.

### Inventory template

| Field | Example (fictional) |
|---|---|
| Identity label | demo-platform-ci |
| Type | Databricks service principal |
| Environment | staging |
| Owner | platform-sre |
| Authentication | Workload federation |
| Workspace assignment | Approved staging |
| Granted scope | Job management on designated resources |
| Cloud identity mapping | Reviewed separately |
| Credential expiry | Not applicable for federation secret / trust review due |
| Last access review | UTC date |
| Emergency contact | IAM on-call |

Never store client secrets, refresh tokens, access tokens, private keys or complete authentication configuration in a public repository.

## 5. Cloud differences

### AWS

Common integration points include IAM roles, instance profiles, cloud trust policies, secrets managers and CI identity federation. Databricks service principal permissions are **not** AWS IAM permissions. Validate role assumption and storage access separately.

### Azure

Microsoft Entra ID identities, managed identities, Azure Key Vault and Azure RBAC may participate in the design. Distinguish an Entra service principal from a Databricks service principal, and confirm which credential flow the specific Databricks API supports.

### GCP

Google Cloud service accounts, Workload Identity Federation, Secret Manager and IAM may participate in cloud access. A GCP service account does not automatically acquire Databricks workspace permissions.

**Acceptance:** a diagram showing issuer, trust, token exchange, workspace assignment, resource grants and cloud data-access path for the actual cloud.

## 6. Lab A — PowerShell authentication preflight

Use an authorized Windows PowerShell session:

~~~powershell
databricks version
databricks auth profiles

$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Replace the example profile with the approved workspace profile. Verify the workspace host and effective identity. Do not capture or publish raw authentication output without redaction.

### Verify available identity commands

~~~powershell
databricks service-principals list --help
databricks service-principals get --help
databricks groups list --help
databricks permissions get --help
~~~

CLI command availability and account-level versus workspace-level APIs may differ by installed version and privilege scope.

## 7. Lab B — Read-only service principal inventory

~~~powershell
databricks service-principals list --output json
~~~

Review pagination and access scope before treating the output as complete.

For an approved service principal, collect only sanitized metadata:
1. Identity display label.
2. Owning team and operational purpose.
3. Workspace assignment.
4. Group membership and resource permissions.
5. Credential method and rotation owner.
6. Recent authentication failure signals.
7. Decommissioning status.

Do not dump all identity attributes into tickets or public documentation.

## 8. Lab C — Secrets inventory without disclosure

Databricks secret scopes can be used for supported workspace secrets; external secret managers may be the authoritative store in other architectures.

Inspect only scope and metadata:

~~~powershell
databricks secrets list-scopes --output json
databricks secrets list-secrets --help
databricks secrets list-acls --help
~~~

For a **known, approved scope**, use the installed CLI help to list secret metadata and ACLs. **Do not call secret retrieval commands**, print secret values or expose environment variables.

### Review checklist

- Scope owner and purpose.
- ACL grants and broad access.
- External secret store integration where applicable.
- Rotation and expiry process.
- Access logging and incident escalation.
- Secret naming and environment separation.

Secret scope permissions do not replace downstream authorization checks.

## 9. Lab D — Workload identity federation design review

For a staging CI or workload identity:

1. Identify trusted issuer and workload identity.
2. Document expected audience and subject claims.
3. Verify the federation policy limits trust to the intended repository, branch, environment or workload.
4. Confirm Databricks service principal mapping and workspace assignment.
5. Test a short-lived authentication flow with an approved read-only operation.
6. Confirm an unauthorized workload cannot obtain the same access.
7. Review audit evidence and incident disable procedure.

**Security warning:** Never use a wildcard trust policy for arbitrary repositories or branches without explicit risk acceptance.

## 10. Lab E — Credential rotation tabletop

For a secret-backed service principal, use the following change-controlled sequence.

### Phase 1 — Prepare

- Identify all consuming jobs, CI pipelines, tools and workspaces.
- Confirm secret store, owner, expiry and current credential status.
- Verify the platform's supported credential overlap and maximum active secrets.
- Capture approved rollback procedure and dependency map.
- Schedule staging verification and production change window.

### Phase 2 — Rotate

1. Create a new credential using the approved identity administration workflow.
2. Store it in the authorized secret manager.
3. Update a staging consumer to reference the new secret version.
4. Run a minimal authorized API operation.
5. Confirm logs do not expose the credential.
6. Promote the reference change to approved production consumers.
7. Verify critical automation and audit events.
8. Revoke the old credential once all consumers have migrated.

### Phase 3 — Validate

- All expected jobs authenticate successfully.
- No consumers still use the old credential.
- Old credential is revoked.
- Authentication failures return to baseline.
- Credential metadata and ownership records are updated.

**Rollback caveat:** After old-credential revocation, rollback cannot safely rely on that revoked credential. Restore service with a newly approved credential and corrected consumer configuration.

## 11. Lab F — Safe access and denial testing

Use an approved staging identity and resource.

| Test | Expected result |
|---|---|
| Approved read-only workspace API | Success |
| Approved job/resource metadata access | Success |
| Unassigned workspace | Denied |
| Unapproved resource operation | Denied |
| Revoked credential | Authentication fails |
| Cloud storage without separate grants | Denied |

Use nonmutating requests or reviewed permissions for negative tests. Do not attempt unauthorized access to real resources.

## 12. Incident scenario A — Expired OAuth client secret

**Symptom:** Scheduled jobs or CI automation begin failing authentication.

**Triage:**
1. Identify failing service principal and exact UTC start.
2. Confirm workspace host, authentication method and credential metadata.
3. Check expiration or revocation, not the secret value.
4. Correlate identity provider and Databricks audit logs.
5. Identify all affected consumers and recent secret deployments.

**Recovery:** rotate through the approved workflow, update consumers and verify scoped access.

**Validation:** successful authorized API call and resumed critical automation.

## 13. Incident scenario B — Federation trust mismatch

**Symptom:** CI workload fails to exchange an identity token after a repository or pipeline change.

**Triage:** issuer, audience, subject, branch/environment restrictions, clock skew, trust configuration and Databricks identity mapping.

**Recovery:** correct the narrowly scoped trust relationship. Never broaden trust to all subjects merely to restore service.

**Validation:** approved workload succeeds and a nonapproved identity remains denied.

## 14. Incident scenario C — API 403 after successful authentication

**Symptom:** Token acquisition succeeds, but a Databricks API or data operation fails.

**Triage:**
- Workspace assignment and group membership.
- Resource ACL and effective service principal.
- Job ownership or compute policy permissions.
- Unity Catalog privileges.
- Separate cloud storage IAM.

**Recovery:** restore only the minimum approved grant.

**Validation:** intended operation succeeds while unrelated resources remain inaccessible.

## 15. Incident scenario D — Credential leak

**Symptom:** A service principal secret appears in source control, logs or an artifact.

**Response:**
1. Declare a security incident and restrict evidence.
2. Revoke the exposed credential promptly under incident procedures.
3. Assess token use, audit events and blast radius.
4. Issue a replacement credential through the approved store.
5. Update consumers and validate workloads.
6. Remediate leaked artifacts and repository history under security policy.
7. Review logging, scanning and prevention controls.

Never paste the compromised secret into the incident ticket or chat.

## 16. Incident scenario E — Secret store unavailable

**Symptom:** New workloads cannot retrieve required credentials.

**Triage:** determine whether failure is secret store availability, network, IAM, DNS, client cache or token expiry.

**Recovery:** restore the approved secret distribution path. Avoid copying secrets into scripts or local files as an emergency workaround.

**Validation:** new workload starts, existing workloads remain healthy and access logs show expected behavior.

## 17. Troubleshooting decision tree

~~~text
Automation authentication failure
             |
       DNS/TLS reachable?
       no -> network/proxy/DNS
       yes
             |
       Token acquired?
       no -> secret expiry, issuer, audience,
             subject, federation, clock
       yes
             |
       Databricks API allowed?
       no -> workspace/resource permissions
       yes
             |
       Data/storage allowed?
       no -> Unity Catalog / cloud IAM
       yes -> investigate workload logic
~~~

HTTP 401 and 403 are useful clues but not absolute proof of one root cause; inspect the actual error response and logs.

## 18. Monitoring and alerting

Track:
- Authentication failure rate by automation identity.
- Credential expiration within the organization's renewal window.
- Federation trust changes.
- Service principal assignment and privilege changes.
- Secret ACL changes and abnormal access patterns.
- Revoked-credential attempts.
- Critical automation failure correlated with identity errors.
- Audit log ingestion health.

Every alert should have an owner, severity, escalation path and tested runbook.

## 19. Production runbook — Identity outage

1. **Detect:** identify failed jobs and affected services.
2. **Scope:** identity, workspace, cloud, authentication method and dependent resources.
3. **Preserve:** UTC timeline, sanitized error, audit events and recent changes.
4. **Classify:** network, authentication, authorization, cloud IAM or secret distribution.
5. **Mitigate:** approved scoped rotation, trust repair or grant restoration.
6. **Validate:** authorized and denied test cases, plus real workload recovery.
7. **Close:** update inventory, expiry controls, RCA and corrective actions.

## 20. Operational acceptance checklist

- [ ] Service principal inventory includes owner and purpose.
- [ ] Human and automation identities are separated.
- [ ] Workspace assignments and resource grants reviewed.
- [ ] Federation is preferred where supported and appropriate.
- [ ] Secret scopes and external secret stores audited without reading values.
- [ ] Credential rotation and revocation runbooks documented.
- [ ] Staging authentication and negative tests completed.
- [ ] Five incident scenarios reviewed.
- [ ] Credential expiry and auth failure alerts owned.
- [ ] Break-glass path is approved and auditable.
- [ ] No production credentials rotated as part of the lab.
- [ ] No secrets or raw identity exports committed to Git.

## 21. Review questions

1. Why is a Databricks service principal different from a cloud IAM identity?
2. What are the operational benefits and risks of workload identity federation?
3. Why can successful OAuth authentication still result in a 403?
4. What must be checked before revoking an old client secret?
5. Why should a compromised credential not be used for rollback?
6. How can a secret store outage affect only newly started workloads?

## 22. References

- Databricks service principals: https://docs.databricks.com/aws/en/admin/users-groups/manage-service-principals
- Databricks authentication: https://docs.databricks.com/aws/en/dev-tools/auth/
- Databricks OAuth: https://docs.databricks.com/aws/en/dev-tools/auth/oauth-m2m
- Databricks secrets: https://docs.databricks.com/aws/en/security/secrets/
- Databricks access control: https://docs.databricks.com/aws/en/security/auth/access-control/
- Azure Databricks administration: https://learn.microsoft.com/azure/databricks/admin/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 018 — Unity Catalog Security and Data Access Governance.
