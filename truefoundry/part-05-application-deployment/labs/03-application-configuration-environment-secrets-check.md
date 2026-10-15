# Lab 5.3 — Application Configuration, Environment Variables and Secrets Check

**Canonical:** [Chapter 5.3](../03-application-configuration-environment-secrets/03-application-configuration-environment-secrets.md)  
**Default:** [SAFE-READ] investigation. **Optional:** [LAB-MUTATE] with a disposable namespace and non-sensitive demonstration data only.

## Objectives

Trace TrueFoundry application configuration into Kubernetes, distinguish ConfigMap and Secret delivery, verify external secret-controller status without exposing values, diagnose missing references and design a safe rotation/rollback.

## 0. Safety preflight

```bash
kubectl config current-context
kubectl get namespaces
kubectl -n <namespace> get deployments,pods
```

Confirm cluster, TrueFoundry workspace, environment and authorized namespace. **Do not** print Secret data, decode Kubernetes Secrets, run `env` in sensitive containers or paste credentials into manifests.

## 1. Configuration inventory [SAFE-READ]

```bash
kubectl -n <namespace> get configmaps
kubectl -n <namespace> get secrets -o custom-columns=NAME:.metadata.name,TYPE:.type
kubectl -n <namespace> get serviceaccounts
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record reference names and ownership only. Treat ConfigMap values as potentially operationally sensitive even when not classified as secrets.

## 2. Trace Pod-template references [SAFE-READ]

```bash
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{range .spec.template.spec.containers[*].env[*]}{.name}{" config="}{.valueFrom.configMapKeyRef.name}{" secret="}{.valueFrom.secretKeyRef.name}{"\n"}{end}'
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
```

Check environment variables, `envFrom`, volumes and projected mounts using an approved redacted manifest review. Determine whether the authoritative source is TrueFoundry, GitOps or another controller.

## 3. Optional non-sensitive ConfigMap smoke test [LAB-MUTATE]

Only in a disposable cluster/namespace created or assigned for this lab. Do not create real credentials.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tfy-config-demo
  namespace: tfy-config-lab
data:
  APP_MODE: "lab"
  FEATURE_FLAG: "false"
---
apiVersion: v1
kind: Pod
metadata:
  name: tfy-config-demo
  namespace: tfy-config-lab
spec:
  restartPolicy: Never
  containers:
    - name: config-check
      image: <approved-shell-image@sha256:digest>
      command: ["/bin/sh", "-c"]
      args: ["test \"$APP_MODE\" = lab && echo CONFIG_REFERENCE_PASS"]
      env:
        - name: APP_MODE
          valueFrom:
            configMapKeyRef:
              name: tfy-config-demo
              key: APP_MODE
```

Replace the image placeholder with an approved immutable image containing `/bin/sh`. Create `tfy-config-lab` only if authorized and absent.

```bash
kubectl create namespace tfy-config-lab
kubectl apply -f tfy-config-demo.yaml
kubectl -n tfy-config-lab get pod tfy-config-demo -o wide
kubectl -n tfy-config-lab describe pod tfy-config-demo
kubectl -n tfy-config-lab logs tfy-config-demo
```

Expected output: `CONFIG_REFERENCE_PASS`. This test proves non-sensitive ConfigMap environment injection, **not** secret confidentiality or rotation.

## 4. External secret reconciliation [SAFE-READ]

```bash
kubectl get crd | grep -E 'externalsecrets|secretstores|secretproviderclasses'
```

If ESO is installed:

```bash
kubectl -n <namespace> get externalsecrets
kubectl -n <namespace> describe externalsecret <external-secret>
```

If CSI is installed:

```bash
kubectl -n <namespace> get secretproviderclasses
kubectl -n <namespace> describe pod <pod>
```

If CRDs are absent, mark **Not installed**; do not install controllers in production as part of this lab. Record sync conditions and errors without printing provider payloads.

## 5. Failure scenarios (tabletop)

| Case | Symptom | Evidence | First failed transition |
|---|---|---|---|
| C1 | Missing ConfigMap key | Pod event | Reference → config lookup |
| C2 | Missing Secret | Pod event / metadata | Reference → Secret lookup |
| C3 | ESO AccessDenied | Controller condition, IAM audit | Identity → provider |
| C4 | CSI mount failure | Pod events, driver logs | Provider → mounted file |
| C5 | Secret rotated, app still uses old value | Update/rollout timeline | Secret refresh → process |
| C6 | Wrong staging/prod endpoint | Deployment source and reference | Environment mapping |
| C7 | Secret leaked into logs | Sanitized incident report | Confidentiality |
| C8 | Kubernetes patch reverted | TrueFoundry/GitOps reconciliation | Desired state |

For each case, identify owner, smallest safe fix, verification and rollback. Do not simulate leaks or modify production credentials.

## 6. Rotation and recovery tabletop

```text
Secret reference (not value):
Environment and owner:
Cloud secret store / region:
Workload or controller identity:
Current version metadata:
Dependent applications:
Rotation overlap window:
Environment variable or mounted file:
Reload / rollout required:
Validation request and SLO:
Revocation plan:
Rollback or new-credential contingency:
Audit and alert evidence:
```

Document why environment-injected values need a Pod restart and why mounted file updates do not guarantee the application reloads.

## 7. Cleanup [LAB-MUTATE]

Only for resources created by this lab:

```bash
kubectl -n tfy-config-lab delete pod tfy-config-demo --ignore-not-found
kubectl -n tfy-config-lab delete configmap tfy-config-demo --ignore-not-found
kubectl -n tfy-config-lab get pods,configmaps
```

Delete the namespace only if it was created solely for this exercise and contains no unrelated resources. Never delete shared Secrets, ExternalSecrets or SecretProviderClasses.

## 8. Acceptance report

```text
Date/time / cluster / namespace:
TrueFoundry workspace / deployment revision:
ConfigMap references:
Secret references (names only):
ServiceAccount and provider identity:
ESO/CSI installed? Conditions:
Environment isolation:
Non-sensitive lab output:
C1–C8 classification:
Rotation/reload/rollback plan:
Sensitive values exposed? (must be No):
Cleanup evidence:
Unexecuted checks:
```

**Pass:** correctly trace references and ownership, document safe rotation, classify eight scenarios, expose no sensitive values and clean up disposable resources. If no mutation access exists, mark the optional smoke test **Not run**.

**Next:** Chapter 5.4 — Resource Requests, Limits, Probes and Workload Reliability.
