# Chapter 10.4 — Hands-on Lab: Secrets Management, Encryption and Credential Rotation

**Part 10 | Disposable staging only | Formal execution pending**

## Mission and guardrails

Validate secret lifecycle, cloud identity, ESO/CSI reconciliation, application reload, TLS renewal, rotation, revocation and rollback. Use **synthetic, non-sensitive credentials only**. **[SAFE-READ]** metadata/health inspection; **[LAB-MUTATE]** staging-owned objects; **[CHANGE-CONTROL]** production approval. Never print or share Secret values, projected tokens, environment dumps, private keys or unredacted provider logs.

## Lab 0 — Inventory and architecture [SAFE-READ]

```bash
kubectl config current-context
kubectl get ns
kubectl get externalsecret,secretstore,clustersecretstore -A 2>/dev/null || true
kubectl get secretproviderclass -A 2>/dev/null || true
kubectl get certificates -A 2>/dev/null || true
kubectl get secrets -A -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name,TYPE:.type'
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,SA:.spec.serviceAccountName'
```

Document installed provider, cloud workload identity, namespace, Secret owner, encryption-at-rest posture, controller/CSI version, rotation interval, consumer reload strategy and security reviewer. If ESO/CSI is not installed, perform a tabletop rather than deploying cluster-wide controllers without approval.

## Lab 1 — Source-of-truth and access review [SAFE-READ]

Trace synthetic secret manager entry → workload identity → ESO/CSI → Kubernetes Secret or mounted file → application. Record storage copies, RBAC, cloud IAM, TLS, logs, backups and expiration. Verify allowed and denied identities through approved provider audit; never retrieve values for evidence.

## Lab 2 — Staging secret synchronization [LAB-MUTATE]

In a disposable namespace and approved synthetic secret store, create an ExternalSecret or SecretProviderClass according to the installed controller/version. Confirm Ready/sync condition, destination metadata, file presence without printing contents and consumer health. Record refresh and ownership/deletion policy.

## Lab 3 — Credential rotation [LAB-MUTATE]

Use a synthetic credential supporting controlled overlap. Establish baseline; publish new version; wait for sync; reload application or roll Pods safely; verify new authentication; retire old version; verify revoked credential fails. If the provider does not support overlap, rehearse a maintenance-window rotation instead.

## Lab 4 — TLS certificate rotation [LAB-MUTATE]

Use a lab-only hostname and certificate issuer. Check live certificate SAN, chain and expiry before and after renewal. Test an invalid lab certificate reference only in an isolated route; restore valid certificate without disabling TLS verification.

## Lab 5 — Faults, incident response and rollback

Rehearse provider 403, missing store, stale sync, mounted file not reloaded, database auth failures, KMS denial, cert expiry and leaked synthetic credential. Capture first failure, alert, safe recovery and rollback. **Compromised credentials must not be restored as rollback.**

## Lab 6 — Cleanup and evidence

Remove only lab-owned Secret references, ExternalSecrets, SecretProviderClasses, credentials and test workloads. Confirm no stale Kubernetes Secret copy remains, no cloud identity privilege drift and no secret values in logs or GitOps commits.

## Thirty failure scenarios

| # | Scenario | Diagnostic focus |
|---|---|---|
| 01 | ESO CRD missing | Installed controller |
| 02 | ExternalSecret not Ready | Status/conditions |
| 03 | SecretStore wrong namespace | Store reference |
| 04 | ClusterSecretStore unauthorized | Scope and policy |
| 05 | Cloud IAM role denied | Workload identity |
| 06 | IRSA/Pod Identity association wrong | EKS trust |
| 07 | AKS federation subject wrong | Entra trust |
| 08 | Provider private endpoint unreachable | DNS/network |
| 09 | Provider returns 429 | Rate limiting |
| 10 | Provider returns 5xx | Availability/retry |
| 11 | External secret key/path wrong | Source reference |
| 12 | Secret version not found | Version stage |
| 13 | KMS Decrypt denied | Key policy |
| 14 | ESO refresh too slow | Reconcile interval |
| 15 | Target Secret deleted | Ownership/recovery |
| 16 | Target Secret unexpectedly shared | RBAC boundary |
| 17 | CSI driver missing | Node plugin |
| 18 | CSI provider unavailable | DaemonSet/driver |
| 19 | SecretProviderClass invalid | Mount config |
| 20 | Mounted secret stale | Rotation/reload |
| 21 | Environment variable not refreshed | Pod restart |
| 22 | Connection pool caches old credential | Consumer behavior |
| 23 | Rotation revokes old credential too early | Overlap plan |
| 24 | Revoked credential still accepted | Cache/session expiry |
| 25 | Certificate expired | Live listener |
| 26 | TLS SAN/SNI mismatch | Certificate identity |
| 27 | CA trust rollover fails | Trust bundle order |
| 28 | Secret exposed in pipeline logs | Redaction/incident |
| 29 | Backup exposes Kubernetes Secret | Backup encryption/RBAC |
| 30 | GitOps restores stale secret reference | Desired-state rollback |

For each scenario: scope, timestamp, synthetic principal, symptom, sanitized audit ID, diagnosis, safe remediation, rollback and reviewer. Perform only staging-safe faults; otherwise record tabletop.

## Ten acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved staging scope and secret inventory | Pending |
| G2 | Source-of-truth and encryption boundaries documented | Pending |
| G3 | Scoped IAM/RBAC and denied-access checks | Pending |
| G4 | Installed ESO/CSI integration and sync validated | Pending |
| G5 | Consumer reload vs restart behavior verified | Pending |
| G6 | Credential rotation and revocation proven | Pending |
| G7 | TLS certificate identity and renewal validated | Pending |
| G8 | Alerts for expiry, sync and provider errors validated | Pending |
| G9 | GitOps rollback, incident response and cleanup proven | Pending |
| G10 | Security/SRE review, exceptions and sign-off | Pending |

**Pass:** each applicable gate has timestamped evidence and owner sign-off. Unexecuted or blocked gates are not passed.

## Evidence worksheet

UTC timestamp, cluster/context, namespace, store/controller version, secret *reference* (not value), identity type, expected/actual status, sync age, application reload, certificate metadata, sanitized audit ID, rollback commit, reviewer.

**Next:** Chapter 10.5 — Cloud IAM, Supply Chain and Runtime Security.
