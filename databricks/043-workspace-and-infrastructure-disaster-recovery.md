# Chapter 043 — Workspace and Infrastructure Disaster Recovery

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 09 — Reliability and Recovery  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, Cloud Platform, Security, Networking and Disaster Recovery Engineering  
**Prerequisites:** Chapters 006–010, 016–020, 026–030, 041–042  
**Safety:** All rebuilds and failover tests require approved staging environments, isolated state, data protection, cost controls and documented rollback. Production commands in this chapter are read-only unless explicitly approved through change management.

## 1. Mission

Rebuild a Databricks workspace and its required infrastructure from version-controlled configuration after a major outage or configuration loss. Recover the complete service dependency chain rather than only recreating the workspace resource.

### Learning outcomes

- Design an infrastructure-as-code recovery architecture.
- Inventory control-plane, compute, network, identity, storage and governance dependencies.
- Distinguish workspace recreation from service recovery.
- Validate Terraform plans and state safely.
- Restore jobs, compute policies, permissions and data access.
- Plan regional recovery across AWS, Azure and GCP.
- Measure recovery progress and verify consumer-visible correctness.
- Perform rollback and controlled failback without duplicate writes.

## 2. Recovery architecture

    Version-controlled IaC and code
                |
        Recovery orchestrator
                |
      Cloud account/subscription
                |
         Network and identity
                |
       Databricks workspace
                |
     Policies / service principals
                |
     Unity Catalog / storage access
                |
         Jobs / compute / SQL
                |
     Recovered consistent datasets
                |
       Validated consumers + SLO

IaC recreates declared resources; it does not automatically restore source data, secret values, workspace-managed metadata, job run history or external integrations.

## 3. Disaster recovery scope

| Component | Rebuild mechanism | Required validation |
|---|---|---|
| Cloud account/project/subscription | Existing approved landing zone | Quota and access |
| VPC/VNet and subnets | Cloud IaC | Routing, isolation |
| NAT, private endpoints and DNS | Cloud IaC | Connectivity from compute |
| Databricks workspace | Supported provider resources | API and login |
| Identity and service principals | IAM/IdP automation | Authentication and least privilege |
| Compute policies and pools | Databricks IaC | Policy enforcement |
| Jobs and schedules | Databricks IaC / bundles | Correct task graph |
| Unity Catalog | Supported recovery approach | Grants, storage access |
| Data files and Delta logs | Consistent backup/replication | Transaction correctness |
| Secret references and KMS | Approved secret/key recovery | Decryption and rotation |
| Monitoring and alerts | IaC/observability config | Synthetic alert delivery |
| Consumers and endpoints | Application runbooks | Correctness and freshness |

## 4. Failure domains

Assess:
- Single compute node and availability zone.
- Subnet, DNS, firewall and private endpoint.
- Workspace configuration and identity.
- Cloud region and quota/capacity.
- Object storage and metadata.
- Cloud account/project/subscription.
- Shared identity, encryption and external dependencies.

A second workspace in the same region may not protect against a regional failure. A second region does not guarantee independent identity or governance dependencies.

## 5. Recovery patterns

| Pattern | Strength | Tradeoff |
|---|---|---|
| Rebuild in place | Simpler resource mapping | Depends on affected region |
| Warm standby workspace | Lower activation time | Ongoing cost and drift |
| Cold standby | Lower steady-state cost | Longer RTO |
| Active/passive regional DR | Controlled writer promotion | Replication lag |
| Active/active | Lower traffic interruption potential | Split-brain and data consistency complexity |

For stateful pipelines, define a single authoritative writer or explicitly tested conflict resolution.

## 6. Terraform source-of-truth design

Suggested repository layout:

    infrastructure/
      environments/
        staging/
        production/
        dr/
      modules/
        cloud-network/
        cloud-identity/
        databricks-workspace/
        databricks-governance/
        databricks-jobs/
        monitoring/
      policies/
      runbooks/
        disaster-recovery.md

Keep remote state encrypted and access-controlled. Separate production and DR state; do not copy state files manually into a new backend.

## 7. Terraform safety model

- Pin provider versions and commit lockfiles.
- Use remote state with locking where supported.
- Back up and protect state through approved backend controls.
- Use separate workspace/state and cloud identities for staging and DR.
- Review plans for unintended destroys or replacements.
- Avoid storing secrets in Terraform variables or state where possible.
- Never use a targeted apply as a substitute for a reviewed full plan.
- Require approval for any production apply.
- Verify resource ownership and import strategy before reconstructing existing resources.

## 8. Lab A — Windows PowerShell tooling preflight

~~~powershell
terraform version
databricks version
databricks auth profiles
aws --version
az version
gcloud version
~~~

Only the CLI for the selected cloud is required. Confirm the working directory and credentials before any Terraform command.

## 9. Lab B — Databricks staging inventory

~~~powershell
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks clusters list --output json
databricks cluster-policies list --output json
~~~

Some CLI subcommands and permissions depend on installed version. Check the relevant command help before execution.

## 10. Lab C — Terraform formatting and validation

Run from an existing authorized IaC directory:

~~~powershell
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
~~~

Backend-disabled initialization supports local validation; it does not prove that the remote state, provider credentials or resources are recoverable.

## 11. Lab D — Safe DR plan

In a dedicated, preconfigured staging DR working directory:

~~~powershell
terraform init
terraform workspace show
terraform plan -out=dr-review.tfplan
terraform show -no-color dr-review.tfplan
~~~

**Guardrails:** verify backend, workspace, provider account/region and expected resources first. Plan files and state can contain sensitive values; keep them encrypted and access-controlled. Never commit them.

Do not run terraform apply as part of the read-only assessment.

## 12. Lab E — Terraform drift review

~~~powershell
terraform plan -refresh-only -out=dr-refresh.tfplan
terraform show -no-color dr-refresh.tfplan
~~~

A refresh-only plan identifies observed state changes; it does not repair drift. Review proposed state changes and avoid applying without approval.

## 13. AWS recovery prerequisites

Verify:
- AWS account and recovery region.
- VPC/subnet routes and required endpoints.
- EC2 instance families and quotas.
- IAM roles and trust policies.
- S3 data replication and KMS keys.
- Workspace-specific cross-account configurations where applicable.
- CloudWatch, CloudTrail and audit continuity.

Regional capacity is not guaranteed simply because quota exists.

## 14. Lab F — AWS read-only recovery checks

~~~powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-vpcs --max-results 20
aws ec2 describe-vpc-endpoints --max-results 20
aws service-quotas list-service-quotas --service-code ec2 --max-results 20
~~~

Use filters to limit output to authorized resources.

## 15. Azure recovery prerequisites

Verify:
- Subscription, recovery region and quotas.
- VNet/subnet delegation and route design.
- NSGs, private endpoints and private DNS zones.
- Managed identities, service principals and role assignments.
- ADLS Gen2 redundancy and storage access.
- Key Vault keys and secret references.
- Workspace deployment requirements and monitoring.

Managed resources and networking patterns differ by workspace architecture.

## 16. Lab G — Azure read-only recovery checks

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az vm list-usage --location eastus -o table
az network vnet list --resource-group demo-dr-rg -o table
az network private-endpoint list --resource-group demo-dr-rg -o table
~~~

Replace placeholder region/resource group with the approved DR location.

## 17. GCP recovery prerequisites

Verify:
- Project, recovery region and quotas.
- VPC, subnets, firewall, Cloud NAT and private access.
- Service accounts and workload identities.
- Cloud Storage protection and encryption keys.
- Databricks workspace requirements for GCP.
- Monitoring, audit logs and recovery automation.

Check product-specific regional support before selecting a target region.

## 18. Lab H — GCP read-only recovery checks

~~~powershell
gcloud config get-value project
gcloud compute regions describe us-central1 --format=json
gcloud compute networks list
gcloud compute routes list
gcloud iam service-accounts list
~~~

Scope the project and region to authorized recovery infrastructure.

## 19. Unity Catalog and governance recovery

Inventory:
- Metastore assignments and regional scope.
- Catalog/schema/table/volume definitions.
- Managed versus external storage.
- Storage credentials and external locations.
- Workspace bindings and privileges.
- Ownership, grants and service principal mapping.
- Recovery of audit and lineage evidence.

Unity Catalog metadata recovery is not equivalent to copying Delta files. Follow supported Databricks procedures; do not manipulate managed metadata directly.

## 20. Lab I — SQL governance discovery

~~~sql
SHOW CATALOGS;
SHOW SCHEMAS IN demo_catalog;
SHOW GRANTS ON CATALOG demo_catalog;
~~~

Use approved staging objects. Record unavailable permissions and missing objects without granting broad privileges.

## 21. Storage and data recovery sequencing

1. Select a verified consistent recovery point.
2. Verify cloud object access and decryption keys.
3. Restore or expose the approved recovered dataset.
4. Confirm Delta log and data-file consistency.
5. Reconstruct supported catalog mappings.
6. Validate schema, row counts, uniqueness and business invariants.
7. Reconcile incremental source data.
8. Resume a single authoritative writer.
9. Validate consumer freshness and permissions.

A replicated copy may contain the same corruption as the source. Verify recovery-point integrity.

## 22. Lab J — Delta recovery inventory

~~~sql
DESCRIBE DETAIL demo_catalog.demo_staging.events;
DESCRIBE HISTORY demo_catalog.demo_staging.events;
~~~

These are read-only. They do not establish an independent backup exists.

## 23. Workflow and scheduling recovery

Restore:
- Job definitions and task dependencies.
- Schedules and timezone settings.
- Concurrency and retry policies.
- Cluster/job compute policies.
- Service principal ownership and grants.
- Libraries, init scripts and artifact references.
- Notifications and alert routing.
- External event triggers.

**Critical:** Keep recovery-region schedules disabled until source writers are fenced and data consistency is verified.

## 24. Lab K — Job configuration inventory

~~~powershell
databricks jobs list --output json
databricks jobs get --help
databricks jobs list-runs --limit 25 --output json
~~~

Capture only sanitized job configuration. A list response may be paginated and does not guarantee all tasks or settings were exported.

## 25. Service dependency graph

| Dependency | Required before | Owner | Evidence |
|---|---|---|---|
| Cloud account and quota | Workspace/compute | Cloud platform | Provider inventory |
| DNS/network | Storage and API access | Networking | Compute-side probe |
| IAM/KMS | Storage decryption | Security | Authorized access |
| Storage recovery | Delta/catalog access | Storage | Consistent snapshot |
| Unity Catalog | Job data access | Data governance | Grants and query |
| Compute policies | Jobs start | Platform | Policy test |
| Job definitions | Workflow execution | Workflow owner | Test run |
| Monitoring | Production activation | SRE | Alert delivery |
| Consumer validation | Incident closure | Service owner | Freshness and correctness |

## 26. Recovery activation runbook

1. Declare incident and business impact.
2. Confirm RTO/RPO targets and last consistent recovery point.
3. Assign incident commander, recovery lead and approver.
4. Fence source writers and unsafe retries.
5. Confirm recovery-region identity, keys, quota and networking.
6. Review Terraform plan and resource drift.
7. Provision approved infrastructure.
8. Restore governance, storage access, jobs and monitoring.
9. Validate data correctness and recovery lag.
10. Enable schedules/consumers in controlled order.
11. Verify SLO, audit and security boundaries.
12. Record actual recovery time, data gap and cost.

## 27. Failback runbook

- Determine authoritative data region and commit position.
- Keep one active writer.
- Reconcile changes made during DR operation.
- Validate destination storage and metadata.
- Restore source-region networking and identity.
- Stage test workloads without activating conflicting schedules.
- Approve traffic/schedule cutback.
- Monitor correctness, freshness and error rates.
- Preserve rollback path until stable.

Do not assume failback is simply the reverse of failover.

## 28. Scenario A — Terraform state unavailable

**Evidence:** backend errors, state backup, locking.

**Recovery:** approved state backend recovery/import procedure.

**Acceptance:** drift-free plan without unexpected destruction.

## 29. Scenario B — Regional VM quota exhausted

**Evidence:** quota, instance family, cluster provisioning events.

**Recovery:** approved quota or alternate capacity.

**Acceptance:** tested compute startup within RTO.

## 30. Scenario C — Private DNS misconfigured

**Evidence:** resolver output, private zone links, routes.

**Recovery:** narrowly scoped approved DNS correction.

**Acceptance:** compute-side connectivity and TLS validation.

## 31. Scenario D — KMS key unavailable

**Evidence:** key state, audit, storage decrypt errors.

**Recovery:** supported key/identity recovery.

**Acceptance:** authorized decryption and least privilege.

## 32. Scenario E — Missing Unity Catalog grants

**Evidence:** 403, catalog permissions and principal mapping.

**Recovery:** restore approved grants and bindings.

**Acceptance:** expected access succeeds, unauthorized access fails.

## 33. Scenario F — Job definitions restored but schedules duplicate

**Evidence:** two active regions, overlapping run IDs and writes.

**Recovery:** disable one writer, reconcile data, re-enable controlled schedules.

**Acceptance:** no duplicate or missing records.

## 34. Scenario G — Replicated data is inconsistent

**Evidence:** Delta transaction state, missing objects, replication lag.

**Recovery:** known-good consistent recovery point and replay.

**Acceptance:** transaction-consistent reads and business invariants.

## 35. Scenario H — Workspace rebuilt but consumers remain down

**Evidence:** downstream endpoints, freshness, permissions and alerts.

**Recovery:** resolve final-mile dependencies and consumer configuration.

**Acceptance:** consumer-visible SLO restored, not merely workspace API healthy.

## 36. Recovery validation worksheet

| Check | Expected result | Evidence |
|---|---|---|
| Correct cloud/region | Approved DR location | Account/region output |
| Terraform plan | No unexpected destroy | Reviewed plan |
| Workspace API | Authorized access | CLI/API probe |
| Compute startup | Within target | Cluster events |
| Network/DNS/TLS | Correct private path | Compute-side test |
| Identity/KMS | Authorized decryption | Audit and probe |
| Unity Catalog | Correct grants | SQL checks |
| Delta data | Consistent recovery point | History/invariants |
| Jobs | Controlled test succeeds | Run result |
| Monitoring | Alerts received | Synthetic test |
| Consumers | Correct and fresh output | Consumer acceptance |
| RTO/RPO | Within targets | UTC timeline |

## 37. Monitoring and DR readiness

Track:
- IaC drift and last successful plan validation.
- Recovery workspace health and quota.
- Storage replication lag and recoverability.
- Key/secret lifecycle and permission drift.
- Backup and restore-test age.
- Job definition parity and schedule state.
- Data freshness/correctness in standby.
- Network/DNS/private endpoint changes.
- Measured RTO/RPO from drills.
- DR operating cost.

## 38. GO / CONDITIONAL GO / NO-GO

**GO:** Recovery environment, identity, data, governance, jobs and consumers validated; RTO/RPO measured; one authoritative writer; rollback ready.

**CONDITIONAL GO:** Only documented noncritical gaps remain with owner, mitigation and deadline.

**NO-GO:** Unknown recovery point, missing keys, unsafe dual writers, unreviewed destructive Terraform plan, unverified Delta consistency or failed consumer correctness.

## 39. Acceptance checklist

- [ ] Recovery topology and failure domains documented.
- [ ] IaC source, backend and state protection verified.
- [ ] Cloud quotas and regional capacity reviewed.
- [ ] Network, DNS and private endpoints mapped.
- [ ] IAM, service principals and KMS recovery verified.
- [ ] Unity Catalog dependencies inventoried.
- [ ] Consistent storage recovery point identified.
- [ ] Jobs and schedules reconstructed safely.
- [ ] Monitoring and alert routing tested.
- [ ] Eight recovery scenarios reviewed.
- [ ] Consumer correctness and freshness validated.
- [ ] RTO/RPO and DR cost measured.
- [ ] Failback and rollback documented.

## 40. Review questions

1. Why is a successful Terraform apply insufficient to declare disaster recovery complete?
2. What prevents two regional jobs from writing the same Delta dataset?
3. Why must Unity Catalog and cloud IAM be recovered together?
4. How does Terraform state loss differ from resource loss?
5. Which consumer-visible checks are required before closing a regional outage?

## 41. Official references

- Databricks disaster recovery: https://docs.databricks.com/aws/en/security/disaster-recovery
- Databricks architecture: https://docs.databricks.com/aws/en/getting-started/high-level-architecture
- Databricks Terraform provider: https://registry.terraform.io/providers/databricks/databricks/latest/docs
- Databricks Unity Catalog: https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- Terraform state: https://developer.hashicorp.com/terraform/language/state
- Terraform plan: https://developer.hashicorp.com/terraform/cli/commands/plan
- AWS Well-Architected Reliability: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html
- Azure Well-Architected Reliability: https://learn.microsoft.com/azure/well-architected/reliability/
- Google Cloud reliability: https://cloud.google.com/architecture/framework/reliability

**Next:** Chapter 044 — RPO, RTO and Recovery Drills.
