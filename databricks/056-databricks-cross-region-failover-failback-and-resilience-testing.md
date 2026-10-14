# Chapter 056 — Databricks Cross-Region Failover, Failback and Resilience Testing

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 12 — Disaster Recovery and Production Resilience  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform Engineering, Data Engineering, Security, DR Coordinators  
**Prerequisites:** Chapters 041–055  
**Safety:** All hands-on mutations are isolated staging only. Production inventory is read-only. Cross-region cutover, DNS changes, writer fencing, grant changes, checkpoint changes, restore and failback require incident/change authorization. Never assume Databricks workspace, Unity Catalog metastore or managed tables can be transparently moved between regions.

## 1. Mission

Design, rehearse and validate cross-region Databricks disaster recovery, including regional failure detection, safe active-passive promotion, data and metadata recovery, controlled workload restart, downstream routing, split-brain prevention and failback. Measure actual recovery objectives rather than equating infrastructure availability with service restoration.

## 2. Learning objectives

- Compare active-passive, warm standby, pilot-light and selective active-active designs.
- Map Databricks workspace, storage, metastore, network, identity and consumer dependencies.
- Define authoritative data ownership, recovery point and replication consistency.
- Enforce writer fencing and avoid duplicate output during cutover.
- Reconstruct infrastructure, bundles, policies and governance safely.
- Validate streaming checkpoint and replay strategies.
- Measure actual RPO/RTO and end-to-end consumer correctness.
- Rehearse failback without causing a second outage.

## 3. Reference architecture

    Primary region                        Recovery region
    --------------                        ---------------
    Ingestion -> Databricks Jobs          Standby Databricks workspace
             -> Delta tables  ----------> Recoverable/replicated data
             -> Consumers                Independent IAM/KMS/network
    IaC + Bundles + policies ------------> Reproducible configuration
    UC grants / metadata  --------------> Governed reconstruction
    Monitoring + audit ------------------> Independent monitoring
                |                               |
         Failure detection                 Authorized promotion
                |                               |
         FENCE primary writers -----> Validate recovery point
                                                |
                                         Resume one writer
                                                |
                                      Validate consumers / SLO
                                                |
                                          Controlled routing

Arrows represent required engineered processes, not automatic platform-native replication guarantees.

## 4. Regional service boundaries

Inventory Databricks workspace IDs/URLs, account and region, compute types, SQL warehouses, jobs, service principals, secret scopes, network paths, Unity Catalog metastore assignments, catalogs, schemas, storage credentials, external locations, object storage, KMS, Terraform backends, GitLab runners, DNS and downstream endpoints. Determine which resources are regional and which dependencies remain global or shared.

## 5. DR architecture patterns

| Pattern | Benefit | Tradeoff |
|---|---|---|
| Pilot light | Lower standby cost | Longer infrastructure recovery |
| Warm standby | Faster readiness | Continuous capacity and synchronization cost |
| Active-passive | Clear single writer and failover ownership | Promotion time and standby maintenance |
| Active-active reads | Geographic read availability | Replication lag and consistency semantics |
| Active-active writes | Potential locality and throughput | Complex conflict resolution and fencing |

Use active-active writes only with a proven application-level conflict-resolution and transactional design. Delta tables do not automatically provide cross-region multi-writer consistency.

## 6. RPO, RTO and recovery completeness

RPO is measured against the last **verified recoverable committed data point**. RTO ends when required consumers, governance, data correctness and service SLOs are restored, not when the alternate workspace starts. Record business-approved thresholds for each service and distinguish target, projected and measured values.

## 7. Replication and recovery point design

Document whether data is restored from independent backups, storage replication, upstream replay or controlled table copy. Validate transaction log and data-file consistency, lag, retention, encryption keys, schema, object ownership and deletion propagation. Replicated data may be inconsistent or compromised if corruption is copied before detection.

## 8. Metadata and Unity Catalog

Catalog names and privileges alone are not sufficient. Verify metastore placement, catalog/schema ownership, external locations, storage credentials, table locations, grants, service principals, network and cloud access. Follow supported Databricks and cloud-specific procedures for the actual metastore and managed/external table types. Avoid claiming generic cross-region metastore portability.

## 9. Single-writer invariant

At any time, for each protected output dataset and checkpoint, there must be one authorized active writer unless an explicitly tested distributed consistency design exists. Fencing mechanisms may include cloud IAM write denial, application lease/leader election, disabled schedules, network isolation or orchestrator controls. A paused schedule alone does not stop already-running jobs.

## 10. Split-brain prevention

Before promoting recovery writers:
1. Determine whether primary writers are reachable.
2. Obtain authoritative incident commander approval.
3. Fence primary writers using a verifiable control.
4. Account for in-flight transactions and external side effects.
5. Verify recovery region is using the selected consistent data point.
6. Establish recovery-region write ownership.
7. Start jobs in dependency order and validate outputs.
8. Record exact ownership transitions and evidence.

When primary fencing cannot be proven, treat promotion as high risk and escalate rather than blindly creating concurrent writers.

## 11. Failover dependency order

Cloud identity and KMS -> networking/DNS -> object storage -> Terraform-managed workspace dependencies -> Unity Catalog and storage credentials -> compute policies -> bundles/jobs -> checkpoints/source replay -> synthetic validation -> consumer routing -> monitoring and observation.

Some dependencies can be provisioned in parallel, but write promotion must respect data consistency.

## 12. Lab A — Preflight [SAFE-READ]

~~~powershell
git rev-parse HEAD
terraform version
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging-primary"
databricks auth describe
databricks current-user me --output json
~~~

Repeat for the explicitly approved staging recovery profile. Record distinct hosts, regions, principals and cloud accounts. Do not assume a profile name guarantees the target.

## 13. Lab B — Cloud identity verification [SAFE-READ]

**AWS**

~~~powershell
aws sts get-caller-identity
aws configure get region
~~~

**Azure**

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
~~~

**GCP**

~~~powershell
gcloud config get-value project
gcloud auth list
~~~

Run separately under source and recovery identities. Check independent key access and quotas.

## 14. Lab C — Workspace inventory [SAFE-READ]

~~~powershell
databricks jobs list --output json
databricks clusters list --output json
databricks cluster-policies list --output json
databricks warehouses list --output json
~~~

Capture approved redacted inventory for both staging workspaces. CLI output can be permission-filtered and paginated; missing resources require confirmation before reporting them absent.

## 15. Lab D — Validate recovery bundles [SAFE-READ]

~~~powershell
databricks bundle validate --target staging
databricks bundle summary --target staging
~~~

Use separately configured primary and recovery targets in the real bundle. Compare job run-as identities, schedules, resource ownership, secrets references and workspace hosts. Do not activate standby schedules merely to validate configuration.

## 16. Lab E — Terraform recovery plan [CONTROLLED-READ]

~~~powershell
terraform -chdir=infrastructure/environments/dr-staging fmt -check -recursive
terraform -chdir=infrastructure/environments/dr-staging init -backend=false
terraform -chdir=infrastructure/environments/dr-staging validate
~~~

If an approved protected backend is available, perform a separately authorized plan. Never apply or expose raw state/plan files in an untrusted runner.

## 17. Lab F — Delta recovery-point discovery [SAFE-READ]

~~~sql
DESCRIBE HISTORY demo_catalog.demo_staging.synthetic_events;
DESCRIBE DETAIL demo_catalog.demo_staging.synthetic_events;
~~~

Record latest committed version and UTC timestamp. Historical availability depends on retained log and data files; a version number alone does not guarantee restore.

## 18. Lab G — Data integrity manifest [SAFE-READ]

~~~sql
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT event_id) AS distinct_ids,
  SUM(CASE WHEN event_id IS NULL THEN 1 ELSE 0 END) AS null_ids,
  MAX(event_time) AS latest_event_time
FROM demo_catalog.demo_staging.synthetic_events;
~~~

Capture the same manifest from an independently prepared staging recovery copy. Compare schema, business invariants, uniqueness and freshness. The tables must already exist and be authorized.

## 19. Lab H — UC authorization validation [SAFE-READ]

~~~sql
SELECT current_user();
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_staging;
~~~

Validate positive access with the intended workload principal and negative access with an approved restricted identity. A successful admin query does not establish workload authorization.

## 20. Lab I — RPO/RTO calculator

~~~powershell
$failureUtc = [datetimeoffset]"2026-10-14T12:00:00Z"
$recoveryPointUtc = [datetimeoffset]"2026-10-14T11:53:00Z"
$consumerAcceptedUtc = [datetimeoffset]"2026-10-14T12:42:00Z"
[pscustomobject]@{
  ActualRPO_Minutes = ($failureUtc - $recoveryPointUtc).TotalMinutes
  ActualRTO_Minutes = ($consumerAcceptedUtc - $failureUtc).TotalMinutes
}
~~~

Expected: 7-minute realized RPO and 42-minute realized RTO. In drills use recorded timestamps and verify the selected recovery point contains committed accepted data.

## 21. Lab J — Controlled failover tabletop [NO MUTATION]

1. Choose one isolated synthetic data product.
2. Identify primary and recovery region dependencies.
3. Declare simulated regional loss and UTC failure onset.
4. Select independently verified recoverable point.
5. Demonstrate how primary writer fencing would be proven.
6. Review Terraform, bundle, UC, key and network prerequisites.
7. Plan single-writer recovery job activation.
8. Define consumer validation and routing.
9. Calculate projected and measured RPO/RTO.
10. Document go/no-go and failback owner.

**Pass:** an independent operator can execute the written recovery sequence without inventing permissions, paths or missing steps.

## 22. Controlled staging failover drill

Only after approval, use synthetic data and isolated jobs. Establish a known-good baseline, stop/fence staging primary writer, verify no in-flight run remains, select consistent recovery data, promote recovery writer, execute deterministic synthetic checks and route only test consumers. Capture each step and abort on integrity mismatch. Never test by deleting live source buckets or disabling shared production infrastructure.

## 23. Monitoring during failover

Track primary and recovery writer status, replication/backup age, job queue and failures, source offsets, checkpoint ownership, compute provisioning, SQL latency, data freshness, cloud KMS/network availability and consumer errors. Monitoring must operate even when the primary region is unreachable.

## 24. DNS and endpoint routing

Use preapproved routing controls, endpoint health checks and documented TTLs. Verify clients, credentials, storage paths, IP allowlists, private connectivity and downstream destinations. DNS switch success is not equivalent to successful client migration; caches and long-lived connections can delay convergence.

## 25. Failback prerequisites

Do not fail back automatically when the primary region returns. Require:
- Primary infrastructure and security posture validated.
- Data divergence and replay gaps reconciled.
- Replication direction and recovery-point consistency established.
- Primary region's jobs and schedules remain fenced until promotion.
- Recovery writer drains or is safely stopped.
- Consumer routing and credential readiness confirmed.
- Explicit failback approval, rollback plan and observation window.

## 26. Controlled failback sequence

1. Declare failback window and freeze unrelated changes.
2. Capture recovery-region final committed watermark.
3. Stop new writes and verify no in-flight transactions.
4. Reconcile recovery outputs to the primary destination.
5. Validate schema, counts, uniqueness and source offsets.
6. Confirm primary identity, KMS, network and governance.
7. Promote primary writer after recovery writer is fenced.
8. Switch test consumers then approved production consumers.
9. Observe SLOs, compare manifests and audit access.
10. Close with final ownership, recovery timings and evidence.

## 27. Disaster recovery versus high availability

High availability handles local faults using redundancy and automated recovery within an expected failure domain. Disaster recovery restores operations across a larger failure domain, often with deliberate human approval and potential data loss. Do not advertise HA failover times as cross-region DR RTO without end-to-end drills.

## 28. Scenario A — Primary region unavailable

**Detect:** widespread compute/network/storage failure.  
**Respond:** declare incident, select recovery point, verify writer fencing and recovery dependencies.  
**Accept:** consumers validated within RPO/RTO.

## 29. Scenario B — Primary writers remain active

**Detect:** active runs or write events after planned fencing.  
**Respond:** stop promotion, enforce authoritative write denial or lease controls.  
**Accept:** single-writer invariant proven.

## 30. Scenario C — Replication lags beyond RPO

**Detect:** recovery watermark older than allowed.  
**Respond:** escalate, choose verified alternate backup or business-approved degraded recovery.  
**Accept:** actual data loss measured and accepted.

## 31. Scenario D — Cross-region key unavailable

**Detect:** recovery data exists but cannot decrypt.  
**Respond:** activate approved KMS/key access recovery, preserve security controls.  
**Accept:** isolated read and workload validation succeed.

## 32. Scenario E — UC grants missing in recovery

**Detect:** jobs fail with access denied after promotion.  
**Respond:** reconstruct approved least-privilege grants and storage access.  
**Accept:** workload success and negative authorization checks.

## 33. Scenario F — Recovery compute capacity exhausted

**Detect:** startup failures, quota errors or excessive queue time.  
**Respond:** use preapproved instance/capacity fallback, assess cost.  
**Accept:** critical jobs meet recovery deadline.

## 34. Scenario G — Streaming replay duplicates output

**Detect:** uniqueness checks fail after checkpoint restart.  
**Respond:** fence writers, assess source retention and idempotent replay.  
**Accept:** deterministic accepted output.

## 35. Scenario H — DNS switched but clients fail

**Detect:** consumer error rate remains elevated.  
**Respond:** check private endpoints, DNS caches, credentials and routing.  
**Accept:** actual consumer queries and writes pass.

## 36. Scenario I — Failback starts both writers

**Detect:** simultaneous writes in primary and recovery regions.  
**Respond:** stop unsafe writers, reconcile commits and checkpoints.  
**Accept:** one authoritative writer and data integrity.

## 37. Scenario J — Monitoring depends on failed region

**Detect:** recovery telemetry unavailable during outage.  
**Respond:** use independently hosted monitors and cloud audit sources.  
**Accept:** recovery-region alerts and dashboards function.

## 38. Production failover runbook

~~~markdown
# Databricks Cross-Region Failover
- Incident ID / commander / severity:
- Business service / data owners:
- Primary and recovery region:
- Failure onset UTC:
- RPO / RTO targets:
- Latest verified recoverable commit:
- Primary writer fencing mechanism and proof:
- Storage / KMS / network / identity readiness:
- Unity Catalog and configuration readiness:
- Jobs and checkpoint promotion order:
- Consumer routing plan:
- Data correctness and freshness acceptance:
- Actual RPO / RTO:
- Monitoring and stakeholder update cadence:
- Failback prerequisites and owner:
~~~

## 39. Production failback runbook

Record failback authorization, recovery writer drain, last committed watermark, reconciliation method, primary restoration status, grant validation, single-writer handover, consumer routing, observation window and rollback authority. Never restart the primary's original schedule before confirming recovery writer fencing.

## 40. Evidence package

    cross-region-dr-YYYYMMDD/
      01-service-scope-rpo-rto.md
      02-regional-dependency-map.md
      03-recovery-point-and-replication.md
      04-fencing-and-writer-ownership.md
      05-identity-network-kms.md
      06-terraform-bundles-uc.md
      07-failover-timeline.md
      08-data-and-consumer-validation.md
      09-observability-and-alerts.md
      10-failback-timeline.md
      11-actual-rpo-rto.md
      12-lessons-and-signoff.md

Restrict state, plan files, credentials and raw audit evidence.

## 41. Acceptance scorecard

| Gate | Required evidence |
|---|---|
| Architecture | Documented failure domains and owners |
| Recovery point | Verified consistent data and lag |
| RPO/RTO | Approved targets and measured actuals |
| Single writer | Fencing and ownership evidence |
| Storage | Data/log completeness and protection |
| Cloud | IAM, KMS, networking and quotas |
| Governance | UC permissions and metadata |
| Deployment | IaC/bundle revisions validated |
| Pipelines | Idempotency and checkpoint plan |
| Consumers | Freshness, correctness and connectivity |
| Monitoring | Independent alert delivery |
| Failback | Tested controlled ownership transfer |
| Closure | Incident/DR owner approval |

## 42. GO / CONDITIONAL GO / NO-GO

**GO:** Recovery point is verified, writer fencing proven, alternate-region dependencies work, consumers pass correctness checks and RPO/RTO are within approved limits.

**CONDITIONAL GO:** Only explicitly accepted noncritical deviations remain with time-bound owners and compensating controls.

**NO-GO:** Potential dual writers, unavailable encryption keys, inconsistent recovery data, missing governance controls, unapproved data loss or no critical monitoring.

## 43. Review questions

1. Why does cross-region replication not automatically guarantee consistency?
2. What is the difference between active-passive and active-active writes?
3. How do you prove the primary writer is fenced?
4. Why does RTO end at consumer acceptance rather than workspace startup?
5. Which metadata and identity dependencies complicate region recovery?
6. What risks arise from checkpoint reuse after failover?
7. Why must failback be a separate approved operation?
8. How do you verify monitoring remains available during regional loss?

## 44. Official references

- Databricks disaster recovery: https://docs.databricks.com/aws/en/security/disaster-recovery
- Databricks Delta history: https://docs.databricks.com/aws/en/delta/history
- Databricks Terraform: https://docs.databricks.com/aws/en/dev-tools/terraform/
- Databricks Bundles: https://docs.databricks.com/aws/en/dev-tools/bundles/
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- AWS S3 replication: https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html
- Azure Storage redundancy: https://learn.microsoft.com/azure/storage/common/storage-redundancy
- Google Cloud Storage replication: https://cloud.google.com/storage/docs/using-bucket-replication
- Terraform state: https://developer.hashicorp.com/terraform/language/state

**Next:** Chapter 057 — Databricks Resilience Engineering and Chaos Testing.
