# Chapter 059 — Databricks Production Operations and Enterprise Reliability Acceptance Project

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 13 — Capacity, Cost Governance and Enterprise Acceptance  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform Engineering, Data Engineering, Security, FinOps, Incident Commanders  
**Prerequisites:** Chapters 001–058  
**Project:** Integrated enterprise production-readiness acceptance  
**Safety:** Execute labs in an isolated staging environment using synthetic data. Production inventory is read-only. Any production changes, replay, grants, DR promotion, fault injection or paging require separate authorization. Never commit tokens, Terraform state, private audit logs or sensitive datasets.

## 1. Mission

Deliver an evidence-based production-readiness assessment for a Databricks data platform. Integrate architecture, cloud IAM, networking, Unity Catalog, Terraform, GitLab CI/CD, Databricks Asset Bundles, jobs, Delta Lake, monitoring, security, SLOs, disaster recovery, performance and FinOps into one reproducible enterprise acceptance project.

The goal is not to produce a green checklist by assertion. Each critical gate must have observable evidence, a responsible owner, an explicit pass/fail rule and a safe recovery procedure.

## 2. Project outcomes

- Inventory an entire workload and its regional/cloud dependencies.
- Validate deployment reproducibility and target isolation.
- Test identity, least privilege, storage access and Unity Catalog governance.
- Prove deterministic data correctness, freshness and idempotency.
- Establish job, SQL, compute and cost baselines.
- Validate alert delivery, incident escalation and observability freshness.
- Rehearse a safe failure and recovery in staging.
- Review backup and DR readiness with RPO/RTO evidence.
- Produce an enterprise GO / CONDITIONAL GO / NO-GO decision package.

## 3. Enterprise reference architecture

    GitHub / GitLab protected release
                 |
       Terraform + Bundles + approvals
                 |
    AWS / Azure / GCP identity and network
                 |
      Databricks workspace + UC governance
                 |
        Jobs / SQL / streaming workloads
                 |
        Delta data + quality contracts
                 |
       Consumer freshness and correctness
                 |
    Observability / audit / SLO / FinOps
                 |
      Backup / DR / incident response
                 |
        Enterprise acceptance board

This is a reference workflow, not a claim that all clouds use identical workspace, identity, network or metastore provisioning semantics.

## 4. Project scope and scenario

A synthetic enterprise pipeline ingests events, validates their schema, deduplicates by event ID, writes a governed Delta output and exposes it to an authorized SQL consumer. Its production analogue has an agreed delivery deadline, cost envelope, access matrix and regional recovery objective.

**Staging acceptance fixture:** deterministic records with stable event IDs, event times, expected count, uniqueness, schema and checksum. Never use real patient, customer or confidential production data in the exercise.

## 5. Acceptance roles

| Role | Responsibility |
|---|---|
| Platform owner | Workspace, policies, compute and infrastructure |
| Data owner | Source contract, output integrity and freshness |
| Security owner | Identity, grants, secrets and audit |
| SRE lead | SLOs, alerting, incident response and recovery |
| Cloud owner | Networking, IAM, KMS, storage and quotas |
| FinOps owner | Cost baseline, attribution and budget |
| Change approver | Release authorization and rollback |
| Acceptance chair | Final evidence and sign-off |

Record accountable names in the internal acceptance package, not public tutorial artifacts.

## 6. Acceptance prerequisites

Confirm an isolated staging workspace, an approved cloud account/subscription/project, a staging-only service principal, a synthetic catalog/schema, restricted GitLab runner, protected Terraform backend, approved bundle targets, read-only monitoring permissions, synthetic alert route and a documented recovery point.

## 7. Suggested repository layout

~~~text
databricks-enterprise-acceptance/
  README.md
  databricks.yml
  resources/
    jobs.yml
  src/
    synthetic_pipeline.py
  infrastructure/
    environments/
      staging/
      dr-staging/
  sql/
    acceptance_checks.sql
  tests/
    test_contract.py
  gitlab/
    acceptance.yml
  evidence/
    README.md
  runbooks/
    incident.md
    rollback.md
    disaster-recovery.md
  acceptance/
    scorecard.md
    signoff.md
~~~

Keep generated evidence, raw state, credentials and confidential inventory outside the public repository.

## 8. Gate 1 — Identity and target isolation

Verify actual workspace URL, cloud account, region and principal. A profile or bundle target name alone does not establish correct deployment destination. The staging and DR environments must be distinguishable by independent identity and inventory evidence.

## 9. Lab A — Operator preflight [SAFE-READ]

~~~powershell
git rev-parse HEAD
terraform version
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

**Pass:** profile host, identity and cloud context match the approved staging charter. **Fail:** wrong target, missing identity or unknown principal. Stop before proceeding.

## 10. Gate 2 — Infrastructure reproducibility

Require reviewed Terraform modules, pinned provider versions, secure state, drift detection, explicit resource ownership and a validated recovery plan. A clean static validation does not prove a safe apply.

## 11. Lab B — Terraform static acceptance [SAFE-READ]

~~~powershell
terraform -chdir=infrastructure/environments/staging fmt -check -recursive
terraform -chdir=infrastructure/environments/staging init -backend=false
terraform -chdir=infrastructure/environments/staging validate
~~~

For a real deployment, a separate authorized pipeline must initialize its protected backend, generate a reviewed plan and enforce approval before apply. Do not copy raw plan or state files into public evidence.

## 12. Gate 3 — Bundle deployment controls

Verify Databricks Asset Bundle target, workspace host, run-as principal, job ownership, schedules, compute policies, environment isolation and rollback commit. Promotion should be immutable and traceable to a protected commit.

## 13. Lab C — Bundle validation [SAFE-READ]

~~~powershell
databricks bundle validate --target staging
databricks bundle summary --target staging
~~~

**Pass:** target and resources match approved inventory. **Fail:** unknown host, unexpected principal, missing resource or unauthorized schedule. No deployment is performed by this lab.

## 14. Gate 4 — Cloud dependencies

Validate object storage, network paths, private endpoints, IAM, KMS/key availability, service quotas, regional capacity and cloud monitoring. Record external dependencies that cannot be reproduced solely through Terraform.

## 15. Lab D — Multi-cloud context [SAFE-READ]

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

Execute the commands relevant to the target cloud and compare results with approved recovery inventory.

## 16. Gate 5 — Unity Catalog governance

Define an access matrix for catalog, schema, tables, external locations and storage credentials. Test authorized workload access and denied unauthorized access using explicitly approved test principals. An admin's successful query does not prove least privilege.

## 17. Lab E — Governance discovery [SAFE-READ]

~~~sql
SELECT current_user();
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_staging;
~~~

**Pass:** effective access matches the approved matrix. **Fail:** unexpected broad grants, missing workload access or unowned resources. Never add broad grants simply to pass the test.

## 18. Gate 6 — Deterministic data quality

Define schema, required fields, unique business keys, accepted event time, row-count expectations, source completeness and downstream read contracts. Include a documented strategy for late events, retries, checkpoint ownership and source replay.

## 19. Lab F — Synthetic data contract [SAFE-READ]

~~~sql
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT event_id) AS unique_events,
  SUM(CASE WHEN event_id IS NULL THEN 1 ELSE 0 END) AS null_event_ids,
  MAX(event_time) AS latest_event_time
FROM demo_catalog.demo_staging.synthetic_events;
~~~

Compare with an independently stored fixture manifest. **Pass:** counts, uniqueness, schema and freshness all satisfy the approved contract. A successful SQL query alone is not acceptance.

## 20. Gate 7 — Job execution and retry safety

Record job definitions, dependencies, run-as identity, concurrency, retry policy, timeout, cluster selection, schedule and task outcomes. Verify bounded retries and idempotency, including any external side effects.

## 21. Lab G — Job and compute inventory [SAFE-READ]

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks cluster-policies list --output json
databricks warehouses list --output json
~~~

For a known authorized run:

~~~powershell
databricks jobs get-run REPLACE_WITH_RUN_ID --output json
~~~

**Pass:** correct run-as identity, successful required run, bounded retries and approved compute policy. Paginate results; an initial sample may be incomplete.

## 22. Gate 8 — SQL and compute performance

Establish workload-specific latency and queueing baselines, compute startup, utilization, shuffle spill, data scan volume, concurrency and cloud quota headroom. Do not compare unrelated workload populations or treat an arbitrary P95 threshold as a business SLO.

## 23. Lab H — Performance source discovery [SAFE-READ]

~~~sql
SHOW TABLES IN system.query;
DESCRIBE TABLE system.query.history;
SELECT * FROM system.query.history LIMIT 10;
~~~

Inspect the account's actual schema, permissions and data freshness before constructing aggregate latency queries. Query text and identity can be sensitive.

## 24. Gate 9 — SLOs and alert readiness

Define job success, data freshness, query performance and consumer availability SLIs. Record numerator, denominator, exclusion policy, window, objective and telemetry freshness. Test synthetic staging alert delivery to the correct owner and validate deduplication and recovery notifications.

## 25. Lab I — Audit and source freshness [SAFE-READ]

~~~sql
SELECT MAX(event_time) AS latest_event, COUNT(*) AS event_count
FROM system.access.audit
WHERE event_date >= date_sub(current_date(), 1);
~~~

**Pass:** telemetry freshness meets approved tolerance and alternate evidence is available for gaps. Audit records may arrive late and may not cover all activities in every deployment.

## 26. Gate 10 — Cost and capacity

Record Databricks usage by SKU/unit, cloud costs, workload attribution, idle compute, retries, storage growth, concurrency and forecast. Use effective contracted pricing for monetary totals. Verify that lower spend does not violate correctness or SLOs.

## 27. Lab J — Billing baseline [SAFE-READ]

~~~sql
SELECT
  usage_date,
  sku_name,
  usage_unit,
  SUM(usage_quantity) AS usage
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 7)
GROUP BY usage_date, sku_name, usage_unit
ORDER BY usage_date DESC, sku_name;
~~~

**Pass:** data freshness and SKU/unit attribution documented; spend reconciled with approved pricing and cloud charges. **Fail:** missing billing access, unknown units or unexplained anomaly. Missing rows are not proof of zero spend.

## 28. Gate 11 — Backup and DR

Require recovery tier, business-approved RPO/RTO, independent recoverable data point, encryption-key access, Terraform/bundle recovery, Unity Catalog reconstruction, writer fencing, checkpoint strategy and consumer acceptance. Delta history alone is not an independent backup.

## 29. Lab K — Recovery-point discovery [SAFE-READ]

~~~sql
DESCRIBE HISTORY demo_catalog.demo_staging.synthetic_events;
DESCRIBE DETAIL demo_catalog.demo_staging.synthetic_events;
~~~

Record version, timestamp, storage location and actual recoverability constraints. Historical files may have been vacuumed or deleted. **Pass:** selected recovery point can be validated under approved DR procedures.

## 30. Lab L — RPO/RTO exercise [LOCAL]

~~~powershell
$failure = [datetimeoffset]"2026-10-14T12:00:00Z"
$recoverable = [datetimeoffset]"2026-10-14T11:55:00Z"
$consumerValidated = [datetimeoffset]"2026-10-14T12:41:00Z"
[pscustomobject]@{
  RPO_Minutes = ($failure - $recoverable).TotalMinutes
  RTO_Minutes = ($consumerValidated - $failure).TotalMinutes
}
~~~

Expected: RPO 5 minutes and RTO 41 minutes. Real drill values must be based on observed UTC evidence and accepted committed data.

## 31. Gate 12 — Incident and operational response

Validate severity model, on-call routing, incident commander, stakeholder updates, first-15-minute checklist, escalation contacts, evidence preservation, containment authority, rollback, data reconciliation and RCA process.

## 32. Lab M — Staging alert and incident tabletop [STAGING]

1. Select an isolated synthetic alert condition.
2. Record expected threshold, evaluation window and routing.
3. Trigger a benign synthetic signal with approval.
4. Verify alert reaches staging channel and links to runbook.
5. Assign incident commander, technical lead and scribe.
6. Record UTC timeline and expected business impact.
7. Resolve synthetic signal and verify recovery notification.
8. Archive redacted evidence and routing gaps.

Never send test pages to production on-call without approval.

## 33. Gate 13 — Failure recovery and idempotency

A resilient job should either recover within its service deadline or fail safely with actionable alerts. Test a synthetic failure and controlled replay in isolated staging, then verify exactly the intended output. Successful job state is insufficient if records are duplicated or lost.

## 34. Lab N — Controlled staging fault exercise [STAGING]

Use a preapproved synthetic job that can fail deterministically via a staging-only parameter. Capture baseline run and fixture; inject one bounded failure; observe retries, alerts and logs; restore baseline; replay if approved; validate row counts, uniqueness, schema and freshness. Abort immediately if the job reaches a production destination or exceeds resource limits.

## 35. Gate 14 — Change and rollback

Require protected GitLab branch, code review, static validation, reviewed Terraform plan, bundle target verification, deployment approval, canary/synthetic test, post-release SLO monitoring and documented rollback authority. Rolling back configuration does not undo Delta mutations or external side effects.

## 36. Release sequence

1. Verify staging identity and clean protected Git revision.
2. Run tests, Terraform validation and bundle validation.
3. Review intended changes and access impact.
4. Obtain change approval and backup/recovery readiness.
5. Deploy to staging using approved pipeline.
6. Run deterministic synthetic data and workload tests.
7. Validate governance, alerts, performance and cost.
8. Generate signed acceptance package.
9. Schedule production release with independent approval.
10. Monitor consumer SLOs and retain rollback readiness.

## 37. Scenario A — Wrong workspace target

**Detect:** host/account mismatch in preflight.  
**Respond:** stop pipeline, correct profile/target and review controls.  
**Accept:** verified staging destination before any mutation.

## 38. Scenario B — Terraform plan replaces critical resource

**Detect:** unexpected destroy/replace actions.  
**Respond:** NO-GO, inspect drift, state and provider changes.  
**Accept:** reviewed plan without unintended destructive actions.

## 39. Scenario C — Unity Catalog grant regression

**Detect:** authorized workload denied or unapproved broad grant.  
**Respond:** investigate effective privileges and audit trail.  
**Accept:** positive and negative access tests.

## 40. Scenario D — Job green but stale output

**Detect:** successful run with failed freshness SLI.  
**Respond:** inspect upstream source, checkpoint and data contract.  
**Accept:** correct accepted data within deadline.

## 41. Scenario E — Duplicate data after retry

**Detect:** unique event count differs from row count.  
**Respond:** fence writers and investigate idempotency.  
**Accept:** deterministic reconciliation and safe replay.

## 42. Scenario F — SQL latency and queue spike

**Detect:** workload-specific P95/P99 and queueing rise.  
**Respond:** inspect warehouse, concurrency, query plans and quotas.  
**Accept:** SLO and cost baseline recovered.

## 43. Scenario G — Audit collector delayed

**Detect:** source freshness watermark stale.  
**Respond:** mark compliance visibility degraded, use approved alternate evidence.  
**Accept:** telemetry restored and gap reconciled.

## 44. Scenario H — Backup exists but key unavailable

**Detect:** restore cannot decrypt protected data.  
**Respond:** follow authorized KMS recovery and escalation.  
**Accept:** isolated restore test and key access succeed.

## 45. Scenario I — Regional failover risks dual writers

**Detect:** primary job still running during recovery promotion.  
**Respond:** block promotion until fencing is verified.  
**Accept:** one authoritative writer and reconciled checkpoints.

## 46. Scenario J — Cost reduction breaks freshness

**Detect:** lower DBU usage with missed business deadline.  
**Respond:** reverse or tune the change using performance evidence.  
**Accept:** cost per accepted result improves without SLO regression.

## 47. Production operations runbook

~~~markdown
# Databricks Enterprise Operations
- Workspace / cloud / region / owners:
- Critical services and consumers:
- Identity and least-privilege status:
- Terraform / Bundle deployed revisions:
- Jobs and compute health:
- Data correctness and freshness:
- SQL latency and concurrency:
- Audit / alert source freshness:
- Cost and capacity baseline:
- Last backup and verified restore:
- DR RPO / RTO and writer-fencing plan:
- Current incidents and change freezes:
- Known risks and exceptions:
- On-call and escalation:
- Next review and approver:
~~~

## 48. Acceptance evidence package

~~~text
enterprise-acceptance-YYYYMMDD/
  01-charter-and-ownership.md
  02-target-and-cloud-identity.md
  03-terraform-and-bundle-validation.md
  04-network-and-storage.md
  05-unity-catalog-security.md
  06-data-contract-and-correctness.md
  07-jobs-and-compute.md
  08-sql-and-performance.md
  09-slo-and-alerting.md
  10-billing-capacity-and-finops.md
  11-backup-and-dr.md
  12-incident-and-failure-drill.md
  13-change-and-rollback.md
  14-open-risks-and-exceptions.md
  15-final-scorecard-and-signoff.md
~~~

Evidence must identify timestamp, source, reviewer, outcome and limitations. Store restricted artifacts in approved secured storage.

## 49. Enterprise scorecard

| Gate | Required acceptance |
|---|---|
| 1. Identity | Verified correct staging target |
| 2. Infrastructure | Terraform reproducibility and safe plan |
| 3. Bundles | Approved target, jobs and run-as |
| 4. Cloud | IAM, network, storage, KMS and quotas |
| 5. Governance | UC positive and negative access tests |
| 6. Data | Counts, schema, uniqueness and freshness |
| 7. Jobs | Success, retry, concurrency and ownership |
| 8. Performance | Workload-specific latency and capacity |
| 9. SLO | Objectives, source freshness and alert delivery |
| 10. FinOps | Usage attribution and approved cost baseline |
| 11. DR | Verified recovery point, RPO/RTO and fencing |
| 12. Incident | Roles, escalation and tabletop evidence |
| 13. Resilience | Bounded fault and correct recovery |
| 14. Change | Approval, promotion and rollback controls |

## 50. GO / CONDITIONAL GO / NO-GO

**GO:** Every critical gate has current, reproducible evidence; data correctness and access controls pass; SLO/alert routing and recovery procedures are validated; owners approve release.

**CONDITIONAL GO:** Only explicitly classified noncritical gaps remain, with named owner, compensating control, deadline and signed exception.

**NO-GO:** Wrong environment, unintended destructive plan, unauthorized access, corrupt output, missing critical monitoring, uncontrolled dual writers, unverified recovery or unacceptable SLO/cost risk.

## 51. Final project deliverables

Submit:
1. Architecture and dependency map.
2. Protected repository revision and validated deployment manifests.
3. Staging identity, governance and data acceptance results.
4. Job, SQL, compute, SLO and FinOps baselines.
5. Synthetic incident and resilience exercise evidence.
6. Backup, RPO/RTO and DR readiness evidence.
7. Completed scorecard, open risks and owner-approved decision.

The acceptance chair must be able to trace each critical decision to evidence without trusting unsupported assertions.

## 52. Review questions

1. What proves a deployment targeted the correct workspace?
2. Why is a clean Terraform validate insufficient for production approval?
3. How are Unity Catalog negative access tests performed safely?
4. Why does a successful job not prove data correctness?
5. What is the difference between an alert source being empty and healthy?
6. Why is Delta history not a substitute for independent backup?
7. How do you demonstrate a single writer during regional recovery?
8. How should cost optimization be judged against business SLOs?
9. What makes a CONDITIONAL GO exception acceptable?
10. What evidence must accompany the final enterprise acceptance decision?

## 53. Official references

- Databricks Terraform: https://docs.databricks.com/aws/en/dev-tools/terraform/
- Databricks Bundles: https://docs.databricks.com/aws/en/dev-tools/bundles/
- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks Jobs monitoring: https://docs.databricks.com/aws/en/jobs/monitor
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Databricks Delta history: https://docs.databricks.com/aws/en/delta/history
- Databricks disaster recovery: https://docs.databricks.com/aws/en/security/disaster-recovery
- Google SRE incident response: https://sre.google/workbook/incident-response/
- Google SRE SLO alerting: https://sre.google/workbook/alerting-on-slos/
- FinOps Foundation: https://www.finops.org/framework/

**Next:** Chapter 060 — Databricks SRE Final Enterprise Capstone and Operational Readiness Assessment.
