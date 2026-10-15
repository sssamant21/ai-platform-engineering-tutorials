# 5.6 — Deployment Revisions, Rollouts, Rollbacks and Troubleshooting

**Part:** 05 — Application Deployment · **Chapter:** 6 of 7  
**Audience:** TrueFoundry platform engineers, SREs, release engineers and application owners  
**Safety:** Inspection is **[SAFE-READ]**. Deployments, scale changes, pause/resume and rollbacks are **[CHANGE-CONTROL]** in production; practice only in an approved disposable workspace.

## 1. Learning objectives

Understand TrueFoundry deployment revision ownership and Kubernetes Deployment/ReplicaSet revisions; plan and validate safe releases; distinguish rolling updates, canaries and blue/green strategies; diagnose stalled rollouts; and recover using an authoritative, verified rollback.

## 2. Desired state and revision ownership

```text
Git commit / CI build / signed image digest
       ↓
TrueFoundry deployment specification (authoritative when configured)
       ↓
TrueFoundry reconciliation / installed cluster integration
       ↓
Kubernetes Deployment → ReplicaSet revision → Pods
       ↓
Readiness → Service EndpointSlices → user traffic
       ↓
SLO monitoring → promote / halt / rollback
```

**Critical:** Kubernetes Deployment revision numbers and TrueFoundry deployment revisions are not necessarily the same. Kubernetes creates new ReplicaSets when the Pod template changes; not every TrueFoundry metadata change creates a new ReplicaSet. Inspect the installed TrueFoundry release workflow and generated Kubernetes objects. Direct `kubectl rollout undo` may be overwritten by reconciliation and should not be the default production rollback.

## 3. Release artifact and configuration contract

Record the following for each release:

| Artifact | Required evidence |
|---|---|
| Source | Git commit, reviewed PR, CI run |
| Image | Immutable OCI digest, platform architecture, registry |
| Deployment | TrueFoundry workspace, app, deployment revision |
| Runtime | CPU/memory, probes, replicas, security context |
| Dependencies | ConfigMap/Secret references, migration compatibility |
| Routing | Service/port, endpoint, ingress or Gateway route |
| Rollout | Strategy, maxSurge, maxUnavailable, capacity budget |
| Observability | Availability, error rate, P95/P99 latency, saturation |
| Recovery | Last known-good digest, configuration and owner |

A rollback that restores only an image may fail if the database schema, configuration, secret version or network route changed incompatibly. Treat application and dependency compatibility as part of the release.

## 4. RollingUpdate behavior

Kubernetes Deployment RollingUpdate uses `maxSurge` to allow additional Pods above desired replicas and `maxUnavailable` to bound unavailable replicas during the update. Percentages are rounded according to Kubernetes rules; inspect effective replica counts and capacity. For a single replica, zero-downtime cannot be assumed, even with a surge Pod: readiness, capacity, dependencies and connection draining matter.

**Illustrative Kubernetes fragment — not a TrueFoundry API schema:**

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 10
  progressDeadlineSeconds: 600
```

`progressDeadlineSeconds` reports stalled progress; it does **not** automatically roll back the Deployment. A PodDisruptionBudget does not guarantee availability through every rollout or failure. Ensure sufficient node quota, headroom and readiness behavior before increasing surge.

## 5. Rolling, canary and blue/green strategies

| Strategy | Benefit | Risk / prerequisite |
|---|---|---|
| Rolling update | Gradual Pod replacement without duplicate full environment | Needs readiness, surge capacity, compatible revisions |
| Canary | Expose small traffic slice and compare SLOs | Requires controller/platform support for weighted routing or safe traffic splitting |
| Blue/green | Keep two environments, switch routing after validation | Double capacity, state/schema compatibility and cutover controls |
| Recreate | Simpler for workloads that cannot run concurrently | Planned downtime; requires explicit approval |

Do **not** claim a TrueFoundry version supports weighted canary/blue-green without verifying its installed capabilities and routing controller. Where unavailable, use an approved GitOps/service-mesh/Gateway release mechanism.

## 6. Read-only rollout investigation

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> rollout history deployment/<deployment>
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

For a failing new Pod:

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> logs <pod> --previous --tail=150
kubectl -n <namespace> get services,endpointslices -o wide
```

The `--previous` command may return no logs when the container has not restarted. Check `Progressing`, `Available`, `ReplicaFailure`, ReplicaSet ownership, new Pod readiness and desired/current/updated/available counts. A completed Kubernetes rollout is not proof of application correctness or business SLO compliance.

### Compare revision artifacts without exposing secrets

```bash
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{.spec.replicas}{"\n"}'
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{.spec.strategy.type}{"\n"}'
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{.status.availableReplicas}{"\n"}'
```

Compare prior and current **source-controlled** specifications for probes, resources, secret references and routing. Avoid dumping environment variables or Secret values into incident tickets.

## 7. Release readiness and monitoring

Before deployment:
1. Confirm immutable image digest, scan/provenance status and approved change.
2. Verify cluster capacity for surge, quota, image pull, storage and dependency connections.
3. Confirm startup/readiness/liveness probes, termination grace and graceful shutdown.
4. Record baseline request volume, error rate, P95/P99 latency, saturation, restarts and endpoint availability.
5. Define canary duration, stop thresholds, on-call owner and rollback time budget.

During rollout, compare new and old revisions using consistent time windows and traffic cohorts. Low traffic can hide failures; a successful health probe alone is not an adequate promotion gate.

### Example stop conditions (customize to SLO)

- New revision has zero ready endpoints beyond its expected startup window.
- Sustained error-rate regression compared with baseline.
- P99 latency or queueing breaches approved guardrails.
- OOMKilled/CrashLoopBackOff or high restart rate.
- Dependency error or schema incompatibility.
- Unexpected traffic exposure or security policy violation.

## 8. Failure scenarios and first failed transition

| Symptom | Evidence | Boundary | Safe response |
|---|---|---|---|
| Deployment does not reconcile | TrueFoundry status, agent health, RBAC | Control plane → cluster | Verify integration and permissions |
| Pod Pending | Events, quota, affinity, node capacity | Scheduler | Resolve capacity/config |
| ImagePullBackOff | Pod events, digest, registry identity | Registry → kubelet | Fix artifact/pull access |
| CrashLoopBackOff | Previous logs, exit reason | Container startup | Fix application/config |
| OOMKilled | Last state, memory metrics | Runtime memory | Restore safe resource budget |
| Startup/readiness probe fails | Probe events, app logs | App → Ready | Fix path/timeout/startup |
| Rollout progress deadline exceeded | Deployment conditions | New ReplicaSet → Available | Inspect first failing Pod |
| Endpoint churn causes 503 | EndpointSlice, route logs | Ready Pods → traffic | Restore availability |
| New revision healthy but errors rise | Traces, SLO and dependency logs | Application → dependency | Halt/rollback |
| Rollback appears to revert again | TrueFoundry/GitOps reconciliation | Authoritative source → cluster | Restore desired state there |
| Canary receives wrong share | Gateway/service mesh route | Route weights → traffic | Correct approved routing |
| Database schema incompatible | Application errors, migration state | Revision → data contract | Execute compatibility recovery |

## 9. Production incident runbook

**A. Stabilize and classify [SAFE-READ]**

1. Record impact, first error time, TrueFoundry workspace/deployment and last good revision.
2. Correlate CI commit, image digest, deployment event and first SLO regression.
3. Identify **Lowest Proven Healthy Layer** and **First Failed Transition**.
4. Confirm whether failure affects new Pods only, all Pods, or routing/dependencies.
5. Halt further promotions and notify owners under the incident process.

**B. Choose recovery [CHANGE-CONTROL]**

- If only the new image/configuration is defective and backward-compatible, revert the authoritative TrueFoundry/GitOps release to the last known-good artifact.
- If a dependent database migration or credential rotation is irreversible, use a compatible forward fix or approved data recovery; do not blindly restore an incompatible image.
- If rollout is blocked by cluster capacity, image pull or policy, repair the dependency with approved changes or revert the release.
- For blue/green/canary, restore the approved traffic route and verify actual traffic distribution before deleting the bad environment.

**C. Validate recovery**

```bash
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Verify the **actual** image digest, Ready endpoints, authenticated application requests, error rate, latency, restarts and dependency health. Observe for a defined stability window.

## 10. Rollback command caveat

```bash
# Read-only discovery
kubectl -n <namespace> rollout history deployment/<deployment>
```

```bash
# CHANGE-CONTROL ONLY — illustrative Kubernetes-native rollback
kubectl -n <namespace> rollout undo deployment/<deployment> --to-revision=<known-good-k8s-revision>
```

The second command mutates production state and may conflict with TrueFoundry reconciliation. Use it **only** if Kubernetes is confirmed to be the authorized source of truth or incident command explicitly approves the temporary action with a plan to update the authoritative configuration. The specified Kubernetes revision must exist and be compatible; ReplicaSet revision history can be pruned.

## 11. Acceptance gates

| Gate | Evidence |
|---|---|
| G1 | Source-of-truth and revision mapping documented |
| G2 | Immutable image, config and dependency compatibility verified |
| G3 | Rolling strategy and capacity headroom evaluated |
| G4 | New Pods become Ready and endpoints are stable |
| G5 | SLOs and application requests pass release gates |
| G6 | Failure classified with first failed transition |
| G7 | Authoritative rollback tested in nonproduction |
| G8 | Post-recovery evidence and cleanup complete |

## 12. Evidence template

```text
Date/time, cluster, namespace:
TrueFoundry workspace/app/deployment:
Git commit / CI run / image digest:
TrueFoundry revision / Kubernetes ReplicaSet revision:
Old vs new resources/probes/config references:
Rollout strategy / surge capacity:
Desired / updated / available replicas:
Readiness / EndpointSlices:
Errors, P95/P99, restarts and dependency health:
Lowest Proven Healthy Layer:
First Failed Transition:
Approval / stop condition:
Authoritative rollback artifact:
Post-recovery SLO validation:
Cleanup and follow-up:
```

**Hands-on lab:** [5.6 — Rollout and Recovery Check](../labs/06-deployment-revisions-rollouts-rollbacks-troubleshooting-check.md)  
**Next:** Chapter 5.7 — Production Application Deployment and Recovery Capstone.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Rolling Update](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
- [Kubernetes Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes PodDisruptionBudgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
