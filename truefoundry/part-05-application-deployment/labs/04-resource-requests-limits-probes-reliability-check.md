# Lab 5.4 — Resource Requests, Limits, Probes and Reliability Check

**Canonical:** [Chapter 5.4](../04-resource-requests-limits-probes-reliability/04-resource-requests-limits-probes-reliability.md)  
**Default:** Production **[SAFE-READ]**. Optional workload deployment or resource/probe changes are **[LAB-MUTATE]** in an approved disposable namespace.

## Objectives

Inventory resource commitments, identify QoS and OOM/eviction evidence, evaluate startup/readiness/liveness probes, correlate CPU throttling and latency, and document safe rollout and rollback.

## 0. Preflight

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods -o wide
kubectl -n <namespace> get resourcequota,limitrange
```

Record TrueFoundry workspace/deployment revision, cluster, namespace and approved scope. Do not patch production Pods or change shared quotas.

## 1. Requests, limits and QoS [SAFE-READ]

```bash
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get pod <pod> -o jsonpath='{.status.qosClass}{"\n"}'
kubectl describe node <node>
```

Fill in:

| Container | CPU request | CPU limit | Memory request | Memory limit | QoS | Node |
|---|---|---|---|---|---|---|
| app | | | | | | |
| sidecar | | | | | | |

Explain why resource requests affect placement while CPU and memory limits affect runtime differently.

## 2. Probe and restart analysis [SAFE-READ]

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" restart="}{.restartCount}{" last="}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{"\n"}{end}'
kubectl -n <namespace> logs <pod> --previous --tail=100
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

`--previous` can fail when no previous terminated container exists; record that as **Not available**. Capture startup, readiness and liveness configuration, observed failures, restart reason and Ready condition.

## 3. CPU and memory telemetry [SAFE-READ]

```bash
kubectl -n <namespace> top pod <pod> --containers
kubectl top node
```

These commands require a metrics API and provide only sampled current usage. For trends, use approved Prometheus/Grafana panels. If available, compare CPU throttled periods, CPU usage, memory working set, memory OOM events and P95/P99 request latency over the same window.

| Signal | Observed | Baseline | Interpretation |
|---|---|---|---|
| CPU usage vs request | | | |
| CPU throttling | | | |
| Memory peak vs limit | | | |
| Restarts / OOMKilled | | | |
| Readiness failures | | | |
| P99 latency / error rate | | | |

## 4. Service availability and rollout [SAFE-READ]

```bash
kubectl -n <namespace> get deployments,pods,pdb -o wide
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get services,endpointslices -o wide
```

Record replicas desired/available, maxSurge/maxUnavailable, PDB availability and ready endpoints. Explain why a PDB does not guarantee survival of node failures or all rolling updates.

## 5. Optional disposable reliability test [LAB-MUTATE]

Use an approved nonproduction TrueFoundry deployment with a known health endpoint and image digest. Change **one parameter at a time** through the supported TrueFoundry workflow, such as a readiness probe path or CPU request, and compare before/after Pod events and health results. Do not deliberately create node pressure or OOM in a shared cluster.

Suggested exercise:
1. Capture current deployment revision, resource/probe settings and readiness baseline.
2. Deploy a disposable service with correct startup/readiness/liveness probes.
3. Verify Ready condition and EndpointSlice membership.
4. In a disposable namespace only, configure a deliberately incorrect readiness path; observe NotReady without disabling liveness.
5. Restore the prior known-good revision through TrueFoundry and verify Ready endpoints and request success.
6. Delete only lab-owned resources and verify cleanup.

Do not assume a specific TrueFoundry CLI schema; follow installed version documentation.

## 6. Eight failure scenarios (tabletop)

| ID | Symptom | Evidence | First failed transition |
|---|---|---|---|
| R1 | CPU request too large | Scheduler events | Capacity → scheduling |
| R2 | Namespace quota exceeded | Admission event, quota | Resource policy → admission |
| R3 | OOMKilled | Last state, memory peaks | Runtime → memory enforcement |
| R4 | CPU throttling with P99 regression | Cgroup metrics, latency | CPU cap → performance |
| R5 | Readiness failure | Probe events, logs | App health → Service |
| R6 | Liveness restart loop | Previous logs, probe config | Probe → kubelet restart |
| R7 | Pod evicted | Node pressure, events | Node resources → eviction |
| R8 | Rollout stalls | Deployment conditions, new Pod Ready | Revision → availability |

For each: record lowest proven healthy layer, actionable owner, minimal approved change, rollback and recovery evidence.

## 7. Recovery and rollback plan

```text
Environment / TrueFoundry revision:
Image digest / workload owner:
Requests and limits before/after:
Probe configuration before/after:
Observed CPU, memory and latency:
Root cause confidence / missing evidence:
Authorized source-of-truth change:
Canary and stop conditions:
Known-good revision / rollback:
Post-rollback Pod Ready / endpoints:
SLO and restart verification:
Lab cleanup:
```

## 8. Acceptance

- [ ] Resource requests, limits, QoS and quota correctly classified.
- [ ] CPU throttling distinguished from CPU utilization.
- [ ] OOMKilled distinguished from generic exit 137 and eviction.
- [ ] Probe behavior and readiness endpoint impact explained.
- [ ] Rollout and PDB limitations understood.
- [ ] Eight failures diagnosed with evidence and safe rollback.
- [ ] No unauthorized production changes; disposable resources cleaned up.

**Next:** Chapter 5.5 — Services, Endpoints, Routing and Application Connectivity.
