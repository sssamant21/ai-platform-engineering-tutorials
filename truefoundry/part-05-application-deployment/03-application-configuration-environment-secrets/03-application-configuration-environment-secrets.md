# 5.3 — Application Configuration, Environment Variables and Secrets

**Part:** 05 — Application Deployment · **Chapter:** 3 of 7  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, DevSecOps and application teams  
**Safety:** Discovery commands are **[SAFE-READ]**. Secret creation, rotation, identity policy, admission and deployment changes are **[CHANGE-CONTROL]**; lab mutations are restricted to an approved disposable namespace.

## 1. Learning objectives

Design reliable, environment-specific application configuration; separate ordinary configuration from credentials; understand Kubernetes ConfigMaps, Secrets, external secret controllers and cloud secret stores; verify TrueFoundry workload identity and configuration delivery; and troubleshoot configuration failures without exposing sensitive values.

## 2. Configuration architecture and ownership

```text
Approved Git configuration (non-secret)
        │
        ├── TrueFoundry deployment configuration / GitOps source
        │         │
        │         └── Kubernetes ConfigMap / Pod environment / volume
        │
Cloud secret manager (AWS Secrets Manager / Azure Key Vault / GCP Secret Manager)
        │
        ├── external-secrets controller or supported platform integration
        │         │
        │         └── Kubernetes Secret (when synchronization is configured)
        │
        └── workload identity / direct application API retrieval (alternative)
                  │
                  ▼
         Application Pod / mounted file / environment
                  │
                  ▼
          Startup → readiness → dependencies
```

The diagram shows **alternative** delivery patterns, not a requirement to synchronize every cloud secret into Kubernetes. TrueFoundry capabilities and generated resources depend on the installed version. Verify the actual workload configuration and authoritative source rather than assuming a particular UI field, SDK schema or Kubernetes controller.

## 3. Choose the right configuration mechanism

| Mechanism | Typical use | Behavior and caution |
|---|---|---|
| Container image | Application code and safe static defaults | Do not bake credentials or environment-specific secrets into layers |
| ConfigMap | Non-sensitive configuration | Base64 is not involved; content is not confidential |
| Kubernetes Secret | Credentials, tokens, certificates | Base64 encoding is **not encryption**; enable encryption at rest and strict RBAC |
| Pod environment variables | Small values consumed at startup | Env changes do not automatically update a running process |
| ConfigMap/Secret projected volume | File-based configuration | Kubernetes can update projected data eventually; app must reload; `subPath` mounts do not receive normal updates |
| External Secrets Operator (ESO) | Sync external secrets into Kubernetes | Reconciliation and permissions depend on operator and store configuration |
| Secrets Store CSI Driver | Mount external secrets as files | Rotation/sync behavior is driver/provider-specific |
| Application direct secret retrieval | Runtime access to cloud secret manager | Requires identity, network, caching and rotation-aware application logic |

A mounted Secret can update on the filesystem while the application continues using an old in-memory value. For environment variables, plan an approved Pod rollout after changing the referenced Secret.

## 4. Configuration and secrets design rules

1. Keep **non-secret** environment-specific values in version-controlled, reviewed configuration.
2. Keep secret **values** out of Git, TrueFoundry deployment descriptions, CI logs, Docker layers, shell history and support tickets.
3. Use stable, meaningful key names and document which application revision expects which keys.
4. Prefer least-privilege cloud workload identity or approved scoped secret delivery rather than static long-lived credentials.
5. Enforce namespace isolation, RBAC, audit logging, encryption at rest and network restrictions.
6. Use separate development, staging and production secret stores/paths and identities where feasible.
7. Set explicit rotation ownership, expiry alerts, incident response and rollback procedures.
8. Ensure deployment readiness fails safely when required configuration is absent, rather than serving partial or unsafe behavior.
9. Avoid logging the entire process environment or Kubernetes Secret manifests during investigation.

## 5. Kubernetes examples (illustrative)

### 5.1 Non-sensitive ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-app-config
  namespace: tfy-config-lab
data:
  LOG_LEVEL: "info"
  APP_MODE: "staging"
  FEATURE_SAMPLE_ENABLED: "false"
```

### 5.2 Secret metadata and key references

In production, create Secrets using an approved external-secret controller or secret-management workflow, **not plaintext values committed to Git**. This snippet demonstrates a Pod reference only:

```yaml
spec:
  template:
    spec:
      containers:
        - name: hello
          image: registry.example.com/team/hello@sha256:<approved-digest>
          env:
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: hello-app-config
                  key: LOG_LEVEL
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: hello-db-credentials
                  key: password
```

This is a Pod-template fragment, not a complete deployable TrueFoundry or Kubernetes object. If the referenced Secret or key is missing and the reference is not optional, container creation may fail.

### 5.3 Mounted file alternative

```yaml
spec:
  template:
    spec:
      containers:
        - name: hello
          image: registry.example.com/team/hello@sha256:<approved-digest>
          volumeMounts:
            - name: app-config
              mountPath: /etc/hello/config
              readOnly: true
      volumes:
        - name: app-config
          configMap:
            name: hello-app-config
```

Do not mount sensitive values into a shared writable path. Projected-volume refresh is eventual, not a guaranteed instant reload; applications need reload or restart behavior.

## 6. External secret stores and workload identity

### AWS Secrets Manager on EKS

Use a supported workload identity method (such as EKS Pod Identity or IRSA where appropriate), a scoped IAM policy and an approved delivery method (ESO, CSI driver or application API). Verify service account trust, secret ARN/region, KMS decrypt permissions when relevant, endpoint/network access and rotation schedule.

### Azure Key Vault on AKS

Use a supported managed workload identity and federated credential model, appropriate Key Vault RBAC/access policy and a configured provider (CSI, ESO or application API). Validate tenant, vault URI, object version, network restrictions and identity bindings.

### Google Secret Manager on GKE

Use supported Workload Identity Federation for GKE or approved node/application identity, least-privilege secret version access and an authorized CSI/ESO/direct API integration. Validate project, secret version, IAM and egress.

**Identity nuance:** image-pull identity, application runtime identity and external-secret controller identity can be different. Never assume a Pod ServiceAccount alone authorizes every path.

## 7. TrueFoundry integration and environment isolation

In the installed TrueFoundry deployment workflow:

1. Confirm workspace, cluster, namespace, application and intended environment.
2. Identify the authoritative configuration source and owner.
3. Define non-secret variables using supported environment/configuration controls.
4. Bind credentials via supported secret references or cloud identity; avoid entering raw values into versioned deployment files.
5. Verify the rendered Pod's **reference names**, ServiceAccount and mounted volume paths without retrieving values.
6. Test readiness and dependent service connectivity in nonproduction.
7. Record the TrueFoundry deployment revision and configuration version for rollback.

Do not assume staging credentials or database endpoints are valid for production. Namespace labels alone do not enforce credential isolation; verify admission and IAM boundaries.

## 8. Safe diagnostic workflow [SAFE-READ]

### 8.1 Context, deployment and configuration references

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,configmaps
kubectl -n <namespace> get secrets -o custom-columns=NAME:.metadata.name,TYPE:.type
kubectl -n <namespace> get serviceaccounts
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

The Secret listing intentionally displays metadata only. Do **not** use `kubectl get secret -o yaml` or decode `.data` values in tickets or terminal transcripts.

### 8.2 Inspect expected references, not secret values

```bash
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{range .spec.template.spec.containers[*].env[*]}{.name}{" config="}{.valueFrom.configMapKeyRef.name}{" secret="}{.valueFrom.secretKeyRef.name}{"\n"}{end}'
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
kubectl -n <namespace> describe configmap <configmap>
```

**Caution:** ConfigMap descriptions can expose non-secret values that may still be operationally sensitive. Limit distribution. For mounted environment-from references and projected volumes, inspect the corresponding Pod-template fields through an approved, redacted process.

### 8.3 ESO/CSI controller checks (only if installed)

```bash
kubectl get crd | grep -E 'externalsecrets|secretstores|secretproviderclasses'
kubectl -n <namespace> get externalsecrets
kubectl -n <namespace> describe externalsecret <external-secret>
kubectl -n <namespace> get secretproviderclasses
```

Custom-resource commands require the respective CRDs. Operator events may include provider identifiers; sanitize logs and avoid retrieving secret values.

## 9. Configuration failure matrix

| Symptom | Evidence | Likely boundary | Safe response |
|---|---|---|---|
| `CreateContainerConfigError` | Pod events, missing key/name | Pod reference → Secret/ConfigMap | Check resource/key existence |
| Secret object absent | ESO status or CSI events | External provider → cluster | Inspect controller identity and sync |
| `AccessDenied` / 403 | Controller or app logs, IAM audit | Identity → secret store | Verify trust and scoped permission |
| Secret stale after rotation | Secret metadata, rollout time, app state | Rotation → application reload | Confirm env vs volume behavior |
| Pod starts with wrong environment | Workspace/config source, env reference names | Deployment config → Pod | Correct authoritative environment binding |
| TLS/DB auth failure | Sanitized app logs, dependency endpoint | App config → dependency | Verify credential version and endpoint |
| `SecretProviderClass` mount failure | CSI events, provider logs | CSI mount → Pod startup | Check driver/provider/identity |
| Secret accidentally logged | Logs and incident timeline | Confidentiality boundary | Escalate, revoke/rotate, contain |
| Direct kubectl patch reverts | TrueFoundry/GitOps reconciliation | Source of truth → cluster | Fix source rather than patch |
| Missing KMS permission | Cloud audit, provider error | Secret read → decrypt | Verify KMS grant/identity |

**Never** fix secret failures by granting wildcard IAM access, copying production credentials into staging, or logging decrypted secret contents.

## 10. Rotation, rollout and recovery

### Rotation design

1. Identify the credential owner, clients, dependent services, expiration and supported overlap window.
2. Create a new credential/version through the authoritative secret manager.
3. Confirm provider sync or direct retrieval and application reload behavior.
4. Roll out workloads if environment-variable injection or application caching requires restart.
5. Validate authenticated requests and readiness; observe error rate and latency.
6. Revoke the old credential after successful transition and approved overlap.
7. Document rollback: restore an approved previous version or issue a new credential if old material has been revoked/compromised.

**Critical distinction:** after revocation or compromise, reusing an old credential may be unsafe or impossible. Recovery must not depend on reviving exposed credentials.

### Recovery acceptance gates

| Gate | Evidence |
|---|---|
| G1 | Correct environment/workspace and configuration version |
| G2 | Expected non-secret configuration references exist |
| G3 | Required Secret/ESO/CSI references resolve |
| G4 | Least-privilege identity verified |
| G5 | Application Ready and dependency authentication succeeds |
| G6 | Rotation/reload behavior validated |
| G7 | No sensitive values in logs/evidence |
| G8 | Approved rollback and cleanup documented |

## 11. Operational checklist

- [ ] Environment-specific configuration is reviewed and versioned.
- [ ] Secret values are never committed or included in image layers.
- [ ] Identity, namespace and secret-store permissions are least privilege.
- [ ] ConfigMap/Secret injection semantics and reload behavior are documented.
- [ ] External secret reconciliation and alerting are monitored.
- [ ] Rotation and emergency revocation are rehearsed in nonproduction.
- [ ] Recovery checks include authenticated application requests.
- [ ] TrueFoundry authoritative configuration and rollback revision are recorded.

**Hands-on lab:** [5.3 — Configuration and Secrets Validation](../labs/03-application-configuration-environment-secrets-check.md)  
**Next:** Chapter 5.4 — Resource Requests, Limits, Probes and Workload Reliability.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes Secret Good Practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Kubernetes Distribute Credentials Securely](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
- [External Secrets Operator](https://external-secrets.io/latest/)
- [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/)
- [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/)
- [Google Secret Manager](https://cloud.google.com/secret-manager/docs)
