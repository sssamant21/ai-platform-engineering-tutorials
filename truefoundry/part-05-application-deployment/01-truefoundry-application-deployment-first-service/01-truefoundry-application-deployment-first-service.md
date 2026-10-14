# 5.1 — TrueFoundry Application Deployment Architecture and First Service

**Part:** 05 — Application Deployment · **Chapter:** 1 of 7  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, application developers  
**Safety:** Default discovery is **[SAFE-READ]**. The first-service exercise is **[LAB-MUTATE]** in an approved nonproduction workspace/namespace. TrueFoundry UI, SDK and API fields vary by version; verify installed release and documentation before applying configuration.

## 1. Objectives

Explain how a TrueFoundry application deployment maps to Kubernetes objects; deploy an approved containerized HTTP service through the organization's supported TrueFoundry workflow; verify readiness, networking and revisions; and diagnose a failed rollout with evidence and rollback.

## 2. Deployment architecture

```text
Developer / CI pipeline
   │  immutable image digest + deployment configuration
   ▼
TrueFoundry control plane (workspace, application, deployment)
   │  reconciled intent through installed cluster integration
   ▼
tfy-agent / cluster integration (architecture varies by release)
   ▼
Kubernetes namespace
   ├── Deployment / ReplicaSet / Pods (where applicable)
   ├── ConfigMap / Secret references (as configured)
   ├── Service and ingress/route (when configured)
   ├── ServiceAccount / workload identity
   └── Events / probes / resource limits / rollout history
   ▼
Endpoint → readiness → request → application logs and metrics
```

This is a conceptual mapping. TrueFoundry resource types, generated names, routing and reconciler behavior depend on product version, workload type and configuration. **Inspect the rendered resources rather than assuming exact object names.**

## 3. Prerequisites and deployment contract

- Access to an approved TrueFoundry nonproduction workspace and a mapped Kubernetes cluster/namespace.
- Working `tfy-agent` or the installed cluster integration, appropriate deployment permissions and image pull access.
- Organization-approved HTTP sample image pinned by immutable digest; image exposes a known port and health endpoint.
- Explicit ownership of ServiceAccount, secrets, registry credentials and any public endpoint.
- Namespace resource quota, CPU/memory requests, Pod Security admission and networking policies checked.
- A rollback image digest or prior TrueFoundry deployment revision.

**Example contract (illustrative, not a TrueFoundry API schema):**

| Setting | Example | Validate |
|---|---|---|
| Application | `hello-platform` | Workspace naming and RBAC |
| Image | `registry.example.com/team/hello@sha256:<digest>` | Real approved digest and architecture |
| Container port | `8080` | Application actually listens there |
| Readiness | `GET /ready` | Endpoint returns success only when ready |
| Liveness | `GET /health` | Should not fail during normal startup |
| CPU/memory | 100m / 256Mi requests | Quota, observed baseline |
| Replicas | 1 for disposable lab | Availability/SLO needs |
| Exposure | Internal-only for lab | Routing and authentication policy |
| Config | Non-secret env + Secret references | No plaintext credentials |

Do not treat the table as directly deployable YAML or a TrueFoundry SDK payload.

## 4. Deployment lifecycle and reconciliation

1. **Build:** create tested container image with minimal privileges and explicit health endpoints.
2. **Publish:** push to approved registry, record immutable digest and scan/sign according to policy.
3. **Configure:** set workspace, cluster, environment, port, resources, probes, variables, secret references and exposure policy in the supported TrueFoundry workflow.
4. **Submit:** create/update the deployment through TrueFoundry UI or a version-matched CLI/SDK/CI integration.
5. **Reconcile:** inspect the generated Kubernetes resources and Pod events.
6. **Validate:** confirm Pod Ready, Service routing, endpoint response, logs, metrics and image digest.
7. **Promote:** move to higher environments only after acceptance; use approved release controls.
8. **Rollback:** restore the last known-good TrueFoundry revision/source configuration, not only a manual Kubernetes patch.

TrueFoundry may reconcile and overwrite direct Kubernetes changes. Always identify the **authoritative desired-state source** before troubleshooting.

## 5. Kubernetes inspection workflow [SAFE-READ]

### 5.1 Verify context and ownership

```bash
kubectl config current-context
kubectl get namespaces
kubectl -n <namespace> get deployments,replicasets,pods,services -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Find resources using the actual TrueFoundry deployment metadata and labels. Avoid broad assumptions about naming.

### 5.2 Inspect the rendered deployment

```bash
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> get deployment <deployment> -o yaml
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=150
```

`rollout status` observes progress; it does not initiate a rollout. Check owner references, replica counts, image digest, security context, readiness/liveness/startup probes, resource requests/limits, ServiceAccount, mounted configuration and environment references. Do not print Secret values.

### 5.3 Inspect routing

```bash
kubectl -n <namespace> get services,endpointslices -o wide
kubectl -n <namespace> describe service <service>
```

The existence of a Service does not prove that ready endpoints are populated or reachable. Inspect selectors, endpoint addresses, targetPort, NetworkPolicy and DNS when routing fails. Avoid exposing the lab service publicly.

### 5.4 Optional internal endpoint test

With approval, use local port-forward to a verified Service:

```bash
kubectl -n <namespace> port-forward service/<service> 8080:<service-port>
```

From another terminal:

```bash
curl -fsS http://127.0.0.1:8080/ready
curl -fsS http://127.0.0.1:8080/health
```

Use the actual Service port and routes; a port-forward bypasses some normal ingress/network paths, so it is **not** end-to-end public endpoint validation.

## 6. Kubernetes workload configuration example

This is a **Kubernetes illustration**, not the canonical TrueFoundry deployment API. Prefer deploying through TrueFoundry for the hands-on exercise; use this to interpret rendered objects.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-platform
  namespace: tfy-app-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-platform
  template:
    metadata:
      labels:
        app: hello-platform
    spec:
      containers:
        - name: hello
          image: registry.example.com/team/hello@sha256:<approved-digest>
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            periodSeconds: 10
          startupProbe:
            httpGet:
              path: /health
              port: 8080
            failureThreshold: 30
            periodSeconds: 2
```

Replace the image with a **real approved immutable digest** and verify the application implements the health routes. The example omits ServiceAccount, security context, service and environment-specific admission requirements; it is not production-ready as written.

## 7. Deployment failure matrix

| Symptom | Evidence | First likely boundary | Actionable owner |
|---|---|---|---|
| No Kubernetes objects created | TrueFoundry status, agent health, namespace/RBAC | Control plane → cluster reconciliation | Platform integration |
| `ImagePullBackOff` | Pod events, image name/digest, registry auth | Registry → kubelet image pull | Image/registry owner |
| `CreateContainerConfigError` | Pod events, missing referenced config/secret | Rendered config → container creation | Deployment config |
| `CrashLoopBackOff` | Previous logs, exit code, probe events | Container process/startup | Application owner |
| Pod Running but not Ready | Readiness probe failures, app logs | App health → Service readiness | Application/platform |
| Service has no endpoints | Pod readiness, selector, EndpointSlice | Ready Pod → Service endpoints | Networking/deployment |
| Endpoint 502/503 | Route, Service endpoints, upstream logs | Gateway → Service/Pod | Networking/serving |
| Pod Pending | Scheduler events, quota, node resources, taints | Scheduling | Kubernetes platform |
| Deployment healthy but app error | Application logs, dependency health, tracing | Request → application/dependency | App team |
| Manual `kubectl` fix reverts | TrueFoundry/GitOps reconciliation | Source of truth → desired state | Platform/deployment |

Do not restart `tfy-agent`, alter cluster RBAC, delete Pods or patch deployments in production without change approval.

## 8. TrueFoundry / Kubernetes troubleshooting sequence

1. Confirm workspace, cluster, namespace, application, deployment revision and desired image.
2. Inspect TrueFoundry deployment status and cluster integration health.
3. Find generated Kubernetes controller and verify owner/labels.
4. Check scheduling, image pull, configuration, container startup and probes in order.
5. Verify Service selector, EndpointSlices and route only after Pod readiness.
6. Compare desired and actual image digest, env references, resource settings and probes.
7. Identify the **Lowest Proven Healthy Layer** and **First Failed Transition**.
8. Propose a source-of-truth change, canary verification, rollback and SLO checks.

## 9. Production deployment controls

- **Security:** immutable scanned image, least-privilege identity, restricted secret access, no unnecessary privileged mode, non-root container when supported.
- **Reliability:** readiness/liveness/startup probes tailored to startup behavior; sufficient resources; disruption and rollout strategy appropriate to replicas.
- **Observability:** deployment revision, image digest, Kubernetes events, application logs, request latency/error metrics.
- **Cost:** CPU/memory baseline, replica count and scaling policies, not arbitrary oversized requests.
- **Change management:** record owner, approval, promotion criteria, rollback artifact and user impact.

### Rollback principles

Use the supported TrueFoundry deployment revision or GitOps/IaC source to revert to a known-good image/configuration. A direct `kubectl rollout undo` may be overwritten by TrueFoundry reconciliation and must not be treated as the authoritative rollback mechanism without verifying ownership.

### Acceptance gates

| Gate | Expected evidence |
|---|---|
| G1 | TrueFoundry deployment submitted and reconciled |
| G2 | Kubernetes controller exists with intended image digest |
| G3 | Pod scheduled, running and Ready |
| G4 | Service has ready endpoints |
| G5 | Health and sample application request succeed |
| G6 | Logs and request metrics available |
| G7 | Rollback revision and procedure documented |
| G8 | Lab cleanup confirmed |

## 10. Evidence template

```text
Date/time / Kubernetes context:
TrueFoundry workspace / deployment / revision:
Namespace / generated controller:
Image registry and digest:
ServiceAccount and secret references (names only):
CPU/memory requests and limits:
Probe paths / status:
Pod state and scheduling:
Service / EndpointSlice / route:
Health and request result:
Application logs and monitoring:
Lowest Proven Healthy Layer:
First Failed Transition:
Change owner / rollback revision:
Acceptance gates G1–G8:
Cleanup:
```

## 11. Completion checklist

- [ ] Explain TrueFoundry desired-state reconciliation and Kubernetes resource mapping.
- [ ] Locate the actual controller, Pods, Service and EndpointSlices.
- [ ] Verify immutable image, resources, probes and security configuration.
- [ ] Diagnose rollout failures from Kubernetes events and application logs.
- [ ] Demonstrate or document an approved internal health request.
- [ ] Identify source-of-truth rollback and complete acceptance evidence.

**Hands-on lab:** [5.1 — First Service Deployment Check](../labs/01-truefoundry-first-application-deployment-check.md)  
**Next:** Chapter 5.2 — Container Images, Registries and Supply Chain.

## References

- [TrueFoundry documentation](https://docs.truefoundry.com/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
