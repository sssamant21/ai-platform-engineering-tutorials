# Chapter 015 — Compute Operations and Failure Recovery Lab

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 03 — Compute Infrastructure and Operations (capstone)  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, Infrastructure, Databricks Administrators, FinOps  
**Prerequisites:** Chapters 011–014  
**Risk:** Read-only production diagnostics; approved disposable staging tests only

## 1. Mission and outcomes

Demonstrate the ability to diagnose and recover Databricks compute incidents without increasing the blast radius. This capstone combines compute type selection, lifecycle, policy governance, autoscaling, instance pools, performance, monitoring and cost controls.

At completion, produce:
- A timestamped inventory of critical compute and dependencies.
- An incident evidence bundle with access-controlled storage.
- Four completed fault-injection/tabletop scenarios.
- Recovery decisions, rollback criteria and validation results.
- An SRE acceptance scorecard and go/no-go recommendation.

## 2. Safety and prerequisites

1. Identify the authorized workspace, cloud, region, environment and on-call owner.
2. Verify incident/change ticket and approval for any synthetic test.
3. Never restart, resize, terminate or change production compute during the lab.
4. Run workload tests only on explicitly approved disposable staging resources.
5. Use synthetic data, bounded workloads and preapproved cloud spend.
6. Store raw IDs, hostnames, configuration, logs and account data privately.
7. Check CLI version, help, permissions and pagination before assuming complete results.

### RACI

| Action | Responsible | Approver |
|---|---|---|
| Evidence collection | SRE | Workspace owner |
| Policy/runtime changes | Platform admin | Change owner |
| Cloud quota/capacity | Cloud infrastructure | Cloud owner |
| Security/access exceptions | Security | Security owner |
| Cost optimization | FinOps/platform | Workload owner |
| Production rollback | Incident commander | Change/incident authority |

## 3. Compute architecture and failure domains

~~~text
                Jobs / Notebooks / SQL
                         |
                  Workspace control
                         |
        +----------------+-----------------+
        |                |                 |
   Classic clusters  SQL warehouses    Serverless
        |                |                 |
  Policies/access   Warehouse scaling  Managed capacity
  Runtime/node      Query queue        Supported connectivity
  Pools/autoscale   Concurrency        Account quotas
        |
   Cloud VM capacity
   IAM / networking
   Storage / external data
        |
   Metrics / events / billing
~~~

A healthy control plane does not prove that classic compute can provision. A running cluster does not prove that Spark workloads meet SLOs. SQL warehouses and serverless must be evaluated using their own capacity and telemetry models.

## 4. Baseline evidence requirements

| Evidence | Required fields |
|---|---|
| Workspace | Cloud, region, environment, UTC timestamp, owner |
| Compute | Type, state, runtime, access mode, node family, worker range |
| Governance | Policy ID (private), allowed node/runtime, permissions |
| Lifecycle | Startup duration, termination reasons, cluster events |
| Workload | Run duration, retries, queue time, success rate |
| Performance | Driver/executor, spill, skew, storage latency |
| Capacity | Cloud quota, available capacity, pool used/idle/pending |
| Cost | Databricks usage and cloud infrastructure costs |
| Recovery | Change owner, action, rollback, verification |

## 5. Lab A — PowerShell preflight and workspace verification

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json

$root = Join-Path $env:USERPROFILE "databricks-sre-ch015"
New-Item -ItemType Directory -Force -Path $root | Out-Null
$utc = (Get-Date).ToUniversalTime().ToString("o")
Write-Host "Assessment UTC: $utc"
~~~

Replace the profile with your approved staging workspace. Confirm that the displayed host is correct. Do not export tokens or authentication configuration.

## 6. Lab B — Compute and policy inventory

~~~powershell
databricks clusters list --output json |
  Out-File (Join-Path $root "clusters.json") -Encoding utf8

databricks cluster-policies list --output json |
  Out-File (Join-Path $root "policies.json") -Encoding utf8

databricks instance-pools list --output json |
  Out-File (Join-Path $root "pools.json") -Encoding utf8

databricks clusters list-node-types --output json |
  Out-File (Join-Path $root "node-types.json") -Encoding utf8

databricks clusters spark-versions --output json |
  Out-File (Join-Path $root "runtimes.json") -Encoding utf8
~~~

Confirm permissions and pagination. A missing resource may indicate restricted visibility rather than absence.

### Jobs and SQL inventory

~~~powershell
databricks jobs list --output json |
  Out-File (Join-Path $root "jobs.json") -Encoding utf8

databricks jobs list-runs --limit 25 --output json |
  Out-File (Join-Path $root "recent-runs-sample.json") -Encoding utf8

databricks warehouses list --output json |
  Out-File (Join-Path $root "warehouses.json") -Encoding utf8
~~~

The job-run result is a bounded sample, not full history. Serverless usage and SQL workloads may require additional supported APIs and system tables.

### Investigate one authorized cluster

~~~powershell
databricks clusters get --help
databricks clusters events --help
databricks jobs get-run --help
databricks instance-pools get --help
~~~

Follow installed CLI argument syntax. Capture cluster state transitions, runtime, policy, node type, autoscaling bounds, pool references and recent errors.

## 7. Lab C — Cloud capacity verification

Run only the applicable cloud section after confirming the active account and region.

### AWS

~~~powershell
aws sts get-caller-identity
aws configure get region
aws service-quotas list-service-quotas --service-code ec2 --max-items 20
~~~

### Azure

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus --output table
~~~

Replace eastus with the workspace region.

### GCP

~~~powershell
gcloud config get-value project
gcloud compute regions describe us-central1 --format="json(quotas)"
~~~

Replace us-central1 with the workspace region.

Quotas, permissions, regional instance availability and subnet capacity are different constraints. A configured maximum worker count is not guaranteed capacity.

## 8. Lab D — Baseline SLIs

Define at least five SLIs with owner, data source, measurement window and target:

| SLI | Source | Example objective |
|---|---|---|
| Critical job completion | Job run history | Meets agreed deadline |
| Cluster startup success | Cluster events | Baseline-dependent |
| Cluster startup p95 | Event timestamps | Baseline-dependent |
| Job duration p95 | Run history | Workload-specific |
| SQL queue time | Query history | Workload-specific |
| Idle compute spend | Usage + cloud billing | Budget-specific |
| Capacity allocation failures | Events + cloud logs | Alert on impact |

Do not choose arbitrary universal SLO thresholds.

## 9. Lab E — Usage and cost baseline

Where Databricks system tables are enabled and authorized:

~~~sql
SELECT
  usage_date,
  workspace_id,
  sku_name,
  usage_unit,
  SUM(usage_quantity) AS total_usage
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 14)
GROUP BY usage_date, workspace_id, sku_name, usage_unit
ORDER BY usage_date DESC, workspace_id, sku_name;
~~~

Usage units and SKUs differ. This query does **not** capture the entire cloud bill. Join conclusions with cloud compute, storage, NAT, egress and monitoring costs using approved billing exports.

## 10. Scenario 1 — Cluster cannot start

**Inject (staging or tabletop):** A critical job requests classic compute, but cluster remains PENDING or enters ERROR.

### Triage
1. Confirm impacted job, workspace, cloud, region and UTC start time.
2. Inspect cluster events and termination reason.
3. Compare requested node type/runtime with policy.
4. Inspect cloud quotas and capacity availability.
5. Review subnet IP availability, egress, identity and storage prerequisites.
6. Check init scripts, image/runtime compatibility and recent changes.

### Decision tree

~~~text
Cluster startup failure
       |
Policy validation rejected?
  yes -> correct approved configuration
  no
       |
Cloud capacity / quota error?
  yes -> approved alternate family or quota remediation
  no
       |
Network / identity / storage failure?
  yes -> repair specific dependency
  no
       |
Runtime / init script failure?
  yes -> revert compatible configuration
  no -> escalate with sanitized events
~~~

**Recovery:** use a previously validated configuration or approved capacity fallback.

**Rollback:** restore original policy/runtime/node configuration where supported; account for compute restart.

**Acceptance:** synthetic job starts and completes; startup error rate recovers.

## 11. Scenario 2 — Autoscaling or pool exhaustion

**Inject:** Worker allocation stalls below the configured max, and multiple jobs queue.

### Triage
- Pool used/idle/pending instances and shared consumers.
- Worker min/max, scaling events and actual workload demand.
- Cloud quota, instance family and zonal availability.
- Driver versus worker bottlenecks.
- Network/subnet constraints and spot interruptions.

**Recovery:** isolate critical workloads, select an approved alternative node family or adjust pool settings after impact review.

**Rollback:** restore previous pool/compute assignments and capacity settings.

**Acceptance:** bounded staging workload scales as expected, and other workloads are not starved.

## 12. Scenario 3 — RUNNING compute, severe slowness

**Inject:** Job duration doubles, although cluster state is RUNNING.

### Triage
1. Compare input volume, job definition and runtime to baseline.
2. Inspect Spark stage/task duration distribution.
3. Check driver CPU, executor loss, GC, memory and disk spill.
4. Correlate object storage latency and network throughput.
5. Inspect scaling events and shared workload concurrency.
6. Compare SQL queueing versus execution if the affected workload is SQL.

**Recovery:** target the demonstrated bottleneck; avoid reflexively increasing workers.

**Rollback:** restore last known-good runtime, node family or concurrency settings through change control.

**Acceptance:** latency and reliability recover with acceptable cost.

## 13. Scenario 4 — Compute cost anomaly

**Inject:** Daily infrastructure cost rises by 40% without a comparable workload increase.

### Triage
- Check all-purpose clusters, idle instance pools and auto-termination.
- Review SQL warehouse usage and serverless SKUs separately.
- Compare VM family, worker count, runtime and job concurrency changes.
- Check cloud network egress, NAT and storage charges.
- Attribute spend to workspace, owner and cost center.

**Recovery:** owner-approved idle-resource cleanup, rightsizing or policy correction.

**Rollback:** restore prior settings if startup SLOs or job reliability regress.

**Acceptance:** cost returns toward baseline without harming critical workload SLIs.

## 14. Scenario 5 — Workload isolation incident

**Inject:** One team's batch job saturates shared compute and degrades another team's critical job.

### Triage
- Confirm shared versus dedicated compute and access mode.
- Identify concurrent workloads and ownership.
- Inspect CPU/memory, scheduling and driver contention.
- Verify compute permissions and data access boundaries.
- Check whether shared instance pools are causing cross-team starvation.

**Recovery:** approved workload isolation, dedicated jobs compute, separate SQL warehouse or supported serverless mode.

**Acceptance:** critical workload SLOs and security boundaries remain intact.

## 15. Incident timeline template

~~~text
Incident ID:
Workspace / cloud / region:
Incident commander:
Start UTC:
Detection UTC:
User impact:
Affected compute and jobs:
First failed signal:
Relevant recent change:
Evidence collected:
Hypotheses tested:
Mitigation approved by:
Mitigation time UTC:
Recovery validation:
Rollback decision:
Residual risk:
RCA/COA owner:
~~~

Keep incident artifacts in approved restricted storage; public tutorial repos must not contain production IDs, hostnames, logs or secrets.

## 16. Observability acceptance

A production-ready compute dashboard should show:
- Startup latency and provisioning failures.
- Cluster lifecycle and termination reasons.
- Job success, queue time, retries and duration.
- SQL warehouse queue/execution latency.
- Driver/executor saturation and loss.
- Autoscaling/pool allocation and cloud quota signals.
- Databricks usage by SKU and cloud cost by workload.
- Runtime/policy drift and owner attribution.

Every alert must have an owner, runbook, severity mapping, escalation path and test evidence.

## 17. Production recovery runbook

1. **Detect:** verify user impact and SLI breach.
2. **Scope:** cloud, workspace, compute type, region and dependent jobs.
3. **Stabilize:** avoid simultaneous uncoordinated policy, quota and runtime changes.
4. **Preserve:** collect UTC timeline, sanitized events, metrics and config.
5. **Classify:** policy, capacity, lifecycle, performance, isolation or cost.
6. **Mitigate:** smallest approved change with known rollback.
7. **Validate:** synthetic test plus real workload and cost signals.
8. **Communicate:** impact, status, next update and owners.
9. **Close:** RCA, corrective actions, runbook and monitoring updates.

## 18. Recovery and rollback controls

Before any production action, document:
- The exact previous configuration.
- Which workloads depend on the resource.
- Required approval and maintenance window.
- Potential restarts or interruptions.
- A bounded validation plan.
- Conditions that trigger rollback.
- Post-rollback verification and cleanup.

A policy rollback alone may not revert already-running compute. Similarly, a cloud quota increase cannot fix data skew or broken identity permissions.

## 19. Capstone acceptance scorecard

Use **Pass**, **Partial**, **Fail**, **Not assessed**.

| Control | Status | Evidence and owner |
|---|---|---|
| Workspace and compute types inventoried | Not assessed | |
| Policy and runtime governance reviewed | Not assessed | |
| Cloud quota and capacity dependencies checked | Not assessed | |
| Autoscaling/pool behavior assessed | Not assessed | |
| Critical job and SQL SLIs established | Not assessed | |
| Performance triage demonstrated | Not assessed | |
| Databricks and cloud costs reconciled | Not assessed | |
| Five failure scenarios completed | Not assessed | |
| Alerting and runbooks validated | Not assessed | |
| Rollback and change controls reviewed | Not assessed | |
| Security/isolation boundaries reviewed | Not assessed | |
| Evidence sanitized and stored securely | Not assessed | |

### Go/no-go decision

**Go:** all critical compute paths have evidence, recovery is tested, and no unresolved critical risks remain.

**Conditional go:** only noncritical gaps remain, with owner, due date and documented risk acceptance.

**No-go:** critical workload cannot start or recover, access boundaries are violated, or required operational evidence is absent.

## 20. Capstone sign-off template

~~~text
Assessment: Databricks Compute Operations and Failure Recovery
Workspace label:
Cloud / region:
Environment:
Assessment UTC:
Assessor:
Platform owner:
Security reviewer:
FinOps reviewer:
Change ticket:

Critical workloads covered:
Compute types covered:
Policy/runtime baseline:
Cloud quota baseline:
Performance baseline:
Cost baseline:

Scenario 1 — startup:
Scenario 2 — scaling/pool:
Scenario 3 — slowness:
Scenario 4 — cost:
Scenario 5 — isolation:

Outstanding findings:
Remediation owner and due date:
Decision: Go / Conditional go / No-go
Approval:
~~~

## 21. Cleanup

- Terminate only approved disposable staging clusters and test jobs.
- Confirm temporary instance pools are no longer consuming idle capacity.
- Remove synthetic data from designated test paths.
- Retain evidence under policy; do not publish raw inventories.
- Verify no production resources were changed.

## 22. Review questions

1. Why does a RUNNING cluster not guarantee workload health?
2. How would you distinguish quota shortage from pool exhaustion?
3. Why can a policy rollback require compute replacement?
4. How do Databricks service charges differ from cloud infrastructure spend?
5. What evidence supports a safe go/no-go decision after recovery?

## 23. References

- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks compute policies: https://docs.databricks.com/aws/en/admin/clusters/policy-definition
- Databricks jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Azure Databricks compute: https://learn.microsoft.com/azure/databricks/compute/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Part 03 complete. Next:** Chapter 016 — Identity, Authentication and Access Control.
