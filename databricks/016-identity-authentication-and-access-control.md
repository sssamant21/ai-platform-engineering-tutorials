# Chapter 016 — Identity, Authentication and Access Control

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 04 — Security, Identity and Governance  
**Audience:** SRE, IAM, Security Engineering, Databricks Administrators  
**Clouds:** AWS, Azure, GCP  
**Lab mode:** Read-only auditing and approved staging identity tests  
**Prerequisites:** Chapters 001–015

## 1. Mission

Establish secure and recoverable Databricks identity operations. This chapter focuses on **who authenticates**, **where authorization is evaluated**, **which permissions are granted**, and **how an SRE safely diagnoses or restores access**.

By the end you should be able to:
- Distinguish account-level and workspace-level identities and administration.
- Explain user, group and service principal identity lifecycles.
- Identify authentication methods and credential exposure risks.
- Audit workspace membership, group assignment and resource permissions.
- Distinguish workspace access, compute permissions, cloud IAM and Unity Catalog authorization.
- Diagnose permission failures with evidence and least-privilege remediation.
- Run safe break-glass and credential compromise tabletop exercises.

## 2. Identity and authorization architecture

~~~text
Enterprise IdP / SSO / SCIM provisioning
                  |
             Databricks account
          Users / Groups / SPs
                  |
         Workspace assignment
                  |
       Workspace authentication
                  |
    +-------------+--------------+
    |             |              |
Workspace ACLs  Compute ACLs  SQL / job permissions
    |             |              |
    +-------------+--------------+
                  |
        Unity Catalog permissions
                  |
        Cloud IAM / storage access
                  |
          Network and data controls
~~~

**Key principle:** authentication success does not imply authorization to a workspace, cluster, job, SQL warehouse, catalog or cloud storage. Each boundary must be checked separately.

## 3. Identity types

| Identity | Typical purpose | Operational risk |
|---|---|---|
| Human user | Interactive administration and analysis | Stale access, MFA/SSO issues |
| Group | Role-based access and ownership | Nested or stale membership, overbroad grants |
| Databricks service principal | Automation and workload identity | Long-lived secrets, excessive privilege |
| Cloud service principal / managed identity / IAM role | Cloud resource access | Trust policy and permission mismatch |
| Break-glass administrator | Emergency recovery | Abuse risk and poor auditability |

A Databricks service principal is not automatically identical to an Azure service principal or AWS IAM role. Map the exact identity and trust chain for each deployment.

## 4. Authentication methods and operational guidance

Use supported OAuth flows and workload identity federation where appropriate. Avoid long-lived personal access tokens for new automation when a supported short-lived alternative exists.

### Security controls

- Centralized SSO and MFA for human users.
- Automated lifecycle management with approved IdP/SCIM design.
- Separate human and automation identities.
- Scoped service principal privileges.
- Secrets stored in an approved secrets manager, not scripts or Git.
- Token expiry and credential rotation procedures.
- Audit logging, anomaly detection and emergency revocation.
- Explicit workspace assignment and group ownership.

Authentication capabilities and supported federation configurations vary by cloud, account and CLI version. Validate against the current documentation.

## 5. Authorization boundaries

| Boundary | Typical decision | SRE evidence |
|---|---|---|
| Account | Account role or administrative scope | Account-level membership and grants |
| Workspace | Workspace assignment and admin status | Workspace access records |
| Compute | CAN USE, CAN MANAGE and related permissions | Compute ACL and policy |
| Jobs | View/manage/run permissions | Job permissions and owner |
| SQL warehouse | Use/manage permissions | Warehouse permissions |
| Unity Catalog | Catalog/schema/table privileges | Grants and effective principal |
| Cloud storage | IAM role, managed identity, service account | Cloud policy and storage logs |
| Network | Endpoint, firewall and routing | Network telemetry |

Do not respond to a data-access error by automatically granting workspace admin.

## 6. Lab A — Validate CLI authentication

Use an approved Windows PowerShell session:

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Replace the example profile with the approved workspace. Verify the host and effective identity. Avoid saving authentication output to a public repository.

### Troubleshooting authentication

- Wrong workspace host or profile.
- Expired SSO/OAuth session.
- Identity not assigned to workspace.
- Token expired or revoked.
- Service principal federation or secret configuration invalid.
- Local clock skew or proxy/TLS interception.
- Network/DNS access failure before authentication.

A 401 usually indicates authentication failure; a 403 often indicates insufficient authorization, although actual API behavior and proxy layers must be checked.

## 7. Lab B — Read-only identity inventory

Check supported CLI commands and permission scope:

~~~powershell
databricks users list --help
databricks groups list --help
databricks service-principals list --help
databricks workspace-assignment list --help
~~~

Depending on CLI version, workspace assignments may be available through account-level commands with different syntax. Do not assume all account APIs are available through a workspace profile.

### Workspace identity listing

~~~powershell
databricks users list --output json
databricks groups list --output json
databricks service-principals list --output json
~~~

These commands require appropriate privileges and may be paginated or permission filtered. Collect sanitized counts and group ownership, not personal data, in the public lab report.

### Evidence worksheet

| Field | Value |
|---|---|
| Workspace | Authorized staging label |
| Cloud/region | Verified |
| Account identity source | Approved IdP |
| Provisioning model | SCIM / other documented model |
| Human administrators | Count and approval reference |
| Service principals | Count and owner coverage |
| Orphaned groups | Findings and owner |
| Review UTC | Timestamp |
| Visibility limitations | Explicit |

## 8. Lab C — Permissions inspection

For a selected approved cluster, job or SQL warehouse:

~~~powershell
databricks permissions get --help
databricks permissions get-permission-levels --help
~~~

The exact object type and object ID are required. Use installed CLI help and a known authorized resource. Confirm:
1. Owner and administrative principals.
2. Effective group-based permissions.
3. Service principal permissions.
4. Whether privilege is necessary for operational duties.
5. Whether the resource is covered by a compute policy.
6. Whether permission changes are audited.

**Never** infer effective data privileges from a cluster ACL alone.

## 9. Lab D — Service principal authentication acceptance

Perform only in staging with a dedicated, approved service principal.

1. Confirm service principal owner, purpose and workspace assignment.
2. Verify the approved OAuth/federation configuration.
3. Test an authorized read-only API operation.
4. Verify an intentionally unauthorized operation is denied using a safe nonmutating endpoint or permissions review.
5. Confirm credentials are not printed, logged or committed.
6. Verify audit events and credential expiration monitoring.
7. Record rotation and emergency revocation contacts.

Do not embed client secrets, tokens or private keys in examples or output artifacts.

## 10. Lab E — Least-privilege access matrix

| Role | Workspace | Compute | Jobs | Catalog/data | Account admin |
|---|---|---|---|---|---|
| Platform SRE | Approved | Inspect/operate as needed | Troubleshoot | Metadata only if needed | Usually no |
| Job service principal | Assigned | Approved policy | Run/manage assigned jobs | Narrow grants | No |
| Data analyst | Assigned | Approved SQL access | Limited | Approved read | No |
| Security auditor | Approved | Read-only evidence | Read-only evidence | Audit scope | Only if explicitly required |
| Break-glass admin | Emergency | Scoped recovery | Scoped recovery | Emergency only | Time-bound and audited |

This is a **design example**, not a universal permission mapping. Implement the minimum supported grants needed for each role.

## 11. Failure scenario A — User cannot access workspace

**Symptoms:** SSO succeeds, but workspace access is denied.

**Triage:**
1. Verify correct account, host and user identity.
2. Check IdP/SCIM provisioning and workspace assignment.
3. Review group membership and entitlements.
4. Correlate audit events and recent access changes.
5. Check whether the error is network/TLS rather than IAM.

**Recovery:** restore the intended assignment through approved IAM workflow.

**Validation:** user can sign in with expected role and cannot access unauthorized resources.

## 12. Failure scenario B — Automation receives 401/403

**Symptoms:** Scheduled automation fails after credential or policy change.

**Triage:**
- Identify service principal and authentication mechanism.
- Confirm token/secret expiration, federation trust and audience.
- Verify workspace assignment and resource ACL.
- Separate Unity Catalog/data authorization from API access.
- Check audit events for denied action and effective principal.

**Recovery:** repair approved credential/federation configuration or narrowly scoped grants.

**Validation:** authorized operation succeeds; unrelated permissions remain denied.

## 13. Failure scenario C — Excessive privilege detected

**Symptoms:** Service principal has workspace admin or broad manage permissions without justification.

**Triage:** review ownership, dependent jobs, group grants and last use.

**Recovery:** design reduced privileges, test in staging, remove broad grants through change control.

**Validation:** critical automation continues; unauthorized administrative operations are denied.

## 14. Failure scenario D — Credential exposure

**Symptoms:** A token or client secret appears in logs or a repository.

**Immediate response:**
1. Activate security incident response and preserve restricted evidence.
2. Revoke/rotate the exposed credential using approved procedures.
3. Assess audit logs for suspicious use and affected resources.
4. Replace credential references in approved secret stores.
5. Remove exposed material from accessible locations and address repository history according to security policy.
6. Validate automation with replacement credentials.
7. Complete incident review and prevention actions.

Do not paste the leaked credential into tickets, chat or investigation summaries.

## 15. Failure scenario E — Emergency admin lockout

**Symptoms:** No standard administrator can manage critical workspace access.

**Triage:** determine whether account-level access remains, check identity provider outage, assignment drift and administrative role changes.

**Recovery:** invoke documented, time-bound, dual-approved break-glass process. Use the vendor-supported recovery path and audit every action.

**Validation:** standard administrative access restored, emergency privileges revoked, audit evidence reviewed.

Never create undocumented permanent super-admin identities as a workaround.

## 16. Troubleshooting matrix

| Symptom | Likely boundary | First evidence |
|---|---|---|
| Cannot resolve workspace host | DNS/network | Resolver and TLS test |
| API 401 | Authentication | Auth profile and identity |
| API 403 | Authorization | Resource ACL and effective principal |
| User missing | Provisioning | IdP/SCIM sync and account identity |
| Job denied | Job/compute ACL | Job owner and permissions |
| Data query denied | Unity Catalog | Catalog grants and identity |
| Storage 403 | Cloud IAM/storage | Storage access logs |
| Automation stops after rotation | Secret/federation | Credential lifecycle |
| Admin lockout | Account/workspace governance | Break-glass runbook |

## 17. Monitoring and alert acceptance

Monitor:
- Authentication failures and abnormal login patterns.
- Service principal credential expiration and failed federation.
- SCIM provisioning failures and stale accounts.
- Workspace administrator and group membership changes.
- Permission grants, revocations and policy drift.
- Repeated 403 failures on critical automation.
- Emergency access activation and expiry.
- Audit pipeline health and retention.

Tune alerts to actual threat models and workload baselines; do not expose sensitive identity details in public dashboards.

## 18. Production access change runbook

1. **Request:** record identity, resource, reason, owner and duration.
2. **Verify:** authenticate requester and obtain appropriate approval.
3. **Review:** existing permissions and least-privilege alternative.
4. **Stage:** test intended grant with nonproduction identity/resource.
5. **Apply:** make the smallest approved change.
6. **Validate:** authorized operation succeeds; forbidden actions remain blocked.
7. **Audit:** record actor, time, approval and resulting effective access.
8. **Rollback:** restore previous grants if unexpected exposure or breakage occurs.
9. **Review:** remove temporary access by expiration date.

## 19. Security acceptance checklist

- [ ] Account and workspace identity boundaries documented.
- [ ] SSO/MFA and provisioning model reviewed.
- [ ] Service principals have owners and approved authentication.
- [ ] Workspace and compute permissions inspected.
- [ ] Unity Catalog and cloud IAM boundaries distinguished.
- [ ] Least-privilege matrix reviewed.
- [ ] Credential rotation/revocation procedure documented.
- [ ] Five incident scenarios completed.
- [ ] Break-glass procedure tested as tabletop.
- [ ] Audit and alert ownership verified.
- [ ] No production permissions changed in lab.
- [ ] No credentials or personal identity records committed.

## 20. Review questions

1. Why can successful SSO still result in workspace access denied?
2. How does a Databricks service principal differ from a cloud IAM identity?
3. Why does workspace admin not automatically solve cloud storage access?
4. What evidence distinguishes 401 from 403 failures?
5. What makes emergency administrative access safe and auditable?

## 21. References

- Databricks account and identity administration: https://docs.databricks.com/aws/en/admin/users-groups/
- Databricks authentication: https://docs.databricks.com/aws/en/dev-tools/auth/
- Databricks permissions: https://docs.databricks.com/aws/en/security/auth/access-control/
- Unity Catalog privileges: https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/
- Azure Databricks identity: https://learn.microsoft.com/azure/databricks/admin/users-groups/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 017 — Service Principals, Secrets and Credential Lifecycle.
