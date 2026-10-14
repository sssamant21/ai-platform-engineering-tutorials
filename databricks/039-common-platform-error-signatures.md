# Chapter 039 — Common Platform Error Signatures

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 08 — Performance and Troubleshooting  
**Clouds:** AWS, Azure, Google Cloud  
**Prerequisites:** Chapters 031–038  
**Audience:** SRE, DBRE, Cloud Platform and Incident Response  
**Safety:** Production diagnostics are read-only. Changes to IAM, compute, networking, data or jobs require approval and rollback plans.

## 1. Mission

Build a repeatable process for identifying the failure domain behind Databricks error messages. Error strings are clues, not root causes. Correlate the first failure, surrounding events, job/task state, cloud metrics and recent changes before choosing remediation.

### Outcomes

- Classify errors by compute, Spark, workflow, identity, networking, storage and Delta Lake.
- Separate root errors from cascading retries and secondary symptoms.
- Use safe Windows PowerShell, Databricks CLI and SQL diagnostics.
- Distinguish Databricks-managed and customer-managed responsibilities.
- Apply cloud-specific AWS, Azure and GCP checks.
- Recover without broadening privileges, duplicating data or triggering retry storms.

## 2. Error triage architecture

    User symptom / failed job
               |
       Collect exact first error
               |
    UTC timestamp + workspace/run
               |
       Identify failure domain
       /      |      |       \
    Compute  Spark  Security  Storage
       \      |      |       /
     Correlate telemetry and changes
               |
      Validate root-cause hypothesis
               |
      Approved bounded remediation
               |
       Correctness + SLO validation
               |
          RCA and prevention

## 3. Evidence contract

Record:
- UTC first failure and detection time.
- Cloud, region, workspace, compute mode and runtime.
- Job/run/task/cluster identifiers.
- Exact sanitized error class and message.
- First failing operation and subsequent retries.
- Relevant Spark driver/executor logs.
- Cluster events, cloud audit and storage/network metrics.
- Last known good run and recent changes.
- Business impact, SLO, data freshness and correctness.
- Recovery action, approval, rollback and evidence.

Never paste bearer tokens, connection strings, signed URLs or sensitive records into tickets or public repositories.

## 4. Severity and classification

| Class | Typical signature | Primary owner |
|---|---|---|
| Compute provisioning | Instance unavailable, quota exceeded | Cloud/platform |
| Runtime initialization | Init script or library failure | Platform/workload |
| Spark execution | Executor lost, OOM, fetch failed | Workload/platform |
| Job orchestration | Run timeout, dependency failure | Workflow owner |
| Authentication | Invalid/expired token, 401 | Identity/platform |
| Authorization | Permission denied, 403 | Data/security owner |
| Network/DNS | Unknown host, timeout, TLS | Cloud networking |
| Object storage | Access denied, throttling, not found | Storage/platform |
| Delta concurrency | Concurrent write/metadata conflict | Data platform |
| SQL/warehouse | Capacity, syntax, query timeout | SQL/workload |
| Telemetry | Missing events or delayed ingestion | Observability owner |

Error text and exact exception classes vary by runtime, SDK and cloud.

## 5. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm the actual workspace URL and principal. A profile named staging can still point to the wrong host.

## 6. Lab B — Job and compute inventory

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks jobs get-run --help
databricks clusters get --help
databricks clusters events --help
~~~

Check installed CLI help before passing IDs and flags. Output can be permission-filtered and paginated.

## 7. Compute provisioning signatures

### Common symptoms

- Cluster remains pending or fails to start.
- Cloud provider reports unavailable instance type.
- Regional vCPU quota prevents worker allocation.
- Bootstrap or initialization fails.
- Policy blocks unsupported node type or configuration.

### Diagnose

1. Review cluster event timeline and first provisioning failure.
2. Check requested node family and regional quota.
3. Inspect instance availability, subnet/IP capacity and policy.
4. Review init script and library failures separately.
5. Correlate cloud provider events.

### Recovery

Use approved capacity, node type, quota or configuration remediation. Do not bypass compute policy without change approval.

## 8. Spark executor lost

A lost executor may be caused by memory exhaustion, instance termination, network failure, disk pressure or infrastructure instability.

Check executor loss reason, first error, memory/GC, disk spill, node events and cloud metrics. Repeated executor loss after retries indicates the root cause persists.

**Avoid:** unlimited retries or blindly doubling workers.

## 9. Out-of-memory signatures

Differentiate:
- Driver heap OOM during planning or collection.
- Executor heap OOM during joins/aggregations.
- Container/worker memory termination.
- Python worker memory pressure.
- Broadcast or shuffle memory pressure.

### Evidence

Spark UI executor metrics, logs, GC, skew and task concurrency.

### Recovery

Test memory-aware query/layout changes or approved sizing in staging. Verify output correctness and cost.

## 10. Shuffle fetch failure

Typical messages mention failed fetch, missing shuffle block or executor unavailable.

Investigate executor termination, network, local disk, shuffle data lifecycle and stage retries. The fetch failure may be a secondary symptom after an OOM or host loss.

**Acceptance:** no repeated fetch failures, stable executors and validated output.

## 11. Task serialization and code errors

Symptoms can include serialization errors, missing classes, unsupported libraries or Python exceptions.

Inspect the first stack trace, runtime/library version, task closure and environment. Reproduce with synthetic input in staging. Avoid changing shared runtime versions without compatibility validation.

## 12. Workflow timeout and retry exhaustion

A timed-out job may have been queued, provisioning or executing. Reconstruct phases before adjusting timeout.

Review task dependencies, retry settings, run concurrency, repair runs and downstream side effects. A retry is safe only if the operation is idempotent or duplicate effects are explicitly prevented.

## 13. Authentication failures

### 401 or invalid credentials

Investigate token expiry, wrong host, principal configuration, OAuth flow and credential rotation. Do not print token values.

~~~powershell
databricks auth describe
databricks current-user me --output json
~~~

An HTTP 401 typically indicates authentication failure; exact semantics depend on the API.

## 14. Authorization failures

### 403 or permission denied

Investigate:
- Workspace permission.
- Unity Catalog privilege.
- Service principal group membership.
- Storage credential and external location.
- Cloud IAM/RBAC or bucket policy.
- Endpoint policy and network scope.

A 403 can originate from Databricks or the cloud storage service. Identify which service returned it before changing permissions.

## 15. Unity Catalog common patterns

| Symptom | Investigation |
|---|---|
| Catalog inaccessible | USE CATALOG privilege |
| Schema inaccessible | USE SCHEMA privilege |
| Table read denied | SELECT and ancestor privileges |
| Volume access denied | Volume privileges and storage authorization |
| External location failure | Credential, external location and cloud IAM |
| Object missing | Catalog/schema resolution and workspace bindings |

Permissions depend on operation and object type. Verify exact privileges using current supported governance commands.

## 16. Lab C — SQL authorization discovery

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
~~~

Use only approved catalog names. If access is denied, record the exact failing operation and owner; do not attempt privilege escalation.

## 17. DNS and connectivity failures

| Symptom | Candidate cause |
|---|---|
| Unknown host | Resolver/private DNS/typo |
| Connection timeout | Routing/firewall/service unreachable |
| Connection refused | Listener/endpoint/proxy |
| TLS certificate error | Hostname/trust chain/proxy |
| Intermittent reset | Network, proxy or service instability |

Test from the correct compute vantage point; local workstation results do not prove Databricks compute connectivity.

## 18. Lab D — Local network checks

~~~powershell
Resolve-DnsName example.com
Test-NetConnection example.com -Port 443
curl.exe -I --connect-timeout 5 --max-time 15 https://example.com
~~~

Replace the placeholder only with an approved endpoint. Do not disable TLS verification.

## 19. Cloud object storage failures

### AWS S3

Investigate IAM role, bucket policy, endpoint policy, object key, KMS permissions and request metrics.

### Azure ADLS Gen2

Investigate identity, RBAC/ACL, DFS/Blob endpoints, storage firewall and private DNS.

### GCP Cloud Storage

Investigate service account, IAM, VPC connectivity and request metrics.

A storage 404 can reflect a wrong path or missing object; do not automatically infer eventual consistency.

## 20. Storage throttling and transient errors

Look for elevated request latency, 429/503-like service errors where applicable, retries and cloud metrics. Distinguish service throttling from application retry storms.

Use bounded exponential backoff with jitter when supported by the client. Avoid unbounded parallelism and duplicate writes.

## 21. Delta transaction conflicts

Concurrent writers can conflict when modifying overlapping files, metadata or table properties.

Investigate:
- Concurrent job/run overlap.
- Delta transaction history.
- Schema/protocol changes.
- MERGE/UPDATE/DELETE overlap.
- Retry and idempotency behavior.

Do not delete transaction logs or force a manual filesystem repair.

## 22. Lab E — Delta history inspection

~~~sql
DESCRIBE DETAIL demo_catalog.demo_staging.events;
DESCRIBE HISTORY demo_catalog.demo_staging.events;
~~~

Use an existing authorized Delta table. Compare commit timestamps with conflicting run IDs and audit events.

## 23. SQL warehouse and query failures

Separate:
- Warehouse startup/admission delay.
- Query syntax or analysis errors.
- Permissions failures.
- Execution timeout or resource contention.
- Data source/network failures.

Use query history/profile when available. Warehouse behavior and observability differ by compute mode.

## 24. Lab F — System telemetry discovery

~~~sql
SHOW SCHEMAS IN system;
SHOW TABLES IN system.access;
SHOW TABLES IN system.compute;
SHOW TABLES IN system.lakeflow;
~~~

If available:

~~~sql
DESCRIBE TABLE system.access.audit;
DESCRIBE TABLE system.lakeflow.job_run_timeline;
~~~

System tables vary by cloud, region, account and permissions. Missing telemetry is a coverage finding, not proof of no incident.

## 25. Lab G — AWS cloud context

~~~powershell
aws sts get-caller-identity
aws configure get region
aws service-quotas list-service-quotas --service-code ec2 --max-results 20
aws ec2 describe-vpc-endpoints --max-results 20
~~~

Use scoped filters for real accounts. AWS CLI identity may differ from the Databricks compute role.

## 26. Lab H — Azure cloud context

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus -o table
az network private-endpoint list --resource-group demo-staging-rg -o table
~~~

Replace example region and resource group with authorized values.

## 27. Lab I — GCP cloud context

~~~powershell
gcloud config get-value project
gcloud compute regions describe us-central1 --format=json
gcloud compute routes list
~~~

Validate project and region. Quota and network results do not establish root cause without matching compute events.

## 28. Lab J — Error signature classification exercise

Use this synthetic incident set:

| Case | Sanitized error | First suspected domain | Required corroboration |
|---|---|---|---|
| A | Executor lost, exit after OOM | Memory | Executor/host metrics |
| B | DNS name not resolved | DNS | Resolver/zone path |
| C | Storage 403 AccessDenied | Authorization | Cloud audit/IAM |
| D | Cluster provisioning quota | Cloud capacity | Regional quota/events |
| E | Concurrent write conflict | Delta concurrency | Transaction history |
| F | Shuffle fetch failed | Spark/network | Earlier executor events |
| G | Query timed out | Multiple | Queue/plan/execution |
| H | Invalid token | Authentication | Principal/host/token lifecycle |

For each case, write a falsifiable hypothesis and the evidence that would disprove it.

## 29. Lab K — Offline classification worksheet

~~~powershell
$cases = @(
  [pscustomobject]@{Error="Executor OOM";Domain="Memory";Evidence="Executor logs and GC"},
  [pscustomobject]@{Error="DNS lookup failed";Domain="DNS";Evidence="Resolver and private zone"},
  [pscustomobject]@{Error="Storage 403";Domain="IAM";Evidence="Cloud audit and principal"},
  [pscustomobject]@{Error="Quota exceeded";Domain="Capacity";Evidence="Provider quota and cluster events"}
)
$cases | Format-Table -AutoSize
~~~

Do not implement production classification using only string matching; exception chains and source services matter.

## 30. Incident A — Cluster fails after node family change

**Symptoms:** provisioning failure after approved rollout.

**Diagnose:** policy, quota, regional availability, network and cluster events.

**Recover:** roll back node type or use approved alternative.

**Validate:** startup, job correctness, cost and alert recovery.

## 31. Incident B — Repeated executor OOM

**Symptoms:** retries and executor churn.

**Diagnose:** first OOM, skew, memory overhead, join and task concurrency.

**Recover:** stop retry amplification and stage workload/compute correction.

**Validate:** no duplicate writes, stable execution.

## 32. Incident C — Service principal access revoked

**Symptoms:** 401/403 after identity change.

**Diagnose:** caller, host, grants, cloud IAM and audit trail.

**Recover:** restore narrowly scoped approved access.

**Validate:** authorized operations succeed and unauthorized operations remain denied.

## 33. Incident D — Private endpoint DNS regression

**Symptoms:** storage timeout or TLS error for one workspace.

**Diagnose:** DNS response, endpoint, route, firewall and recent change.

**Recover:** approved DNS/network rollback.

**Validate:** compute-path connectivity and job freshness.

## 34. Incident E — Delta write conflict

**Symptoms:** repeated concurrent transaction exceptions.

**Diagnose:** overlapping writers, job schedule, transaction history.

**Recover:** coordinate writers and retry safely with idempotency.

**Validate:** no missing/duplicate records.

## 35. Incident F — SQL timeout after data growth

**Symptoms:** slow scan, high shuffle, query timeout.

**Diagnose:** query profile, input growth, skew, file layout and compute.

**Recover:** approved staging-tested optimization.

**Validate:** p95, correctness and cost.

## 36. Incident G — Storage throttling and retry storm

**Symptoms:** request errors, delayed job, elevated storage latency.

**Diagnose:** provider metrics, concurrency, file counts and retries.

**Recover:** bounded backoff and workload control.

**Validate:** restored throughput and no duplicate effects.

## 37. Incident H — Missing audit events hide cause

**Symptoms:** no audit evidence for relevant change.

**Diagnose:** table availability, log pipeline, permissions and ingestion lag.

**Recover:** restore collection; use corroborating cloud audit.

**Validate:** synthetic test event delivered within defined lag.

## 38. Root-cause decision tree

    Failed job or API operation
                |
      Identify first root exception
                |
      Was provisioning successful?
        no -> quota/policy/network/init
        yes
                |
      Authentication/authorization?
        yes -> identity/grants/IAM
        no
                |
      DNS/TLS/transport failure?
        yes -> network path
        no
                |
      Storage request failure?
        yes -> cloud IAM/service/latency
        no
                |
      Spark task/executor failure?
        yes -> OOM/skew/shuffle/host
        no
                |
      Delta/SQL/workflow failure?
        yes -> transaction/plan/schedule
        no -> escalate with evidence

## 39. Production SRE incident runbook

1. Declare severity from business impact, not message wording.
2. Capture first failure, UTC timeline and run IDs.
3. Verify workspace, principal, compute and cloud context.
4. Trace error origin and exception chain.
5. Correlate cluster events, Spark UI, audit and cloud metrics.
6. Identify recent changes and shared dependencies.
7. Contain retry storms and prevent duplicate writes.
8. Choose least-risk approved mitigation and rollback.
9. Validate job completion, correctness, freshness and SLO.
10. Record RCA, preventive alert and runbook update.

## 40. Escalation matrix

| Domain | Primary team | Evidence |
|---|---|---|
| Provisioning/quota | Platform/cloud | Cluster events, quotas |
| Spark/SQL | Workload/platform | Plan, Spark UI, logs |
| IAM/Unity Catalog | Security/data governance | Grants, principal, audit |
| Networking | Cloud networking | DNS, routes, flow logs |
| Object storage | Storage/cloud | Provider errors/metrics |
| Delta conflicts | Data platform | Transaction history |
| Missing telemetry | Observability | Collector lag, coverage |

Escalate to Databricks or cloud support with sanitized timestamps, request IDs, region and reproducible evidence where available.

## 41. Prevention and alerting

Monitor:
- Cluster provisioning failures and startup latency.
- Executor losses, OOM, retries and task failure rates.
- Authentication/authorization failure trends.
- DNS/TLS/connection failure rates.
- Storage throttling, 4xx/5xx and request latency.
- Delta write conflict rate.
- SQL query timeout and deadline misses.
- Audit telemetry completeness and delay.
- Correctness and freshness SLIs.

Page for sustained impact, not merely a transient isolated error.

## 42. Acceptance checklist

- [ ] Exact first error captured and sanitized.
- [ ] Source service and failure domain identified.
- [ ] Workspace and cloud context verified.
- [ ] Job/cluster/Spark evidence correlated.
- [ ] Cloud IAM/network/storage evidence reviewed.
- [ ] Root cause separated from retry symptoms.
- [ ] Remediation and rollback approved.
- [ ] Correctness and freshness validated.
- [ ] Eight incident scenarios reviewed.
- [ ] Alert and runbook updated.
- [ ] No credentials or sensitive data leaked.

## 43. Review questions

1. Why is a shuffle fetch failure sometimes only a secondary symptom?
2. How do you distinguish a Databricks 403 from a cloud storage 403?
3. What evidence differentiates driver OOM from executor OOM?
4. Why can raising a job timeout hide a provisioning issue?
5. What validation is required after retrying a failed Delta write?

## 44. Official references

- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Databricks Delta Lake: https://docs.databricks.com/aws/en/delta/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Apache Spark monitoring: https://spark.apache.org/docs/latest/monitoring.html
- AWS service quotas: https://docs.aws.amazon.com/servicequotas/latest/userguide/intro.html
- Azure Monitor: https://learn.microsoft.com/azure/azure-monitor/overview
- Google Cloud Monitoring: https://cloud.google.com/monitoring/docs

**Next:** Chapter 040 — End-to-End Slowness Incident Lab (Part 08 capstone).
