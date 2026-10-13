# Chapter 018 — Unity Catalog Security and Data Access Governance

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 04 — Security, Identity and Governance  
**Audience:** SRE, Platform Engineering, Security, Data Governance  
**Clouds:** AWS, Azure, Google Cloud  
**Lab mode:** Read-only governance assessment and approved staging tests  
**Prerequisites:** Chapters 016–017

## 1. Mission

Operate Unity Catalog as a security and reliability boundary, rather than as an application development feature. The objective is to understand **which principal can access which securable, through which storage identity and network path**, and to recover safely when a legitimate workload is denied.

### Learning outcomes

1. Map metastore, catalog, schema and object governance boundaries.
2. Distinguish account/workspace identity, Unity Catalog grants, cloud IAM and network authorization.
3. Inspect ownership, grants, storage credentials and external locations.
4. Diagnose access denied, credential failure and external storage connectivity issues.
5. Detect excessive privileges and governance drift.
6. Establish evidence, approval, rollback and incident response controls.

## 2. Reference architecture

~~~text
          Account identities and groups
                     |
             Workspace assignment
                     |
                Unity Catalog
                 Metastore
                     |
               Catalog / Schema
                     |
         Tables / Views / Volumes / Functions
                     |
           UC privileges and ownership
                     |
       Storage credential / external location
                     |
          Cloud IAM identity / permissions
                     |
       VPC / VNet / VPC, DNS and endpoints
                     |
             Cloud object storage
~~~

The exact hierarchy and access requirements vary by securable and access mode. A Databricks workspace administrator does not automatically have unrestricted privileges over every Unity Catalog object.

## 3. Core securables and operational roles

| Securable | SRE responsibility |
|---|---|
| Metastore | Assignment, ownership, regional boundaries and governance availability |
| Catalog | Environment/domain isolation, ownership and access reviews |
| Schema | Namespace-level grants and controlled object creation |
| Table/view | Access verification, audit and incident scoping |
| Volume | File-access governance and workload troubleshooting |
| Storage credential | Cloud identity linkage and blast-radius management |
| External location | Governed storage path, overlap and access controls |
| Connection / other supported securables | Ownership and permissions review |

Catalog and schema privileges do not necessarily imply object privileges. Check inherited permissions and effective grants using current Unity Catalog semantics.

## 4. Identity and authorization chain

For each failed request, test the layers independently:

1. **Authentication:** Can the user/service principal authenticate?
2. **Workspace:** Is the principal assigned to the workspace and permitted to use the compute?
3. **Unity Catalog:** Does the principal have required privileges on the target securable and ancestors?
4. **Storage identity:** Is the configured cloud IAM identity valid and authorized?
5. **Network:** Can the execution environment reach the required storage endpoint?
6. **Workload:** Is the object/path valid and the operation supported?

**Do not fix a storage 403 by blindly granting catalog admin.** Conversely, a successful network probe does not establish Unity Catalog permission.

## 5. Multi-cloud storage integration

| Cloud | Common storage | Cloud identity boundary | Operational checks |
|---|---|---|---|
| AWS | S3 | IAM role/trust and bucket policy | Role trust, bucket policy, endpoint and region |
| Azure | ADLS Gen2 | Managed identity / service principal and Azure RBAC/ACL | Credential, storage firewall, private endpoint and DNS |
| GCP | Cloud Storage | Service account and IAM | Service account, bucket IAM and network controls |

Use the current vendor-supported Unity Catalog storage credential mechanism for the chosen cloud. Do not embed cloud access keys in notebooks or policy files.

## 6. Lab A — Read-only workspace preflight

On Windows PowerShell:

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Replace the example profile with the approved workspace. Confirm the correct host and effective identity. Never print or export tokens.

### Inspect CLI capabilities

~~~powershell
databricks metastores list --help
databricks catalogs list --help
databricks schemas list --help
databricks storage-credentials list --help
databricks external-locations list --help
databricks grants get --help
~~~

Account privileges and workspace assignment affect which results are visible.

## 7. Lab B — Governance inventory

For an authorized workspace:

~~~powershell
databricks metastores list --output json
databricks catalogs list --output json
databricks storage-credentials list --output json
databricks external-locations list --output json
~~~

Review pagination and permissions before treating output as complete. The metadata may contain sensitive storage paths and cloud identity references; store raw output only in restricted evidence storage.

### Inventory worksheet

| Field | Evidence |
|---|---|
| Workspace / region | Verified configuration |
| Assigned metastore | Metastore assignment and region |
| Catalogs | Count, owner and environment classification |
| Schemas | Owner and grant review |
| External locations | Approved path inventory (restricted) |
| Storage credentials | Owner, cloud identity, purpose |
| Audit coverage | Supported audit/system tables |
| Review time | UTC |
| Visibility limitations | Documented |

## 8. Lab C — SQL-based privilege review

Run only against an approved catalog/schema with a principal authorized to inspect grants.

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_schema;
~~~

Replace identifiers with actual approved names. For a specific table, use the corresponding supported SHOW GRANTS syntax. Some privileges may be inherited; review the effective authorization path, not just a single object's explicit grants.

### Permission review questions

- Who owns the catalog/schema?
- Which groups can use the catalog and schema?
- Who can create, modify or manage objects?
- Are broad groups granted sensitive permissions?
- Do service principals have only required privileges?
- Is ownership transfer controlled and audited?

## 9. Lab D — External location and storage credential review

For each authorized external location:

1. Confirm path, purpose, owner and environment.
2. Identify referenced storage credential.
3. Confirm the cloud identity and trust configuration.
4. Review Unity Catalog permissions to use or manage the location.
5. Validate network/private endpoint requirements for the compute mode.
6. Confirm auditability and change history.
7. Check for unintended overlapping paths or broad grants.

Do not run destructive path operations, rotate cloud identities or modify external locations during the read-only lab.

## 10. Lab E — Staging access acceptance

Use a designated synthetic dataset and two approved staging principals.

| Test | Expected result |
|---|---|
| Authorized principal reads test object | Success |
| Principal without catalog/schema use | Denied |
| Principal without object privilege | Denied |
| Approved compute mode accesses storage | Success |
| Unapproved storage path access | Denied |
| Audit event or query history | Evidence available |

Negative tests must be non-destructive and restricted to authorized synthetic resources. Record exact effective identity, UTC time, compute mode and error classification.

## 11. Governance drift detection

Compare:
- **Desired state:** reviewed ownership, grants, credentials and external location definitions.
- **Actual state:** current API/SQL inventory and permissions.
- **Observed use:** audit logs, job principals and access patterns.

### Drift report

| Object class | Drift example | Risk | Remediation |
|---|---|---|---|
| Catalog | Unexpected MANAGE grant | Privilege escalation | Review and revoke via change |
| Schema | Missing USE SCHEMA | Workload failure | Restore narrow grant |
| External location | Unapproved storage path | Data exposure | Investigate and contain |
| Storage credential | Cloud role changed | Access failure or excess access | Validate trust and IAM |
| Ownership | Owner no longer active | Recovery risk | Approved ownership transfer |

Treat any unexpected grant change as a security finding until reviewed.

## 12. Incident scenario A — USE CATALOG denied

**Symptom:** Job fails with a catalog access error after identity or grant changes.

**Triage:**
1. Confirm job service principal and workspace.
2. Identify catalog, schema and object.
3. Review effective grants on the relevant ancestors.
4. Correlate recent permission changes and audit events.
5. Verify compute access separately.

**Recovery:** restore the minimum approved USE CATALOG / USE SCHEMA / object privilege combination as required.

**Validation:** synthetic query succeeds and unauthorized principals remain denied.

## 13. Incident scenario B — Storage 403 despite UC grants

**Symptom:** Unity Catalog permissions appear correct, but data access fails at cloud storage.

**Triage:** identify storage credential, cloud identity, role trust, bucket/container policy, storage ACL and network endpoint. Correlate cloud logs.

**Recovery:** repair the specific IAM/storage/network boundary. Do not widen Unity Catalog privileges unnecessarily.

**Validation:** approved synthetic read/write on designated test path only.

## 14. Incident scenario C — External location misconfiguration

**Symptom:** A new external table or volume cannot access its storage path.

**Triage:** external location URL, credential mapping, path overlap, privileges, storage identity, DNS and compute-mode network path.

**Recovery:** correct reviewed location/credential configuration in staging first.

**Validation:** designated synthetic resource succeeds without opening unrelated paths.

## 15. Incident scenario D — Unexpected privilege escalation

**Symptom:** Broad group gains MANAGE or write privileges on sensitive data.

**Response:**
1. Escalate to security and preserve audit evidence.
2. Determine scope, actor, time and affected objects.
3. Contain through approved grant changes.
4. Assess data access during the exposure window.
5. Restore least privilege and verify intended workloads.
6. Document incident and preventive controls.

Do not erase audit logs or perform undocumented emergency grant changes.

## 16. Incident scenario E — Metastore or workspace assignment issue

**Symptom:** A workspace cannot see expected catalogs after provisioning or regional change.

**Triage:** verify workspace, region, metastore assignment, workspace-catalog bindings where configured, principal visibility and relevant service status.

**Recovery:** follow account administration change control; avoid ad hoc metastore reassignment in production.

**Validation:** required catalogs visible and authorized workloads healthy.

## 17. Troubleshooting decision tree

~~~text
Data access failure
       |
Authenticated and workspace assigned?
  no -> identity / workspace
  yes
       |
Compute permitted and compatible?
  no -> compute ACL / access mode
  yes
       |
UC grants on ancestors and object?
  no -> catalog/schema/object grants
  yes
       |
Storage credential / cloud IAM valid?
  no -> cloud trust / IAM / ACL
  yes
       |
Network and DNS path healthy?
  no -> endpoints / firewall / DNS
  yes -> object metadata / workload issue
~~~

A 403 alone cannot distinguish these boundaries; use actual error context and correlated logs.

## 18. Audit and monitoring

Monitor:
- Catalog/schema ownership and grant changes.
- Storage credential and external location changes.
- Failed data access for critical service principals.
- Metastore assignment and workspace binding changes.
- Broad grants or dormant owners.
- Cloud IAM trust and storage policy changes.
- Audit ingestion delay or gaps.

Where system tables are enabled, validate their schemas, retention and access controls before creating alerts. Do not assume every event type is available in every workspace.

## 19. Production governance change runbook

1. Record requester, object, reason, principal and requested privileges.
2. Verify owner approval and data classification.
3. Capture existing grants, ownership and dependencies.
4. Test the minimum change in staging.
5. Define rollback and potential impact on active jobs.
6. Apply through approved change control.
7. Validate both allowed and denied operations.
8. Verify audit events and update desired-state records.
9. Remove temporary access at expiration.

Rollback must restore the prior authorized state without unintentionally granting broad access.

## 20. Acceptance checklist

- [ ] Metastore/workspace relationship documented.
- [ ] Catalog/schema/object ownership inventoried.
- [ ] Effective grants and inheritance reviewed.
- [ ] Storage credentials and external locations assessed.
- [ ] Cloud IAM and network boundaries distinguished.
- [ ] Staging positive/negative access tests completed.
- [ ] Privilege drift controls reviewed.
- [ ] Five incident scenarios completed.
- [ ] Audit alerts and ownership established.
- [ ] Change and rollback runbook approved.
- [ ] No production grants or credentials modified in lab.
- [ ] No sensitive storage paths or raw exports published.

## 21. Review questions

1. Why can a user authenticate successfully but fail USE CATALOG?
2. How do Unity Catalog grants differ from cloud object storage IAM?
3. Why must external locations and storage credentials be reviewed together?
4. How can workspace/metastore assignment affect catalog visibility?
5. What evidence distinguishes a network failure from a storage permission denial?

## 22. References

- Unity Catalog overview: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Unity Catalog privileges: https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/
- Unity Catalog external locations: https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Azure Databricks Unity Catalog: https://learn.microsoft.com/azure/databricks/data-governance/unity-catalog/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 019 — Audit Logging, Security Monitoring and Compliance Operations.
