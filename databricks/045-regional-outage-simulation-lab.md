# Chapter 045 — Regional Outage Simulation Lab

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 09 — Reliability and Recovery (capstone)  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Cloud Platform, Security, Networking, Data Governance and Incident Command  
**Prerequisites:** Chapters 041–044; recommended Chapters 006–010, 016–020, 026–030  
**Exercise policy:** Tabletop and isolated staging only. Never disable production infrastructure, alter production DNS, invoke regional failover, destroy resources, or activate duplicate writers without approved change control.

## 1. Mission

Prove that a Databricks data service can recover from a regional outage, not merely that a replacement workspace can be provisioned. Execute a controlled simulation covering infrastructure, identity, private networking, Unity Catalog, object storage, Delta transaction consistency, workflows, observability and downstream consumer acceptance.

### Learning outcomes

- Establish an incident command structure and objective-based recovery decision.
- Identify region-scoped and shared failure domains.
- Validate independent recovery-region dependencies.
- Restore infrastructure from reviewed Terraform plans.
- Select a consistent recovery point and safely reconcile data.
- Fence source writers and prevent split-brain.
- Measure actual RTO and RPO.
- Validate failover, failback and consumer correctness.
- Produce a complete capstone evidence package and improvement backlog.

## 2. Scenario

A synthetic hourly data service named demo-hourly-refresh runs in Region A. At 10:00 UTC, the primary region becomes unavailable for compute provisioning and storage connectivity. The organization has a prepared Region B staging recovery environment. The incident team must activate Region B safely and restore validated downstream data.

**Synthetic targets:**

| Item | Exercise value |
|---|---|
| Incident start | 10:00 UTC |
| Target RTO | 90 minutes |
| Target RPO | 15 minutes |
| Last validated primary commit | 09:56 UTC |
| Last independently recoverable consistent point | 09:51 UTC |
| Expected consumer | demo-reporting-consumer |
| Production blast radius | None; staging/tabletop only |
| Recovery strategy | Controlled active/passive |
| Success | Correct and fresh consumer output, one writer |

These values are examples, not Databricks service guarantees.

## 3. Regional failure architecture

    Region A (simulated unavailable)
    +------------------------------+
    | Databricks workspace         |
    | Compute + Jobs               |
    | Unity Catalog dependencies   |
    | Object storage / Delta       |
    +------------------------------+
                   |
       Protected consistent data
       + versioned configuration
                   |
                   v
    Region B (isolated staging DR)
    +------------------------------+
    | Cloud network + IAM/KMS      |
    | Databricks recovery workspace|
    | Governance + storage access  |
    | Controlled compute + Jobs    |
    | Validated consumer           |
    +------------------------------+

The diagram is conceptual. Actual metastore and storage regional support, workspace networking and control-plane dependencies must be verified for the selected cloud and deployment.

## 4. Incident roles

| Role | Responsibility |
|---|---|
| Incident commander | Severity, decisions, communications |
| Recovery lead | Executes approved DR sequence |
| Cloud platform | Quota, VPC/VNet/VPC, Terraform |
| Network engineer | DNS, routes, private endpoints |
| Security/IAM | Identity, grants, KMS and secrets |
| Data platform | Delta consistency and replay |
| Workflow owner | Jobs, schedules and writer fencing |
| SRE observer | UTC timeline, metrics and evidence |
| Business/data owner | Correctness and recovery acceptance |

Separate approval authority from execution where required.

## 5. Preconditions and abort triggers

Before starting:
- Confirm staging isolation and disposable synthetic data.
- Verify Region B workspace and cloud resources are authorized.
- Confirm backups and recovery-point evidence.
- Record active writer ownership and fencing mechanism.
- Verify IaC remote state/backend and read-only plan access.
- Agree on cost/time limits and cleanup.
- Establish emergency stop and rollback contacts.
- Ensure audit logging and consumer validation are functional.

**Abort immediately** if production impact is possible, recovery data is inconsistent, source writers cannot be fenced, credentials are unavailable, unexpected Terraform destruction appears, or cost cap is exceeded.

## 6. Recovery dependency order

1. Incident command and scope.
2. Writer fencing and data protection.
3. Recovery-region cloud identity, quotas and keys.
4. Network, private DNS, routes and endpoints.
5. Workspace and governance configuration.
6. Consistent object storage/Delta recovery point.
7. Compute policies and job definitions.
8. Controlled replay and single-writer activation.
9. Data quality, consumer freshness and security validation.
10. Failback planning, cleanup and RCA.

Some independent tasks may run in parallel. Track critical-path elapsed time rather than summing overlapping durations.

## 7. Lab A — Windows PowerShell tool preflight

~~~powershell
terraform version
databricks version
databricks auth profiles
aws --version
az version
gcloud version
~~~

Only the selected cloud CLI is required. Do not use production credentials for the exercise.

## 8. Lab B — Validate recovery workspace identity

~~~powershell
$env:DATABRICKS_CONFIG_PROFILE = "demo-dr-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks clusters list --output json
~~~

Confirm actual host, region, principal and scope. The profile name alone is not evidence of isolation.

## 9. Lab C — Review Terraform configuration

From an existing, approved recovery IaC working directory:

~~~powershell
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
~~~

For a separately configured and authorized DR state backend:

~~~powershell
terraform init
terraform workspace show
terraform plan -out=regional-dr-review.tfplan
terraform show -no-color regional-dr-review.tfplan
~~~

**Do not run terraform apply.** Confirm the plan has no unexpected destroys/replacements. State and plan files may contain sensitive values; do not commit or share publicly.

## 10. Lab D — AWS Region B readiness

~~~powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-vpc-endpoints --max-results 20
aws service-quotas list-service-quotas --service-code ec2 --max-results 20
~~~

Review EC2 capacity, VPC routes, IAM trust, S3 recovery objects, KMS and CloudWatch. Verify that recovery region and storage replication are actually configured.

## 11. Lab E — Azure Region B readiness

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus -o table
az network private-endpoint list --resource-group demo-dr-rg -o table
~~~

Review VNet injection where applicable, private DNS, ADLS Gen2 recovery, managed identity, Key Vault, regional quota and Azure Monitor. Replace placeholder region and resource group.

## 12. Lab F — GCP Region B readiness

~~~powershell
gcloud config get-value project
gcloud compute regions describe us-central1 --format=json
gcloud compute routes list
gcloud iam service-accounts list
~~~

Review VPC, private access, Cloud Storage recovery location, service accounts, KMS, quota and Cloud Monitoring. Confirm Databricks regional support.

## 13. Lab G — Databricks governance inventory

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
~~~

Record metastore/workspace assignment, storage credentials, external locations, table and volume ownership and privileges. Do not assume catalog metadata can be recovered by copying storage files alone.

## 14. Lab H — Recovery-point inspection

~~~sql
DESCRIBE DETAIL demo_catalog.demo_staging.events;
DESCRIBE HISTORY demo_catalog.demo_staging.events;
~~~

Select a verified, recoverable Delta transaction snapshot. Ensure its transaction log and data files are available in Region B. Time travel is not an independent backup.

## 15. Lab I — Historical snapshot check

~~~sql
SELECT COUNT(*) AS historical_rows
FROM demo_catalog.demo_staging.events VERSION AS OF 0;
~~~

Version 0 is illustrative; choose an actually retained and readable version from history. This is a read-only query, not a restore. Never assume the latest replicated files form a consistent Delta snapshot.

## 16. Lab J — Synthetic consumer invariants

For an approved synthetic table with event_id and event_time:

~~~sql
SELECT
  COUNT(*) AS total_events,
  COUNT(DISTINCT event_id) AS unique_events,
  MIN(event_time) AS first_event,
  MAX(event_time) AS latest_event
FROM demo_catalog.demo_staging.events;
~~~

Compare recovered results with the source-of-truth event manifest. Add domain-specific sums, null checks, schema verification and expected checkpoint position.

## 17. Lab K — System telemetry discovery

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

System tables vary by cloud, region and account and can lag. Use provider telemetry and Jobs API as additional evidence.

## 18. Lab L — Controlled run discovery

~~~powershell
databricks jobs list-runs --limit 25 --output json
databricks jobs get-run --help
databricks clusters events --help
~~~

Inspect exact installed CLI syntax before fetching a selected run. Do not restart or repair a stateful workflow until idempotency and writer fencing are proven.

## 19. Writer fencing design

Use an approved authoritative control that prevents Region A and Region B from writing concurrently. Examples include an external lease/fencing token, a centrally governed scheduler ownership flag with strong consistency, or a supported single-writer orchestration mechanism.

Merely pausing a job schedule does not guarantee that existing runs or external writers have stopped. Confirm running tasks, credentials/leases and downstream commit ownership.

## 20. Recovery-point selection

Record:
- Last primary committed transaction before incident.
- Last independently protected transaction-consistent snapshot.
- Replication lag and any missing data files.
- Upstream source retention/replay position.
- Encryption key availability.
- Chosen recovered Delta version and validation.
- Estimated data gap and replay requirement.

A backup manifest and Delta history should agree on a recoverable consistent state.

## 21. Lab M — RTO and RPO measurement

~~~powershell
$incident = [datetimeoffset]"2026-10-14T10:00:00Z"
$recoveryPoint = [datetimeoffset]"2026-10-14T09:51:00Z"
$consumerRestored = [datetimeoffset]"2026-10-14T11:12:00Z"
$rtoTarget = 90
$rpoTarget = 15

$observedRto = ($consumerRestored-$incident).TotalMinutes
$potentialGap = ($incident-$recoveryPoint).TotalMinutes

[pscustomobject]@{
  ObservedRTOMinutes=$observedRto
  PotentialRPOGapMinutes=$potentialGap
  RTOPass=($observedRto -le $rtoTarget)
  RPOTimeGapWithinTarget=($potentialGap -le $rpoTarget)
}
~~~

Synthetic result: RTO 72 minutes, potential RPO gap 9 minutes. RPO only passes after validating actual recovered and replayed data.

## 22. Lab N — Recovery phase timeline

| UTC | Phase | Owner | Evidence |
|---|---|---|---|
| 10:00 | Simulated incident starts | Incident commander | Charter |
| 10:06 | Alert acknowledged | SRE | Alert |
| 10:15 | DR decision approved | Incident commander | Decision log |
| 10:22 | Writer fencing verified | Workflow owner | Lease/run evidence |
| 10:38 | Network/IAM ready | Cloud platform | Probe |
| 10:50 | Recovery snapshot validated | Data platform | Delta checks |
| 11:03 | Recovery workflow completes | Workflow owner | Job run |
| 11:12 | Consumer accepts output | Business owner | Validation |

This timeline is synthetic. Real drills must record actual UTC event times and identify overlapping phases.

## 23. Lab O — Critical-path and recovery cost worksheet

| Phase | Elapsed minutes | Parallelizable | Cost impact |
|---|---|---|---|
| Detection/decision | | Partly | On-call |
| Writer fencing | | No for activation | Service delay |
| Network/IAM | | Partly | Cloud resources |
| Storage restore | | Partly | Requests/transfer |
| Workspace/jobs | | Partly | Compute |
| Replay | | Depends | Compute/storage |
| Validation | | Partly | Test compute |

Estimate total cost using provider billing data. Avoid inferring DR cost solely from worker-hours.

## 24. Scenario A — Entire primary region unavailable

**Inject/tabletop:** provider regional outage affects workspace and compute.

**Detect:** independent status/health evidence.

**Recover:** activate Region B via approved procedure.

**Pass:** consumer correctness and RTO/RPO targets met.

## 25. Scenario B — Recovery-region capacity shortfall

**Tabletop:** required worker family cannot be allocated.

**Diagnose:** quota, regional availability and approved alternatives.

**Recover:** validated smaller/fallback capacity or quota escalation.

**Pass:** deadline and cost remain acceptable.

## 26. Scenario C — Storage replica lag

**Tabletop:** latest Delta commit has not reached Region B.

**Diagnose:** replication metrics, manifest and log/data consistency.

**Recover:** select older consistent snapshot and replay retained events.

**Pass:** verified recovered records within RPO.

## 27. Scenario D — Missing encryption key

**Tabletop:** replicated objects cannot be decrypted.

**Diagnose:** KMS key and principal state.

**Recover:** supported key recovery without privilege broadening.

**Pass:** authorized reads and audit visibility.

## 28. Scenario E — Private DNS misconfiguration

**Tabletop:** recovery compute cannot reach required storage endpoint.

**Diagnose:** compute-side resolver, routes, firewall and TLS.

**Recover:** approved DNS/endpoint correction.

**Pass:** authorized private connectivity works.

## 29. Scenario F — Unity Catalog permission mismatch

**Tabletop:** recovery job receives 403/permission denied.

**Diagnose:** metastore assignment, grants, storage credential and cloud IAM.

**Recover:** restore approved least-privilege grants.

**Pass:** intended access succeeds, unauthorized access fails.

## 30. Scenario G — Duplicate writers after promotion

**Tabletop:** primary comes back while standby writes.

**Diagnose:** active run state, leases, schedules and Delta history.

**Recover:** retain one authoritative writer and reconcile data.

**Pass:** no duplicate or conflicting commits.

## 31. Scenario H — Corrupt latest replica

**Tabletop:** replication copied logical corruption.

**Diagnose:** transaction timeline, known-good protected snapshots.

**Recover:** isolated consistent backup and replay.

**Pass:** business invariants and RPO verified.

## 32. Scenario I — Workspace restored, consumer still broken

**Tabletop:** job succeeds but consumer endpoint/permissions fail.

**Diagnose:** downstream DNS, identity, access and freshness.

**Recover:** restore end-to-end dependency.

**Pass:** consumer-visible SLO restored.

## 33. Scenario J — Failback causes data divergence

**Tabletop:** primary region returns with stale state.

**Diagnose:** authoritative commit position and DR-region writes.

**Recover:** reconcile primary before controlled cutback.

**Pass:** no split-brain, lost or duplicate events.

## 34. Full incident commander runbook

1. Declare simulated severity and impact.
2. Verify affected failure domain and independent evidence.
3. Capture incident start, target RTO/RPO and data owners.
4. Stop unsafe retries and fence source writers.
5. Decide repair versus Region B promotion.
6. Confirm Region B quotas, network, IAM and KMS.
7. Review Terraform plan and configuration parity.
8. Select and verify transaction-consistent recovery point.
9. Restore catalog permissions, compute and jobs.
10. Replay missed source events idempotently.
11. Validate schema, row counts, uniqueness and freshness.
12. Verify consumer access and service SLO.
13. Record measured RTO/RPO and recovery costs.
14. Prepare controlled failback and preserve rollback.
15. Publish RCA and remediation actions.

## 35. Failback procedure

- Keep Region B authoritative until Region A is validated.
- Confirm primary region stability and resource readiness.
- Replicate/reconcile all Region B commits back to Region A.
- Validate Delta consistency and downstream checkpoints.
- Stage primary jobs with schedules disabled.
- Recheck IAM, private networking, keys and monitoring.
- Fence Region B writers before Region A activation.
- Perform controlled cutback with approval.
- Validate consumer output, SLO and rollback route.
- Retain evidence and cleanup plan.

Failback is a separate recovery operation, not simply reversing DNS.

## 36. Capstone evidence package

    regional-dr-capstone/
      01-exercise-charter.md
      02-dependency-map.md
      03-recovery-point.md
      04-cloud-readiness.md
      05-terraform-plan-review.md
      06-writer-fencing.md
      07-incident-timeline.csv
      08-databricks-run-evidence.md
      09-delta-consistency.md
      10-consumer-validation.md
      11-rto-rpo-results.md
      12-cost-and-rollback.md
      13-rca-and-actions.md
      14-acceptance-decision.md

Keep operational artifacts private, access-controlled and redacted. Never publish raw credentials, signed URLs or customer data.

## 37. Capstone acceptance scorecard

| Requirement | Status | Evidence |
|---|---|---|
| Approved staging scope and isolation | Not assessed | |
| Failure domains mapped | Not assessed | |
| Incident command and timeline | Not assessed | |
| RTO/RPO defined | Not assessed | |
| Recovery cloud capacity checked | Not assessed | |
| DNS/network/IAM/KMS validated | Not assessed | |
| Terraform plan reviewed | Not assessed | |
| Writer fencing demonstrated | Not assessed | |
| Consistent recovery point verified | Not assessed | |
| Unity Catalog and jobs restored | Not assessed | |
| Data replay idempotent | Not assessed | |
| Consumer correctness/freshness proven | Not assessed | |
| Actual RTO/RPO measured | Not assessed | |
| Failback and rollback documented | Not assessed | |
| Ten scenarios reviewed | Not assessed | |
| RCA and improvement owners assigned | Not assessed | |

## 38. GO / CONDITIONAL GO / NO-GO

**GO:** Recovery service meets approved RTO and RPO; one authoritative writer; data, permissions and consumers validated; rollback and evidence complete.

**CONDITIONAL GO:** Objectives met and only noncritical gaps remain with approved owner, mitigation and deadline.

**NO-GO:** Unverified recovery point, missing keys, dual writers, unsafe Terraform changes, failed data correctness, missed critical objectives or unvalidated consumer service.

## 39. Post-exercise RCA template

    Exercise ID:
    UTC start / detection / decision / restoration:
    Affected services and failure domains:
    Primary and recovery regions:
    Target RTO / observed RTO:
    Target RPO / observed data gap:
    Recovery-point version and consistency:
    Writer-fencing evidence:
    Cloud networking / IAM / KMS findings:
    Infrastructure and workspace findings:
    Unity Catalog / storage findings:
    Job and replay findings:
    Consumer correctness/freshness:
    Cost and rollback:
    Detection gaps:
    Recovery bottlenecks:
    Preventive actions, owners and deadlines:
    Final GO / CONDITIONAL GO / NO-GO:

## 40. Review questions

1. Why does a second-region workspace not prove disaster recovery readiness?
2. How do you verify that a replicated Delta snapshot is transaction-consistent?
3. What evidence proves that primary writers were fenced?
4. Why does successful job completion not necessarily meet RTO?
5. Why must failback reconcile data written during DR operation?

## 41. Official references

- Databricks disaster recovery: https://docs.databricks.com/aws/en/security/disaster-recovery
- Databricks architecture: https://docs.databricks.com/aws/en/getting-started/high-level-architecture
- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks Delta Lake: https://docs.databricks.com/aws/en/delta/
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Databricks Terraform provider: https://registry.terraform.io/providers/databricks/databricks/latest/docs
- Terraform plan: https://developer.hashicorp.com/terraform/cli/commands/plan
- AWS Well-Architected Reliability: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html
- Azure Well-Architected Reliability: https://learn.microsoft.com/azure/well-architected/reliability/
- Google Cloud reliability: https://cloud.google.com/architecture/framework/reliability
- Google SRE workbook: https://sre.google/workbook/

**Part 09 complete. Next:** Chapter 046 — Terraform Provisioning and Drift Detection (Part 10 — Automation and GitOps).
