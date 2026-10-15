# 5.4 — Resource Requests, Limits, Probes and Workload Reliability

**Part:** 05 — Application Deployment · **Chapter:** 4 of 7  
**Audience:** TrueFoundry platform engineers, application owners and Kubernetes SREs  
**Safety:** Diagnostics are **[SAFE-READ]**. Resource, probe, replica and rollout changes require **[CHANGE-CONTROL]**; experiments belong in a disposable nonproduction environment.

## 1. Learning outcomes

Size CPU and memory requests/limits using workload evidence; understand Kubernetes QoS, CPU throttling, OOMKilled and eviction; configure startup, readiness and liveness probes correctly; investigate unstable TrueFoundry deployments; and validate availability, rollout and rollback under controlled conditions.

## 2. Reliability architecture

```text
TrueFoundry application deployment configuration
           ↓
Rendered Kubernetes Deployment / Pod template
           ├─ CPU/memory requests and limits
           ├─ startup/readiness/liveness probes
           ├─ replica count / rolling update strategy
           ├─ service account / security policy
           └─ topology / scheduling constraints
           ↓
Scheduler → node resource availability
           ↓
Kubelet → cgroup enforcement / probes / restarts
           ↓
Ready Pods → Service endpoints → request success / SLO
```

TrueFoundry may reconcile rendered Kubernetes objects. Modify the authoritative TrueFoundry or GitOps configuration, not an ad-hoc production Pod patch.

## 3. Requests versus limits

| Setting | Purpose | Failure mode |
|---|---|---|
| CPU request | Scheduler reservation and relative CPU share | Oversizing strands capacity; undersizing risks contention |
| CPU limit | CPU bandwidth cap | Sustained throttling can increase latency |
| Memory request | Scheduler reservation, QoS/eviction input | Undersizing increases eviction risk |
| Memory limit | Memory ceiling enforced by cgroup | Exceeding limit can trigger OOM kill |
| Ephemeral storage request/limit | Local temporary storage accounting | Disk pressure / eviction |
| GPU extended resource | Device-plugin allocation | Pending if insufficient GPU; see Part 04 |

Requests are **not** dedicated physical CPU cores in the ordinary Kubernetes CPU scheduling model. CPU limits may be omitted when policy permits and contention risk is controlled; memory limits require workload-specific sizing and monitoring.

### Example resource budget

```yaml
resources:
  requests:
    cpu: "250m"
    memory: 512Mi
  limits:
    cpu: "1000m"
    memory: 1Gi
```

These are illustrative values, not universal production recommendations. Size from P95/P99 load tests, startup peaks, runtime memory behavior and node capacity.

## 4. QoS classes and eviction

- **Guaranteed:** every container has equal CPU request/limit and equal memory request/limit (including relevant init-container semantics); strongest QoS classification but not immunity from all failures.
- **Burstable:** at least one CPU/memory request or limit, but not Guaranteed.
- **BestEffort:** no CPU/memory requests or limits for any container.

QoS influences eviction and OOM handling but does not replace priority, resource quota or node pressure investigation. An OOMKilled container may be due to its own memory cgroup limit or node-level memory conditions; inspect evidence.

```bash
kubectl -n <namespace> get pod <pod> -o jsonpath='{.status.qosClass}{"\n"}'
kubectl -n <namespace> describe pod <pod>
kubectl describe node <node>
```

## 5. CPU throttling and memory pressure

**CPU:** sustained latency with throttling may indicate a restrictive CPU limit, heavy concurrency, noisy neighbors or inefficient code. Correlate `container_cpu_cfs_throttled_periods_total` / `container_cpu_cfs_periods_total` where available, CPU usage, requests, limits and request latency. cgroup version and metric availability differ; do not assume the metrics exist.

**Memory:** `OOMKilled` often appears with exit code 137, but exit 137 alone does not prove memory exhaustion. Inspect Pod last state, events, node pressure and memory working set. Distinguish container memory limit from application heap, file cache and GPU VRAM.

```bash
kubectl -n <namespace> get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" restart="}{.restartCount}{" last="}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{"\n"}{end}'
kubectl -n <namespace> logs <pod> --previous --tail=150
kubectl -n <namespace> top pod <pod> --containers
kubectl top node
```

`kubectl top` requires Metrics Server or compatible metrics API. It shows current sampled usage, not historical peaks; use Prometheus/other telemetry for trends.

## 6. Startup, readiness and liveness probes

| Probe | Meaning | Effect of failure |
|---|---|---|
| Startup | Has slow-starting application initialized? | Repeated failures can restart container; suppresses other probes until success |
| Readiness | Can the Pod receive traffic now? | Removes Pod from ready Service endpoints |
| Liveness | Is process stuck and needs restart? | Repeated failures restart container |

Do not make liveness depend on an optional remote service or transient downstream latency: this can create cascading restarts. Readiness should reflect whether the instance can safely serve traffic. Startup should accommodate real initialization time without masking permanent failure indefinitely.

### Example probes (illustrative Pod-template fragment)

```yaml
containers:
  - name: app
    image: registry.example.com/team/app@sha256:<approved-digest>
    ports:
      - containerPort: 8080
    resources:
      requests:
        cpu: "250m"
        memory: 512Mi
      limits:
        memory: 1Gi
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 5
      failureThreshold: 24
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 3
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 10
      timeoutSeconds: 2
      failureThreshold: 3
```

Startup allows approximately 120 seconds before failure based on period and threshold; actual probe scheduling/timing may vary. Validate endpoints and load characteristics. This is not a full TrueFoundry API object.

## 7. Replicas, availability and rollout

A single replica cannot provide service continuity through every restart or node outage. Production replica count, rolling-update `maxUnavailable`/`maxSurge`, topology spread, readiness and disruption budgets must align with SLO and capacity.

A PodDisruptionBudget (PDB) constrains certain voluntary evictions; it does **not** protect against node failures, OOM or all rolling-update behavior. It also does not create spare capacity.

```bash
kubectl -n <namespace> get deployments,pods,pdb -o wide
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get endpointslices -o wide
```

## 8. Resource and reliability inspection [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get resourcequota,limitrange
kubectl -n <namespace> get deployment <deployment> -o yaml
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> logs <pod> --previous --tail=150
kubectl describe node <node>
```

Inspect the rendered Pod resources and probe configuration, rollout strategy, Pod conditions, events, restart counts, last termination reason, node pressure and ready Service endpoints. Do not print Secret values or confidential application payloads.

## 9. Troubleshooting matrix

| Symptom | Evidence | Likely boundary | First action |
|---|---|---|---|
| Pending Pod, `Insufficient cpu` | Scheduler events, node allocations | Request → scheduler | Check capacity and request sizing |
| Pending Pod, quota exceeded | Admission events, ResourceQuota | Namespace quota | Verify budget/owner |
| `OOMKilled` | Container last state, memory telemetry | Memory limit/runtime | Compare peaks and limit |
| Latency spike, high CPU throttling | CPU metrics, P99 | CPU limit → runtime | Validate limit and load |
| Running but not Ready | Readiness events, app logs | App health → Service | Check endpoint and dependencies |
| Repeated probe restarts | Liveness/startup failures | Probe → kubelet | Check thresholds and app startup |
| Evicted Pod | Node pressure, Pod events | Node resources → eviction | Inspect pressure source |
| Rollout stalls | Deployment conditions, new Pod readiness | New revision → Ready | Compare old/new image and probes |
| Service 503 | EndpointSlice, Pod readiness | Ready Pods → Service | Check ready endpoints |
| RestartCount rises, no OOM | Previous logs, exit status | App/process or probe | Diagnose exit reason |

Do not “fix” repeated restarts by disabling probes, setting unlimited memory, deleting other workloads or scaling production blindly.

## 10. Reliability tuning workflow

1. Capture baseline load, CPU/memory utilization, restart rate, P95/P99 latency and error budget.
2. Separate steady-state from startup and burst demand.
3. Choose requests based on observed demand and scheduling objectives; test limits against peaks.
4. Configure probes based on measured startup and real readiness semantics.
5. Canary the authoritative TrueFoundry configuration in nonproduction.
6. Validate rollouts, dependency behavior, capacity and request success under load.
7. Promote only with approved change window, rollback revision and SLO monitoring.
8. Reassess after code, runtime, node type or traffic-pattern changes.

## 11. Incident recovery and rollback

**Incident record:**

```text
Timestamp / cluster / TrueFoundry workspace:
Deployment revision and image digest:
Pod / node / namespace:
CPU/memory requests and limits:
QoS class / quota:
Pod state / restart count / last exit:
Probe configuration and failures:
Node pressure / eviction:
CPU throttling / memory peaks:
Service endpoints / request latency and errors:
Lowest Proven Healthy Layer:
First Failed Transition:
Proposed authoritative configuration change:
Approval / canary / rollback:
Post-change acceptance:
```

**Rollback:** restore the previous known-good TrueFoundry deployment revision or GitOps configuration, verify rendered Pod resources/probes, Ready endpoints and application SLO. Direct `kubectl` changes may be overwritten by reconciliation.

## 12. Acceptance gates

| Gate | Required evidence |
|---|---|
| G1 | Resource requests/limits reflect measured workload |
| G2 | QoS, quota and node scheduling are understood |
| G3 | Startup/readiness/liveness endpoints behave correctly |
| G4 | Pod restarts/OOM/evictions are diagnosed from evidence |
| G5 | Rolling update preserves agreed availability |
| G6 | CPU throttling and memory telemetry correlated with SLO |
| G7 | Authoritative rollback documented and validated |
| G8 | Lab cleanup and production safety confirmed |

**Lab:** [5.4 — Resource and Probe Reliability Check](../labs/04-resource-requests-limits-probes-reliability-check.md)  
**Next:** Chapter 5.5 — Services, Endpoints, Routing and Application Connectivity.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Pod QoS](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
- [Kubernetes Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes Configure Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes Node Pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
