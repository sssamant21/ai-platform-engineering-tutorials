# Chapter 048 — CI/CD for Infrastructure Changes

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 10 — Automation and GitOps  
**Clouds:** AWS, Azure, Google Cloud  
**Primary CI/CD:** GitLab CI/CD  
**Audience:** SRE, Platform Engineering, Security, Cloud Infrastructure and Change Management  
**Prerequisites:** Chapters 041–047  
**Safety:** All execution examples target approved isolated staging environments. Production apply, bundle deploy, resource destruction, identity changes and regional promotion require protected environments, peer approval, rollback readiness and change authorization.

## 1. Mission

Build a secure, auditable CI/CD workflow that provisions and changes Databricks infrastructure without granting untrusted pipelines unrestricted cloud or workspace access.

### Learning outcomes

- Separate infrastructure and workload deployment responsibilities.
- Implement GitLab CI/CD validation, plan, approval and deployment stages.
- Use Terraform for cloud/workspace resources and Databricks Declarative Automation Bundles for approved workload resources.
- Establish short-lived identity and remote state protections.
- Detect destructive plans and unauthorized drift.
- Promote changes through Dev, Test, Staging and Production with controlled approvals.
- Validate service health and data correctness after changes.
- Define rollback, recovery and incident escalation procedures.

## 2. Reference architecture

    Engineer / Merge Request
              |
       GitLab repository
              |
    Format + validate + scan
              |
      Policy and cost checks
              |
      Terraform plan review
              |
      Protected approval gate
              |
    Apply infrastructure changes
              |
      Databricks bundle deploy
              |
      Smoke + security tests
              |
      SLO and drift monitoring
              |
       Change evidence archive

Infrastructure and workload releases may use separate pipelines. Never let Terraform and a bundle manage the same resource without a documented migration of ownership.

## 3. Pipeline ownership boundaries

| Resource | Controller | Validation |
|---|---|---|
| Cloud network, storage and IAM | Cloud Terraform | Plan + network tests |
| Databricks workspace | Terraform | Plan + API access |
| Compute policies | Terraform | Policy enforcement |
| Unity Catalog grants/locations | Governed Terraform/SQL | Least privilege |
| Jobs and workload artifacts | Bundle or Terraform | One owner per job |
| Secrets | Approved secret manager | References only |
| Monitoring | IaC | Alert test |
| Delta data | Data recovery procedure | Transaction consistency |

A green CI pipeline does not establish data correctness or production recovery.

## 4. Repository layout

    databricks-platform/
      infrastructure/
        modules/
        environments/
          dev/
          test/
          staging/
          production/
      bundles/
        demo-workflows/
          databricks.yml
          resources/
          src/
      policies/
      scripts/
      tests/
      runbooks/
      .gitlab-ci.yml

Maintain separate state backends and deployment identities for each environment. Avoid copying state between environments.

## 5. Branching and promotion

Recommended:
- Merge requests require peer review and passing validation.
- Protected main branch represents approved source.
- Dev/Test use automated validation and controlled deployment.
- Staging validates production-like changes.
- Production requires protected environment approval, change window and a fresh reviewed plan.
- Emergency fixes require incident-linked authorization and post-change reconciliation.

Do not reuse a staging plan file against production state.

## 6. Trust and runner isolation

Use hardened GitLab runners with:
- Ephemeral execution where feasible.
- Restricted network access to required APIs.
- No privileged execution for untrusted merge requests.
- Protected variables and environments.
- Short-lived cloud and Databricks credentials.
- Artifact retention controls.
- Audited runner updates and dependency pinning.

A runner executing untrusted fork code must not receive production tokens, state access or cloud permissions.

## 7. Terraform state security

Each environment requires:
- Dedicated encrypted remote state.
- Concurrency locking supported by backend.
- Least-privilege backend identity.
- State versioning and backup/recovery procedure.
- Audit and access logging.
- Sensitive artifact retention policy.

Plan files and state may include secrets. Never commit them, print them into unrestricted job logs or expose them in public merge request artifacts.

## 8. Identity and authentication

Prefer supported OIDC workload identity federation to AWS, Azure or GCP and approved Databricks service-principal authentication. Validate exact federation support for the cloud and Databricks authentication method used.

If federation is not supported for a dependency, use a tightly scoped approved secret with rotation, protected CI variables and restricted environments. Never embed personal access tokens in YAML.

## 9. Lab A — Windows PowerShell preflight

~~~powershell
git --version
terraform version
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
git status --short
~~~

Confirm Git remote, branch, actual workspace host, cloud account, region and principal.

## 10. Lab B — Terraform formatting and validation

From an approved Terraform environment directory:

~~~powershell
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
~~~

This is a static validation. It does not prove cloud credentials, backend access or a safe apply.

## 11. Lab C — Plan-only staging review

~~~powershell
terraform init
terraform workspace show
terraform plan -out=staging-review.tfplan
terraform show -no-color staging-review.tfplan
~~~

Run only in an approved staging working directory with an authorized backend. Verify account, region, state, workspace and resource changes. Do not commit the plan.

## 12. Lab D — Exit-code based drift detection

~~~powershell
terraform plan -detailed-exitcode -no-color
$code = $LASTEXITCODE
switch ($code) {
  0 { "No differences" }
  1 { "Plan error" }
  2 { "Differences detected; approval required" }
  default { "Unexpected exit code: $code" }
}
~~~

Differences may reflect approved code changes, provider updates or unauthorized drift. Correlate with Git and audit history.

## 13. GitLab CI/CD baseline

The following example is a safe **static validation** pipeline and does not contact production backends:

~~~yaml
stages:
  - validate

terraform_validate:
  stage: validate
  image:
    name: hashicorp/terraform:1.9.8
    entrypoint: [""]
  script:
    - terraform -chdir=infrastructure/environments/staging fmt -check -recursive
    - terraform -chdir=infrastructure/environments/staging init -backend=false
    - terraform -chdir=infrastructure/environments/staging validate
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
~~~

Pin the image by approved immutable digest in production and verify the selected Terraform version against provider constraints. The example deliberately contains no privileged credentials or apply stage.

## 14. Lab E — GitLab pipeline lint and dry run

Before merging:
1. Inspect the .gitlab-ci.yml in the GitLab CI Lint UI.
2. Verify job rules for merge requests and protected branches.
3. Confirm no production secrets are exposed to untrusted branches.
4. Confirm artifact retention and masked/protected variables.
5. Run the static validation job in an isolated runner.
6. Review job logs for accidental sensitive output.

Use a staging test repository/project where feasible.

## 15. Planning pipeline design

A separate protected planning job should:
- Authenticate using a restricted environment identity.
- Initialize the correct remote backend.
- Acquire state lock.
- Run terraform plan with a controlled artifact destination.
- Parse and summarize resource additions/changes/replacements/destroys.
- Scan for policy and cost violations.
- Preserve plan securely for an approved short retention window.
- Publish a redacted merge request summary.

Do not upload raw plan JSON to public logs or comments.

## 16. Approval and apply architecture

An approved apply must:
- Target the exact environment and state reviewed.
- Confirm plan freshness and immutable commit SHA.
- Use a protected GitLab environment and trusted runner.
- Require authorized human approval for production.
- Reject unexpected destroy/replacement.
- Record executor identity and UTC timeline.
- Validate runtime behavior after apply.
- Retain rollback/compensation instructions.

Saved plans become stale if state or external resources change. Replan and reapprove as necessary.

## 17. Lab F — Review plan resource actions

~~~powershell
terraform show -json staging-review.tfplan |
  Out-File -FilePath staging-plan.json -Encoding utf8
~~~

Treat staging-plan.json as sensitive and store only in a protected temporary workspace. Review resource_changes and action types using approved tooling. Delete temporary artifacts through your organization's secure cleanup procedure.

## 18. Databricks bundles and workload release

Use Databricks Declarative Automation Bundles for workload resources with a clear ownership boundary.

~~~powershell
databricks bundle validate --target staging
databricks bundle summary --target staging
databricks bundle deploy --help
~~~

Validation and summary are non-deploying checks. Deploy only from an approved protected pipeline. Validate CLI and bundle schema for the installed version.

## 19. Example bundle targets

~~~yaml
bundle:
  name: demo-workflows

targets:
  dev:
    mode: development
    workspace:
      host: https://REPLACE_WITH_DEV_HOST
  staging:
    mode: production
    workspace:
      host: https://REPLACE_WITH_STAGING_HOST
  prod:
    mode: production
    workspace:
      host: https://REPLACE_WITH_PRODUCTION_HOST
~~~

Hosts are placeholders. Define resources separately and verify target modes, permissions, unique resource names and deployment identities. Production mode is a bundle setting, not an authorization gate.

## 20. Deployment order and dependencies

1. Cloud network and IAM.
2. Workspace and connectivity.
3. Unity Catalog governance and storage access.
4. Compute policies and permitted compute.
5. Bundle-managed jobs and artifacts.
6. Synthetic validation job.
7. Monitoring and alert delivery.
8. Controlled schedule activation.
9. Consumer correctness and SLO verification.

Do not activate production schedules before data consistency and single-writer ownership are verified.

## 21. Lab G — Jobs and policies inventory

~~~powershell
databricks jobs list --output json
databricks cluster-policies list --output json
databricks clusters list --output json
~~~

Inventory can be permission-filtered or paginated. Compare actual resource IDs/settings against version-controlled definitions.

## 22. Lab H — Unity Catalog access validation

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
~~~

Test expected and denied access with approved identities. A successful admin query does not prove that the production run-as principal has correct permissions.

## 23. Lab I — Synthetic data correctness

For an authorized synthetic staging table:

~~~sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(DISTINCT event_id) AS unique_events,
  MAX(event_time) AS latest_event
FROM demo_catalog.demo_staging.events;
~~~

Replace the schema and column names as appropriate. Compare against a known source manifest, not only the previous run's row count.

## 24. AWS integration

Check:
- GitLab-to-AWS workload identity and IAM trust.
- Cross-account Databricks permissions.
- VPC, private endpoints, S3 and KMS.
- EC2 quotas and node type policy.
- CloudTrail and CloudWatch.
- State backend access boundaries.

Read-only verification:

~~~powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-vpc-endpoints --max-results 20
~~~

## 25. Azure integration

Check:
- GitLab-to-Azure workload identity.
- Subscription and resource group scope.
- VNet, private DNS and Private Link.
- ADLS Gen2, managed identities and Key Vault.
- Azure Monitor and Activity Logs.
- Workspace and compute quota.

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az network private-endpoint list --resource-group demo-staging-rg -o table
~~~

## 26. GCP integration

Check:
- GitLab-to-GCP Workload Identity Federation.
- Project, service accounts and IAM.
- VPC, firewall, routing and private access.
- Cloud Storage and Cloud KMS.
- Cloud Audit Logs and Cloud Monitoring.
- Regional quota and Databricks support.

~~~powershell
gcloud config get-value project
gcloud compute networks list
gcloud iam service-accounts list
~~~

## 27. Change windows and operational coordination

Production change planning includes:
- Affected workspaces and data products.
- Expected outage or performance risk.
- On-call staffing and incident commander.
- Data freshness/SLO thresholds.
- Workflow pause/resume policy.
- Backup and recovery-point validation.
- Abort triggers and rollback owner.
- Stakeholder communications.

## 28. Drift detection after deployment

Use Terraform plans, Databricks API inventory, cloud audit logs and Git commit history to detect out-of-band changes.

Automated drift checks should report and triage; do not automatically apply production changes merely because a plan differs.

## 29. Lab J — Post-deployment read-only checks

~~~powershell
databricks auth describe
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
~~~

Combine with network connectivity tests, security checks, data invariants and monitoring evidence.

## 30. Scenario A — Pipeline targets wrong workspace

**Detect:** workspace host/account mismatch.

**Response:** stop before plan/apply, correct protected environment binding.

**Pass:** independent target verification and no unauthorized changes.

## 31. Scenario B — Production secret exposed to merge request

**Detect:** pipeline permissions and log review.

**Response:** revoke/rotate compromised credential, isolate runner and investigate access.

**Pass:** no privileged credentials in untrusted contexts.

## 32. Scenario C — Terraform plan proposes destruction

**Detect:** replace/destroy actions.

**Response:** halt, inspect state ownership and migration requirements.

**Pass:** no unapproved destruction; reviewed alternative.

## 33. Scenario D — Remote state lock contention

**Detect:** backend lock error.

**Response:** identify active operation and coordinate.

**Pass:** single authorized writer; no unsafe force-unlock.

## 34. Scenario E — Bundle and Terraform manage same job

**Detect:** repeated configuration oscillation or unexpected plan.

**Response:** assign one controller and migrate ownership safely.

**Pass:** stable configuration after repeated validation.

## 35. Scenario F — Unity Catalog permission regression

**Detect:** job run 403, grant inventory.

**Response:** restore approved least-privilege access.

**Pass:** expected access succeeds and unauthorized access fails.

## 36. Scenario G — Successful deployment, failed consumer

**Detect:** freshness, schema, row-count or downstream SLO failure.

**Response:** declare deployment failure, stop unsafe writes and execute recovery.

**Pass:** validated consumer correctness and service health.

## 37. Scenario H — Rollback commit fails to restore service

**Detect:** stateful side effects, incompatible schema or missing data.

**Response:** use compensating infrastructure changes and supported data recovery.

**Pass:** service correctness, not just Git revision parity.

## 38. Scenario I — Regional capacity unavailable during deployment

**Detect:** cloud quota/capacity and cluster provisioning failures.

**Response:** approved alternative capacity or defer release.

**Pass:** SLO and cost remain within limits.

## 39. Scenario J — CI/CD runner compromise

**Detect:** suspicious pipeline execution, identity/audit events.

**Response:** disable runner, revoke credentials, preserve evidence and rotate trust.

**Pass:** hardened runner, least-privilege identity and clean redeployment.

## 40. Production deployment runbook

1. Confirm ticket, scope, owner and release commit.
2. Verify protected GitLab environment and runner trust.
3. Confirm cloud account, workspace, region and backend.
4. Review formatting, validation, security and policy results.
5. Review Terraform plan and bundle configuration.
6. Check data protection, workflow concurrency and DR impact.
7. Obtain change and security approval.
8. Execute approved deployment with audit trail.
9. Verify IAM, networking, compute, jobs and data correctness.
10. Observe consumer SLO, alerting and costs.
11. Record GO/rollback decision and evidence.
12. Reconcile drift and close change.

## 41. Rollback and recovery

Terraform has no universal automatic rollback. A Git revert may not undo stateful data mutations, job runs, credential rotation or destructive changes.

Use:
- Last-known-good configuration and compensating plan.
- Protected Terraform state history.
- Approved Delta/data backup recovery.
- Writer fencing and idempotent replay.
- Controlled bundle redeployment.
- Post-recovery consumer and security validation.

## 42. CI/CD evidence package

    release-YYYYMMDD/
      01-change-request.md
      02-commit-and-review.md
      03-validation-summary.md
      04-security-and-policy.md
      05-plan-summary.md
      06-approvals.md
      07-deployment-timeline.md
      08-runtime-checks.md
      09-data-validation.md
      10-rollback-readiness.md
      11-final-decision.md

Keep raw plans, state, tokens and sensitive logs in restricted storage, not public Git.

## 43. Acceptance scorecard

| Check | Required evidence |
|---|---|
| Branch/review protection | GitLab rules |
| Trusted runner | Runner policy |
| Least-privilege identity | IAM review |
| Secure state backend | Backend controls |
| Terraform validation | CI result |
| Bundle validation | CLI result |
| No unapproved destroy | Plan approval |
| Environment targeting | Host/account/state |
| Unity Catalog access | Authorized/denied test |
| Job behavior | Synthetic run |
| Data correctness | Invariants and freshness |
| Monitoring | Alert evidence |
| Rollback readiness | Recovery runbook |
| Audit trail | Change record |

## 44. GO / CONDITIONAL GO / NO-GO

**GO:** Approved and correctly targeted deployment, secure identities/state, no unreviewed destruction, successful service and data validation.

**CONDITIONAL GO:** Only noncritical issues remain with approved mitigation, owner and deadline.

**NO-GO:** Wrong environment, credential exposure, destructive plan without approval, uncontrolled writers, missing rollback or failed consumer correctness.

## 45. Review questions

1. Why should validation and apply use different trust boundaries?
2. Why must a production plan be tied to a specific commit and state?
3. What makes a green infrastructure pipeline insufficient for SRE acceptance?
4. Why can a Git revert fail to restore a stateful Databricks workload?
5. How do you prevent untrusted merge requests from accessing production credentials?

## 46. Official references

- GitLab CI/CD: https://docs.gitlab.com/ci/
- GitLab protected environments: https://docs.gitlab.com/ci/environments/protected_environments/
- GitLab OIDC: https://docs.gitlab.com/ci/secrets/id_token_authentication/
- Databricks Terraform: https://docs.databricks.com/aws/en/dev-tools/terraform/
- Databricks Declarative Automation Bundles: https://docs.databricks.com/aws/en/dev-tools/bundles/
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks unified authentication: https://docs.databricks.com/aws/en/dev-tools/auth/unified-auth
- Terraform plan: https://developer.hashicorp.com/terraform/cli/commands/plan
- Terraform state: https://developer.hashicorp.com/terraform/language/state
- AWS Well-Architected Operational Excellence: https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/welcome.html
- Azure Well-Architected Operational Excellence: https://learn.microsoft.com/azure/well-architected/operational-excellence/
- Google Cloud operational excellence: https://cloud.google.com/architecture/framework/operational-excellence

**Next:** Chapter 049 — Policy Validation and Operational Testing.
