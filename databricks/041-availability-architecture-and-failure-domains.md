# Chapter 041 — Availability Architecture and Failure Domains

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 09 — Reliability and Recovery  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Cloud Architecture, Platform Engineering and Incident Command  
**Prerequisites:** Chapters 001–010, 021–030, 031–040  
**Safety:** Production investigation is read-only. Fault injection, failover and infrastructure changes are restricted to approved staging environments with rollback, data protection and cost controls.

## 1. Mission

Design Databricks service availability around actual failure domains rather than assuming a workspace is automatically highly available. Identify dependencies across the control plane, compute plane, cloud networking, object storage, identity, metadata and consumers. Define service-specific recovery objectives and test failure isolation safely.

### Learning outcomes

- Map Databricks-managed and customer-managed failure boundaries.
- Identify cloud region, zone, workspace, account and shared-dependency risks.
- Separate high availability from disaster recovery and backup.
- Define RTO, RPO, SLI and SLO for critical workflows.
- Evaluate active/passive and active/active designs realistically.
- Plan cross-region data, code, identity and infrastructure recovery.
- Validate failure detection, escalation, safe recovery and output correctness.

## 2. Reference availability architecture

    Users / upstream systems / schedulers
                    |
             Databricks workspace
                    |
         Control-plane dependencies
                    |
           Jobs / SQL / governance
                    |
        Compute-plane dependencies
          /         |          \
       Workers   Networking    Identity
          \         |          /
          Cloud object storage
                    |
         Delta data and metadata
                    |
         Validated data products
                    |
       Downstream consumers / SLAs

Availability of a data product depends on the entire critical path, including correct and timely output, not just successful API access.

## 3. Databricks shared-responsibility boundaries

| Domain | Key questions | Operational evidence |
|---|---|---|
| Control plane | Which services are provider-managed? | Status, API errors, vendor communications |
| Workspace configuration | Is it reproducible? | IaC, policies, jobs, permissions |
| Compute | Can workers provision? | Cluster events, quota, capacity |
| Networking | Are private endpoints/routes resilient? | DNS, flow logs, route inventory |
| Cloud identity | Can compute access required resources? | IAM/RBAC and audit |
| Storage | Are data and metadata recoverable? | Versioning, replication, restore tests |
| Unity Catalog | Which regional/account dependencies exist? | Metastore/workspace mapping |
| Workflows | Can schedules and state be restored? | Jobs definitions, run history |
| Consumers | Can downstream applications tolerate delay? | Freshness and correctness SLOs |

Managed-service responsibilities and feature availability vary by cloud, region and compute mode. Confirm current product-specific guarantees before final design approval.

## 4. Failure-domain hierarchy

1. Task/process: one failed Spark task or driver process.
2. Node/VM: executor or driver host failure.
3. Availability zone: zone-scoped capacity or network disruption.
4. Subnet/network: route, DNS, firewall, NAT or private endpoint failure.
5. Workspace: configuration, permission or workspace-scoped incident.
6. Cloud region: compute, network or service disruption.
7. Cloud account/project/subscription: identity, policy, quota or billing failure.
8. Shared control/identity/metadata dependency: broader blast radius.
9. Downstream data consumer: a healthy Databricks job can still fail the end-to-end service.

Do not assume a multi-AZ compute deployment, automatic failover or multi-region metadata replication without verifying the exact Databricks architecture.

## 5. Availability, reliability and durability

**Availability:** service can meet a defined user request or delivery promise.

**Reliability:** service consistently produces correct outcomes over time.

**Durability:** persisted data survives defined failures.

**Recovery:** service and data can be restored after disruption.

A highly durable object store does not imply that a workflow is available, nor that a corrupted Delta table can be recovered to a required point in time.

## 6. Service-level objectives

Possible SLIs:
- Validated job completion before deadline.
- SQL request success within latency target.
- Dataset freshness and correctness.
- Time to detect critical failures.
- Time to restore consumer service.
- Percentage of eligible runs with validated output.

Define eligibility, maintenance exclusions, retries, cancellations and incomplete telemetry explicitly.

## 7. Lab A — Event-based availability calculation

~~~powershell
$eligibleRuns = 1000
$goodRuns = 996
$target = 0.995
$availabilityPct = 100 * $goodRuns / $eligibleRuns
$allowedBad = $eligibleRuns * (1 - $target)
$badRuns = $eligibleRuns - $goodRuns

[pscustomobject]@{
  EligibleRuns = $eligibleRuns
  GoodRuns = $goodRuns
  AvailabilityPct = [math]::Round($availabilityPct,2)
  AllowedBadRuns = $allowedBad
  BadRuns = $badRuns
  BudgetConsumedPct = [math]::Round(100*$badRuns/$allowedBad,1)
}
~~~

Synthetic result: 99.6% good runs and 80% of the event error budget consumed. Event-based SLOs are not interchangeable with time-based uptime SLOs.

## 8. Recovery objectives: RTO and RPO

**RTO:** maximum acceptable time to restore service after disruption.

**RPO:** maximum acceptable data loss or staleness measured from the recovery point.

Both must be defined per service and data asset. A job with replayable source data may have different RPO requirements from an externally mutated dataset.

| Service tier | RTO | RPO | Evidence |
|---|---|---|---|
| Critical | Business-defined | Business-defined | Recovery exercise |
| Important | Business-defined | Business-defined | Restore test |
| Noncritical | Business-defined | Business-defined | Documented procedure |

Never invent universal RTO/RPO values or claim that replication alone guarantees them.

## 9. Lab B — Recovery timeline calculation

~~~powershell
$impactStart = [datetimeoffset]"2026-10-14T10:00:00Z"
$recoveredAt = [datetimeoffset]"2026-10-14T10:47:00Z"
$lastRecoverableCommit = [datetimeoffset]"2026-10-14T09:53:00Z"

[pscustomobject]@{
  ObservedRecoveryMinutes = ($recoveredAt-$impactStart).TotalMinutes
  DataRecoveryGapMinutes = ($impactStart-$lastRecoverableCommit).TotalMinutes
}
~~~

This synthetic example reports 47 minutes to recovery and a 7-minute gap relative to impact start. RPO compliance must be assessed against the agreed reference point and actual recovered data, not simply the timestamp of a backup.

## 10. Regional architecture considerations

A regional failure can affect:
- Workspace access and control-plane APIs.
- VM provisioning and compute capacity.
- Private connectivity, DNS and cloud endpoints.
- Cloud object storage or replication.
- Identity, secrets and key-management dependencies.
- Unity Catalog access and governance.
- Job schedules, state and external integrations.

A second-region workspace is not automatically a working failover environment. Validate code, permissions, networking, storage, credentials and consumers end to end.

## 11. AWS availability considerations

Assess:
- Workspace region and compute networking.
- EC2 capacity and regional vCPU quotas.
- VPC subnets, routing, security groups and endpoints.
- S3 data protection, replication and KMS.
- IAM role trust and cross-region permissions.
- Jobs definitions and configuration recovery.
- CloudWatch, CloudTrail and regional dependency visibility.

S3 replication is asynchronous unless a specific product capability and contract establishes otherwise; verify lag and recovery semantics.

## 12. Azure availability considerations

Assess:
- Workspace region and managed resource boundaries.
- VM allocation, quotas and availability-zone support.
- VNet, NSG, private endpoints and private DNS.
- ADLS Gen2 redundancy/replication and access policies.
- Managed identities, service principals and Key Vault.
- Unity Catalog and workflow configuration.
- Azure Monitor and Activity Logs.

Storage redundancy options differ in failover behavior; do not equate geo-redundancy with immediate application failover.

## 13. GCP availability considerations

Assess:
- Workspace and compute region.
- Compute Engine regional quotas and zone capacity.
- VPC, firewall, DNS and private access.
- Cloud Storage location/replication and IAM.
- Service account and secret/key dependencies.
- Jobs and workspace configuration recovery.
- Cloud Monitoring and Cloud Audit Logs.

Cloud Storage durability and geographic redundancy do not guarantee immediate recovery of Databricks jobs or workspace metadata.

## 14. Active/passive versus active/active

| Pattern | Strength | Risk |
|---|---|---|
| Single-region with robust recovery | Simpler operations | Regional outage exposure |
| Warm standby | Faster planned recovery | Configuration drift and replication lag |
| Active/passive | Controlled failover | Promotion and reconciliation complexity |
| Active/active | Potentially lower regional disruption | Split-brain, duplicate writes, cost |
| Multi-cloud DR | Provider diversification | High complexity and portability limits |

For stateful pipelines, dual-active writes require explicit conflict, idempotency and ownership design. Never use two active writers without validated consistency controls.

## 15. Dependency graph lab

Create a worksheet for demo-hourly-refresh:

| Dependency | Failure domain | Redundancy | Detection | Recovery owner |
|---|---|---|---|---|
| Workspace API | Region/provider | Verify | API SLI | Platform |
| Compute | VM/zone/region | Verify | Startup metrics | Platform |
| DNS/private endpoint | Network | Verify | Synthetic probe | Networking |
| Storage | Bucket/account/region | Verify | Read/write probe | Storage |
| Unity Catalog | Governance/region | Verify | Authorized query | Data platform |
| Scheduler | Workspace/job | Verify | Deadline SLI | Workflow |
| Consumer | Downstream | Verify | Freshness check | Service owner |

Mark unknown architecture details as **unverified**, not redundant.

## 16. Lab C — Windows PowerShell Databricks preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks clusters list --output json
~~~

Confirm the host and caller. Inventory may be paginated or permission-filtered.

## 17. Lab D — Workspace recovery inventory

Collect authorized, read-only evidence for:
- Workspace URL, region and account mapping.
- Compute policies, instance pools and allowed node families.
- Job definitions and schedules.
- Secret scope names and credential dependencies (not secret values).
- Service principals and group mapping.
- Unity Catalog catalog/schema and external location references.
- Network/private endpoint/DNS dependencies.
- Terraform or other IaC state and source repository.
- External consumers and alert routing.

A configuration export is not a complete backup of data, secrets, runtime state or managed metadata.

## 18. Lab E — System table discovery

~~~sql
SHOW SCHEMAS IN system;
SHOW TABLES IN system.access;
SHOW TABLES IN system.compute;
SHOW TABLES IN system.lakeflow;
~~~

Where authorized and available:

~~~sql
DESCRIBE TABLE system.access.audit;
DESCRIBE TABLE system.lakeflow.job_run_timeline;
~~~

Availability, retention and schemas vary. Record gaps and alternative telemetry sources.

## 19. Lab F — Delta recovery capability inventory

~~~sql
DESCRIBE DETAIL demo_catalog.demo_staging.events;
DESCRIBE HISTORY demo_catalog.demo_staging.events;
~~~

Check retention, deletion, object storage protection, concurrent writers and external data dependencies. Delta history alone is not a complete offsite backup and may be constrained by retention/VACUUM.

## 20. Lab G — AWS failure-domain inventory

~~~powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-availability-zones --all-availability-zones
aws service-quotas list-service-quotas --service-code ec2 --max-results 20
aws ec2 describe-vpc-endpoints --max-results 20
~~~

Scope commands to the authorized environment. Quota availability does not prove capacity can be provisioned during a region-wide incident.

## 21. Lab H — Azure failure-domain inventory

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus -o table
az network private-endpoint list --resource-group demo-staging-rg -o table
~~~

Replace placeholder region and resource group. Verify the selected VM family and workspace-specific architecture.

## 22. Lab I — GCP failure-domain inventory

~~~powershell
gcloud config get-value project
gcloud compute regions describe us-central1 --format=json
gcloud compute zones list
gcloud compute routes list
~~~

Check the actual project, region and zone. Resource listing is not evidence of recovery readiness.

## 23. Lab J — Synthetic dependency failure analysis

Consider these cases:

| Scenario | Primary failure | Secondary impact |
|---|---|---|
| One executor lost | Node/process | Task retry |
| VM quota exhausted | Region/account | Cluster cannot scale |
| Private DNS wrong | Network | Storage unavailable |
| Cloud storage denied | IAM/policy | Job failure |
| Workspace API unavailable | Control plane | New runs/operations impacted |
| Delta corruption | Data integrity | Incorrect consumer output |
| Regional disruption | Region | Multiple dependencies |
| Identity provider outage | Shared identity | Authentication and automation |

For each case, identify blast radius, detector, recovery path and validation criteria.

## 24. Lab K — Failure-domain risk register

| Risk | Likelihood | Impact | Detection | Mitigation | Owner |
|---|---|---|---|---|---|
| Regional VM capacity | | | | | |
| Shared identity failure | | | | | |
| Unreplicated metadata | | | | | |
| Storage replication lag | | | | | |
| Duplicate replay writes | | | | | |
| Configuration drift | | | | | |

Use organization-approved risk scoring rather than invented probabilities.

## 25. Recovery ordering

1. Confirm incident scope and data-integrity risk.
2. Freeze unsafe duplicate writers or automated retries.
3. Verify cloud identity and key-management availability.
4. Validate network/DNS and storage access in recovery location.
5. Verify metadata, catalog permissions and configuration.
6. Provision approved compute within quota and budget.
7. Restore jobs and schedules in a controlled order.
8. Replay or reconcile data according to the recovery point.
9. Validate row counts, business invariants, freshness and consumer access.
10. Resume production traffic only after explicit acceptance.

Recovery sequence varies by service dependencies; document deviations.

## 26. Scenario A — Executor failure

**Impact:** single-task or stage retry.

**Evidence:** executor loss reason, Spark UI, node events.

**Recovery:** supported task/job retry if safe.

**Acceptance:** correct output without duplicate side effects.

## 27. Scenario B — Zone-level compute disruption

**Impact:** worker capacity reduction or cluster failure.

**Evidence:** zone and cloud instance events.

**Recovery:** approved alternate capacity within supported architecture.

**Acceptance:** startup and workflow SLO restored.

## 28. Scenario C — Private network dependency outage

**Impact:** storage and service connectivity failure.

**Evidence:** DNS, routes, flow logs and TLS.

**Recovery:** narrow approved network correction.

**Acceptance:** authorized paths work; prohibited access remains blocked.

## 29. Scenario D — Workspace configuration corruption

**Impact:** jobs, permissions or policies no longer match baseline.

**Evidence:** audit logs, IaC diff, configuration inventory.

**Recovery:** approved version-controlled restore.

**Acceptance:** functional jobs and least-privilege permissions.

## 30. Scenario E — Object storage unavailability

**Impact:** failed reads/writes or stale data.

**Evidence:** provider status, request errors, storage telemetry.

**Recovery:** approved retry/failover path with data reconciliation.

**Acceptance:** data correctness and agreed RPO validated.

## 31. Scenario F — Regional outage

**Impact:** multiple services unavailable.

**Evidence:** provider incident, Databricks status, shared dependencies.

**Recovery:** execute tested DR plan, not improvised multi-region writes.

**Acceptance:** recovery within agreed RTO/RPO, no split-brain.

## 32. Scenario G — Identity/key dependency outage

**Impact:** compute cannot authenticate to storage or catalog.

**Evidence:** IAM/KMS/secret errors and audit.

**Recovery:** approved identity/key restoration.

**Acceptance:** authorized access restored without overprivileging.

## 33. Scenario H — Green job, stale output

**Impact:** consumer SLO breached despite successful run state.

**Evidence:** dataset timestamps and correctness checks.

**Recovery:** replay or correct data safely.

**Acceptance:** consumer-visible freshness and validation restored.

## 34. Incident response runbook

1. Declare severity from consumer impact and RTO/RPO risk.
2. Identify affected failure domains and shared dependencies.
3. Capture UTC timeline and last validated recovery point.
4. Protect data integrity and prevent duplicate writes.
5. Check provider/Databricks incident communications.
6. Evaluate in-region recovery versus approved DR promotion.
7. Execute least-risk recovery with change approval.
8. Validate compute, network, IAM, storage and metadata.
9. Validate output correctness, freshness and consumer access.
10. Record actual RTO/RPO, RCA and improvement owners.

## 35. Architecture acceptance scorecard

| Requirement | Status | Evidence |
|---|---|---|
| Critical services mapped | Not assessed | |
| Failure domains documented | Not assessed | |
| Shared dependencies identified | Not assessed | |
| RTO/RPO approved | Not assessed | |
| Recovery location capacity verified | Not assessed | |
| Storage/data protection verified | Not assessed | |
| Workspace config reproducible | Not assessed | |
| Identity and keys recoverable | Not assessed | |
| Failover/failback ownership defined | Not assessed | |
| Data correctness checks defined | Not assessed | |
| DR drill scheduled and scoped | Not assessed | |
| Cost and security reviewed | Not assessed | |

## 36. GO / CONDITIONAL GO / NO-GO

**GO:** Failure domains understood, RTO/RPO defined, critical dependencies recoverable and recovery path validated.

**CONDITIONAL GO:** Only documented noncritical gaps remain with owner, mitigation and deadline.

**NO-GO:** Unverified recovery point, uncontrolled dual writers, missing keys/identity, unknown metadata dependencies or inability to validate data correctness.

## 37. Review questions

1. Why is a second workspace not sufficient to claim regional disaster recovery?
2. How do durability, availability and RPO differ?
3. Which dependencies must recover before replaying a stateful pipeline?
4. Why can active/active architecture create duplicate or conflicting data?
5. What proves that a failover achieved the consumer's actual SLO?

## 38. Official references

- Databricks disaster recovery: https://docs.databricks.com/aws/en/security/disaster-recovery
- Databricks architecture: https://docs.databricks.com/aws/en/getting-started/high-level-architecture
- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Databricks Delta Lake: https://docs.databricks.com/aws/en/delta/
- AWS Well-Architected Reliability Pillar: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html
- Azure Well-Architected Reliability: https://learn.microsoft.com/azure/well-architected/reliability/
- Google Cloud reliability: https://cloud.google.com/architecture/framework/reliability
- Google SRE workbook: https://sre.google/workbook/

**Next:** Chapter 042 — Backup, Recovery and Data Protection Boundaries.
