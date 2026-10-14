# Chapter 038 — Network, DNS and Storage Bottlenecks

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 08 — Performance and Troubleshooting  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Cloud Networking and Platform Engineering  
**Prerequisites:** Chapters 006–010, 026–030, 031–037  
**Safety:** Production investigation is read-only. Network, firewall, DNS, IAM and storage configuration changes require approved change control. Connectivity tests must not expose credentials or production data.

## 1. Mission

Diagnose Databricks connectivity and storage failures using layered evidence rather than treating every slow job as a Spark compute problem. Distinguish DNS, routing, private endpoints, TLS, cloud identity, object-store authorization, throughput and throttling.

### Learning outcomes

- Trace control-plane, compute-plane and storage traffic paths.
- Distinguish DNS failures, TCP timeouts, TLS errors and HTTP authorization failures.
- Diagnose AWS VPC, Azure VNet and GCP VPC private connectivity.
- Correlate S3, ADLS Gen2 and GCS latency with Spark task metrics.
- Separate application-level retries from network or storage bottlenecks.
- Build a safe, reproducible cross-cloud incident timeline.
- Validate remediations using synthetic staging workloads and rollback criteria.

## 2. End-to-end dependency architecture

    Databricks user / workflow / SQL
                    |
         Databricks control plane
                    |
         Private/public connectivity
                    |
        VPC / VNet / VPC networking
                    |
       DNS -> route -> firewall -> TLS
                    |
       Databricks compute plane
                    |
        Cloud identity / credentials
                    |
        S3 / ADLS Gen2 / GCS
                    |
       Delta metadata and data files
                    |
        Spark tasks and job outcome

Cloud deployment models differ. Serverless and classic compute may use different networking and observability paths. Confirm the actual workspace architecture before testing.

## 3. Fault-domain model

| Layer | Example symptom | First evidence |
|---|---|---|
| DNS | Unknown host, wrong address | Resolver and private DNS records |
| Routing | Connection timeout | Route tables and flow logs |
| Firewall/ACL | Refused or dropped connection | Security rules and flow logs |
| TLS | Certificate or handshake error | Trust chain, hostname, proxy |
| Identity | Access denied | Databricks identity, cloud IAM |
| Storage service | Slow/failed requests | Provider metrics and status |
| Spark | Slow reads/shuffle | Spark UI stage/task metrics |
| Data layout | Too many tiny files | File count, metadata, scan metrics |

A 403 is not a DNS failure; a timeout does not prove the storage service is down.

## 4. Establish a symptom timeline

Record:
- First and last failure UTC.
- Workspace, region, compute type and job/run IDs.
- Hostname and resource type (without secrets).
- Error category and HTTP status where available.
- DNS answers and resolver path.
- Network route, security policy and private endpoint.
- Cloud storage request latency, errors and throttling.
- Spark input/shuffle wait, retries and failed tasks.
- Recent infrastructure and identity changes.

Avoid publishing signed URLs, SAS tokens, access keys, bearer tokens or private endpoint details.

## 5. DNS troubleshooting workflow

1. Identify the exact hostname from sanitized error evidence.
2. Confirm whether it should resolve publicly or privately.
3. Query the configured resolver from an authorized network vantage point.
4. Compare the returned address with expected endpoint architecture.
5. Inspect private DNS zone links, forwarding and split-horizon behavior.
6. Confirm TTL and recent DNS changes.
7. Test transport only after resolution is correct.

DNS success does not prove routing, TLS or authorization.

## 6. Lab A — Windows PowerShell DNS checks

~~~powershell
Resolve-DnsName example.com
Test-NetConnection example.com -Port 443
Get-DnsClientServerAddress
~~~

These test the local workstation, not necessarily Databricks compute. Replace example.com with an approved non-sensitive endpoint only after confirming the correct network vantage point.

## 7. Lab B — HTTPS and TLS diagnostics

~~~powershell
curl.exe -I --connect-timeout 5 --max-time 15 https://example.com
Test-NetConnection example.com -Port 443
~~~

A successful TCP connection does not prove successful TLS or authorized object access. Never disable certificate validation as a production workaround.

## 8. Lab C — Databricks workspace preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks clusters list --output json
databricks jobs list-runs --limit 25 --output json
~~~

Verify host, principal and compute environment before attributing a local connectivity result to a workspace problem.

## 9. AWS networking and S3 path

Review:
- VPC subnet and route tables.
- Security groups and network ACLs.
- NAT/egress configuration where used.
- S3 gateway or interface endpoints as applicable.
- Private DNS and resolver forwarding.
- IAM roles, bucket policies and endpoint policies.
- CloudWatch storage and EC2 network metrics.

A VPC endpoint policy can deny an otherwise authorized principal. Cloud-specific S3 endpoint design depends on region and architecture.

## 10. Lab D — AWS read-only inventory

~~~powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-vpc-endpoints --max-results 20
aws ec2 describe-route-tables --max-results 20
aws ec2 describe-security-groups --max-results 20
~~~

Limit output to authorized VPCs using filters in real environments. Do not assume the workstation's AWS identity matches the Databricks compute role.

## 11. Azure networking and ADLS Gen2 path

Review:
- VNet and subnet routes.
- NSGs, firewalls and private endpoints.
- Private DNS zone links and storage subresource names.
- Managed identity/service principal and RBAC.
- Storage account firewall and hierarchical namespace.
- Blob/DFS endpoint DNS and network paths.
- Azure Monitor storage latency, throttling and errors.

An ADLS operation can involve both DFS and Blob endpoints. Validate all required endpoints for the operation.

## 12. Lab E — Azure read-only inventory

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az network private-endpoint list --resource-group demo-staging-rg -o table
az network private-dns zone list --resource-group demo-staging-rg -o table
az network nsg list --resource-group demo-staging-rg -o table
~~~

Use authorized resource groups; examples are placeholders. Avoid exporting sensitive resource configuration into public issue trackers.

## 13. GCP networking and Cloud Storage path

Review:
- VPC routes and firewall rules.
- Private Google Access or Private Service Connect where relevant.
- DNS policy, forwarding and private zones.
- Cloud NAT and egress behavior if applicable.
- Service account identity and bucket IAM.
- Cloud Storage request latency, error and quota metrics.
- Compute Engine network and instance health.

Do not assume a Google-managed API endpoint behaves like a privately hosted VM service.

## 14. Lab F — GCP read-only inventory

~~~powershell
gcloud config get-value project
gcloud compute networks list
gcloud compute routes list
gcloud compute firewall-rules list
gcloud dns managed-zones list
~~~

Results reflect the configured project and caller permissions. Confirm project and region before investigating a specific Databricks workspace.

## 15. Object storage performance model

Object-store performance depends on request patterns, object size, concurrency, network path, client retries and service behavior.

| Symptom | Candidate cause | Validate with |
|---|---|---|
| High request count | Many small files | Delta/file inventory |
| Slow reads | Network or storage latency | Cloud metrics and Spark input time |
| Throttling | Request limits or burst pattern | Provider errors and retry metrics |
| 403 access denied | IAM/policy/credential issue | Audit and identity evaluation |
| 404 not found | Path/version/consistency assumption | Object metadata and transaction state |
| Slow commit | Metadata, transaction contention | Delta history and commit errors |

Do not interpret object storage like a block device with guaranteed local disk latency.

## 16. Spark evidence for storage bottlenecks

Use Spark UI to inspect:
- Scan input bytes and records.
- Task duration and executor CPU time.
- Remote read/fetch wait.
- Retry count and error signatures.
- File scan and metadata overhead.
- Shuffle versus object-store I/O.
- Per-task skew and executor distribution.

Low CPU plus long task wall time may indicate I/O wait, but requires corroboration.

## 17. Lab G — Read-only SQL metadata checks

~~~sql
SHOW SCHEMAS IN system;
SHOW TABLES IN system.compute;
SHOW TABLES IN system.access;
~~~

For an approved staging Delta table:

~~~sql
DESCRIBE DETAIL demo_catalog.demo_staging.events;
DESCRIBE HISTORY demo_catalog.demo_staging.events;
~~~

Table names are examples. Check that the table exists and that the caller is authorized. File count, size and transaction history can help distinguish layout issues from networking.

## 18. Lab H — Synthetic request-latency analysis

~~~powershell
$latencyMs = @(45, 48, 52, 51, 47, 60, 58, 420, 390, 55)
$sorted = @($latencyMs | Sort-Object)
$avg = ($latencyMs | Measure-Object -Average).Average
$p95Index = [math]::Ceiling(0.95 * $sorted.Count) - 1
[pscustomobject]@{
  Samples = $sorted.Count
  MeanMs = [math]::Round($avg, 1)
  P95NearestRankMs = $sorted[$p95Index]
  MaxMs = $sorted[-1]
}
~~~

The small synthetic sample is only for demonstrating percentile mechanics. Production alerting needs representative volume and a documented aggregation window.

## 19. Lab I — Cross-layer timing worksheet

| Stage | Baseline ms | Incident ms | Evidence |
|---|---|---|---|
| DNS resolution | | | |
| TCP connect | | | |
| TLS handshake | | | |
| Storage first byte | | | |
| Object download | | | |
| Spark task compute | | | |
| Delta commit | | | |

These timings may not be individually exposed by every Databricks compute mode. Do not fabricate measurements; label estimates and source limitations.

## 20. Lab J — Controlled staging comparison

**Preconditions:** approved staging storage, synthetic dataset, fixed compute, network owner, cost cap and rollback plan.

1. Record baseline network/storage topology.
2. Capture healthy job duration, bytes read and request errors.
3. Capture DNS and private endpoint configuration from approved vantage points.
4. Run identical synthetic workload at least three times.
5. Change one approved variable only.
6. Repeat while controlling cache and concurrency.
7. Compare p50/p95 job latency, request metrics, retries and cost.
8. Verify output correctness and security controls.
9. Restore the original configuration if degraded.
10. Preserve sanitized evidence and record residual risk.

Never weaken bucket/container permissions to make a benchmark pass.

## 21. Incident A — Private DNS resolves to wrong endpoint

**Symptoms:** unexpected public/private address, timeout or TLS hostname mismatch.

**Triage:** resolver path, private zone links, forwarding and endpoint configuration.

**Recovery:** restore approved DNS association or record; respect propagation and TTL.

**Acceptance:** correct resolution from compute vantage point and authorized connectivity.

## 22. Incident B — Firewall blocks required egress

**Symptoms:** TCP timeout to approved service with correct DNS.

**Triage:** effective route, NSG/security group/firewall, flow logs and recent policy change.

**Recovery:** add only the narrowly scoped approved rule.

**Acceptance:** required traffic succeeds; prohibited traffic remains denied.

## 23. Incident C — S3/ADLS/GCS access denied

**Symptoms:** 403 errors after identity or policy change.

**Triage:** principal, credential expiration, object policy, endpoint policy, Unity Catalog configuration and audit logs.

**Recovery:** restore least-privilege authorization, not public access.

**Acceptance:** authorized read/write succeeds and unauthorized action fails.

## 24. Incident D — Storage throttling increases job time

**Symptoms:** elevated retry count, request errors and read latency.

**Triage:** provider metrics, request concurrency, file layout and client behavior.

**Recovery:** approved request-pattern or workload adjustment, service-limit remediation where applicable.

**Acceptance:** request errors and end-to-end latency normalize.

## 25. Incident E — TLS handshake failure

**Symptoms:** certificate verification error, handshake reset or proxy incompatibility.

**Triage:** hostname, trust chain, TLS policy, proxy and recent certificate changes.

**Recovery:** correct certificate/trust or proxy configuration through approved process.

**Acceptance:** validated TLS connection without bypassing verification.

## 26. Incident F — Many small files amplify I/O overhead

**Symptoms:** high metadata/request count and slow scan despite modest bytes.

**Triage:** Delta file inventory, scan plan, object request rate and task scheduling.

**Recovery:** staging-tested file compaction/layout or ingestion design correction.

**Acceptance:** fewer unnecessary requests, correct output and acceptable cost.

## 27. Incident G — Regional networking degradation

**Symptoms:** several jobs fail simultaneously with network timeouts.

**Triage:** cloud status, network flow metrics, shared endpoints and affected regions.

**Recovery:** follow approved failover or provider escalation; avoid unsafe ad hoc rerouting.

**Acceptance:** service SLO and storage access restored; failover data consistency verified.

## 28. Incident H — Endpoint policy regression

**Symptoms:** access failure only from a particular private route or workspace.

**Triage:** endpoint policy, route selection, cloud IAM and recent changes.

**Recovery:** revert the specific approved policy change.

**Acceptance:** affected workload restored without broadening permissions.

## 29. Troubleshooting decision tree

    Databricks job slow / storage error
                   |
           DNS resolution valid?
           no -> resolver / private DNS
           yes
                   |
           TCP route reachable?
           no -> route / firewall / NAT
           yes
                   |
           TLS handshake valid?
           no -> cert / proxy / policy
           yes
                   |
           Authorized request succeeds?
           no -> IAM / endpoint policy
           yes
                   |
           Storage request latency high?
           yes -> throttling / request layout
           no
                   |
           Spark shuffle or compute wait?
           yes -> Spark / executor diagnosis
           no -> deeper service investigation

## 30. Production network/storage incident runbook

1. Identify affected service, workspace, region, compute and business deadline.
2. Capture sanitized error and UTC incident timeline.
3. Determine DNS, transport, TLS, identity or storage performance failure.
4. Validate the correct compute network vantage point.
5. Review recent network, DNS, IAM and storage changes.
6. Correlate Spark task metrics with cloud provider telemetry.
7. Select least-privilege, least-blast-radius remediation.
8. Test in staging where feasible; obtain production approval.
9. Validate data correctness, freshness, latency and denied-access boundaries.
10. Record RCA, rollback, cost impact and monitoring improvements.

## 31. Monitoring and alerts

- DNS failure rate and resolution anomalies.
- Network connection timeout/reset rate.
- TLS handshake failures.
- Storage 4xx/5xx and throttling trends.
- Object storage request latency p95/p99.
- Spark input wait, retries and scan duration.
- Delta metadata/file count growth.
- Private endpoint and route change audit events.
- Business workflow deadline and correctness SLOs.

Page on sustained user-impacting failures; avoid paging on isolated transient requests without service impact.

## 32. Change management and rollback

Before modifying network or storage configuration:
- Record existing route, DNS, policy and endpoint state.
- Identify all affected workspaces, regions and workloads.
- Verify cloud IAM and least-privilege requirements.
- Establish test case and expected result.
- Define rollback owner and propagation timing.
- Preserve audit evidence.
- Confirm both authorized success and unauthorized denial after change.
- Validate SLO, cost and correctness.

## 33. Acceptance checklist

- [ ] Network and storage architecture mapped for actual compute type.
- [ ] DNS resolver and private endpoint behavior verified.
- [ ] Routing and firewall rules reviewed.
- [ ] TLS and certificate validation preserved.
- [ ] Cloud identity and storage permissions correlated.
- [ ] Spark I/O versus compute bottleneck distinguished.
- [ ] Storage latency and throttling metrics captured.
- [ ] Staging benchmark and rollback validated.
- [ ] Eight incident scenarios reviewed.
- [ ] Security and data correctness acceptance completed.
- [ ] No production credentials or signed URLs exposed.

## 34. Review questions

1. Why can successful DNS resolution coexist with a storage timeout?
2. How do you distinguish a 403 authorization failure from an endpoint routing problem?
3. Why must connectivity be tested from the Databricks compute vantage point?
4. What evidence separates slow object storage from expensive Spark shuffle?
5. Why can small files increase job time even when total data size is unchanged?

## 35. Official references

- Databricks network connectivity: https://docs.databricks.com/aws/en/security/network/
- Databricks compute metrics: https://docs.databricks.com/aws/en/compute/cluster-metrics
- Databricks Unity Catalog storage: https://docs.databricks.com/aws/en/connect/unity-catalog/
- AWS VPC endpoints: https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html
- AWS S3 performance: https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html
- Azure Private Endpoint: https://learn.microsoft.com/azure/private-link/private-endpoint-overview
- Azure Storage monitoring: https://learn.microsoft.com/azure/storage/common/monitor-storage
- Google Private Google Access: https://cloud.google.com/vpc/docs/private-google-access
- Google Cloud Storage monitoring: https://cloud.google.com/storage/docs/monitoring
- Apache Spark monitoring: https://spark.apache.org/docs/latest/monitoring.html

**Next:** Chapter 039 — Common Platform Error Signatures.
