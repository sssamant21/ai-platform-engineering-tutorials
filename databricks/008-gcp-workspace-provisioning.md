# Chapter 008 — GCP Workspace Provisioning

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 02 — Cloud Provisioning and Networking  
**Audience:** SRE, Google Cloud infrastructure and platform operations  
**Lab:** Read-only preflight, Terraform design and optional approved staging deployment  
**Risk:** Deployment can create billable resources

## 1. Learning objectives

After completing this chapter, you can:
1. Explain Databricks on Google Cloud account/workspace and cloud project boundaries.
2. Identify required Google Cloud APIs, IAM roles, service accounts and regional quotas.
3. Understand workspace networking, Private Service Connect and Cloud NAT dependencies where applicable.
4. Distinguish classic compute dependencies from serverless managed compute.
5. Prepare a secure, reviewed Terraform deployment workflow.
6. Diagnose failed provisioning, compute startup, network and storage access.
7. Validate deployment, document operational ownership and plan teardown.

## 2. Conceptual architecture

~~~text
       SRE / Terraform / Google Cloud CLI
                      |
           Google Cloud project + IAM
                      |
              Databricks account
                      |
          Workspace provisioning API
                      |
          Databricks workspace
                      |
         +------------+------------+
         |                         |
    Classic compute          Serverless compute
  (supported GCP model)      (managed execution)
         |
   VPC / subnets / routes / firewall
         |
   Cloud Storage / external services
         |
 Cloud Logging / Monitoring / billing
~~~

The exact compute placement, networking model and workspace-creation flow depend on the current Databricks on Google Cloud product capabilities. Do not assume the AWS or Azure deployment model applies unchanged to GCP.

## 3. Responsibility boundaries

| Layer | Operational owner / concern |
|---|---|
| Databricks account and workspace | Account admins, workspace admins, Databricks managed service |
| Google Cloud project | Cloud organization, IAM, quotas, billing and resource governance |
| Google IAM and service accounts | Least privilege, identity lifecycle, workload access |
| VPC and connectivity | Cloud networking, DNS, routing, firewall, NAT/PSC as designed |
| Cloud Storage | Bucket location, encryption, IAM, access and lifecycle |
| Classic compute | Provisioning, regional capacity, policy and startup health |
| Serverless compute | Managed execution, supported connectivity, limits and cost |
| Audit and observability | Databricks telemetry plus relevant Cloud Logging/Monitoring |
| Disaster recovery | Customer-owned recovery objectives, data and configuration continuity |

### Important architectural distinctions

- A Google Cloud project is not the same object as a Databricks workspace.
- Google Cloud CLI's active project/region is not proof of the workspace's deployment project/region.
- Serverless compute may have different networking and service boundaries than classic compute.
- Creating a workspace is not the same as configuring Unity Catalog, production data access or DR.
- An account/workspace permission failure may occur even when Google Cloud IAM appears valid.

## 4. Preflight requirements

- An approved non-production Google Cloud project with billing enabled.
- Databricks account provisioning privileges appropriate to the selected GCP deployment method.
- Required Google Cloud APIs and service accounts approved by the cloud administrator.
- Supported region confirmed in current Databricks GCP documentation.
- Approved VPC, subnet, DNS, firewall and egress architecture.
- Cloud Storage access and residency design.
- Terraform, gcloud and Databricks CLI installed.
- Secure remote Terraform state, cost budget and teardown owner.
- A change record for any resource creation or IAM modification.

Never place service account keys, OAuth credentials, Terraform state, saved plans or internal inventory outputs in GitHub.

## 5. Hands-on lab A — Read-only Google Cloud preflight

The following commands are examples for PowerShell on Windows.

### Step 1 — Validate tools and active identity

~~~powershell
gcloud version
terraform version
databricks version

gcloud auth list
gcloud config get-value project
gcloud config get-value compute/region
gcloud config list
~~~

Use an organization-approved authenticated identity. Do not infer that a locally configured project is the correct deployment target.

### Step 2 — Set the approved staging project

~~~powershell
$projectId = "<approved-staging-project-id>"
$region = "us-central1"  # Example only: verify Databricks support
gcloud config set project $projectId
gcloud projects describe $projectId --format="json(projectId,projectNumber,lifecycleState)"
~~~

The region must be checked against current Databricks GCP workspace availability, not just Google Cloud region availability.

### Step 3 — Inspect API enablement

~~~powershell
gcloud services list --enabled --project=$projectId
~~~

Record the APIs required by the selected deployment pattern, such as relevant compute, IAM, networking and storage services. **Do not enable APIs during the read-only lab**; request approval first.

### Step 4 — Review networking

~~~powershell
gcloud compute networks list --project=$projectId
gcloud compute networks subnets list --project=$projectId
gcloud compute firewall-rules list --project=$projectId
gcloud compute routers list --project=$projectId
~~~

Review network ownership, subnet IP capacity, routes, DNS, firewall policies, Cloud NAT and Private Service Connect only where they are part of the approved architecture.

### Step 5 — Inspect regional quotas

~~~powershell
gcloud compute regions describe $region --project=$projectId --format="json(quotas)"
~~~

Quota availability does not guarantee zonal compute capacity. A quota response alone does not prove that a Databricks workspace can provision compute.

### Step 6 — Inspect service accounts and storage

~~~powershell
gcloud iam service-accounts list --project=$projectId
gcloud storage buckets list --project=$projectId
~~~

These commands may reveal sensitive names. Keep raw output private and avoid unnecessary enumeration outside the authorized project.

## 6. Terraform provisioning design

For GCP, workspace provisioning requires a **current, supported Databricks account deployment pattern**, not just a Google provider workspace resource. The exact Databricks Terraform resource, provider alias, account host, credential model and prerequisite objects can change.

This chapter deliberately separates a runnable **GCP prerequisite inspection scaffold** from the **workspace deployment module** to avoid presenting unverified cross-cloud resource schemas as ready to apply.

### versions.tf

~~~hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
    databricks = {
      source  = "databricks/databricks"
      version = "~> 1.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Account-level Databricks provider authentication must use
# the documented GCP account host and supported OAuth method.
# Configure separately after validating official provider docs.
provider "databricks" {
  alias = "accounts"
}
~~~

### variables.tf

~~~hcl
variable "project_id" {
  type        = string
  description = "Approved Google Cloud staging project"
}
variable "region" {
  type        = string
  description = "Verified Databricks-supported GCP region"
}
variable "workspace_name" {
  type        = string
  description = "Unique staging workspace label"
}
~~~

### terraform.tfvars.example

~~~hcl
project_id     = "<approved-staging-project-id>"
region         = "us-central1"
workspace_name = "demo-sre-staging"
~~~

This scaffold defines providers and inputs but **does not create a Databricks workspace**. A functional deployment must add provider-supported Databricks account resources and any required cloud prerequisites based on current official documentation.

### Validate the scaffold

~~~powershell
terraform init
terraform fmt -check
terraform validate
terraform providers
~~~

The scaffold is useful for provider authentication and architecture review, but a successful validate is not proof of deployment readiness.

## 7. Complete the deployment module safely

Before adding resource definitions or running terraform apply:

1. Confirm the Databricks GCP account workspace-creation method currently supported for the target account.
2. Confirm Google Cloud project and region compatibility.
3. Establish approved Databricks account OAuth and Google Cloud workload identity.
4. Implement documented cloud networking and required service account roles.
5. Configure supported workspace provisioning resources and dependencies.
6. Review encryption, data residency, audit, egress and storage access.
7. Configure remote state encryption, access control and locking.
8. Run Terraform plan, review the exact resource graph and approve costs.
9. Deploy only into a disposable staging project.
10. Validate the workspace and compute, then execute the teardown plan.

Official sources:
- https://docs.databricks.com/gcp/
- https://registry.terraform.io/providers/databricks/databricks/latest/docs
- https://cloud.google.com/docs/terraform

Do not guess Databricks account resource names or IAM trust configurations from AWS examples.

## 8. Deployment approval gate

| Control | Required evidence |
|---|---|
| Correct project and billing account | Project owner approval |
| Supported Databricks GCP region | Vendor documentation |
| Databricks account privileges | Approved principal and scope |
| Google IAM and service accounts | Reviewed role bindings |
| VPC, subnet, DNS and egress | Architecture review |
| Storage and data residency | Security review |
| Terraform state and plan protection | Platform standard |
| Cost estimate and teardown | Budget owner approval |
| Rollback and incident ownership | Change record |

A saved Terraform plan may contain sensitive values. Restrict access and do not publish it.

## 9. Post-deployment validation

After a fully documented and approved deployment:

- Verify workspace provisioning state and correct region in the Databricks account console/API.
- Verify workspace URL and authorized OAuth authentication.
- Check workspace policy and governance configuration.
- Validate approved classic compute provisioning, if applicable.
- Verify serverless availability and connectivity using supported signals.
- Confirm Cloud Storage authorization, DNS and egress for a minimal synthetic workload.
- Confirm Cloud Logging/Monitoring and Databricks audit visibility.
- Review resource labels, billing attribution and service limits.

A reachable workspace UI is not proof of a healthy compute plane.

## 10. Failure scenarios

### Scenario A — Missing API or IAM permission

**Symptom:** workspace provisioning fails with a Google API disabled or PermissionDenied error.

**Triage:** confirm active project, service account identity, required API, IAM role bindings, organization policies and Databricks account permissions.

**Mitigation:** obtain scoped authorization and enable only required APIs through approved change control. Do not grant project Owner as a blanket workaround.

### Scenario B — Network path failure

**Symptom:** workspace API is available but classic compute or data access cannot establish required connections.

**Triage:** inspect subnet selection, firewall egress, routes, DNS, NAT, private connectivity and endpoint requirements.

**Mitigation:** correct the documented network path; do not open unrestricted firewall rules.

### Scenario C — Capacity or quota

**Symptom:** cluster provisioning fails in one region/zone.

**Triage:** distinguish Compute Engine quota exhaustion, zonal capacity, policy limits and node type restrictions.

**Mitigation:** use an approved quota increase or alternative compute configuration and validate in staging.

### Scenario D — Cloud Storage access denied

**Symptom:** workload cannot read an authorized bucket.

**Triage:** compare the actual workload identity, bucket IAM, object paths, service account impersonation, network constraints and Databricks governance permissions.

**Mitigation:** grant only the minimum necessary access and validate with a synthetic test.

## 11. Troubleshooting matrix

| Symptom | Likely boundary | First safe check |
|---|---|---|
| Databricks account 401/403 | Account OAuth and roles | Validate account host and identity |
| Google PermissionDenied | Project IAM / org policy | Inspect active principal and resource scope |
| Workspace unavailable | Vendor service / DNS / network | Check status, endpoint and auth |
| Classic compute pending | Quota / capacity / networking | Inspect cluster events and region |
| Serverless errors | Managed compute / connectivity | Check supported workload telemetry |
| Storage denied | IAM / path / governance | Trace effective principal and grants |
| Unexpected charges | Short-lived compute / serverless | Correlate billing and workload usage |

## 12. Incident runbook

1. **Detect:** workspace availability, job errors, compute startup and storage access.
2. **Scope:** Databricks account, workspace, GCP project, region and affected workloads.
3. **Preserve:** UTC timeline, sanitized errors, compute/job identifiers in private records and relevant Cloud Audit Logs.
4. **Classify:** vendor service, IAM, network, compute capacity, storage or workload configuration.
5. **Mitigate:** smallest approved change, bounded retries or verified rollback.
6. **Validate:** authenticated API, synthetic workload, storage access and normal alerts.
7. **Document:** impact, actual cause or uncertainty, owner and follow-up actions.

## 13. Rollback and cleanup

For an isolated disposable staging deployment only:

1. Verify no production users, jobs or retained data depend on the workspace.
2. Preserve required configuration and audit evidence.
3. Review workspace deletion semantics and retained storage resources.
4. Review terraform plan -destroy for the exact managed resources.
5. Obtain approval; run terraform destroy only for the isolated stack.
6. Confirm service accounts, networking, storage and billing have no unintended leftovers.
7. Keep or securely dispose of Terraform state according to retention policy.

Never delete shared networks, buckets or service accounts simply because they appear in the same project.

## 14. Acceptance checklist

- [ ] Correct project, account and region verified.
- [ ] GCP APIs, IAM, network and quota preflight completed.
- [ ] Databricks managed-service boundaries documented.
- [ ] Terraform scaffold initialized and validated.
- [ ] Complete deployment module sourced from current official documentation before apply.
- [ ] Security, state, cost and rollback review completed.
- [ ] Four failure scenarios triaged.
- [ ] Approved staging deployment validated or explicitly recorded as not executed.
- [ ] No secrets or sensitive Terraform artifacts committed.

## 15. Review questions

1. Why does the active gcloud region not prove the workspace's region?
2. What separates Databricks account authorization from Google Cloud IAM?
3. How do classic and serverless networking differ?
4. What evidence separates a quota error from zonal capacity shortage?
5. Why must Terraform destroy be reviewed against shared dependencies?

## 16. References

- Databricks on Google Cloud: https://docs.databricks.com/gcp/
- Databricks Terraform provider: https://registry.terraform.io/providers/databricks/databricks/latest/docs
- Google Cloud Terraform: https://cloud.google.com/docs/terraform
- Google Cloud IAM: https://cloud.google.com/iam/docs
- Google Cloud VPC: https://cloud.google.com/vpc/docs
- Google Cloud quotas: https://cloud.google.com/docs/quotas

**Next:** Chapter 009 — VPC/VNet, Private Endpoints, DNS and Egress.
