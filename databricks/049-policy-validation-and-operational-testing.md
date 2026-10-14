# Chapter 049 — Policy Validation and Operational Testing

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 10 — Automation and GitOps  
**Clouds:** AWS, Azure, Google Cloud  
**Primary automation:** GitLab CI/CD, Terraform and Databricks CLI  
**Audience:** SRE, Platform Engineering, Cloud Security, Infrastructure Operations and Change Management  
**Prerequisites:** Chapters 041–048  
**Safety:** Execute mutation and negative tests only in an explicitly authorized, isolated staging workspace. Production examples are read-only. Never deliberately deny access, delete resources, induce quota failures, modify production policies or run disruptive fault injection without a separately approved change.

## 1. Mission

Establish repeatable, auditable validation gates that prove infrastructure changes comply with organizational policy **and** remain operable after deployment.

A passing Terraform syntax check, successful Databricks bundle validation or green GitLab pipeline does not prove least privilege, workload correctness, quota headroom, monitoring coverage or rollback readiness.

### Learning outcomes

- Distinguish static policy checks, authorization tests, integration tests and operational acceptance.
- Design policy-as-code gates for Terraform plans and approved Databricks configuration.
- Verify cluster policies, Unity Catalog privileges, workspace identity and cloud boundaries.
- Use positive and negative tests without disrupting production.
- Test infrastructure dependency health, platform APIs and synthetic workload behavior.
- Validate evidence, test isolation, approval boundaries and policy exceptions.
- Triage failures and decide GO, CONDITIONAL GO or NO-GO.

## 2. Policy and operational validation architecture

    Merge Request
        |
    Format / Lint / Static Checks
        |
    Terraform Validate + Bundle Validate
        |
    Terraform Plan (protected identity)
        |
    Policy-as-Code + Risk / Cost Review
        |
    Protected Staging Approval
        |
    Staging Apply / Bundle Deploy
        |
    Identity + Network + UC Tests
        |
    Synthetic Job + Data Invariants
        |
    Metrics / Alerts / SLO / Recovery Tests
        |
    Evidence Package + Production Approval

**Separate trust domains:** untrusted merge-request jobs receive no production tokens or protected state access. Privileged plan, apply and runtime tests execute on protected, isolated runners with short-lived identities.

## 3. Four layers of testing

| Layer | Purpose | Typical tools | Pass criterion |
|---|---|---|---|
| 1 — Static | Reject invalid files and insecure patterns before access to cloud systems | Terraform fmt/validate, linters, secret scan, bundle validate | No syntax, secrets or prohibited configuration |
| 2 — Planned-state | Evaluate proposed resource changes before apply | Terraform plan JSON, OPA/Conftest, approved policy engine | No unapproved destructive or out-of-policy change |
| 3 — Runtime security | Prove identities and privileges work as designed | Databricks CLI/SQL, cloud IAM and audit APIs | Required access allowed; prohibited access denied |
| 4 — Operational | Demonstrate service, data and recovery readiness | Synthetic jobs, metrics, logs, alert tests and runbooks | Published acceptance thresholds met |

Policy evaluation must be bound to a specific commit, target account/workspace, state revision and approved rule-set revision.

## 4. Policy ownership and enforcement boundaries

| Control | Authoritative layer | Verification |
|---|---|---|
| Cloud identity, network and encryption | Cloud IAM, networking, KMS | Terraform plan + cloud inventory |
| Workspace and platform settings | Databricks account/workspace administration | API inventory + audit events |
| Compute configuration | Databricks compute policies and cloud quotas | Policy definition + denied-create test |
| Data permissions | Unity Catalog grants and securable ownership | Positive + negative run-as tests |
| CI security | GitLab branch/environment/runner protection | Rules review + untrusted pipeline simulation |
| Resource cost controls | Cloud budgets, compute rules and policy gates | Budget and plan review |
| Monitoring and response | Cloud and Databricks telemetry plus on-call | Alert delivery and runbook tests |

Terraform, bundle deployment and manual administration must have documented ownership; do not manage the same object from two controllers.

## 5. Reference repository layout

    databricks-platform/
      infrastructure/
        environments/staging/
      bundles/demo-workflows/
      policies/
        terraform/
          deny_destructive.rego
        exceptions/
      scripts/
        summarize_plan.py
      tests/
        static/
        authorization/
        network/
        synthetic/
        observability/
      evidence/
        .gitkeep
      runbooks/
        policy-failure.md
        operational-acceptance.md
      .gitlab-ci.yml

Never commit state, saved Terraform plans, credentials, protected runtime logs, production inventory exports or data samples.

## 6. Policy baseline and severity

| Severity | Example | Outcome |
|---|---|---|
| Critical | Wrong cloud account, exposed credential or protected resource deletion | Immediate NO-GO |
| High | Public exposure, privilege escalation, missing encryption or uncontrolled compute | NO-GO pending remediation |
| Medium | Missing noncritical labels, weak observability or unapproved cost increase | Remediate or formally approve documented exception |
| Low | Formatting, documentation or minor noncritical drift | Track and close per standards |

Security and data-integrity failures are never silently downgraded by a pipeline variable.

## 7. Lab A — Windows PowerShell tooling preflight [SAFE-READ]

~~~powershell
git --version
terraform version
databricks version
python --version
git status --short
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
~~~

Confirm the actual Databricks workspace host, cloud account, subscription/project, region and identity before running any authenticated checks. `demo-staging` is a placeholder profile name.

## 8. Lab B — Static Terraform and bundle validation [SAFE-READ]

From approved checkout locations:

~~~powershell
terraform -chdir=infrastructure/environments/staging fmt -check -recursive
terraform -chdir=infrastructure/environments/staging init -backend=false
terraform -chdir=infrastructure/environments/staging validate
databricks bundle validate --target staging
databricks bundle summary --target staging
~~~

`terraform validate` checks configuration structure; it does not assess real cloud authorization or enforce organizational policy. Bundle validation may require workspace connectivity depending on configuration.

## 9. Terraform plan as policy input [CONTROLLED-READ]

In the protected staging planning job with an authorized backend:

~~~powershell
terraform -chdir=infrastructure/environments/staging init
terraform -chdir=infrastructure/environments/staging plan -out=staging.tfplan
terraform -chdir=infrastructure/environments/staging show -json staging.tfplan |
  Out-File -Encoding utf8 staging-plan.json
~~~

**Security:** Both the plan and JSON may contain secrets or sensitive attributes even when the human-readable CLI shows masked values. Keep files in restricted temporary storage; do not publish raw JSON to unrestricted GitLab artifacts or merge-request comments.

## 10. Lab C — Inspect proposed resource actions [CONTROLLED-READ]

Run this PowerShell from the directory containing the generated plan JSON:

~~~powershell
$plan = Get-Content .\staging-plan.json -Raw | ConvertFrom-Json
$plan.resource_changes |
  ForEach-Object {
    [pscustomobject]@{
      Address = $_.address
      Actions = ($_.change.actions -join ",")
    }
  } | Format-Table -AutoSize
~~~

Distinguish `create`, `update`, `delete`, `no-op`, and replacement expressed as `delete,create` or `create,delete`. Review `resource_drift` separately. Output remains sensitive; publish only redacted summaries.

## 11. Lab D — Policy-as-code denying deletion and replacement

OPA/Conftest example to run only in an **isolated lab** after reviewing the rule with your policy owners.

File: `policies/terraform/deny_destructive.rego`

~~~rego
package main

import rego.v1

deny contains msg if {
  some rc in input.resource_changes
  "delete" in rc.change.actions
  msg := sprintf("Destructive action proposed for %s", [rc.address])
}
~~~

Because replacements include a delete action, the same rule blocks replacements. This deliberately strict example is a starting gate, **not** a complete production policy or a universal reason to reject harmless lifecycle replacements. A real policy requires a documented exception workflow, critical resource classifications, and tests against intended schema.

~~~powershell
conftest test staging-plan.json --policy policies/terraform
if ($LASTEXITCODE -ne 0) { throw "Policy check failed" }
~~~

Run this only where Conftest is installed and the policy directory exists. Pin and verify your approved OPA/Conftest version and policy revision.

## 12. Lab E — Local policy unit test without cloud access

File: `tests/static/destructive-plan.json`

~~~json
{
  "resource_changes": [
    {
      "address": "databricks_cluster.demo_staging",
      "change": { "actions": ["delete", "create"] }
    }
  ]
}
~~~

~~~powershell
conftest test tests/static/destructive-plan.json --policy policies/terraform
$code = $LASTEXITCODE
if ($code -eq 0) { throw "Expected policy denial did not occur" }
"PASS: replacement rejected by deny rule"
~~~

This demonstrates a **fail-closed** policy test: the sample is supposed to be rejected. Add a no-op/create fixture that must pass to prevent policies that reject every change.

## 13. Policy test assertions and quality

A robust policy suite must explicitly test:
- Expected pass cases, not only denials.
- Cloud, environment and resource classification.
- Destructive and replacement actions.
- Encryption and public connectivity restrictions.
- Allowed compute types and runtime families.
- Required ownership, cost-center and expiry tags.
- Run-as identity and permission boundaries.
- Exception expiration, approver identity and change-ticket linkage.
- Unknown or missing attributes: fail closed when critical evidence is absent.

Do not assume Terraform plan JSON alone can prove effective runtime IAM, data grants or network reachability.

## 14. GitLab CI/CD static policy gate

Example **non-deploying** job; use organization-approved pinned images and signatures for production:

~~~yaml
stages:
  - validate
  - policy

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

policy_unit_tests:
  stage: policy
  image:
    name: openpolicyagent/conftest:v0.56.0
    entrypoint: [""]
  script:
    - conftest test tests/static/allowed-plan.json --policy policies/terraform
    - |
      if conftest test tests/static/destructive-plan.json --policy policies/terraform; then
        echo "ERROR: destructive fixture was unexpectedly allowed"
        exit 1
      fi
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
~~~

The `allowed-plan.json` positive fixture must exist in the repository; for example:

~~~json
{"resource_changes":[{"address":"databricks_cluster.demo_staging","change":{"actions":["no-op"]}}]}
~~~

This job tests sample policy fixtures only. A separate **protected** pipeline must securely create and scan the actual plan; do not make a privileged Terraform plan accessible to untrusted forks. Verify CI image versions, runner policies and exact GitLab rules in your environment.

## 15. Secure plan gate design

The protected policy gate should:
1. Read the approved commit SHA and protected deployment environment.
2. Verify cloud account, Databricks host and state backend independently.
3. Build a fresh Terraform plan with short-lived, scoped credentials.
4. Evaluate policy against the actual plan **inside a protected workspace**.
5. Generate a redacted resource-action and rule-result summary.
6. Bind approval to plan digest, policy revision, environment and commit.
7. Reject stale plans and changes to state between approval and apply.
8. Retain restricted evidence with appropriate expiry and access control.

Never use a raw saved plan from staging as a production deployment artifact.

## 16. Compute policy validation [SAFE-READ]

~~~powershell
databricks cluster-policies list --output json
databricks clusters list --output json
~~~

For an authorized policy ID in a staging workspace:

~~~powershell
databricks cluster-policies get "REPLACE_WITH_STAGING_POLICY_ID" --output json
~~~

Review permitted node types, auto-termination, instance pools, runtime family, maximum workers, tags and permissions. Check supported field names against the API and cloud-specific policy schema.

## 17. Compute negative test [STAGING-MUTATION]

Using a disposable staging identity and an approved UI/API-based test, request a cluster configuration prohibited by the policy—for example a worker count exceeding the enforced maximum. **Expected result:** policy rejects the request before resources are provisioned.

Do not run this test in production. A test that accidentally creates a cluster is a failure; terminate the disposable cluster through the approved cleanup process and investigate the policy binding.

## 18. Unity Catalog least-privilege matrix

| Principal | Allowed | Denied |
|---|---|---|
| Staging workload identity | Read/write approved staging schema | Access restricted production schema |
| Staging observer | Read approved monitoring views | Change grants or write data |
| Platform admin | Approved administrative operations | Unapproved break-glass escalation |

Execute as each **actual principal**, not only as a workspace administrator.

## 19. Lab F — Unity Catalog positive authorization [SAFE-READ]

Using an approved SQL warehouse and the staging workload identity:

~~~sql
SELECT current_user();
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_staging;
SELECT COUNT(*) AS row_count
FROM demo_catalog.demo_staging.synthetic_events;
~~~

The objects are sample placeholders. Record the acting principal, execution time and expected grants; do not expose row-level sensitive data.

## 20. Lab G — Unity Catalog negative authorization [STAGING CONTROLLED]

As a nonprivileged, approved test identity, attempt a read against a **staging-only restricted synthetic object** that the identity must not access.

~~~sql
SELECT COUNT(*)
FROM demo_catalog.restricted_staging.synthetic_secret;
~~~

**Pass:** an authorization denial occurs and audit evidence identifies the test principal. A "table not found" error is not necessarily evidence of correct access control; verify object existence using an authorized separate identity.

Never test access denial by revoking production grants.

## 21. Cloud identity validation [SAFE-READ]

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

Identity inventory alone does not prove permissions. Match these values to the expected GitLab workload federation trust, scoped cloud role and target Databricks workspace before any privileged action.

## 22. Network and storage validation [SAFE-READ]

Check the correct cloud-specific network dependencies:

| AWS | Azure | Google Cloud |
|---|---|---|
| VPC routes, security groups, endpoints, S3/KMS access | VNet routes, NSGs, Private Link, ADLS/Key Vault | VPC routes, firewall rules, private access, GCS/KMS |
| AWS audit and flow logs | Azure Activity Logs / Network Watcher | Cloud Audit Logs / VPC Flow Logs |

From an **approved staging runner** test DNS resolution and TCP reachability to the intended workspace or private endpoints. Reachability is not evidence of successful authentication or authorized data access. Avoid probing unauthorized networks.

## 23. Lab H — Databricks API read checks [SAFE-READ]

~~~powershell
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks clusters list --output json
databricks warehouses list --output json
~~~

CLI command surfaces can change; verify with `databricks --help` and subcommand `--help`. Results may be permission-filtered and paginated. A successful API call proves only that particular API operation under that identity.

## 24. Synthetic workload acceptance plan

Use a small approved staging job that:
1. Reads a fixed synthetic input manifest.
2. Performs deterministic transformations.
3. Writes to a dedicated disposable staging dataset.
4. Checks row count, uniqueness and schema.
5. Records duration, retries and cost indicators.
6. Cleans up only its explicitly owned artifacts under separate approval.

Use a distinct dataset or run ID for each test so retries cannot double-write real business data.

## 25. Lab I — Synthetic data invariants [SAFE-READ]

~~~sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(DISTINCT event_id) AS distinct_event_ids,
  SUM(CASE WHEN event_id IS NULL THEN 1 ELSE 0 END) AS null_ids,
  MAX(event_time) AS newest_event_time
FROM demo_catalog.demo_staging.synthetic_events;
~~~

Compare results with the expected fixture manifest. A row count by itself is insufficient: schema changes, duplicate writes and stale timestamps can still break consumers.

## 26. Performance and reliability acceptance

Define test thresholds **before** running:
- Synthetic job completion time and success rate.
- Cluster startup latency and task retries.
- API error/retry budget, throttling and quota headroom.
- Job queue wait time and concurrency.
- Data freshness and downstream consumer checks.
- Logs, traces, metrics and alert routing.
- Recovery objective and rollback time.

Thresholds must be environment-specific and based on recorded baselines/SLOs; this chapter intentionally invents no universal latency or cost target.

## 27. Lab J — Read-only recent run inventory [SAFE-READ]

~~~powershell
databricks jobs list-runs --limit 25 --output json
~~~

Inspect failure and retry patterns, run-as principal, queueing, termination codes and operation timelines for authorized jobs. Correlate with cloud capacity, workspace audit logs and GitLab pipeline execution.

## 28. Observability and alert delivery tests

Validate:
- A deliberately generated **benign staging** synthetic alert reaches its intended channel.
- Notification contains workspace, environment, severity, runbook, owner and correlation ID.
- Alert resolves with the synthetic condition.
- No production paging occurs from staging.
- Audit and telemetry retention meet policy.

Do not inject production outages to prove paging works. An acknowledged test notification is not proof that the incident can be recovered.

## 29. Failure injection boundaries

Permitted under approved isolated staging change:
- Invalid non-sensitive policy fixture.
- Prohibited staging compute request.
- Synthetic missing grant on a disposable restricted object.
- Simulated plan deletion/replacement fixture.
- Synthetic job that intentionally fails before side effects.
- Benign alert-routing test.

Excluded by default: production IAM revocations, storage destruction, real credential exposure, forced data corruption, accidental cost spikes or large-scale capacity exhaustion.

## 30. Scenario A — Terraform plan passes syntax but violates policy

**Detect:** valid Terraform plan proposes a prohibited instance class.  
**Triage:** inspect resource address, effective policy and documented exception.  
**Response:** reject protected gate; modify configuration or obtain a time-bound authorized exception.  
**Pass:** allowed configuration passes both policy and runtime checks.

## 31. Scenario B — Plan replacement bypasses destroy-only text scan

**Detect:** `["delete","create"]` or `["create","delete"]` in JSON actions.  
**Response:** use structured JSON policy parsing, not grep of the human-readable plan.  
**Pass:** all replacement combinations are detected.

## 32. Scenario C — Test runs under admin, workload principal fails

**Detect:** admin SQL succeeds; job run-as principal receives an authorization error.  
**Response:** validate effective identity and minimal required grants.  
**Pass:** workload succeeds under its real identity; restricted principal remains denied.

## 33. Scenario D — Workspace host or cloud account mismatch

**Detect:** token resolves against unexpected Databricks host/cloud tenant/account.  
**Response:** stop testing and deployment immediately; inspect protected variables, profiles and federation.  
**Pass:** independent account, state and host checks agree with approved target.

## 34. Scenario E — Compute policy exists but does not bind target user

**Detect:** forbidden test cluster configuration is accepted in staging.  
**Response:** investigate permissions, default policy selection and enforcement path.  
**Pass:** expected policy denial with no cluster created.

## 35. Scenario F — Policy rule changes without approval

**Detect:** policy digest/version differs from reviewed release.  
**Response:** invalidate approval and rerun policy tests and secure plan evaluation.  
**Pass:** evidence binds policy revision to exact plan and commit.

## 36. Scenario G — Operational test is green while consumer data is stale

**Detect:** job succeeds but synthetic freshness or downstream query check fails.  
**Response:** stop release promotion and inspect pipelines, retries and storage visibility.  
**Pass:** all data invariants and freshness windows meet documented baseline.

## 37. Scenario H — Staging alarm pages production on-call unexpectedly

**Detect:** synthetic alert routes to live critical pager instead of designated staging channel.  
**Response:** stop test; correct alert labels/routing and document incident.  
**Pass:** intended channel receives and resolves the synthetic alert.

## 38. Scenario I — Cloud quota exhausted during staging validation

**Detect:** compute creation fails due to quota/capacity, unrelated to logical policy denial.  
**Response:** classify as infrastructure capacity failure, not a passed policy negative test.  
**Pass:** capacity is restored or approved test scope adjusted; policies remain effective.

## 39. Scenario J — Evidence is missing or cannot be reproduced

**Detect:** pass/fail claimed without commit, policy version, principal, workspace, execution time or expected outcome.  
**Response:** classify gate as inconclusive and rerun in controlled staging.  
**Pass:** evidence is complete, restricted and traceable.

## 40. Production gate runbook

1. Verify change ticket, release revision and owner.
2. Identify scope, cloud account, workspace and target environment.
3. Verify protected runner, environment approval and short-lived principal.
4. Confirm backend, state revision and plan source.
5. Execute static validation and policy unit tests.
6. Run protected plan-time policy evaluation and redact summary.
7. Obtain required approvals for exceptions and destructive actions.
8. Complete approved staging apply and bundle promotion tests.
9. Verify effective compute rules, UC grants and cloud boundaries.
10. Execute deterministic synthetic job and data acceptance tests.
11. Verify performance baseline, metrics, alert delivery and runbooks.
12. Review rollback, data restore, writer fencing and replay readiness.
13. Record GO/CONDITIONAL GO/NO-GO and release evidence.
14. Apply production changes only under separately approved deployment procedures.

## 41. Response and rollback

**Policy gate failure:** stop before apply; correct configuration or route a formal exception. Do not bypass with `allow_failure: true` or unreviewed variable overrides.

**Runtime test failure:** halt promotion, protect data writers and inspect scope. Roll back using a **reviewed compensating plan** and/or last-known-good bundle deployment; restore data separately if required. A Git revert does not automatically reverse state changes or data writes.

**Credential/runner compromise:** revoke/restrict identities, isolate runner, retain logs securely and investigate unauthorized changes before resuming.

## 42. Evidence package

    policy-validation-YYYYMMDD/
      01-change-and-target.md
      02-reviewed-commit.md
      03-static-validation-summary.md
      04-plan-action-redacted-summary.md
      05-policy-revision-and-results.md
      06-approvals-and-exceptions.md
      07-run-as-access-matrix.md
      08-network-and-cloud-identity.md
      09-synthetic-job-and-data-results.md
      10-monitoring-and-alert-evidence.md
      11-recovery-readiness.md
      12-final-go-no-go.md

Retain only appropriately redacted evidence in approved restricted storage; saved plans, state, tokens, data and full privileged API responses are not appropriate public artifacts.

## 43. Acceptance scorecard

| Acceptance gate | Evidence | Required result |
|---|---|---|
| Approved target | Account, host, environment, region | Exact match |
| Runner security | Protected runner and identity evidence | Trusted |
| Terraform / bundle syntax | Validation logs | Pass |
| Policy unit tests | Positive and negative fixtures | Pass |
| Proposed changes | Restricted plan summary | Approved |
| Destructive action control | Structured actions and approval | No unapproved delete/replace |
| Cloud identity | AWS/Azure/GCP principal record | Expected principal |
| Compute control | Read policy and staging denied-create test | Enforced |
| Unity Catalog privileges | Real-principal positive and negative checks | Least privilege |
| Network/storage | Staging endpoint and service checks | Required connectivity |
| Synthetic workload | Run record and deterministic assertions | Pass |
| SLO baseline | Startup, success, retry and freshness measurements | Within defined limits |
| Alerting | Benign staging alert evidence | Correct route |
| Recovery | Validated rollback/data recovery plan | Ready |
| Audit | Revision, time, owner, approval and exception | Complete |

## 44. GO / CONDITIONAL GO / NO-GO

**GO:** Every mandatory policy gate passes, the target is verified, positive/negative auth tests succeed, synthetic job/data acceptance passes, monitoring functions and recovery readiness is documented.

**CONDITIONAL GO:** A noncritical, specifically permitted control has a signed, time-bounded exception with owner and compensating measures. Never use conditional approval to bypass credential security, environment correctness or data-integrity checks.

**NO-GO:** Unapproved deletion/replacement, wrong workspace, missing privileges, unaudited bypass, exposed secret, stale or absent evidence, failed consumer correctness or no viable rollback/recovery plan.

## 45. Review questions

1. Why must Terraform plan policy checks parse structured resource actions?
2. What is the difference between positive and negative authorization tests?
3. Why is a successful admin query not evidence that a workload principal can read data?
4. Which tests belong in isolated staging, and which are safe for production read-only discovery?
5. How do you prove the tested policy revision matches the approved deployment?
6. Why is a successful job status insufficient for release acceptance?
7. Which evidence is safe for a merge-request summary and which must stay restricted?
8. How can a rollback require data recovery even after infrastructure configuration is reverted?

## 46. Official references

- GitLab CI/CD: https://docs.gitlab.com/ci/
- GitLab protected environments: https://docs.gitlab.com/ci/environments/protected_environments/
- GitLab CI/CD security: https://docs.gitlab.com/ci/security/
- Terraform validate: https://developer.hashicorp.com/terraform/cli/commands/validate
- Terraform show JSON: https://developer.hashicorp.com/terraform/internals/json-format
- Terraform plan: https://developer.hashicorp.com/terraform/cli/commands/plan
- OPA Terraform policy: https://www.openpolicyagent.org/docs/terraform
- Conftest: https://www.conftest.dev/
- Databricks compute policies (AWS): https://docs.databricks.com/aws/en/compute/policies
- Databricks Unity Catalog privileges (AWS): https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/
- Databricks audit logs (AWS): https://docs.databricks.com/aws/en/admin/account-settings/audit-logs
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks bundles: https://docs.databricks.com/aws/en/dev-tools/bundles/

**Next:** Chapter 050 — Production Change Management and Release Acceptance.
