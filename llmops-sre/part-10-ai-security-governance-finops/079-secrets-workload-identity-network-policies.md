# Chapter 079 — Secrets, Workload Identity and Network Policies

**Part 10 — AI Security, Governance and FinOps · Chapter 079 of 100**  
**Status:** Chapter and lab published. Live staging execution and independent technical/security review pending.

## Learning objectives
Design and operate secrets lifecycle management, short-lived workload identity, least-privilege service access, Kubernetes NetworkPolicy and egress controls for enterprise LLM inference. Demonstrate fail-closed authorization and incident recovery without embedding secrets in prompts, images, Git, logs or model artifacts.

## 1. Reference architecture
```text
external client -> TLS ingress -> API gateway -> inference namespace
                        |              |          |
                  OIDC validation      |       GPU workers
                                       |          |
                            workload identity / projected token
                                       |
                        trusted STS / secrets manager
                                       |
                       scoped vector DB / object store / tool broker
                                       |
                    default-deny network + audited egress paths
```
Boundaries include external ingress, pod service-account identity, token exchange, secret retrieval, east-west network traffic, egress to registries/object stores, and audit exporters. A Kubernetes service account name alone does not prove cloud IAM authorization; the cluster OIDC trust configuration, subject/audience binding and cloud-side role policy must also be validated.

## 2. Secrets lifecycle
- **Inventory:** API keys, database passwords, encryption keys, service tokens, registry credentials, TLS certificates and webhook secrets.
- **Storage:** approved secret manager or encrypted Kubernetes Secret with etcd encryption-at-rest, RBAC, admission controls and restricted backups. Base64 in a Secret manifest is **not encryption**.
- **Delivery:** prefer workload identity and short-lived credentials; use a supported CSI/sidecar integration where necessary. Avoid injecting long-lived secrets into environment variables when alternatives exist.
- **Rotation:** define ownership, expiry, overlap period, reload semantics, verification and revocation. Test zero-downtime rotation in staging.
- **Revocation:** revoke compromised credentials, inspect access logs and rotate dependent credentials; ensure workloads do not continue using cached revoked tokens indefinitely.
- **Audit:** record access decision and resource identifier, never secret values.

## 3. Workload identity
Use Kubernetes projected service-account tokens with bounded lifetime and intended audience, plus a provider-supported federation mechanism. Verify service-account subject, namespace, audience, issuer and cloud IAM trust conditions. For AWS EKS use approved IRSA or EKS Pod Identity design; for Azure AKS use Microsoft Entra Workload ID; for GKE use Workload Identity Federation. The mechanisms differ: do not copy provider-specific annotations blindly.

Recommended controls:
- one service account per trust role, `automountServiceAccountToken: false` unless required;
- no static cloud keys in container images, Git or prompts;
- short token lifetime, least-privilege roles and scoped resource permissions;
- policy-bound tool credentials separate from inference-serving credentials;
- regular review of federation trust and permissions.

## 4. Network isolation
Kubernetes NetworkPolicy is additive and applies only if the cluster CNI enforces it. Default-deny ingress and egress within inference namespaces, then permit narrowly scoped paths: gateway to inference service, inference to DNS, approved model store/vector database, token-exchange endpoint, metrics exporter and required control-plane services. Standard NetworkPolicy does not provide portable FQDN allowlisting; use an approved egress proxy/firewall or CNI-specific extension. Validate DNS, model download, image pull and health checks after policy rollout.

### Example default-deny policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: llm-inference
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```
Apply only in a disposable staging namespace after verifying an enforcement-capable CNI. This policy blocks selected pod ingress/egress unless other policies allow it. It does not guarantee host-network or node-level isolation.

### Example ingress allowlist
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-gateway
  namespace: llm-inference
spec:
  podSelector:
    matchLabels: {app: inference}
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels: {kubernetes.io/metadata.name: llm-gateway}
          podSelector:
            matchLabels: {app: gateway}
      ports:
        - {protocol: TCP, port: 8000}
```
The gateway namespace label and pod labels must be validated. This policy permits only the described ingress **in addition to** any other policies that may apply.

## 5. Secure deployment checklist
1. Verify cluster CNI supports NetworkPolicy enforcement and the test namespace is isolated.
2. Pin image and model digests, run as non-root, use read-only filesystem where supported.
3. Disable unnecessary service-account token mounts and privilege escalation.
4. Bind only the required workload identity to the inference service account.
5. Limit secret-manager read permissions to exact paths/resources.
6. Apply default-deny, then required DNS, gateway, artifact and telemetry flows.
7. Verify denied cross-namespace access and unapproved egress from test pods.
8. Exercise token expiry/refresh and secret rotation without logging credentials.
9. Confirm rollbacks restore only the intended network access, not blanket allow-all.
10. Archive sanitized audit evidence, ownership and change approval.

## 6. Failure scenarios
| Symptom | Diagnose | Safe response |
|---|---|---|
| 401 from secrets manager | issuer/audience/subject, token expiry | repair trust binding, never bypass auth |
| model artifact download fails | egress proxy, DNS, object-store allowlist | restore approved network path |
| DNS failures after default-deny | DNS egress and CNI behavior | add minimal UDP/TCP 53 access |
| old secret still used | sidecar sync/reload and token cache | force controlled reload, revoke old value |
| unauthorized egress succeeds | CNI enforcement, hostNetwork, other policies | investigate policy bypass |
| secret appears in log | app/logger/exporter | stop export, rotate, incident process |
| pod receives unexpected token | automount setting and projected volumes | restrict service account and redeploy |

## 7. Production runbooks

### RB-079-A — Suspected credential compromise
Contain affected identity, revoke token/key, preserve restricted audit evidence, rotate dependent secrets and assess impact. Never paste secret values into tickets.

### RB-079-B — Workload identity outage
Check projected token audience/expiry, issuer discovery, federation trust, IAM role policy, DNS and clock skew. Fail closed if identity cannot be verified. Restore and test least-privilege access.

### RB-079-C — NetworkPolicy outage
Inspect CNI enforcement, effective additive policies, DNS, gateway selectors and egress routes. Roll back only the offending policy through change control, not to unrestricted networking.

### RB-079-D — Rotation failure
Use approved overlap/rollback window, verify application reload behavior, invalidate stale credentials and confirm success with sanitized metrics.

## 8. Security acceptance (100 points)
| Domain | Points |
|---|---:|
| Secrets inventory, storage, rotation and revocation | 20 |
| Federated workload identity and least privilege | 20 |
| Ingress/egress NetworkPolicy and enforcement tests | 20 |
| Observability, incident handling and recovery | 20 |
| Automated lab, audit and documented residual risk | 20 |
| **Total** | **100** |

Any real secret leak, unauthorized resource access or uncontrolled egress is a release blocker. Publication does not imply live validation or security approval.

## 9. Executable lab
[Chapter 079 — Secrets, Workload Identity and Network Policies Lab](../labs/079-secrets-workload-identity-network-policies-check.md) contains a deterministic offline policy evaluator and optional staging-only manifests.

## References
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes Service Accounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [NIST SP 800-57](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)

**Next:** Chapter 080 — Model Artifact Supply Chain, SBOM and Provenance.
