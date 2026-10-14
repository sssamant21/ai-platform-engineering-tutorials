# Chapter 060 — Databricks SRE Final Enterprise Capstone and Operational Readiness Assessment

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 13 — Capacity, Cost Governance and Enterprise Acceptance  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, DBRE, Platform Engineering, Data Engineering, Security, FinOps, Incident Commanders  
**Prerequisites:** Chapters 001–059  
**Status:** Final curriculum capstone — Chapter 060 of 060  
**Safety:** Use isolated staging, synthetic datasets and test-only alert routing for all active exercises. Production access is read-only unless an independently approved change authorizes a specific action. Never inject faults into production, modify real grants, delete checkpoints, expose secrets, force-unlock Terraform state or trigger actual customer-impacting failover for assessment purposes.

## 1. Capstone mission

Demonstrate end-to-end ownership of a production-grade Databricks platform by designing, deploying, operating, investigating, recovering and reviewing an isolated synthetic enterprise data service. The candidate must prove outcomes with reproducible evidence, not merely explain concepts.

The capstone consolidates platform architecture, multi-cloud dependencies, Terraform, GitLab CI/CD, Databricks Asset Bundles, Unity Catalog, Delta Lake, jobs, SQL warehouses, security, observability, SLOs, incident response, disaster recovery, resilience, capacity and FinOps.

## 2. Final learning outcomes

A successful candidate can:
- Explain the failure domains and dependencies of a Databricks service.
- Verify workspace, cloud and deployment identity before changes.
- Apply infrastructure and release safety controls.
- Implement least-privilege access and negative authorization tests.
- Validate data contracts, idempotency and consumer freshness.
- Investigate job, Spark, SQL, storage, network and IAM incidents.
- Operate actionable SLO monitoring and incident command.
- Demonstrate backup recoverability and controlled DR decision-making.
- Benchmark capacity, performance and cost per accepted output.
- Produce an evidence-based GO / CONDITIONAL GO / NO-GO decision.

## 3. Capstone scenario

An enterprise data product processes synthetic business events from a governed staging source into a Delta table and serves authorized SQL consumers. Its SLA requires successful scheduled refreshes, correct records, controlled access, bounded cost and documented disaster recovery.

The assessment includes an unexpected job failure, a simulated permission regression, a freshness degradation, a performance/cost anomaly and a regional recovery tabletop. All faults are staged or simulated; no live production disruption is required.

## 4. Assessment architecture

    Protected Git commit + GitLab pipeline
                   |
         Terraform + Databricks Bundles
                   |
    Cloud identity / network / storage / KMS
                   |
      Workspace + Unity Catalog governance
                   |
       Synthetic source -> Databricks Job
                   |
        Delta curated table -> SQL consumer
                   |
    SLOs + audit + billing + cloud monitoring
                   |
       Incident response + safe recovery
                   |
      Backup/DR + capacity/FinOps review
                   |
        Evidence -> scored acceptance

## 5. Assessment format

| Domain | Weight | Required evidence |
|---|---:|---|
| Architecture and cloud dependencies | 10 | Diagram, inventory and failure domains |
| IaC, release and rollback | 15 | Terraform and Bundle validation, protected change |
| Identity and Unity Catalog | 15 | Least-privilege positive/negative tests |
| Data pipelines and correctness | 15 | Fixture contract, idempotency and freshness |
| Observability and incident response | 15 | SLOs, alert test, incident timeline and RCA |
| Backup, DR and resilience | 15 | Recovery point, fencing, RPO/RTO and exercise |
| Performance, capacity and FinOps | 10 | Benchmark, attribution and forecast |
| Documentation and operational handoff | 5 | Runbooks, owners and evidence |
| **Total** | **100** | **No critical gate may fail** |

**Suggested scoring:** 90–100 = exemplary; 80–89 = pass if all critical gates pass; 70–79 = remediation and reassessment; below 70 = not ready. Scores never override critical security, data integrity or recovery failures.

## 6. Critical fail conditions

Immediate NO-GO for wrong environment, unauthorized access, unbounded blast radius, destructive unreviewed Terraform plan, lost/duplicated accepted records, uncontrolled dual writers, absent recoverable data, unavailable critical encryption keys, unverified rollback or active customer-impacting fault injection.

## 7. Preparation checklist

Confirm an isolated staging workspace, synthetic catalog and source, dedicated cloud identity, protected repository branch, Terraform backend, bundle targets, read-only audit/billing permissions, a test alert route, backup/recovery fixture, documented owners, approved budget and an abort operator.

## 8. Suggested assessment repository

~~~text
databricks-sre-capstone/
  README.md
  databricks.yml
  resources/
    jobs.yml
  src/
    synthetic_pipeline.py
  tests/
    test_contract.py
    test_idempotency.py
  infrastructure/
    environments/
      staging/
      dr-staging/
  sql/
    contract_checks.sql
    governance_checks.sql
  runbooks/
    incident.md
    rollback.md
    disaster-recovery.md
    failback.md
  assessment/
    charter.md
    scoring.md
    evidence-index.md
    final-decision.md
~~~

Do not commit credentials, raw Terraform state, confidential query text or unredacted audit data.

## 9. Lab A — Environment identity [SAFE-READ]

~~~powershell
git rev-parse HEAD
terraform version
databricks version
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
~~~

**Evidence:** actual workspace host, principal, region and protected revision. **Pass:** approved staging identity. **Abort:** any target mismatch.

## 10. Lab B — Multi-cloud identity [SAFE-READ]

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

Use the commands relevant to the chosen cloud. Document object storage, IAM, network, KMS, quota and regional dependencies.

## 11. Lab C — Terraform readiness [SAFE-READ]

~~~powershell
terraform -chdir=infrastructure/environments/staging fmt -check -recursive
terraform -chdir=infrastructure/environments/staging init -backend=false
terraform -chdir=infrastructure/environments/staging validate
~~~

**Evidence:** validated configuration and reviewed provider/module versions. A separately approved backend-connected plan is required for deployment; static validation alone is not a release authorization.

## 12. Lab D — Bundle readiness [SAFE-READ]

~~~powershell
databricks bundle validate --target staging
databricks bundle summary --target staging
~~~

**Evidence:** exact host, run-as principal, job dependencies, policies, schedules and release revision. **Abort:** unauthorized target or unexpected resources.

## 13. Lab E — Workload inventory [SAFE-READ]

~~~powershell
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks clusters list --output json
databricks cluster-policies list --output json
databricks warehouses list --output json
~~~

Use pagination and permission-aware inventory. For an authorized run:

~~~powershell
databricks jobs get-run REPLACE_WITH_RUN_ID --output json
~~~

Record failure state, retry policy, task dependencies, compute and timeline.

## 14. Lab F — Unity Catalog least privilege [SAFE-READ]

~~~sql
SELECT current_user();
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON SCHEMA demo_catalog.demo_staging;
~~~

Use approved separate test principals to demonstrate permitted and denied operations. Do not grant administrator privileges to make a failing assessment pass.

## 15. Lab G — Data contract [SAFE-READ]

~~~sql
SELECT
  COUNT(*) AS row_count,
  COUNT(DISTINCT event_id) AS unique_events,
  SUM(CASE WHEN event_id IS NULL THEN 1 ELSE 0 END) AS null_event_ids,
  MAX(event_time) AS newest_event_time
FROM demo_catalog.demo_staging.synthetic_events;
~~~

Compare against a deterministic fixture manifest including expected schema, counts, accepted event IDs and freshness deadline. **Pass:** all invariants hold; green job state alone is insufficient.

## 16. Lab H — Delta recoverability [SAFE-READ]

~~~sql
DESCRIBE HISTORY demo_catalog.demo_staging.synthetic_events;
DESCRIBE DETAIL demo_catalog.demo_staging.synthetic_events;
~~~

Identify last committed version, storage location, retention assumptions and independently protected recovery point. Delta time travel is not an independent backup.

## 17. Lab I — Audit evidence [SAFE-READ]

~~~sql
SELECT
  MAX(event_time) AS newest_audit_event,
  COUNT(*) AS observed_events
FROM system.access.audit
WHERE event_date >= date_sub(current_date(), 1);
~~~

Validate source freshness and permission coverage. Audit events may be delayed; zero records are not proof of healthy monitoring.

## 18. Lab J — SQL observability [SAFE-READ]

~~~sql
SHOW TABLES IN system.query;
DESCRIBE TABLE system.query.history;
SELECT * FROM system.query.history LIMIT 10;
~~~

Build workload-specific latency, queueing and failure measurements using the account's actual schema and authorized access. Redact sensitive query text.

## 19. Lab K — Billing and cost attribution [SAFE-READ]

~~~sql
SELECT
  usage_date,
  sku_name,
  usage_unit,
  SUM(usage_quantity) AS total_usage
FROM system.billing.usage
WHERE usage_date >= date_sub(current_date(), 7)
GROUP BY usage_date, sku_name, usage_unit
ORDER BY usage_date DESC, sku_name;
~~~

Do not combine unlike usage units or treat list prices as contracted effective costs. Reconcile cloud infrastructure charges separately.

## 20. Lab L — RPO/RTO measurement [LOCAL]

~~~powershell
$incident = [datetimeoffset]"2026-10-14T12:00:00Z"
$recoverable = [datetimeoffset]"2026-10-14T11:54:00Z"
$consumerAccepted = [datetimeoffset]"2026-10-14T12:43:00Z"
[pscustomobject]@{
  ActualRPO_Minutes = ($incident - $recoverable).TotalMinutes
  ActualRTO_Minutes = ($consumerAccepted - $incident).TotalMinutes
}
~~~

Expected: 6-minute RPO and 43-minute RTO. Replace example timestamps with measured UTC evidence in an actual drill.

## 21. Lab M — Synthetic incident exercise [STAGING]

1. Record baseline synthetic job run, expected output and alert route.
2. Inject one bounded failure through a staging-only test parameter.
3. Verify job failure, retries, alert and incident timeline.
4. Assign incident commander, technical lead and scribe.
5. Collect redacted evidence from jobs, Spark logs and audit.
6. Restore the staging test configuration through approved procedure.
7. Re-run safely and verify count, uniqueness and freshness.
8. Draft a blameless RCA with causal evidence and corrective actions.

**Abort:** wrong target, unexpected external side effect, unbounded retries or missing recovery control.

## 22. Lab N — Idempotent replay [STAGING]

Submit the same deterministic synthetic event batch twice through an approved staging replay. Verify output uniqueness and exactly the intended accepted records. Check external effects separately; a Delta MERGE does not automatically make notifications or other sinks idempotent.

## 23. Lab O — DR failover tabletop [NO MUTATION]

1. Declare a simulated primary-region outage.
2. Identify independent recovery workspace, identity, storage, network and KMS.
3. Select a verified consistent recovery point.
4. Explain how primary writers are fenced and in-flight work is reconciled.
5. Validate Unity Catalog metadata and grants.
6. Define job and checkpoint restart order.
7. Define consumer acceptance and routing.
8. Measure projected versus evidence-based RPO/RTO.
9. Record NO-GO triggers and failback prerequisites.
10. Obtain DR owner sign-off.

## 24. Lab P — Performance and cost comparison [STAGING]

Choose two approved compute/query configurations and a representative synthetic dataset. Execute comparable repetitions, record queue/startup, duration, P95/P99 where applicable, total billable consumption, data correctness and cost per successful result. Reject an apparent cost saving if it violates freshness or correctness.

## 25. Practical assessment 1 — Job failure

**Inject:** deterministic synthetic task exception.  
**Candidate must:** find run/task failure, establish UTC onset, distinguish trigger from root cause, evaluate retry safety, recover and validate consumer output.  
**Pass:** evidence-based diagnosis and correct recovery.

## 26. Practical assessment 2 — Permission regression

**Simulate:** pre-created staging principal lacks required narrow privilege.  
**Candidate must:** inspect effective grants, cloud storage identity and audit, propose least-privilege correction and negative test.  
**Pass:** no broad privilege escalation.

## 27. Practical assessment 3 — SQL latency regression

**Simulate:** approved staging query with unfavorable join or scan.  
**Candidate must:** separate queueing from execution, inspect plan, identify skew/scan, benchmark correction and assess cost.  
**Pass:** reproducible improvement without output change.

## 28. Practical assessment 4 — Data freshness failure

**Simulate:** pause only the isolated synthetic source.  
**Candidate must:** correlate source watermark, job completion and consumer deadline; recover backlog safely.  
**Pass:** no missing or duplicate accepted events.

## 29. Practical assessment 5 — Cluster provisioning failure

**Tabletop:** regional quota or cloud capacity unavailable.  
**Candidate must:** distinguish policy, identity, quota and capacity issues; select approved fallback and cost controls.  
**Pass:** realistic mitigation and acceptance criteria.

## 30. Practical assessment 6 — Storage access denied

**Tabletop:** storage credential or KMS access regression.  
**Candidate must:** trace UC, cloud IAM, network and encryption dependency without exposing secrets.  
**Pass:** least-privilege resolution and verification plan.

## 31. Practical assessment 7 — Duplicate replay

**Simulate:** retry of the same synthetic input batch.  
**Candidate must:** prove idempotency, identify external side effects and reconcile checkpoint ownership.  
**Pass:** deterministic accepted output.

## 32. Practical assessment 8 — Monitoring blind spot

**Tabletop:** audit collector or query history ingestion delayed.  
**Candidate must:** distinguish missing telemetry from healthy zero events, use alternate signals and document coverage gap.  
**Pass:** alert and compliance visibility restored.

## 33. Practical assessment 9 — Cost anomaly

**Tabletop:** DBU use spikes after a release.  
**Candidate must:** attribute SKU, workspace, workload, retries, cloud charges and changes.  
**Pass:** measured root cause and SLO-safe correction.

## 34. Practical assessment 10 — Regional DR failover

**Tabletop:** primary region unavailable while writer status uncertain.  
**Candidate must:** refuse unsafe promotion until fencing proven, choose consistent recovery point and define consumer acceptance.  
**Pass:** no split brain and measurable RPO/RTO.

## 35. Incident command exercise

Roles: incident commander, technical lead, communications lead, data owner and scribe. Produce an initial statement within the organization's defined response target, a UTC timeline, hypothesis table, mitigation authorization, consumer verification and a post-incident RCA. Do not declare resolution merely because the job status turns green.

## 36. Root cause analysis template

~~~markdown
# Final Capstone Incident RCA
- Incident ID / severity:
- UTC onset / detection / mitigation / consumer acceptance:
- Affected services, data and consumers:
- Detection source and alert latency:
- Verified facts and evidence:
- Trigger:
- Causal mechanism:
- Contributing conditions:
- Alternative hypotheses ruled out:
- Containment and authorized recovery:
- Data integrity / freshness / security validation:
- Corrective actions (owner, due date, verification):
- Residual risks and closure approver:
~~~

## 37. Security and compliance assessment

Review identity federation/service principals, least-privilege grants, network boundaries, storage credentials, KMS, secret references, audit retention, incident evidence restrictions and change approvals. Verify no hard-coded tokens, public bucket access, unexpected privilege inheritance or broad emergency grants remain.

## 38. Reliability and SLO assessment

For each service define eligible event population, success condition, time window, objective, error budget, burn-rate policy, source freshness and owner. Include data correctness and consumer deadline as first-class signals. Document alert noise, missing data and recovery notifications.

## 39. Backup and disaster recovery assessment

Verify independent backup recoverability, source retention, Delta log/data consistency, key access, cross-region storage availability, infrastructure reconstruction, UC permissions, single-writer fencing, checkpoint/replay, consumer validation and failback. Record actual or explicitly tabletop-only RPO/RTO; do not present projections as measured outcomes.

## 40. Capacity and FinOps assessment

Review workload growth, queueing, peak concurrency, worker/warehouse sizing, cloud quotas, storage layout, DBU usage, cloud infrastructure and egress charges, tagging coverage, budgets and cost per accepted result. Recommend changes with expected SLO impact and a rollback.

## 41. Final operations runbook

~~~markdown
# Databricks Production Operations — Final Assessment
- Platform / workspace / cloud / region:
- Business service and owners:
- Approved Git, Terraform and Bundle revisions:
- Current jobs / compute / SQL health:
- Data contract and freshness:
- Unity Catalog and cloud identity:
- Monitoring source freshness and SLO:
- Cost and capacity baseline:
- Backup, DR and last tested recovery point:
- Incident severity and escalation:
- Change freeze and rollback authority:
- Known risks / exceptions:
- Decision and approver:
~~~

## 42. Final evidence package

~~~text
databricks-sre-final-capstone/
  01-charter-and-scope.md
  02-architecture-and-dependencies.md
  03-target-and-identity-proof.md
  04-terraform-and-bundle-validation.md
  05-security-and-unity-catalog.md
  06-data-contract-and-idempotency.md
  07-jobs-compute-and-sql.md
  08-slo-alerting-and-audit.md
  09-incident-timeline-and-rca.md
  10-backup-dr-and-resilience.md
  11-capacity-performance-and-finops.md
  12-runbooks-and-handoff.md
  13-scorecard-and-remediation.md
  14-final-signoff.md
~~~

Each item must identify evidence source, UTC timestamp, assessor, result and limitations. Store sensitive artifacts in access-controlled systems, not public Git.

## 43. Final scorecard

| Critical gate | Pass criteria |
|---|---|
| Target identity | Workspace and cloud principal verified |
| Release safety | Protected revision, reviewed plan and rollback |
| Security | Least privilege and negative tests |
| Data | Schema, counts, uniqueness, freshness |
| Jobs | Bounded retries and correct output |
| SQL | Workload-specific SLO and baseline |
| Observability | Fresh telemetry and actionable alert |
| Incident response | Roles, evidence, recovery and RCA |
| Backup | Independently recoverable data point |
| DR | Writer fencing, recovery order, RPO/RTO |
| Capacity | Peak demand, quotas and headroom |
| FinOps | Usage, cloud charges and unit economics |
| Documentation | Runbooks, owners and evidence |
| Sign-off | Explicit accountable approval |

## 44. Scoring and certification decision

**PASS:** At least 80/100 and every critical gate passed with verified evidence.

**CONDITIONAL PASS / REMEDIATION:** Score meets threshold but a noncritical deliverable needs time-bound correction; do not label the platform production-ready until the exception is approved.

**FAIL / NO-GO:** Any critical security, data integrity, recovery, environment isolation or change-control gate fails, regardless of numeric score.

This is a curriculum assessment rubric, not an externally accredited professional certification.

## 45. Candidate self-assessment

Answer with evidence:
1. Can I identify the precise workspace, principal and cloud account?
2. Can I reproduce a deployment from a protected revision?
3. Can I explain which Terraform actions require approval?
4. Can I prove positive and negative UC access controls?
5. Can I demonstrate idempotent replay and data correctness?
6. Can I distinguish job failures from infrastructure failures?
7. Can I trace a SQL latency spike to queueing, scan or execution?
8. Can I calculate SLO and error-budget impact?
9. Can I produce a blameless RCA with causal evidence?
10. Can I prove a backup is actually recoverable?
11. Can I explain single-writer fencing and safe failback?
12. Can I connect performance optimization to total cost per accepted output?

## 46. Acceptance review meeting agenda

1. Scope, owners and known limitations.
2. Architecture and dependency map.
3. Identity, governance and release controls.
4. Data, jobs, SQL and SLO evidence.
5. Incident exercise and RCA.
6. Backup, DR and resilience evidence.
7. Capacity and FinOps recommendations.
8. Critical failures, exceptions and remediation.
9. Final GO / CONDITIONAL GO / NO-GO vote.
10. Handoff, follow-up ownership and review date.

## 47. Final GO / CONDITIONAL GO / NO-GO

**GO:** Critical gates pass, evidence is current and reproducible, data and security are verified, rollback and recovery are credible, and accountable owners approve.

**CONDITIONAL GO:** Only documented noncritical gaps remain with owner, deadline, expiry and compensating control, and the organization explicitly accepts the residual risk.

**NO-GO:** Wrong target, unauthorized access, missing recoverability, uncontrolled writers, incorrect data, missing critical alerting or unapproved destructive change.

## 48. Curriculum completion audit

Before declaring the repository fully complete, separately verify that all expected chapter files 001–060 exist, are readable, have unique numbering, consistent naming, working relative links, official references and no secret-bearing content. Validate labs against current CLI/API versions and actual cloud deployments. **This chapter's publication alone does not establish that every earlier chapter passed a repository-wide audit.**

## 49. Official references

- Databricks CLI: https://docs.databricks.com/aws/en/dev-tools/cli/
- Databricks Terraform: https://docs.databricks.com/aws/en/dev-tools/terraform/
- Databricks Bundles: https://docs.databricks.com/aws/en/dev-tools/bundles/
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Databricks Jobs monitoring: https://docs.databricks.com/aws/en/jobs/monitor
- Databricks system tables: https://docs.databricks.com/aws/en/admin/system-tables/
- Databricks Delta history: https://docs.databricks.com/aws/en/delta/history
- Databricks disaster recovery: https://docs.databricks.com/aws/en/security/disaster-recovery
- Google SRE incident response: https://sre.google/workbook/incident-response/
- Google SRE alerting on SLOs: https://sre.google/workbook/alerting-on-slos/
- FinOps Foundation: https://www.finops.org/framework/

## 50. Graduation and next steps

**Chapter 060 completes the planned 60-chapter Databricks SRE & Infrastructure Engineering learning path.**

Recommended post-capstone activities:
- Run the final exercise in an isolated staging environment and retain signed evidence.
- Perform a separate repository-wide 001–060 completeness, link and consistency audit.
- Review every operational runbook with platform, security and data owners.
- Rehearse recovery periodically and update the curriculum for Databricks/cloud changes.
- Track remediation actions through closure and re-test.

**End of curriculum — Databricks SRE & Infrastructure Engineering, Chapters 001–060.**
