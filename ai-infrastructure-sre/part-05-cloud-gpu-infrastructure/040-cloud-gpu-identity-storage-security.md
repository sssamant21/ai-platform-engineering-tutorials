# Chapter 040 — Cloud GPU Identity, Storage and Security

**AI Infrastructure Engineering & SRE | Part 05 — Cloud GPU Infrastructure | Chapter 040/100**

**Publication state:** Tutorial and offline lab published. AWS/Azure staging validation, threat modeling, independent security review and production signoff are separate pending activities.

## Learning objectives

Design least-privilege GPU infrastructure spanning AWS EKS/EC2 and Azure AKS/VMs. Separate node identity, workload identity, human access, service-to-service identity and break-glass operations. Secure datasets, models and checkpoints across object, block and parallel storage. Implement encryption, key management, network isolation, image provenance, secret handling, tenant isolation, audit evidence, incident response, RPO/RTO and a fail-closed release gate.

## 1. Enterprise reference architecture

```text
Identity plane                         Data plane
SSO / human RBAC                       S3 / Azure Blob
  |                                      |
Change approval + break glass          KMS / Key Vault
  |                                      |
EKS IRSA / EKS Pod Identity            FSx / managed disks / parallel FS
AKS Microsoft Entra Workload ID        |
  |                                   datasets / model artifacts
GPU node role / managed identity        |
  |                                    training checkpoint + retention
Kubernetes RBAC / admission            |
  |                                    workload read/write grants
  +------------ GPU workload -----------+
                       |
          container image + signed provenance
                       |
         private networking + egress controls
                       |
       audit / telemetry / DLP / incident response
```

**Security principle:** Cloud instance/node identity is not an appropriate default identity for every training or inference pod. Workloads receive scoped, short-lived identities for only the datasets, model artifacts and checkpoint locations they need. Identity authentication, authorization, network reachability and storage encryption are separate controls.

## 2. Threat model and trust boundaries

| Asset | Threat | Primary defense |
|---|---|---|
| GPU node | privileged escape, driver exploit | hardened image, patching, runtime boundary |
| Training dataset | exfiltration, overbroad read | scoped workload identity, private endpoint |
| Model weights | theft, substitution | encryption, artifact signatures, immutable digests |
| Checkpoint | corruption, ransomware, rollback attack | versioning, integrity, retention, restore drill |
| GPU tenancy | cross-tenant process or memory access | dedicated pools, device allocation, admission |
| CI/CD | poisoned image or IaC | protected branches, signed provenance, policy gates |
| Secrets | token theft from logs/env | workload federation, secret manager, rotation |
| Observability | sensitive prompts/data leakage | redaction, restricted retention and access |
| KMS keys | outage or unauthorized decrypt | scoped key policy, availability, recovery plan |

GPU sharing and time-slicing do not automatically provide strong tenant isolation. MIG can improve resource separation on supported hardware but is not a substitute for threat modeling, runtime hardening and authorization controls.

## 3. AWS identity: EKS and EC2

For EKS workloads use supported **EKS Pod Identity** or **IAM Roles for Service Accounts (IRSA)**, with narrowly scoped IAM permissions and trust conditions. EKS node IAM roles should support node infrastructure functions, not grant broad dataset access to arbitrary pods. For standalone EC2 GPU workloads, use instance profiles only for the host's necessary permissions; consider per-workload credential isolation where supported.

AWS examples: restrict S3 read to approved dataset prefixes, checkpoint write to job-specific prefixes, KMS decrypt/encrypt only for authorized key and encryption context, and FSx access to required network and file permissions. Validate bucket policy, IAM policy, key policy, VPC endpoint policy and SCP interplay. Never assume IAM allow overrides explicit deny.

**Read-only discovery:**

```bash
export AWS_PROFILE=approved-readonly
aws sts get-caller-identity --profile "$AWS_PROFILE"
aws eks describe-cluster --name APPROVED_CLUSTER --region us-east-1 \
  --profile "$AWS_PROFILE" --query 'cluster.{Name:name,Identity:identity,Endpoint:resourcesVpcConfig.endpointPublicAccess}'
aws s3api get-bucket-encryption --bucket APPROVED_BUCKET --profile "$AWS_PROFILE"
aws s3api get-public-access-block --bucket APPROVED_BUCKET --profile "$AWS_PROFILE"
aws kms describe-key --key-id APPROVED_KEY_ARN --profile "$AWS_PROFILE" \
  --query 'KeyMetadata.{KeyId:KeyId,State:KeyState,Enabled:Enabled}'
```

Do not include full IAM policies or secret-bearing environment variables in public evidence without redaction.

## 4. Azure identity: AKS and GPU VMs

Use **Microsoft Entra Workload ID** for AKS service accounts where supported, with federated identity credentials and scoped Azure RBAC. For GPU VMs, use managed identities rather than stored service-principal secrets. Scope Blob/ADLS access to the necessary container/path and applicable data-plane role. Separate Azure management-plane Contributor permissions from storage data-plane roles. Use Azure Key Vault with RBAC and private endpoints where approved.

**Read-only discovery:**

```bash
az account show --query '{subscription:name,id:id,tenant:tenantId}' -o json
az aks show --resource-group APPROVED_RG --name APPROVED_AKS \
  --query '{Name:name,WorkloadIdentity:securityProfile.workloadIdentity,OIDC:oidcIssuerProfile.enabled}'
az storage account show --resource-group APPROVED_RG --name APPROVED_STORAGE \
  --query '{Name:name,PublicNetworkAccess:publicNetworkAccess,MinTls:minimumTlsVersion,Encryption:encryption}'
az keyvault show --name APPROVED_VAULT \
  --query '{Name:name,PublicNetworkAccess:properties.publicNetworkAccess,EnableRbacAuthorization:properties.enableRbacAuthorization}'
```

Exact CLI field availability varies by Azure API and CLI release. Do not assume that a management-plane role grants Blob data read.

## 5. Kubernetes GPU workload isolation

Apply least-privilege Kubernetes RBAC, Pod Security Admission, workload-specific namespaces, ResourceQuota, LimitRange, NetworkPolicy and dedicated GPU node pools. Enforce non-root and restricted security contexts where compatible with the NVIDIA runtime. The NVIDIA GPU Operator and device plugin may require privileged platform-managed components; keep those permissions separate from tenant pods. Prevent unreviewed hostPath mounts, hostPID/hostNetwork, service account token automount and access to the container runtime socket.

Use admission controls to require image digest pinning, approved registries, identity bindings, GPU resource limits, namespace ownership and signed provenance. Validate whether the chosen CNI implements required egress restrictions; NetworkPolicy is not a universal cloud firewall.

## 6. Secure GPU data and storage

| Layer | AWS | Azure | Reliability/security concern |
|---|---|---|---|
| Object data | S3 | Blob/ADLS | encryption, prefix/container RBAC, versioning |
| Block volume | EBS | Managed Disks | encryption, snapshots, key lifecycle |
| Parallel filesystem | FSx for Lustre and supported options | Azure Managed Lustre / supported alternatives | POSIX access, throughput, network boundary |
| Scratch | local NVMe | local ephemeral disks | disposable; not durable checkpoint |
| Key management | KMS | Key Vault/managed HSM | authorization, availability, rotation |
| Audit | CloudTrail / S3 access logs where configured | Azure Activity/resource/data access logs where configured | coverage, retention, integrity |

Encryption at rest and TLS in transit are necessary but insufficient. Validate data classification, region residency, lifecycle/retention, backups, object versioning, restore permissions, integrity hashes, key deletion protection and disaster recovery. Data pipelines may require multiple identities: reader for source, writer for checkpoint, model publisher for registry.

## 7. Checkpoint and model artifact integrity

A durable checkpoint must be complete, attributable to the expected job, immutable or versioned where required, integrity-checked and restorable. Record artifact manifest with job ID, model revision, training step, object version/ETag or cryptographic checksum, encryption key reference, UTC timestamp, owner and retention policy. Validate restore in a separate controlled environment. A successful upload does not prove recovery. Never put plaintext access tokens in checkpoint metadata.

RPO is maximum tolerable data loss; RTO is maximum tolerable recovery time. Use tested restore evidence, not a backup existence check.

## 8. Network isolation and data exfiltration controls

Use private EKS/AKS API access where appropriate, private subnets, VPC/VNet endpoints, controlled egress, restricted security groups/NSGs, DNS controls, logging and data-access monitoring. Private endpoints can reduce exposure but do not replace workload IAM/RBAC. Restrict access to artifact registries and model repositories; allow necessary driver/package dependencies through approved mirrors. Verify that training jobs cannot use unauthorized cloud credentials from node metadata endpoints.

## 9. Secrets and key lifecycle

Prefer short-lived workload federation. Store unavoidable secrets in AWS Secrets Manager or Azure Key Vault, rotate with owner-approved processes, and limit Kubernetes Secret access. Never commit secrets, tokens, KMS plaintext material, kubeconfigs or service-account credentials to Git. Key rotation requires compatibility tests for previously encrypted datasets and checkpoints. Key revocation/deletion may make restored checkpoints unusable: include key availability in DR acceptance.

## 10. Supply chain and GPU runtime security

Pin container images by immutable digest, validate SBOM/signature/provenance, scan dependencies, maintain NVIDIA driver/CUDA/NCCL compatibility matrix, restrict privileged Operator DaemonSets, patch host OS/kernel and validate rollback. A GPU driver update may alter CUDA compatibility or device allocation behavior. Test signed images and runtime compatibility before promoting to production.

## 11. Observability and audit

| Signal | Alert or acceptance gate |
|---|---|
| anomalous workload identity token use | credential compromise |
| denied S3/Blob/KMS/Key Vault access | misconfiguration or attack |
| public storage/network policy drift | data exposure |
| privileged pod or hostPath admission | isolation regression |
| unsigned/unapproved image | supply-chain failure |
| unexpected checkpoint deletion | ransomware/retention incident |
| failed restore or key decrypt | DR failure |
| missing audit trail / stale telemetry | compliance blind spot |
| cross-tenant GPU process evidence | isolation breach |
| exfiltration egress anomaly | security incident |

Redact secrets, customer data and model contents from logs. Audit log access must be restricted, retained per policy and tamper-resistant where required.

## 12. Twelve troubleshooting scenarios

1. EKS pod receives AccessDenied despite node role permissions.
2. IRSA trust condition or Pod Identity association misconfigured.
3. AKS federated identity issuer/subject mismatch.
4. Azure RBAC management role exists but Blob data-plane access denied.
5. KMS/Key Vault decrypt denied after key policy change.
6. GPU pod unexpectedly accesses node metadata credentials.
7. NetworkPolicy allows traffic but cloud SG/NSG blocks storage.
8. Private endpoint DNS resolves incorrectly.
9. FSx/parallel filesystem permissions or mount options fail.
10. Checkpoint uploaded but integrity or restore validation fails.
11. GPU Operator privileged permissions leak to tenant namespace.
12. Signed image or model provenance verification fails.

## 13. Ten incident simulations

1. Compromised training pod attempts cross-tenant S3 access.
2. Azure workload identity federated credential accidentally broadened.
3. Public object storage exposure detected.
4. KMS/Key Vault key disabled before restore.
5. GPU node metadata token exfiltration attempt.
6. Malicious/unsigned training container rejected by admission.
7. Checkpoint deletion or corruption during incident.
8. Private endpoint outage blocks training datasets.
9. Privileged GPU runtime update breaks isolation.
10. Audit pipeline failure during a suspected model exfiltration event.

Run only in isolated approved staging or offline synthetic mode. Preserve forensic evidence and escalate according to incident severity.

## 14. Eight operational runbooks

**RB-040-A — Identity triage:** determine human/node/workload principal, token audience, trust, IAM/RBAC and explicit denies.

**RB-040-B — S3/Blob access:** inspect data-plane authorization, endpoint policy, encryption and object ownership.

**RB-040-C — KMS/Key Vault:** verify key state, key policy/RBAC, grant and restore path without exposing key material.

**RB-040-D — Network isolation:** validate DNS, endpoints, SG/NSG, CNI policy and egress.

**RB-040-E — GPU tenant containment:** isolate workload, preserve evidence, revoke identity and assess node compromise.

**RB-040-F — Checkpoint integrity:** verify version/hash, encryption, manifest, restore and RPO/RTO.

**RB-040-G — Supply chain:** quarantine unapproved image, verify signature/SBOM, pin digest and roll back.

**RB-040-H — Audit and RCA:** correlate cloud audit, Kubernetes events, GPU telemetry, owner approvals and COA.

## 15. Hands-on labs

1. Map AWS EKS node IAM role versus workload identity and scoped S3/KMS permissions.
2. Map AKS managed identity, Workload ID federation and Blob/Key Vault data-plane roles.
3. Review secure storage design: encryption, versioning, private endpoints and restore.
4. Validate GPU namespace admission, quota, RBAC, security context and egress controls.
5. Simulate stolen workload identity, disabled key and public bucket drift **offline**.
6. Test checkpoint checksum and restore using disposable synthetic data in staging.
7. Verify image provenance, audit trail and approved break-glass procedures.
8. Produce threat model, redacted evidence, SLO/RPO/RTO and security owner signoff.

[Lab 040 — Cloud GPU Identity, Storage and Security Gate](../labs/040-cloud-gpu-identity-storage-security-check.md) contains a Python 3.11+ fail-closed evaluator with pytest tests. It does not query AWS/Azure or enforce production policy.

## 16. 100-point assessment

| Domain | Points |
|---|---:|
| AWS/Azure workload identity and authorization | 20 |
| Storage, encryption, keys and checkpoints | 20 |
| GPU tenant isolation, networking and supply chain | 20 |
| Incident response, audit and recovery | 20 |
| Executable tests, staging evidence and security review | 20 |
| **Total** | **100** |

Pass >=90 and **zero critical** identity, data exposure, key, tenant isolation, checkpoint or audit blocker.

## Official references

- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [EKS IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [AWS S3 security best practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [AWS KMS key policies](https://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html)
- [AKS Workload Identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [Azure Blob authorization](https://learn.microsoft.com/en-us/azure/storage/blobs/authorize-access-azure-active-directory)
- [Azure Key Vault security](https://learn.microsoft.com/en-us/azure/key-vault/general/security-features)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [NVIDIA GPU Operator security](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)

**Next: Chapter 041 — Multi-Cloud GPU Capacity and Cost.**
