# Chapter 051 — Infrastructure Drift Detection and Configuration Compliance

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 11 — Compliance, Observability and Operational Assurance  
**Clouds:** AWS, Azure, Google Cloud  
**Primary tools:** Terraform, Databricks CLI, GitLab CI/CD, cloud audit logs  
**Audience:** SRE, DBRE, Platform Engineering, Security, Cloud Operations  
**Prerequisites:** Chapters 041–050  
**Safety:** Production discovery and drift assessment are read-only. Never automatically apply drift remediation, import resources, force-unlock Terraform state, change grants or destroy resources in production. Execute mutation labs only in approved isolated staging.

## 1. Mission

Detect, classify, investigate and safely remediate configuration drift across Databricks workspaces, cloud infrastructure, compute policies, Unity Catalog, jobs and CI/CD-managed resources. Establish evidence-based compliance checks without letting drift automation become an uncontrolled production writer.

## 2. Learning outcomes

- Distinguish desired-state drift, unauthorized changes and intentional emergency changes.
- Use Terraform detailed exit codes and structured plan inspection.
- Compare live Databricks configuration with approved declarative definitions.
- Detect governance, identity, compute and job configuration deviations.
- Build protected scheduled GitLab drift-reporting pipelines.
- Correlate deviations with audit logs, deployment history and change tickets.
- Classify risk and choose reversion, import, migration or approved baseline update.
- Produce operational runbooks, alert thresholds and compliance evidence.

## 3. Drift concepts

**Configuration drift:** live infrastructure no longer matches the approved desired configuration.

**State drift:** Terraform state disagrees with observed infrastructure; a refresh/plan can reveal it.

**Policy drift:** effective privileges, compute rules, network boundaries or controls differ from approved requirements.

**Version drift:** provider, runtime, module or bundle versions diverge from supported/approved baselines.

**Ownership drift:** multiple controllers or operators change the same resource.

Not every difference is unauthorized. A planned change awaiting deployment also produces a Terraform diff. Confirm intent and change history before labeling a deviation an incident.

## 4. Reference architecture

    Approved Git revision + policy baseline
                     |
        Scheduled protected read-only job
                     |
       Cloud / Terraform / Databricks inventory
                     |
       Plan + API / grant / policy comparisons
                     |
        Normalize and classify deviations
                     |
         Change-ticket + audit correlation
                     |
           Alert / ticket / evidence
                     |
         Human review and remediation
                     |
         Revalidate runtime and compliance

The detector must not share an unrestricted production apply identity.

## 5. Drift coverage matrix

| Surface | Desired-state source | Detection | Risk |
|---|---|---|---|
| Cloud network/IAM/storage | Terraform + cloud policy | Terraform plan, cloud audit | Exposure, access loss |
| Databricks workspace settings | Terraform / approved baseline | Provider/API comparison | Security or availability |
| Compute policies | Terraform/policy repository | Policy inventory and diff | Cost, quota, guardrail bypass |
| Unity Catalog grants | Governed SQL/Terraform | SHOW GRANTS, effective-principal tests | Privilege escalation or outages |
| Jobs and schedules | Bundle or Terraform (single owner) | Bundle summary/API inventory | Missed or duplicate runs |
| Cluster configuration | Approved compute rules | Cluster inventory | Cost, runtime and reliability |
| GitLab deployment settings | Protected project policy | Settings/audit review | CI credential compromise |
| Monitoring and alert routes | IaC / monitoring baseline | Alert and rule inventory | Missed incidents |

Inventory can be incomplete due to permissions and pagination. A missing API result is not proof of deletion.

## 6. Ownership and reconciliation

Every managed resource must have a documented controller and owner. If Terraform and Databricks Bundles both claim a job, stop automatic reconciliation and establish one owner through an approved migration. Break-glass changes require an incident record and subsequent review: either restore the approved configuration or explicitly update desired state.

## 7. Severity classification

| Severity | Example | Action |
|---|---|---|
| Critical | Public network exposure, unexpected admin grant, production workspace mismatch | Security/incident escalation |
| High | Disabled encryption, removed essential grant, unapproved destructive plan | Block promotion, urgent review |
| Medium | Changed compute policy, altered schedule, missing mandatory tags | Ticket and planned remediation |
| Low | Cosmetic metadata difference or approved temporary exception | Record and reconcile |

Prioritize by business impact, exploitability, affected resources and evidence confidence; never treat every diff as an emergency.

## 8. Lab A — Windows PowerShell preflight [SAFE-READ]

~~~powershell
git --version
terraform version
databricks version
git status --short
git rev-parse HEAD
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm workspace host, principal, region, cloud account and Git revision against the approved target. Profile names are placeholders.

## 9. Lab B — Terraform static validation [SAFE-READ]

~~~powershell
terraform -chdir=infrastructure/environments/staging fmt -check -recursive
terraform -chdir=infrastructure/environments/staging init -backend=false
terraform -chdir=infrastructure/environments/staging validate
~~~

This verifies configuration structure but does not detect live drift.

## 10. Lab C — Read-only Terraform drift plan [CONTROLLED-READ]

From an approved staging Terraform working directory with authorized backend:

~~~powershell
terraform init
terraform plan -refresh-only -detailed-exitcode -no-color
$exit = $LASTEXITCODE
switch ($exit) {
  0 { "PASS: no refresh-only differences" }
  1 { throw "ERROR: Terraform planning failed" }
  2 { "DRIFT: remote objects differ from recorded state; investigate" }
  default { throw "Unexpected Terraform exit code: $exit" }
}
~~~

A refresh-only plan evaluates state differences without applying them. It does not prove that live infrastructure matches desired configuration. Use a **normal plan** for configuration reconciliation assessment. Do not run `terraform apply -refresh-only` automatically.

## 11. Lab D — Normal plan difference classification [CONTROLLED-READ]

~~~powershell
terraform plan -detailed-exitcode -no-color -out=drift-review.tfplan
$exit = $LASTEXITCODE
if ($exit -eq 1) { throw "Terraform plan error" }
if ($exit -eq 2) { "Differences require change and drift review" }
if ($exit -eq 0) { "No planned changes" }
terraform show -no-color drift-review.tfplan
~~~

A normal plan may show proposed configuration changes, not just drift. Correlate with commit history, recent approved changes and audit logs. The saved plan is sensitive; restrict access.

## 12. Lab E — Structured Terraform plan analysis [CONTROLLED-READ]

~~~powershell
terraform show -json drift-review.tfplan |
  Out-File -Encoding utf8 drift-review.json
$plan = Get-Content .\drift-review.json -Raw | ConvertFrom-Json
$plan.resource_changes | ForEach-Object {
  [pscustomobject]@{
    Address = $_.address
    Actions = ($_.change.actions -join ",")
  }
} | Format-Table -AutoSize
~~~

Treat JSON and state as sensitive. Inspect `resource_drift` separately from `resource_changes` when present. A replacement can be `delete,create` or `create,delete`; do not detect destruction using plain-text grep alone.

## 13. Lab F — Compare Terraform plans safely

Capture two redacted summaries from independently authorized staging plans. Compare resource address and action, provider configuration, drift reason and approved change tickets. Never compare raw sensitive plan files in public merge-request comments. Differences from data sources or computed attributes may need provider-specific interpretation.

## 14. Lab G — Databricks compute policy inventory [SAFE-READ]

~~~powershell
databricks cluster-policies list --output json
databricks clusters list --output json
~~~

For a known approved staging policy:

~~~powershell
databricks cluster-policies get "REPLACE_WITH_POLICY_ID" --output json
~~~

Review enforced limits, auto-termination, node families, runtime constraints, tags and permissions. Record stable IDs and approved policy revisions; redact sensitive configuration.

## 15. Lab H — Jobs and schedule drift [SAFE-READ]

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks bundle validate --target staging
databricks bundle summary --target staging
~~~

Compare job ID, run-as identity, task graph, schedule, pause status, retry policy and permissions against approved source. Bundle summary is not a complete inventory of all workspace jobs. Account for API pagination.

## 16. Lab I — Unity Catalog grants [SAFE-READ]

~~~sql
SELECT current_user();
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_staging;
~~~

Run as an authorized auditor. Compare expected privileges and actual effective permissions for named principals. Include inherited privileges and ownership semantics in the review; an admin query alone does not prove workload access.

## 17. Lab J — Cloud account verification [SAFE-READ]

**AWS**

~~~powershell
aws sts get-caller-identity
aws configure get region
~~~

**Azure**

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
~~~

**Google Cloud**

~~~powershell
gcloud config get-value project
gcloud auth list
~~~

Correlate cloud audit trails with the last approved deployment, IAM changes and out-of-band console actions.

## 18. Policy compliance checks

Validate:
- Approved cloud identity and Databricks service principal.
- No unauthorized public endpoints or network rule expansion.
- Encryption and storage controls remain enabled.
- Compute policies restrict cost and unsafe node types.
- Unity Catalog grants follow least privilege.
- Required ownership, environment and cost-center tags exist.
- Jobs use approved run-as principals and schedules.
- Monitoring, alerts and audit logging remain active.

A Terraform diff is not a complete security posture scan. Combine declarative checks with runtime authorization and cloud configuration evidence.

## 19. Protected GitLab scheduled drift pipeline

A minimal **static-only** example; add privileged remote-state planning exclusively in a separate protected, approved pipeline:

~~~yaml
stages:
  - validate

drift_static_check:
  stage: validate
  image:
    name: hashicorp/terraform:1.9.8
    entrypoint: [""]
  script:
    - terraform -chdir=infrastructure/environments/staging fmt -check -recursive
    - terraform -chdir=infrastructure/environments/staging init -backend=false
    - terraform -chdir=infrastructure/environments/staging validate
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
~~~

Configure schedules on protected refs, hardened runners and least-privilege read/plan identities. Pin images by approved immutable digest. Never expose production state, raw plan artifacts or tokens to untrusted merge requests.

## 20. Protected drift detection job design

1. Verify protected branch, scheduled identity, environment and backend.
2. Check account, workspace and region independently.
3. Acquire backend state lock using approved workflow.
4. Execute refresh-only plan and record exit code.
5. Execute normal plan for proposed reconciliation changes.
6. Evaluate policy and classify destructive actions.
7. Compare Databricks API inventory and governed grants.
8. Correlate change tickets, Git commits and audit events.
9. Publish **redacted** summary to an approved alert/ticket destination.
10. Exit without apply or mutation.

The drift detector must never silently accept drift into Terraform state or auto-revert production resources.

## 21. Drift report format

~~~markdown
# Databricks Configuration Drift Report
- Detection UTC:
- Environment / cloud / region / workspace:
- Git revision / policy revision:
- Terraform backend and state version (redacted):
- Detection identity:
- Resource address / resource ID:
- Expected configuration:
- Observed configuration (redacted):
- Refresh-only plan result:
- Normal plan result:
- Audit/change correlation:
- Severity / owner:
- Data/security/cost impact:
- Recommended action:
- Change ticket / exception expiry:
- Revalidation evidence:
~~~

## 22. Alerting and noise reduction

Deduplicate by resource ID, drift signature and environment. Suppress only documented, time-bounded exceptions; do not permanently ignore high-risk IAM or exposure changes. Use separate signals for failed detection jobs and confirmed drift. A detector that stops running must alert independently.

## 23. Remediation decision matrix

| Situation | Preferred response |
|---|---|
| Unauthorized manual change | Review incident and approved compensating plan |
| Legitimate emergency fix | Document and update approved desired state or revert |
| Resource exists but is unmanaged | Evaluate import/migration under change control |
| Provider/computed-only difference | Confirm provider behavior and filter only justified noise |
| Terraform/bundle conflict | Resolve controller ownership before deployment |
| Privilege escalation | Contain and escalate through security process |
| Data correctness impact | Fence writers and follow data recovery runbook |

Never use broad `ignore_changes` to conceal critical drift.

## 24. Scenario A — Manual production cluster policy change

**Detect:** policy definition differs from reviewed Terraform baseline.  
**Response:** inspect workspace audit and approved tickets; block promotion.  
**Pass:** policy restored or approved baseline updated, and runtime enforcement verified.

## 25. Scenario B — Unexpected Unity Catalog admin grant

**Detect:** grants differ from least-privilege matrix.  
**Response:** security escalation and controlled review; do not blindly revoke active access.  
**Pass:** approved privilege model restored and authorized/denied tests pass.

## 26. Scenario C — Terraform plan shows replacement

**Detect:** structured actions include delete/create.  
**Response:** investigate resource identity, state ownership and dependency impact.  
**Pass:** no unapproved replacement and fresh approved plan.

## 27. Scenario D — Provider upgrade generates noisy differences

**Detect:** mass plan changes after provider version change.  
**Response:** compare lockfile, provider schema and release notes; reproduce in staging.  
**Pass:** reviewed provider upgrade and stable post-upgrade plan.

## 28. Scenario E — Job schedule changed outside Git

**Detect:** API schedule differs from bundle definition.  
**Response:** identify owner and whether emergency pause is active.  
**Pass:** approved schedule reconciled without duplicate runs.

## 29. Scenario F — Terraform state lock contention

**Detect:** scheduled drift check cannot acquire lock.  
**Response:** check active operations and retry under policy; never automatically force-unlock.  
**Pass:** detection succeeds after legitimate writer completes.

## 30. Scenario G — Drift detector lacks visibility

**Detect:** API returns incomplete inventory or permission errors.  
**Response:** mark detection inconclusive, validate auditor identity and pagination.  
**Pass:** complete authorized scope checked, not falsely reported compliant.

## 31. Scenario H — Workspace or cloud target mismatch

**Detect:** actual host/account differs from approved scope.  
**Response:** stop before any plan or API mutation; investigate profile/federation.  
**Pass:** independently verified target.

## 32. Scenario I — Terraform and bundle manage same job

**Detect:** repeated oscillating job changes.  
**Response:** freeze reconciliation and migrate to one controller with approved procedure.  
**Pass:** stable inventory across repeated checks.

## 33. Scenario J — Emergency fix never reconciled

**Detect:** approved incident change remains out of baseline after deadline.  
**Response:** assign owner and review whether to adopt or revert.  
**Pass:** closed ticket, current desired state and clean validation.

## 34. Production drift investigation runbook

1. Verify detector health and freshness.
2. Confirm cloud account, region, workspace and principal.
3. Preserve detection timestamp, Git revision and redacted diff.
4. Distinguish refresh-only drift from normal planned changes.
5. Check GitLab deployments, incident tickets and cloud/Databricks audit logs.
6. Determine controller ownership and affected consumers.
7. Assess security, availability, cost and data-integrity impact.
8. Escalate critical drift; contain only through authorized change.
9. Create remediation ticket with owner and approved method.
10. Replan in protected environment and obtain approvals.
11. Execute staged remediation through normal release controls.
12. Verify permissions, synthetic workloads, data invariants and monitoring.
13. Record closure evidence and preventive actions.

## 35. Recovery and rollback

Terraform does not offer universal rollback. A Git revert may not undo stateful side effects. Use last-known-good configuration, a reviewed compensating plan, supported bundle redeployment and separate Delta/data restore or replay where necessary. If grants or identity changes break access, restore only the documented minimum privileges with audit trail.

## 36. Compliance evidence package

    drift-audit-YYYYMMDD/
      01-scope-and-baseline.md
      02-git-and-policy-revisions.md
      03-detector-health.md
      04-redacted-terraform-diff.md
      05-databricks-inventory-summary.md
      06-unity-catalog-access-review.md
      07-cloud-audit-correlation.md
      08-exceptions-and-approvals.md
      09-remediation-and-recovery.md
      10-post-change-validation.md
      11-owner-signoff.md

Keep raw plans, state, privileged inventory, credentials and sensitive logs in restricted approved storage.

## 37. Acceptance scorecard

| Control | Required evidence |
|---|---|
| Target isolation | Account/workspace/region verified |
| Detector identity | Least-privilege protected identity |
| Baseline integrity | Git SHA, policy and provider lock |
| Terraform drift | Refresh-only and normal plan classification |
| Structured change review | Replace/destroy detection |
| Workspace settings | Authorized API comparison |
| Compute policies | Definition and effective enforcement |
| Jobs | Schedule, owner, run-as and task comparison |
| Unity Catalog | Effective grants and negative access checks |
| Cloud security | IAM/network/encryption inventory |
| Audit correlation | Actor, time and change record |
| Alerting | Confirmed drift and detector-failure paths |
| Remediation | Approved ticket and safe execution |
| Post-change validation | Runtime, data and SLO acceptance |
| Closure | Owner, approver and evidence |

## 38. GO / CONDITIONAL GO / NO-GO

**GO:** Detector is healthy, target is verified, mandatory security controls comply and all differences are reconciled or explicitly approved.

**CONDITIONAL GO:** Only low/noncritical drift has a time-limited documented exception, accountable owner and compensating controls.

**NO-GO:** Unknown production target, unauthorized elevated access, public exposure, unapproved destruction, incomplete detector visibility, missing critical audit evidence or failed data correctness.

## 39. Review questions

1. Why does `terraform plan -refresh-only` not fully replace a normal plan?
2. How do you distinguish a pending approved change from unauthorized drift?
3. Why should drift detectors never auto-apply production changes?
4. What is the risk of having both Terraform and Bundles manage the same job?
5. Why are permission-filtered inventories insufficient for a compliance claim?
6. When is an emergency configuration change adopted rather than reverted?
7. How should detector failure be monitored separately from confirmed drift?
8. Which artifacts must remain restricted even when the summary is shared?

## 40. Official references

- Terraform plan: https://developer.hashicorp.com/terraform/cli/commands/plan
- Terraform refresh-only mode: https://developer.hashicorp.com/terraform/cli/commands/plan#refresh-only-mode
- Terraform JSON format: https://developer.hashicorp.com/terraform/internals/json-format
- Terraform state: https://developer.hashicorp.com/terraform/language/state
- GitLab scheduled pipelines: https://docs.gitlab.com/ci/pipelines/schedules/
- GitLab protected environments: https://docs.gitlab.com/ci/environments/protected_environments/
- Databricks Terraform: https://docs.databricks.com/aws/en/dev-tools/terraform/
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks compute policies: https://docs.databricks.com/aws/en/compute/policies
- Unity Catalog privileges: https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/
- Databricks audit logs: https://docs.databricks.com/aws/en/admin/account-settings/audit-logs

**Next:** Chapter 052 — Databricks Platform Observability and Compliance Monitoring.
