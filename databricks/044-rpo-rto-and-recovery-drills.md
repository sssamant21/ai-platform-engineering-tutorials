# Chapter 044 — RPO, RTO and Recovery Drills

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 09 — Reliability and Recovery  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform Engineering, Security, Data Owners and Incident Command  
**Prerequisites:** Chapters 041–043  
**Exercise policy:** Execute fault injection and restore tests only against approved disposable staging resources. Production evidence collection is read-only. Do not promote DR writers, change DNS, delete data or run Terraform apply without approved change control.

## 1. Mission and learning outcomes

Turn recovery architecture into measured, repeatable disaster recovery capability. By the end, you will:

- Define business-approved RTO and RPO per data product.
- Measure actual recovery from an agreed incident start and validated recovery point.
- Separate backup age, replication lag, detection delay and restore duration.
- Plan tabletop, component, workspace and regional drills.
- Validate Delta transaction consistency, Unity Catalog access and consumer freshness.
- Identify blockers that prevent meeting recovery objectives.
- Document an auditable recovery exercise and remediation plan.

## 2. The recovery contract

| Concept | Meaning | Example evidence |
|---|---|---|
| RTO | Maximum acceptable consumer-visible recovery duration | First healthy validated consumer operation |
| RPO | Maximum acceptable loss or staleness at recovery | Last recovered consistent transaction |
| MTTD | Time from incident start to detection | Alert timestamps |
| MTTA | Time from detection to acknowledgement | Incident timeline |
| MTTR | Defined recovery duration metric; specify start/end | Incident record |
| Recovery point | Transaction-consistent data state selected for restoration | Delta history and backup manifest |
| Failover | Activate recovery environment | Approval, writer fencing and routing |
| Failback | Return service to primary safely | Reconciliation and cutback tests |

Never claim RTO success based only on workspace API health or successful cluster startup. The consumer-visible service and correctness must be restored.

## 3. Recovery timeline architecture

    Last consistent protected commit
                  |
           Incident begins
                  |
         Detection and paging
                  |
       Triage and DR decision
                  |
       Source writer fencing
                  |
      Infrastructure recovery
                  |
     Storage/catalog/job restore
                  |
        Replay and validation
                  |
        Consumer acceptance
                  |
         Recovery declared

The chosen RPO measurement reference must be documented. Distinguish data committed before incident start from data arriving while the service is unavailable.

## 4. Sample business requirements

The following are synthetic exercise targets, not Databricks guarantees.

| Service | Target RTO | Target RPO | Consumer validation |
|---|---|---|---|
| demo-critical-ingest | 60 minutes | 15 minutes | No duplicate event IDs |
| demo-hourly-refresh | 120 minutes | 60 minutes | Fresh validated report |
| demo-batch-archive | 24 hours | 24 hours | Full file inventory |

Actual targets require business owner and risk approval.

## 5. RTO decomposition

Break recovery into:

1. Detection and acknowledgement.
2. Diagnosis and failover decision.
3. Writer fencing and data protection.
4. Cloud identity/network/workspace restoration.
5. Storage and governance restoration.
6. Compute provisioning and workflow reactivation.
7. Data replay/reconciliation.
8. Validation and consumer acceptance.

These phases may overlap. For critical-path calculations, do not simply sum overlapping durations.

## 6. RPO risk model

RPO depends on:
- Source commit frequency and checkpointing.
- Backup/replication consistency and lag.
- Retention and corruption detection delay.
- Availability of Delta logs and underlying files.
- Ability to replay upstream events.
- Key and identity availability.
- Recovery-point validation.

A copy that is five minutes old is not necessarily a valid five-minute RPO if its transaction log or required objects are inconsistent.

## 7. Lab A — Windows PowerShell preflight

~~~powershell
terraform version
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Verify workspace host, principal, cloud and region. Do not use production profiles for recovery drills.

## 8. Lab B — Calculate observed RTO and potential RPO

~~~powershell
$incidentStart = [datetimeoffset]"2026-10-14T10:00:00Z"
$detected = [datetimeoffset]"2026-10-14T10:07:00Z"
$lastValidCommit = [datetimeoffset]"2026-10-14T09:51:00Z"
$consumerRecovered = [datetimeoffset]"2026-10-14T10:53:00Z"

$rtoTargetMinutes = 60
$rpoTargetMinutes = 15
$observedRto = ($consumerRecovered - $incidentStart).TotalMinutes
$potentialGap = ($incidentStart - $lastValidCommit).TotalMinutes

[pscustomobject]@{
  DetectionMinutes = ($detected - $incidentStart).TotalMinutes
  ObservedRTOMinutes = $observedRto
  PotentialRPOGapMinutes = $potentialGap
  RTOPass = $observedRto -le $rtoTargetMinutes
  RPOTimeGapWithinTarget = $potentialGap -le $rpoTargetMinutes
}
~~~

The example yields 7-minute detection, 53-minute RTO and 9-minute potential gap. The RPO time-gap check is provisional until recovered records and transactions are validated.

## 9. Lab C — Model recovery critical path

~~~powershell
$phases = @(
  [pscustomobject]@{Phase="Detect";Start=0;End=7},
  [pscustomobject]@{Phase="Triage";Start=7;End=15},
  [pscustomobject]@{Phase="Fence writers";Start=15;End=20},
  [pscustomobject]@{Phase="Network and identity";Start=20;End=31},
  [pscustomobject]@{Phase="Restore and replay";Start=28;End=45},
  [pscustomobject]@{Phase="Validate consumers";Start=45;End=53}
)
$phases | Select-Object Phase,Start,End,@{Name="Duration";Expression={$_.End-$_.Start}} |
  Format-Table -AutoSize
"Elapsed minutes: $((($phases | Measure-Object End -Maximum).Maximum) - (($phases | Measure-Object Start -Minimum).Minimum))"
~~~

Network recovery and restore overlap from minutes 28–31; adding phase durations would overstate elapsed time.

## 10. Lab D — Databricks workflow inventory

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks jobs get-run --help
~~~

Identify run state, schedule, task graph, compute dependencies, retry policy and downstream consumers. Some responses are paginated or permission-filtered.

## 11. Lab E — Delta recovery-point evidence

~~~sql
DESCRIBE DETAIL demo_catalog.demo_staging.events;
DESCRIBE HISTORY demo_catalog.demo_staging.events;
~~~

Use an authorized staging Delta table. Record transaction versions, operation timestamps, source ingestion checkpoints and expected row counts. Delta history is not an independent backup.

## 12. Lab F — Read-only historical consistency test

~~~sql
SELECT COUNT(*) AS recovered_version_count
FROM demo_catalog.demo_staging.events VERSION AS OF 0;
~~~

Version 0 is illustrative. Select a retained valid version from history; old versions may not be readable after retention/VACUUM. Do not run RESTORE against production for this exercise.

## 13. Lab G — Data quality invariants

For an approved synthetic dataset:

~~~sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(DISTINCT event_id) AS unique_events,
  MIN(event_time) AS earliest_event,
  MAX(event_time) AS latest_event
FROM demo_catalog.demo_staging.events;
~~~

Replace columns and table with the actual synthetic schema. Add null checks, domain-specific totals and deterministic source-to-target reconciliation. Row counts alone do not prove correctness.

## 14. Lab H — Telemetry discovery

~~~sql
SHOW SCHEMAS IN system;
SHOW TABLES IN system.access;
SHOW TABLES IN system.compute;
SHOW TABLES IN system.lakeflow;
~~~

Where supported and authorized:

~~~sql
DESCRIBE TABLE system.access.audit;
DESCRIBE TABLE system.lakeflow.job_run_timeline;
~~~

System table availability, schemas, retention and ingestion delay vary. Cross-check with provider logs and Jobs API.

## 15. Lab I — Terraform recovery plan

In a dedicated staging DR working directory:

~~~powershell
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
~~~

For an existing approved DR backend:

~~~powershell
terraform init
terraform workspace show
terraform plan -out=dr-exercise.tfplan
terraform show -no-color dr-exercise.tfplan
~~~

Inspect region, account, workspace, state backend and resource changes. Plan files can contain sensitive values. Do not commit them or run apply without approval.

## 16. AWS readiness checks

~~~powershell
aws sts get-caller-identity
aws configure get region
aws service-quotas list-service-quotas --service-code ec2 --max-results 20
aws ec2 describe-vpc-endpoints --max-results 20
~~~

Check S3 replication and recoverable object versions using authorized read-only inventory, IAM/KMS dependencies, VPC routing, EC2 capacity and CloudWatch alarms. Quota does not guarantee available capacity during a regional outage.

## 17. Azure readiness checks

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus -o table
az network private-endpoint list --resource-group demo-dr-rg -o table
~~~

Validate ADLS Gen2 protection, managed identity, Key Vault, VNet, private DNS, Azure Monitor and target-region capacity. Replace placeholder names.

## 18. GCP readiness checks

~~~powershell
gcloud config get-value project
gcloud compute regions describe us-central1 --format=json
gcloud compute routes list
gcloud iam service-accounts list
~~~

Validate Cloud Storage protection, service accounts, KMS, Cloud DNS, networking, Cloud Monitoring and regional quotas. Use the authorized project.

## 19. Drill types and progression

| Drill | Production impact | Purpose |
|---|---|---|
| Tabletop | None | Validate decisions and ownership |
| Backup discovery | Read-only | Prove recovery point exists |
| Isolated restore | Staging only | Prove data and metadata recovery |
| Workspace rebuild | Staging only | Validate IaC and configuration |
| Dependency failure simulation | Staging/tabletop | Test detection and blast radius |
| Regional failover simulation | Staging | Test full recovery sequence |
| Failback exercise | Staging | Test reconciliation and safe cutback |

Progress from tabletop to isolated component tests before any approved production resilience exercise.

## 20. Drill charter

Every drill requires:
- Business service and data owner.
- Scenario and failure domain.
- Approved RTO/RPO.
- Staging region, workspace and synthetic dataset.
- Start/end time and cost budget.
- Incident commander and recovery roles.
- Source writer fencing strategy.
- Last validated recovery point.
- Recovery steps and abort triggers.
- Correctness and security checks.
- Evidence and reporting location.
- Cleanup and rollback plan.

## 21. Drill execution timeline

| UTC time | Action | Owner | Evidence |
|---|---|---|---|
| | Start drill clock | Incident commander | Record |
| | Detect simulated failure | SRE | Alert |
| | Acknowledge | On-call | Page |
| | Decide recovery path | Incident commander | Decision log |
| | Fence source writers | Workflow owner | Schedule state |
| | Validate identity/network | Platform | Probe |
| | Restore consistent data | Data platform | Backup manifest |
| | Restore jobs and policies | Platform | Config diff |
| | Run synthetic workflow | Workflow owner | Run ID |
| | Verify data correctness | Data owner | Invariants |
| | Accept consumer service | Business owner | SLO result |
| | Complete cleanup | Platform | Evidence |

Record actual timestamps; never backfill a timeline with estimates as facts.

## 22. Exercise A — Missed workflow deadline

**Inject/tabletop:** synthetic job completes after agreed deadline.

**Detect:** job deadline/freshness alert.

**Recover:** approved repair/replay without duplicate effects.

**Pass:** consumer output correct and measured RTO within target.

## 23. Exercise B — Lost executor or cluster

**Inject/tabletop:** staging compute becomes unavailable.

**Detect:** executor loss, cluster events and workflow status.

**Recover:** supported restart or alternate approved compute.

**Pass:** no duplicate writes and restored deadline compliance.

## 24. Exercise C — Cloud VM quota exhaustion

**Tabletop:** recovery region cannot provision requested worker family.

**Detect:** provisioning errors and quota/capacity evidence.

**Recover:** pre-approved alternate capacity or quota path.

**Pass:** measured provisioning fits RTO and budget.

## 25. Exercise D — Private endpoint/DNS outage

**Tabletop:** compute cannot reach storage privately.

**Detect:** DNS/transport/TLS and cloud flow evidence.

**Recover:** approved network configuration rollback.

**Pass:** authorized connectivity restored; prohibited traffic remains blocked.

## 26. Exercise E — Storage replica lag

**Tabletop:** recovery copy trails primary commits.

**Detect:** replication metrics and Delta consistency checks.

**Recover:** select a consistent point and replay source events.

**Pass:** actual recovered data gap within RPO.

## 27. Exercise F — Missing KMS/secret dependency

**Tabletop:** recovery environment cannot decrypt data.

**Detect:** cloud IAM/KMS error and audit event.

**Recover:** approved key and identity recovery.

**Pass:** least-privilege access and successful decryption.

## 28. Exercise G — Workspace metadata/config drift

**Inject/tabletop:** recovery workspace has missing job/policy/grant.

**Detect:** IaC diff and authorization test.

**Recover:** approved configuration reconciliation.

**Pass:** restored functional parity and governance controls.

## 29. Exercise H — Duplicate writes during failover

**Tabletop:** primary and standby writers overlap.

**Detect:** simultaneous runs and duplicate event IDs.

**Recover:** fence one writer, reconcile data and rerun idempotently.

**Pass:** one authoritative writer and verified uniqueness.

## 30. Exercise I — Recovery meets RTO but violates RPO

**Scenario:** service recovers quickly from an older consistent snapshot.

**Investigate:** missing events, checkpoint gap, backup age.

**Decision:** classify drill as failed for data protection despite green job status.

**Pass after remediation:** restored data falls within both approved objectives.

## 31. Exercise J — RPO met but consumer remains unavailable

**Scenario:** data is intact but downstream access is blocked.

**Investigate:** Unity Catalog grants, service principal, DNS and consumer endpoint.

**Decision:** RTO remains open until consumer-visible acceptance.

**Pass:** authorized consumer can read correct and fresh data.

## 32. Recovery sequencing runbook

1. Declare simulated incident and freeze unsafe automated retries.
2. Record incident start and last known good transaction.
3. Check regional/cloud/Databricks status and failure domain.
4. Decide in-place repair versus DR activation.
5. Fence authoritative writers and prevent split-brain.
6. Validate recovery cloud account, quota, DNS, IAM and KMS.
7. Restore workspace/IaC, governance and consistent storage.
8. Restore job definitions and run synthetic validation.
9. Reconcile missed data and verify business invariants.
10. Confirm consumer access, freshness, RTO and RPO.
11. Approve activation, document failback and close exercise.

## 33. Abort and rollback criteria

Stop a drill if:
- Production data or schedules could be affected.
- Source and recovery writers cannot be reliably fenced.
- Recovery point is inconsistent or unverified.
- Permissions would require unsafe broadening.
- Encryption keys are missing or compromised.
- Terraform plan includes unexpected destroys.
- Cost/time cap is exceeded.
- Monitoring cannot establish correctness.

Abort is a valid, useful drill outcome when documented accurately.

## 34. Drill evidence package

    dr-drill-YYYYMMDD/
      01-charter.md
      02-service-dependencies.md
      03-incident-timeline.csv
      04-recovery-point.md
      05-iac-plan-review.md
      06-job-and-cluster-evidence.md
      07-cloud-telemetry.md
      08-data-validation.md
      09-rto-rpo-results.md
      10-issues-and-actions.md
      11-acceptance-decision.md

Store evidence in approved private storage. Redact account IDs, storage URLs, credentials, tokens and sensitive data. Do not publish raw incident artifacts.

## 35. RTO/RPO results worksheet

| Metric | Target | Observed | Status |
|---|---|---|---|
| Detection delay | Approved | | |
| Acknowledgement delay | Approved | | |
| RTO | Approved | | |
| RPO | Approved | | |
| Dataset correctness | 100% approved invariants | | |
| Unauthorized access denied | Required | | |
| Single authoritative writer | Required | | |
| Recovery cost | Budget | | |
| Cleanup/rollback | Complete | | |

A drill passes only when both service restoration and data integrity criteria are satisfied.

## 36. Recovery drill scoring

- **Pass:** objectives achieved with complete evidence, correct data and approved security posture.
- **Conditional pass:** objectives achieved, only noncritical documentation or observability gaps remain with owners.
- **Fail:** RTO/RPO missed, integrity unverified, unsafe writers, unauthorized access, or critical dependency absent.
- **Aborted:** safety stop triggered; preserve evidence and remediation plan.

Never relabel a failed drill as passed because infrastructure was eventually restored.

## 37. Post-drill review template

    Drill ID:
    Date/time UTC:
    Service and cloud/region:
    Scenario:
    Incident commander:
    Business owner:
    Target RTO/RPO:
    Observed RTO/RPO:
    Last consistent recovery point:
    Source writer fencing evidence:
    Infrastructure and storage findings:
    Data correctness results:
    Security/access validation:
    Consumer acceptance:
    Costs and resource cleanup:
    Unexpected dependencies:
    Detection and runbook gaps:
    Corrective actions / owners / due dates:
    Next drill:
    Final GO / CONDITIONAL GO / NO-GO:

## 38. Monitoring and drill cadence

Track:
- Backup age and verified recoverability.
- Replication lag and consistent recovery-point age.
- Recovery workspace IaC drift.
- Cloud quota and capacity risk.
- IAM/KMS/secret availability.
- Job schedule parity and writer ownership.
- Data freshness and correctness.
- Recovery time by phase.
- Open DR remediation actions.

Set drill frequency by service criticality, business risk and change rate; do not assume one cadence fits every workload.

## 39. Acceptance checklist

- [ ] Business RTO/RPO defined and approved.
- [ ] Failure domain and drill scope documented.
- [ ] Production isolation and cost cap verified.
- [ ] Backup/recovery point consistency proven.
- [ ] Cloud identity, network and capacity checked.
- [ ] Terraform plan reviewed without unapproved apply.
- [ ] Writer fencing and replay strategy tested.
- [ ] Recovery timeline captured in UTC.
- [ ] Data correctness and consumer freshness validated.
- [ ] Actual RTO and RPO calculated.
- [ ] Ten exercise scenarios reviewed.
- [ ] Abort/rollback and failback procedures documented.
- [ ] Owners and remediation deadlines assigned.

## 40. Review questions

1. Why does a successful cluster restart not establish RTO compliance?
2. What is the difference between backup age and actual RPO?
3. Why must recovery time include consumer acceptance?
4. How can a recovery drill pass RTO but fail RPO?
5. Why is a tested writer-fencing mechanism essential during regional failover?

## 41. Official references

- Databricks disaster recovery: https://docs.databricks.com/aws/en/security/disaster-recovery
- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks Delta Lake: https://docs.databricks.com/aws/en/delta/
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Databricks Terraform provider: https://registry.terraform.io/providers/databricks/databricks/latest/docs
- AWS Well-Architected Reliability: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html
- Azure Well-Architected Reliability: https://learn.microsoft.com/azure/well-architected/reliability/
- Google Cloud reliability: https://cloud.google.com/architecture/framework/reliability
- Google SRE workbook: https://sre.google/workbook/

**Next:** Chapter 045 — Regional Outage Simulation Lab (Part 09 capstone).
