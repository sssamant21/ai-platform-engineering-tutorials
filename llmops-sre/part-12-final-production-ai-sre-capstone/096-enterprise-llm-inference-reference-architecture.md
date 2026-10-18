# Chapter 096 — Enterprise LLM Inference Reference Architecture

**Part 12 — Final Production AI SRE Capstone · Chapter 096 of 100**  
**Status:** Published with offline architecture acceptance lab. Cloud provisioning, performance measurements and independent security/architecture review pending.

## Objectives
Design an enterprise-grade LLM platform with a documented trust model, reproducible serving, GPU capacity, HA/DR, evaluation, observability, cost governance and operational ownership. Produce architecture decision records (ADRs), an environment matrix and measurable production readiness gates.

## 1. Reference architecture
```text
                 Clients / Applications / Agents
                              |
                  WAF / API Gateway / mTLS
                              |
           Identity, RBAC, tenant authorization, quotas
                              |
                 AI gateway / model router
                /          |            \
          guardrails    prompt cache    budget ledger
                \          |            /
                   serving control plane
                TrueFoundry / Ray / KServe
                              |
             Kubernetes + GPU node pools
                 /         |         \
            model A     model B     fallback
                 \         |         /
          signed model + tokenizer artifacts
                              |
              RAG retrieval / reranking
                   |                  |
             vector index       tool broker
                   |                  |
          source documents    external APIs
                              |
              OpenTelemetry + Prometheus
                   |                  |
             SLOs / alerting / audit
                              |
               CI/CD + GitOps + policy
```
**Control plane** reconciles deployments and policy. **Data plane** executes inference. Separate their failure domains and privileges. This architecture is a design pattern, not a claim that a particular product automatically provides all controls.

## 2. Functional and nonfunctional requirements
| Area | Minimum architecture requirement | Evidence |
|---|---|---|
| Authentication | workload identity, mTLS where appropriate, short-lived tokens | access tests |
| Authorization | per-tenant model, retrieval and tool ACLs | negative tests |
| Provenance | immutable signed model, tokenizer, runtime and prompt revisions | verified manifests |
| Availability | zone-spread replicas, tested N-1 capacity, safe retries | failure drills |
| Performance | TTFT, TPOT, full-stream completion, concurrency and throughput | load tests |
| Data protection | encryption, retention, privacy-safe telemetry, regional residency | controls audit |
| Quality/safety | golden evals, canary gating, policy tests and human escalation | release reports |
| Observability | traces, metrics, logs and per-slice SLOs with cardinality controls | dashboards |
| FinOps | token budgets, GPU utilization, unit economics, cost allocation | ledger |
| Recovery | signed artifacts, backups, RPO/RTO, fencing and regional DR | restore drills |

## 3. Environment isolation
Define **dev, staging and production** with separate identities, policies, secrets, quotas and approvals. Use dedicated production namespaces or clusters based on risk and tenant requirements. Do not allow staging credentials or test data to access production model registries or customer data. Enforce network policies, admission controls, workload identity and least-privilege artifact access.

## 4. Deployment topology
- Regional ingress and API gateway enforce identity, token budgets, request limits and route authorization.
- Model router selects approved immutable revisions and applies safe fallback policies.
- Ray Serve/KServe/TrueFoundry or other orchestrators reconcile serving workloads under declared ownership.
- Kubernetes scheduling uses GPU node selectors, taints/tolerations, topology spread, resource requests and device plugin health.
- Model artifacts are pinned by digest and verified with trusted provenance.
- RAG retrieval and tool broker are separately authorized; side effects use idempotency and audit.
- Telemetry captures request outcomes and performance without recording raw sensitive prompts by default.
- CI/CD performs static validation, security policy checks, evaluation gates and staged rollout with rollback.

## 5. GPU and capacity model
Size capacity using measured prefill/decode token mix, context lengths, batch behavior, KV-cache consumption, GPU memory, quantization, p95 TTFT/TPOT, stream completion and expected concurrency. A single requests-per-second benchmark does not capture heterogeneous workloads.

```text
N-1 safe capacity = sum(zone safe capacities) - max(zone safe capacity)
Headroom = N-1 safe capacity - peak forecast demand
```
Reserve headroom for burst, model warmup, node failures and delayed GPU scale-out. Treat fallback to smaller or different models as a **quality and policy change**, not a transparent infrastructure optimization.

## 6. Threat model
| Threat | Control |
|---|---|
| cross-tenant retrieval | authorization at document retrieval, tests and audit |
| prompt injection via retrieved data | trust boundaries, tool allowlists, output validation |
| compromised model artifact | signature/provenance verification and immutability |
| leaked credentials | workload identity, rotation and scoped secrets |
| unbounded token/tool spend | quotas, budgets, bounded retries and circuit breakers |
| unsafe tool side effects | human approval for high-impact actions, idempotency |
| model/prompt regression | evaluation, canary, rollback |
| region outage | multi-zone N-1, DR checkpoints and fencing |

## 7. Infrastructure-as-code and GitOps layout
```text
platform/
  environments/{dev,staging,prod}/
  terraform/{network,identity,gpu,storage,observability}/
  kubernetes/{base,overlays}/
  gitops/{applications,policies}/
  models/{manifests,release-approvals}/
  evals/{golden,adversarial,release-gates}/
  runbooks/{incident,capacity,dr,security}/
  tests/{architecture,integration,load,chaos}/
```
Use versioned modules and immutable container/model digests. Protect production branches and require peer approval for policy or identity changes. Store no plaintext credentials in Git.

## 8. Observability and SLO design
Measure 5xx/429, successful stream completion, p95 TTFT, TPOT, queue age, GPU utilization and memory, KV-cache pressure, retrieval freshness, grounding, tool outcomes, tenant quota denials, model revision, release failures and spend per 1M tokens. Alert on user-impacting SLO burn rates and critical security gates; avoid per-request high-cardinality metric labels.

## 9. Runbooks
**RB-096-A — New region rollout:** verify residency, IAM/KMS, GPU quotas, artifact replica, retrieval data, policy and synthetic canary before traffic.

**RB-096-B — Serving degradation:** identify gateway, router, GPU worker, RAG or tool fault; preserve model identity, cap retries, recover SLO.

**RB-096-C — Security isolation:** halt risky route, revoke affected permissions through approved response, verify tenant ACL and audit.

**RB-096-D — Cost anomaly:** compare token volumes, GPU utilization, fallback routing, retries and reserved capacity; restore bounded admission.

**RB-096-E — DR readiness:** validate N-1 capacity, artifact provenance, per-dependency RPO/RTO, fencing, region IAM and end-to-end recovery.

## 10. Architecture review deliverables
Produce: C4-style context/container diagrams, trust-boundary diagram, dependency graph, failure-mode analysis, capacity worksheet, SLO definitions, RPO/RTO inventory, IAM matrix, data classification, model release policy, cost model, ADRs, threat model, operational ownership, test plan and unresolved-risk register.

## 11. 100-point acceptance rubric
| Domain | Points |
|---|---:|
| Architecture and trust boundaries | 20 |
| GPU capacity, HA and DR | 20 |
| Model, RAG and tool security | 20 |
| Observability, quality and FinOps | 20 |
| IaC/GitOps, runbooks and test evidence | 20 |
| **Total** | **100** |

Educational target >=90/100 with zero critical isolation, provenance, budget or data-integrity violations. Architecture documentation alone does not certify production readiness.

## 12. Companion lab
[Chapter 096 — Enterprise Architecture Acceptance Lab](../labs/096-enterprise-llm-inference-reference-architecture-check.md) validates a synthetic architecture manifest, zone capacity and security/reliability gates using Python and pytest.

## References
- [Kubernetes Production Best Practices](https://kubernetes.io/docs/setup/best-practices/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [OpenTelemetry](https://opentelemetry.io/docs/)
- [Google SRE Workbook](https://sre.google/workbook/)
- [SLSA](https://slsa.dev/)

**Next:** Chapter 097 — Build a Secure Multi-Environment GPU Serving Platform.
