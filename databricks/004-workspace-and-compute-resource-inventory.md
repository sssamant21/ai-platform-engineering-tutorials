# Chapter 004 — Workspace and Compute Resource Inventory

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 01 — Architecture and Platform Foundations  
**Audience:** SRE, Cloud Infrastructure and Platform Operations  
**Clouds:** AWS, Azure, GCP  
**Lab risk:** Read-only; no compute creation or termination

## 1. Objectives

By the end of this chapter, you will be able to:
1. Produce a repeatable inventory of Databricks workspace objects and classic compute resources.
2. Distinguish classic clusters, job compute, SQL warehouses and serverless workloads.
3. Map compute policies, ownership, runtime versions, auto-termination and resource limits.
4. Identify stale, unmanaged or expensive resources for **review**, without deleting them.
5. Build a private, sanitized inventory suitable for capacity reviews, incidents and audits.
6. Detect inventory blind spots caused by pagination, permissions and account/workspace boundaries.

## 2. Why SREs need a resource inventory

A resource inventory answers operational questions before an incident:
- Which workspaces and regions are affected?
- Who owns a cluster or job?
- Which compute is running, pending, terminated or failing?
- Which workloads use classic versus serverless compute?
- Which runtime versions, node types and policies are deployed?
- Which workloads can auto-terminate and which may incur persistent costs?
- Where are the quotas, service limits and infrastructure dependencies?

**Do not confuse a resource listing with a full platform inventory.** Workspace CLI results can be permission-filtered, paginated and limited to specific resource classes. Serverless compute and SQL warehouses need their own inventory sources.

## 3. Resource hierarchy and operational relationships

```text
Databricks account
  |
  +-- Workspace A (cloud/region)
  |     +-- Workspace configuration
  |     +-- Users, groups and service principals
  |     +-- Compute policies
  |     +-- Classic clusters / job clusters
  |     +-- Jobs and task definitions
  |     +-- SQL warehouses
  |     +-- Serverless workloads (where enabled)
  |     +-- Permissions, secrets and storage access
  |
  +-- Workspace B (cloud/region)
        +-- Independent workspace-level resources

External dependencies:
Cloud IAM, network, object storage, quotas, logs, audit and billing
```

A job can reference existing compute, create job-specific compute or use supported serverless execution. A cluster list alone cannot establish complete job-compute ownership or billing.

## 4. Inventory fields and sources

| Resource | Minimum fields | Preferred source |
|---|---|---|
| Workspace | Label, workspace ID, host, cloud, region, environment, owner | Account admin inventory / workspace settings |
| Classic cluster | ID, name, state, creator, policy, runtime, node type, workers, auto-termination | Clusters API / CLI |
| Compute policy | ID, name, permissions, restrictions, owner | Cluster Policies API / CLI |
| Job | ID, name, owner, schedule, tasks, compute reference, retry settings | Jobs API / CLI |
| SQL warehouse | ID, name, type, size, state, auto-stop, owner | SQL Warehouses API / CLI |
| Serverless workload | Type, owner, usage, limits, health and cost attribution | Relevant workload APIs, UI, system tables/billing |
| Cloud resources | Region, VM family quota, network path, IAM dependencies | Cloud console / approved cloud CLI |

Record `collected_at_utc`, collector identity, scope, pagination completion and access gaps for every inventory run.

## 5. Prerequisites and safety

- Current Databricks CLI and a named, approved authentication profile.
- Read-only workspace access; additional account privileges only when legitimately required.
- Optional cloud CLI access with least-privilege inventory permissions.
- A private output directory outside public Git repositories.
- Never commit unredacted workspace IDs, internal hostnames, principal names, secrets or full inventory JSON.
- Do not start, stop, resize, restart or delete any resource during this lab.

## 6. Hands-on lab — Workspace and compute inventory

**Objective:** collect evidence, review it and produce a sanitized inventory worksheet.

### Step 1 — Validate the active workspace

```powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
```

Replace `demo-staging` with an existing authorized CLI profile. Verify that the displayed host matches the intended workspace before proceeding.

### Step 2 — Prepare a private evidence folder

```powershell
$inventoryRoot = Join-Path $env:USERPROFILE "databricks-sre-inventory"
New-Item -ItemType Directory -Path $inventoryRoot -Force | Out-Null
$collectedAtUtc = (Get-Date).ToUniversalTime().ToString("o")
Write-Host "Collection time (UTC): $collectedAtUtc"
```

This local folder is for sensitive operational evidence; restrict its access and follow your organization's retention policy. Do not copy raw outputs into the tutorial repository.

### Step 3 — Inventory classic clusters

```powershell
databricks clusters list --output json |
  Out-File -FilePath (Join-Path $inventoryRoot "clusters.json") -Encoding utf8

databricks clusters list --help
```

Review the installed CLI's pagination options. Verify that all pages were retrieved; some commands or API responses may not return all resources in one call. Record missing permissions and any scope limitations.

**Inspect:** cluster state, creator, node type, runtime, min/max workers, autotermination and attached policy. Cluster state is a snapshot, not historical utilization.

### Step 4 — Inventory compute policies

```powershell
databricks cluster-policies list --output json |
  Out-File -FilePath (Join-Path $inventoryRoot "cluster-policies.json") -Encoding utf8
```

**Inspect:** policy coverage, approved node types, runtime restrictions, worker limits and access permissions. Do not infer a policy is enforced simply because it exists; confirm assignments and usage.

### Step 5 — Inventory jobs and recent runs

```powershell
databricks jobs list --output json |
  Out-File -FilePath (Join-Path $inventoryRoot "jobs.json") -Encoding utf8

databricks jobs list-runs --limit 25 --output json |
  Out-File -FilePath (Join-Path $inventoryRoot "recent-job-runs.json") -Encoding utf8
```

For large workspaces, use documented pagination to ensure full job inventory. A list of 25 runs is deliberately a **sample**, not complete run history.

**Inspect:** ownership, schedules, task definitions, retry behavior and whether tasks use existing, job-created or serverless compute.

### Step 6 — Inventory SQL warehouses

```powershell
databricks warehouses list --output json |
  Out-File -FilePath (Join-Path $inventoryRoot "warehouses.json") -Encoding utf8
```

If the command is unavailable or access is denied, use `databricks warehouses --help` and the approved UI/API. Record the gap instead of marking the inventory complete. Warehouse metadata does not automatically describe every serverless workload.

### Step 7 — Identify serverless inventory gaps

Use the workspace UI and authorized workload-specific APIs or system tables to document serverless jobs, SQL and other supported compute categories. Access and telemetry vary by cloud, region and feature. Do not attempt to map serverless workloads to customer-owned VMs.

### Step 8 — Verify cloud context (optional)

Use only the provider relevant to the workspace:

```powershell
# AWS identity and configured region
aws sts get-caller-identity
aws configure get region

# Azure subscription
az account show --query "{subscription:id,tenant:tenantId}" -o json

# GCP project
gcloud config get-value project
```

These commands identify your local cloud CLI context, **not** the Databricks workspace's verified deployment region or network. Cross-check with account/workspace configuration.

## 7. Build the inventory worksheet

| Field | Example (fictional) |
|---|---|
| collected_at_utc | 2026-10-13T21:00:00Z |
| workspace | demo-staging |
| cloud / region | AWS / us-east-1 |
| resource_type | classic_cluster |
| resource_id | REDACTED |
| owner | platform-team |
| state | RUNNING |
| runtime | record actual |
| node_type | record actual |
| min / max workers | 2 / 6 |
| policy | approved-general-purpose |
| auto_termination | 30 minutes |
| inventory_status | verified / partial |
| evidence_source | clusters API |
| action | review / none |

Store sensitive identifiers in an access-controlled inventory system rather than a public Markdown file.

### Inventory quality controls

- **Coverage:** compare expected workspace and workload counts with actual results.
- **Freshness:** timestamp every collection and define a refresh cadence.
- **Completeness:** capture pagination status and permissions limitations.
- **Ownership:** mark resources with missing or stale owners for follow-up.
- **Consistency:** compare runtime, node types and policy assignments to approved standards.
- **Change detection:** diff sanitized inventories over time, excluding volatile fields.

## 8. Failure exercise — Unexpected cost and unknown compute owner

**Scenario:** cloud spend increases, but the classic cluster inventory shows few running clusters.

Tabletop workflow:
1. Confirm the selected workspace, collection time and permission scope.
2. Check whether jobs create short-lived compute that is absent from the current cluster snapshot.
3. Review SQL warehouse and serverless usage separately.
4. Check billing/system usage telemetry and cloud cost attribution where authorized.
5. Correlate spend with job run history, workload owners and configuration changes.
6. Propose policy, auto-termination or scheduling improvements through change control.

**Do not** immediately terminate unidentified compute: it may support production workloads.

Expected finding: a point-in-time classic cluster listing is insufficient to explain total Databricks and cloud infrastructure spend.

## 9. Troubleshooting matrix

| Symptom | Likely explanation | Next safe step |
|---|---|---|
| Cluster list empty | No visible classic clusters or missing permissions | Verify workspace and access |
| Missing jobs | Permission filtering or pagination | Check owner scope and pagination |
| Warehouse inventory denied | Missing SQL warehouse permissions | Request approved read access |
| Cost exceeds cluster estimate | Serverless, SQL, short-lived jobs or other charges | Correlate billing/system tables |
| Resource owner unknown | Deleted user, service principal or missing metadata | Check audit/ownership records |
| Different results between runs | Resource lifecycle or collection time | Compare timestamps and run history |
| Cloud region mismatch | CLI context differs from workspace deployment | Verify authoritative workspace region |

## 10. Production runbook — Inventory drift investigation

**Trigger:** a scheduled inventory diff finds a new resource, policy change, owner change or unexpected compute configuration.

1. Validate collection completeness and source permissions.
2. Confirm that the change is real rather than a transient resource lifecycle event.
3. Identify resource owner, change request and audit event.
4. Determine blast radius: affected jobs, data access, quotas, security and cost.
5. Escalate unapproved changes to the platform owner.
6. Remediate only under approved change control; do not auto-delete unknown resources.
7. Recollect inventory and validate the expected state.
8. Document the change and any monitoring improvements.

**Rollback:** use the approved prior configuration and verify dependent workloads before reverting. Do not blindly restore a stale inventory snapshot.

## 11. Acceptance criteria

- [ ] Active workspace and identity verified.
- [ ] Classic clusters, compute policies and jobs inventoried or access gaps documented.
- [ ] SQL warehouses inventoried or access gaps documented.
- [ ] Serverless inventory sources and limitations recorded.
- [ ] Pagination/completeness checks performed.
- [ ] Sanitized worksheet contains UTC collection time and ownership.
- [ ] Cost anomaly tabletop completed with plausible evidence.
- [ ] No resources changed and no secrets or sensitive IDs committed.

## 12. Review questions

1. Why is a classic cluster list not a complete compute inventory?
2. What makes a resource inventory auditable and reproducible?
3. How do pagination and permissions create false confidence?
4. Why should SREs compare job history and billing data with current compute state?
5. Which evidence should be collected before investigating an unowned resource?

## 13. References

- Databricks CLI: https://docs.databricks.com/dev-tools/cli/
- Databricks REST APIs: https://docs.databricks.com/api/
- Azure Databricks: https://learn.microsoft.com/azure/databricks/
- Databricks on GCP: https://docs.databricks.com/gcp/
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/

**Next:** Chapter 005 — Infrastructure Architecture Validation Lab.
