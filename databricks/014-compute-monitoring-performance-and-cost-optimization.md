# Chapter 014 — Compute Monitoring, Performance and Cost Optimization

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 03 — Compute Infrastructure and Operations  
**Audience:** SRE, Platform Engineering, Infrastructure Operations and FinOps  
**Lab mode:** Read-only diagnostics and approved staging performance exercises  
**Prerequisites:** Chapters 011–013

## 1. Learning objectives

After this chapter, you can:
1. Define practical compute SLIs for availability, startup, performance and spend.
2. Collect evidence from cluster events, job runs, SQL warehouses and system tables.
3. Separate driver, executor, storage, network and external dependency bottlenecks.
4. Understand Databricks billing versus underlying cloud infrastructure cost.
5. Build an actionable monitoring and cost optimization plan.
6. Investigate performance and cost anomalies without changing production workloads.
7. Validate recovery, rollback and operational acceptance.

## 2. Observability architecture

~~~text
Applications / notebooks / jobs / SQL queries
                    |
              Databricks compute
          +---------+----------+
          |                    |
     Classic clusters      SQL / serverless
          |                    |
  Driver / executors      Query and service
  Spark UI / events       performance signals
          +---------+----------+
                    |
           Workspace / account APIs
           System tables (if enabled)
           Cloud metrics and billing
                    |
              SRE dashboards
         Alerts / incident response
~~~

**Critical distinction:** A cluster marked RUNNING is not necessarily healthy. A successful job can still violate latency SLOs. A low Databricks usage charge does not guarantee low underlying cloud cost for classic compute.

## 3. Monitoring layers

| Layer | Primary evidence | Typical issue |
|---|---|---|
| Workspace and APIs | Availability, auth errors, API latency | Access/control-plane problem |
| Compute lifecycle | State transitions, cluster events, startup time | Capacity, IAM, init script, network |
| Job execution | Run state, duration, retries, queue delay | Reliability or scheduling |
| Spark execution | Stages, tasks, skew, shuffle, spill, GC | Execution bottleneck |
| SQL warehouse | Queue time, query duration, scaling, failures | Concurrency and capacity |
| Cloud infrastructure | VM CPU/memory, disk, network, quota | Resource pressure |
| Storage dependencies | Object store latency, throttling, auth | External I/O bottleneck |
| Cost | DBUs, cloud VM/storage/network charges | Idle spend, overprovisioning |

Not every metric is exposed identically across AWS, Azure, GCP, classic and serverless. Record source and collection scope.

## 4. SLI and SLO design

Suggested SLIs:
- Successful job runs / total eligible job runs.
- p50/p95/p99 job duration for comparable workloads.
- Compute startup latency and startup failure rate.
- SQL query latency, queue wait and error rate.
- Percentage of time critical workloads are blocked by capacity.
- Idle classic compute hours.
- Cost per completed workload or useful data volume.

Define SLOs based on application requirements, measurement windows and observed baselines. Do not use a single global threshold for all workload types.

### Example (illustrative)

~~~text
SLI: critical daily pipeline completed by deadline
SLO: >= 99% of scheduled runs complete by 07:00 UTC over 30 days
Alert: forecasted deadline miss based on current queue and run time
Owner: data platform on-call
Exclusions: approved maintenance only if explicitly documented
~~~

## 5. Lab A — Read-only workspace inventory

Use an approved PowerShell session.

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Replace the example profile and verify the workspace host before querying.

### Cluster and job status

~~~powershell
databricks clusters list --output json
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks warehouses list --output json
~~~

**Note:** Job runs are a sample. API pagination, visibility permissions and retention affect completeness.

### Inspect event APIs

~~~powershell
databricks clusters get --help
databricks clusters events --help
databricks jobs get-run --help
databricks warehouses get --help
~~~

Use the installed CLI's documented flags for one authorized resource. Collect UTC timestamps, lifecycle events, state, retry count, compute configuration and termination reason. Keep raw IDs and logs private.

## 6. Lab B — Spark performance triage

For an approved job with a reproducible slowdown:

1. Establish the same input volume, runtime and compute configuration as the baseline.
2. Review the Spark UI for long stages, skewed task durations and failed tasks.
3. Check shuffle read/write, spill, executor loss and garbage collection.
4. Compare driver versus executor resource pressure.
5. Check autoscaling events and cloud quota/capacity signals.
6. Correlate object storage latency and network errors.
7. Separate compute saturation from a query/data layout regression.
8. Preserve sanitized screenshots and metric summaries in restricted evidence storage.

### Symptom mapping

| Observation | Hypothesis | Validation |
|---|---|---|
| High driver CPU, workers underused | Driver bottleneck | Driver timeline and scheduling |
| Few tasks dominate stage duration | Data skew | Task distribution |
| Large disk spill | Memory/shuffle pressure | Spill bytes and executor memory |
| Repeated executor loss | OOM, preemption, node instability | Executor and cloud events |
| Low CPU, long I/O waits | Storage/network dependency | Request latency and throughput |
| Workers scale late | Scaling lag or cloud capacity | Cluster events and demand |

Do not conclude that adding workers fixes every slow Spark job.

## 7. Lab C — SQL warehouse monitoring

Inspect the approved SQL warehouse and query history using workspace-supported APIs/UI:

~~~powershell
databricks warehouses list --output json
databricks warehouses get --help
~~~

Capture:
- warehouse type and size;
- auto-stop and scaling settings;
- query queue duration versus execution duration;
- concurrency, failures and timeouts;
- usage and cost attribution;
- recent configuration or workload changes.

A queueing issue is not the same as slow execution. Serverless SQL performance and cost must be analyzed using the supported warehouse telemetry and billing model.

## 8. Lab D — System tables for compute usage

Where enabled and authorized, Databricks system tables can provide account-level billing usage and price metadata. Schema and retention may evolve; inspect current columns before adopting any query.

### Example usage aggregation

~~~sql
SELECT
  usage_date,
  workspace_id,
  sku_name,
  SUM(usage_quantity) AS usage_quantity
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 7)
GROUP BY usage_date, workspace_id, sku_name
ORDER BY usage_date DESC, workspace_id, sku_name;
~~~

**Interpretation:** usage_quantity must be interpreted with usage_unit and SKU semantics. Do not label all quantities as DBUs without checking the records. Access may require account-level permissions and system table enablement.

### Example with unit awareness

~~~sql
SELECT
  usage_date,
  sku_name,
  usage_unit,
  SUM(usage_quantity) AS total_usage
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 7)
GROUP BY usage_date, sku_name, usage_unit
ORDER BY usage_date DESC;
~~~

This query summarizes Databricks-reported usage, **not** total cloud spend. For classic compute, include cloud VM, storage, NAT, data transfer and related infrastructure charges.

## 9. Cost model and attribution

### Total cost components

~~~text
Total platform cost =
  Databricks service usage
+ cloud compute charges (where separately billed)
+ storage and I/O
+ network egress/NAT/private connectivity
+ logging and monitoring
+ support and other contracted charges
~~~

Actual billing depends on cloud, SKU, pricing agreement and compute mode. Avoid applying one flat DBU price across all SKUs or dates.

### Attribution controls

- Workspace, environment and cost center tags.
- Job ownership and workload identity.
- Compute policy and node family.
- Usage records with time and SKU.
- Cloud billing exports and discounts.
- Idle resource inventory and pool utilization.

## 10. Lab E — Simple cost efficiency worksheet

Illustrative values only:

- Classic cluster: 6 workers + 1 driver.
- Total cloud VM rate assumed: $2.10/hour.
- Databricks service rate assumed: $1.40/hour.
- Cluster running 10 hours/day.
- Productive execution 6 hours/day.

~~~powershell
$cloudHourly = 2.10
$serviceHourly = 1.40
$runningHours = 10
$productiveHours = 6

$dailyCost = ($cloudHourly + $serviceHourly) * $runningHours
$productiveShare = $productiveHours / $runningHours
$unproductiveCostEstimate = $dailyCost * (1 - $productiveShare)

[pscustomobject]@{
  DailyCostUSD = [math]::Round($dailyCost, 2)
  ProductiveSharePct = [math]::Round($productiveShare * 100, 1)
  IllustrativeUnproductiveCostUSD = [math]::Round($unproductiveCostEstimate, 2)
}
~~~

The calculation is a simplified allocation exercise, not a real pricing estimate. Replace assumptions with actual SKU-specific Databricks and cloud billing data, and account for workload overlap and non-linear charges.

## 11. Failure scenario A — Job latency regression

**Symptom:** p95 job duration increases after a runtime or data change.

**Triage:** compare input volume, runtime, stage/task skew, spill, executor loss, storage latency, autoscaling and recent deployments.

**Mitigation:** restore a verified compatible runtime/configuration or fix the actual bottleneck through change control.

**Validation:** representative workload meets its SLO without unacceptable cost.

## 12. Failure scenario B — Cluster startup failures

**Symptom:** jobs remain queued or compute fails during provisioning.

**Triage:** cluster events, node type, quotas, capacity, network, instance profile/managed identity and init scripts.

**Mitigation:** approved fallback node family or corrected dependency; avoid broad IAM/network permissions.

**Validation:** synthetic job starts and completes; failure rate returns to baseline.

## 13. Failure scenario C — SQL warehouse queueing

**Symptom:** BI queries time out during peak traffic.

**Triage:** queue time versus execution time, concurrency, scaling limits, query workload mix and downstream data access.

**Mitigation:** approved workload isolation, query optimization or warehouse capacity adjustment.

**Validation:** query latency, queue time and cost meet agreed targets.

## 14. Failure scenario D — Cost spike without workload growth

**Symptom:** daily spend increases while job count remains stable.

**Triage:** idle all-purpose clusters, instance pools, SQL warehouses, serverless usage, changed node types, network egress and cloud billing anomalies.

**Mitigation:** identify attributable resource and owner before rightsizing or termination.

**Validation:** spend returns toward baseline without reducing reliability.

## 15. Troubleshooting matrix

| Symptom | Likely boundary | First evidence |
|---|---|---|
| Cluster RUNNING, job slow | Spark/data/storage | Stage and task metrics |
| Long cluster startup | Quota/capacity/init/network | Cluster event timeline |
| Frequent executor loss | Memory/preemption/node | Executor and cloud logs |
| SQL query queued | Warehouse concurrency | Query queue metrics |
| SQL query slow executing | Query/data/compute | Query profile |
| Billing usage rises | SKU/workload change | system.billing.usage |
| Cloud bill rises, DBUs stable | VM/network/storage | Cloud cost export |
| Serverless cost anomaly | Managed usage and SKU | Supported usage records |

## 16. Monitoring and alert acceptance

Minimum production dashboard:
1. Critical job success rate and deadline adherence.
2. Cluster startup latency and failure reasons.
3. Job duration distributions and retries.
4. SQL warehouse queueing and error rates.
5. Spark resource pressure and executor loss.
6. Databricks usage by workspace/SKU.
7. Cloud compute, storage and network costs.
8. Idle compute and instance pool waste.
9. Alert ownership, runbook links and suppression policy.

Alerts must be actionable, linked to user impact and tested in staging.

## 17. Production incident runbook

1. **Detect:** identify violated SLI or cost anomaly.
2. **Scope:** workspace, cloud, compute type, job/warehouse and affected users.
3. **Preserve:** UTC timeline, metrics, sanitized events and recent changes.
4. **Classify:** provisioning, compute saturation, query, storage, network, identity or billing.
5. **Mitigate:** smallest safe approved action with rollback.
6. **Validate:** synthetic workload plus real SLO and cost signals.
7. **Document:** root cause or uncertainty, impact, cost, owners and prevention.

## 18. Change and rollback plan

Before changing node type, runtime, autoscaling, pool or warehouse settings:
- Capture baseline configuration and utilization.
- Record dependent workloads and access controls.
- Estimate cost and quota impact.
- Test the candidate in staging.
- Define rollback configuration and restart implications.
- Roll out to a limited cohort.
- Verify latency, reliability and spend after the change.

Restoring a policy alone may not undo a change to already-running compute.

## 19. Acceptance checklist

- [ ] Workspace and compute mode verified.
- [ ] Cluster/job/warehouse telemetry sources inventoried.
- [ ] Performance baseline and relevant SLIs defined.
- [ ] Spark bottleneck investigation performed or simulated.
- [ ] Billing usage queries reviewed with units and SKU context.
- [ ] Databricks versus cloud cost separated.
- [ ] Four failure scenarios completed.
- [ ] Dashboard, alerts and runbook reviewed.
- [ ] Rollback procedure documented.
- [ ] No production workload changed or secrets exposed.

## 20. Review questions

1. Why can RUNNING compute still violate an availability or latency SLO?
2. How do queue time and execution time differ for SQL warehouses?
3. Why can adding workers fail to improve a Spark job?
4. Why does Databricks usage not equal the full cloud infrastructure bill?
5. What evidence is needed before terminating an apparently idle resource?

## 21. References

- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks jobs monitoring: https://docs.databricks.com/aws/en/jobs/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks billing usage: https://docs.databricks.com/aws/en/admin/system-tables/billing
- Databricks SQL warehouses: https://docs.databricks.com/aws/en/compute/sql-warehouse/
- Azure Databricks monitoring: https://learn.microsoft.com/azure/databricks/admin/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 015 — Compute Operations and Failure Recovery Lab, the Part 03 capstone.
