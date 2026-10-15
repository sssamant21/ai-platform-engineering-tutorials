# Lab 5.7 — Production Application Deployment and Recovery Acceptance

**Canonical:** [Chapter 5.7](../07-production-application-deployment-recovery-capstone/07-production-application-deployment-recovery-capstone.md)  
**Mode:** Read-only discovery by default; deploy, rollback and fault injection only in an approved isolated nonproduction TrueFoundry workspace.

## Objective

Demonstrate end-to-end readiness, deployment, health, routing, release safety, incident triage, authoritative rollback and cleanup with auditable evidence. This is the Part 05 final acceptance lab.

## 0. Safety gate and ownership

- Confirm TrueFoundry workspace, Kubernetes context, namespace, release owner and change approval.
- Confirm disposable resources, internal-only endpoint, test data and explicit cleanup owner.
- Confirm immutable approved image digests A (known good) and B (candidate), image-pull identity, secret references and rollback availability.
- Define stop conditions and an observation window. No shared-cluster pressure tests, credential leaks, public exposure or production fault injection.

```bash
kubectl config current-context
kubectl -n <namespace> get resourcequota,limitrange
kubectl -n <namespace> get deployments,pods,services -o wide
```

## 1. Predeployment review

| Check | Evidence to collect |
|---|---|
| Workspace/cluster mapping | TrueFoundry workspace and integration status |
| Artifact | Git commit, CI run, immutable digest, architecture |
| Supply chain | SBOM, scan, signature/provenance policy |
| Identity | ServiceAccount, image-pull identity, cloud secret-store role |
| Configuration | Non-secret variables, Secret references (names only) |
| Capacity | Node quota, CPU/memory requests, surge headroom |
| Reliability | Replicas, startup/readiness/liveness, graceful termination |
| Exposure | Internal endpoint, DNS/TLS/auth requirements |
| Rollback | Known-good revision A and compatible dependencies |

If an item cannot be verified, mark **Not run** and assign an owner; do not fabricate approval.

## 2. Deploy baseline A [LAB-MUTATE]

Through the installed supported TrueFoundry UI/SDK/CI workflow:
1. Deploy approved image digest A in the disposable workspace.
2. Configure port, identity, config/secret references, requests/limits and health probes.
3. Select internal-only exposure and appropriate replica count.
4. Record TrueFoundry deployment revision and generated Kubernetes controller.
5. Verify reconciliation:

```bash
kubectl -n <namespace> get deployments,replicasets,pods,services -o wide
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get endpointslices -o wide
```

If the platform renders a non-Deployment controller, use its actual inspection workflow.

## 3. Verify Pod, resources and probes [SAFE-READ]

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get pod <pod> -o jsonpath='{.status.qosClass}{"\n"}'
kubectl -n <namespace> get pod <pod> -o jsonpath='{.spec.containers[*].image}{"\n"}'
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> top pod <pod> --containers
```

Record CPU/memory requests and limits, restart count, termination reasons, readiness and liveness behavior. Metrics Server may not be installed; mark `top` unavailable if so.

## 4. Verify Service and endpoint [SAFE-READ]

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

Port-forward does not test the ordinary ingress/TLS/NetworkPolicy path. Separately validate the authorized TrueFoundry endpoint from the expected client network.

## 5. Roll out candidate B [LAB-MUTATE]

1. Record baseline A's image, TrueFoundry revision, available Pods, endpoint addresses and request SLO.
2. Validate B's image digest, compatibility and surge capacity.
3. Submit candidate B through the **authoritative** TrueFoundry workflow.
4. Observe:

```bash
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

5. Compare request success, error rate, P95/P99 latency, restarts and dependency health.
6. Stop and revert if agreed thresholds are breached.

## 6. Authoritative rollback [LAB-MUTATE]

Restore known-good digest A and compatible configuration through TrueFoundry/GitOps. **Do not** rely on a Kubernetes-only `rollout undo` when TrueFoundry reconciles the workload.

Verify actual Pod image, Ready endpoints, application request success and stable SLO. Record rollback duration and remaining risk. If a schema migration or credential revocation prevents restoration, document the compatible forward-recovery path instead.

## 7. Sixteen incident scenarios (tabletop by default)

| ID | Scenario | First failed transition |
|---|---|---|
| F01 | Control-plane reconciliation denied | TrueFoundry → Kubernetes |
| F02 | Image manifest missing | Registry → pull |
| F03 | Image-pull credentials denied | Identity → registry |
| F04 | Missing ConfigMap/Secret key | Pod spec → configuration |
| F05 | External secret sync denied | Controller → secret provider |
| F06 | Pod Pending / quota | Resource budget → scheduler |
| F07 | OOMKilled | Runtime → memory enforcement |
| F08 | CPU throttling / P99 spike | CPU limit → application |
| F09 | Readiness 503 | Application → Ready |
| F10 | Liveness restart loop | Probe → kubelet |
| F11 | Service selector/port mismatch | Service → Pod |
| F12 | DNS/TLS/Ingress mismatch | Client → gateway |
| F13 | NetworkPolicy/firewall denied | Source → destination |
| F14 | Rollout stalls / endpoints drain | ReplicaSet → availability |
| F15 | Dependency incompatibility | Application → data contract |
| F16 | Manual patch reverts | Authoritative desired state → cluster |

For each scenario, record **lowest proven healthy layer**, evidence, responsible team, smallest safe repair, rollback condition and validation.

## 8. Acceptance scorecard

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G01 Platform reconciliation | | |
| G02 Artifact and supply chain | | |
| G03 Identity and secrets | | |
| G04 Scheduling and capacity | | |
| G05 Runtime and probes | | |
| G06 Normal endpoint connectivity | | |
| G07 Observability and SLO | | |
| G08 Candidate rollout | | |
| G09 Authoritative rollback | | |
| G10 Cleanup and incident handoff | | |

**Pass:** all required gates supported by observed evidence, no secret exposure, no unapproved production changes, and lab-owned resources removed. If access is unavailable, the exercise remains **Not run** rather than falsely passed.

## 9. Cleanup [LAB-MUTATE]

Delete only the application and routes created for this lab through the supported TrueFoundry workflow. Verify residual Pods, Services, EndpointSlices, load balancers and cloud resources. Do not delete shared namespaces, Secrets, external secret controllers or registry repositories.

```bash
kubectl -n <namespace> get deployments,replicasets,pods,services
kubectl -n <namespace> get endpointslices
```

## 10. Evidence report

```text
Date/time, lab approver and owner:
TrueFoundry version/workspace/application:
Cluster/namespace and controller:
Git commits, CI runs, digest A/B:
Image scan/SBOM/signature:
Config and Secret references (no values):
Identity and registry pull method:
CPU/memory, probes, replicas, QoS:
Service/EndpointSlice and normal endpoint test:
Baseline and candidate SLO:
Rollout observations:
Rollback authority, revision and duration:
F01–F16 diagnostic worksheet:
G01–G10 scorecard:
Cleanup evidence:
Formal vendor/SRE review status:
Open follow-ups:
```

**Next:** Part 06 — Model Serving, Chapter 6.1 — Model-Serving Fundamentals and Architecture.
