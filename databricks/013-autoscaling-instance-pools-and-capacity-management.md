# Chapter 013 — Autoscaling, Instance Pools and Capacity Management

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 03 — Compute Infrastructure and Operations  
**Audience:** SRE, Platform Engineering, Cloud Infrastructure and FinOps  
**Lab mode:** Read-only inventory, capacity calculations and approved staging exercises  
**Prerequisites:** Chapters 011–012

## 1. Learning objectives

After this chapter, you should be able to:

1. Explain fixed-size versus autoscaling classic compute.
2. Describe instance pool warm capacity, idle cost and exhaustion.
3. Identify regional cloud quotas, instance availability and concurrency constraints.
4. Build a baseline for cluster startup, worker utilization and workload queueing.
5. Estimate capacity needs using workload demand, headroom and failure scenarios.
6. Diagnose slow scaling, unavailable nodes and pool starvation.
7. Execute safe capacity changes with explicit rollback and acceptance checks.

## 2. Architecture

~~~text
Databricks jobs / notebooks / SQL workloads
                  |
         Compute policy and quota
                  |
          Capacity allocation
             /         \
      Classic compute   Serverless compute
       fixed/autoscale   managed capacity model
             |
      +------+------+
      |             |
  Direct VMs     Instance pool
                 warm / idle VMs
      |             |
      +------+------+
             |
       Driver + workers
             |
      CPU / memory / disk
      cloud network / storage
             |
     Metrics, events and billing
~~~

**Key distinction:** classic cluster autoscaling, instance pools, SQL warehouse scaling and serverless scaling are different mechanisms. Do not apply a classic worker-count model to SQL warehouses or serverless workloads without confirming the feature's actual capacity semantics.

## 3. Compute capacity dimensions

| Dimension | SRE interpretation |
|---|---|
| Worker count | Parallel executor capacity; not a guarantee of throughput |
| Driver size | Orchestration, metadata and driver-side bottlenecks |
| VM family | CPU, RAM, local disk, network and price |
| Autoscaling minimum | Reserved floor while cluster is active |
| Autoscaling maximum | Upper bound; not guaranteed allocatable capacity |
| Instance pool minimum idle | Warm instances with associated idle cost |
| Instance pool maximum capacity | Pool allocation ceiling |
| Regional quota | Account/subscription/project capacity entitlement |
| Zonal capacity | Physical instance availability at a point in time |
| Job concurrency | Number of simultaneous runs/tasks competing for compute |
| Startup latency | Time from request to usable compute |
| Data dependencies | Storage, metastore, network and identity bottlenecks |

## 4. Fixed-size and autoscaling compute

**Fixed-size clusters** use a configured worker count, simplifying capacity predictability but potentially wasting idle resources.

**Autoscaling clusters** operate within configured minimum and maximum workers, adjusting allocated workers based on supported workload signals and platform behavior. Scaling is not instantaneous. Short jobs may finish before additional workers become useful.

### Operational tradeoffs

| Concern | Fixed workers | Autoscaling workers |
|---|---|---|
| Predictability | Higher resource stability | Capacity varies |
| Cost | Can overprovision | Can reduce idle worker spend |
| Burst response | Limited by fixed size | Depends on scaling delay and cloud capacity |
| Troubleshooting | Simpler resource baseline | Requires scaling-event correlation |
| Failure mode | Saturation | Saturation plus allocation/scale lag |

**Important:** increasing max workers does not fix a single-threaded driver bottleneck, data skew, storage I/O saturation or cloud quota shortage.

## 5. Instance pool operations

Instance pools can maintain reusable cloud instances for supported classic compute configurations. Pools may reduce startup latency, but idle instances can generate charges and consume capacity.

Inspect:
- Pool node type and supported configurations.
- Minimum idle instances and maximum capacity.
- Idle instance auto-termination behavior.
- Number of used, idle and pending instances.
- Clusters referencing the pool.
- Cloud quota and regional capacity.
- Driver and worker pool assignments where supported.

**Risk:** a shared pool can become a contention point across teams. A pool's configured maximum is not proof that its full capacity can be acquired.

## 6. Cloud-specific constraints

| Cloud | Capacity dependencies |
|---|---|
| AWS | EC2 family quotas, AZ capacity, spot interruption, instance profile and VPC |
| Azure | VM family/regional quotas, SKU availability, subnet IPs, NSG and outbound routing |
| GCP | Compute Engine quotas, zone capacity, machine family, VPC and service account |

Validate cloud quotas in the **actual workspace region and cloud account**. A workspace may have permission to request compute but still fail due to cloud allocation limits.

## 7. Lab A — Read-only Databricks inventory

Use an authorized PowerShell session.

### Verify workspace identity

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Replace the sample profile with your approved workspace profile.

### Inventory compute and pools

~~~powershell
databricks clusters list --output json
databricks instance-pools list --output json
databricks cluster-policies list --output json
databricks clusters list-node-types --output json
~~~

Check permission scope and pagination. Some workspaces or compute modes do not support pools; record **Not applicable** instead of assuming a failure.

### Inspect a specific cluster and pool

~~~powershell
databricks clusters get --help
databricks clusters events --help
databricks instance-pools get --help
~~~

Use the installed CLI's documented identifier arguments for one authorized resource. Capture sanitized values: worker bounds, node type, pool assignment, current state, event timeline and auto-termination.

## 8. Lab B — Cloud quota preflight

### AWS

~~~powershell
aws sts get-caller-identity
aws configure get region
aws service-quotas list-service-quotas --service-code ec2 --max-items 20
~~~

Quota listings may be paginated; filter to the instance-family quota relevant to your planned compute.

### Azure

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus --output table
~~~

Replace eastus with the approved workspace region.

### GCP

~~~powershell
gcloud config get-value project
gcloud compute regions describe us-central1 --format="json(quotas)"
~~~

Replace us-central1 with the verified region. Regional quotas are not a promise of zonal availability.

## 9. Lab C — Capacity worksheet

Start with observed peak workload concurrency, average workers per active job and a justified safety margin.

### Illustrative calculation

- Peak simultaneous classic job clusters: **8**
- Average workers per cluster: **4**
- Driver instances per cluster: **1**
- Estimated instances at peak: **8 × (4 + 1) = 40**
- Proposed 25% planning headroom: **40 × 1.25 = 50 instances**

This is an **illustrative instance-count estimate**, not a quota approval or cost forecast. Different node families, driver sizes, autoscaling peaks, pools and shared compute may invalidate this simplification.

### PowerShell worksheet

~~~powershell
$peakClusters = 8
$workersPerCluster = 4
$driversPerCluster = 1
$headroom = 0.25

$baseInstances = $peakClusters * ($workersPerCluster + $driversPerCluster)
$plannedInstances = [math]::Ceiling($baseInstances * (1 + $headroom))

[pscustomobject]@{
  PeakClusters = $peakClusters
  BaseInstances = $baseInstances
  PlannedInstances = $plannedInstances
  HeadroomPercent = $headroom * 100
}
~~~

For a real forecast, separate worker/driver instance families, CPU/memory quotas, spot/on-demand quotas, zones and pool idle capacity. Validate with measured startup and workload behavior.

## 10. Lab D — Autoscaling acceptance in staging

Only after approval:

1. Select a disposable staging cluster under an approved compute policy.
2. Record node family, runtime, min/max workers and cost limits.
3. Capture baseline cluster startup time and worker count.
4. Execute an authorized bounded synthetic workload with representative parallelism.
5. Observe scaling events, worker allocation, job duration and cloud capacity errors.
6. Compare scaling behavior against workload demand.
7. Stop the workload and verify scale-down or auto-termination behavior.
8. Terminate disposable compute and review billing.

**Do not** run an unbounded stress test or assume that a short-lived workload must trigger scale-up.

## 11. Lab E — Pool acceptance in staging

Where instance pools are supported:

1. Confirm the pool is disposable and not shared with production.
2. Document pool node type, min idle, max capacity and idle termination.
3. Measure cold-start time without a pool using a small approved workload.
4. Measure startup with a properly warmed staging pool.
5. Inspect used/idle/pending pool capacity and associated cost.
6. Verify pool behavior after workload termination.
7. Remove disposable pool resources only after confirming ownership.

**Acceptance:** startup improvement must justify idle capacity cost and operational complexity. Results are workload- and region-specific.

## 12. Failure scenario A — Autoscaling cannot reach max workers

**Symptoms:** job latency increases, worker count remains below configured max.

**Investigate:**
- Cloud quota and actual instance capacity.
- Cluster events and allocation errors.
- Node family, zone, network/subnet IP capacity.
- Pool max capacity or pool starvation.
- Whether workload demand actually triggers scale-up.

**Mitigation:** approved fallback node family, pool change or quota request. Do not increase autoscaling max without checking the limiting resource.

**Validate:** expected scaling under a bounded staging workload and recovery of job SLOs.

## 13. Failure scenario B — Shared instance pool exhausted

**Symptoms:** multiple jobs queue or fail to acquire instances.

**Investigate:** pool utilization, competing clusters, pending instances, cloud quotas, node type and idle configuration.

**Mitigation:** isolate critical workloads, adjust pool capacity within quotas or use approved direct provisioning.

**Validate:** critical job startup and absence of unintended starvation for other tenants.

## 14. Failure scenario C — High cost despite low workload

**Symptoms:** cloud compute costs remain elevated overnight.

**Investigate:** all-purpose clusters, idle pools, auto-termination, SQL warehouses and serverless usage separately. Correlate usage with tags, owners and billing.

**Mitigation:** review min idle, cluster schedules, auto-stop and policy defaults with workload owners.

**Validate:** reduced idle cost without increased job startup failures.

## 15. Failure scenario D — Scale-out worsens performance

**Symptoms:** more workers are allocated, but job duration does not improve.

**Investigate:** driver CPU, task skew, shuffle, object storage I/O, network throughput, executor utilization and scheduling overhead.

**Mitigation:** correct the actual bottleneck; rightsizing may mean **fewer** workers or a different node family.

**Validate:** improved latency and cost efficiency under representative workloads.

## 16. Troubleshooting matrix

| Symptom | Likely boundary | First evidence |
|---|---|---|
| Cluster PENDING | Quota, capacity, network, IAM | Cluster events + cloud limits |
| Slow startup | Cold VM acquisition, init scripts, pools | Startup timeline |
| Pool starvation | Max capacity, competing workloads | Pool usage and pending counts |
| Scale-out stalls | Quota, allocation or workload signals | Worker events and demand |
| Scale-down stalls | Active workload, platform behavior | Workload state and events |
| Job still slow after scaling | Driver, skew, storage or shuffle | Task/compute metrics |
| Unexpected compute spend | Idle pools, clusters, warehouses | Billing and usage attribution |
| Spot worker churn | Cloud interruption | Termination reasons |

## 17. Monitoring and alert design

Track:
- Cluster startup p50/p95/p99 by compute type and region.
- Provisioning failures and termination reasons.
- Autoscaling desired versus observed workers where available.
- Instance pool used/idle/pending capacity.
- Cloud quotas and quota-related errors.
- Job queue time, run duration and retries.
- Driver/worker CPU, memory, disk and network pressure.
- Idle compute hours and cost per owner/workload.
- Serverless and SQL warehouse capacity signals separately.

Use workload-specific SLOs and measured baselines rather than universal alert thresholds.

## 18. Production capacity change runbook

1. **Baseline:** record concurrency, startup time, utilization, errors and cost.
2. **Model:** estimate capacity by node family, region, workload and failure headroom.
3. **Check dependencies:** cloud quotas, subnet IPs, IAM, network and storage throughput.
4. **Review:** policy limits, shared pool impact, budget and blast radius.
5. **Stage:** test proposed settings with a bounded representative workload.
6. **Approve:** obtain owner and change-control sign-off.
7. **Roll out:** apply to a limited cohort and monitor startup and job SLOs.
8. **Rollback:** restore previous policy/pool/cluster settings if performance or cost regresses.
9. **Verify:** workloads healthy, costs acceptable, no orphaned idle resources.
10. **Document:** evidence, capacity decision and follow-up owners.

Changing pool or policy definitions may not immediately resize running compute. Account for replacement/restart needs in the rollback plan.

## 19. Acceptance checklist

- [ ] Authorized workspace and cloud region verified.
- [ ] Cluster, policy and pool inventory completed or visibility gaps recorded.
- [ ] Classic, SQL and serverless capacity models distinguished.
- [ ] Regional quotas and instance availability constraints documented.
- [ ] Capacity worksheet completed with assumptions.
- [ ] Autoscaling and pool staging tests completed or explicitly skipped.
- [ ] Four failure scenarios investigated.
- [ ] Monitoring and cost attribution reviewed.
- [ ] Rollback and teardown plans approved.
- [ ] No production compute changed or sensitive inventory committed.

## 20. Review questions

1. Why does a configured maximum worker count not guarantee available workers?
2. When does an instance pool reduce startup time but increase total cost?
3. How do cloud quotas differ from actual regional/zonal capacity?
4. Why can adding workers fail to improve a job?
5. What must be checked before changing a shared pool used by multiple teams?

## 21. References

- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks instance pools: https://docs.databricks.com/aws/en/compute/pool-index
- Databricks compute policies: https://docs.databricks.com/aws/en/admin/clusters/policy-definition
- Azure Databricks compute: https://learn.microsoft.com/azure/databricks/compute/
- Databricks on GCP: https://docs.databricks.com/gcp/
- AWS Service Quotas: https://docs.aws.amazon.com/servicequotas/
- Azure quotas: https://learn.microsoft.com/azure/quotas/
- Google Cloud quotas: https://cloud.google.com/docs/quotas

**Next:** Chapter 014 — Compute Monitoring, Performance and Cost Optimization.
