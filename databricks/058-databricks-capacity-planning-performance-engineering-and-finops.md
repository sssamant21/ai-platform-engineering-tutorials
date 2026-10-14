# Chapter 058 — Databricks Capacity Planning, Performance Engineering and FinOps

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 13 — Capacity, Cost Governance and Enterprise Acceptance  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform Engineering, Data Engineering, FinOps and Cloud Operations  
**Prerequisites:** Chapters 041–057  
**Safety:** Production discovery is read-only. Performance stress, warehouse resizing, autoscaling changes, policy updates and billing configuration changes require change approval and should first run in isolated staging. Querying billing tables can itself incur compute charges. Never export sensitive query text, identities, raw plans or credentials to public artifacts.

## 1. Mission

Build a repeatable Databricks capacity and FinOps program that connects workload demand, service-level objectives, compute architecture, SQL performance, data layout, cloud constraints and financial accountability. Measure and optimize **cost per successful business outcome**, not simply raw DBUs or cluster size.

## 2. Learning outcomes

- Inventory workload classes, growth drivers and service dependencies.
- Establish reproducible capacity, latency, throughput and spend baselines.
- Understand DBUs, cloud infrastructure charges, storage and networking costs.
- Forecast compute and storage using utilization, growth and scenario analysis.
- Right-size job clusters and SQL warehouses without sacrificing SLOs.
- Diagnose queueing, skew, spill, excessive scans and autoscaling delays.
- Implement tagging, budgets, anomaly detection and showback.
- Run staging performance and cost acceptance experiments.
- Write production capacity runbooks and GO/NO-GO decisions.

## 3. Capacity and cost architecture

    Demand: schedules / concurrency / data growth
                         |
          Jobs + SQL + streaming + ML workloads
                         |
           Databricks compute + storage layout
                         |
          Cloud VM / object storage / networking
                         |
           Performance, reliability, data quality
                         |
       System billing tables + cloud billing data
                         |
        Unit economics + forecast + cost controls
                         |
         Approved change + measurement + review

A lower-cost configuration is not an optimization if it increases retries, misses delivery deadlines, corrupts data or requires unacceptable operator effort.

## 4. Workload classification

| Class | Primary capacity driver | Typical acceptance |
|---|---|---|
| Scheduled ETL | Input volume, shuffle, deadlines | Completed correctly before deadline |
| Streaming | Arrival rate, state size, lag | Freshness and stable backlog |
| Interactive SQL | Concurrency, queueing, query complexity | P95/P99 and success SLO |
| Ad hoc analytics | Unpredictable query mix | Budget and fair resource access |
| ML training | Data size, accelerators, iterations | Cost per successful training run |
| Model inference | Requests, tokens, concurrency | Latency, throughput, cost/request |
| Governance jobs | Catalog scale, audit volume | Coverage and completion |

Separate baselines by environment, region, SKU, workload and time window.

## 5. Establish a performance baseline

Capture at least one representative operating cycle: peak/median demand, number of eligible runs, runtime distribution, input/output bytes, rows, failures, retries, queue time, cluster startup time, SQL query P50/P95/P99, data freshness, compute utilization, storage growth and cloud billing. Annotate deployments, holidays, backfills and incidents.

## 6. Define unit economics

Useful measures include:
- DBUs per successful job or per terabyte processed.
- Cloud infrastructure cost per accepted dataset refresh.
- Total cost per million valid records delivered.
- SQL cost per 1,000 successful eligible queries.
- Cost per training completion or inference request.

The denominator must exclude invalid outputs and account for retries, reruns and downstream effects.

## 7. Cost components

**Databricks:** Metered product/SKU usage, applicable pricing and commitments.

**Cloud:** Compute instances, attached storage, object storage requests/capacity, network egress, load balancers, NAT, KMS and monitoring.

**Operational:** Data reprocessing, failures, overprovisioned standby capacity and engineering time.

Billing usage quantity is not necessarily dollars. Apply effective prices by SKU, time, cloud and contract, and reconcile with the cloud bill. Do not assume a public list price equals a contracted invoice.

## 8. Capacity forecasting model

For an approximately linear workload:

    forecast_input = baseline_input × growth_factor
    forecast_compute = baseline_compute × growth_factor × efficiency_factor
    required_capacity = peak_demand / target_utilization

These are starting estimates, not universal Databricks sizing formulas. Account for nonlinear shuffle, joins, state growth, query concurrency, startup overhead, skew and storage throughput. Model baseline, expected growth and stress scenarios.

## 9. Concurrency and queueing

Observe submitted versus executing queries, queue duration, task scheduling delay and concurrency caps. Increasing cluster size may not fix serialized work, skew or external source throttling. For SQL warehouses, test scale-up versus scale-out and auto-stop behavior with representative queries and concurrency.

## 10. Spark performance fundamentals

Investigate execution plans, partition pruning, scan volume, shuffle bytes, skewed keys, spilled data, executor memory pressure, garbage collection and task distribution. Verify optimization with workload-level measurements, not a single faster query. Changing partition counts can improve or worsen performance depending on data distribution.

## 11. Delta Lake data layout

Review small files, compaction, clustering strategies supported by the selected runtime/table type, statistics, pruning, retention and ingestion patterns. Avoid blanket partitioning by high-cardinality columns. Assess optimization operations for extra compute and storage cost and potential contention.

## 12. Autoscaling tradeoffs

Measure provisioning delay, min/max worker bounds, idle time, retry behavior, cloud quotas and SLO impact. Scale-to-zero or aggressive auto-stop saves idle cost but may increase startup latency. Set guardrails based on actual traffic and business deadlines.

## 13. Multi-cloud constraints

**AWS:** EC2 instance families, regional capacity, EBS and S3 characteristics, VPC/NAT and egress, IAM and quotas.

**Azure:** VM SKUs and quotas, managed disks, ADLS, Private Link, Azure Monitor and networking charges.

**GCP:** Compute Engine machine types and quotas, Cloud Storage, Cloud NAT, VPC networking and Cloud Monitoring.

Account for availability zones, cloud discounts, regional pricing, data residency and cross-region transfer costs.

## 14. Lab A — Windows PowerShell preflight [SAFE-READ]

~~~powershell
git rev-parse HEAD
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm the intended staging workspace and identity before accessing usage or changing resources.

## 15. Lab B — System table availability [SAFE-READ]

~~~sql
SELECT current_user();
SHOW CATALOGS;
SHOW SCHEMAS IN system;
SHOW TABLES IN system.billing;
SHOW TABLES IN system.lakeflow;
SHOW TABLES IN system.compute;
SHOW TABLES IN system.query;
~~~

Availability, schema, access and freshness vary by cloud/account. Missing telemetry is an observability gap, not zero consumption.

## 16. Lab C — DBU usage trend [SAFE-READ]

~~~sql
SELECT
  usage_date,
  sku_name,
  usage_unit,
  SUM(usage_quantity) AS total_usage
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 14)
GROUP BY usage_date, sku_name, usage_unit
ORDER BY usage_date DESC, sku_name;
~~~

Do not sum different units as one metric. Usage tables can arrive late. Limit query scans to the needed dates and columns.

## 17. Lab D — Workspace and SKU attribution [SAFE-READ]

~~~sql
SELECT
  workspace_id,
  sku_name,
  usage_unit,
  SUM(usage_quantity) AS usage
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 7)
GROUP BY workspace_id, sku_name, usage_unit
ORDER BY workspace_id, usage DESC;
~~~

Workspace-level attribution is not necessarily team-level chargeback. Reconcile job, resource and custom tag metadata with billing records before allocating cost.

## 18. Lab E — Pricing schema discovery [SAFE-READ]

~~~sql
DESCRIBE TABLE system.billing.list_prices;
SELECT *
FROM system.billing.list_prices
LIMIT 10;
~~~

Pricing schemas and availability can change. List prices are not guaranteed to match negotiated rates or the final invoice. Build an approved effective-price mapping with finance rather than assuming one universal rate.

## 19. Lab F — Job duration and retry investigation [SAFE-READ]

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
~~~

For a known authorized run:

~~~powershell
databricks jobs get-run REPLACE_WITH_RUN_ID --output json
~~~

Capture run ID, final status, duration, queueing, retries and cluster association. Paginate for a representative sample; do not extrapolate from only 25 runs.

## 20. Lab G — SQL query history [SAFE-READ]

~~~sql
SHOW TABLES IN system.query;
DESCRIBE TABLE system.query.history;
SELECT * FROM system.query.history LIMIT 10;
~~~

Inspect actual schema, then calculate workload-specific P50/P95/P99 and failure rates using authorized columns. Query text and user identity are sensitive; use aggregation and redaction.

## 21. Lab H — Compute and warehouse inventory [SAFE-READ]

~~~powershell
databricks clusters list --output json
databricks cluster-policies list --output json
databricks warehouses list --output json
~~~

Record node types, autoscaling bounds, policies, schedules, ownership and warehouse sizes. Check regional instance availability and quotas before proposing changes.

## 22. Lab I — Capacity scenario calculator [LOCAL]

~~~powershell
$baselineDailyGb = 800
$annualGrowth = 0.35
$years = 2
$targetUtilization = 0.65
$forecastDailyGb = $baselineDailyGb * [math]::Pow(1 + $annualGrowth, $years)
$peakMultiplier = 1.8
[pscustomobject]@{
  ForecastDailyGB = [math]::Round($forecastDailyGb, 1)
  PeakEquivalentGB = [math]::Round($forecastDailyGb * $peakMultiplier, 1)
  CapacityAtTargetUtilGB = [math]::Round(($forecastDailyGb * $peakMultiplier) / $targetUtilization, 1)
}
~~~

Expected: 1,458 GB/day forecast, 2,624.4 GB peak equivalent and about 4,037.5 GB nominal headroom-equivalent. These are demand indices, not physical node sizing; calibrate with actual workload benchmarks.

## 23. Lab J — Cost-per-successful-run calculator [LOCAL]

~~~powershell
$totalCostUsd = 126.00
$successfulRuns = 420
$failedRuns = 30
[pscustomobject]@{
  TotalRuns = $successfulRuns + $failedRuns
  CostPerSuccessfulRunUSD = [math]::Round($totalCostUsd / $successfulRuns, 4)
  FailureFraction = [math]::Round($failedRuns / ($successfulRuns + $failedRuns), 4)
}
~~~

Expected: $0.30 per successful run and 6.67% failed runs. Use actual blended cloud plus Databricks costs, not fabricated production prices.

## 24. Staging performance acceptance experiment

1. Choose a representative synthetic workload and approved staging budget.
2. Record input size, data distribution, baseline runtime, queue time and correctness.
3. Define success thresholds for SLO, spend, throughput and resource pressure.
4. Change exactly one variable: worker size, autoscaling bound, warehouse size or query optimization.
5. Run multiple comparable repetitions, including a cold-start case where relevant.
6. Validate row counts, uniqueness, schema and freshness.
7. Compare median and tail latency plus total cost per accepted result.
8. Roll back staging configuration if thresholds fail.
9. Document evidence, tradeoffs and approval.

Avoid extrapolating from tiny synthetic datasets to production without calibration.

## 25. FinOps ownership and tags

Define cost center, environment, workload, service owner, project and business unit tags where supported. Validate propagation into billable usage metadata. Tagging gaps are an attribution problem; avoid inventing chargeback precision.

## 26. Budget and anomaly controls

Set approved budget per environment/team, daily/weekly trend checks, forecast-to-budget, idle compute detection, unbounded retries and unexpected SKU changes. Route critical overspend risk to platform and FinOps owners, with change context and false-positive handling. Cost alerts should not automatically stop critical production jobs without an approved policy.

## 27. Right-sizing decision matrix

| Observation | Investigate first | Candidate action |
|---|---|---|
| High idle time | Schedules and startup requirements | Auto-stop / job clusters |
| Long queue, low execution time | Concurrency and startup | Warehouse scale/concurrency |
| High shuffle spill | Join, skew and memory | Query tuning or memory adjustment |
| High retry spend | Root causes and backoff | Fix failure and idempotency |
| Small files | Ingestion pattern and pruning | Appropriate compaction strategy |
| Persistent CPU saturation | Parallelism and workload mix | Benchmark alternate compute |
| High cross-region egress | Data placement and consumer routing | Architecture/data locality review |
| Storage growth | Retention and lineage | Governed lifecycle optimization |

Every candidate requires SLO and correctness acceptance.

## 28. Scenario A — DBU usage rises after release

**Detect:** SKU/workspace usage exceeds baseline.  
**Investigate:** changed schedules, compute sizing, retries and data volume.  
**Accept:** explained spend or validated corrective action.

## 29. Scenario B — SQL P99 latency regresses

**Detect:** tail latency increases for a defined query class.  
**Investigate:** queueing, query plans, concurrency and warehouse startup.  
**Accept:** P99 and failure SLO restored with cost evidence.

## 30. Scenario C — Autoscaling reacts too slowly

**Detect:** backlog grows while nodes provision.  
**Investigate:** min workers, cloud quotas, startup and workload burst.  
**Accept:** deadline met without unacceptable idle spend.

## 31. Scenario D — Executor memory pressure

**Detect:** OOM, GC pressure or shuffle spill.  
**Investigate:** skew, joins, partitioning and memory footprint.  
**Accept:** stable runs and validated output at acceptable cost.

## 32. Scenario E — Data layout causes full scans

**Detect:** scan bytes and runtime grow disproportionately.  
**Investigate:** pruning, filters, statistics and file layout.  
**Accept:** reduced scan volume and reproducible query improvement.

## 33. Scenario F — Unbounded retry cost

**Detect:** repeated failed attempts dominate compute usage.  
**Investigate:** root cause, retry limits, backoff and idempotency.  
**Accept:** bounded retries and correct final results.

## 34. Scenario G — Billing attribution missing

**Detect:** usage has incomplete workload/team tags.  
**Investigate:** tag policy, compute type and system-table metadata.  
**Accept:** approved attribution coverage or documented unallocated cost.

## 35. Scenario H — Cross-region egress spike

**Detect:** cloud network charges increase.  
**Investigate:** storage/compute region mismatch, replication and consumers.  
**Accept:** architecture or spend exception approved with measured impact.

## 36. Scenario I — Warehouse auto-stop harms availability

**Detect:** first-query latency breaches dashboard SLO.  
**Investigate:** cold start, schedule and demand pattern.  
**Accept:** acceptable latency/cost tradeoff with tested settings.

## 37. Scenario J — Cost reduction breaks data freshness

**Detect:** DBUs decrease but business deadline is missed.  
**Investigate:** undersizing, queueing, throttling and retries.  
**Accept:** restored freshness and correct unit-economics assessment.

## 38. Capacity review runbook

~~~markdown
# Databricks Capacity Review
- Review period / UTC:
- Workspace, region and owner:
- Critical workload classes:
- Demand baseline and peak:
- Data growth and retention:
- Jobs success / runtime / queue:
- SQL P95/P99 and concurrency:
- Compute utilization and startup:
- Cloud quota and capacity risks:
- DBU usage and effective spend:
- Cost per accepted business outcome:
- Forecast scenarios and confidence:
- Recommended changes and expected SLO effect:
- Staging benchmark and rollback:
- Approver / next review:
~~~

## 39. Cost anomaly incident runbook

1. Verify data freshness and billing ingestion lag.
2. Identify affected account, workspace, SKU and usage unit.
3. Correlate deployment, job schedules and run retries.
4. Compare cloud bill and Databricks usage, recognizing timing differences.
5. Check tags and workload ownership.
6. Estimate forecast exposure and service impact.
7. Propose the smallest approved corrective action.
8. Validate performance, correctness and spend afterward.
9. Record owner, decision and residual risk.

## 40. Evidence package

    databricks-capacity-finops-YYYYMMDD/
      01-scope-and-workload-map.md
      02-slo-and-demand-baseline.md
      03-billing-usage-and-pricing.md
      04-cloud-cost-reconciliation.md
      05-job-performance.md
      06-sql-performance.md
      07-compute-and-autoscaling.md
      08-data-layout-and-storage.md
      09-forecast-scenarios.md
      10-staging-benchmark.md
      11-recommendations-and-rollback.md
      12-acceptance-and-owners.md

Store restricted billing contracts, query text and identities only in approved access-controlled systems.

## 41. Acceptance scorecard

| Gate | Evidence |
|---|---|
| Demand | Representative workload and peak baseline |
| SLOs | Job, SQL and freshness objectives |
| Correctness | Output counts and invariants |
| Billing | SKU/unit mapping and freshness |
| Cloud costs | Infrastructure and egress reconciliation |
| Attribution | Owner and tag coverage |
| Capacity | Utilization, queue and quotas |
| Forecast | Baseline, growth and stress scenario |
| Benchmark | Comparable repeated staging results |
| Change | Approved implementation and rollback |
| Alerts | Budget, cost anomaly and SLO |
| Ownership | Platform, workload and FinOps sign-off |

## 42. GO / CONDITIONAL GO / NO-GO

**GO:** Representative benchmarks meet SLO and data correctness, projected capacity has approved headroom, cost attribution is credible and the change has a rollback.

**CONDITIONAL GO:** Noncritical gaps have explicit owners, expiration dates and compensating monitoring.

**NO-GO:** Missing workload baseline, unbounded spend, violated freshness/correctness, unverified cloud quotas, unknown billing units or no recovery path.

## 43. Review questions

1. Why is DBU reduction alone an incomplete optimization target?
2. How should cost per successful run handle retries?
3. Why must SKU usage units be separated?
4. What is the difference between cloud infrastructure charges and Databricks charges?
5. How can SQL queueing dominate P99 latency?
6. Why might aggressive auto-stop worsen business SLOs?
7. Which signals distinguish skew from insufficient cluster capacity?
8. How should growth forecasts be validated against actual workloads?

## 44. Official references

- Databricks system billing tables: https://docs.databricks.com/aws/en/admin/system-tables/billing
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks Jobs monitoring: https://docs.databricks.com/aws/en/jobs/monitor
- Databricks SQL query history: https://docs.databricks.com/aws/en/sql/user/queries/query-history
- Databricks compute policies: https://docs.databricks.com/aws/en/compute/policies
- Databricks performance optimization: https://docs.databricks.com/aws/en/optimizations/
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- AWS Cost Explorer: https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html
- Azure Cost Management: https://learn.microsoft.com/azure/cost-management-billing/costs/overview-cost-management
- Google Cloud billing: https://cloud.google.com/billing/docs/how-to/manage-billing-account
- FinOps Foundation: https://www.finops.org/framework/

**Next:** Chapter 059 — Databricks Production Operations and Enterprise Reliability Acceptance Project.
