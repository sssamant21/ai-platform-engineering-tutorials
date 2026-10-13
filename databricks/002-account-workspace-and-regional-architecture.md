# Chapter 002 — Account, Workspace and Regional Architecture

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 01 — Architecture and Platform Foundations  
**Audience:** SRE / Infrastructure / Cloud Operations  
**Clouds:** AWS, Azure, GCP  
**Lab risk:** Read-only; no infrastructure changes

## 1. Objectives
After this chapter, an operator can:
- Distinguish Databricks account-level objects from workspace-level objects.
- Inventory workspace IDs, deployment regions, URLs and administrative ownership.
- Map regional dependencies, failure domains and data residency constraints.
- Identify which workloads use classic compute versus serverless compute.
- Triage regional or workspace-specific outages without making unsafe failover assumptions.
- Produce an auditable workspace topology and on-call escalation matrix.

## 2. Mental model: account, workspace, region

An **account** is an administrative boundary for identities, workspaces, billing/governance and other account-level features. A **workspace** is a collaboration and execution boundary with its own URL and workspace-level permissions and resources. A **region** is the cloud deployment geography associated with the workspace and its compute/data dependencies.

```text
Enterprise identity / cloud organization
               |
      Databricks account scope
       /          |          \
Workspace A   Workspace B   Workspace C
Region 1      Region 1      Region 2
   |              |             |
Classic/serverless compute as configured
   |              |             |
Cloud network / storage / identity dependencies
   \______________|_____________/
     Observability, governance, on-call
```

A workspace is not inherently a high-availability replica of another workspace. Workspaces in different regions do not automatically replicate jobs, identities, notebooks, catalogs, permissions, secrets, network settings or data. Design and validate DR explicitly.

### Account-level versus workspace-level responsibilities

| Scope | Examples to inspect | Primary operational concern |
|---|---|---|
| Account | Account admins, identities, workspace inventory, billing settings | Governance and blast radius |
| Workspace | Workspace admins, users/groups, compute policies, jobs, settings | Access and execution |
| Region | Workspace deployment region, cloud quotas, endpoints, storage geography | Residency, latency, outages |
| Compute | Classic vs serverless, node types, runtime and autoscaling | Availability, performance and cost |
| Data | Unity Catalog metastore/catalog assignments and storage paths | Authorization and data continuity |

**Cloud differences:** The account console, deployment APIs, supported features and account/workspace management patterns differ among AWS, Azure and GCP. Verify feature availability and use the correct cloud-specific documentation rather than assuming identical API or UI workflows.

## 3. Architecture decisions SREs must record

For each workspace capture:
1. Business criticality and owning team.
2. Workspace URL, workspace ID, cloud and region.
3. Environment mapping (dev/test/staging/prod) and isolation model.
4. Classic/serverless compute modes and ownership boundaries.
5. Identity provider, account/workspace admins and service principal model.
6. Private connectivity, DNS, outbound internet and storage paths.
7. Unity Catalog metastore/region dependencies, if enabled.
8. Observability, audit destinations, SLOs and on-call escalation.
9. Recovery method, RTO/RPO objectives and last validated drill.
10. Cost center, quotas and budget guardrails.

Never assume separate workspaces per environment; a shared workspace may host multiple environments with isolation controls, and that tradeoff must be documented.

## 4. AWS, Azure and GCP regional considerations

| Dimension | AWS | Azure | GCP |
|---|---|---|---|
| Cloud geography | AWS Region and availability zones | Azure region and availability zones where supported | Google Cloud region and zones |
| Network | VPC/subnets/routes/security groups | VNet/subnets/NSGs/routes | VPC/subnets/firewall/routes |
| Identity | IAM roles and Databricks identities | Entra ID, Azure roles and Databricks identities | Google Cloud IAM/service accounts and Databricks identities |
| Data storage | S3 | ADLS Gen2 | Cloud Storage |
| Capacity signals | EC2 quotas and availability | Azure VM family quotas and capacity | Compute Engine quotas and availability |
| DR considerations | Cross-region storage and configuration restoration | Cross-region storage and configuration restoration | Cross-region storage and configuration restoration |

Cloud provider AZ redundancy, Databricks-managed service redundancy, compute availability and application-level disaster recovery are different guarantees. Confirm contractual service capabilities before asserting any HA/RTO/RPO value.

## 5. Prerequisites and guardrails

- Modern Databricks CLI installed; `databricks version` succeeds.
- Approved workspace read access and an authenticated profile.
- Account admin or delegated account read permissions only if account-level inventory is needed.
- Cloud console/CLI read access if regional infrastructure must be inspected.
- Do not publish account IDs, workspace IDs, internal hosts, principal names or storage paths.
- Do not create, delete, migrate or reconfigure a production workspace in this chapter.

## 6. Hands-on lab — Build a regional workspace inventory

**Goal:** create a topology sheet and identify a regional failure boundary using read-only commands.

### Step 1 — Verify CLI and authentication

```powershell
databricks version
databricks auth profiles
databricks auth describe
databricks current-user me --output json
```

Expected: current CLI version, an active profile/host and a successful identity lookup. `auth describe` may reveal metadata; sanitize before sharing.

### Step 2 — Inspect workspace context

```powershell
# Use a named profile if multiple workspaces are configured.
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks workspace list / --output json
```

The `workspace list` command lists workspace objects at a path; it does **not** enumerate all workspaces in the account. A permission error means your identity lacks the required workspace access. Avoid interpreting an empty list as an empty account.

### Step 3 — Inventory compute and jobs in the selected workspace

```powershell
databricks clusters list --output json
databricks jobs list --output json
databricks cluster-policies list --output json
```

Classic compute listings do not provide a complete inventory of serverless usage. Check workspace UI, relevant APIs and billing/system telemetry where permitted. Handle pagination and permission filtering; save sanitized output only.

### Step 4 — Enumerate account workspaces (authorized operators only)

Account-level workspace inventory requires **account-scope authentication and authorization**. The API endpoint, account host and commands vary by cloud. Consult `databricks account workspaces --help` for your CLI version and provider; do not point a workspace-scoped token at an account endpoint.

```powershell
databricks account workspaces --help
# Run an account-level list operation only after validating
# the cloud-specific account host, profile and permissions.
```

If the account inventory is inaccessible, record the access gap and obtain an export from the account admin. Do not escalate privileges merely to complete a tutorial.

### Step 5 — Cloud region verification

Use your cloud's read-only inventory tools with authorized credentials:

```powershell
# AWS
aws sts get-caller-identity
aws configure get region

# Azure
az account show --query "{subscription:id,tenant:tenantId}" -o json
az account list-locations --query "[].name" -o tsv

# GCP
gcloud config get-value project
gcloud config get-value compute/region
```

**Caution:** These commands reveal the CLI's current cloud context, **not** proof of the Databricks workspace deployment region. Cross-check the workspace/account console and cloud resources. The Azure locations list shows available region names, not the workspace region.

### Step 6 — Produce a topology worksheet

| Field | Sample (fictional) |
|---|---|
| Workspace label | demo-staging |
| Cloud | AWS |
| Region | us-east-1 |
| URL | REDACTED |
| Workspace ID | REDACTED |
| Compute modes | Classic, serverless (verify actual) |
| Identity owner | Platform IAM team |
| Network owner | Cloud networking team |
| Storage dependency | Private S3 bucket (redacted) |
| On-call | Platform SRE |
| RTO / RPO | Not yet validated |

Repeat for every authorized workspace. Record data residency and external dependencies explicitly.

## 7. Failure exercise — Region unavailable (tabletop only)

**Scenario:** jobs fail in one region while another region's workspace remains reachable.

1. Define the blast radius: which workspaces, jobs, storage endpoints and downstream systems?
2. Determine whether the outage is Databricks-managed, cloud capacity, network, identity or data dependency.
3. Check Databricks and cloud provider status pages and correlate UTC timestamps.
4. Identify any pre-provisioned DR workspace, replicated configuration and validated storage recovery.
5. If no tested DR path exists, do **not** redirect production workloads or promise instant failover.
6. Escalate according to incident severity; capture an impact estimate and recovery decision.
7. In a disposable environment, rehearse restoring configuration and validating a minimal test job.

**Expected evidence:** dependency map, incident timeline, runbook owner, approved DR decision and recovery test result.

## 8. Troubleshooting matrix

| Symptom | Possible cause | Safe investigation |
|---|---|---|
| One workspace inaccessible | DNS, auth, service or regional problem | Compare status, DNS, authentication and peer workspaces |
| Account listing denied | Wrong account profile or insufficient role | Verify account host, identity and delegated access |
| Compute unavailable only in one region | Quota, capacity, network or policy | Inspect compute events and provider quotas |
| Cross-region access unexpectedly slow | Storage geography or network routing | Check endpoints, egress path and latency |
| Workspace UI healthy, jobs failing | Compute, policy, identity or storage | Correlate run errors and infrastructure events |
| DR workspace cannot run workloads | Missing configs, permissions or data | Validate recovery prerequisites without cutover |

## 9. Operational runbook — Regional incident

**Detect:** alert on job failure rates, compute startup failures, API health and data access symptoms.  
**Assess:** determine region, workspace IDs, affected environments, customer impact and dependencies.  
**Contain:** pause risky changes; avoid untested cutover and duplicate processing.  
**Diagnose:** compare provider status, cloud quotas, workspace API, identity, network and storage.  
**Recover:** use a previously approved DR strategy only after checking data consistency and dependency readiness.  
**Validate:** run authorized synthetic checks and minimal jobs; verify downstream data and access.  
**Close:** document actual RTO/RPO, contributing factors, timeline and corrective actions.

### Rollback and data consistency

A regional switch may cause duplicate processing, split writes or inconsistent outputs. Define a single writer, checkpoint/idempotency strategy, fencing and rollback criteria before any real failover. Do not equate a reachable backup workspace with recovered service.

## 10. Acceptance criteria

- [ ] Workspace-to-account hierarchy accurately documented.
- [ ] Region and cloud verified from authoritative workspace settings.
- [ ] Read-only CLI commands executed or access gaps recorded.
- [ ] Classic/serverless distinction captured.
- [ ] Identity, network, storage and governance dependencies mapped.
- [ ] Regional failure tabletop completed with evidence.
- [ ] DR assumptions explicitly separated from tested guarantees.
- [ ] No production resources modified and no secrets committed.

## 11. Review questions

1. What is the difference between listing workspace files and listing account workspaces?
2. Why does a workspace in another region not automatically provide DR?
3. Which components might remain available while compute startup fails?
4. What data consistency controls are necessary before cross-region failover?
5. How do you verify workspace region without confusing it with your local CLI's default region?

## 12. References

- Databricks documentation: https://docs.databricks.com/
- Databricks CLI: https://docs.databricks.com/dev-tools/cli/
- Azure Databricks: https://learn.microsoft.com/azure/databricks/
- Databricks on GCP: https://docs.databricks.com/gcp/
- Databricks status: https://status.databricks.com/

**Next:** Chapter 003 — Control Plane vs Compute Plane and Cloud Responsibilities.
