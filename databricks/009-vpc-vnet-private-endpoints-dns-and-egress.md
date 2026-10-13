# Chapter 009 — VPC/VNet, Private Endpoints, DNS and Egress

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 02 — Cloud Provisioning and Networking  
**Clouds:** AWS, Azure, Google Cloud  
**Lab risk:** Read-only inspection and approved synthetic connectivity checks  
**Prerequisites:** Chapters 001–008

## 1. Learning objectives

By the end of this chapter, you can:
1. Map the network paths required by classic and serverless Databricks compute.
2. Distinguish control-plane access, compute-to-service traffic and data access.
3. Troubleshoot VPC/VNet routing, NAT, DNS, firewall and private endpoint failures.
4. Identify the AWS, Azure and Google Cloud equivalents without assuming they behave identically.
5. Collect incident evidence with timestamps and determine the failing network layer.
6. Propose the smallest approved network correction, verify recovery and document rollback.

## 2. Network architecture

~~~text
                   Users / CI / SRE
                          |
                 Identity + TLS + DNS
                          |
                Databricks workspace
                    Control plane
                          |
         +----------------+----------------+
         |                                 |
  Classic compute                    Serverless compute
  cloud-specific                     managed execution
  customer network                   separate connectivity
         |                                 |
    VPC / VNet                       supported private
    DNS + routes                     connectivity model
    firewall / NSG                         |
    NAT / endpoints                         |
         +----------------+----------------+
                          |
              Object storage / databases
              KMS / logging / APIs / Git
~~~

**Do not assume** all serverless workloads run inside the customer VPC/VNet or that a private workspace URL guarantees every downstream data connection is private.

## 3. Define the traffic flows first

| Flow | Questions to answer |
|---|---|
| User/CI → workspace | Public or private access? Which DNS name, identity and TLS path? |
| Classic compute → control plane | Which required outbound endpoints, routes and policies? |
| Classic compute → storage | Private endpoint or internet/NAT? Correct storage IAM? |
| Classic compute → external APIs | Proxy, NAT, firewall allowlists and TLS trust? |
| Serverless → data services | Which supported managed connectivity pattern applies? |
| Workspace → audit/monitoring | Are logs and network telemetry delivered reliably? |

Document **source, destination, port/protocol, DNS answer, route, security control and owner** for each critical flow. A successful DNS lookup is not proof of TCP connectivity, and a successful TCP handshake is not proof of application authorization.

## 4. Cross-cloud network mapping

| Concern | AWS | Azure | Google Cloud |
|---|---|---|---|
| Network | VPC | VNet | VPC |
| Subnet | VPC subnet | VNet subnet | VPC subnet |
| Traffic filtering | Security groups / NACLs | NSGs / firewall | VPC firewall rules / policies |
| Routes | Route tables | Route tables / UDR | VPC routes |
| Outbound translation | NAT gateway | NAT Gateway / Azure Firewall | Cloud NAT |
| Private service connectivity | VPC endpoints / PrivateLink | Private Endpoint / Private Link | Private Service Connect |
| DNS | Route 53 Resolver / private zones | Azure Private DNS / custom DNS | Cloud DNS / private zones |
| Flow evidence | VPC Flow Logs | NSG flow logs where supported / virtual network flow logs | VPC Flow Logs |
| Diagnostics | Reachability Analyzer / network tools | Network Watcher | Connectivity Tests |

Names are analogous, not interchangeable. For example, endpoint DNS behavior, routing and supported Databricks traffic vary by cloud and service mode.

## 5. Safety and prerequisites

- Use only authorized workspaces and cloud accounts/projects.
- Prefer read-only cloud CLI and local DNS/TCP tests.
- Never modify production routes, firewall rules, DNS zones or private endpoints as part of this lab.
- Avoid logging authentication tokens, connection strings, internal addresses or sensitive hostnames in public tickets or repositories.
- Use a disposable staging workload for in-compute tests; do not run arbitrary probes against restricted networks.
- Check current Databricks cloud-specific network requirements before implementing changes.

## 6. Lab A — Baseline workspace endpoint DNS and TLS (PowerShell)

### Step 1 — Record environment and time

~~~powershell
$collectedAt = (Get-Date).ToUniversalTime().ToString("o")
Write-Host "UTC: $collectedAt"
$workspaceHost = "example-workspace.cloud.databricks.com" # Replace with approved host
~~~

The example hostname is illustrative; do not infer the cloud provider or workspace URL format from it.

### Step 2 — Inspect DNS answers

~~~powershell
Resolve-DnsName $workspaceHost
Resolve-DnsName $workspaceHost -Type CNAME -ErrorAction SilentlyContinue
~~~

Record resolver used, answer type, TTL, CNAME chain and whether answers are expected for the network location. Private endpoints may require split-horizon DNS; a public answer can indicate incorrect DNS configuration but may also be expected for a different access mode.

### Step 3 — Test TCP 443

~~~powershell
Test-NetConnection -ComputerName $workspaceHost -Port 443 -InformationLevel Detailed
~~~

**Interpretation:** TCP success establishes basic transport reachability only. It does not prove TLS certificate validity, Databricks authentication or compute-plane connectivity.

### Step 4 — Check HTTP/TLS without credentials

~~~powershell
curl.exe -I --connect-timeout 10 --max-time 20 "https://$workspaceHost/"
~~~

A redirect or authorization response may be normal. Never paste bearer tokens into a terminal command that could enter shell history.

### Step 5 — Trace network path (optional)

~~~powershell
tracert $workspaceHost
~~~

Intermediate hops may suppress ICMP; an incomplete traceroute is not conclusive evidence of a service outage.

## 7. Lab B — AWS VPC network inventory

Verify the correct AWS account and region first:

~~~powershell
aws sts get-caller-identity
aws configure get region
aws ec2 describe-vpcs --query "Vpcs[].{VpcId:VpcId,Cidr:CidrBlock}" --output table
aws ec2 describe-subnets --query "Subnets[].{SubnetId:SubnetId,VpcId:VpcId,AZ:AvailabilityZone,Cidr:CidrBlock}" --output table
aws ec2 describe-route-tables --query "RouteTables[].{Id:RouteTableId,VpcId:VpcId}" --output table
aws ec2 describe-nat-gateways --query "NatGateways[].{Id:NatGatewayId,State:State,VpcId:VpcId}" --output table
aws ec2 describe-vpc-endpoints --query "VpcEndpoints[].{Id:VpcEndpointId,Service:ServiceName,State:State,VpcId:VpcId}" --output table
~~~

Review the exact workspace VPC, not arbitrary resources in the account. Validate security group egress, subnet associations, route targets, endpoint policies and private DNS configuration with approved read access.

**Failure patterns:** incorrect private endpoint DNS, missing NAT for required outbound services, endpoint policy denying storage, asymmetric routes, blocked security group egress.

## 8. Lab C — Azure VNet network inventory

~~~powershell
az account show --query "{subscription:id,tenant:tenantId}" -o json
az network vnet list --query "[].{name:name,rg:resourceGroup,location:location}" -o table
az network nsg list --query "[].{name:name,rg:resourceGroup}" -o table
az network route-table list --query "[].{name:name,rg:resourceGroup}" -o table
az network private-endpoint list --query "[].{name:name,rg:resourceGroup,location:location}" -o table
az network private-dns zone list --query "[].{name:name,rg:resourceGroup}" -o table
~~~

For a VNet-injected workspace, confirm dedicated subnets, required delegation, NSG association, effective routes and outbound design. A private endpoint requires correct DNS integration and access policies. Do not manually alter the Databricks managed resource group.

**Failure patterns:** custom DNS forwarding to the wrong resolver, missing private DNS zone link, firewall blocking control-plane egress, route table directing traffic to an unavailable appliance, storage private endpoint misconfiguration.

## 9. Lab D — Google Cloud VPC network inventory

~~~powershell
gcloud config get-value project
gcloud compute networks list
gcloud compute networks subnets list
gcloud compute routes list
gcloud compute firewall-rules list
gcloud compute routers list
gcloud compute forwarding-rules list
~~~

Filter results to the authorized project and region. Review Cloud NAT on the relevant router, Cloud DNS private zones, Private Service Connect endpoints and firewall policies where used.

**Failure patterns:** absent NAT path for private compute, firewall egress deny, DNS mismatch, incorrect PSC endpoint/service attachment, regional resource mismatch.

## 10. Lab E — Controlled in-compute connectivity test

Only in an approved disposable staging compute environment:

1. Confirm the workload's compute mode and actual network attachment.
2. Identify one authorized destination, such as a test storage endpoint.
3. Resolve its DNS name using an approved shell or notebook command.
4. Attempt a bounded TCP/TLS connection; do not scan networks.
5. Correlate results with cloud flow logs and the destination's access logs.
6. Stop the test compute and document the results.

**Do not** run network tests from a laptop and assume they represent cluster egress. Source location, DNS resolver, identity and routing can differ.

## 11. Network incident decision tree

~~~text
Application connection fails
          |
          v
Is DNS resolution correct?
    | no                 | yes
DNS zone/forwarder       v
and resolver checks   Is TCP reachable?
                         | no           | yes
                  Routes / NAT /        v
                  firewall / endpoint  Does TLS succeed?
                                          | no       | yes
                                    Certificate/    v
                                    proxy/SNI     Is auth allowed?
                                                   | no    | yes
                                             IAM/ACL/     Application
                                             policy      /service issue
~~~

A single failed probe may be transient. Confirm with repeatable, bounded tests and service-side evidence before declaring root cause.

## 12. Production failure scenarios

### Scenario A — Private endpoint DNS drift

**Inject:** after a DNS change, a staging job resolves an external service to an unexpected address.

**Evidence:** DNS answer before/after, private zone links, resolver/forwarder configuration, source network, timestamp and endpoint status.

**Mitigation:** restore the approved DNS record or forwarding configuration through change control.

**Validation:** expected DNS answer plus TCP/TLS and authorized application request from the affected compute mode.

**Rollback:** revert the specific DNS change; account for TTL and resolver caching.

### Scenario B — NAT or egress outage

**Inject:** classic compute starts but cannot reach required external APIs.

**Evidence:** effective route, NAT/firewall health, flow logs, destination DNS and endpoint response.

**Mitigation:** restore the approved egress path. Do not temporarily open all outbound traffic without review.

**Validation:** approved synthetic connection and normal job success rate.

**Rollback:** revert only the reviewed network change if it fails to restore connectivity.

### Scenario C — Storage private connectivity failure

**Inject:** storage access fails after a private endpoint or route change.

**Evidence:** DNS, endpoint status, storage firewall, source identity, authorization logs and network telemetry.

**Mitigation:** correct the network path and separately confirm storage IAM; a 403 can be authorization rather than network.

**Validation:** synthetic read/write limited to an approved test path.

### Scenario D — Serverless connectivity mismatch

**Inject:** classic compute reaches a private database but serverless fails.

**Evidence:** compute mode, supported serverless network connectivity configuration, destination allowlist and connection error.

**Mitigation:** implement the officially supported serverless connectivity pattern; do not assume VNet injection settings cover serverless.

## 13. Troubleshooting matrix

| Symptom | Likely area | Safe check |
|---|---|---|
| NXDOMAIN | DNS zone, record or forwarding | Compare authoritative and source resolver |
| Public address when private expected | Private DNS integration | Check zone link and endpoint DNS |
| TCP timeout | Routes, firewall, NAT, endpoint | Effective path and flow logs |
| TLS certificate mismatch | DNS, proxy, SNI, trust | Certificate chain and expected hostname |
| HTTP 401/403 | Identity, permissions, endpoint policy | Effective principal and service logs |
| Only classic fails | Customer compute network | VPC/VNet and classic routes |
| Only serverless fails | Serverless connectivity design | Supported managed network configuration |
| Intermittent failures | NAT exhaustion, DNS/cache, capacity | Time-correlated metrics and logs |

## 14. Production SRE runbook

1. **Declare scope:** workspace, cloud, region, source compute mode, destination and impacted workloads.
2. **Capture UTC timeline:** first failure, last known good and recent network changes.
3. **Validate DNS:** resolver, expected private/public answer and TTL.
4. **Validate transport:** bounded TCP/TLS from the actual affected source.
5. **Inspect path:** subnet, route, NAT, firewall, endpoint and cloud flow telemetry.
6. **Separate authorization:** check IAM, bucket policy, Unity Catalog or database permissions.
7. **Mitigate:** smallest approved change with a documented rollback.
8. **Verify:** synthetic connectivity and real workload health, plus error-rate recovery.
9. **Document:** evidence, owner, incident impact, preventive controls and follow-up.

## 15. Monitoring and SLO signals

Recommended signals include:
- DNS resolution failures and latency from approved synthetic tests.
- Network connection timeouts and TLS failures by destination.
- NAT gateway/firewall saturation and drop counters where available.
- Private endpoint availability and configuration drift.
- Cluster startup failure rates and job network-related errors.
- Storage access failures split by transport versus authorization.
- Serverless connectivity failure rate and cost/usage context.

Alert on sustained user impact rather than one isolated probe failure. Ensure network telemetry retention matches incident investigation needs.

## 16. Acceptance checklist

- [ ] Source/destination flow inventory completed.
- [ ] Classic versus serverless networking documented.
- [ ] DNS and TCP/TLS baseline collected from authorized location.
- [ ] Cloud-specific VPC/VNet routing and endpoint inventory reviewed.
- [ ] Private DNS, NAT and egress owners identified.
- [ ] Four failure scenarios investigated.
- [ ] Incident runbook, validation and rollback steps reviewed.
- [ ] No production networking changed.
- [ ] No sensitive endpoints, tokens or internal identifiers committed.

## 17. Review questions

1. Why does DNS success not prove application connectivity?
2. Why can a connection work from a laptop but fail from Databricks compute?
3. What distinguishes a storage 403 from a network timeout?
4. Why might classic compute and serverless have different private access behavior?
5. Which evidence would justify rolling back a DNS or NAT change?

## 18. References

- Databricks on AWS networking: https://docs.databricks.com/aws/en/security/network/
- Azure Databricks networking: https://learn.microsoft.com/azure/databricks/security/network/
- Databricks on Google Cloud: https://docs.databricks.com/gcp/
- AWS VPC: https://docs.aws.amazon.com/vpc/
- Azure Virtual Network: https://learn.microsoft.com/azure/virtual-network/
- Google Cloud VPC: https://cloud.google.com/vpc/docs

**Next:** Chapter 010 — Cross-Cloud Networking Acceptance Lab, the Part 02 capstone.
