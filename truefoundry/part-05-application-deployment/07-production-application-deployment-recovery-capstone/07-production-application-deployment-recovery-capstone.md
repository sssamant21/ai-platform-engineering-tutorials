# 5.7 — Production Application Deployment and Recovery Capstone

**Part:** 05 — Application Deployment · **Chapter:** 7 of 7 · **Capstone**  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, release managers and application owners  
**Safety:** Default inspection is **[SAFE-READ]**. Deployments, route changes, fault injection and rollbacks are **[LAB-MUTATE]** in an approved isolated nonproduction workspace. Production actions require explicit change/incident approval.

## 1. Mission and outcomes

Design, deploy, validate, troubleshoot and recover a representative HTTP application on TrueFoundry with verifiable evidence across the image supply chain, configuration and secrets, resource sizing, probes, networking, rollout safety and incident response. This is the Part 05 acceptance exercise, integrating Chapters 5.1–5.6.

**Success means:** the application is deployed through its authoritative TrueFoundry workflow, immutable artifact and identity are verified, healthy Pods serve authorized traffic, the team can diagnose failures at the correct layer, and a known-good release can be restored safely.

## 2. Reference architecture

```text
Git commit / reviewed PR / CI run
      ↓ tests, SBOM, scan, provenance
OCI registry: image@sha256:<verified-digest>
      ↓
TrueFoundry workspace / application / deployment revision
      ├─ non-secret environment configuration
      ├─ external secret references / workload identity
      ├─ resource requests and limits
      ├─ startup, readiness, liveness probes
      ├─ replica count and rollout strategy
      └─ internal endpoint policy
      ↓
Cluster integration / desired-state reconciliation
      ↓
Kubernetes Deployment → ReplicaSet → Pods
      ↓                         ↘ events / logs / metrics
Service → ready EndpointSlices → Gateway/Ingress (if configured)
      ↓
Authenticated client request / latency / error SLO
      ↓
Release gates → promote, stop or authoritative rollback
```

The mapping is conceptual. TrueFoundry APIs, deployment fields and generated Kubernetes resources vary by installed version and workload type. Verify actual rendered objects; do not assume every installation uses the same routing controller.

## 3. Capstone environment contract

Use a dedicated **nonproduction** workspace and disposable namespace. All identifiers below are examples, not live values.

| Item | Example contract | Validation |
|---|---|---|
| Workspace | `platform-sre-lab` | Approved TrueFoundry workspace |
| Namespace | `tfy-app-capstone` | Isolated, authorized |
| Application | `hello-platform-capstone` | Owner and cleanup policy |
| Known-good image | `registry.example.com/team/hello@sha256:<digest-A>` | Real digest, scanned, correct architecture |
| Candidate image | `registry.example.com/team/hello@sha256:<digest-B>` | Real digest, compatible with dependencies |
| Port | `8080` | Actual container listener |
| Health endpoints | `/health`, `/ready` | Implemented and verified |
| Requests | CPU 250m, memory 512Mi | Adjust to measured baseline/quota |
| Limits | Memory 1Gi | Validate against load tests |
| Replicas | 2 minimum for availability exercise | Confirm surge headroom |
| Exposure | Internal-only | No public route by default |
| Secret delivery | Approved external secret integration or reference | Least privilege; never expose values |
| Release | TrueFoundry revision + Git commit + digest | Source of truth recorded |

**Do not use placeholders in an actual deployment.** For this capstone, use an approved image that implements the health endpoints and a harmless sample request. No production customer data.

## 4. Readiness gate — before deployment

### 4.1 Cluster and platform discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl get namespaces
kubectl -n <namespace> get resourcequota,limitrange
kubectl -n <namespace> get serviceaccounts
kubectl -n <namespace> get networkpolicies
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Verify TrueFoundry workspace-to-cluster mapping, cluster integration health, namespace RBAC, image pull identity, quota, node capacity and endpoint policy.

### 4.2 Supply-chain controls

Confirm immutable image digest, base image and dependency versions, OS/architecture, vulnerability scan results, SBOM, signature/provenance where enforced, and registry retention of known-good digest A. A locally computed image ID is not necessarily the registry manifest digest.

### 4.3 Security and configuration controls

Keep ordinary configuration in reviewed source; deliver credentials using approved secret manager and scoped identity. Distinguish application runtime identity from kubelet image-pull identity. Do not print Secret values, dump `env`, commit tokens or disable TLS validation.

### 4.4 Capacity and availability

Check request/limit budget for all containers, quotas, node scheduling, replicas, surge capacity and graceful shutdown. A PodDisruptionBudget is not a substitute for spare capacity and does not protect from every outage.

## 5. Deploy the known-good baseline [LAB-MUTATE]

In the **supported TrueFoundry UI or installed version-matched deployment workflow**:

1. Select approved workspace, cluster and namespace.
2. Deploy the approved known-good image digest A.
3. Configure actual port, non-secret values, secret references and least-privilege identity.
4. Set resource budget, startup/readiness/liveness probes and replica count.
5. Configure an **internal-only** endpoint and approved routing.
6. Submit deployment and record TrueFoundry revision, CI run and image digest.
7. Wait for reconciliation; inspect generated Kubernetes resources.

No guessed TrueFoundry CLI/SDK payload is supplied; validate syntax against installed vendor documentation.

### 5.1 Kubernetes reconciliation checks [SAFE-READ]

```bash
kubectl -n <namespace> get deployments,replicasets,pods,services -o wide
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get endpointslices -o wide
```

If the TrueFoundry workload type does not render a Kubernetes Deployment, identify and inspect its actual controller.

### 5.2 Pod-level checks

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get pod <pod> -o jsonpath='{.status.qosClass}{"\n"}'
kubectl -n <namespace> get pod <pod> -o jsonpath='{.spec.containers[*].image}{"\n"}'
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> top pod <pod> --containers
```

`kubectl top` requires Metrics Server or a compatible metrics API. Correlate restarts, memory usage, CPU throttling, probe events, request latency and error rates with monitoring dashboards.

## 6. Validate application connectivity

Inspect Service selectors, `port`/`targetPort`, Pod labels and ready EndpointSlice addresses. Use authorized internal testing.

```bash
kubectl -n <namespace> describe service <service>
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> port-forward service/<service> 18080:<service-port>
```

From another terminal:

```bash
curl -fsS http://127.0.0.1:18080/ready
curl -fsS http://127.0.0.1:18080/health
```

Port-forward is a Kubernetes API tunnel and **does not** prove the ordinary ingress, DNS, TLS, authentication or NetworkPolicy path. Validate the actual TrueFoundry endpoint from an authorized client separately.

## 7. Controlled candidate rollout [LAB-MUTATE]

1. Capture baseline digest A, revision, Ready endpoints, request success and SLO metrics.
2. Check compatibility of candidate B with config, secrets, database schema and dependencies.
3. Define approved stop conditions and last-known-good recovery path.
4. Deploy digest B through the authoritative TrueFoundry workflow.
5. Observe Kubernetes ReplicaSet and Pod transitions and Service endpoint continuity.
6. Compare error rate, P95/P99 latency, restarts, dependency health and saturation.
7. Promote only after all gates pass; otherwise halt and initiate approved rollback.

Kubernetes `progressDeadlineSeconds` reports stalled rollout progress but does not automatically roll back the Deployment. TrueFoundry revision IDs and Kubernetes ReplicaSet revisions need not match.

## 8. Failure-injection design and troubleshooting

**Default exercise is tabletop.** Active faults are permitted only in a dedicated disposable namespace with owner approval and a tested recovery plan. Never introduce faults into shared production infrastructure.

| ID | Scenario | Evidence | First failed transition |
|---|---|---|---|
| F01 | TrueFoundry reconciliation failure | Platform status, integration logs | Desired state → Kubernetes |
| F02 | Wrong image digest | Pod events, registry manifest | Registry → image pull |
| F03 | Missing pull authorization | Node/pull identity, events | Identity → registry |
| F04 | Missing ConfigMap/Secret key | Pod events, reference names | Pod spec → config lookup |
| F05 | External secret provider denied | ESO/CSI status, cloud audit | Identity → secret provider |
| F06 | Pod Pending due to quota/capacity | Scheduler/admission events | Resource policy → placement |
| F07 | Container OOMKilled | Last state, memory peaks | Runtime → memory enforcement |
| F08 | CPU throttling and latency regression | CPU metrics, P99 | CPU cap → runtime |
| F09 | Readiness endpoint failure | Probe events, app logs | Application → Ready |
| F10 | Liveness restart loop | Previous logs, probe settings | Probe → restart |
| F11 | Service selector/port mismatch | Service, EndpointSlice, labels | Service → backend |
| F12 | DNS/TLS/ingress failure | Client test, route/cert metadata | Client → gateway |
| F13 | NetworkPolicy or firewall denial | Policy and source network | Source → destination |
| F14 | Stalled rollout / 503 | ReplicaSet, endpoints, SLO | New revision → availability |
| F15 | Candidate breaks dependency | Traces, errors, migration status | Application → dependency |
| F16 | Manual kubectl patch reverts | Reconciliation status | Source of truth → cluster |

For each, document the **Lowest Proven Healthy Layer**, **First Failed Transition**, owner, smallest approved fix, stop condition, rollback and validation evidence.

## 9. Production-style incident command

### Phase A — Detect and contain

Record time, affected endpoint, request volume, error rate, P95/P99, current TrueFoundry revision and CI deployment event. Freeze promotions when error budget or security controls are breached. Avoid indiscriminate restarts, blanket NetworkPolicy allowances or broad IAM grants.

### Phase B — Isolate

Use a layered path: platform reconciliation → scheduling → image pull → config/identity → process startup → readiness → Service/EndpointSlice → gateway/DNS/TLS → application/dependencies. Capture events and logs at the **first failed transition**, not only the last visible symptom.

### Phase C — Recover

Prefer the smallest authorized repair or restore the **authoritative** TrueFoundry/GitOps known-good revision A. Ensure image, config, secret version, route and data contract are compatible. An irreversible schema migration or compromised credential may require a forward fix or new credential, not a blind rollback.

### Phase D — Validate and close

```bash
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Confirm actual digest, Ready endpoints, authenticated request success, baseline error/latency, restarts and dependencies over an agreed observation window. Record incident timeline, corrective action, preventive action and residual risk.

## 10. Acceptance scorecard

| Gate | Evidence | Pass condition |
|---|---|---|
| G01 Platform | Workspace/cluster integration | Deployment reconciles |
| G02 Supply chain | Digest, scan, provenance | Approved artifact |
| G03 Security | Workload identity and secret references | Least privilege, no leaks |
| G04 Scheduling | Quota, requests/limits, Pod placement | Pods scheduled |
| G05 Runtime | Probe and restart evidence | Pods Ready |
| G06 Networking | Service, EndpointSlice, normal client request | Intended route works |
| G07 Observability | Errors, latency, CPU/memory, logs | Baseline and alert evidence |
| G08 Release | Candidate rollout and stop conditions | Safe rollout |
| G09 Recovery | Authoritative rollback/recovery | Known-good behavior restored |
| G10 Operations | Cleanup, runbook, owner, RCA/COA | Complete evidence |

**Publication is not live execution.** Mark any unavailable check **Not run**, not Pass.

## 11. Evidence and handoff template

```text
Lab date / approver / incident commander:
TrueFoundry version, workspace, deployment:
Kubernetes context, namespace, controller:
Git commit, CI build, image digest A/B:
SBOM / scan / signature status:
ConfigMap/Secret reference names and identities:
CPU/memory requests and limits / QoS:
Probe settings, restarts, Ready Pods:
Service selector / port / EndpointSlice:
Client endpoint / DNS / TLS / auth result:
Baseline errors, P95/P99, saturation:
Rollout revision and observed behavior:
Injected/tabletop failure ID:
Lowest Proven Healthy Layer:
First Failed Transition:
Remediation / owner / approval:
Rollback revision and elapsed time:
Post-recovery request/SLO result:
Cleanup and residual resources:
G01–G10 status:
Open actions / formal reviews pending:
```

## 12. Part 05 completion criteria

- All seven canonical chapters and seven companion labs published and linked.
- Master TOC, Part 05 layout and Status Tracker updated.
- Repository files verified after publication.
- Formal technical/vendor validation, production/SRE review and **live lab execution** tracked separately and never implied by publication.
- Ready to proceed to **Part 06 — Model Serving**.

**Hands-on capstone:** [5.7 — Production Application Deployment and Recovery Acceptance Lab](../labs/07-production-application-deployment-recovery-capstone-check.md)

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
