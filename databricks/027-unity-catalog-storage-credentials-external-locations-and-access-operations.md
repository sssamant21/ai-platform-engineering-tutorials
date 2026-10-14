# Chapter 027 — Unity Catalog Storage Credentials, External Locations and Access Operations

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 06 — Storage, Data Access and Infrastructure Reliability  
**Clouds:** AWS, Azure, GCP  
**Audience:** SRE, DBRE, Platform Engineering, Cloud Security  
**Prerequisites:** Chapters 016–018 and 026  
**Safety:** Production read-only. Changes and negative authorization tests require approved staging resources.

## 1. Mission and outcomes

Operate Unity Catalog storage credentials and external locations as critical infrastructure dependencies. Diagnose access failures across the Databricks permission layer, cloud identity and storage policy layer, and network layer without granting excessive access.

By completion, you should be able to inventory credential-to-location relationships, identify owners and blast radius, verify least privilege, design rotation and rollback, detect drift, and validate recovery with synthetic data.

## 2. Architecture and trust boundaries

    Databricks workspace and run-as principal
                       |
               Unity Catalog grants
                       |
                External location
                 (URL + credential)
                       |
               Storage credential
                       |
           Cloud identity / trust policy
                       |
        Bucket / container IAM, ACL, policy
                       |
            DNS / private routing / TLS
                       |
                S3 / ADLS / GCS

**Key rule:** Catalog authorization, cloud authorization, and network connectivity are separate controls. Fix the failed boundary, not all three indiscriminately.

## 3. Core objects

| Object | Operational role | SRE concern |
|---|---|---|
| Metastore | Governance boundary | Regional assignment and ownership |
| Catalog/schema | Organize governed objects | Privilege scope and lifecycle |
| Storage credential | Represents cloud access identity | Trust, privilege, rotation |
| External location | Governs an object storage path using a credential | Overlap, URL scope, grants |
| Managed storage location | Location for managed objects | Lifecycle and ownership |
| External table | Governed table referencing external data | Metadata/data consistency |
| External volume | Governed non-tabular storage | File access and path safety |

An external location does not itself grant arbitrary access to every path. Effective privileges and cloud controls must be evaluated for the specific operation.

## 4. Cloud identity mapping

| AWS | Azure | GCP |
|---|---|---|
| IAM role and role trust used by the supported credential mechanism | Access connector/managed identity or other supported credential mechanism | Service account / supported identity mechanism |
| IAM role permissions, bucket policy, KMS as applicable | Azure RBAC, hierarchical namespace ACLs, encryption controls | IAM bindings, bucket policies, encryption controls |
| VPC endpoints, routing, DNS | Private endpoints, firewall, DNS | Private connectivity, routing, DNS |

Do not assume a human's cloud CLI identity is the same identity Databricks uses for a storage credential.

## 5. Operational design principles

1. Use one accountable owner per credential and location.
2. Scope cloud permissions to necessary buckets, containers, prefixes and operations.
3. Separate staging and production credentials and paths.
4. Avoid broad grants such as ALL PRIVILEGES as an incident shortcut.
5. Inventory all dependent tables, volumes, jobs and workspaces before changing a credential.
6. Preserve an approved rollback path before rotations.
7. Log and review privileged changes.
8. Use short-lived/federated cloud identity mechanisms where supported; avoid embedded long-lived secrets.
9. Treat cross-workspace and cross-region exposure as explicit design decisions.
10. Validate negative access controls as well as positive access.

## 6. Permission troubleshooting model

A storage access request can fail at:
- Workspace identity or job run-as principal.
- Catalog/schema/table/volume privileges.
- External location privileges.
- Storage credential privileges for administrative operations.
- Cloud role trust or managed identity association.
- Cloud bucket/container permissions or ACLs.
- KMS/encryption key authorization.
- Firewall, DNS, private endpoint or TLS.
- Object existence, lifecycle or storage service availability.

Distinguish permission to **use data** from permission to **create or manage governance objects**.

## 7. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks catalogs list --output json
~~~

Confirm the host and workspace before any investigation. Never export access tokens or full secret-bearing configuration into evidence bundles.

## 8. Lab B — Inventory storage credentials and locations

~~~powershell
databricks storage-credentials list --output json
databricks external-locations list --output json
databricks storage-credentials get --help
databricks external-locations get --help
~~~

The installed CLI may expose additional read-only commands and flags. Check help before use. Results are permission-filtered and may be paginated.

### Inventory worksheet

| External location | Storage URL | Credential | Owner | Environment | Critical dependents |
|---|---|---|---|---|---|
| demo-landing | s3://demo-staging-data/landing/ | demo-storage-credential | | staging | |
| | | | | | |

The sample URL is illustrative, not a real provisioned resource.

## 9. Lab C — Inspect governance grants

Use only objects you are authorized to inspect.

~~~sql
SHOW CATALOGS;
SHOW GRANTS ON CATALOG demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_schema;
SHOW GRANTS ON EXTERNAL LOCATION demo_landing;
SHOW GRANTS ON STORAGE CREDENTIAL demo_storage_credential;
~~~

Verify the supported SQL syntax in your Databricks runtime. Some statements require elevated visibility; inability to list grants does not prove no grants exist.

## 10. Lab D — AWS trust boundary assessment

~~~powershell
aws sts get-caller-identity
aws configure get region
aws iam get-role --role-name demo-staging-databricks-storage-role
~~~

The IAM role name is illustrative. Run only if authorized and redact trust-policy details in shared evidence. Inspect:
- Trust relationship with the correct Databricks principal and external ID where required.
- Scoped bucket/prefix access.
- Explicit deny conditions.
- S3 bucket policy and VPC endpoint policy.
- KMS key grants when customer-managed encryption applies.

A successful get-role call does not prove data-plane authorization.

## 11. Lab E — Azure identity and ACL assessment

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az storage account show --name demostagingstore --resource-group demo-staging-rg --query "{name:name,location:location}" -o json
~~~

Check the actual supported access connector/managed identity, RBAC scope, ADLS Gen2 directory ACLs, storage firewall, private endpoints and DNS. Distinguish management-plane access from storage data-plane access.

## 12. Lab F — GCP identity and bucket assessment

~~~powershell
gcloud config get-value project
gcloud storage buckets describe gs://demo-staging-data --format=json
~~~

Confirm the identity used by the Databricks storage credential, not merely the interactive shell account. Review service account IAM, bucket permissions, object paths, audit logging and private routing.

## 13. Lab G — Build a credential dependency map

For each credential, identify:
- External locations referencing it.
- Catalogs/schemas, external tables and volumes on those locations.
- Jobs and service principals accessing those objects.
- Workspaces attached to the governing metastore.
- Cloud storage paths, encryption keys and network controls.
- Business owners, SLOs, maintenance windows and recovery dependencies.

Do not rotate a credential until the blast radius is understood.

## 14. Lab H — Safe staging access validation

**Prerequisites:** approved staging resources, synthetic data, change owner, rollback and time limit.

1. Capture current grants and cloud policies.
2. Verify an authorized principal can access the intended synthetic object.
3. Verify an intentionally unauthorized staging principal is denied, if approved.
4. Record the exact boundary and sanitized error.
5. Restore original access configuration if changed.
6. Confirm cloud audit and Databricks audit evidence.
7. Validate that no unrelated path became accessible.

Never use production data for negative authorization tests.

## 15. Credential rotation runbook

1. Inventory dependents and confirm owner approvals.
2. Identify whether the supported identity mechanism rotates automatically or needs a controlled change.
3. Review current cloud trust, grants, encryption and endpoint dependencies.
4. Establish a staging rehearsal with synthetic reads/writes.
5. Create or configure replacement identity through the approved cloud procedure.
6. Validate replacement access and deny unrelated paths.
7. Plan production cutover with rollback, monitoring and a maintenance window if required.
8. Update the Databricks credential association only through supported APIs/UI and approved change.
9. Validate critical jobs, tables, volumes and storage audit events.
10. Retire obsolete identity only after confirmed dependency migration.

Do not assume storage credentials support an in-place identity swap in every cloud or deployment; use the documented cloud-specific mechanism.

## 16. Drift detection

Track approved state for:
- Storage credential owner and identity reference.
- External location URL and associated credential.
- Databricks privileges and workspace bindings.
- Cloud IAM trust, bucket/container permissions and ACLs.
- Private endpoints, DNS and storage firewall.
- Encryption key access.
- Audit log destinations and retention.

Alert on meaningful unauthorized differences. Preserve approved changes as baseline updates to prevent perpetual false positives.

## 17. Incident A — External location 403 after role rotation

**Symptoms:** previously working jobs fail while catalog metadata remains visible.

**Triage:** correlate change time, credential association, cloud trust, bucket policy and encryption key access.

**Recovery:** restore approved identity mapping or rollback the change.

**Acceptance:** authorized workload succeeds; unrelated storage remains inaccessible.

## 18. Incident B — Unexpected privilege escalation

**Symptoms:** a principal can access a location outside its approved scope.

**Triage:** inspect recent grants, credential usage, inherited roles and cloud policy changes.

**Recovery:** contain excess privileges through incident change control; investigate audit events.

**Acceptance:** negative tests fail as expected, with authorized access unaffected.

## 19. Incident C — Azure RBAC passes but ADLS path fails

**Symptoms:** management-plane checks succeed, data-plane path access fails.

**Triage:** inspect actual identity, container RBAC, hierarchical namespace ACLs and private networking.

**Recovery:** correct only the missing approved authorization or network rule.

**Acceptance:** intended path succeeds without widening container-level access unnecessarily.

## 20. Incident D — External location path overlap

**Symptoms:** governance conflicts or confusing ownership across nested prefixes.

**Triage:** inventory existing locations, storage paths and managed/external object references.

**Recovery:** redesign boundaries using supported configuration and planned migration.

**Acceptance:** paths have unambiguous ownership and no data is deleted.

## 21. Incident E — KMS key permission regression

**Symptoms:** storage listing works but encrypted object operations fail.

**Triage:** inspect encryption key policy, principal, key state and audit records.

**Recovery:** restore approved encryption access.

**Acceptance:** encrypted synthetic object access works, unrelated key access remains denied.

## 22. Incident F — Private endpoint regression looks like IAM failure

**Symptoms:** timeouts or endpoint failures after network change.

**Triage:** validate DNS and routes from comparable compute network; correlate firewall and endpoint logs.

**Recovery:** restore approved private path without opening public storage access.

**Acceptance:** private access and authorization both validated.

## 23. Troubleshooting matrix

| Failure | Databricks layer | Cloud layer | Network layer |
|---|---|---|---|
| Permission denied | Run-as and grants | IAM/ACL/key policy | Usually not primary |
| Timeout | Job/compute logs | Storage health | DNS, route, firewall |
| Path missing | External location URL | Bucket/object lifecycle | Usually not primary |
| Unexpected access | Grants and bindings | Overbroad policy | Endpoint exposure |
| Slow access | Job/task timing | Storage latency/throttle | Throughput/packet loss |

## 24. Production incident runbook

1. Establish affected storage path, jobs, identities and business impact.
2. Capture UTC timeline, sanitized errors and recent change history.
3. Confirm whether failure is catalog authorization, cloud IAM, network or object state.
4. Review blast radius and dependent critical workflows.
5. Apply smallest approved correction or rollback.
6. Test authorized and unauthorized access.
7. Validate data correctness, downstream processing and audit events.
8. Record incident evidence, root cause and preventive controls.

## 25. Monitoring and SLOs

Recommended operational signals:
- Authorized synthetic access success rate.
- Access denied anomalies by principal and location.
- Credential rotation success and time to recover.
- Unauthorized grant or cloud policy drift.
- Cloud storage throttling, timeout and encryption errors.
- Audit log ingestion lag and coverage.
- Critical workflow failures attributable to storage access.

Set thresholds from baseline and criticality, not universal fixed values.

## 26. Change controls and rollback

Before changes to grants, storage credentials or locations:
- Document owner, affected objects, expected behavior and rollback.
- Export current approved configuration securely.
- Validate cloud and Unity Catalog permissions independently.
- Stage the change with synthetic data.
- Confirm alert coverage and on-call ownership.
- Revalidate after cutover and revoke superseded privileges.
- Never delete storage objects as a routine credential rollback.

## 27. Operational acceptance checklist

- [ ] Storage credentials and external locations inventoried.
- [ ] Credential-to-location-to-job dependency map complete.
- [ ] Ownership and production/staging boundaries documented.
- [ ] Cloud identity trust and least-privilege policies reviewed.
- [ ] Encryption and private connectivity dependencies checked.
- [ ] Positive and approved negative staging access tests passed.
- [ ] Credential rotation and rollback rehearsed.
- [ ] Privilege drift and audit monitoring defined.
- [ ] Six incident scenarios reviewed.
- [ ] No production permissions or storage objects changed in lab.

## 28. Review questions

1. Why can Unity Catalog permission success coexist with a cloud storage denial?
2. What is the operational blast radius of rotating one shared storage credential?
3. Why does cloud CLI access not prove Databricks data-plane access?
4. How do you validate least privilege after a credential cutover?
5. What distinguishes IAM failure from a private endpoint regression?

## 29. Official references

- Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Storage credentials: https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/storage-credentials
- External locations: https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/external-locations
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- AWS IAM: https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
- Azure Storage authorization: https://learn.microsoft.com/azure/storage/blobs/authorize-data-operations-portal
- Google Cloud Storage IAM: https://cloud.google.com/storage/docs/access-control/iam

**Next:** Chapter 028 — Delta Lake Storage Reliability, Transaction Logs and Recovery Operations.
