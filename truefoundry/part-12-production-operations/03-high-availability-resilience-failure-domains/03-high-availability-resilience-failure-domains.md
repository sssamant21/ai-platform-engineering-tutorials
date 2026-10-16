# Chapter 12.3 — High Availability, Resilience and Failure Domains

**Part 12 — Production Operations | TrueFoundry SRE canonical tutorial**

## Objectives and safety

Design and validate high availability (HA) for TrueFoundry-managed inference workloads across Pods, GPU nodes, availability zones, Kubernetes control-plane dependencies, cloud networking, model artifacts and observability. Define explicit failure domains, availability targets, capacity reserve, RTO/RPO and recovery runbooks.

**[SAFE-READ]** inspect current cluster state. **[LAB-MUTATE]** approved disposable staging. **[CHANGE-CONTROL]** production routing, node drain, replica changes, failover, cluster upgrades or model rollback. Never deliberately fail a production GPU node or disrupt customer inference. Verify the TrueFoundry edition, deployed agent, Kubernetes/cloud platform, vLLM engine and available traffic-management features before applying examples.

## 1. Failure-domain model

```text
Client
  → DNS / certificate / public or private edge
  → cloud load balancer / ingress / gateway
  → TrueFoundry inference deployment
       ├─ zone A: node A1 → Pod/model replica A
       ├─ zone B: node B1 → Pod/model replica B
       └─ zone C: node C1 → Pod/model replica C
  → model artifacts / object storage / secrets / IAM
  → monitoring and on-call escalation
```

Replica count alone does not imply HA. Multiple Pods on one GPU node, node group or availability zone share a failure domain. Shared ingress, regional cloud quota, artifact storage, identity provider, GPU driver versions and DNS are also common-mode dependencies.

**Scope boundaries:** multi-AZ within a region protects some zone/node failures; multi-region requires separate network, model artifact, identity, data, capacity and routing design. Do not equate multi-zone with regional disaster recovery.

## 2. Reliability targets and budgets

Define customer-visible availability SLI (eligible successful requests / eligible requests), streaming completion, TTFT and latency objectives. State rolling window, exclusions, retry semantics and how no-data is treated. Record **RTO** (target time to restore service) and **RPO** (acceptable data loss) for stateful components; stateless inference may have near-zero application data RPO but still depend on versioned model artifacts, configuration and audit/usage records.

For a 99.9% request-based SLO, error budget is 0.1% of eligible requests over the specified window. Do not treat this as a fixed number of downtime minutes without a time-based SLI.

## 3. Capacity reserve for N-1 and zone failure

For each failure domain, compute ready capacity that remains after the **largest** domain fails:

```text
remaining_ready_capacity =
    total_ready_slo_constrained_capacity
    - largest_failure_domain_ready_capacity

pass only if remaining_ready_capacity >= required_priority_demand
```

Example: three zones each provide 400 sustainable output tokens/sec under TTFT/TPOT SLOs. A one-zone failure leaves 800 tokens/sec. If required priority demand is 900 tokens/sec, this design fails the N-1 test despite having three zones. Add warm capacity, reduce admission demand or define a verified fallback.

Account for GPU memory fit, model cold-start, cloud SKU/region quota, pending node time, zone-balanced replicas, node daemon overhead, PDBs and drain of long-lived streams. Cold standby is not the same as immediately available capacity.

## 4. Kubernetes scheduling and placement [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get deployments,statefulsets -A
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl get nodes -o custom-columns='NAME:.metadata.name,ZONE:.metadata.labels.topology\\.kubernetes\\.io/zone'
```

Inspect `topologySpreadConstraints`, node/pod affinity, taints/tolerations, GPU resource requests, zone availability and resource quota. Avoid hard anti-affinity when GPU scarcity would make all replicas Pending; document whether a constraint is hard or soft.

Illustrative workload fragment; integrate with the actual deployment and test scheduling first:

```yaml
spec:
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: inference
      terminationGracePeriodSeconds: 180
```

`DoNotSchedule` can block capacity in an undersupplied zone. Review `minDomains` and version-specific scheduling behavior if used; a soft `ScheduleAnyway` policy may be preferable with explicit risk acceptance. Labels must match the actual Pods.

## 5. PDBs and planned disruption

A PodDisruptionBudget limits *voluntary* evictions through the Eviction API; it does not prevent involuntary node failure or guarantee an application's health. For three healthy replicas, an illustrative `minAvailable: 2` can allow one voluntary disruption, subject to actual readiness and policy. Verify that the node drain, upgrade and autoscaler use the Eviction API and that sufficient GPU capacity exists elsewhere.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: inference-pdb
  namespace: inference-staging
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: inference
```

Do not apply a PDB with `minAvailable` equal to the current replica count and expect smooth maintenance. Avoid bypassing PDBs with forced deletes except under separately approved incident procedures.

## 6. Streaming drain and readiness

For long-running inference streams, remove the Pod from new routing before termination, honor termination grace, allow active streams to finish within an approved deadline and verify the gateway's connection-draining semantics. PreStop hooks, readiness probes, EndpointSlice updates and external load-balancer propagation can race; test actual request paths. Record what happens when the deadline is exceeded and whether clients can safely retry.

Startup/readiness probes must accommodate model loading and GPU initialization. Liveness probes should detect deadlocks, not restart busy but healthy model servers.

## 7. Multi-AZ and regional recovery

Validate load-balancer health checks, DNS/TLS, zone-aware storage/PVC behavior, image registry access, model-artifact integrity, workload identity and cloud GPU quota in every recovery zone. For regional failover, define independent target-region capacity, model/config synchronization, data retention, encryption keys, DNS TTL, routing controls, RTO/RPO and explicit failback.

Avoid split-brain operational control: assign one active change authority and verify that automation will not route incompatible model revisions simultaneously unless explicitly designed.

## 8. Observability and incident detection

Dashboard per-zone ready replicas, GPU allocatable/allocated, Pending Pods, PDB disruptionsAllowed, deployment rollout, gateway errors, stream completion, TTFT/TPOT, queue, model warmup, cloud quota and telemetry coverage. Alert on loss of redundancy **before** the customer SLO fails, but page according to documented urgency. No-data must not be displayed as healthy.

## 9. Twelve operational runbooks

1. **GPU node loss:** identify affected Pods, device health, surviving capacity and placement blockers.
2. **AZ outage:** evaluate remaining ready throughput, ingress routing, quota and priority admission.
3. **PDB blocks maintenance:** inspect ready replicas, desired availability and available replacement GPU capacity.
4. **Replica co-location:** review spread rules, node selectors, GPU pools and failure-domain labels.
5. **Model cold-start delays recovery:** inspect artifact fetch, image pull, startup probe and warm pools.
6. **Streaming requests dropped during drain:** inspect EndpointSlice propagation, gateway timeouts and grace period.
7. **Regional cloud GPU quota exhausted:** review SKU alternatives and preapproved reserved capacity.
8. **Ingress/load balancer unhealthy:** inspect target groups, health checks, DNS/TLS and gateway logs.
9. **Model artifact store unavailable:** inspect cache, permissions, regional replication and digest validation.
10. **Control-plane or agent reconciliation failure:** inspect Kubernetes API, TrueFoundry agent status and safe manual controls.
11. **Cross-region failover incomplete:** inspect identity, networking, model revision, telemetry and RTO.
12. **Failback causes regression:** verify traffic shift, version consistency, warm capacity and rollback.

## 10. Acceptance criteria

Approve only with an explicit failure-domain map, measured N-1 capacity, tested placement/PDBs, streaming-drain behavior, synthetic failover evidence, target RTO/RPO, routing/failback plan, alert ownership and signed risk register. A successful Pod restart does not prove end-to-end HA.

Complete the [companion lab](../labs/03-high-availability-resilience-failure-domains-check.md): **32 failure scenarios, five game days and 12 acceptance gates**. Formal technical review and live execution pending.

**Next:** Chapter 12.4 — Production Releases, Upgrades and Rollback.
