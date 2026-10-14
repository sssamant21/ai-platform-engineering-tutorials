# Chapter 050 — Production Change Management and Release Acceptance

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 10 — Automation and GitOps  
**Audience:** SRE, Platform Engineering, Cloud Infrastructure, Security, Release Managers  
**Clouds:** AWS, Azure, Google Cloud  
**Prerequisites:** Chapters 041–049  
**Safety:** Production deployments, grants, cluster changes, job schedule changes, and rollback require approved change authorization. All active exercises use isolated staging. Production discovery is read-only.

## 1. Mission

Design a repeatable, auditable production change process for Databricks infrastructure and managed workloads. A deployment is not complete when Terraform apply or a Databricks bundle deploy exits successfully: service behavior, data correctness, monitoring, ownership and rollback must be accepted.

## 2. Learning outcomes

- Classify standard, normal, high-risk and emergency changes.
- Define change scope, blast radius, dependencies and maintenance windows.
- Implement pre-deployment, deployment and post-deployment gates.
- Bind approvals to immutable revisions and reviewed infrastructure plans.
- Coordinate SRE, security, data owners and downstream consumers.
- Set objective abort and rollback criteria.
- Validate production health, data freshness and access control.
- Build an evidence package and formally close a change.

## 3. Change lifecycle

    Request -> Risk assessment -> Peer review -> Staging validation
       -> Protected approval -> Preflight -> Controlled deployment
       -> Runtime + data acceptance -> Observe -> Sign-off -> Closure
                                        |
                                     Failure
                                        |
                              Stop / contain / recover
                                        |
                              Validate / incident review

## 4. Change types

| Type | Examples | Approval |
|---|---|---|
| Standard | Preapproved low-risk, repeatable configuration | Documented standard process |
| Normal | Workspace configuration, compute policy or bundle release | Owner and designated change approver |
| High risk | IAM, networking, Unity Catalog, state migration, data access | Security/data owners and explicit rollback review |
| Emergency | Restore availability or contain security incident | Incident-linked emergency authorization and retrospective |

Do not use the emergency path to bypass normal review for convenience.

## 5. Scope and blast radius

Record affected cloud account/subscription/project, region, workspace, Terraform state, network path, cluster policy, Unity Catalog metastore/catalog, service principal, jobs, schedules, external storage and downstream consumers. Distinguish control-plane changes from data-plane changes and identify whether running jobs or existing sessions will be affected.

## 6. Change risk matrix

| Dimension | Low | Elevated |
|---|---|---|
| Scope | Isolated staging or single disposable resource | Shared production workspace or cross-region |
| Reversibility | Stateless setting with tested reversion | State migration, IAM lockout, data mutation |
| Availability | No active workload interruption | Scheduled jobs or consumers may fail |
| Data integrity | Synthetic/no data effects | Schema, checkpoint, permission or write changes |
| Observability | Validated dashboards and paging | Missing baseline or unknown alert behavior |
| Recovery | Tested restore path | Unproven recovery or RPO/RTO exposure |

Any unmitigated critical risk is a NO-GO.

## 7. Responsibility and approvals

| Role | Responsibility |
|---|---|
| Change owner | Scope, implementation, communications, evidence |
| SRE/on-call | Health baselines, abort criteria, observation, recovery |
| Cloud/platform owner | Terraform state, cloud IAM, networking and workspace |
| Security | Identity, least privilege, secret and audit implications |
| Data owner | Schema, freshness, data correctness, replay and consumer acceptance |
| Approver | Risk acceptance and protected production authorization |
| Incident commander | Coordinates recovery if deployment causes an incident |

Separate proposer, reviewer and production executor where policy requires.

## 8. Preconditions and freeze windows

Check deployment freeze calendar, competing maintenance, upstream ingestion, peak traffic, downstream SLAs, regional capacity, on-call coverage and support availability. Avoid overlapping high-risk changes against the same workspace, state or metastore.

## 9. Release identity and artifact integrity

Record repository, commit SHA, Terraform version/provider lock file, policy revision, plan digest, Databricks CLI version, bundle target, cloud account, workspace URL and approver. A plan is specific to the state and environment against which it was created. Replan and reapprove if the plan is stale or relevant inputs change.

## 10. Lab A — Windows PowerShell preflight [SAFE-READ]

~~~powershell
git status --short
git rev-parse HEAD
git remote -v
terraform version
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

Compare output against the approved staging change record. Do not assume the active shell profile or cloud context is correct.

## 11. Lab B — Cloud identity and target validation [SAFE-READ]

**AWS**
~~~powershell
aws sts get-caller-identity
aws configure get region
~~~

**Azure**
~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
~~~

**GCP**
~~~powershell
gcloud config get-value project
gcloud auth list
~~~

Match the expected identity, account and region to the approved change ticket. Never continue on a mismatch.

## 12. Lab C — Terraform validation [SAFE-READ]

~~~powershell
terraform -chdir=infrastructure/environments/staging fmt -check -recursive
terraform -chdir=infrastructure/environments/staging init -backend=false
terraform -chdir=infrastructure/environments/staging validate
~~~

Static validation cannot prove policy compliance or deployment safety.

## 13. Lab D — Protected plan review [CONTROLLED-READ]

Only in an authorized staging environment with a protected backend:

~~~powershell
terraform -chdir=infrastructure/environments/staging init
terraform -chdir=infrastructure/environments/staging plan -out=review.tfplan
terraform -chdir=infrastructure/environments/staging show -no-color review.tfplan
~~~

Review additions, modifications, replacements, destroys, IAM changes, network exposure, storage and cost. Plan files may contain secrets: never commit or publish them to unrestricted CI artifacts.

## 14. Lab E — Bundle validation [SAFE-READ]

~~~powershell
databricks bundle validate --target staging
databricks bundle summary --target staging
~~~

Confirm workspace host, resource ownership, run-as identity, target mode, permissions and schedule activation. A bundle's production mode is not a security approval gate.

## 15. Lab F — Inventory workloads and policy [SAFE-READ]

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks cluster-policies list --output json
databricks warehouses list --output json
~~~

Outputs may be paginated or permission-filtered. Preserve only redacted inventory in approved evidence storage.

## 16. Data-protection preflight

For changes affecting writes or schemas, document:
- Last verified backup or recoverable table version and retention constraints.
- Delta transaction/checkpoint consistency.
- Current stream offsets and job concurrency.
- Writer fencing and replay/idempotency strategy.
- Expected data freshness, uniqueness and schema invariants.
- Recovery point and recovery time objectives.
- Impact on external consumers and governance.

Never assume Git revert or Terraform rollback restores mutated data.

## 17. Dependency and maintenance-window planning

Sequence network/IAM, workspace connectivity, governance and storage, compute policies, jobs, synthetic checks and schedule activation. Identify exact pause/resume owners and communicate when consumer freshness may be affected.

## 18. Pre-deployment acceptance checklist

- Approved ticket and named accountable owner.
- Exact immutable revision and environment.
- Static, security, policy and cost gates passed.
- Reviewed plan with no unapproved delete/replace.
- Staging test results and exception approvals recorded.
- Cloud quota and regional capacity confirmed.
- Valid backup/recovery and writer-control procedure.
- Baseline dashboards and on-call routing verified.
- Stakeholders notified and abort authority assigned.

## 19. GitLab protected deployment design

Use a protected environment, trusted runner, short-lived deployment identity and manual authorization for production. Prevent untrusted merge requests from reading production variables, state, plan files or workspace tokens. A manual job alone is not sufficient authorization unless protected environment and role rules are enforced.

Illustrative non-deploying gate:

~~~yaml
stages:
  - validate
  - approve

static_validation:
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

release_approval_record:
  stage: approve
  script:
    - echo "Approval placeholder only; no production deployment"
  when: manual
  allow_failure: false
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
~~~

This example is **not** a complete protected production gate; configure GitLab protected environments, reviewers and runner access separately. Pin approved container digests in production.

## 20. Deployment execution protocol

1. Open the change bridge or agreed communications channel.
2. Record UTC start, executor and approved revision.
3. Reconfirm target workspace/account and deployment identity.
4. Confirm approved plan and state freshness.
5. Pause only specifically authorized writers or schedules.
6. Execute the reviewed deployment using the approved pipeline.
7. Record actual actions and errors.
8. Run security, platform and synthetic acceptance checks.
9. Observe baseline metrics and consumer behavior.
10. Resume schedules only after data-integrity checks pass.
11. Announce GO, rollback or incident escalation.

Do not manually re-run a failed apply without understanding partial state and lock ownership.

## 21. Lab G — Synthetic read-only data acceptance [SAFE-READ]

Against an authorized synthetic staging table:

~~~sql
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT event_id) AS unique_event_ids,
  SUM(CASE WHEN event_id IS NULL THEN 1 ELSE 0 END) AS null_event_ids,
  MAX(event_time) AS latest_event_time
FROM demo_catalog.demo_staging.synthetic_events;
~~~

Compare with a known fixture manifest and predeclared freshness target. A successful SQL query or job exit code alone does not establish correctness.

## 22. Lab H — Unity Catalog access [SAFE-READ]

~~~sql
SELECT current_user();
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_staging;
~~~

Use the actual workload principal for positive access tests and a separately approved nonprivileged test identity for negative tests. Never revoke production grants to simulate failure.

## 23. Lab I — Post-deployment health [SAFE-READ]

~~~powershell
databricks auth describe
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks warehouses list --output json
~~~

Correlate runs with deployment time, cloud metrics, API errors, retry rates, startup latency, query latency, freshness and consumer error budgets. CLI availability and command syntax depend on installed version.

## 24. Operational acceptance thresholds

Define before deployment:

| Signal | Acceptance criterion |
|---|---|
| Availability | No breach of documented service SLO |
| Job success | Synthetic and required production jobs meet baseline |
| Data correctness | Counts, uniqueness, schema and expected manifest agree |
| Freshness | Within agreed consumer SLA |
| Latency | Within predeclared baseline or error budget |
| Capacity | No unresolved quota/compute provisioning errors |
| Access | Authorized identities succeed, prohibited identities denied |
| Alerts | Required telemetry and routing functioning |
| Recovery | Rollback/restore decision and owner ready |

Use measured service-specific thresholds; there is no universal acceptable Databricks job duration.

## 25. Observation period and sign-off

Specify a minimum observation window based on job cadence, business-critical consumers and change risk. A five-minute green dashboard cannot validate a daily scheduled job. Close only after the relevant consumer and workload checks have completed or explicitly document outstanding monitoring with an accountable owner.

## 26. Abort criteria

Immediately stop promotion or further changes for:
- Wrong account, workspace, region or state.
- Unapproved destruction/replacement or privilege escalation.
- Credential exposure or untrusted runner execution.
- Data corruption, duplicate writes or lost checkpoint ownership.
- Breached critical consumer SLO or failed synthetic invariant.
- Loss of required telemetry or unavailable recovery path.

Escalate to incident command when impact exceeds the approved change envelope.

## 27. Rollback decision tree

    Failure detected?
        |
       Yes
        |
    Is continued execution safe? -- No --> Stop/fence writers
        |
    Can reviewed config reversion restore service?
        | Yes                        | No
    Compensating plan / bundle     Data/identity/network recovery
        |                             |
    Verify runtime + data <----------+
        |
    Restore traffic/schedules only after acceptance

Terraform has no universal rollback command. For data mutations use approved restore, replay or compensation procedures.

## 28. Lab J — Staging rollback tabletop

Without modifying production:
1. Choose a hypothetical failed compute-policy change.
2. Identify the last-known-good configuration and its revision.
3. Record how to identify affected jobs and identities.
4. Decide whether a compensating Terraform plan or bundle redeploy is needed.
5. Determine whether running jobs must be stopped or allowed to drain.
6. Specify success checks for policy, job runs and data freshness.
7. Estimate recovery time against the documented RTO.
8. Produce a tabletop evidence record and identify gaps.

**Pass:** an independent operator can follow the written procedure and knows when to escalate.

## 29. Scenario A — Wrong production workspace selected

**Detect:** workspace host does not match ticket.  
**Response:** NO-GO; stop before apply, correct protected identity/profile.  
**Acceptance:** independent host/account/state verification.

## 30. Scenario B — Unexpected Terraform destroy

**Detect:** reviewed plan contains delete/replacement.  
**Response:** block deployment; investigate ownership and approved migration.  
**Acceptance:** reviewed safe plan or explicitly approved controlled migration.

## 31. Scenario C — Change approved but plan becomes stale

**Detect:** state, policy or code changed after approval.  
**Response:** regenerate plan and repeat review/approval.  
**Acceptance:** approved revision and plan/state bindings match.

## 32. Scenario D — GitLab runner receives production credentials in untrusted job

**Detect:** CI variables, audit events or logs show exposure.  
**Response:** isolate runner, revoke/rotate credentials, preserve evidence.  
**Acceptance:** protected isolation restored and security review complete.

## 33. Scenario E — Compute capacity unavailable

**Detect:** cluster startup fails due to regional capacity or quota.  
**Response:** defer, use preapproved capacity alternative or roll back.  
**Acceptance:** workload runs within performance and cost guardrails.

## 34. Scenario F — Unity Catalog grant regression

**Detect:** workload identity receives access denied after release.  
**Response:** restore least-privilege grant through approved change.  
**Acceptance:** expected identity works and prohibited access remains denied.

## 35. Scenario G — Successful deployment, failed downstream consumer

**Detect:** freshness, schema or output invariant fails.  
**Response:** treat as failed release; fence unsafe writers and recover data path.  
**Acceptance:** consumer correctness restored, not merely pipeline green.

## 36. Scenario H — Concurrent jobs double-write after resume

**Detect:** duplicate IDs, concurrent writers or checkpoint conflict.  
**Response:** stop overlapping writers, assess affected transactions, execute approved dedupe/replay.  
**Acceptance:** exactly intended output and single-writer control.

## 37. Scenario I — Rollback config succeeds but data remains inconsistent

**Detect:** previous Git revision deployed yet output diverges.  
**Response:** run separate data recovery/compensation and consumer validation.  
**Acceptance:** integrity, freshness and access tests all pass.

## 38. Scenario J — Alerts absent during change

**Detect:** monitoring ingestion or alert routing unavailable.  
**Response:** NO-GO for high-risk promotion; restore telemetry and paging.  
**Acceptance:** synthetic benign alert routes correctly and resolves.

## 39. Incident transition runbook

Declare incident severity according to business impact. Assign incident commander, SRE lead, communications lead and data owner. Freeze concurrent changes, preserve logs and audit history, identify first failing signal, isolate scope, choose recovery method, record timestamps and validate recovery against service/data SLOs. Follow up with RCA and corrective actions.

## 40. Production change record template

~~~markdown
# Databricks Production Change
- Ticket / change type:
- Business purpose:
- Owner / approver / executor:
- Cloud / region / workspace:
- Affected resources and consumers:
- Commit / plan digest / policy revision:
- Maintenance window (timezone):
- Baseline SLOs / data freshness:
- Preflight and staging evidence:
- IAM / networking / cost impact:
- Data protection and writer strategy:
- Deployment sequence:
- Abort triggers:
- Recovery owner / RPO / RTO:
- Observation window:
- Stakeholder communications:
- GO / CONDITIONAL GO / NO-GO:
- Actual start/end UTC:
- Final verification and closure:
~~~

## 41. Evidence package

    change-YYYYMMDD/
      01-request-risk-and-scope.md
      02-reviewed-revision.md
      03-static-security-policy-results.md
      04-redacted-plan-summary.md
      05-approvals.md
      06-baseline-and-preflight.md
      07-deployment-timeline.md
      08-runtime-and-access-checks.md
      09-data-and-consumer-acceptance.md
      10-observation-and-alerts.md
      11-recovery-readiness.md
      12-final-decision-and-closure.md

Keep sensitive plan/state, tokens and raw privileged logs in restricted storage, not public Git.

## 42. Acceptance scorecard

| Gate | Required result |
|---|---|
| Scope and risk | Documented and approved |
| Target account/workspace | Independently verified |
| Protected CI identity | Trusted and least privilege |
| Immutable release | Commit, plan and policy recorded |
| Static and policy checks | Passed |
| Destructive actions | None unapproved |
| Staging evidence | Current and relevant |
| Recovery readiness | Tested or explicitly approved |
| Runtime/API health | Within baseline |
| UC/security | Least privilege verified |
| Data correctness | Invariants and consumers pass |
| Alerts and telemetry | Available and tested |
| Observation | Covers affected workload cadence |
| Closure | Owner and approver sign-off |

## 43. GO / CONDITIONAL GO / NO-GO

**GO:** All mandatory controls and runtime/data acceptance checks pass, with documented approval and recovery readiness.

**CONDITIONAL GO:** Only explicitly permitted noncritical gaps remain with signed, time-bound exceptions, owners and compensating controls.

**NO-GO:** Wrong target, exposed credential, unapproved destructive plan, failed data-integrity checks, absent critical monitoring, untested critical recovery path or missing approval.

## 44. Review questions

1. Why is a green GitLab pipeline not sufficient for production release acceptance?
2. When should a change be classified as high risk?
3. How do you bind a Terraform plan to its approval?
4. What is the difference between a configuration rollback and a data recovery?
5. Why should observation periods account for job cadence?
6. What signals should immediately abort a release?
7. How do you prevent simultaneous writers during recovery?
8. What evidence is needed to close a production change?

## 45. Official references

- GitLab protected environments: https://docs.gitlab.com/ci/environments/protected_environments/
- GitLab CI/CD: https://docs.gitlab.com/ci/
- Terraform plan: https://developer.hashicorp.com/terraform/cli/commands/plan
- Terraform state: https://developer.hashicorp.com/terraform/language/state
- Databricks Terraform: https://docs.databricks.com/aws/en/dev-tools/terraform/
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks bundles: https://docs.databricks.com/aws/en/dev-tools/bundles/
- Databricks compute policies: https://docs.databricks.com/aws/en/compute/policies
- Unity Catalog privileges: https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/
- AWS operational excellence: https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/welcome.html
- Azure operational excellence: https://learn.microsoft.com/azure/well-architected/operational-excellence/
- Google Cloud operational excellence: https://cloud.google.com/architecture/framework/operational-excellence

**Next:** Chapter 051 — Infrastructure Drift Detection and Configuration Compliance.
