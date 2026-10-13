# Chapter 012 — Cluster Policies, Node Types and Runtime Governance

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 03 — Compute Infrastructure and Operations  
**Audience:** SRE, Cloud Platform, Databricks Administrators  
**Lab mode:** Read-only inspection, offline policy validation and approved staging rollout  
**Prerequisite:** Chapter 011

## 1. Learning objectives

After this chapter you should be able to:
1. Explain compute policy enforcement, permissions and default settings.
2. Assess node families, worker counts, autoscaling and cloud quota constraints.
3. Govern Databricks Runtime versions, LTS upgrades and compatibility.
4. Detect policy drift and configuration exceptions.
5. Safely propose, stage, validate and roll back compute policy changes.
6. Diagnose policy rejections, compute capacity failures and runtime regressions.

## 2. Governance architecture

~~~text
Platform standards / Security / FinOps
                 |
       Approved compute policies
                 |
        Workspace entitlements
        and policy permissions
                 |
     +-----------+------------+
     |                        |
Interactive compute       Jobs compute
     |                        |
     +-----------+------------+
                 |
       Node family / runtime
       autoscaling / limits
                 |
       Cloud capacity and IAM
                 |
       Metrics / logs / billing
~~~

Policies constrain eligible configurations; they do not by themselves guarantee isolation, availability, cost savings or data authorization. Workspace privileges, cloud controls, Unity Catalog permissions and network restrictions remain separate.

## 3. What policies govern

Databricks compute policies can constrain or default supported compute configuration attributes such as:
- node type and driver node type;
- minimum and maximum worker counts;
- autoscaling or fixed worker count;
- Databricks Runtime selection;
- autotermination;
- instance pools where supported;
- tags and supported security settings.

Policy definition syntax and supported attributes vary with the current Databricks product and API. Always validate against the installed CLI and official documentation.

### Policy rule types

| Rule type | Purpose | SRE consideration |
|---|---|---|
| fixed | Enforce an exact value | Strong standardization; can block legitimate workloads |
| allowlist | Permit a reviewed set | Maintain approved node/runtime catalog |
| range | Constrain numeric values | Bound capacity and spend |
| forbidden | Prohibit an attribute | Validate fallback behavior |
| unlimited | Expose value with optional default | Not a security boundary by itself |

Policy rules can also include defaults and UI visibility options where supported. A default does not necessarily enforce a value; distinguish user-selectable defaults from hard restrictions.

## 4. Node type selection and cloud differences

| Dimension | Operational impact |
|---|---|
| vCPU and memory | Executor sizing, GC and concurrency |
| Local disk | Shuffle spill, temporary storage and I/O bottlenecks |
| Instance family | CPU/memory/storage optimized tradeoffs |
| Availability | Region, zone, SKU and cloud capacity |
| Quota | Cloud limits independent of Databricks policy |
| Spot/preemptible | Lower cost, higher interruption risk |
| Driver sizing | Scheduling, metadata and orchestration bottlenecks |
| Network bandwidth | Storage throughput and shuffle |

**AWS:** EC2 instance family, capacity, availability zone, instance profile and VPC prerequisites.

**Azure:** VM SKU, regional quota, VNet injection/egress and managed identity prerequisites.

**GCP:** Compute Engine machine types, project quotas, regional capacity and service account/network prerequisites.

A node type appearing in a workspace API does not prove that it can be provisioned at the required time.

## 5. Runtime governance

Maintain a supported runtime baseline by workload class:
1. Prefer supported LTS runtimes where compatibility permits.
2. Track Spark, Scala, Python, connector and library compatibility.
3. Test init scripts, instance profiles/identities, data permissions and network access.
4. Validate workloads against a staging candidate runtime.
5. Roll out in waves and observe job/SQL SLOs.
6. Retain a rollback-compatible runtime and artifact set during the change window.

Do not blindly upgrade production clusters based solely on a runtime appearing in the list. Runtime availability and support windows change.

## 6. Lab A — Read-only CLI inventory

Use an approved Windows PowerShell session. Verify profile and workspace:

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Replace the profile with an authorized workspace. Do not export credentials.

### Inventory policies and node types

~~~powershell
databricks cluster-policies list --output json
databricks clusters list-node-types --output json
databricks clusters spark-versions --output json
~~~

These are workspace-specific results and may be permission filtered. The Spark versions API output is a list of available versions, not a substitute for checking support lifecycle and library compatibility.

### Inspect one approved policy

~~~powershell
databricks cluster-policies get --help
~~~

Use the installed CLI's documented policy ID argument. Review its definition, ownership, permissions and references to jobs/clusters. Do not edit the policy during the read-only lab.

### Inspect a cluster and events

~~~powershell
databricks clusters list --output json
databricks clusters get --help
databricks clusters events --help
~~~

Select an authorized cluster and record node type, driver node type, runtime, autoscale bounds, policy ID, access mode, tags and autotermination. Preserve sanitized output in restricted storage.

## 7. Lab B — Offline policy design exercise

The following **illustrative** policy JSON demonstrates fixed runtime constraints, allowed node families and bounded autoscaling. Verify field paths, rule support and runtime IDs in your target workspace before any API submission.

~~~json
{
  "spark_version": {
    "type": "allowlist",
    "values": ["<approved-runtime-id-1>", "<approved-runtime-id-2>"]
  },
  "node_type_id": {
    "type": "allowlist",
    "values": ["<approved-node-type-1>", "<approved-node-type-2>"]
  },
  "autoscale.min_workers": {
    "type": "range",
    "minValue": 1,
    "maxValue": 4
  },
  "autoscale.max_workers": {
    "type": "range",
    "minValue": 2,
    "maxValue": 12
  },
  "autotermination_minutes": {
    "type": "range",
    "minValue": 10,
    "maxValue": 120
  }
}
~~~

**Important:** This is a policy design sample, not a production-ready policy. It does not enforce a relationship between min_workers and max_workers on its own, and may not apply to serverless compute or SQL warehouses. Exact fields and supported rules must be validated.

### Validate JSON syntax locally

~~~powershell
Get-Content .\staging-policy.json -Raw | ConvertFrom-Json | Out-Null
Write-Host "JSON syntax parsed; policy semantics not yet validated"
~~~

Do not confuse valid JSON with an accepted Databricks policy.

## 8. Lab C — Safe staging rollout procedure

1. Export the existing policy definition and permission assignments into restricted change evidence.
2. Identify all dependent clusters, jobs, owners and workflows.
3. Create a **new staging policy**, rather than editing a widely shared production policy.
4. Validate policy schema using current Databricks CLI/API help.
5. Apply policy permissions only to the approved test group.
6. Create a disposable cluster or test job using the new policy.
7. Verify allowed node/runtime combinations and expected rejection cases.
8. Record startup latency, runtime health, permissions, network/storage access and cost.
9. Compare with baseline, obtain approval and roll out to a small production cohort.
10. Preserve rollback instructions and monitor for regressions.

For actual mutation commands, use:

~~~powershell
databricks cluster-policies create --help
databricks cluster-policies edit --help
databricks cluster-policies delete --help
~~~

Do not run create/edit/delete in production as part of this tutorial.

## 9. Policy drift detection

Compare three layers:
- **Declared policy:** approved definition and version in source control.
- **Effective workspace policy:** current API definition and permissions.
- **Effective compute:** existing cluster and job configurations, including historical exceptions.

Drift can arise from manual edits, stale jobs, ungoverned compute or inconsistent policy assignment. Existing compute may not immediately adopt changed defaults; verify behavior instead of assuming automatic reconciliation.

### Drift report template

| Policy | Approved version | Workspace version | Impacted workloads | Risk | Owner |
|---|---|---|---|---|---|
| staging-interactive | v2 | v2 | 3 test clusters | None observed | Platform |
| prod-jobs | v5 | v4 | 12 jobs | Review required | SRE |

Example values are fictional. Record evidence links privately.

## 10. Failure scenario A — Policy rejection

**Symptom:** User cannot create or resize a cluster.

**Check:** selected policy, permitted node types, runtime ID, autoscale bounds, hidden fixed fields, permissions and requested configuration.

**Mitigation:** correct the requested configuration or propose an approved policy exception. Do not grant unrestricted cluster creation to bypass the failure.

**Validation:** authorized configuration succeeds and disallowed configuration remains blocked.

## 11. Failure scenario B — Capacity shortage

**Symptom:** Configuration passes policy validation but compute remains pending.

**Check:** cloud region/zone, VM family quotas, instance availability, spot capacity, pool availability, network and identity prerequisites.

**Mitigation:** approved fallback instance family or quota request, respecting policy and workload requirements.

**Validation:** staging workload starts and completes with acceptable performance.

## 12. Failure scenario C — Runtime upgrade regression

**Symptom:** Job errors rise after a runtime upgrade.

**Check:** runtime release changes, Spark/Python/Scala compatibility, connectors, init scripts, access mode, library conflicts and external dependencies.

**Mitigation:** roll back to the last approved compatible runtime where supported, or deploy a tested compatibility fix.

**Validation:** critical jobs and synthetic data-access checks pass.

## 13. Failure scenario D — Cost and idle compute drift

**Symptom:** Compute spend rises after a policy change.

**Check:** worker bounds, driver sizing, instance pools, auto-termination, workload concurrency and tags.

**Mitigation:** compare utilization and job SLOs before rightsizing. Do not lower limits blindly.

**Validation:** spend decreases without introducing job failures or unacceptable queueing.

## 14. Troubleshooting matrix

| Symptom | Boundary | Evidence |
|---|---|---|
| Policy permission denied | Databricks ACL | Policy permissions and principal |
| Invalid policy definition | Policy schema | API validation error |
| Node type rejected | Policy allowlist | Requested versus allowed type |
| Runtime rejected | Policy/runtime | Allowed IDs and runtime catalog |
| PENDING compute | Cloud capacity/network | Cluster events and quota |
| Startup ERROR | Runtime/init scripts/IAM | Driver logs and events |
| Cost spike | Policy/capacity usage | Billing and utilization |
| Workload regression | Runtime or node family | Baseline and staging comparison |

## 15. Operational monitoring

Track:
- Policy adoption by workload and owner.
- Number of unmanaged/exception clusters.
- Runtime version distribution and unsupported versions.
- Provisioning failure rate by node type and region.
- Cluster startup latency, autoscaling events and driver failures.
- Idle hours, compute spend and cost per workload.
- Runtime upgrade regressions and rollback frequency.

Use baselines and service objectives rather than fixed universal alert thresholds.

## 16. Production change and rollback runbook

1. Capture existing policy definition, permissions and dependent workloads.
2. Document proposed rules, cloud availability and runtime support.
3. Review blast radius, security, FinOps and exception process.
4. Validate JSON/schema and deploy a new staging policy.
5. Test allowed/rejected configurations and critical workloads.
6. Roll out to a small cohort with monitoring and a change owner.
7. If regression occurs, restore previous approved policy assignment/definition and compatible compute configuration.
8. Verify job health, access controls and cost signals after rollback.
9. Close with evidence, owner and follow-up actions.

Rollback may require restarting or replacing compute; a policy definition rollback alone may not restore an already-changed running cluster.

## 17. Acceptance checklist

- [ ] Authorized workspace and CLI identity verified.
- [ ] Policies, node types and runtime versions inventoried.
- [ ] Approved workload isolation and access modes documented.
- [ ] Offline policy JSON parsed and semantics reviewed.
- [ ] Cloud quota and capacity dependencies identified.
- [ ] Runtime compatibility and upgrade plan reviewed.
- [ ] Four failure scenarios completed.
- [ ] Staging rollout and rollback runbook approved.
- [ ] No production policies or compute modified.
- [ ] No tokens, raw inventory or Terraform state committed.

## 18. Review questions

1. Why does an approved node type not guarantee compute availability?
2. What is the difference between a policy default and a fixed restriction?
3. Why can existing clusters remain noncompliant after a policy edit?
4. What must be tested before promoting a new Databricks Runtime?
5. Why is restoring a policy definition sometimes insufficient for rollback?

## 19. References

- Databricks compute policies: https://docs.databricks.com/aws/en/admin/clusters/policy-definition
- Databricks compute: https://docs.databricks.com/aws/en/compute/
- Databricks Runtime: https://docs.databricks.com/aws/en/release-notes/runtime/
- Azure Databricks compute policies: https://learn.microsoft.com/azure/databricks/admin/clusters/policy-definition
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 013 — Autoscaling, Instance Pools and Capacity Management.
