# Chapter 12.3 — Hands-on Lab: High Availability, Resilience and Failure Domains

**Part 12 | Approved staging | Live execution pending**

## Mission and safety

Validate TrueFoundry inference availability under node/zone failure assumptions, GPU scheduling constraints, PDBs, streaming drain and regional failover design. Use only synthetic traffic and disposable staging workloads. **[SAFE-READ]** inspect; **[LAB-MUTATE]** approved staging; **[CHANGE-CONTROL]** production or cloud routing changes. Never intentionally drain or fail production GPU nodes.

## Lab 0 — Inventory

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl get deployments -A
```

Record cluster/region, GPU SKU, zones, replicas, node pools, PDBs, TrueFoundry/vLLM version, ingress, model digest, cloud GPU quota, service SLO and approval.

## Lab 1 — Failure-domain map

Map DNS/TLS, load balancer, gateway, TrueFoundry agent, Pods, GPU nodes, zones, model artifacts, cloud IAM, storage and telemetry. Mark shared regional dependencies and single points of failure.

## Lab 2 — N-1 capacity calculation

For each zone record sustainable inference throughput under approved TTFT/TPOT/error SLO. Subtract the largest domain's ready capacity from total. Compare remaining capacity to priority peak demand, including queue and cold-start reserve. Document pass/fail; do not count Pending replicas.

## Lab 3 — Placement and PDB inspection

Inspect spread constraints, Pod affinity, GPU requests, taints, zone labels and PDB disruptionsAllowed. If authorized, apply a disposable staging PDB and placement policy, verify behavior and restore configuration. Do not force eviction to bypass a PDB.

## Lab 4 — Five game days

1. **Node loss tabletop or isolated Pod termination:** confirm surviving capacity and alerts.
2. **Zone loss tabletop:** calculate routing and warm capacity under largest-zone loss.
3. **Streaming drain staging:** verify graceful completion, new request routing and timeout handling.
4. **Cold-start/replacement:** time node provision, image pull, model load and first healthy response.
5. **Regional failover tabletop:** validate independent GPU quota, model artifact, identity, routing, RTO/RPO and failback.

Each game day records approved scope, synthetic traffic, UTC timeline, detection, impact, mitigation, recovery, rollback and reviewer. Tabletop exercises do not count as executed failover.

## Thirty-two failure scenarios

| # | Scenario | Diagnostic focus |
|---|---|---|
| 01 | All replicas on one GPU node | Failure domain |
| 02 | All replicas in one zone | Placement |
| 03 | Zone label missing | Scheduler |
| 04 | Hard spread constraint blocks scheduling | Placement |
| 05 | GPU SKU absent in secondary zone | Capacity |
| 06 | Node NotReady | Recovery |
| 07 | GPU allocatable missing | Device plugin |
| 08 | GPU Pod Pending | Scheduling |
| 09 | Namespace quota prevents replacement | Quota |
| 10 | Cloud quota prevents node launch | Provider |
| 11 | PDB minAvailable too strict | Eviction |
| 12 | PDB selector matches wrong Pods | Policy |
| 13 | Drain exceeds grace period | Streaming |
| 14 | EndpointSlice propagation delay | Routing |
| 15 | Gateway continues routing terminating Pod | Readiness |
| 16 | Stream drops during rollout | Connection drain |
| 17 | Model cold-start exceeds RTO | Warm capacity |
| 18 | Image registry unavailable | Dependency |
| 19 | Model artifact access denied | IAM |
| 20 | Cross-zone storage attachment fails | Storage |
| 21 | DNS failover slow | TTL |
| 22 | TLS certificate missing in target | Security |
| 23 | Load balancer health check misconfigured | Edge |
| 24 | TrueFoundry agent reconciliation stalled | Platform |
| 25 | Kubernetes API degraded | Control plane |
| 26 | GPU driver version drift | Compatibility |
| 27 | Remaining capacity below priority demand | N-1 |
| 28 | Autoscaler consolidates warm reserve | Scaling |
| 29 | Regional model digest mismatch | Integrity |
| 30 | RPO assumptions invalid | Data |
| 31 | Failback shifts traffic prematurely | Change |
| 32 | Monitoring reports no-data as healthy | Detection |

For every scenario capture expected/actual, UTC time, safe evidence, owner, mitigation, rollback and reviewer; unsafe faults are tabletop.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Scope, approvals and safety limits | Pending |
| G2 | End-to-end failure-domain inventory | Pending |
| G3 | Zone/node replica placement verified | Pending |
| G4 | GPU and quota reserve verified | Pending |
| G5 | N-1 capacity measured against SLO | Pending |
| G6 | PDB and voluntary drain behavior validated | Pending |
| G7 | Streaming completion during drain tested | Pending |
| G8 | Model replacement/cold-start timing measured | Pending |
| G9 | Ingress and shared dependencies reviewed | Pending |
| G10 | RTO/RPO and regional failover plan approved | Pending |
| G11 | Recovery, rollback and monitoring validated | Pending |
| G12 | SRE/platform/security sign-off recorded | Pending |

**Pass** requires timestamped executed evidence and reviewer approval. A design-only regional failover must remain labeled tabletop.

**Next:** Chapter 12.4 — Production Releases, Upgrades and Rollback.
