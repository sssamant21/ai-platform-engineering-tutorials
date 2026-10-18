# Chapter 091 — High Availability, N-1 and Multi-Zone Recovery

**Part 11 — Incident Response, HA and Disaster Recovery · Chapter 091 of 100**  
**Publication:** Tutorial and offline lab published. Real staging failover, GPU-capacity validation, recovery timing and independent review pending.

## Learning objectives
Design and validate zone-resilient LLM serving with N-1 capacity, Kubernetes topology placement, load-balancer failover, bounded inference queues, model warmup, GPU resource availability, and SLO-preserving recovery. Distinguish availability, capacity, recoverability and data consistency.

## 1. Reference architecture
```text
                clients / agents
                       |
            regional AI gateway + WAF
                       |
           routing / budget / auth PDP
                       |
              regional load balancer
             /          |          \
          zone A      zone B       zone C
        inference    inference    inference
        GPU pool     GPU pool     GPU pool
             \          |          /
          model registry + signed artifact cache
                       |
             RAG / tool dependencies
                       |
          telemetry + incident command
```
Independent failure domains matter: multiple replicas on one node or zone are not high availability. Availability requires healthy **serving** replicas, not merely scheduled pods. Recovery depends on model download/warmup, token throughput, gateway quotas, retrieval dependencies and GPU quota.

## 2. Capacity and N-1 mathematics
For zones with safe serving capacities `C_i` (requests/s or output tokens/s, measured at acceptable latency), regional N-1 safe capacity is:
```text
N-1 capacity = sum(C_i) - max(C_i)
Headroom after loss = N-1 capacity - peak demand
Utilization after loss = peak demand / N-1 capacity
```
Use the same throughput unit for demand and capacity. Validate with real token-length mix, prefill/decode workload, concurrency, p95 TTFT/TPOT, quantization, KV cache and model warmup. Do not equate GPU count with interchangeable capacity across SKUs.

**Example:** Three zones each safely serving 120 requests/s yield N-1 capacity 240 requests/s. A 200 requests/s peak leaves 40 requests/s headroom and 83.3% post-failure utilization. At 260 requests/s peak, the design fails N-1 despite healthy three-zone steady state.

## 3. HA control planes and dependencies
- Run sufficient gateway replicas with topology spread and disruption budgets; check LB and DNS health.
- Verify model-serving workers are distributed across failure domains; account for zone-specific GPU availability and node bootstrap time.
- Keep registry/artifact storage resilient and pin signed model/tokenizer digests; use verified local caches when approved.
- Make retrieval/vector index, Redis, PostgreSQL, Kafka and tool services zone resilient or document degraded modes.
- Separate regional failover from zonal failover; cross-region recovery introduces identity, residency, latency and data consistency concerns.
- Preserve tenant ACLs, gateway budgets and signed model provenance during fallback.
- Ensure observability and incident-command channels survive a zone outage.

## 4. Kubernetes deployment example
Adapt this fragment to your controller and GPU hardware. It is **illustrative** and requires actual node labels, sufficient replicas and workload readiness checks.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-example
  namespace: llm-staging
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inference-example
  template:
    metadata:
      labels:
        app: inference-example
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: inference-example
      containers:
        - name: model-server
          image: registry.example.invalid/approved/model-server@sha256:REPLACE_WITH_VERIFIED_DIGEST
          resources:
            requests:
              nvidia.com/gpu: "1"
            limits:
              nvidia.com/gpu: "1"
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
```
A topology-spread constraint alone does not guarantee one replica in each zone or enough capacity after a zone failure. Confirm actual placement, model readiness and node autoscaling limits. Replace the placeholder digest before applying. Configure PDBs, HPA/KEDA and ingress according to tested workload semantics.

## 5. Read-only preflight
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -n llm-staging -o wide
kubectl get deploy,pdb,hpa -n llm-staging
kubectl get events -n llm-staging --sort-by=.lastTimestamp
kubectl describe deploy inference-example -n llm-staging
```
Verify zone labels, per-zone ready endpoints, GPU allocatable resources, model warmup time, PDB coverage, cloud quota and expected scale-out lead time. Keep real secrets, tenant prompts and credentials out of shared incident logs.

## 6. Failure injection procedure — staging only
1. Define peak and burst workloads with input/output token distributions and quality/SLO targets.
2. Measure steady-state per-zone safe throughput and actual GPU saturation.
3. Confirm N-1 arithmetic **before** disruption and obtain incident/game-day approval.
4. Route synthetic load; record baseline success, 429/5xx, p95 TTFT, TPOT, queue age, stream completion and cost.
5. Simulate loss of a serving zone through a **controlled traffic-routing test** or isolated staging infrastructure, not production-wide node deletion.
6. Confirm remaining zones maintain identity checks, retrieval ACLs, token budgets and signed model revisions.
7. Measure detection, traffic diversion, model warmup and time-to-recovery against RTO; verify no unexpected duplicate billing.
8. Restore traffic, compare error budget and reconcile workload/cost metrics.
9. Document failure modes, changes, residual risk and repeatability.

## 7. Troubleshooting matrix
| Failure | Likely cause | Response |
|---|---|---|
| post-failover 429 | N-1 shortfall or quota misconfiguration | shed load fairly, cap retries, add approved capacity |
| 503 during scale-up | GPU shortage or slow model initialization | warm standby, validate node pool and registry |
| cross-zone skew | topology spread, affinity or taints | inspect scheduler events and actual placement |
| 504/TTFT spike | overloaded prefill/queue | reduce concurrency, bound queue and token limits |
| registry unavailable | single-zone storage/control dependency | restore verified cache or resilient registry |
| fallback wrong model | route config drift | pin approved model digest and rollback |
| stream truncation | LB drain/idle timeout | test SSE completion and cancellation |
| cost spike | retries, cross-zone egress, standby GPU | reconcile FinOps ledger and retry policy |

## 8. Production runbooks
**RB-091-A — Zone loss:** declare SEV, verify zone health and surviving capacity, stop risky promotions, direct traffic only to healthy approved serving replicas, monitor SLOs and customer impact.

**RB-091-B — N-1 capacity breach:** enforce bounded admission and tenant fairness, disable recursive retries, scale only when verified GPU quota and cost allow; escalate if no safe capacity exists.

**RB-091-C — Slow warmup:** inspect model artifact download, image pull, driver/device plugin, CUDA initialization and readiness probe; keep unready pods out of service.

**RB-091-D — Restore zone:** verify node/GPU health, artifact signatures, model version, authorization and SLOs; canary traffic before normal balancing.

**RB-091-E — Cross-region escalation:** confirm residency, data consistency, retrieval freshness, secrets/identity and budget controls before shifting regional traffic.

## 9. RPO/RTO and recovery criteria
Stateless inference may not need data restoration, but chat/session state, tool execution, retrieval indexes, quota reservations and audit logs may. Specify RPO separately for each stateful dependency. RTO includes detection, traffic shift, scheduling, model download, warmup and validation. Closure requires sustained end-to-end synthetic and real-user SLO success, not just `Ready` pods.

## 10. Acceptance rubric — 100 points
| Domain | Points |
|---|---:|
| Failure-domain design and N-1 sizing | 20 |
| GPU scheduling, topology and dependency HA | 20 |
| Safe zone-loss drills and escalation | 20 |
| RTO/RPO, inference SLO and security validation | 20 |
| Automated tests, evidence and RCA/COA | 20 |
| **Total** | **100** |

Pass >=90/100 and zero critical tenant isolation, budget, signed-artifact or data-loss gate failures. Offline arithmetic tests do not demonstrate real failover.

## 11. Companion lab
[Chapter 091 — Executable N-1 and Multi-Zone Recovery Lab](../labs/091-high-availability-n-1-multi-zone-recovery-check.md) includes Python capacity, zone-distribution and recovery gates plus pytest tests.

## References
- [Kubernetes Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes Pod Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Google SRE Workbook — Disaster Recovery](https://sre.google/workbook/disaster-recovery/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)

**Next:** Chapter 092 — Model Artifact, Registry and Configuration Recovery.
