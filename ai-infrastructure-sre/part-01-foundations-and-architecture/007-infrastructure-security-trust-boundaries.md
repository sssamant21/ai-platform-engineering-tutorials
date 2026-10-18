# Chapter 007 — Infrastructure Security and Trust Boundaries

**Part 01 — Foundations and Architecture · Chapter 007 of 100**  
**Status:** Published. Offline lab execution, live security validation, independent review and production approval pending.

## Learning objectives
1. Identify AI infrastructure trust boundaries from user ingress through GPU hosts, Kubernetes, model registry, data storage and distributed training fabric.
2. Apply workload identity, least privilege, image and model provenance, secrets management, tenant isolation and network segmentation.
3. Build a threat model and enforce security gates before GPU workload admission.
4. Investigate GPU platform security incidents without destroying evidence or violating change controls.
5. Execute a fail-closed Python policy lab with automated tests.

## 1. AI infrastructure security is a cross-layer responsibility
AI infrastructure handles customer prompts, training data, embeddings, model weights, checkpoints, adapters and credentials. Security boundaries exist not only between tenants but also between the control plane and GPU data plane, between workloads and host drivers, and between a model-serving service and external retrieval/tool endpoints.

```text
 external user / partner
          |
    [Boundary 1: ingress]
          |
 WAF / authn / gateway / quota
          |
    [Boundary 2: tenant identity]
          |
 API / inference / training scheduler
          |
    [Boundary 3: cluster RBAC]
          |
 Kubernetes API / admission / policy
          |
    [Boundary 4: GPU host]
          |
 container runtime / driver / GPU
          |
    [Boundary 5: data access]
          |
 model registry / datasets / checkpoints
          |
    [Boundary 6: network egress]
          |
 retrieval APIs / tool endpoints / SaaS
```

The security architecture must consider compromised credentials, malicious images, model supply-chain tampering, prompt-data exposure, container escape, host kernel/driver vulnerabilities, cross-tenant GPU residue, unauthorized checkpoint reads and uncontrolled external network access.

## 2. Threat model and security controls
| Asset / boundary | Threat | Preventive control | Detection / evidence |
|---|---|---|---|
| API ingress | unauthorized requests, abusive load | identity, rate limits, WAF, tenant quotas | access audit, anomalous request rate |
| Kubernetes API | privilege escalation | least-privilege RBAC, policy admission | audit logs, denied requests |
| Container images | vulnerable or tampered runtime | immutable digests, scanning, signatures | provenance attestations |
| Model weights/adapters | artifact poisoning or replacement | verified digests, signed promotion | registry audit, digest mismatch |
| GPU hosts | driver exploit, privileged container | patched kernel/driver, restricted host access | EDR, node audit, Xid/driver logs |
| Shared GPU | cross-tenant leakage or interference | validated isolation, dedicated pools as needed | policy and placement evidence |
| Dataset and checkpoints | exfiltration, unauthorized modification | scoped identity, encryption, integrity manifests | object access and checksum audit |
| RDMA/network | unauthorized fabric access, lateral movement | segmentation and fabric ACLs | flow logs and fabric counters |
| Secrets/KMS | leaked token, overly broad decrypt | workload identity, rotation, scoped KMS grants | IAM/KMS audit |
| RAG/tools | unauthorized retrieval or action | per-tenant ACL, tool authorization | retrieval/tool access logs |

Treat model artifacts and checkpoints as sensitive assets. Encryption at rest is not sufficient if workload identity can read the wrong tenant's objects.

## 3. Identity, RBAC and secrets
Use short-lived workload identity where supported (AWS EKS Pod Identity/IRSA, Azure AKS workload identity) instead of long-lived static cloud keys. Scope access by namespace, service account, environment and resource path. Separate read-only model loading from training artifact write permissions and from infrastructure administration.

**Kubernetes example — dedicated service account and restricted workload:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ai-inference
  namespace: ai-inference-staging
automountServiceAccountToken: false
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-inference
  namespace: ai-inference-staging
spec:
  replicas: 2
  selector:
    matchLabels: {app: ai-inference}
  template:
    metadata:
      labels: {app: ai-inference}
    spec:
      serviceAccountName: ai-inference
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        seccompProfile: {type: RuntimeDefault}
      containers:
        - name: inference
          image: registry.example.invalid/ai/inference@sha256:REPLACE_WITH_REAL_64_HEX_DIGEST
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: {drop: ["ALL"]}
          resources:
            requests: {cpu: "1", memory: "4Gi", "nvidia.com/gpu": "1"}
            limits: {cpu: "2", memory: "8Gi", "nvidia.com/gpu": "1"}
```
**Illustrative only:** the image digest placeholder is intentionally invalid. The image, writable scratch mounts, probes, network policy, cloud identity integration, supported runtime, GPU resource key and deployment namespace must be validated before use. Disabling service account token automount is appropriate only when the workload does not need Kubernetes API credentials; cloud workload identity may require projected tokens configured separately.

## 4. Kubernetes and GPU host isolation
- Use namespaces, dedicated service accounts, RBAC, admission policy, ResourceQuota and LimitRange.
- Apply default-deny ingress/egress NetworkPolicy with only approved DNS, telemetry, model registry and storage flows; ensure the CNI enforces policies.
- Require Pod Security Standards appropriate to the runtime; document exceptions for GPU Operator or driver components, never grant broad privileges to application workloads.
- Separate host administration, device plugin/operator privileges and user GPU pods. Restrict hostPath, hostPID, hostNetwork and privileged containers.
- Verify GPU device isolation on the actual GPU, driver, container runtime and virtualization/MIG configuration. MIG is hardware-dependent and is not a universal security boundary.
- Keep host kernel, NVIDIA driver, CUDA and operator compatibility matrices; stage upgrades with rollback and signed artifacts.

## 5. Data, model and checkpoint boundaries
Model and dataset lifecycle:
```text
approved source -> malware/license/security scan -> digest
       -> signed artifact registry -> promotion approval
       -> workload identity scoped download -> runtime
       -> audited output/checkpoint -> retention/deletion policy
```
Pin immutable image and model revisions. Separate dev/staging/prod registries and keys. Validate checkpoint integrity and restore ownership. Enforce region/data residency before scheduling. Redact prompts, PHI/PII, tokens, signed URLs and model secrets from logs, traces and tickets.

## 6. Distributed training and network security
Multi-node training may use high-performance networking and collectives that do not fit ordinary east-west traffic patterns. Document every allowed rank-to-rank flow, host identity, fabric partition, encryption limitations and vendor requirements. A permissive `0.0.0.0/0` rule is not a substitute for a tested NCCL/RDMA policy. Limit job submission rights and checkpoint/object-store permissions independently.

## 7. [SAFE-READ] Security posture discovery
Use only approved cluster contexts and do not publish raw metadata:
```bash
kubectl config current-context
kubectl get namespaces --show-labels
kubectl get serviceaccounts -A
kubectl get roles,rolebindings,clusterroles,clusterrolebindings -A
kubectl get networkpolicy -A
kubectl get resourcequota -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl auth can-i --list -n ai-inference-staging
```
The `kubectl auth can-i --list` output describes the current identity's effective permissions for the namespace and may not include every external authorization layer. A read-only command can still expose sensitive security metadata. Do not use `kubectl get secrets -o yaml` in routine diagnostics.

## 8. Threat and incident simulations
**Scenario A — Compromised service account:** A workload can list other namespaces. Freeze unsafe rollout, preserve audit evidence, revoke/rotate compromised credentials through approved security response, correct RBAC and verify tenant isolation.

**Scenario B — Unsigned model artifact:** A model digest differs from the approved release. Quarantine artifact, block promotion, compare signed provenance and audit registry access.

**Scenario C — Privileged GPU pod:** Application deployment requests hostPath or privileged mode. Reject via admission policy; investigate the requesting identity. Distinguish vendor-managed GPU operator exceptions from application permissions.

**Scenario D — Checkpoint data exfiltration:** Unexpected object-store reads from another tenant. Contain identity, preserve logs, verify scope and invoke privacy/security incident process.

**Scenario E — Egress to unauthorized endpoint:** Model-serving pod attempts external tool call. Verify default-deny network enforcement, DNS and egress gateway; investigate application tool policy.

**Scenario F — GPU driver vulnerability:** Vendor advisory affects installed driver. Inventory affected hosts, assess exploitability and coordinate maintenance windows, canary upgrades, rollback and attestation.

## 9. Troubleshooting matrix
| Signal | Suspected boundary | Evidence | Owner |
|---|---|---|---|
| forbidden Kubernetes API call | RBAC / identity | audit and `auth can-i` | Kubernetes/security |
| model digest mismatch | artifact supply chain | registry manifest, signatures | ML platform/security |
| unexpected cross-tenant access | storage/IAM | object access logs, role scope | security/DBRE |
| privileged pod rejected | admission | admission policy event | Kubernetes platform |
| model download 403 | cloud workload identity | token audience, role policy | cloud IAM |
| GPU workload cannot reach NCCL peer | network/fabric policy | flow and rank logs | network/HPC |
| sensitive data in traces | observability boundary | trace samples, exporter config | security/observability |

## 10. Production runbooks
**RB-007-A — Workload security admission:** verify named owner, approved data classification/residency, identity, namespace, policy, signed artifacts, quotas, network restrictions and recovery plan.

**RB-007-B — Identity compromise:** declare security incident, preserve audit and timeline, contain credentials, assess affected tenants, rotate/revoke under approved procedures and verify access after recovery.

**RB-007-C — Supply-chain anomaly:** quarantine artifact, stop promotion, verify image/model digests and signatures, restore approved revision and document provenance.

**RB-007-D — GPU host hardening:** verify driver/kernel matrix, host patch level, runtime policy and privileged component inventory; roll through staging before production.

**RB-007-E — Data exposure:** stop unsafe paths, preserve access logs, coordinate security/privacy/legal review and restore only after authorization and tenant checks.

**RB-007-F — Post-incident security acceptance:** confirm containment, restore, evidence chain, least-privilege tests, audit completeness, RCA/COA and accountable closure.

## 11. 100-point assessment
| Domain | Points |
|---|---:|
| Threat model and trust boundaries | 20 |
| Workload identity and Kubernetes isolation | 20 |
| Artifact, data and checkpoint protection | 20 |
| Incident handling and security runbooks | 20 |
| Executable lab and validation evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical trust-boundary or data-integrity violation. Synthetic policy tests do not prove real platform security.

## 12. Companion lab
[Chapter 007 — Trust Boundary and Admission Policy Lab](../labs/007-infrastructure-security-trust-boundaries-check.md) includes a fail-closed Python 3.11+ validator and pytest regression suite.

## References
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [SLSA Supply-chain Security](https://slsa.dev/)

**Next:** Chapter 008 — Foundations Architecture Review Lab.
