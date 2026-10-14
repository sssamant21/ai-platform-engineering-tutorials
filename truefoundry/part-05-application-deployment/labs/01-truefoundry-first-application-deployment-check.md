# Lab 5.1 — TrueFoundry First Application Deployment Check

**Canonical:** [Chapter 5.1](../01-truefoundry-application-deployment-first-service/01-truefoundry-application-deployment-first-service.md)  
**Default:** [SAFE-READ] discovery. Deployment and cleanup are **[LAB-MUTATE]** in an approved nonproduction workspace.

## Objectives

Deploy one approved HTTP container through the installed TrueFoundry workflow; verify its generated Kubernetes resources, probes and internal endpoint; capture a failed-rollout diagnosis; and remove only resources created by this lab.

## 0. Safety and prerequisites

Confirm nonproduction TrueFoundry workspace, mapped cluster/namespace, appropriate RBAC, approved immutable image digest, application port, `/ready` and `/health` routes, resource quota and cleanup ownership. **Do not** deploy to production for this lab.

```bash
kubectl config current-context
kubectl get namespaces
kubectl -n <namespace> get resourcequota,limitrange
```

Record TrueFoundry product version and the installed deployment UI/SDK/CLI procedure. Do not assume example Kubernetes YAML is a valid TrueFoundry API payload.

## 1. Deployment plan [LAB-MUTATE]

In the **supported TrueFoundry UI or version-matched deployment workflow**:

1. Select approved workspace and nonproduction cluster/namespace.
2. Create application `hello-platform` (or an approved unique name).
3. Set approved immutable image `registry.example.com/team/hello@sha256:<real-digest>`.
4. Set container port `8080` only if that is the application's actual listen port.
5. Configure readiness `/ready`, liveness `/health` and suitable startup delay/probe, only if those routes exist.
6. Set CPU request `100m`, memory request `256Mi`, memory limit `512Mi` if quota allows.
7. Use one replica and internal-only exposure for the disposable exercise.
8. Select least-privilege workload identity and reference secrets by approved secret objects; do not paste credentials.
9. Submit deployment and record TrueFoundry revision/operation identifier.

**Version-aware instruction:** UI field names and SDK object schemas may differ; follow the actual installed TrueFoundry documentation. No guessed `tfy deploy` command is provided.

## 2. Verify reconciliation [SAFE-READ]

```bash
kubectl -n <namespace> get deployments,replicasets,pods,services -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
```

If the workload type does not generate a Kubernetes Deployment, identify and inspect the actual controller. Compare image digest, replica count and labels with TrueFoundry desired state.

## 3. Inspect Pod and health [SAFE-READ]

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> get services,endpointslices -o wide
kubectl -n <namespace> describe service <service>
```

Confirm PodScheduled, container state, Ready condition, probe behavior and ready Service endpoints. Avoid printing secrets.

## 4. Internal request check [SAFE-READ with authorized port-forward]

```bash
kubectl -n <namespace> port-forward service/<service> 8080:<service-port>
```

In a second terminal:

```bash
curl -i http://127.0.0.1:8080/ready
curl -i http://127.0.0.1:8080/health
```

If your sample exposes an application endpoint, call it as well. A port-forward bypasses external ingress; record that it validates internal Service routing only.

## 5. Failure exercises (non-destructive)

| Scenario | Evidence | Expected diagnosis |
|---|---|---|
| Invalid image digest | Pod events | ImagePullBackOff / registry |
| Missing Secret reference | Pod events | CreateContainerConfigError |
| Wrong readiness path | Probe events | Running but not Ready |
| Service selector mismatch | Service and EndpointSlice | No ready endpoints |
| CPU/memory request exceeds quota | Admission/scheduling events | Rejected or Pending |
| TrueFoundry reconciles manual patch | Controller desired-state comparison | Authoritative source mismatch |

Do not intentionally introduce these faults into a shared environment. For each, record the first failed transition, actionable owner and source-of-truth fix.

## 6. Rollback and cleanup [LAB-MUTATE]

Record the previous TrueFoundry revision before any update. If the lab deployment fails, revert or delete it **through the supported TrueFoundry workflow**; do not rely on manual Kubernetes patches that reconciliation may undo.

Delete only the application and supporting resources created for this lab, following workspace ownership rules. Then verify:

```bash
kubectl -n <namespace> get deployments,replicasets,pods,services
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Do not delete a shared namespace or unrelated Services/Secrets. Record any residual resources and owner.

## 7. Acceptance report

```text
Date/time / cluster context:
TrueFoundry workspace and product version:
Deployment workflow / revision:
Approved image digest:
Namespace and Kubernetes controller:
CPU/memory and probe settings:
Pod readiness / events:
Service and ready EndpointSlices:
Health and application request result:
Logs / metrics availability:
First failed transition (if any):
Rollback method and revision:
Cleanup evidence:
G1 TrueFoundry reconciled:
G2 Correct controller/image:
G3 Pod Ready:
G4 Ready Service endpoints:
G5 Request success:
G6 Observability:
G7 Rollback documented:
G8 Cleanup:
```

**Pass:** approved nonproduction deployment and health checks succeed, evidence is recorded and lab resources are cleaned up. If no deployment access exists, mark execution **Not run** and complete the read-only diagnostic exercise without claiming a successful deployment.

**Next:** Chapter 5.2 — Container Images, Registries and Supply Chain.
