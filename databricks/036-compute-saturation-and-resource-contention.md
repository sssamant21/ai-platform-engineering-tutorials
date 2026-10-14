# Chapter 036 — Compute Saturation and Resource Contention

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 08 — Performance and Troubleshooting  
**Clouds:** AWS, Azure, GCP  
**Audience:** SRE, DBRE, Cloud Platform and Performance Engineering  
**Prerequisites:** Chapters 011–015, 031–035  
**Safety:** Production diagnostics are read-only. Compute modifications and load tests require approved staging and a cost cap.

## 1. Mission

Identify, quantify and safely remediate Databricks compute saturation and resource contention. Distinguish insufficient capacity from skew, scheduling delay, driver bottlenecks, executor memory pressure, disk spill, network/storage waits and quota restrictions.

**Outcome:** An SRE can determine which resource is limiting a workload, what evidence proves it, and which low-risk remediation will improve the service-level objective without unnecessary spend.

## 2. Compute performance architecture

    Scheduled job / interactive query
                  |
         Queue / admission limits
                  |
          Cluster provisioning
                  |
             Spark driver
            /     |      \
        Executor Executor Executor
            \     |      /
        CPU / heap / disk / network
                  |
         Shuffle / cloud storage
                  |
       SLO, correctness and cost

A long-running job does not automatically mean CPU saturation. Separate waiting from executing and the slowest task from the average executor.

## 3. Resource contention taxonomy

| Bottleneck | Symptoms | Primary evidence |
|---|---|---|
| CPU saturation | High runnable work, long task execution | Per-node CPU and task CPU time |
| Memory pressure | GC, spills, OOM, executor loss | Heap/off-heap and executor logs |
| Driver pressure | Slow planning, driver OOM | Driver logs, CPU/memory, job plan |
| Local disk pressure | Spill slowdown, disk-full errors | Disk usage, spill bytes, I/O latency |
| Network pressure | Shuffle fetch delays and failures | Shuffle read wait, network telemetry |
| Cloud storage wait | Low CPU but slow tasks | Object request latency/throttling |
| Scheduling contention | Queue time and task backlog | Job/run timeline and stage tasks |
| Quota/capacity | Failed or delayed provisioning | Cluster events, provider quotas |
| Noisy neighbor | Variable latency across shared workloads | Concurrent run and capacity timeline |

## 4. Four diagnostic rules

1. Diagnose the **phase** first: queued, provisioning, initialization, Spark execution or output commit.
2. Compare **per-node and per-task distributions**, not only averages.
3. Correlate workload telemetry with cloud infrastructure events.
4. Validate **correctness, latency and cost together** before declaring improvement.

## 5. Establish a healthy baseline

Capture at least:
- p50/p95 job duration and business deadline.
- Queue and cluster startup time.
- Worker count and instance type.
- Driver and executor CPU/memory.
- JVM garbage collection time.
- Shuffle read/write and disk spill.
- Task count, median/p95/max duration and failed tasks.
- Cloud storage request errors, latency and network.
- Cost per validated successful run.

Separate scheduled peak periods from low-load windows.

## 6. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm the actual workspace URL, principal and staging environment.

## 7. Lab B — Compute and job inventory

~~~powershell
databricks clusters list --output json
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters get --help
databricks clusters events --help
databricks jobs get-run --help
~~~

Inventory may be permission-filtered or paginated. Verify supported CLI flags before running specific requests.

| Job | Compute ID | Node family | Workers | Autoscale range | Owner | Deadline |
|---|---|---|---|---|---|---|
| demo-batch | | | | | | |

## 8. Lab C — Run lifecycle decomposition

For one healthy and one slow authorized run:

| Phase | Healthy seconds | Slow seconds | Delta |
|---|---|---|---|
| Queue | | | |
| Provisioning | | | |
| Initialization | | | |
| Spark execution | | | |
| Output/commit | | | |
| Total | | | |

Use actual job timeline, compute events and task evidence. Label estimates when exact phase boundaries are unavailable.

## 9. Lab D — Spark UI investigation

1. Open the slow run's Spark UI, if retained.
2. Find the longest job and stage.
3. Compare task duration distribution and failed/retried tasks.
4. Inspect input bytes/records, shuffle and spill.
5. Review executor CPU, GC, memory and task imbalance.
6. Compare driver logs and cluster events.
7. Correlate with cloud CPU, disk, network and object storage metrics.

Avoid assuming a slow stage is caused by insufficient cores; it may be blocked on storage or dominated by one skewed partition.

## 10. CPU saturation versus task skew

### CPU saturation evidence

- Most workers consistently busy with runnable work.
- Stage throughput scales with available cores in controlled tests.
- CPU time is a substantial portion of task duration.
- Queue/task backlog grows under concurrent load.

### Skew evidence

- A few tasks run much longer than peers.
- Most workers idle while a small subset remains active.
- Disproportionate shuffle/input volume in straggler tasks.
- Increasing worker count has limited effect on completion time.

Averaged cluster CPU can obscure both situations.

## 11. Lab E — Offline skew assessment

~~~powershell
$seconds = @(10, 11, 11, 12, 12, 13, 14, 140)
$sorted = $seconds | Sort-Object
$median = ($sorted[3] + $sorted[4]) / 2
$max = ($sorted | Measure-Object -Maximum).Maximum
[pscustomobject]@{
  MedianTaskSec = $median
  MaxTaskSec = $max
  MaxMedianRatio = [math]::Round($max / $median, 2)
}
~~~

Synthetic long-tail evidence. Confirm skew using task input and shuffle distribution rather than duration alone.

## 12. Memory and garbage collection

Spark memory pressure can arise from:
- Large in-memory aggregations and joins.
- Skewed partitions and shuffle buffers.
- Caching without sufficient headroom.
- Executor heap, off-heap and container overhead.
- Driver-side collection or excessive metadata.
- Concurrent tasks on memory-constrained workers.

Inspect executor loss reason, OOM logs, GC duration, peak memory and spill together. Raising heap size blindly can increase GC pauses and cost.

## 13. Lab F — Offline GC pressure model

~~~powershell
$taskDurationMs = 120000
$gcTimeMs = 18000
[pscustomobject]@{
  TaskSeconds = $taskDurationMs / 1000
  GcSeconds = $gcTimeMs / 1000
  GcPct = [math]::Round(100 * $gcTimeMs / $taskDurationMs, 1)
}
~~~

Synthetic 15% GC time warrants investigation in context; there is no universal failure threshold.

## 14. Disk spill and local I/O

Spill may be a normal part of a memory-intensive stage. It becomes an operational bottleneck when correlated with high disk latency, insufficient local capacity, retries or deadline misses.

Check:
- Memory and disk spill bytes by stage.
- Local disk available capacity and IOPS.
- Shuffle fetch failures and executor losses.
- Instance family and ephemeral disk characteristics.
- Concurrent workload pressure.

Do not delete local runtime files on active production workers to recover space without a supported procedure.

## 15. Driver saturation

Symptoms:
- Job planning and scheduling slow while executors are underutilized.
- Driver CPU or heap grows disproportionately.
- Large query plans or excessive metadata.
- Driver OOM or repeated control-path failures.

Investigate driver logs, Spark plan complexity, number of tasks, metadata operations and collection patterns. Driver resizing may be appropriate only after confirming driver pressure.

## 16. Autoscaling and admission contention

Autoscaling reacts to demand but does not eliminate provisioning time, quotas, node availability or workload skew.

Investigate:
- Minimum and maximum workers.
- Scaling events and time to provision.
- Concurrent job and task limits.
- Instance pool capacity where applicable.
- Workload isolation and shared cluster contention.
- Cloud capacity and policy restrictions.

Avoid expanding autoscaling limits without confirming account quotas, budget and operational need.

## 17. Lab G — Offline capacity and queueing model

~~~powershell
$tasks = 240
$coresPerWorker = 8
$workers = 4
$slots = $coresPerWorker * $workers
$idealWaves = [math]::Ceiling($tasks / $slots)
[pscustomobject]@{
  Tasks = $tasks
  ParallelSlots = $slots
  MinimumTaskWaves = $idealWaves
}
~~~

This is a simplified upper-level capacity illustration. Real task concurrency depends on scheduler, task CPU requirements, memory, I/O, skew and other overhead.

## 18. Cloud-specific resource limits

| Area | AWS | Azure | GCP |
|---|---|---|---|
| VM capacity | EC2 availability and service quotas | VM family regional quota/allocation | Compute Engine quota/availability |
| CPU/memory | EC2 instance family | Azure VM SKU | GCE machine type |
| Local storage | Instance store/EBS characteristics | Temporary/managed disk | Local SSD/persistent disk |
| Network | VPC, security groups, endpoints | VNet, NSG, private endpoints | VPC, firewall, private access |
| Storage | S3 | ADLS Gen2 | Cloud Storage |
| Observability | CloudWatch/CloudTrail | Azure Monitor/Activity Logs | Cloud Monitoring/Logging |

Databricks compute deployment details vary by cloud, compute type and workspace networking configuration.

## 19. Lab H — Cloud context and quotas

~~~powershell
# AWS
aws sts get-caller-identity
aws configure get region
aws service-quotas list-service-quotas --service-code ec2 --max-results 20

# Azure
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus -o table

# GCP
gcloud config get-value project
gcloud compute regions describe us-central1 --format=json
~~~

Run only commands applicable to the authorized environment. Quota output may be extensive; filter and interpret only relevant regional resources. These checks do not prove actual VM availability.

## 20. Lab I — Staging load acceptance test

**Prerequisites:** representative synthetic workload, approved staging compute, budget cap, rollback and isolated schedule.

1. Record configuration, input volume and healthy baseline.
2. Run at normal concurrency; capture SLO, cost and correctness.
3. Increase synthetic concurrency by one approved step.
4. Record queue time, startup, CPU, memory, GC, spill, failures and cloud metrics.
5. Continue only while inside approved cost and safety limits.
6. Stop if error rate, correctness, cost or latency guardrail is breached.
7. Compare per-run and aggregate throughput, not just average CPU.
8. Restore original compute and scheduling configuration.
9. Verify staging resources terminate and costs are accounted for.

Do not use production datasets or trigger uncontrolled autoscaling.

## 21. Incident A — CPU-bound workload misses SLA

**Symptoms:** sustained CPU demand, growing backlog and missed deadlines.

**Triage:** compare task CPU time, worker saturation, stage parallelism and cloud capacity.

**Mitigation:** workload isolation, approved capacity adjustment or scheduling change.

**Acceptance:** deadline met at acceptable cost with validated output.

## 22. Incident B — Executor OOM and retry storm

**Symptoms:** executor losses, GC spikes, failed stages and repeated retries.

**Triage:** heap/off-heap usage, skew, memory overhead, task concurrency and driver/executor logs.

**Mitigation:** contain retry amplification, tune workload or compute safely.

**Acceptance:** no repeated OOM and no duplicate output.

## 23. Incident C — Disk spill and shuffle bottleneck

**Symptoms:** excessive spill, slow stages and disk/network contention.

**Triage:** shuffle read/write, spill, disk latency, fetch failures and executor distribution.

**Mitigation:** staging-tested layout, partition or compute changes.

**Acceptance:** lower tail latency without data regression.

## 24. Incident D — Autoscaling stalls on cloud quotas

**Symptoms:** desired worker count rises but nodes cannot provision.

**Triage:** Databricks cluster events, regional quota, VM availability, policies and network prerequisites.

**Mitigation:** approved quota/capacity correction, alternate allowed node family or scheduling.

**Acceptance:** stable provisioning within startup SLO.

## 25. Incident E — Shared cluster noisy neighbor

**Symptoms:** unrelated jobs degrade during concurrent execution.

**Triage:** overlap of job timelines, resource usage, queueing and cluster policy.

**Mitigation:** isolate critical workloads or enforce approved concurrency limits.

**Acceptance:** predictable latency and chargeback visibility.

## 26. Incident F — Low CPU but slow object storage

**Symptoms:** idle executors, blocked reads and long job duration.

**Triage:** S3/ADLS/GCS request latency, throttling, DNS/network, file layout and metadata.

**Mitigation:** correct storage/network bottleneck, not unnecessary CPU scale-out.

**Acceptance:** I/O wait and end-to-end duration improve.

## 27. Incident G — Driver OOM during planning

**Symptoms:** driver failure before substantial executor work.

**Triage:** driver heap, metadata volume, task count, query planning and logs.

**Mitigation:** reduce planning/metadata pressure or adjust approved driver sizing.

**Acceptance:** stable planning and unchanged correct results.

## 28. Troubleshooting decision tree

    Slow or failing workload
                |
      Delay before execution?
       yes -> queue / quota / provisioning
       no
                |
      Executors lost or OOM?
       yes -> memory / runtime / skew
       no
                |
      CPU busy across workers?
       yes -> CPU / parallelism / contention
       no
                |
      Few tasks dominate?
       yes -> skew / partition imbalance
       no
                |
      Spill or shuffle wait high?
       yes -> memory / disk / network
       no
                |
      External I/O wait high?
       yes -> storage / DNS / IAM
       no -> driver / plan / downstream

## 29. Production SRE runbook

1. Identify affected job, workspace, compute and business deadline.
2. Capture UTC incident timeline and last healthy baseline.
3. Split total duration into queue, startup, execution and commit.
4. Review Spark jobs, stages, tasks, executors and driver logs.
5. Correlate cloud quotas, CPU, memory, disk, network and storage.
6. Determine whether bottleneck is global saturation, skew or external wait.
7. Select least-risk mitigation and test in staging.
8. Apply approved change with explicit rollback and cost cap.
9. Validate output correctness, p95 latency, failure rate and spend.
10. Document RCA, capacity forecast and preventive monitoring.

## 30. Dashboard and alerting

Monitor:
- Job p50/p95 completion and deadline compliance.
- Queue and startup duration.
- CPU/memory/GC by node and executor.
- Failed tasks, executor loss and retries.
- Shuffle bytes, fetch failures and spill.
- Worker scaling events and cloud quota failures.
- Storage request latency and throttling.
- Cost per validated successful run.

Alert on sustained service impact or credible deadline risk. Avoid paging solely on short CPU spikes without corroborating impact.

## 31. Change and rollback controls

Before changing node family, worker count, autoscaling bounds, runtime or concurrency:
- Record configuration and baseline.
- Check cloud quota and regional availability.
- Confirm business schedule and blast radius.
- Set measurable SLO and cost acceptance criteria.
- Run a controlled staging comparison.
- Define rollback and escalation owner.
- Confirm no correctness regression.
- Verify compute termination and post-change cost.

## 32. Acceptance checklist

- [ ] Lifecycle phase breakdown documented.
- [ ] Per-executor and per-task metrics reviewed.
- [ ] CPU versus skew distinction demonstrated.
- [ ] GC, OOM and memory overhead investigated.
- [ ] Shuffle, spill and local disk evidence assessed.
- [ ] Driver pressure and scheduling contention assessed.
- [ ] Cloud quotas and provisioning events reviewed.
- [ ] Representative staging load test completed.
- [ ] Cost and correctness included in acceptance.
- [ ] Seven incident scenarios reviewed.
- [ ] Rollback, ownership and alerting documented.
- [ ] No unapproved production load testing performed.

## 33. Review questions

1. Why can adding workers fail to improve a skewed stage?
2. How can a job be slow when average CPU utilization is low?
3. What distinguishes driver OOM from executor OOM?
4. Why is spill volume alone insufficient to diagnose disk pressure?
5. How do you decide whether autoscaling, workload isolation or query changes are appropriate?

## 34. Official references

- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks compute metrics: https://docs.databricks.com/aws/en/compute/cluster-metrics
- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Apache Spark monitoring: https://spark.apache.org/docs/latest/monitoring.html
- AWS EC2 service quotas: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html
- Azure VM quotas: https://learn.microsoft.com/azure/quotas/quotas-overview
- Google Cloud quotas: https://cloud.google.com/compute/quotas
- AWS CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs

**Next:** Chapter 037 — Slow Jobs and Spark UI Interpretation.
