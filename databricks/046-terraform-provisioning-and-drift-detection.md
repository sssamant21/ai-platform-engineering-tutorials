# Chapter 046 — Terraform Provisioning and Drift Detection

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 10 — Automation and GitOps  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, Cloud Platform, Infrastructure Engineering, Security and Change Management  
**Prerequisites:** Chapters 006–010, 016–020, 041–045  
**Safety:** All applies and destructive simulations are restricted to approved isolated staging environments. Production workflows are plan-only unless change approval, peer review, rollback and state protection are complete.

## 1. Mission

Operate Databricks infrastructure as code using Terraform, detect configuration drift, and restore intended state without accidental destruction or privilege escalation. Treat cloud networking, identity, Databricks account/workspace resources and monitoring as one controlled lifecycle.

### Learning outcomes

- Understand cloud and Databricks provider responsibilities.
- Design modules and environment-specific state boundaries.
- Pin providers, protect remote state and avoid secrets exposure.
- Validate and review plans before approved changes.
- Detect out-of-band drift and separate it from legitimate configuration updates.
- Establish GitOps pull-request, policy and approval controls.
- Reconcile drift safely and verify production outcomes.
- Apply cross-cloud AWS/Azure/GCP operational checks.

## 2. Architecture and source of truth

    Git repository / pull request
                 |
      Formatting and validation
                 |
      Security and policy checks
                 |
       Terraform plan review
                 |
        Human approval gate
                 |
     Approved staging/prod apply
                 |
       Cloud + Databricks APIs
                 |
       Observability and drift
                 |
          Remediation PR

Git defines desired configuration. Terraform state maps that configuration to real objects. Neither a clean plan nor successful apply proves application correctness; run post-change acceptance tests.

## 3. Databricks Terraform boundaries

Terraform can manage supported resources such as jobs, policies, permissions, catalogs and selected account/workspace objects. Cloud-specific providers manage VPC/VNet, storage, IAM, private endpoints and related infrastructure.

Some account-level operations require a different API host and permissions from workspace-level operations. Confirm the selected provider resources, cloud support, account configuration and API authentication for the installed provider version.

## 4. Reference repository layout

    infrastructure/
      modules/
        cloud-network/
        cloud-identity/
        databricks-workspace/
        databricks-compute/
        databricks-governance/
        databricks-jobs/
        observability/
      environments/
        staging/
          main.tf
          providers.tf
          versions.tf
          backend.tf
          variables.tf
          outputs.tf
        production/
        disaster-recovery/
      policies/
      tests/
      runbooks/
        drift-response.md

Separate environments by backend/state, identity, approvals and blast radius. Never rely on Terraform workspace names alone for isolation.

## 5. Provider and version pinning

Example Terraform version constraints:

~~~hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.0"
    }
  }
}

provider "databricks" {
  # Use supported environment-based authentication.
}
~~~

The version range is illustrative, not a recommendation to downgrade. Select a tested current provider release, pin it to your approved compatibility window and commit the generated dependency lockfile.

## 6. Authentication and secret handling

Prefer workload identity or approved short-lived service principal credentials. Avoid hardcoding tokens, client secrets, storage keys or cloud account IDs into committed Terraform configuration.

Provider authentication differs by AWS, Azure and GCP. Verify the supported unified authentication configuration and ensure the executing identity has only required privileges. Terraform state and plan files can contain sensitive values even when output is redacted.

## 7. Remote state and locking

Use an approved backend with:
- Encryption at rest and in transit.
- Access control and least privilege.
- Versioning/backup and restore testing.
- State locking or concurrency controls supported by the backend.
- Separate state per environment and trust boundary.
- Audit trails and retention.
- Documented break-glass procedures.

Never manually copy production state into a staging backend or force-unlock without confirming ownership and an abandoned lock.

## 8. Lab A — Windows PowerShell preflight

~~~powershell
terraform version
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
git status --short
~~~

Verify the Git branch, cloud account, region, workspace host and principal before continuing.

## 9. Lab B — Terraform formatting and static validation

From an approved Terraform directory:

~~~powershell
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
~~~

A successful validation confirms configuration structure, not cloud access, plan safety or runtime health.

## 10. Lab C — Plan-only change review

From a dedicated staging backend and working directory:

~~~powershell
terraform init
terraform workspace show
terraform plan -out=staging-review.tfplan
terraform show -no-color staging-review.tfplan
~~~

Inspect additions, changes, replacements and destroys. Plan files are sensitive; store them in controlled build artifacts, never public repositories.

## 11. Lab D — Detect drift without proposing repair

~~~powershell
terraform plan -refresh-only -out=refresh-review.tfplan
terraform show -no-color refresh-review.tfplan
~~~

A refresh-only plan reveals changes to Terraform state observations. It does not automatically repair infrastructure drift. Do not apply refresh-only changes without reviewing state implications.

## 12. Lab E — Compare desired state with reality

~~~powershell
terraform plan -detailed-exitcode -no-color
$code = $LASTEXITCODE
switch ($code) {
  0 { "No differences detected" }
  1 { "Plan error; investigate before continuing" }
  2 { "Differences detected; review before remediation" }
  default { "Unexpected Terraform exit code: $code" }
}
~~~

This example runs a plan that may access cloud APIs and remote state but does not apply changes. An exit code of 2 means differences exist; it does not prove they were caused by unauthorized drift.

## 13. Distinguish drift from intended change

| Observation | Possible cause | Next action |
|---|---|---|
| Plan shows update | PR changed desired config | Review commit and approval |
| Plan shows update, no Git change | Out-of-band drift | Inspect cloud/Databricks audit |
| Plan shows replacement | Immutable field or resource identity | Stop and assess downtime |
| Plan shows destroy | Resource removed from config | Verify ownership and intent |
| Plan errors | Auth, state or provider issue | Resolve evidence gap |
| No differences | Managed state matches plan | Still run runtime acceptance checks |

## 14. Lab F — State inspection, read-only

~~~powershell
terraform state list
terraform state show 'databricks_job.example'
~~~

Replace the example resource address with one returned by state list. State output may expose sensitive metadata. Do not publish it. Do not use state rm, mv, import or force-unlock without a documented state change procedure.

## 15. Lab G — Databricks resource inventory

~~~powershell
databricks jobs list --output json
databricks clusters list --output json
databricks cluster-policies list --output json
~~~

Inventory may be permission-filtered or paginated. Compare resource IDs and names with Terraform state and approved configuration.

## 16. Lab H — Safe example job definition

Example for an isolated staging workspace with an existing approved notebook:

~~~hcl
resource "databricks_job" "staging_validation" {
  name = "demo-staging-validation"

  task {
    task_key = "validate"
    existing_cluster_id = var.approved_staging_cluster_id

    notebook_task {
      notebook_path = var.validation_notebook_path
    }
  }
}
~~~

Define variables in the module, use an authorized staging cluster and notebook, and review provider documentation for your pinned version. This snippet is illustrative; do not apply it to production without policy, identity and job concurrency controls.

## 17. Lab I — Detect job configuration drift

Record:
- Job name and ID.
- Task graph and dependencies.
- Schedule/timezone.
- Retry and concurrency settings.
- Compute policy and node types.
- Run-as principal.
- Notifications.
- Git source and library versions.

Compare approved Terraform configuration to current Jobs API and audit events. Changes in unmanaged fields may not appear in a Terraform plan.

## 18. Lab J — Unity Catalog governance drift

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
~~~

Use an approved staging catalog. Detect unexpected ownership, grants, workspace bindings and external location access. Do not remediate by granting broad privileges.

## 19. AWS cloud drift inventory

~~~powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-vpcs --max-results 20
aws ec2 describe-vpc-endpoints --max-results 20
~~~

Review VPC routes, security groups, IAM roles, endpoints, S3 bucket policies and KMS dependencies. Compare cloud inventory with cloud-provider Terraform state.

## 20. Azure cloud drift inventory

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az network vnet list --resource-group demo-staging-rg -o table
az network private-endpoint list --resource-group demo-staging-rg -o table
~~~

Review VNet/subnet, NSG, private DNS, ADLS permissions, managed identity and Key Vault. Replace placeholder resource group.

## 21. GCP cloud drift inventory

~~~powershell
gcloud config get-value project
gcloud compute networks list
gcloud compute routes list
gcloud iam service-accounts list
~~~

Review VPC firewall, Cloud NAT, service accounts, Cloud Storage IAM and KMS. Confirm the authorized project before inventory.

## 22. CI/CD GitOps pipeline

A safe pull-request pipeline should:
1. Verify branch and approved change ticket.
2. Run Terraform formatting and validation.
3. Run security and policy checks.
4. Plan against the intended backend and identity.
5. Publish a redacted change summary.
6. Flag resource replacement/destruction.
7. Require appropriate peer/security approval.
8. Apply only after environment-specific authorization.
9. Run post-apply smoke tests.
10. Archive sanitized evidence and rollback notes.

Never allow an untrusted fork pull request to execute with privileged cloud or Databricks credentials.

## 23. Lab K — Example GitHub Actions validation workflow

~~~yaml
name: terraform-validate
on:
  pull_request:
    paths:
      - "infrastructure/**"
jobs:
  validate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - name: Format
        run: terraform -chdir=infrastructure/environments/staging fmt -check -recursive
      - name: Initialize without backend
        run: terraform -chdir=infrastructure/environments/staging init -backend=false
      - name: Validate
        run: terraform -chdir=infrastructure/environments/staging validate
~~~

Pin action revisions to approved immutable SHAs in production. This sample deliberately does not access remote state, plan or apply; those require separate secure workflows.

## 24. Policy-as-code checks

Require:
- Approved region and cloud account.
- No public storage or unexpected ingress.
- Private network connectivity where mandated.
- Approved instance types and cost limits.
- Mandatory tags/labels and ownership.
- Restricted admin privileges.
- Encrypted state and protected secrets.
- No unreviewed destroys/replacements.
- Backup/DR and monitoring requirements.

Use organization-approved tools and policies; Terraform validate alone does not enforce these rules.

## 25. Drift detection cadence

Select a schedule based on change rate and risk:
- Critical network/IAM controls: frequent automated assessment.
- Workspace policies and jobs: scheduled and post-change checks.
- Low-risk metadata: less frequent inventory review.
- After incidents: immediate targeted read-only assessment.

Throttle API calls and account for provider limits. Do not trigger automatic applies from every detected difference.

## 26. Drift response runbook

1. Capture plan output, resource IDs and UTC time.
2. Confirm cloud account, region, workspace and state.
3. Identify Git changes versus out-of-band activity.
4. Inspect cloud and Databricks audit logs.
5. Determine business impact and security risk.
6. Decide whether to update desired code or revert drift.
7. Review a new full plan and potential replacements.
8. Obtain approvals and execute in staging first.
9. Validate connectivity, jobs, permissions and SLO.
10. Record RCA, owner and preventive control.

## 27. Scenario A — Manual cluster policy change

**Detect:** Terraform plan and Databricks audit.

**Risk:** unapproved node type or cost increase.

**Response:** approved policy reconciliation.

**Pass:** policy enforcement restored and no workload regression.

## 28. Scenario B — Public network exposure

**Detect:** cloud network drift and security posture checks.

**Risk:** unauthorized access path.

**Response:** security incident workflow and approved network correction.

**Pass:** private access verified and unintended exposure closed.

## 29. Scenario C — Privilege escalation

**Detect:** IAM/Unity Catalog grant drift.

**Risk:** unauthorized data access.

**Response:** contain access and restore least privilege.

**Pass:** authorized principal succeeds, unauthorized principal denied.

## 30. Scenario D — Resource replacement in plan

**Detect:** destroy/create or replacement marker.

**Risk:** workspace/job disruption and state loss.

**Response:** halt apply; evaluate import, migration and rollback.

**Pass:** no unapproved destruction.

## 31. Scenario E — Remote state lock contention

**Detect:** Terraform backend lock error.

**Risk:** conflicting changes.

**Response:** identify active owner; wait or coordinate.

**Pass:** single authorized change execution; never force-unlock casually.

## 32. Scenario F — Provider upgrade regression

**Detect:** unexpected plan after version update.

**Risk:** changed defaults, schema or resource behavior.

**Response:** restore pinned version, test migration and review release notes.

**Pass:** expected plan and runtime validation.

## 33. Scenario G — DR workspace configuration drift

**Detect:** standby job/policy/grant parity mismatch.

**Risk:** DR failure during incident.

**Response:** staged reconciliation with DR activation disabled.

**Pass:** readiness checks and no dual writers.

## 34. Scenario H — Job schedule changed out of band

**Detect:** job API, audit and Terraform plan.

**Risk:** missed SLA or duplicate runs.

**Response:** restore approved schedule and verify timezone/concurrency.

**Pass:** expected trigger and validated output.

## 35. Rollback strategy

Terraform does not provide a universal automatic rollback. A previous configuration may not safely reverse destructive operations or external side effects.

Use:
- Version-controlled last-known-good configuration.
- Reviewed compensating Terraform plan.
- Backups and restore for stateful resources.
- Change freeze and writer fencing where relevant.
- Validation of data integrity, permissions and consumer SLO.

Avoid applying stale saved plans after infrastructure or state changes.

## 36. Post-change acceptance

| Check | Required evidence |
|---|---|
| Intended account/region/workspace | Identity and endpoint |
| Terraform state healthy | Backend and lock status |
| No unexpected destroy | Reviewed plan |
| Network/private connectivity | Compute-side probe |
| IAM and Unity Catalog | Authorized/denied access tests |
| Jobs and schedules | Test run and task graph |
| Compute policy | Enforcement check |
| Data integrity | Synthetic validation |
| Monitoring and alerts | Synthetic alert |
| Cost impact | Budget review |
| Rollback readiness | Approved plan and owner |

## 37. Incident and change report template

    Change/incident ID:
    UTC timestamp:
    Repository / commit:
    Cloud account / region / workspace:
    Terraform backend / state:
    Desired change:
    Observed drift:
    Audit evidence:
    Affected resources:
    Security and availability impact:
    Plan additions/updates/replacements/destroys:
    Approval and execution owner:
    Post-change validation:
    Rollback plan:
    RCA and corrective actions:

## 38. Acceptance checklist

- [ ] Providers and Terraform versions pinned.
- [ ] Environment state boundaries documented.
- [ ] Remote state encrypted, locked and backed up.
- [ ] Identity and secret handling approved.
- [ ] Plan-only staging validation completed.
- [ ] Drift detection tested with exit-code handling.
- [ ] Cloud and Databricks inventories compared.
- [ ] Security and cost policies reviewed.
- [ ] CI/CD approval gates documented.
- [ ] Eight drift scenarios reviewed.
- [ ] Rollback and state recovery procedures defined.
- [ ] Post-change consumer validation completed.

## 39. GO / CONDITIONAL GO / NO-GO

**GO:** Reviewed plan, secure state, approved changes, correct permissions and successful post-change tests.

**CONDITIONAL GO:** Noncritical gaps documented with owner and mitigation.

**NO-GO:** Unknown state ownership, unexpected destroys, privilege escalation, production credentials in untrusted CI, missing rollback or failed data correctness.

## 40. Review questions

1. Why can Terraform show differences even without unauthorized drift?
2. Why is a refresh-only plan not the same as remediation?
3. What risks arise from storing secrets in Terraform state?
4. Why should production plans and applies use controlled identities?
5. Why is a successful apply insufficient to close a change?

## 41. Official references

- Databricks Terraform provider: https://registry.terraform.io/providers/databricks/databricks/latest/docs
- Databricks Terraform documentation: https://docs.databricks.com/aws/en/dev-tools/terraform/
- Databricks unified authentication: https://docs.databricks.com/aws/en/dev-tools/auth/unified-auth
- Terraform plan: https://developer.hashicorp.com/terraform/cli/commands/plan
- Terraform state: https://developer.hashicorp.com/terraform/language/state
- Terraform dependency lock file: https://developer.hashicorp.com/terraform/language/files/dependency-lock
- GitHub Actions security: https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions
- AWS Well-Architected Operational Excellence: https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/welcome.html
- Azure Well-Architected Operational Excellence: https://learn.microsoft.com/azure/well-architected/operational-excellence/
- Google Cloud operational excellence: https://cloud.google.com/architecture/framework/operational-excellence

**Next:** Chapter 047 — Workspace Configuration as Code.
