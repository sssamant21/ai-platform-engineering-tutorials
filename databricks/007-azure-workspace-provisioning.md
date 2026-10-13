# Chapter 007 — Azure Workspace Provisioning

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 02 — Cloud Provisioning and Networking  
**Cloud:** Microsoft Azure  
**Risk:** Read-only preflight; optional billable staging deployment

## 1. Objectives

- Explain Azure Databricks workspace, managed resource group, classic and serverless compute boundaries.
- Compare managed networking, VNet injection and private connectivity.
- Validate Azure subscription, region, resource provider, quotas, identity, storage and networking.
- Create a minimal staging workspace through Terraform after change approval.
- Investigate provisioning failures and verify safe rollback and cleanup.

## 2. Architecture

~~~text
SRE / Terraform / Azure CLI
          |
Azure subscription + Entra ID + RBAC
          |
Resource group -> Azure Databricks workspace
                        |
                Databricks control plane
                        |
            +-----------+-----------+
            |                       |
       Classic compute        Serverless compute
     (workspace model)       (Databricks-managed)
            |
      VNet / NSG / routes
            |
       ADLS Gen2 / services
~~~

Azure Databricks is a managed service. Azure hosts the resource and infrastructure services; customers govern identity, workspace settings, access, policies, network design, data protection and operations. The Databricks **managed resource group** contains service-managed resources. Avoid direct edits to those resources except where expressly supported.

Serverless networking is distinct from customer VNet-injected classic compute. Never assume serverless workloads run in the injected VNet.

## 3. Deployment models

| Model | Operational considerations |
|---|---|
| Managed networking | Simpler workspace creation; confirm enterprise network policy and supported egress |
| VNet injection | Customer-managed VNet, dedicated subnets, delegation, NSGs, routing and supported outbound paths |
| Private connectivity | Private Link/DNS design and endpoint connectivity; not a substitute for all required egress |
| Serverless | Separate managed execution, access, networking and cost controls |

**VNet injection:** Review current subnet size, delegation to Microsoft.Databricks/workspaces, NSG associations, UDR, NAT and service connectivity requirements. Do not infer that a VNet can be retrofitted to any existing workspace.

## 4. Responsibility and dependencies

| Layer | SRE verification |
|---|---|
| Azure subscription | Tenant, RBAC, policy, provider registration, quotas, budget |
| Resource group | Scope, tags, ownership, lifecycle |
| Workspace | Name, region, SKU, provisioning state, access |
| Identity | Entra principals, service principals, managed identities, scoped roles |
| Networking | VNet, DNS, subnets, NSG, routing, egress, Private Link |
| Storage | ADLS Gen2 RBAC/ACL, network, Unity Catalog grants where applicable |
| Compute | VM quotas, family capacity, policy, runtime, startup events |
| Telemetry | Azure Activity Log, diagnostics, Databricks events, cost attribution |

## 5. Hands-on lab A — Read-only Azure preflight

**Prerequisites:** Azure CLI, Terraform, approved subscription access and an authorized staging target. Do not copy tenant IDs, secrets, internal hosts or sensitive outputs into GitHub.

### Step 1 — Validate tools and identity

~~~powershell
az version
terraform version
databricks version
az login
az account show --query "{name:name,id:id,tenantId:tenantId}" -o json
az account list --query "[].{name:name,id:id,isDefault:isDefault}" -o table
~~~

Use organizationally approved interactive or federated login. Do not run interactive login unattended.

### Step 2 — Select the staging subscription

~~~powershell
$subscriptionId = "<approved-staging-subscription-id>"
az account set --subscription $subscriptionId
az account show --query "{name:name,id:id}" -o json
~~~

### Step 3 — Verify provider and regional capacity

~~~powershell
az provider show --namespace Microsoft.Databricks --query registrationState -o tsv
az vm list-usage --location eastus -o table
az account list-locations --query "[].name" -o tsv
~~~

Replace eastus with the verified target region. VM quotas are not a guarantee of physical capacity. If Microsoft.Databricks is unregistered, obtain approval before registering it.

### Step 4 — Review existing networks and workspaces

~~~powershell
az network vnet list --query "[].{name:name,rg:resourceGroup,location:location}" -o table
az network nsg list --query "[].{name:name,rg:resourceGroup}" -o table
az network route-table list --query "[].{name:name,rg:resourceGroup}" -o table
az databricks workspace list --query "[].{name:name,rg:resourceGroup,location:location,sku:sku.name}" -o table
~~~

Record network ownership, address space, subnet delegation, NSG, routes, DNS, egress and private endpoint dependencies. Listing a VNet does not establish that it is attached to a workspace.

## 6. Hands-on lab B — Terraform staging workspace

The following minimal example uses **managed networking**. It does not implement VNet injection, private connectivity, customer-managed keys, Unity Catalog configuration or production controls. Use a disposable resource group and a reviewed state backend.

### main.tf

~~~hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id
}

variable "subscription_id" {
  type = string
}
variable "location" {
  type    = string
  default = "eastus"
}
variable "resource_group_name" {
  type    = string
  default = "rg-demo-databricks-staging"
}
variable "workspace_name" {
  type    = string
  default = "demo-sre-staging"
}

resource "azurerm_resource_group" "staging" {
  name     = var.resource_group_name
  location = var.location
  tags = {
    environment = "staging"
    owner       = "platform-sre"
    managed_by  = "terraform"
  }
}

resource "azurerm_databricks_workspace" "staging" {
  name                = var.workspace_name
  resource_group_name = azurerm_resource_group.staging.name
  location            = azurerm_resource_group.staging.location
  sku                 = "premium"
  tags = {
    environment = "staging"
    owner       = "platform-sre"
    managed_by  = "terraform"
  }
}

output "workspace_resource_id" {
  value = azurerm_databricks_workspace.staging.id
}
~~~

### terraform.tfvars.example

~~~hcl
subscription_id     = "<approved-staging-subscription-id>"
location            = "eastus"
resource_group_name = "rg-demo-databricks-staging"
workspace_name      = "demo-sre-staging"
~~~

Create terraform.tfvars privately from the example. Review workspace naming uniqueness, region support, subscription policy, billing, security and state management before deployment.

### Validate and plan

~~~powershell
terraform fmt -check
terraform init
terraform validate
terraform plan -var-file="terraform.tfvars" -out="staging.tfplan"
terraform show staging.tfplan
~~~

**Safety:** Terraform state and saved plans may contain sensitive information. Do not commit them or print them in public CI logs. A successful validate is not a security review.

### Optional approved staging apply

~~~powershell
terraform apply staging.tfplan
terraform output
~~~

Apply only after reviewing the plan, selecting the intended subscription and obtaining change approval.

## 7. VNet injection design extension

For a separate customer-managed networking variant, document:
1. VNet and appropriately sized dedicated host/container subnets.
2. Subnet delegation, NSG associations and supported network policies.
3. Route tables, outbound connectivity, NAT/firewall and DNS.
4. Workspace Terraform custom parameters and subnet references.
5. Private Link endpoints and DNS zones if required.
6. Change control and rollback plan.

Use current official Azure Databricks guidance. Never disable NSGs or open unrestricted network access to work around provisioning errors.

## 8. Post-deployment checks

~~~powershell
az databricks workspace show --resource-group rg-demo-databricks-staging --name demo-sre-staging --query "{name:name,location:location,state:provisioningState,url:workspaceUrl}" -o json
~~~

Verify workspace resource state, intended region and SKU, authenticated API access, tags, audit and cost visibility. Test classic compute only in approved staging; workspace resource success does not prove compute capacity or storage access.

## 9. Troubleshooting and failure exercises

| Symptom | Investigation | Safe mitigation |
|---|---|---|
| Deployment 403 | Subscription/tenant, RBAC, Azure Policy, deny assignment | Request scoped permissions |
| Provider not registered | Microsoft.Databricks registration | Approved registration |
| VNet injection fails | Delegation, subnet sizes, NSG, routes | Correct reviewed network configuration |
| Workspace healthy, cluster pending | VM quota/capacity, network, DNS, policy | Inspect compute events; use approved remediation |
| ADLS access denied | Identity, RBAC, ACL, firewall, Unity Catalog | Grant minimum required access |
| Terraform drift | Out-of-band changes and state | Review plan before reconciliation |

**Tabletop A:** A workspace creation fails due to Azure Policy. Collect error code, policy assignment, target subscription and deployment operation details; propose a compliant design, not a policy bypass.

**Tabletop B:** Workspace UI works but classic compute cannot start. Distinguish VM quota, capacity, subnet routing and identity causes using timestamped evidence.

**Tabletop C:** Storage access fails. Compare principal identity, storage firewall/private endpoints, RBAC, ACL and catalog grants. Do not disable storage protections.

## 10. SRE incident runbook

1. **Detect:** workspace, compute startup and job failures.
2. **Scope:** subscription, region, resource group, workspace, compute mode and affected workloads.
3. **Preserve:** UTC timestamps, Azure Activity Log, Databricks events and sanitized error messages.
4. **Classify:** RBAC/policy, service provisioning, network/DNS, capacity, identity or storage.
5. **Mitigate:** make the smallest approved change and preserve rollback information.
6. **Validate:** workspace API, synthetic job, data access and alert recovery.
7. **Document:** timeline, cause, uncertainty, remediation, cost and follow-up owners.

## 11. Teardown and rollback

For the isolated staging stack only:
1. Confirm no jobs, data or downstream users depend on the workspace.
2. Preserve approved audit evidence and configuration.
3. Review terraform plan -destroy and identify any shared resources.
4. Obtain approval and run terraform destroy for the isolated stack.
5. Check the managed resource group lifecycle and orphaned cloud resources.
6. Verify cost stops accruing and retain state/audit records according to policy.

Never manually delete service-managed resources or shared VNets/storage as a shortcut.

## 12. Acceptance checklist

- [ ] Subscription, tenant, region and provider registration verified.
- [ ] Networking model and responsibility boundaries documented.
- [ ] Identity, network, compute and storage dependencies mapped.
- [ ] Terraform formatted, initialized and validated.
- [ ] Plan reviewed for security, cost and policy.
- [ ] Apply performed only with staging approval or explicitly skipped.
- [ ] Provisioning and compute health validated where deployed.
- [ ] Three failure exercises completed.
- [ ] Teardown plan reviewed.
- [ ] No secrets, state or saved plans committed.

## 13. Review questions

1. Why should the Databricks managed resource group not be manually edited?
2. What distinguishes VNet-injected classic compute from serverless networking?
3. Why might workspace provisioning succeed while cluster startup fails?
4. How do Azure RBAC, storage ACLs and Unity Catalog permissions differ?
5. What evidence is required before destroying a failed staging deployment?

## 14. References

- Azure Databricks: https://learn.microsoft.com/azure/databricks/
- Workspace administration: https://learn.microsoft.com/azure/databricks/admin/workspace/
- VNet injection: https://learn.microsoft.com/azure/databricks/security/network/classic/vnet-inject
- Terraform AzureRM resource: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/databricks_workspace
- Azure quotas: https://learn.microsoft.com/azure/quotas/

**Next:** Chapter 008 — GCP Workspace Provisioning.
