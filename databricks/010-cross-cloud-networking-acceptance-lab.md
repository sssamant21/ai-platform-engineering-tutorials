# Chapter 010 — Cross-Cloud Networking Acceptance Lab

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 02 — Cloud Provisioning and Networking (capstone)  
**Clouds:** AWS, Azure, Google Cloud  
**Audience:** SRE, Network Infrastructure, Cloud Platform and Security Operations  
**Risk:** Read-only evidence collection and approved synthetic probes only  
**Prerequisites:** Chapters 006–009

## 1. Mission

Validate that a Databricks workspace's **documented and authorized** network paths are ready for production operations. The exercise covers workspace access, classic compute, serverless connectivity, cloud storage, private endpoints, DNS, routing, egress, observability and rollback.

**Acceptance is per workspace and compute mode.** Passing on AWS does not certify Azure or GCP. Passing from a laptop does not certify compute egress.

### Expected outcomes

- An accurate source-to-destination traffic matrix.
- A verified inventory of cloud network resources and ownership.
- DNS, TCP/TLS and application-layer evidence from approved test locations.
- Separate classic and serverless connectivity findings.
- Four incident simulations with mitigation and rollback.
- An auditable pass/partial/fail/not-assessed scorecard and go/no-go recommendation.

## 2. Scope and safety

**In scope:** cloud read-only APIs, Databricks workspace inventory, approved DNS and bounded TCP/TLS probes, synthetic test data and tabletop exercises.

**Out of scope:** changing production firewall rules, modifying routes, deleting private endpoints, broad port scans, disabling DNS security, rotating production credentials or intentionally disrupting production.

Before testing:
1. Identify the exact workspace, cloud, region and environment.
2. Confirm approved test destinations and owners.
3. Use an approved staging compute resource for compute-origin probes.
4. Keep raw IPs, hostnames, IDs, certificates, logs and identities in restricted evidence storage.
5. Record every test's UTC timestamp, source network and compute mode.

## 3. Reference architecture

~~~text
                   SRE / CI / user
                        |
               DNS / identity / TLS
                        |
               Databricks workspace
                        |
              +---------+---------+
              |                   |
        Classic compute      Serverless compute
        customer network     managed connectivity
              |                   |
        VPC / VNet / VPC          |
        subnet / routes           |
        firewall / NAT            |
        private endpoints         |
              +---------+---------+
                        |
            Storage / APIs / databases
                        |
               Audit / monitoring
~~~

Not every arrow is a direct network path in every cloud. Confirm the supported traffic flows and current vendor requirements for the selected workspace.

## 4. Cloud equivalence and differences

| Capability | AWS | Azure | GCP |
|---|---|---|---|
| Virtual network | VPC | VNet | VPC |
| Private connectivity | PrivateLink/VPC endpoints | Private Link/private endpoints | Private Service Connect |
| Egress translation | NAT gateway | NAT Gateway/Azure Firewall | Cloud NAT |
| DNS | Route 53 Resolver/private hosted zones | Azure DNS/private zones/custom DNS | Cloud DNS/private zones |
| Packet filtering | Security groups/NACL | NSG/firewall | VPC firewall rules |
| Routing | VPC route tables | UDR/route tables | VPC routes |
| Network evidence | VPC Flow Logs | Virtual network flow logs/Network Watcher | VPC Flow Logs/Connectivity Tests |

Cloud services are not one-to-one equivalents. Endpoint DNS, control-plane connectivity and serverless access patterns vary by deployment.

## 5. Deliverables

| Artifact | Minimum evidence |
|---|---|
| Workspace profile | Cloud, verified region, workspace label, environment, owners |
| Traffic flow matrix | Source, destination, DNS, port, route, policy, owner |
| Cloud inventory | Network, subnets, routes, firewall, NAT, private endpoints, DNS |
| Test record | UTC time, source, method, expected/actual result, evidence location |
| Compute-mode coverage | Classic and serverless independently assessed |
| Failure reports | Hypothesis, supporting evidence, mitigation, validation, rollback |
| Acceptance scorecard | Status, reviewer, owner and remediation ticket |

## 6. Lab A — Establish an evidence baseline

Use PowerShell on Windows:

~~~powershell
$root = Join-Path $env:USERPROFILE "databricks-sre-ch010"
New-Item -ItemType Directory -Force -Path $root | Out-Null
$timestampUtc = (Get-Date).ToUniversalTime().ToString("o")
Write-Host "Assessment UTC: $timestampUtc"

databricks version
databricks auth profiles
databricks auth describe
~~~

Verify the active Databricks profile points to the authorized workspace. Do not export auth profiles or tokens to the evidence directory.

### Workspace profile worksheet

| Field | Example only |
|---|---|
| Workspace label | demo-staging |
| Cloud | AWS / Azure / GCP |
| Region | Verified from workspace/account configuration |
| Compute modes | Classic, SQL, serverless as applicable |
| Primary network owner | Platform networking |
| Security owner | Cloud security |
| Assessment UTC | Current time |
| Change/ticket | Approved assessment record |

## 7. Lab B — Build the traffic flow matrix

For each approved critical flow, record:

| Source | Destination | Protocol | DNS expectation | Route/endpoint | Owner | Status |
|---|---|---|---|---|---|---|
| SRE workstation | Workspace UI/API | HTTPS/443 | Per access design | Public/private | Platform | Not assessed |
| Classic compute | Required control services | Vendor documented | Per cloud | NAT/private path | Platform | Not assessed |
| Classic compute | Test object storage | HTTPS/443 | Private if designed | Endpoint/NAT | Data platform | Not assessed |
| Serverless | Approved test data service | Supported protocol | Per serverless design | Supported managed path | Platform | Not assessed |
| Compute | Monitoring/audit sink | Vendor documented | Per design | Approved path | SRE | Not assessed |

**Do not assume** a port or route without checking the relevant vendor requirements. Separate authorization (IAM/catalog permissions) from transport connectivity.

## 8. Lab C — DNS and transport validation

Use only an approved workspace hostname:

~~~powershell
$workspaceHost = "<approved-workspace-hostname>"

Resolve-DnsName $workspaceHost
Resolve-DnsName $workspaceHost -Type CNAME -ErrorAction SilentlyContinue
Test-NetConnection -ComputerName $workspaceHost -Port 443 -InformationLevel Detailed
curl.exe -I --connect-timeout 10 --max-time 20 "https://$workspaceHost/"
~~~

### Interpret results

- **DNS fails:** investigate resolver, forwarding, private zone and record ownership.
- **DNS returns unexpected public/private address:** check whether split-horizon DNS matches the approved design.
- **TCP fails:** inspect routes, firewall, NAT, endpoints and source location.
- **TLS fails:** check hostname, certificate chain, SNI and interception/proxy.
- **HTTP 401/403:** connectivity may be healthy; investigate identity or endpoint authorization.
- **HTTP succeeds:** does not prove classic compute or serverless connectivity.

Capture sanitized outputs and the actual source network. Never publish internal hostnames or addresses.

## 9. Lab D — Cloud network inventory

Run **only the block for the relevant cloud** after verifying its active account, subscription or project.

### AWS

~~~powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-vpcs --query "Vpcs[].{Id:VpcId,Cidr:CidrBlock}" --output table
aws ec2 describe-subnets --query "Subnets[].{Id:SubnetId,Vpc:VpcId,AZ:AvailabilityZone}" --output table
aws ec2 describe-route-tables --query "RouteTables[].{Id:RouteTableId,Vpc:VpcId}" --output table
aws ec2 describe-nat-gateways --query "NatGateways[].{Id:NatGatewayId,State:State}" --output table
aws ec2 describe-vpc-endpoints --query "VpcEndpoints[].{Id:VpcEndpointId,Service:ServiceName,State:State}" --output table
~~~

Confirm actual subnet-to-route-table association, security group egress, endpoint policy and DNS settings; resource listing alone is insufficient.

### Azure

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az network vnet list --query "[].{name:name,rg:resourceGroup}" -o table
az network nsg list --query "[].{name:name,rg:resourceGroup}" -o table
az network route-table list --query "[].{name:name,rg:resourceGroup}" -o table
az network private-endpoint list --query "[].{name:name,rg:resourceGroup}" -o table
az network private-dns zone list --query "[].{name:name,rg:resourceGroup}" -o table
~~~

For VNet injection, verify required subnet delegation, NSG associations, DNS and outbound routing. Do not modify the Databricks-managed resource group.

### Google Cloud

~~~powershell
gcloud config get-value project
gcloud compute networks list
gcloud compute networks subnets list
gcloud compute routes list
gcloud compute firewall-rules list
gcloud compute routers list
gcloud compute forwarding-rules list
~~~

Inspect the approved project, relevant region, Cloud NAT configuration, DNS private zones and PSC design where applicable.

### Cloud acceptance evidence

Record:
1. Expected network and subnet IDs (privately).
2. Effective routes and outbound dependencies.
3. Firewall policy owner and relevant allow/deny decisions.
4. Private endpoint status and DNS association.
5. Logging/monitoring enabled and accessible.
6. Configuration drift or permission gaps.

## 10. Lab E — Classic compute source validation

In an approved disposable staging cluster only:
1. Record cluster ID privately, compute policy and actual network.
2. Resolve the approved storage or test-service hostname from the cluster.
3. Run a bounded TCP/TLS connection to that destination.
4. Execute a harmless authorized synthetic read (and write only to a designated test path).
5. Correlate failures with cloud flow logs and service-side logs.
6. Terminate disposable compute after collecting evidence.

**Do not** assume the workstation DNS result is the same as the cluster DNS result.

## 11. Lab F — Serverless connectivity acceptance

Where serverless is enabled:
1. Identify the serverless workload type and supported networking configuration.
2. Confirm its access path to the designated test service.
3. Execute an approved minimal synthetic query or workload.
4. Record DNS/transport or service-level evidence exposed by supported tooling.
5. Compare results with classic compute without assuming the same network source.
6. Mark untestable serverless paths **Not assessed**, not Pass.

## 12. Incident simulation 1 — Private DNS drift

**Inject:** a private endpoint's DNS configuration changes and a test workload resolves an unexpected address.

**Evidence:** source resolver, CNAME/A record, TTL, zone link, endpoint state, timestamp.

**Diagnosis:** determine whether the answer is incorrect for the intended access mode, not simply whether it is public.

**Mitigation:** restore the reviewed DNS integration.

**Validation:** correct resolution and authorized TCP/TLS/application request from the affected source.

**Rollback:** revert the approved DNS change and account for caching.

## 13. Incident simulation 2 — NAT or firewall egress failure

**Inject:** classic compute can start but cannot reach an approved external dependency.

**Evidence:** route table, NAT/firewall health, flow logs, DNS, destination port and timing.

**Diagnosis:** separate blocked traffic, absent route, NAT exhaustion, remote service outage and proxy/TLS issues.

**Mitigation:** repair the specific approved path. Do not create unrestricted allow-all rules.

**Validation:** successful bounded synthetic request and recovery of affected job errors.

## 14. Incident simulation 3 — Storage private endpoint failure

**Inject:** object storage access fails after a network configuration change.

**Evidence:** private endpoint/PSC status, DNS, firewall, storage authorization, cloud logs.

**Diagnosis:** a storage 403 may indicate IAM/ACL rather than network transport. Compare network and authorization evidence.

**Mitigation:** restore correct network access and scoped identity permissions through change control.

**Validation:** authorized synthetic operation on a disposable test path.

## 15. Incident simulation 4 — Serverless and classic divergence

**Inject:** classic compute reaches a private service while serverless cannot.

**Evidence:** compute modes, actual network paths, supported serverless connectivity configuration and destination logs.

**Diagnosis:** do not attribute the failure to classic VPC/VNet routing without proof.

**Mitigation:** use the vendor-supported serverless connectivity architecture.

**Validation:** independent synthetic success for each required compute mode.

## 16. Troubleshooting decision tree

~~~text
Connection failure
       |
       v
DNS correct for source?
  | no                | yes
  v                   v
Resolver / zone     TCP reachable?
                      | no          | yes
                      v             v
                  Route / NAT /   TLS valid?
                  firewall /       | no     | yes
                  endpoint         v        v
                               SNI/cert/  App auth?
                               proxy      | no   | yes
                                          v      v
                                        IAM/   Service /
                                        ACL    workload
~~~

Use time-correlated evidence. Do not declare a root cause based on one failed probe.

## 17. Acceptance scorecard

Use **Pass**, **Partial**, **Fail** or **Not assessed**. Each row must have evidence and a named owner.

| Control | Rating | Evidence / owner |
|---|---|---|
| Workspace cloud, region and network mode verified | Not assessed | |
| Workspace API DNS/TCP/TLS baseline | Not assessed | |
| Classic compute connectivity tested | Not assessed | |
| Serverless connectivity tested or limitation recorded | Not assessed | |
| Required storage access validated | Not assessed | |
| Private endpoint and DNS design validated | Not assessed | |
| NAT/egress and firewall rules reviewed | Not assessed | |
| Network flow telemetry accessible | Not assessed | |
| Four failure scenarios completed | Not assessed | |
| Change rollback documented and reviewed | Not assessed | |
| Security and operational owners identified | Not assessed | |
| Outstanding risks tracked | Not assessed | |

### Exit decision

**Pass:** all critical flows have evidence and no unresolved critical failures.

**Conditional pass:** documented noncritical gaps with owners, due dates and explicit risk acceptance.

**Fail:** a required production flow is broken, unowned, untestable without risk acceptance, or lacks a safe recovery path.

A successful CLI inventory or laptop HTTPS request cannot override a failed critical compute-to-data path.

## 18. Capstone submission template

~~~text
Assessment: Databricks Cross-Cloud Networking Acceptance
Workspace label:
Cloud and verified region:
Environment:
Assessment UTC:
Assessor / reviewer:
Change or test approval:
Classic compute network model:
Serverless connectivity model:

Traffic matrix location (private):
DNS/TLS evidence location (private):
Cloud inventory evidence location (private):
Storage test evidence location (private):

Scenario 1 — DNS drift:
Scenario 2 — NAT/firewall:
Scenario 3 — Storage private connectivity:
Scenario 4 — Serverless/classic divergence:

Critical findings:
Risk acceptance:
Remediation owners and dates:
Go / Conditional go / No-go:
Reviewer sign-off:
~~~

## 19. Cleanup and safety verification

- Stop or terminate only the **approved disposable** synthetic compute.
- Remove temporary synthetic data from designated test paths.
- Preserve required evidence under the organization's retention policy.
- Confirm no production network, IAM or DNS resources were modified.
- Do not upload raw network logs, internal IPs, account identifiers or secrets to public GitHub.

## 20. Review questions

1. Why must connectivity acceptance be performed per source and compute mode?
2. How does private DNS drift differ from an IAM failure?
3. What distinguishes a quota/capacity problem from a routing problem?
4. Why does a reachable secondary workspace not guarantee DR connectivity?
5. What evidence is sufficient to approve a production network change?

## 21. References

- Databricks on AWS networking: https://docs.databricks.com/aws/en/security/network/
- Azure Databricks networking: https://learn.microsoft.com/azure/databricks/security/network/
- Databricks on GCP: https://docs.databricks.com/gcp/
- AWS VPC: https://docs.aws.amazon.com/vpc/
- Azure Virtual Network: https://learn.microsoft.com/azure/virtual-network/
- Google Cloud VPC: https://cloud.google.com/vpc/docs

**Part 02 complete. Next:** Chapter 011 — Compute Types, Lifecycle and Workload Isolation.
