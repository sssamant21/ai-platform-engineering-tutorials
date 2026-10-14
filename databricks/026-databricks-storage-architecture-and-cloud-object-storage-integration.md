# Chapter 026 — Databricks Storage Architecture and Cloud Object Storage Integration

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 06 — Storage, Data Access and Infrastructure Reliability  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, Infrastructure, Cloud Platform, Security and Data Platform Operations  
**Prerequisites:** Chapters 001–025  
**Lab safety:** Read-only production inspection; approved staging tests only

## 1. Mission

Understand and operate the storage boundaries used by Databricks, including Unity Catalog managed storage, external locations, cloud object storage, identity and network dependencies. The objective is to diagnose and recover storage access failures without weakening security or disrupting production data.

### Learning objectives

- Explain the separation between Databricks compute, metadata and cloud object storage.
- Distinguish managed tables, external tables, volumes and workspace storage.
- Map Unity Catalog storage credentials and external locations to cloud IAM.
- Validate S3, ADLS Gen2 and GCS authorization and network paths.
- Diagnose 401/403, DNS, TLS, endpoint, throttling and region failures.
- Establish monitoring, recovery runbooks and operational acceptance controls.

## 2. Storage reference architecture

    Databricks account / workspace
                 |
            Unity Catalog
          /              \
    Metastore          Grants
          \              /
         Storage credential
                 |
          External location
                 |
       Cloud identity / trust
                 |
       Network, DNS and TLS
                 |
          Object storage
         /      |       \
        S3   ADLS Gen2   GCS
                 |
         Tables / volumes

The catalog defines governance and access control; the cloud provider enforces its own identity, object storage and network policies. Successful Unity Catalog authorization does not guarantee that the cloud storage request succeeds.

## 3. Storage categories and ownership

| Storage concept | Primary purpose | Operational concern |
|---|---|---|
| Managed table storage | Governed table data managed by platform semantics | Location, lifecycle, permissions |
| External table storage | Table data stored in externally governed paths | Path ownership, lifecycle, IAM |
| Unity Catalog volumes | Governed non-tabular files | Grants, storage path, applications |
| Workspace/default storage | Workspace-managed platform assets, depending on deployment | Scope, lifecycle, supported usage |
| Job logs and artifacts | Operational output and logs | Retention, sensitive content |
| Cloud object storage | Durable underlying data | IAM, network, availability, request cost |

Managed versus external data lifecycle behavior must be verified for the specific object and platform version. Never delete cloud objects as a substitute for dropping catalog metadata.

## 4. Unity Catalog storage objects

**Storage credential:** a governed reference to a cloud identity mechanism authorized to access storage.

**External location:** a governed association between a cloud storage URL and a storage credential.

**Managed storage location:** a configured location for governed managed objects, at the applicable metastore/catalog/schema scope.

**Volume:** a governed file access abstraction for supported non-tabular files.

Separate:
1. Workspace and account identity.
2. Unity Catalog permissions.
3. Storage credential configuration.
4. Cloud IAM/trust and storage ACLs.
5. Network connectivity and endpoint policy.

Do not grant workspace administrator permissions to fix a cloud IAM denial.

## 5. Cloud mapping

| Dimension | AWS | Azure | GCP |
|---|---|---|---|
| Storage | Amazon S3 | ADLS Gen2 | Google Cloud Storage |
| Identity | IAM role and trust | Managed identity / Entra principal, depending on configuration | Service account / IAM |
| Authorization | Role and bucket policies | Azure RBAC and ACLs | Bucket/object IAM |
| Network | VPC endpoints, routes, DNS | Private endpoints, DNS, firewall | Private Google Access / routing and DNS |
| Audit | CloudTrail, S3 access telemetry where configured | Azure Monitor and storage diagnostics | Cloud Audit Logs |
| Capacity | Request rate, network, quotas | Account throttling, bandwidth | Quotas, request/network performance |

Exact Databricks credential configuration differs by cloud and feature. Consult cloud-specific documentation before provisioning or rotating a credential.

## 6. Failure domains

| Symptom | Likely boundary | First checks |
|---|---|---|
| UC privilege error | Catalog authorization | Effective principal and grants |
| Cloud storage 403 | Cloud IAM/ACL | Credential trust and storage policy |
| DNS failure | Network/name resolution | Private DNS and endpoint |
| TLS/connect timeout | Network/egress | Routing, firewall, proxy, certificates |
| Intermittent 429/5xx | Throttling/service | Request rates and provider telemetry |
| Path not found | Object path/lifecycle | Exact URL, case, partition, deletion |
| Wrong-region behavior | Region/endpoint | Workspace, bucket and endpoint regions |
| Slow reads | Storage, compute or network | Request latency, Spark metrics, throughput |

An HTTP status code is only a starting clue. Preserve the complete sanitized error and correlate cloud and Databricks telemetry.

## 7. Lab A — PowerShell workspace preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Verify the workspace host, account and identity before continuing. Never export tokens or secrets to evidence files.

## 8. Lab B — Read-only Unity Catalog storage inventory

~~~powershell
databricks metastores list --output json
databricks catalogs list --output json
databricks storage-credentials list --output json
databricks external-locations list --output json
databricks volumes list --help
~~~

Visibility depends on privileges and may be incomplete. Storage URLs and identity references can be sensitive; sanitize output before sharing.

### Inspection worksheet

| Object | Scope | Owner | Credential / identity | Risk |
|---|---|---|---|---|
| Metastore | Account/region | | | |
| Catalog | Governance | | | |
| External location | Storage URL | | | |
| Storage credential | Cloud identity | | | |
| Volume | Governed file path | | | |

## 9. Lab C — Read-only SQL governance checks

Use authorized example objects; replace identifiers:

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_schema;
~~~

Where supported and authorized, inspect grants on the specific external location and storage credential using current SQL syntax and permissions. Do not assume catalog access implies cloud object access.

## 10. Lab D — AWS S3 context and access review

~~~powershell
aws sts get-caller-identity
aws configure get region
aws s3api get-bucket-location --bucket demo-staging-data
~~~

Replace the example bucket with an authorized bucket. The commands validate caller and bucket region, **not** the effective role used by Databricks compute or Unity Catalog. Inspect role trust, bucket policy, endpoint policy and storage credential separately.

### AWS troubleshooting sequence

1. Confirm the exact bucket and object prefix.
2. Verify storage credential IAM role and trust configuration.
3. Review applicable IAM and bucket policy denies.
4. Check VPC endpoint policy, routing and DNS.
5. Correlate CloudTrail/S3 telemetry if enabled.
6. Validate with approved synthetic access, not production object mutation.

## 11. Lab E — Azure ADLS Gen2 context and access review

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az storage account show --name demostagingstore --resource-group demo-staging-rg --query "{name:name,location:location,kind:kind}" -o json
~~~

Replace the example resource identifiers. Review the actual Databricks-managed identity or approved principal, Azure RBAC, hierarchical namespace ACLs, firewall rules and private endpoint DNS.

A successful Azure CLI login does not prove the Databricks runtime's identity has data-plane access.

## 12. Lab F — GCP Cloud Storage context and access review

~~~powershell
gcloud config get-value project
gcloud storage buckets describe gs://demo-staging-data --format=json
~~~

Replace the bucket and confirm authorization. Review the actual service account/identity binding, bucket IAM, routing and cloud audit logs.

## 13. Lab G — Private connectivity diagnostics

From an **approved test host** on the relevant network, use:

~~~powershell
Resolve-DnsName example-storage-endpoint.invalid
Test-NetConnection example-storage-endpoint.invalid -Port 443
~~~

The hostname is intentionally non-routable; replace it with an approved real storage endpoint. These tests are meaningful only from a network path comparable to the Databricks compute plane. A successful TCP connection does not establish IAM authorization or application-layer success.

### Network checklist

- Correct endpoint and region.
- Private DNS resolves to intended destination.
- Subnet routes and firewall rules allow approved traffic.
- TLS interception and proxy policies are compatible.
- Storage firewall permits the approved network identity.
- No unintended public egress is introduced.

## 14. Lab H — Controlled staging access validation

**Preconditions:** approved staging workspace, synthetic data, named owner, cost cap and rollback.

1. Identify the approved external location or volume.
2. Record existing grants and storage credential configuration.
3. Perform an authorized read-only metadata/listing operation.
4. Confirm data access through an approved Databricks interface.
5. Test a **synthetic unauthorized principal** only with explicit approval and disposable resources.
6. Correlate Databricks audit events and cloud authorization logs where available.
7. Confirm intended access succeeds and unintended access is denied.
8. Restore staging state and retain sanitized evidence.

Do not weaken production bucket policies, ACLs, external locations or network controls to make a test pass.

## 15. Storage reliability SLIs

| SLI | Definition |
|---|---|
| Authorized access success | Successful authorized synthetic requests / eligible requests |
| Storage error rate | Storage errors / total observed storage operations |
| Read/write latency | p50/p95/p99 by operation and workload |
| Throttling rate | Provider throttling responses / requests |
| Data freshness | Latest validated object/table update versus expectation |
| Permission drift | Unexpected changes to storage IAM or UC grants |
| Audit coverage | Critical storage event classes monitored |

Set thresholds from workload baselines and contractual needs. Cloud object storage service availability is not equivalent to application-level data availability.

## 16. Incident A — Unity Catalog permission denied

**Triage:** confirm run-as identity, workspace assignment, catalog/schema/object grants and recent governance changes.

**Recovery:** restore minimum approved grants.

**Acceptance:** synthetic authorized access succeeds; negative test remains denied.

## 17. Incident B — S3/ADLS/GCS 403 after credential change

**Triage:** storage credential mapping, cloud identity trust, IAM/ACL, policy denies and recent rotation.

**Recovery:** restore the intended identity or approved policy; never grant blanket administrator access.

**Acceptance:** intended operation succeeds and unrelated paths remain restricted.

## 18. Incident C — Private endpoint DNS regression

**Triage:** endpoint FQDN resolution from relevant network, private DNS zone, forwarding, routes and firewall.

**Recovery:** restore approved DNS/routing configuration.

**Acceptance:** intended private connectivity works without opening unnecessary public access.

## 19. Incident D — Storage throttling during concurrent jobs

**Triage:** correlate job concurrency, request rate, object layout, provider metrics and retry storms.

**Recovery:** reduce contention, apply bounded backoff and coordinate capacity/performance improvements.

**Acceptance:** latency and error rate normalize without duplicate output.

## 20. Incident E — Data path or lifecycle mistake

**Triage:** exact object URL, catalog reference, lifecycle rules, recent deletion/change and available recovery features.

**Recovery:** follow cloud-specific recovery and backup process; do not blindly recreate or overwrite objects.

**Acceptance:** expected data restored, catalog metadata consistent and access controlled.

## 21. Incident F — Cloud storage audit gap

**Triage:** confirm which data events are enabled, log destinations, filters, retention and SIEM ingestion.

**Recovery:** restore logging with approved change, backfill supported records and document gaps.

**Acceptance:** synthetic event observed end-to-end; remaining blind spots explicitly recorded.

## 22. Troubleshooting decision tree

    Storage operation fails
            |
    Workspace identity valid?
       no -> auth / workspace assignment
       yes
            |
    Unity Catalog grant valid?
       no -> catalog / schema / object privilege
       yes
            |
    Cloud identity and policy valid?
       no -> trust / IAM / ACL
       yes
            |
    DNS, TLS and route healthy?
       no -> endpoint / network
       yes
            |
    Object exists and service healthy?
       no -> path / lifecycle / provider incident
       yes -> workload / performance investigation

## 23. Production recovery runbook

1. Declare impact, affected workspace and storage path classification.
2. Preserve sanitized errors, UTC timeline and recent changes.
3. Identify the failing boundary: UC, cloud IAM, network, storage or workload.
4. Confirm whether a shared cloud incident or quota issue exists.
5. Choose the smallest approved mitigation with rollback.
6. Validate authorized access and negative access controls.
7. Check data correctness, downstream workflows and cost.
8. Communicate restoration and residual risks.
9. Document RCA, owner and preventive monitoring.

## 24. Change and rollback

Before modifying storage credentials, external locations, IAM, firewall, private DNS or lifecycle:
- Inventory dependent jobs, tables, volumes and workspaces.
- Capture approved current configuration.
- Test using synthetic data in staging.
- Plan dual-control approval and maintenance window.
- Define rollback that preserves data and least privilege.
- Monitor errors, access denials, latency and audit events.
- Never assume rolling back catalog metadata reverses cloud object changes.

## 25. Operational acceptance checklist

- [ ] Storage categories and ownership mapped.
- [ ] Unity Catalog credential/location relationships inventoried.
- [ ] Cloud IAM/trust and storage policies reviewed.
- [ ] Private connectivity, DNS and firewall paths validated.
- [ ] Read-only AWS/Azure/GCP context checks completed as applicable.
- [ ] Synthetic staging authorized access validated.
- [ ] Unauthorized access remains denied.
- [ ] Storage SLIs and audit coverage documented.
- [ ] Six incident scenarios reviewed.
- [ ] Recovery and rollback runbooks approved.
- [ ] No production storage, IAM or network configuration changed in lab.

## 26. Review questions

1. Why can a user have Unity Catalog privileges but receive a cloud storage 403?
2. What is the difference between a storage credential and an external location?
3. Why must private DNS tests originate from a comparable network path?
4. How can retry storms make storage throttling worse?
5. What evidence demonstrates storage recovery without weakening security?

## 27. Official references

- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Databricks external locations: https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/external-locations
- Databricks storage credentials: https://docs.databricks.com/aws/en/connect/unity-catalog/cloud-storage/storage-credentials
- AWS S3: https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- Azure Data Lake Storage Gen2: https://learn.microsoft.com/azure/storage/blobs/data-lake-storage-introduction
- Google Cloud Storage: https://cloud.google.com/storage/docs
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/

**Next:** Chapter 027 — Unity Catalog Storage Credentials, External Locations and Access Operations.
