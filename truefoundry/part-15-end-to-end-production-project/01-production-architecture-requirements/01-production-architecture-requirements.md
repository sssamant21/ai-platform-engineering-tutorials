# Chapter 15.1 — End-to-End Production Architecture and Requirements

**Part 15 — TrueFoundry Enterprise Production Project | Capstone 1 of 7**

## Project brief

Design a production-ready, auditable GPU inference platform with Kubernetes, TrueFoundry, NVIDIA GPUs and vLLM. This chapter produces an architecture decision record (ADR), threat model, SLOs, capacity and cost assumptions, release and recovery design, ownership matrix and go/no-go criteria. Subsequent chapters implement and test the design. **Publication is not proof of deployment, security approval or live acceptance.**

### Target system and explicit assumptions

- Cloud: EKS, AKS or GKE; choose **one primary implementation** before provisioning. Differences in GPU availability, workload identity, ingress, storage and autoscaling must be documented.
- TrueFoundry: approved platform/control plane and registered Kubernetes cluster with vendor-supported agent integration; version-specific setup commands and connectivity requirements must be verified against deployed documentation.
- Serving: vLLM behind authenticated gateway with synthetic OpenAI-compatible request tests **only if supported by selected runtime**; immutable model weights, tokenizer, container digest and runtime config.
- Workloads: one staging model and one production model; single tenant initially, extensible to multi-tenant isolation.
- Reliability: at least two failure domains where available, planned N-1 GPU capacity or explicitly documented degraded-mode risk, measured warmup, streaming-safe deployment.
- Observability: customer-path synthetic checks, Kubernetes, TrueFoundry, gateway, vLLM and DCGM telemetry.
- Security: least-privilege identity, private artifact access, approved ingress/egress, TLS, network policy, secret manager and audit.
- FinOps: GPU-hour budget, quota, cost per successful output token, idle/warm spare accounting and teardown safeguards.

## 1. Architecture and data/control-plane separation

```text
Git + reviewed IaC + CI security/provenance
                 │
                 ▼
        TrueFoundry control plane
                 │ agent connectivity/reconciliation
                 ▼
Cloud VPC/VNet → Private Kubernetes GPU cluster
    ├─ Identity / namespaces / policies / secrets
    ├─ tfy-agent (actual installation name/version discovered)
    ├─ GPU node pools / NVIDIA device plugin / DCGM
    ├─ vLLM inference Pods / model artifact storage
    └─ Services → Gateway / TLS / authorized inference clients
                 │
                 ▼
        Synthetic customer-path probes
                 │
     Prometheus / logs / traces / SLO alerting
```

Control plane can fail while existing inference remains healthy; distinguish **new deployment/reconciliation SLO** from **existing customer-serving SLO**. Avoid making serving traffic depend synchronously on a control-plane API unless architecture requires it.

## 2. Requirements and traceability matrix

| ID | Requirement | Acceptance evidence | Owner |
|---|---|---|---|
| FR-01 | Deploy immutable model and tokenizer | digests and reconciled revision | Model/platform |
| FR-02 | Authenticated inference and streaming | synthetic request + complete SSE | App/security |
| FR-03 | Canary, promotion and rollback | staging release and recovery drill | Release |
| FR-04 | Versioned IaC and GitOps | PR, plan, policy and audit trail | Platform |
| NFR-01 | Availability SLO | approved eligible requests and error budget | SRE |
| NFR-02 | TTFT and TPOT targets | baseline by prompt/output token cohort | Model SRE |
| NFR-03 | Capacity and N-1 | schedulable GPU slots + measured load | Cloud SRE |
| NFR-04 | Security isolation | IAM/RBAC/policy/secret review | Security |
| NFR-05 | Observability and paging | synthetic, GPU, vLLM, agent alert drills | On-call |
| NFR-06 | RTO/RPO and DR | approved scenario-based recovery tests | SRE |
| NFR-07 | FinOps guardrails | budget alerts and cost/token baseline | FinOps |
| NFR-08 | Compliance and retention | data classification and evidence controls | Governance |

SLO targets, RTO/RPO and budgets must be explicitly approved for the chosen business service; this chapter does not invent contractual numbers.

## 3. Capacity model and sizing worksheet

Record model weights size, quantization, tokenizer, max context, KV-cache footprint, runtime overhead, GPU SKU/VRAM, tensor parallelism, replicas, peak input/output tokens, request arrival rate and measured throughput. **Do not estimate production concurrency from model weight size alone.**

For one-GPU replicas:

```text
schedulable_slots = sum(node.allocatable[nvidia.com/gpu]) - sum(pod.requests[nvidia.com/gpu])
steady_GPU_demand = replicas * GPUs_per_replica
rolling_overlap = old_replica_GPU_demand + new_replica_GPU_demand
```

Actual allocation must account for per-node packing, MIG resource type, taints, AZ placement, multi-GPU topology, quotas, cloud GPU supply and model warmup. N-1 means surviving loss of the specified failure domain under the approved SLO or a documented degraded-mode objective. Include rolling deployment overlap, emergency headroom and reserved GPU-hours.

## 4. Network, trust and security design

Define ingress identity, TLS termination, private vs public endpoint, gateway rate limiting, tenant isolation, DNS, egress to control plane/registries/model storage, cluster identity and Kubernetes service accounts. Require explicit trust boundaries and data flow diagrams. Map secret rotation, token expiration, logging redaction, image/model provenance, vulnerability scanning and audit retention. No plaintext keys in Git, Helm values or incident bundles.

### Threat model

| Threat | Control | Validation |
|---|---|---|
| Unauthenticated inference | identity-aware gateway | denied unauthenticated synthetic request |
| Cross-tenant data access | namespaces, RBAC, routing and application auth | authorized isolation test |
| Secret exposure | secret manager, redaction, rotation | config/log review |
| Model supply-chain tampering | immutable digests and artifact ACL | provenance check |
| GPU resource exhaustion | quotas, rate limits, budgets | bounded load test |
| Egress exfiltration | network policy/firewall | approved egress test |
| Platform drift | authoritative TrueFoundry/GitOps reconciliation | drift exercise |
| Incomplete streaming | proxy/drain config, SSE probes | full-stream acceptance |

## 5. Release and recovery architecture

- Build and attest image/model revision; pin tokenizer, vLLM, CUDA and GPU compatibility.
- Validate staging startup, output contract, TTFT/TPOT, SSE completion, GPU memory and security.
- Use vendor-supported TrueFoundry deployment promotion and rollback; avoid out-of-band patches.
- Set canary SLO abort gates and streaming-safe termination; ensure old artifact remains available.
- Design multi-zone placement and measured warm capacity; check cloud quotas and autoscaler provisioning.
- Define incident command, communications, RTO, rollback owner, evidence retention and RCA/COA workflow.

## 6. Observability and operational SLO design

Customer SLIs: eligible success, p95 TTFT, p95 TPOT, complete SSE streams, HTTP 429/5xx and queue wait. Infrastructure SLIs: GPU allocatable/requests, device-plugin and DCGM freshness, XID/ECC, Kubernetes Pod/node health, autoscaler limits. Control plane SLIs: agent connectivity and desired-to-observed reconciliation age where supported. Define scrape/metric names only after version verification. Alert on sustained customer SLO breaches, telemetry gaps, severe GPU errors and reconciliation failures with clear ownership and runbook links.

## 7. Cost, quota and operational limits

Estimate GPU-hours for steady replicas, rolling overlap, N-1 reserve, staging load and recovery game days. Record cloud quota, max node pool size, subnet IPs, AZ GPU offerings, spot/on-demand tradeoffs, model artifact storage, observability and network egress. Cost per successful 1,000 output tokens should include idle warm capacity and failed/retried work. Require spend cap, budget alarms and approved cleanup.

## 8. Architecture decision records

Create ADRs for: cloud/region and zones; TrueFoundry control-plane topology; GPU SKU and autoscaler; model/runtime; gateway and identity; secrets and egress; metrics/logs/traces; release strategy; HA/DR; FinOps. Each ADR states context, options, decision, tradeoffs, rejected alternatives, rollback and owner.

## 9. Implementation work packages

| Chapter | Delivery | Dependency |
|---|---|---|
| 15.1 | Architecture and requirements | business objectives |
| 15.2 | Kubernetes/GPU infrastructure foundation | approved ADRs |
| 15.3 | TrueFoundry cluster integration and secure model serving | cluster and identity |
| 15.4 | Networking, security and GitOps delivery | serving baseline |
| 15.5 | Observability, SLOs and load/capacity validation | deployed staging |
| 15.6 | Failure injection, incident response and recovery | monitoring and rollback |
| 15.7 | Production readiness and final enterprise acceptance | all evidence |

## 10. Fourteen architecture runbooks

1. Requirements discovery and service ownership.
2. Data/control-plane mapping.
3. GPU workload characterization.
4. Model and runtime compatibility.
5. Kubernetes/cloud placement.
6. TrueFoundry agent and reconciliation topology.
7. Network and identity trust boundaries.
8. Secrets and supply-chain security.
9. Canary and rollback design.
10. Streaming and graceful drain.
11. SLO/error-budget specification.
12. N-1 and RTO/RPO design.
13. Cost/quota risk review.
14. Architecture acceptance and handoff.

## 11. Thirty-six architecture failure scenarios

01 unclear owner; 02 missing SLO; 03 unapproved cloud region; 04 single-AZ risk; 05 quota shortfall; 06 no N-1; 07 wrong GPU SKU; 08 VRAM undersized; 09 tokenizer mismatch; 10 CUDA incompatibility; 11 GPU plugin missing; 12 agent egress blocked; 13 stale reconciliation; 14 RBAC overprivileged; 15 workload identity denied; 16 plaintext secret; 17 public inference without auth; 18 cross-tenant access; 19 unapproved egress; 20 unsigned image; 21 mutable model tag; 22 no staging; 23 rollback artifact missing; 24 no canary; 25 streaming truncation; 26 probe too permissive; 27 TTFT target undefined; 28 TPOT target undefined; 29 missing DCGM; 30 metrics stale; 31 paging absent; 32 no RTO/RPO; 33 no DR test; 34 GPU budget exceeded; 35 missing teardown; 36 no reviewer sign-off.

## 12. Four architecture game days (tabletop)

1. Region/AZ GPU shortage with failed N-1 objective.
2. TrueFoundry control-plane outage while serving stays healthy.
3. Unauthorized inference access and secret exposure scenario.
4. Failed canary/rollback with incomplete SSE streams.

Each captures UTC timeline, owner, evidence, expected/actual, blast radius, decision, rollback, security review and follow-up.

## 13. Fourteen acceptance gates

G1 approved business scope; G2 architecture diagram; G3 immutable model/runtime; G4 GPU sizing and N-1; G5 cloud quota and AZ; G6 TrueFoundry integration; G7 gateway/identity; G8 network/secret security; G9 release/rollback; G10 synthetic TTFT/TPOT/SSE SLO; G11 telemetry/alerting; G12 RTO/RPO and DR; G13 cost/spend cap; G14 ADR/reviewer sign-off.

See the [hands-on design lab](../labs/01-production-architecture-requirements-check.md). All gates remain pending until actual review.

**Next:** Chapter 15.2 — Kubernetes and GPU Infrastructure Foundation.
