# Lab 5.6 — Deployment Revisions, Rollouts, Rollbacks and Troubleshooting Check

**Canonical:** [Chapter 5.6](../06-deployment-revisions-rollouts-rollbacks-troubleshooting/06-deployment-revisions-rollouts-rollbacks-troubleshooting.md)  
**Safety:** Default **[SAFE-READ]**. Optional deployments, route changes and rollbacks are **[LAB-MUTATE]** in an approved disposable TrueFoundry workspace. No production fault injection.

## 1. Objectives

Map TrueFoundry and Kubernetes revisions; inspect rolling-update strategy, ReplicaSets, Pod readiness and endpoint continuity; classify failed rollouts; demonstrate an approved nonproduction rollback; and capture evidence.

## 2. Preflight and release contract

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> get resourcequota,limitrange
```

Record TrueFoundry workspace/app, installed version, source of truth, approved namespace, image digests for **known-good** and **candidate** versions, application port/probes, resource budget and owner. Verify both images exist and are compatible with dependencies. Do not use floating `latest` tags.

## 3. Baseline revision discovery [SAFE-READ]

```bash
kubectl -n <namespace> rollout history deployment/<deployment>
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> get replicasets -o wide
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
kubectl -n <namespace> get services,endpointslices -o wide
```

Document desired, updated, ready and available replicas; ReplicaSet ownership; rolling strategy; current TrueFoundry revision; and current image digest. **Do not assume TrueFoundry revision IDs equal Kubernetes rollout revision numbers.**

## 4. Monitor a controlled nonproduction rollout [LAB-MUTATE]

1. In the supported installed TrueFoundry UI/SDK/CI workflow, deploy a disposable HTTP service at known-good digest A.
2. Confirm readiness, EndpointSlice membership, application request success and baseline error/latency.
3. Record release A's TrueFoundry revision and Kubernetes ReplicaSet.
4. Submit candidate digest B through the same authoritative workflow with approved RollingUpdate settings.
5. Observe:

```bash
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get endpointslices -o wide
```

6. Verify the new image digest, health endpoints and application response. Compare against release A's baseline. Do not promote if agreed stop conditions are breached.

If no deployment access is available, perform steps 2–5 as read-only observation and mark the mutation exercise **Not run**.

## 5. Controlled rollback [LAB-MUTATE]

Use the TrueFoundry or GitOps authoritative release workflow to restore known-good digest A and its compatible configuration. Do not rely on a Kubernetes-only `rollout undo` unless Kubernetes is the authorized source of truth.

```bash
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get deployment <deployment> -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
kubectl -n <namespace> get endpointslices -o wide
```

Verify request success, readiness, error rate, P99 latency and stable replica count. Record rollback duration and whether any old Pods remained ready.

## 6. Failure scenarios (tabletop, no shared-cluster fault injection)

| ID | Symptom | Evidence | First failed transition |
|---|---|---|---|
| D1 | TrueFoundry deployment not reconciled | Platform status, agent events | Control plane → Kubernetes |
| D2 | Pending new Pods | Scheduler events, quota | Capacity → placement |
| D3 | ImagePullBackOff | Pod events, digest/registry | Registry → kubelet |
| D4 | CrashLoopBackOff | Previous logs, exit reason | Container startup |
| D5 | OOMKilled | Last state, memory peaks | Runtime → memory limit |
| D6 | Startup/readiness probe fails | Probe events, app logs | App → Ready |
| D7 | Progress deadline exceeded | Deployment conditions | New ReplicaSet → Available |
| D8 | 503 during rollout | EndpointSlice and route logs | Ready endpoints → gateway |
| D9 | App requests fail despite Ready | Traces, dependency errors | App → dependency |
| D10 | Kubernetes patch reverts | Reconciliation logs | Source of truth → cluster |
| D11 | Wrong canary traffic share | Gateway route policy | Route → traffic |
| D12 | Schema incompatibility blocks rollback | DB migration and app logs | Revision → data contract |

For each case, state the lowest proven healthy layer, responsible owner, smallest approved remediation, rollback constraint and post-recovery test.

## 7. Release acceptance worksheet

| Gate | Required proof | Pass / Fail / Not run |
|---|---|---|
| G1 | TrueFoundry and Kubernetes revisions mapped | |
| G2 | Both digests and config compatibility checked | |
| G3 | Surge/headroom and quotas checked | |
| G4 | Candidate Pod Ready, endpoints populated | |
| G5 | Request SLOs and dependencies validated | |
| G6 | Failure cases classified | |
| G7 | Source-of-truth rollback successful | |
| G8 | Cleanup and evidence complete | |

## 8. Evidence report

```text
Date/time / cluster / namespace:
TrueFoundry version / workspace / deployment:
Known-good Git commit / image digest:
Candidate Git commit / image digest:
TrueFoundry revision IDs:
Kubernetes ReplicaSet revisions:
Rolling strategy / replicas / surge:
Baseline request rate, errors, P95/P99:
Candidate request and readiness results:
EndpointSlice continuity:
Stop conditions and observations:
Rollback authority / execution / duration:
Post-rollback health and SLO:
D1–D12 classification:
Cleanup / residual resources:
Unexecuted steps:
```

## 9. Cleanup

Remove only the disposable TrueFoundry deployment and routes created by this lab through the supported platform workflow. Verify no lab-owned Pods, Services, EndpointSlices or external load balancers remain. Never delete shared namespaces, Secrets or registry artifacts without separate approval.

**Pass:** revision ownership is understood, release and rollback are verified where authorized, failure evidence is recorded, and no unapproved production changes occur.

**Next:** Chapter 5.7 — Production Application Deployment and Recovery Capstone.
