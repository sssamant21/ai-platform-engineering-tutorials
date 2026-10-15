# Chapter 10.4 — Secrets Management, Encryption and Credential Rotation

**Part 10 — Security | Canonical SRE/Platform Engineering Tutorial**

## 1. Objectives, prerequisites and safety

Implement and operate secrets management for TrueFoundry applications, inference workloads, cluster agents and CI/CD. Compare Kubernetes Secrets, external secret stores, workload identity, CSI-mounted secrets, TLS certificates and encryption controls; design rotation without unnecessary downtime; diagnose sync, access, expiry and recovery failures.

**Safety contract:** [SAFE-READ] inspection without reading secret values; [LAB-MUTATE] isolated staging with synthetic credentials; [CHANGE-CONTROL] approved production changes, monitoring and rollback. Verify installed TrueFoundry integrations, External Secrets Operator (ESO), Secrets Store CSI Driver, cloud provider and Kubernetes version. Never assume a component is installed or a vendor integration exists. Never publish passwords, tokens, private keys, full secret YAML or unredacted environment dumps.

## 2. Secret lifecycle and trust boundaries

```text
Human / CI → approved secret manager (AWS Secrets Manager, Azure Key Vault, Vault, etc.)
                           ↓ workload identity / scoped cloud IAM
                    ESO or Secrets Store CSI Driver
                           ↓ sync Kubernetes Secret OR mount projected file
TrueFoundry workload spec → Kubernetes Pod → application / inference server
                                      ↘ TLS client / database / model registry
Kubernetes API ↔ etcd encryption-at-rest provider and access control
```

**Secret source of truth:** establish whether the secret manager, Kubernetes API or another platform owns the value. A synced Kubernetes Secret creates a second copy subject to Kubernetes RBAC, etcd encryption, backups and audit. CSI mount-only mode may avoid a persisted Kubernetes Secret but requires application file reload behavior. Distinguish encryption in transit, at rest, envelope encryption and runtime memory exposure.

## 3. Inventory and safe discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl get ns
kubectl get externalsecret,secretstore,clustersecretstore -A 2>/dev/null || true
kubectl get secretproviderclass -A 2>/dev/null || true
kubectl get certificates -A 2>/dev/null || true
kubectl get secrets -A -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name,TYPE:.type,AGE:.metadata.creationTimestamp'
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,SA:.spec.serviceAccountName'
```

Listing Secret **metadata** still reveals names and architecture: limit scope and redact when sharing. Do not use `kubectl get secret -o yaml`, `kubectl describe secret` where it exposes sensitive metadata, `kubectl exec ... env` or verbose CI logs. Check actual CRD discovery before assuming ESO/CSI exists.

## 4. Kubernetes Secrets and encryption

Kubernetes Secret values are base64-encoded in manifests, **not encrypted by base64**. Kubernetes API RBAC governs read/write access, and Pods with access to mounted Secrets can expose values. Encryption at rest for etcd depends on control-plane configuration; managed EKS/AKS provider capabilities and customer-managed key options vary. Confirm actual control-plane settings rather than assuming a KMS provider is enabled.

- Scope Secret reads and creation to the minimum namespaces and identities.
- Prefer immutable, versioned deployment references when supported; avoid mutable credentials with unclear refresh behavior.
- Use volume mounts rather than environment variables when application reload is needed; env-based Secret values generally require Pod restart to update.
- Set restrictive file permissions where possible, avoid unnecessary token automount and avoid copying secret values into ConfigMaps.
- Backups/snapshots containing Secrets require independent encryption, access control, retention and restore tests.
- Avoid hardcoding secrets in Helm values, Terraform state, GitOps manifests and pipeline artifacts.

## 5. External Secrets Operator and CSI patterns

**ESO:** an `ExternalSecret` refers to a `SecretStore` or `ClusterSecretStore`; the controller authenticates to the external provider and creates/updates a Kubernetes Secret. Confirm refresh interval, provider permissions, version selection, target ownership/deletion behavior, status conditions and reconciliation logs. A ClusterSecretStore may expand the trust boundary; constrain which namespaces and principals can reference it according to the installed implementation.

**Secrets Store CSI Driver:** a Pod mounts external values as files via `SecretProviderClass`. The provider plugin, workload identity, driver scheduling and rotation feature must all be present. Optional synchronization to Kubernetes Secrets changes storage exposure. Rotated files do **not** guarantee that the application reloads the value.

**TrueFoundry:** treat platform-level secret references, environment injection and deployment configuration as version-specific; verify the actual platform feature and whether the resulting value becomes a Kubernetes Secret, Pod env var or mounted file.

## 6. Cloud identity and permission design

**AWS EKS:** use IRSA or EKS Pod Identity as actually configured; grant `secretsmanager:GetSecretValue` only for approved ARNs, and KMS `Decrypt` only when required. Restrict trust to the intended namespace/ServiceAccount or Pod Identity association. Account for VPC endpoints, endpoint policies, DNS and security groups.

**Azure AKS:** validate AKS Workload Identity and Entra federated credential subject/audience; grant minimum Key Vault data-plane role/permissions, confirm firewall/private endpoint and DNS. Azure Key Vault CSI provider and ESO have different operational behavior.

**HashiCorp Vault or other manager:** review auth method, policy paths, TTL/leases, revocation, renewals and audit devices. Dynamic database credentials require application connection pool refresh and controlled overlap.

A network connection to the secret manager does not imply authorization. A successful cloud token exchange does not imply permission to read a particular secret.

## 7. TLS certificates, keys and rotation

TLS private keys are secrets. For cert-manager or cloud-managed certificates, inventory issuer, SAN, renewal window, ownership and expiry alerts. Validate chain and SNI on the **live listener**, not merely the Kubernetes Certificate resource. Rotate leaf certificates and signing trust with appropriate overlap; for mTLS or CA changes, stage trust-bundle distribution before switching signing identity. Do not disable TLS validation to resolve an expiry incident.

## 8. Zero/low-downtime credential rotation

1. **Inventory consumers:** Pods, CI jobs, agents, database connections, SDKs, TLS listeners and cached credentials.
2. **Classify rotation support:** dual active credentials, version stages, dynamic leases, hot reload or restart-only.
3. **Baseline:** auth failures, readiness, request success, queue/connection state and secret sync age.
4. **Stage:** create a new synthetic/staging credential with the same minimal permissions; do not revoke the old one yet if overlap is supported.
5. **Distribute:** update manager version/reference; verify ESO/CSI sync and application reload or controlled rollout.
6. **Verify:** new connections succeed, old credentials are no longer used, denied actions remain denied, no plaintext appears in logs.
7. **Revoke:** retire old credential only after all consumers have transitioned and rollback has been evaluated.
8. **Rollback:** use approved version rollback or reissue a credential; **do not restore a known-compromised secret** as rollback.
9. **Audit:** record rotation time, owner, versions (not values), impacted consumers and incident/change ID.

**Critical nuance:** revocation of a compromised credential may need to happen immediately, even if it causes service disruption. Security containment takes precedence over a seamless overlap.

## 9. Failure modes and observability

Measure secret sync age, controller reconciliation errors, provider 401/403/429/5xx, lease renewal failure, token expiration, certificate expiry, mounted file age, app authentication errors, restart frequency and failed connection pools. Avoid secret names and metadata in broad telemetry if they are sensitive. Alert on unexpected Secret reads, wildcard RBAC, deleted stores, stale synced values and failed rotations.

## 10. Twelve production troubleshooting runbooks

### RB-01 — ESO ExternalSecret not Ready
Inspect CRD, controller health, store readiness, conditions, provider reachability and refresh. Repair only the failing layer; do not dump the target Secret.

### RB-02 — Cloud secret access denied
Validate workload identity, IAM/Key Vault data-plane permissions, provider resource scope and network/firewall. Avoid wildcard admin grants.

### RB-03 — CSI volume mount failure
Check SecretProviderClass, driver/provider DaemonSet, node scheduling, token projection and Pod events. Restore a healthy provider and restart only affected staging workloads if needed.

### RB-04 — Secret updated but app still uses old value
Differentiate environment-variable snapshot from mounted-file reload and connection pool caching. Perform a controlled rollout or reload supported by the application.

### RB-05 — Rotation breaks database connectivity
Check dual-user/dual-password overlap, connection pool lifetime, authentication errors and server-side active sessions. Recover using a newly authorized credential, not an exposed one.

### RB-06 — Secret manager throttling/outage
Inspect 429/5xx, retry jitter, cache policy, stale-value risk and dependency SLO. Avoid unbounded retries and accidental credential disclosure.

### RB-07 — KMS decrypt failure
Check key state, key policy, role grants, encryption context and region. Reauthorize narrowly and preserve audit.

### RB-08 — TLS certificate expired or wrong SAN
Validate live listener, SNI, cert chain and renewal logs. Rebind a valid approved certificate; never turn off verification.

### RB-09 — CA or mTLS trust rollover fails
Check client/server trust bundles, rollout order and intermediate chain. Restore compatible trust through approved change.

### RB-10 — Kubernetes Secret deleted unexpectedly
Identify GitOps/ESO ownership and deletion policy, audit events and consumer impact. Restore from authoritative manager without exposing values.

### RB-11 — Secret leaked to logs/Git/artifact
Contain exposure, revoke/rotate, inspect access history, preserve evidence and remove copies under incident policy. Rewriting Git history alone is insufficient.

### RB-12 — GitOps drift reintroduces old credential reference
Inspect desired-state source, operator ownership and reconciliation. Revert bad Git commit and verify the effective version, consumers and audit.

## 11. Production acceptance

Required artifacts: secret inventory and ownership, trust-boundary diagram, encryption evidence, scoped cloud IAM, sync/reload behavior, rotation runbook, expiry alert, denied-access tests, emergency revocation path, rollback and reviewer sign-off. Do not claim zero downtime until tested with the actual consumer.

Complete the [companion hands-on lab](../labs/04-secrets-management-encryption-credential-rotation-check.md) with **30 failure scenarios and 10 acceptance gates**. Publication does not imply execution, vendor review or production approval.

**Next:** Chapter 10.5 — Cloud IAM, Supply Chain and Runtime Security.
