# Chapter 005 — Infrastructure Architecture Validation Lab

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 01 — Architecture and Platform Foundations (capstone)  
**Audience:** SRE / Cloud Infrastructure / Platform Operations  
**Clouds:** AWS, Azure, GCP  
**Risk:** Read-only inspection and tabletop failure simulation  
**Prerequisites:** Chapters 001–004

## 1. Mission and success criteria

Perform a production-style architectural readiness assessment of **one authorized Databricks workspace** without modifying it. The outcome is an evidence-backed inventory, dependency map, risk register and incident triage runbook.

You pass when you can:
- identify the correct account/workspace/region and compute execution modes;
- distinguish managed control-plane services from customer-configured infrastructure;
- inventory classic clusters, policies, jobs and SQL warehouses, while documenting serverless gaps;
- verify identity, network, storage, quotas and observability ownership;
- diagnose two tabletop failures and propose safe, testable remediation;
- produce an operational readiness scorecard and a reviewable evidence package.

## 2. Lab topology

```text
                  SRE / CLI / API
                         |
                   Identity + TLS
                         |
                Databricks workspace
                  Managed control plane
                  /                 \
         Jobs and policies       Governance
                  |
           +------+------+
           |             |
     Classic compute   Serverless compute
           |             |
  Cloud VM/network     Managed execution
           \             /
            Storage + external services
                |
        Logs / audit / cost telemetry
```

The actual deployment topology depends on cloud and workspace features. Never assume serverless nodes live in the customer network, or that another workspace is an automatically synchronized DR replica.

## 3. Scope and safety agreement

**In scope:** read-only CLI/API calls, approved cloud inventory, sanitized evidence, configuration review and tabletop exercises.

**Out of scope:** creating/deleting clusters, modifying IAM or network policies, rotating secrets, forcing outages, restarting production jobs, or cross-region failover.

**Access needed:** an authorized Databricks workspace CLI profile; read permissions to selected compute/jobs/policies; optional read-only cloud access. Missing permissions are a valid finding, not an invitation to elevate privileges.

**Data handling:** keep raw outputs in access-controlled local storage. Never commit tokens, internal hosts, workspace/account IDs, service principal names, full job definitions or sensitive configuration to public GitHub.

## 4. Deliverables

| Deliverable | Evidence |
|---|---|
| Workspace identity sheet | Sanitized workspace label, cloud, verified region, environment and owner |
| Architecture diagram | Account, workspace, compute modes and cloud dependencies |
| Resource inventory | Classic clusters, policies, jobs, SQL warehouses and serverless coverage |
| Responsibility matrix | Databricks, cloud provider and customer SRE ownership |
| Failure-domain register | Region, network, identity, compute, storage and shared dependencies |
| Incident tabletop reports | Root-cause hypothesis, evidence, safe mitigation, validation and rollback |
| Readiness scorecard | Pass / Partial / Fail / Not assessed for each control |

## 5. Hands-on lab

### Step 1 — Prepare an evidence directory

Use PowerShell on Windows. This directory is **not** part of the Git repository.

```powershell
$root = Join-Path $env:USERPROFILE "databricks-sre-ch005"
New-Item -ItemType Directory -Force -Path $root | Out-Null
$timestampUtc = (Get-Date).ToUniversalTime().ToString("o")
Write-Host "Assessment UTC: $timestampUtc"
```

Record the assessor, authorized workspace label and change/ticket reference privately.

### Step 2 — Confirm Databricks CLI identity and workspace

```powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
```

Replace the example profile with your approved profile. Verify the host and current identity before collecting inventory. If a command fails, record the exact sanitized error and stop that branch of the lab.

### Step 3 — Collect read-only resource inventory

```powershell
databricks clusters list --output json |
  Out-File (Join-Path $root "clusters.json") -Encoding utf8

databricks cluster-policies list --output json |
  Out-File (Join-Path $root "policies.json") -Encoding utf8

databricks jobs list --output json |
  Out-File (Join-Path $root "jobs.json") -Encoding utf8

databricks warehouses list --output json |
  Out-File (Join-Path $root "warehouses.json") -Encoding utf8

databricks jobs list-runs --limit 25 --output json |
  Out-File (Join-Path $root "sample-job-runs.json") -Encoding utf8
```

**Important:** inspect each command's `--help` and pagination behavior. A successful output file does not prove complete inventory. The 25 recent runs are a sample. Classic cluster listings do not capture every serverless workload. Record permission-filtered or inaccessible resources as **unknown**.

### Step 4 — Verify cloud context and regional dependencies

Use only the commands applicable to the workspace's provider:

```powershell
# AWS
aws sts get-caller-identity
aws configure get region

# Azure
az account show --query "{subscription:id,tenant:tenantId}" -o json

# GCP
gcloud config get-value project
gcloud config get-value compute/region
```

These commands establish the CLI's current cloud context, **not** authoritative proof of Databricks workspace region. Confirm the actual workspace region and networking architecture through authorized workspace/account settings and cloud resource mapping.

For the verified region, document: VM quota and capacity exposure (classic), network/DNS/egress, identity and storage access, and any private endpoints or NAT dependencies actually used.

### Step 5 — Build the ownership and dependency matrix

| Layer | Operational owner | Evidence | Health |
|---|---|---|---|
| Workspace API/control plane | Databricks + workspace administrators | CLI identity and status | Pass / Partial / Fail / Unknown |
| Authentication and RBAC | Identity / platform team | Approved access review | Pass / Partial / Fail / Unknown |
| Classic compute provisioning | Platform / cloud / Databricks | Compute events, policies, quotas | Pass / Partial / Fail / Unknown |
| Serverless execution | Databricks + workload owners | UI/APIs and service signals | Pass / Partial / Fail / Unknown |
| Network/DNS | Cloud networking | Architecture and approved checks | Pass / Partial / Fail / Unknown |
| Storage authorization | Data governance / cloud IAM | Access paths and grants | Pass / Partial / Fail / Unknown |
| Jobs and scheduling | Workload owner / platform | Jobs inventory and run evidence | Pass / Partial / Fail / Unknown |
| Observability | SRE / security / vendor | Logs, dashboards and alerts | Pass / Partial / Fail / Unknown |
| Cost and quotas | FinOps / platform | Billing, limits and guardrails | Pass / Partial / Fail / Unknown |

Do not mark a control **Pass** without evidence. Document the actual owner, not only the team category.

### Step 6 — Review configuration and blast radius

Answer these questions in the assessment:
1. Which workspaces share identity, networking, storage, governance or account-level dependencies?
2. What happens when the workspace API is reachable but classic compute cannot start?
3. Which workloads are serverless and cannot be diagnosed through customer VM listings?
4. What is the region and storage residency boundary?
5. Are RTO/RPO targets defined and tested, or merely assumed?
6. Are compute policies, ownership, auto-termination and cost controls applied consistently?
7. Can the on-call team obtain logs and provider support without emergency privilege escalation?

## 6. Tabletop incident A — Classic compute provisioning failure

**Inject (simulated):** a scheduled production job cannot start because its classic cluster remains pending. Workspace API and authentication checks succeed.

**Investigation:**
1. Confirm job ID, compute configuration, owner and UTC failure time.
2. Review compute events and job error messages.
3. Check compute policy and permissions.
4. Verify cloud quota/capacity in the actual workspace region.
5. Check network and storage prerequisites, if implicated by evidence.
6. Correlate recent infrastructure changes and provider status.

**Decision:** classify the failure as compute capacity, policy/identity, network, runtime/startup or unresolved. Avoid claiming root cause before evidence.

**Mitigation:** approved retry/backoff, an allowed compute type or rollback of a verified bad change; never silently weaken a production policy.

**Validation:** one authorized synthetic job starts and completes; related error alerts return to normal.

**Rollback:** revert the specific approved configuration change if validation fails.

## 7. Tabletop incident B — Regional outage and DR readiness

**Inject (simulated):** a cloud-region incident affects the primary workspace and a secondary workspace is reachable elsewhere.

**Investigation:**
1. Identify affected jobs, data paths and downstream consumers.
2. Check provider incidents and shared identity/network dependencies.
3. Confirm whether the secondary workspace has pre-provisioned jobs, policies, permissions and storage access.
4. Validate data replication status, checkpoints and write ownership.
5. Check actual tested RTO/RPO evidence and cutover approval.

**Critical principle:** a reachable second workspace is **not** proof of recoverability. An untested failover may duplicate writes or produce inconsistent data.

**Decision:** document whether failover is authorized and technically safe. If not, remain in incident mitigation and communicate recovery constraints.

**Validation:** use a non-production or approved synthetic workload to verify the recovery path.

## 8. Troubleshooting matrix

| Symptom | Layer to inspect | Safe action |
|---|---|---|
| CLI 401/403 | Auth profile, principal, grants | Validate identity and access |
| API timeout | DNS/TLS/network/vendor | Compare provider status and network evidence |
| Classic cluster pending | Compute policy, quota, capacity, startup | Inspect cluster events and cloud limits |
| Jobs absent from inventory | Permissions/pagination | Check scope and complete page retrieval |
| Warehouse list denied | SQL access permissions | Record gap and request authorized read access |
| Storage access denied | Cloud IAM, Unity Catalog, endpoint | Review scoped grants and logs |
| Spend unexplained by clusters | SQL/serverless/ephemeral compute | Correlate billing and workload history |
| Regional failover uncertain | DR plan and data consistency | Run tabletop; avoid live cutover |

## 9. Readiness scorecard

Use **Pass**, **Partial**, **Fail** or **Not assessed**; add an evidence reference and remediation owner.

| Control | Rating | Evidence / owner |
|---|---|---|
| Workspace identity and region verified | Not assessed | |
| Account/workspace responsibility documented | Not assessed | |
| Classic compute and policy inventory complete | Not assessed | |
| Jobs and SQL warehouse inventory complete | Not assessed | |
| Serverless visibility documented | Not assessed | |
| Identity and network dependencies mapped | Not assessed | |
| Storage permissions and residency mapped | Not assessed | |
| Cloud capacity and quotas reviewed | Not assessed | |
| Logs, audit and alerts accessible | Not assessed | |
| DR strategy and data consistency tested | Not assessed | |
| On-call escalation and runbooks current | Not assessed | |
| Cost and compute guardrails reviewed | Not assessed | |

### Exit decision

- **Ready for Part 02:** critical dependencies have owners and evidence, and major gaps have tracked remediation.
- **Conditionally ready:** no immediate safety blocker, but explicit follow-up tasks and deadlines exist.
- **Not ready:** missing critical access, undocumented failure domains, unsafe DR assumptions or unowned production dependencies.

A numeric score alone cannot override critical operational risk.

## 10. Capstone submission template

```text
Assessment: Databricks SRE Architecture Validation
Workspace label:
Cloud / verified region:
Environment:
Assessment UTC:
Assessor / reviewer:
Change or assessment ticket:

Inventory completeness:
Classic compute:
Jobs:
Policies:
SQL warehouses:
Serverless visibility:
Pagination and access gaps:

Dependency owners:
Identity:
Networking:
Storage:
Compute:
Databricks support:
On-call:

Tabletop A conclusion:
Tabletop B conclusion:
Critical risks:
Required remediations:
Readiness decision:
Evidence location (private):
Reviewer approval:
```

## 11. Cleanup

No resources were created or modified. Keep raw evidence only according to organizational retention policy. Remove local copies when no longer required, after verifying that any required audit evidence has been stored securely.

## 12. Acceptance checklist

- [ ] Read-only scope maintained throughout.
- [ ] Workspace/account/region verified from authoritative sources.
- [ ] Compute, jobs, policies and warehouses inventoried or gaps documented.
- [ ] Serverless visibility limitations explicitly recorded.
- [ ] Identity, network, storage, capacity and observability owners mapped.
- [ ] Two failure scenarios investigated with evidence-based hypotheses.
- [ ] Safe mitigation, rollback and recovery validation described.
- [ ] Readiness scorecard reviewed and remediation owners assigned.
- [ ] No credentials or sensitive identifiers committed.

## 13. References

- Databricks documentation: https://docs.databricks.com/
- Databricks CLI: https://docs.databricks.com/dev-tools/cli/
- Azure Databricks: https://learn.microsoft.com/azure/databricks/
- Databricks on GCP: https://docs.databricks.com/gcp/
- Databricks status: https://status.databricks.com/

**Part 01 complete. Next:** Chapter 006 — AWS Workspace Provisioning, beginning Part 02: Cloud Provisioning and Networking.
