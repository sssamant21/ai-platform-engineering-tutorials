# Chapter 003 — Control Plane vs Compute Plane and Cloud Responsibilities

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 01 — Architecture and Platform Foundations  
**Audience:** SRE, Cloud Infrastructure, Platform Operations  
**Clouds:** AWS, Azure, GCP  
**Lab risk:** Read-only; tabletop failure exercise

## 1. Learning objectives

By the end, an SRE should be able to:
1. Distinguish the Databricks control plane, classic compute plane and serverless compute.
2. Identify the ownership boundary between Databricks, the cloud provider and the customer.
3. Map the API, identity, network, storage and compute dependencies for a workspace.
4. Isolate a workspace/API failure from a compute provisioning or workload failure.
5. Gather evidence for vendor escalation without exposing secrets.
6. Validate recovery without introducing unnecessary production changes.

## 2. Architecture: the three operating boundaries

```text
                    SRE / CI / API clients
                             |
                       Auth + DNS + TLS
                             |
                DATABRICKS CONTROL PLANE
            Workspace APIs / Jobs orchestration
            Metadata / compute configuration
                             |
            +----------------+----------------+
            |                                 |
       CLASSIC COMPUTE                   SERVERLESS COMPUTE
  Cloud infrastructure configured       Databricks-managed
  under supported workspace model       compute infrastructure
            |                                 |
    Worker/driver resources              Managed execution
    Network / identity / storage         + supported connectivity
            |                                 |
            +----------------+----------------+
                             |
             Cloud data and external services
           S3 / ADLS Gen2 / Cloud Storage / APIs
```

This diagram is conceptual. Control-plane hosting, classic compute placement, serverless connectivity and cloud account boundaries vary by provider and deployment type. Do not assume that all compute runs in a customer-managed network.

### Control plane

The control plane provides management interfaces such as workspace UI and APIs, job orchestration and configuration. Databricks operates the managed service, but customers remain responsible for their identities, permissions, policy settings, workspace configurations and integrations.

**SRE symptoms:** workspace UI/API inaccessible, authorization errors, job orchestration anomalies, API rate limits, service incidents.

### Classic compute plane

Classic compute runs supported compute resources provisioned for a workspace according to the cloud's Databricks architecture. The exact account/subscription/project and networking responsibility depends on cloud and workspace setup.

**SRE symptoms:** cluster stuck pending, driver/worker startup failure, node shortage, instance quota, DNS or outbound connectivity, initialization failures, storage authorization errors.

### Serverless compute

Serverless compute uses Databricks-managed infrastructure, with separate supported connectivity, access and billing models. Operators still manage workload permissions, configuration, policy/governance and performance/cost, but generally do not troubleshoot underlying VM/node infrastructure as if it were their own classic cluster.

**SRE symptoms:** workload startup/availability problems, unsupported connectivity, service limits, authorization errors, workload regressions.

## 3. Shared responsibility matrix

| Area | Databricks | Cloud provider | Customer SRE/platform team |
|---|---|---|---|
| Managed control-plane service | Operates service | Underlying cloud infrastructure | Access configuration, monitoring, escalation |
| Workspace identity | Supports identity integration | Cloud IAM/identity services | Principals, permissions, lifecycle, least privilege |
| Classic compute | Platform orchestration and runtime capabilities | VM, network and storage services | Policies, quotas, network/storage configuration as applicable |
| Serverless compute | Managed execution platform | Underlying provider services | Workload settings, access, connectivity and cost governance |
| Data access | Catalog/access-control capabilities | Object storage and key services | Credentials, policies, paths, data protection |
| Observability | Platform logs/telemetry capabilities | Cloud metrics and audit services | Collection, dashboards, alerts, response |
| DR | Published platform capabilities | Cloud regional capabilities | Workload/data recovery plan, RTO/RPO tests |

This is an operational summary, not a substitute for the vendor's cloud-specific shared-responsibility documentation.

## 4. AWS, Azure and GCP investigation map

| Investigation | AWS | Azure | GCP |
|---|---|---|---|
| Classic compute capacity | EC2 quotas, instance availability | VM family quotas and capacity | Compute Engine quotas and availability |
| Network | VPC, routes, NAT, security groups, endpoints | VNet, routes, NAT, NSG, private endpoints | VPC, routes, Cloud NAT, firewall, Private Service Connect where used |
| Cloud identity | IAM roles and policies | Entra ID, managed identity and RBAC | IAM roles and service accounts |
| Object storage | S3 | ADLS Gen2 | Cloud Storage |
| Cloud telemetry | CloudWatch / CloudTrail | Azure Monitor / Activity Log | Cloud Logging / Cloud Monitoring |

Use the relevant services only where they are actually part of your workspace architecture.

## 5. Prerequisites and safety

- Modern Databricks CLI installed and authenticated to an authorized workspace.
- Read-only permission to inspect jobs and compute, or a documented access limitation.
- Cloud CLI configured with least-privilege access if cloud checks are performed.
- Do not stop, resize or restart production clusters in this lab.
- Do not expose access tokens, internal hosts, account IDs or sensitive cloud configuration in committed evidence.

## 6. Hands-on lab — Trace the request-to-compute path

**Goal:** classify the health of the control-plane API, classic compute and data dependencies independently.

### Step 1 — Verify workspace authentication and API reachability

```powershell
databricks version
databricks auth describe
databricks current-user me --output json
```

**Expected:** successful identity lookup. An authentication failure is not proof of a Databricks control-plane outage. Confirm host/profile, credential validity and permissions before escalating.

### Step 2 — Inventory classic compute

```powershell
databricks clusters list --output json
databricks cluster-policies list --output json
```

Record a sanitized inventory of compute states, policies, runtime versions and relevant node types. This command does not inventory all serverless execution.

### Step 3 — Inspect jobs and run evidence

```powershell
databricks jobs list --output json
databricks jobs list-runs --limit 10 --output json
```

Check your installed CLI help if the `list-runs` arguments differ. Collect timestamps, run IDs, error category and compute configuration, and follow pagination where necessary.

### Step 4 — Read-only cloud dependency checks

Run only the section matching your environment.

**AWS:**

```powershell
aws sts get-caller-identity
aws ec2 describe-vpcs --region us-east-1 --query "Vpcs[].VpcId"
aws service-quotas list-service-quotas --service-code ec2 --region us-east-1 --max-results 20
```

**Azure:**

```powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus --output table
az network vnet list --query "[].{name:name,resourceGroup:resourceGroup}" -o table
```

**GCP:**

```powershell
gcloud config get-value project
gcloud compute regions describe us-central1 --format="json(quotas)"
gcloud compute networks list
```

Replace example regions with the **verified workspace compute region**. These commands inspect your current cloud credentials and may list unrelated resources. They do not by themselves prove that a particular VPC/VNet or quota belongs to the workspace. Correlate with the workspace architecture inventory.

### Step 5 — Build a dependency table

| Check | Result | Owner | Evidence |
|---|---|---|---|
| Workspace API and identity | Pass / Fail / Unknown | Platform IAM / Databricks | Sanitized CLI output |
| Job orchestration | Pass / Fail / Unknown | Databricks / Platform | Job run metadata |
| Classic compute provisioning | Pass / Fail / Unknown | Platform / Cloud | Compute events |
| Network/DNS/egress | Pass / Fail / Unknown | Cloud networking | Approved network diagnostics |
| Data authorization | Pass / Fail / Unknown | Data governance / IAM | Scoped error and access checks |
| Serverless availability | Pass / Fail / Unknown | Databricks / Platform | Supported workload health signals |

Never label an untested dependency as healthy.

## 7. Failure simulation — Control plane healthy, compute unavailable

**Scenario:** the Databricks workspace API returns valid results, but a new classic cluster remains pending and a scheduled job cannot start.

This is a **tabletop exercise** unless performed in an approved disposable workspace.

1. Confirm the API and identity lookup succeed.
2. Check whether the job references the intended compute policy and identity.
3. Inspect cluster events and the first actionable provisioning error.
4. Check cloud quotas and capacity in the verified region.
5. Check relevant network/DNS/storage prerequisites.
6. Compare the timeline against recent infrastructure or policy changes.
7. Propose the smallest approved remediation, then define validation and rollback.

**Expected conclusion:** a healthy workspace API does not guarantee classic compute availability. Conversely, a compute error does not prove a vendor control-plane outage.

## 8. Troubleshooting decision tree

```text
Databricks workload failing
  |
  +-- Workspace API reachable with valid identity?
  |      No -> DNS/TLS/auth/network/vendor status
  |
  +-- Job definition and permissions valid?
  |      No -> owner, grants, policy, configuration
  |
  +-- Compute type?
         |
         +-- Classic
         |     +-- provisioning events
         |     +-- cloud quota/capacity
         |     +-- node startup/runtime
         |     +-- network/storage dependencies
         |
         +-- Serverless
               +-- service availability
               +-- supported network connectivity
               +-- workload limits/permissions
               +-- Databricks support escalation
```

### Symptom-to-owner matrix

| Symptom | First investigation | Escalation |
|---|---|---|
| API 401/403 | Auth profile, token/OAuth, permissions | Identity/workspace admin |
| API timeout | DNS, TLS, client network, status | Network team / Databricks |
| Cluster pending | Compute events, quotas, node capacity | Cloud platform / Databricks |
| Driver starts but storage fails | Storage grants, credentials, endpoints | Data IAM / Storage |
| Serverless workload fails | Workload limits, connectivity, service status | Databricks support |
| Multiple regions impacted | Identity/shared dependencies and vendor status | Incident command / providers |

## 9. Production incident runbook

**Detect:** correlate alerts for API failures, compute startup duration, job failures and data access errors.

**Scope:** affected workspace, cloud/region, classic/serverless mode, workload, start time and business impact.

**Preserve evidence:** UTC timestamps, sanitized error messages, compute/job IDs, recent changes, quota signals and provider incident references.

**Diagnose:** isolate control plane, identity, classic provisioning, serverless service, network or storage. Assign an owner to each unverified dependency.

**Mitigate:** use only approved alternatives such as retry with bounded backoff, permitted compute capacity, rollback of a recent bad configuration or a validated recovery path.

**Validate:** successful authenticated API check, approved synthetic workload, stable job scheduling, expected data access and normalized alerts.

**Close:** incident timeline, root cause or known uncertainty, corrective actions, ownership, follow-up alerts and cost impact.

**Do not:** rotate production secrets, modify routing, disable security controls or force cross-region failover solely to test a hypothesis.

## 10. Acceptance criteria

- [ ] Control plane, classic compute and serverless compute differentiated.
- [ ] Cloud-specific responsibilities mapped for the actual workspace.
- [ ] Read-only Databricks API checks completed or permission gap documented.
- [ ] Classic compute and jobs inventoried without changes.
- [ ] Cloud quota/network evidence tied to the correct region.
- [ ] Tabletop failure correctly classified and escalation owner assigned.
- [ ] Recovery and rollback plan includes explicit validation.
- [ ] No secrets committed and no production changes performed.

## 11. Review questions

1. Why does a successful Databricks API call not prove cluster capacity is healthy?
2. Which checks differ between classic and serverless compute?
3. How do you separate an authorization failure from a network failure?
4. Why is the configured cloud CLI region insufficient evidence of workspace region?
5. What evidence should be included in a vendor escalation?

## 12. References

- Databricks documentation: https://docs.databricks.com/
- Databricks CLI: https://docs.databricks.com/dev-tools/cli/
- Azure Databricks: https://learn.microsoft.com/azure/databricks/
- Databricks on Google Cloud: https://docs.databricks.com/gcp/
- Databricks status: https://status.databricks.com/

**Next:** Chapter 004 — Workspace and Compute Resource Inventory.
