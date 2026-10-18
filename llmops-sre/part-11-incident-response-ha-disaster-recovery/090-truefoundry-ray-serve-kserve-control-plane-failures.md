# Chapter 090 — TrueFoundry, Ray Serve and KServe Control Plane Failures

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 090 of 100**  
**Status:** Published with a synthetic executable lab. Live platform drills, integration tests and independent operational/security review pending.

## Outcomes
Diagnose and recover LLM serving control-plane failures across TrueFoundry, Ray Serve and KServe. Distinguish reconciliation, admission, routing, orchestration and Kubernetes scheduling failures from GPU/data-plane failures. Preserve tenant isolation, model integrity, SLOs and rollback safety.

## 1. Control plane versus data plane

```text
Git / CI / approved deployment configuration
                  |
       TrueFoundry / Ray Serve / KServe
        reconciliation and desired state
                  |
      Kubernetes API / controllers / CRDs
                  |
      scheduler / autoscaler / node pools
                  |
       serving workloads / GPU workers
                  |
       ingress / gateway / inference traffic
```

The control plane configures and manages serving; the data plane serves requests. An unhealthy controller does not necessarily imply existing inference traffic has stopped, and healthy Kubernetes pods do not prove that routes, model loading or stream completion work.

**Product distinctions:** TrueFoundry provides platform-level deployment and operations workflows; Ray Serve uses application deployments and Ray control components; KServe uses Kubernetes-native serving resources and controllers. Actual APIs, CRDs, deployment topology and commands vary by installed version and serving mode. Confirm installed versions and operator documentation before modifying production.

## 2. Fault-domain matrix

| Domain | Symptoms | Evidence | Initial action |
|---|---|---|---|
| TrueFoundry reconciliation | deploy stuck, config drift, revision not progressing | platform deployment status, controller logs, K8s events | compare desired and observed state |
| Ray head/GCS/control components | deployment unavailable, actor placement errors | Ray status/logs, head pod, GCS health, actor failures | preserve logs and check quorum/dependencies |
| Ray Serve controller/proxies | route missing, replica not ready, stalled rollout | Serve deployment status, controller/proxy logs | identify revision and unhealthy replicas |
| KServe controller/webhook | InferenceService not Ready, rejected admission | InferenceService conditions, webhook and controller logs | inspect reconciliation/admission |
| Kubernetes API/etcd | stale reconciliation, watch errors | API latency, controller work queues, etcd health | escalate platform control plane |
| GPU scheduler/device plugin | Pending pods, no allocatable GPU | node allocatable, events, plugin/operator status | validate capacity and runtime |
| networking/certificates | 502/503, DNS or TLS errors | EndpointSlices, Service, Gateway, cert expiry | restore healthy route/cert |
| autoscaling | insufficient replicas or scale-to-zero cold starts | HPA/KPA/KEDA events, metrics adapter | restore bounded capacity |
| artifact/secret mounts | model fails to initialize | init container, volume and credential errors | verify signed artifact and workload identity |

## 3. Incident triage — first 15 minutes

1. Declare incident and severity under Chapter 086; record UTC onset and affected endpoints.
2. Verify external inference success, 429/5xx, TTFT, stream completion and tenant blast radius.
3. Determine whether only new deployments are blocked or existing serving traffic is affected.
4. Record desired revision, observed revision, deployment ID, model digest and recent changes.
5. Inspect Kubernetes control-plane health, relevant CRDs, admission webhooks, controller restarts and work queues.
6. Inspect Ray Serve deployments/actors or KServe InferenceService conditions as appropriate.
7. Check node capacity, GPU device plugin, taints, quotas, PVC/object storage, secrets and service routing.
8. Freeze risky promotions and choose rollback/containment under an approved change process.
9. Validate inference quality, authorization, SLOs and budget after recovery.

## 4. Safe diagnostic commands

```bash
# Read-only discovery; select your approved context/namespace first.
kubectl config current-context
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl get crd | grep -Ei 'serving|ray|inference'
kubectl get deployments -A
kubectl get endpointslices -A
kubectl get nodes -o wide

# KServe (only if CRD installed):
kubectl get inferenceservices -A
kubectl describe inferenceservice <name> -n <namespace>

# Ray/KubeRay (only if CRDs installed):
kubectl get rayclusters,rayservices -A
kubectl describe rayservice <name> -n <namespace>

# Example controller logs; discover exact deployment/namespace first:
kubectl logs -n <controller-namespace> deploy/<controller-deployment> --since=30m
```

Ray Serve application status can also be inspected using the supported Ray dashboard/Serve CLI or APIs for the deployed version; avoid assuming a particular command or endpoint exists. For TrueFoundry, use the approved platform deployment view and supported CLI/API version, correlate with underlying Kubernetes resources, and avoid exposing access tokens.

## 5. Production failure scenarios

### F01 — TrueFoundry desired/observed drift
Deployment reports pending while Kubernetes pods are running. Inspect rollout ownership, reconciliation errors, configuration validation and RBAC. Do not manually edit platform-managed resources without an approved break-glass process; reconciliation may overwrite them.

### F02 — Ray head/GCS disruption
Inspect Ray head pod, GCS connectivity, actor state, network policy and persistent control-plane dependencies. Do not restart the entire Ray cluster as the first mitigation. Assess live replica impact and recover using version-appropriate documented procedures.

### F03 — Ray Serve controller or proxy outage
Inspect Serve deployment health, controller/proxy logs, route registration, worker replicas and rolling update. Compare app config and model artifact digests. Roll back the deployment rather than bypassing tenant or model policies.

### F04 — KServe admission webhook failure
Check webhook service endpoints, certificates, API-server reachability, namespace selectors and controller logs. Do **not** weaken failurePolicy to Ignore merely to restore deployment; this can bypass validation. Restore the admission path and test a safe staging object.

### F05 — KServe InferenceService NotReady
Inspect conditions, predictor pods, model storage initialization, image pulls, GPU requests and ingress route. Verify whether failure is in reconciliation, storage, runtime or data plane before changing settings.

### F06 — Autoscaling and capacity deadlock
Pending GPU replicas may result from quotas, topology constraints, taints, device plugin issues, autoscaler limits or unavailable GPU SKUs. Validate N-1 capacity, disruption budgets and cost ceilings before scaling or draining.

### F07 — Certificate or network regression
Verify API webhook TLS, serving ingress, DNS, NetworkPolicy and EndpointSlice readiness. Restore a trusted certificate or route; never disable authentication or mTLS as a quick fix.

### F08 — Model registry/secret dependency
Check workload identity, approved artifact digest, secret mount and storage availability. Keep artifact signature and tenant authorization gates active during recovery.

## 6. Decision tree

```text
Is live inference failing?
  yes -> protect data plane, mitigate customer impact, check SLO
  no  -> freeze new promotions, preserve serving capacity
        |
Can Kubernetes API and admission function?
  no -> platform escalation; investigate API/webhook/etcd
  yes
        |
Is desired state reconciled?
  no -> platform controller / Ray / KServe status
  yes
        |
Are pods scheduled and GPU devices allocatable?
  no -> capacity / device plugin / quotas / placement
  yes
        |
Do endpoints, routes and synthetic inference pass?
  no -> gateway / network / model runtime
  yes -> validate rollout status and sustained SLO recovery
```

## 7. Runbooks

**RB-090-A — Control-plane outage:** identify which reconciler is unhealthy; preserve logs and revision IDs; halt promotions; protect serving; engage owning platform team; restore controllers through supported procedures.

**RB-090-B — Rollout stuck:** compare desired and observed revision, admission conditions, model initialization, scheduler and readiness; restore last approved signed configuration; validate with canary.

**RB-090-C — Webhook/certificate failure:** inspect webhook endpoints and certificate validity; restore trusted certificates and availability; retest admission and isolation; never bypass policy.

**RB-090-D — Ray Serve disruption:** distinguish head/GCS, controller, proxy and worker faults; use version-matched recovery; validate route registration, actors and end-to-end inference.

**RB-090-E — KServe NotReady:** review InferenceService conditions and child resources, restore dependencies, validate predictor and ingress, then measure TTFT/stream completion.

**RB-090-F — Post-incident recovery:** check multi-tenant isolation, model provenance, p95/TTFT, error rate, retry/budget burn, streaming completion and alert clearance; document RCA/COA.

## 8. Game-day plan
Use disposable staging clusters and synthetic requests. Inject controller unavailability, an invalid webhook certificate, a missing GPU resource, Ray Serve deployment configuration error, model storage denial and ingress endpoint failure. Capture impact and exact revisions, verify rollback and confirm that control-plane recovery does not silently change security policy. Never simulate etcd destruction or production credential revocation in a shared cluster.

## 9. Monitoring
Track reconciliation error rate and age, desired-vs-observed revision lag, controller restart count, webhook failures/latency, Ray actor availability, KServe readiness conditions, unschedulable GPU pods, EndpointSlice ready endpoints, inference 5xx/TTFT, stream completion and deployment rollback rate. Correlate with approved change events; use low-cardinality and privacy-safe labels.

## 10. Acceptance rubric — 100 points

| Domain | Points |
|---|---:|
| Platform fault classification | 20 |
| Kubernetes, Ray, KServe and TrueFoundry diagnosis | 20 |
| Safe mitigation, security and change controls | 20 |
| Recovery SLO and rollout verification | 20 |
| Lab tests, evidence, escalation and RCA/COA | 20 |
| **Total** | **100** |

Target >=90/100, with no bypassed admission/tenant policies, unapproved production mutation or unsafe data-plane outage. The companion simulator is educational; production acceptance requires platform-specific staging drills and reviewer approval.

## 11. Companion lab
[Chapter 090 — Executable Control Plane Failure Lab](../labs/090-truefoundry-ray-serve-kserve-control-plane-failures-check.md) provides a deterministic classifier and release recovery gates.

## References
- [KServe documentation](https://kserve.github.io/website/)
- [Ray Serve documentation](https://docs.ray.io/en/latest/serve/index.html)
- [KubeRay documentation](https://docs.ray.io/en/latest/cluster/kubernetes/index.html)
- [Kubernetes debugging](https://kubernetes.io/docs/tasks/debug/)
- [TrueFoundry documentation](https://docs.truefoundry.com/)

**Next:** Chapter 091 — High Availability, N-1 and Multi-Zone Recovery.
