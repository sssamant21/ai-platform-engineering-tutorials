# Chapter 002 — Production LLM Service Architecture and SRE Responsibilities

**Part 01 — AI/LLM SRE Fundamentals | Chapter 002 of 100**

## Learning outcomes

Design and review an end-to-end LLM inference service, distinguish control plane and data plane, define RACI and service ownership, identify dependency and failure boundaries, construct customer-centered SLOs, and build actionable operational runbooks and an architecture acceptance checklist.

**Publication status:** tutorial and companion lab published; formal technical/security review, live execution and production acceptance are pending. Commands below are examples requiring environment-specific validation.

## 1. Reference architecture and trust boundaries

```text
                External / internal clients
                           │
          [WAF, TLS, DNS, authentication]
                           │
       API gateway: routing, token budget, quotas
                           │
        [tenant isolation / policy boundary]
                           │
           Kubernetes Ingress / Gateway
                           │
              Service / EndpointSlice
                           │
       ┌───────────────────┴───────────────────┐
       │ vLLM / Triton / Ray Serve / KServe    │
       │ tokenizer → prefill → KV → decode    │
       └───────────────────┬───────────────────┘
                           │
          GPU nodes: driver, CUDA, DCGM, VRAM
                           │
            model registry / object store
                           │
       ┌───────────────────┴───────────────────┐
       │ Telemetry / audit / synthetic probes │
       │ Prometheus / OTel / logs / alerts    │
       └───────────────────────────────────────┘

Control plane: GitOps + CI/CD + model registry + platform
              operator/TrueFoundry agent + Kubernetes API
Data plane: client → gateway → serving pods → tokens
```

**Critical distinction:** control-plane reconciliation failures may block deployment updates without immediately affecting already-running inference. Never equate an agent dashboard status with customer-path availability.

### Dependency graph

| Dependency | Serving impact | Owner | Required fallback |
|---|---|---|---|
| DNS/TLS | endpoint unreachable, handshake failures | network/security | tested renewal and rollback |
| API gateway | 401/403/429/502/503/504, SSE truncation | API/platform | traffic drain, prior policy |
| Kubernetes Service | no ready endpoints | Kubernetes SRE | restore healthy endpoints |
| Inference runtime | queue, OOM, latency, incorrect tokens | model serving | compatible rollback |
| GPU driver/plugin | Pending/CrashLoop, no CUDA | GPU platform | verified runtime matrix |
| Model artifact store | cold starts fail, new replicas blocked | ML platform | immutable cached/replicated artifact |
| Cloud GPU quota | unable to scale | cloud/FinOps | reserved capacity/degraded mode |
| GitOps/TrueFoundry | reconciliation stalls/drift | platform | freeze rollout, preserve serving |
| Observability | blind customer impact | SRE | independent synthetic probes |

## 2. Control plane versus data plane

The **data plane** is the latency-sensitive path producing inference responses. The **control plane** manages desired deployments, policy, configuration, routing and lifecycle. Different systems may implement the boundaries differently: inspect the actual deployed architecture rather than assuming every TrueFoundry, KServe or Ray Serve control component sits on the request path.

Design principles:
- Keep serving available when safe during transient control-plane loss.
- Never create two authoritative reconcilers for the same Kubernetes objects.
- Pin container image digest, model checksum, tokenizer and compatible CUDA/driver matrix.
- Use separate staging and production change permissions and quotas.
- Do not rely on mutable latest tags or unversioned external model downloads for rollback.
- Define behavior when gateway, object storage, DNS, telemetry or autoscaler is unavailable.

## 3. Production SRE responsibility model

| Activity | Accountable | Responsible | Consulted |
|---|---|---|---|
| Customer SLO and eligibility | service owner | SRE + product | ML/platform |
| Model evaluation and quality | ML/model owner | ML engineering | security/product |
| Gateway/TLS/auth | security/API owner | platform engineering | SRE |
| GPU nodes/CUDA/plugin | platform owner | GPU SRE | ML serving |
| Runtime deployment | inference service owner | ML platform/SRE | cloud |
| GitOps rollout/rollback | release owner | platform/SRE | security |
| Quota/capacity/FinOps | cloud owner | SRE + FinOps | service owner |
| Pager/incident command | on-call owner | SRE | all SMEs |
| Security incidents | security lead | security response | SRE/platform |
| DR/RTO/RPO approval | business/service owner | SRE/DR lead | cloud/vendor |

RACI is organization-specific; fill named owners, deputies, escalation and decision rights before accepting the architecture.

## 4. Service level objectives

Define eligible customer requests, complete-stream success, error classes, latency cohorts and observation window. Recommended SLIs include:
1. **Availability:** eligible inference requests completed correctly / eligible attempts.
2. **Streaming completion:** streams with valid final protocol event / eligible streams.
3. **TTFT:** start to first generated output token, by input length and model.
4. **TPOT / ITL:** decode speed and token spacing, by output length and concurrency.
5. **E2E latency:** start to complete usable response.
6. **Model-quality guardrail:** evaluation pass/fail, distinct from infrastructure availability.
7. **GPU capacity:** N-1 safe throughput, VRAM, queue and warmup.
8. **Operational freshness:** synthetic probe, metrics, traces and alert routing.

A 200 response with truncated SSE is not a successful inference. Define whether 429 is user quota enforcement or capacity failure; never silently exclude all 429s.

Example only: if an approved request SLO is 99.9%, the allowed failure fraction is 0.001. Actual targets and exclusions require service-owner approval.

## 5. Infrastructure, security and GPU design

Use at least two failure domains where available and justified; ensure topology spreading, PDBs, autoscaler and cloud quota align with GPU capacity. A two-zone deployment is not automatically N-1 if one zone lacks enough GPUs. Calculate model weights + KV cache + activation/workspace + runtime overhead + fragmentation; benchmark before selecting GPU SKU.

Security architecture must address workload identity, tenant authorization, model artifact ACLs, secrets, egress, audit, encryption, prompt retention, provenance and abuse prevention. Logs and traces should capture opaque request IDs and metrics, not raw customer prompts or PHI.

### Capacity planning worksheet

```text
Model revision / tokenizer / context:
GPU SKU / driver / CUDA / runtime:
Measured safe tokens/s at SLO:
Peak request/s and prompt/output cohort:
Steady required replicas:
Rollout surge GPUs:
N-1 node/zone GPU headroom:
Cloud quota / autoscaler limits:
Model warmup time:
Cost per 1,000 successful output tokens:
Degraded mode / traffic admission policy:
```

## 6. Release topology and change ownership

```text
PR → CI unit/security/policy/eval checks
   → signed immutable release bundle
   → staging GPU smoke + synthetic SSE
   → measured load/latency + security negatives
   → change approval → canary
   → SLO/error-budget analysis
   → gradual promotion or authoritative rollback
   → steady-state verification and handoff
```

Rollback must preserve compatibility of image, driver/CUDA, model, tokenizer, quantization, tensor parallelism and gateway contract. Avoid direct `kubectl rollout undo` for a controller-managed workload unless the owning platform explicitly supports that path.

## 7. Safe architecture discovery

**[SAFE-READ]** only in an approved environment. Inspect before choosing a namespace.

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get namespaces
kubectl get deploy,sts,ds,pods -A -o wide
kubectl get svc,endpointslices,ingress -A
kubectl get networkpolicies -A
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For approved namespace and deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Redact log output. Examine Pod scheduling events, image digests, resource requests, GPU allocatable and device plugin, Service endpoints and readiness. For DCGM and TrueFoundry reconciliation, use approved platform dashboards and version-specific supported tooling.

## 8. Failure-domain triage

```text
Customer synthetic failing?
 ├─ DNS/TLS? → cert expiry, resolution, gateway
 ├─ 401/403? → identity, authz, tenant policy
 ├─ 429? → contractual quota versus overloaded serving
 ├─ 502/503/504? → gateway, Service, endpoints, Pod readiness
 ├─ 200 but incomplete SSE? → buffering, timeouts, drain
 ├─ high TTFT? → queue, prefill, cold start, artifact fetch
 ├─ high TPOT? → decode, GPU contention, KV cache
 ├─ Pod Pending? → taints, quota, topology, cloud GPU stock
 ├─ CrashLoop / CUDA? → driver, toolkit, model compatibility
 ├─ model-quality regression? → eval, tokenizer, config
 └─ missing telemetry? → probes, scrape, collector, paging
```

Contain retries and queue growth before scaling. If there is suspected cross-tenant data exposure, follow security incident response and stop unsafe rollout.

## 9. Eighteen production failure scenarios

| ID | Scenario | First evidence |
|---|---|---|
| F01 | DNS misconfiguration | resolver, synthetic |
| F02 | TLS expired | certificate and handshake |
| F03 | Gateway authentication regression | 401/403 by cohort |
| F04 | Rate-limit misconfiguration | 429 policy, quota |
| F05 | Empty EndpointSlice | Service/endpoints |
| F06 | Readiness prematurely green | synthetic vs readiness |
| F07 | GPU Pod Pending | scheduler events |
| F08 | Missing NVIDIA device plugin | node allocatable |
| F09 | Driver/CUDA mismatch | runtime logs |
| F10 | GPU CUDA OOM | VRAM, vLLM logs |
| F11 | Model artifact ACL denied | storage/identity |
| F12 | Model/tokenizer mismatch | release manifest |
| F13 | TTFT regression | queue/prefill cohort |
| F14 | TPOT regression | decode/GPU metrics |
| F15 | SSE truncated | client protocol probe |
| F16 | TrueFoundry/GitOps conflict | desired/observed drift |
| F17 | Observability or pager outage | scrape and synthetic |
| F18 | GPU quota/zone shortage | cloud limits, pending |

## 10. Twelve operational runbooks

1. Customer-path synthetic and incident declaration.
2. DNS/TLS and gateway availability.
3. Authn/authz and tenant isolation.
4. Rate limit versus saturation.
5. Kubernetes Service/EndpointSlice and readiness.
6. GPU scheduling and node capacity.
7. CUDA/driver/device plugin.
8. Model artifact, image and tokenizer provenance.
9. vLLM TTFT/TPOT and queue/KV cache.
10. SSE completion and proxy drain.
11. TrueFoundry/GitOps reconciliation and rollback.
12. SLO burn, paging, incident communication and RCA/COA.

Each runbook must include severity criteria, owner, read-only diagnostics, containment, change-controlled mutation, rollback, validation and escalation.

## 11. Six architecture game days

1. Gateway 503 with healthy Pods (tabletop).
2. GPU node failure with insufficient N-1 headroom (tabletop).
3. Model artifact store unavailable during cold start (tabletop).
4. TrueFoundry reconciliation unavailable while existing inference serves (tabletop).
5. Complete-stream SLO regression from SSE timeout (approved staging/tabletop).
6. Missing metrics with independent synthetic detection (tabletop).

Do not inject production faults. Require approval, abort conditions, cost ceiling, evidence, cleanup and reviewer.

## 12. Twelve acceptance gates

G01 documented data/control planes; G02 dependency and trust-boundary diagram; G03 named RACI and escalation; G04 approved SLI eligibility; G05 complete SSE success criteria; G06 immutable model/runtime release; G07 GPU N-1 capacity worksheet; G08 identity/tenant/network controls; G09 GitOps/controller ownership; G10 synthetic and telemetry coverage; G11 recovery/rollback/runbooks; G12 independent review.

All gates remain **pending** until evidenced and signed off. Use the [Chapter 002 companion lab](../labs/002-production-llm-service-architecture-sre-responsibilities-check.md).

**Next: Chapter 003 — SLIs, SLOs, Error Budgets and Inference User Journeys.**
