# Chapter 047 — Workspace Configuration as Code

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 10 — Automation and GitOps  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, Platform Engineering, Security, Cloud Infrastructure and Release Engineering  
**Prerequisites:** Chapters 016–020, 021–025, 041–046  
**Safety:** Labs use approved isolated staging workspaces. Production operations remain read-only or plan-only unless a reviewed change, rollback, security approval and maintenance window are in place.

## 1. Mission

Make Databricks workspace configuration reproducible, reviewable and auditable. Separate cloud infrastructure, Databricks account/workspace settings, workload deployment, permissions and secrets into clear sources of truth with explicit ownership.

### Learning outcomes

- Classify workspace resources by management mechanism.
- Design version-controlled workspace configuration repositories.
- Apply Terraform and Databricks Declarative Automation Bundles (formerly Asset Bundles) appropriately.
- Control jobs, compute policies, workspace files, permissions and governance.
- Separate Dev, Test, Staging and Production configurations safely.
- Validate configuration without modifying production.
- Detect drift and manage exceptions.
- Test recovery from source-controlled configuration.

## 2. Configuration architecture

    Git pull request
          |
    Static validation
          |
    Policy/security review
          |
     Approved pipeline
       /         \
  Terraform    Bundles
     |            |
  Account,      Jobs, pipelines,
  workspace,    workload files
  governance    and deployment
       \         /
       Databricks APIs
              |
    Drift + runtime checks
              |
      Evidence and RCA

Avoid having Terraform and a bundle independently manage the same resource. Define ownership and resource lifecycle clearly.

## 3. What belongs in code?

| Configuration | Preferred source of truth | Notes |
|---|---|---|
| Cloud VPC/VNet, IAM, storage | Cloud Terraform | Separate cloud state |
| Workspace provisioning | Terraform | Cloud-specific requirements |
| Account identity/groups | IdP + supported automation | Avoid duplicate ownership |
| Compute policies | Terraform | Review constraints and privileges |
| Unity Catalog definitions/grants | Terraform or governed SQL | Track ownership and migration |
| Jobs/workflows | Bundle or Terraform | Choose one owner per job |
| Workspace notebooks/source | Git + bundle | Avoid manual-only edits |
| SQL warehouses | Supported IaC/API | Check provider support |
| Monitoring/alerts | IaC and observability config | Test delivery |
| Secrets | Secret manager and references | Never commit values |
| Runtime job state/history | Platform telemetry | Not declarative configuration |

## 4. Ownership boundaries

Define a configuration ownership register containing:
- Resource identifier and environment.
- Terraform module, bundle, or external owner.
- Cloud and region.
- Review/approval group.
- Backup/rebuild procedure.
- Drift handling policy.
- Recovery criticality.

A resource with two controllers may oscillate or overwrite changes.

## 5. Reference repository structure

    databricks-platform/
      infrastructure/
        modules/
          workspace/
          compute-policies/
          unity-catalog/
          observability/
        environments/
          dev/
          test/
          staging/
          production/
      bundles/
        demo-workflows/
          databricks.yml
          resources/
            jobs.yml
          src/
            validate.py
      policies/
      tests/
      runbooks/
      .github/workflows/

Keep environment-specific backend state, credentials, permissions and deployment targets isolated.

## 6. Environment separation

| Environment | Intended purpose | Deployment control |
|---|---|---|
| Dev | Iterative configuration testing | Restricted developers |
| Test | Integration and regression | Automated tests |
| Staging | Production-like validation | Reviewed promotion |
| Production | Business-critical workloads | Approval + change control |

Namespaces are a Kubernetes concept; Databricks workspace/environment isolation uses supported workspace, catalog/schema, identity and policy boundaries. Do not assume one workspace is sufficient for all risk levels.

## 7. Terraform provider configuration

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
  # Use supported unified authentication from environment/identity.
}
~~~

The provider range is illustrative. Pin a tested approved current release and commit the dependency lockfile.

## 8. Terraform example — staging cluster policy

~~~hcl
resource "databricks_cluster_policy" "demo_staging" {
  name = "demo-staging-compute-policy"
  definition = jsonencode({
    "autotermination_minutes" = {
      type  = "fixed"
      value = 30
    }
    "spark_version" = {
      type       = "unlimited"
      isOptional = false
    }
  })
}
~~~

This is a teaching example, not a complete enterprise policy. Real policies must constrain approved runtimes, instance families, security mode, worker limits and cost. Validate against your provider version and platform policies.

## 9. Lab A — Windows PowerShell preflight

~~~powershell
git --version
terraform version
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Confirm actual workspace host, cloud, region and principal. Do not trust profile names alone.

## 10. Lab B — Inventory existing configuration

~~~powershell
databricks jobs list --output json
databricks clusters list --output json
databricks cluster-policies list --output json
databricks workspace list /Shared --output json
~~~

Results may be paginated or permission-filtered. Record missing permissions rather than granting broader access for convenience.

## 11. Lab C — Terraform static checks

~~~powershell
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
~~~

Run in the approved IaC directory. Validation does not confirm that the remote state, workspace or runtime is healthy.

## 12. Lab D — Safe Terraform plan

In a dedicated staging state backend:

~~~powershell
terraform init
terraform workspace show
terraform plan -out=workspace-staging.tfplan
terraform show -no-color workspace-staging.tfplan
~~~

Review any destroy/replacement and verify the target account, region, workspace and backend. Plan artifacts may contain sensitive data and must not be committed.

## 13. Declarative Automation Bundles

Databricks bundles package and deploy supported workload resources such as jobs and pipelines together with source code, configuration and target overrides.

Use bundles for workload deployment, while Terraform commonly manages broader platform resources. Supported resource types and bundle syntax evolve; validate against the installed CLI.

## 14. Example bundle configuration

~~~yaml
bundle:
  name: demo-workflows

resources:
  jobs:
    validation_job:
      name: demo-staging-validation
      tasks:
        - task_key: validate
          spark_python_task:
            python_file: ./src/validate.py
          new_cluster:
            spark_version: "REPLACE_WITH_APPROVED_RUNTIME"
            node_type_id: "REPLACE_WITH_CLOUD_NODE_TYPE"
            num_workers: 1

targets:
  staging:
    default: true
    workspace:
      host: https://REPLACE_WITH_STAGING_WORKSPACE_HOST
~~~

The runtime, node type and workspace host are placeholders. Check workspace policies, compute mode support and current bundle schema before deployment. Never commit credentials.

## 15. Lab E — Bundle validation

~~~powershell
databricks bundle --help
databricks bundle validate --target staging
databricks bundle summary --target staging
~~~

Run from the bundle root. Validation checks configuration but does not prove that deployment, runtime access or business correctness will succeed.

## 16. Lab F — Bundle deployment preview and permissions

~~~powershell
databricks bundle deploy --help
databricks bundle run --help
databricks bundle destroy --help
~~~

Review supported deployment modes and target overrides. Actual deploy, run and destroy are excluded from the read-only lab. Never destroy a shared or production bundle target without explicit approval.

## 17. Workspace source and notebooks

Prefer Git as the source of truth for code. Use bundle-managed files or supported repository integrations for delivery. Manual notebook edits in a workspace should not silently become the canonical version.

For recovery, validate that code revisions, dependencies, runtime versions and permissions can be reconstructed.

## 18. Jobs as code

Track:
- Job ID/name and task graph.
- Task type and source revision.
- Compute and policy references.
- Schedules and timezone.
- Concurrency, retries and timeout.
- Run-as principal.
- Notifications and observability.
- Idempotency and replay safety.

Separate deployment of job configuration from activation of schedules in DR scenarios.

## 19. Lab G — Jobs API comparison

~~~powershell
databricks jobs list --output json
databricks jobs get --help
databricks jobs list-runs --limit 25 --output json
~~~

Compare the approved job definition to current API state. A job list alone does not contain all settings; fetch the selected job by ID with the installed CLI.

## 20. Unity Catalog as code

Manage catalog/schema definitions, storage credentials, external locations, volumes, grants and workspace bindings using supported automation and explicit ownership.

A Terraform resource definition is not a backup of underlying table data. Protect Delta transaction logs and object data separately.

## 21. Lab H — Governance read-only validation

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
~~~

Verify grants and ownership against approved policy. Do not use blanket privileges to make a lab pass.

## 22. Identity and secret boundaries

Use managed identity, service principals or supported workload identity. Secrets should be stored in approved secret managers and referenced securely.

Never commit:
- Personal access tokens.
- Client secrets or cloud access keys.
- Terraform state or plan artifacts.
- Passwords, private keys or signed URLs.
- Raw audit records containing sensitive identifiers.

Ensure credentials can be recovered and rotated without relying on an unavailable primary region.

## 23. AWS-specific configuration

Coordinate Databricks configuration with:
- VPC and subnet resources.
- IAM role trust and cross-account access.
- S3 buckets, encryption and external locations.
- Private connectivity and DNS.
- EC2 instance policies and quotas.
- CloudWatch/CloudTrail.

Read-only checks:

~~~powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-vpc-endpoints --max-results 20
~~~

## 24. Azure-specific configuration

Coordinate:
- Azure Databricks workspace resource and VNet.
- Private Link and private DNS.
- Managed identities/service principals.
- ADLS Gen2 and Key Vault.
- Azure Monitor and role assignments.
- VM quotas and runtime availability.

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az network private-endpoint list --resource-group demo-staging-rg -o table
~~~

## 25. GCP-specific configuration

Coordinate:
- Databricks workspace and supported GCP network architecture.
- VPC, routes, firewall and private connectivity.
- Service accounts and IAM.
- Cloud Storage and KMS.
- Cloud Monitoring and regional quotas.

~~~powershell
gcloud config get-value project
gcloud compute networks list
gcloud iam service-accounts list
~~~

## 26. Configuration drift model

| Drift type | Example | Detection |
|---|---|---|
| Terraform-managed | Policy manually edited | Terraform plan |
| Bundle-managed | Job task changed in UI | Bundle deployment diff/API comparison |
| Security | Catalog grant added | Audit + grant inventory |
| Cloud | Private endpoint removed | Cloud IaC plan |
| Secret lifecycle | Credential expired | Authentication probe |
| Runtime | Approved version retired | Job/cluster startup test |
| Schedule | Production trigger changed | Jobs API + audit |

Do not auto-reconcile every difference. Investigate whether the source of truth or the live environment is correct.

## 27. Lab I — Detect Terraform drift

~~~powershell
terraform plan -detailed-exitcode -no-color
$code = $LASTEXITCODE
switch ($code) {
  0 { "No differences detected" }
  1 { "Terraform plan failed" }
  2 { "Differences detected; review required" }
  default { "Unexpected exit code $code" }
}
~~~

Differences can arise from a legitimate Git change, provider behavior or out-of-band edits. Correlate with change history.

## 28. Lab J — Refresh-only state assessment

~~~powershell
terraform plan -refresh-only -out=workspace-refresh.tfplan
terraform show -no-color workspace-refresh.tfplan
~~~

This observes external changes and proposes state refresh, not resource remediation. Do not apply without approval.

## 29. CI/CD validation pipeline

Recommended stages:
1. Pull-request scope and ownership check.
2. Formatting and schema validation.
3. Secret scanning and dependency review.
4. Terraform plan or bundle validate.
5. Policy and cost review.
6. Security and change approval.
7. Approved deployment.
8. Runtime, permissions and data checks.
9. Evidence archival and drift baseline update.

Avoid exposing privileged credentials to untrusted pull requests.

## 30. Lab K — GitHub Actions static validation

~~~yaml
name: databricks-config-validation
on:
  pull_request:
    paths:
      - "infrastructure/**"
      - "bundles/**"
jobs:
  terraform-validation:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform -chdir=infrastructure/environments/staging fmt -check -recursive
      - run: terraform -chdir=infrastructure/environments/staging init -backend=false
      - run: terraform -chdir=infrastructure/environments/staging validate
~~~

Pin actions to approved immutable revisions in production. Bundle validation can be added in a separate job with a pinned Databricks CLI and restricted credentials if the bundle requires API access.

## 31. Scenario A — Manual cluster policy drift

**Evidence:** plan and audit event.

**Risk:** unsupported runtime, excessive cost or weakened isolation.

**Recovery:** reviewed configuration reconciliation.

**Acceptance:** policy restrictions and approved compute startup.

## 32. Scenario B — Job changed outside Git

**Evidence:** job API diff, audit and source revision.

**Risk:** unreviewed behavior and unreliable recovery.

**Recovery:** decide whether to revert live drift or merge a reviewed change.

**Acceptance:** job configuration matches source of truth and tests pass.

## 33. Scenario C — Missing Unity Catalog grant

**Evidence:** 403 and grant inventory.

**Risk:** workflow outage.

**Recovery:** restore approved least-privilege grant.

**Acceptance:** intended access works and prohibited access fails.

## 34. Scenario D — Secret rotated but workload fails

**Evidence:** auth failures and rotation timeline.

**Risk:** job failure and possible data freshness breach.

**Recovery:** restore correct reference and authorized principal.

**Acceptance:** no secret exposure and successful synthetic job.

## 35. Scenario E — Conflicting Terraform and bundle ownership

**Evidence:** resource repeatedly changes after deployments.

**Risk:** controller conflict and unexpected deletion.

**Recovery:** assign one authoritative manager and migrate state carefully.

**Acceptance:** repeat plans/deployments are stable.

## 36. Scenario F — Environment mis-targeting

**Evidence:** wrong workspace host, account or backend.

**Risk:** unintended production changes.

**Recovery:** stop execution, verify credentials and enforce environment approval.

**Acceptance:** independent target validation and no unauthorized change.

## 37. Scenario G — Recovery workspace lacks configuration

**Evidence:** missing job/policy/grant during DR drill.

**Risk:** RTO breach.

**Recovery:** rebuild from versioned configuration with schedules disabled.

**Acceptance:** tested job and security parity without dual writers.

## 38. Scenario H — Provider/CLI upgrade changes behavior

**Evidence:** validation or plan differences after upgrade.

**Risk:** resource replacement or deployment failure.

**Recovery:** pin tested versions, review release notes and stage migration.

**Acceptance:** stable plan, deploy and runtime tests.

## 39. Production runbook — Controlled configuration change

1. Confirm Git revision, change ticket and resource owner.
2. Verify workspace host, cloud account and state backend.
3. Run formatting, validation, policy and secret checks.
4. Review Terraform plan and bundle diff.
5. Assess downtime, resource replacement and job concurrency.
6. Confirm backup, rollback and DR impact.
7. Obtain environment-specific approval.
8. Deploy via the designated controller.
9. Verify job execution, access controls, data correctness and alerts.
10. Archive sanitized evidence and monitor for drift.

## 40. Rollback and disaster recovery

Rollback may require a compensating deployment, stateful data restoration or resource import. Reverting a Git commit does not automatically undo data mutations, secrets rotation or destructive resource changes.

For DR, rebuild infrastructure and configuration from code, recover storage and metadata from protected copies, fence writers, then validate consumers before enabling production schedules.

## 41. Acceptance scorecard

| Requirement | Evidence |
|---|---|
| Resource ownership documented | Ownership register |
| Provider/CLI versions pinned | Lockfile/build manifest |
| Environment isolation verified | Host/account/state |
| Terraform validation passes | CI log |
| Bundle validation passes | CLI output |
| Security/secret scan passes | CI report |
| No unreviewed destruction | Plan approval |
| IAM/UC grants correct | Access tests |
| Jobs and policies match Git | API comparison |
| Runtime and consumer checks pass | Synthetic test |
| Drift alerting operational | Test alert |
| Recovery procedure tested | DR evidence |

## 42. GO / CONDITIONAL GO / NO-GO

**GO:** Approved source-of-truth ownership, isolated targets, reviewed changes, least-privilege access and successful runtime acceptance.

**CONDITIONAL GO:** Only documented noncritical gaps remain with owners and deadlines.

**NO-GO:** Conflicting controllers, wrong target, unreviewed destroys, secrets in code/state artifacts exposed, broken access controls or untested rollback.

## 43. Review questions

1. Which resources should Terraform manage versus a Databricks bundle?
2. Why must one controller own each resource?
3. Why is bundle validation insufficient to prove workload recovery?
4. How can an environment be mis-targeted despite a staging profile name?
5. Why does reverting Git not guarantee rollback of stateful operations?

## 44. Official references

- Databricks Declarative Automation Bundles: https://docs.databricks.com/aws/en/dev-tools/bundles/
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks Terraform: https://docs.databricks.com/aws/en/dev-tools/terraform/
- Databricks Terraform provider: https://registry.terraform.io/providers/databricks/databricks/latest/docs
- Databricks unified authentication: https://docs.databricks.com/aws/en/dev-tools/auth/unified-auth
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Terraform state: https://developer.hashicorp.com/terraform/language/state
- Terraform plan: https://developer.hashicorp.com/terraform/cli/commands/plan
- GitHub Actions security: https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions

**Next:** Chapter 048 — CI/CD for Infrastructure Changes.
