# Chapter 011 — Compute Types, Lifecycle and Workload Isolation

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 03 — Compute Infrastructure and Operations  
**Audience:** SRE, Platform Operations, Cloud Infrastructure  
**Clouds:** AWS, Azure, GCP  
**Lab risk:** Read-only inventory and tabletop scenarios; optional approved disposable staging test

## 1. Learning objectives

After this chapter, you should be able to:
1. Differentiate all-purpose compute, jobs compute, SQL warehouses and serverless execution.
2. Explain cluster lifecycle states, startup dependencies, termination and failure recovery.
3. Map resource isolation, access modes, cluster policies and ownership.
4. Identify idle cost, concurrency contention, misconfiguration and startup failure signals.
5. Build a compute inventory with explicit visibility gaps.
6. Diagnose operational incidents without disrupting active workloads.

## 2. Compute architecture

~~~text
Users / Jobs / SQL / APIs
          |
      Databricks
      control plane
          |
    +-----+-----------------+-----------------+
    |                       |                 |
All-purpose compute     Jobs compute     SQL warehouses
Interactive sessions    Scheduled tasks  SQL queries / BI
    |                       |                 |
    +----------+------------+                 |
               |                              |
         Classic compute                 Classic or
         cloud VM-backed                 serverless mode
               |
      Worker/driver lifecycle

Serverless jobs/notebooks and other supported serverless
workloads use a distinct Databricks-managed execution model.
~~~

**Important:** SQL warehouses are not equivalent to general-purpose Spark clusters. Serverless features, availability and configuration differ by cloud, region and workspace. Use the actual workload's compute type when investigating incidents.

## 3. Compute type comparison

| Type | Typical use | Operational considerations |
|---|---|---|
| All-purpose classic compute | Interactive notebooks, exploration | Shared sessions, idle termination, access mode, cost and contention |
| Classic jobs compute | Automated task execution | Startup latency, task lifecycle, retries, job ownership and cluster policies |
| SQL warehouse (classic/pro/serverless as supported) | SQL queries, BI and dashboards | Auto-stop, scaling, queueing, query concurrency, warehouse permissions |
| Serverless jobs / notebooks (where supported) | Managed execution | Service quotas, access controls, usage attribution, distinct network model |
| Instance pools (where supported) | Faster classic compute acquisition | Pool sizing, idle capacity cost, capacity constraints and lifecycle |

Avoid treating a single classic cluster list as a complete compute inventory. Query job definitions, SQL warehouses, serverless telemetry and billing/system usage data separately.

## 4. Classic compute lifecycle

~~~text
Configured
    |
    v
PENDING / provisioning
    |
    v
RUNNING
    |
    +--> RESIZING --> RUNNING
    |
    +--> RESTARTING --> RUNNING
    |
    +--> TERMINATING --> TERMINATED
    |
    +--> ERROR / failed startup

TERMINATED --> PENDING (approved restart)
~~~

State names and transitions depend on the relevant Databricks API and compute type. For incident triage, capture:
- requested action and timestamp;
- current and previous state;
- termination reason / event history;
- node type, runtime and policy;
- cloud capacity/quota signals;
- networking, identity and storage errors.

**Lifecycle does not equal health:** RUNNING does not prove that workloads are healthy, and a TERMINATED job cluster can be normal after task completion.

## 5. Isolation and security boundaries

Isolation is a combination of:
- Workspace boundaries and account governance.
- Compute access modes and identity behavior.
- User/group permissions and cluster policies.
- Dedicated versus shared compute.
- Cloud network and storage boundaries for classic compute.
- Data permissions through Unity Catalog and underlying cloud access.
- Job-level ownership, service principals and secret access.

Do not assume that separate jobs running on shared compute are isolated to the same degree as separate dedicated compute. Validate the supported access modes and governance model for the selected runtime and cloud.

### Isolation decision worksheet

| Question | Evidence |
|---|---|
| Who can attach to the compute? | Compute permissions |
| Which identity accesses data? | Effective principal and data grants |
| Is the compute shared or dedicated? | Access mode and job definition |
| What resources are shared? | Driver/workers, pool, warehouse or serverless limits |
| Can one workload affect another? | Concurrency and resource saturation metrics |
| What is the blast radius? | Downstream jobs, data and shared dependencies |

## 6. Lab A — Read-only compute inventory

**Prerequisites:** Authorized Databricks CLI profile, read access to compute/jobs/warehouses and a private evidence directory.

### Step 1 — Verify identity

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Replace the example profile with your approved workspace profile. Verify the host before proceeding.

### Step 2 — Prepare evidence storage

~~~powershell
$root = Join-Path $env:USERPROFILE "databricks-sre-ch011"
New-Item -ItemType Directory -Path $root -Force | Out-Null
$utc = (Get-Date).ToUniversalTime().ToString("o")
Write-Host "Inventory UTC: $utc"
~~~

Never commit raw JSON, workspace IDs, principal names, internal hostnames or authentication details to a public repository.

### Step 3 — Collect classic compute and policies

~~~powershell
databricks clusters list --output json |
  Out-File (Join-Path $root "classic-clusters.json") -Encoding utf8

databricks cluster-policies list --output json |
  Out-File (Join-Path $root "cluster-policies.json") -Encoding utf8

databricks instance-pools list --output json |
  Out-File (Join-Path $root "instance-pools.json") -Encoding utf8
~~~

If instance pools are not supported or accessible, document the gap. Check each command's --help and pagination behavior; a successful output file does not necessarily mean every page or every resource is included.

### Step 4 — Inventory jobs and SQL warehouses

~~~powershell
databricks jobs list --output json |
  Out-File (Join-Path $root "jobs.json") -Encoding utf8

databricks jobs list-runs --limit 25 --output json |
  Out-File (Join-Path $root "sample-job-runs.json") -Encoding utf8

databricks warehouses list --output json |
  Out-File (Join-Path $root "sql-warehouses.json") -Encoding utf8
~~~

The 25 runs are a sample, not complete job history. Some resources may be permission-filtered. Review job definitions to identify existing clusters, job clusters or supported serverless execution; warehouse listing is not a complete serverless inventory.

### Step 5 — Inspect one approved classic cluster

~~~powershell
databricks clusters get --help
databricks clusters events --help
~~~

Use the installed CLI's documented flags to inspect a specific authorized cluster and its events. Do not terminate or restart it. Record state, recent events, runtime, worker configuration, access mode, policy and auto-termination settings.

## 7. Inventory template

| Field | Example (fictional) |
|---|---|
| Workspace | demo-staging |
| Compute category | Classic jobs compute |
| Resource identifier | REDACTED |
| Owner | platform-team |
| State | RUNNING |
| Runtime | Actual version |
| Access mode | Actual configuration |
| Node type | Actual type |
| Workers / autoscale | Actual min/max |
| Policy | Approved policy |
| Auto-termination | Actual setting |
| Last active | From authorized telemetry |
| Cost source | Billing/system tables |
| Visibility | Complete / Partial / Unknown |

Capture UTC collection time, permission scope, pagination status and data source. A state snapshot cannot establish historical utilization.

## 8. Lifecycle monitoring and SLOs

Useful SRE signals:
- Cluster provisioning/startup latency and failure rate.
- Driver/worker loss, unexpected termination and restart count.
- Job queue delay, run success rate, retry rate and duration.
- SQL warehouse queueing, scaling, query failures and auto-stop behavior.
- Resource saturation: CPU, memory, disk, network and executor pressure where exposed.
- Idle compute hours and cost by owner/workload.
- Runtime version and policy drift.
- Cloud quota, regional capacity and pool utilization.

Set alert thresholds using actual workload baselines and user impact. Do not use arbitrary universal values as production SLOs.

## 9. Failure scenario A — Cluster stuck pending

**Inject (tabletop):** A scheduled job cannot start its classic compute.

**Triage sequence:**
1. Confirm job, cluster configuration, timestamp and affected tasks.
2. Review cluster events and termination/error reason.
3. Verify compute policy, permissions and runtime/node type.
4. Check cloud quota and available instance capacity in the correct region.
5. Check required networking, storage and instance profile/managed identity access.
6. Correlate recent configuration changes and cloud/vendor status.

**Mitigation:** bounded retries, approved alternative compute or a verified rollback. Do not increase permissions or loosen policies without review.

**Validation:** approved synthetic job completes and startup metrics return to normal.

## 10. Failure scenario B — Running cluster, slow workloads

**Inject:** Cluster reports RUNNING but job duration doubles.

**Triage:** examine driver/executor CPU, memory, GC, disk spill, skew, concurrency, autoscaling events and storage/network latency. Compare with a known-good baseline.

**Decision:** determine whether the problem is compute saturation, shared-resource contention, workload change or external dependency.

**Mitigation:** approved workload isolation, concurrency controls or capacity change after evidence review.

**Validation:** job latency and error rates recover without excessive cost.

## 11. Failure scenario C — Idle cost anomaly

**Inject:** Billing increases while scheduled job volume remains stable.

**Triage:**
- Check all-purpose clusters without appropriate auto-termination.
- Check instance pools and idle capacity.
- Review SQL warehouse auto-stop and serverless usage separately.
- Correlate billing with owner, job runs and timestamps.
- Identify orphaned resources without assuming they are safe to delete.

**Mitigation:** owner-approved policy, schedule, auto-termination or rightsizing changes.

**Validation:** sustained reduction in unused compute without breaking legitimate workloads.

## 12. Failure scenario D — Workload isolation violation

**Inject:** Two teams share compute and one team's burst affects the other's SLO.

**Triage:** confirm compute access mode, attached workloads, resource contention, data permissions and policy scope.

**Mitigation:** propose dedicated job compute, appropriate access mode, separate warehouses or serverless isolation where supported. Test in staging first.

**Validation:** workload performance and data-access boundaries remain correct.

## 13. Troubleshooting matrix

| Symptom | Likely cause | First safe check |
|---|---|---|
| PENDING too long | Capacity, quota, IAM, network | Cluster events and cloud limits |
| ERROR at startup | Runtime/init scripts/storage | Detailed startup errors |
| Unexpected termination | Auto-termination, policy, spot/preemptible loss | Termination reason and events |
| RUNNING but slow | Resource saturation, skew, contention | Compute metrics and task timeline |
| Job runs missing | Permissions or pagination | Scope and complete API retrieval |
| Warehouse queues grow | Concurrency/capacity/scaling | Warehouse metrics and query history |
| Serverless access fails | Managed connectivity or grants | Supported serverless telemetry |
| Idle spend rises | All-purpose/pools/warehouse/serverless | Billing and usage attribution |

## 14. Production runbook

1. **Detect:** job failures, slow queries, cluster events or cost anomalies.
2. **Scope:** workspace, cloud/region, compute category, owner and impacted workloads.
3. **Preserve:** UTC timestamps, sanitized errors, relevant metrics and configuration snapshots.
4. **Classify:** provisioning, runtime, capacity, isolation, network, identity or data dependency.
5. **Mitigate:** smallest approved change with a documented rollback.
6. **Validate:** synthetic job or query plus workload SLO recovery.
7. **Close:** incident timeline, cost impact, corrective actions and owners.

## 15. Acceptance checklist

- [ ] Compute categories and their operational differences documented.
- [ ] Classic cluster, policy, jobs and SQL warehouse inventory completed or gaps recorded.
- [ ] Serverless coverage and visibility limitations documented.
- [ ] Cluster lifecycle and termination reasons understood.
- [ ] Access mode, ownership and workload isolation reviewed.
- [ ] Startup, performance, cost and isolation tabletop scenarios completed.
- [ ] Runbook and monitoring signals reviewed.
- [ ] No production compute modified.
- [ ] No secrets or raw inventory committed.

## 16. Review questions

1. Why is a terminated jobs cluster often normal?
2. What is the difference between a classic SQL warehouse and general-purpose compute?
3. Why can shared compute create performance and security risks?
4. How would you distinguish EC2/VM quota failure from runtime startup failure?
5. Why can billing rise even when the cluster list looks empty?

## 17. References

- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks SQL warehouses: https://docs.databricks.com/aws/en/compute/sql-warehouse/
- Azure Databricks compute: https://learn.microsoft.com/azure/databricks/compute/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 012 — Cluster Policies, Node Types and Runtime Governance.
