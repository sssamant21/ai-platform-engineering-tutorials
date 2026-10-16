# Chapter 12.7 — Production Readiness and Recovery Capstone

**Part 12 — Production Operations | TrueFoundry SRE final capstone**

## Mission

Conduct an evidence-based production readiness review and staged recovery exercise for a TrueFoundry-hosted vLLM inference service on Kubernetes GPU nodes. Integrate capacity, multi-zone resilience, security, releases, FinOps, observability, maintenance and incident command from Chapters 12.1–12.6. Produce a defensible **GO / CONDITIONAL GO / NO-GO** decision with accountable owners.

**Publication is not certification.** This document is a test plan; live execution, technical/security review and acceptance remain pending until real evidence and approvals are recorded.

## Safety and prerequisites

**[SAFE-READ]** inspect Kubernetes, TrueFoundry, cloud quota, telemetry and billing. **[LAB-MUTATE]** only approved disposable staging with synthetic traffic and spend caps. **[CHANGE-CONTROL]** production GPU nodes, routing, model releases, identity, quota, failover and rollback. Do not drain production nodes, generate uncontrolled load or expose prompts, completions, PHI, secrets, model weights or customer identifiers.

Confirm current TrueFoundry/agent, Kubernetes, vLLM, NVIDIA driver/device plugin, GPU SKU, ingress, autoscaler and observability integrations. Use the supported TrueFoundry source-of-truth workflow for changes; do not assume direct Kubernetes mutation persists.

## 1. Target architecture and readiness inventory

```text
Synthetic client / external health probe
                │
         DNS / TLS / ingress
                │
         TrueFoundry deployment
         ├─ zone A GPU pool → vLLM replica A
         ├─ zone B GPU pool → vLLM replica B
         └─ zone C GPU pool → vLLM replica C
                │
        model artifact / identity
                │
    Prometheus / logs / tracing / alerts
                │
        on-call / incident command
```

Map every shared failure domain: gateway, region, GPU quota, model registry, secrets, cluster control plane, cloud networking, storage, metrics backend and on-call escalation. A three-zone picture is not proof of three-zone placement; verify actual topology.

## 2. Production acceptance dossier

| Area | Required evidence |
|---|---|
| Ownership | Service owner, SRE on-call, escalation, maintenance policy |
| Security | RBAC/workload identity, secret rotation, network isolation, image/model provenance, log privacy |
| Availability | Measured eligible-request SLI, TTFT/TPOT, streaming completion, error budget |
| Capacity | Synthetic demand distribution, measured throughput per replica, GPU memory, quotas, warm reserve |
| HA | N-1 capacity after largest zone/node loss, Pod spread, PDB, draining, RTO/RPO |
| Releases | Immutable image/model digests, compatibility matrix, canary, rollback |
| FinOps | Billed/allocated GPU-hours, cost per million output tokens, budget alerts |
| Observability | Metrics, logs, traces, synthetic checks, no-data alerting, paging |
| Incident operations | Runbooks, severity, commander, communications, postmortem |
| Recovery | Reproducible drills, timeline, validated rollback and service restoration |

Use **measured** values with UTC time, cluster/model version, sample size and reviewer. Mark projections, tabletop scenarios and assumptions explicitly; do not substitute them for executed evidence.

## 3. Read-only preflight

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get deployments,statefulsets -A
kubectl get resourcequota,limitrange -A
kubectl get pdb -A
kubectl get hpa -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

Inspect only approved namespaces if cluster-wide access is not authorized. `kubectl top` requires Metrics Server and is not a substitute for DCGM or inference metrics. Verify actual installed custom metrics and TrueFoundry capabilities before assuming availability.

## 4. Capacity and failure reserve

Measure representative steady/peak synthetic input/output token distributions, request concurrency, p95 TTFT/TPOT, stream completion and GPU memory. Determine SLO-constrained throughput per **ready** replica. Compute remaining throughput after the largest failure domain is removed:

```text
surviving_capacity = total_ready_capacity - largest_domain_ready_capacity
pass only if surviving_capacity >= approved_priority_peak_demand
```

Include cloud GPU SKU availability, namespace and account quotas, model warmup and autoscaler provisioning lag. N-1 must not depend on unavailable cold capacity for a strict recovery target.

## 5. Release and rollback verification

Verify immutable image/model digests, tokenizer, CUDA/driver/PyTorch/vLLM compatibility, config and gateway route. Canary synthetic traffic with revision-specific metrics; test streaming drain and verify a known-good rollback. Stop on approved error-budget burn, TTFT/TPOT regression, GPU OOM, stream failures, missing telemetry or security alerts. Rollback may require restoring platform desired state rather than a direct Kubernetes rollout command.

## 6. Six staged recovery exercises

1. **GPU scheduling failure:** isolate Pending Pod cause (quota, taints, allocatable GPU, cloud supply) and restore readiness.
2. **Model/runtime failure:** simulate bad revision or CUDA incompatibility in disposable staging; restore known-good artifact.
3. **Inference saturation:** bound synthetic burst; validate queue, TTFT/TPOT, admission controls and warm scale-up.
4. **Streaming release interruption:** verify gateway drain, termination grace and client behavior.
5. **Zone loss / regional DR tabletop:** assess N-1 throughput, routing, target-region identity/artifacts, RTO/RPO and failback. Actual destructive failover requires separate approval.
6. **Observability/on-call failure:** rehearse missing metrics, broken alert route, escalation, customer update and incident closure.

For each exercise: preconditions, blast radius, change ticket, UTC start/end, synthetic traffic profile, detection, diagnosis, mitigation, rollback, SLI recovery, cost impact, evidence and reviewer. Tabletop results are not executed failover evidence.

## 7. Incident command and recovery proof

Assign incident commander, technical lead, communications lead and scribe. Use explicit SEV criteria and an incident timeline. Measure time-to-detect, time-to-acknowledge, time-to-mitigate and time-to-verify separately. Recovery is complete only when user-visible SLOs and redundancy are restored, not merely when Pods are Running.

## 8. FinOps and security acceptance

Reconcile cloud invoice and workload allocation, including idle warm reserve, shared network/storage/observability and unallocated spend. Verify that rightsizing does not remove N-1 capacity. Confirm least privilege, TLS, model artifact integrity, secret handling, network boundaries, retention and sanitized telemetry. Never store real inference payloads in evidence.

## 9. Sixteen capstone runbooks

1. GPU Pod Pending and quota exhaustion.
2. GPU driver/device plugin unhealthy.
3. CUDA OOM and model memory-fit failure.
4. vLLM queue saturation and TTFT regression.
5. Streaming completion regression.
6. Model artifact or registry outage.
7. Gateway/DNS/TLS outage.
8. TrueFoundry agent/reconciliation failure.
9. Kubernetes node/zone failure.
10. Autoscaler and GPU provisioning failure.
11. PDB-blocked maintenance.
12. Canary or runtime upgrade regression.
13. Rollback incompatibility.
14. Alert routing/no-data incident.
15. GPU cost spike and budget breach.
16. Regional failover/failback tabletop and escalation.

Each runbook must specify trigger, scope, read-only diagnostics, impact, safe mitigation, rollback, verification, owner and escalation.

## 10. Forty failure scenarios to assess

| Domain | Scenarios |
|---|---|
| Identity/security | 01 missing workload identity; 02 expired credential; 03 overly broad RBAC; 04 sensitive logging; 05 artifact digest mismatch |
| Scheduling/GPU | 06 quota exhausted; 07 taint mismatch; 08 MIG mismatch; 09 device plugin failure; 10 GPU node NotReady |
| Model/runtime | 11 CUDA mismatch; 12 model OOM; 13 KV cache pressure; 14 tokenizer mismatch; 15 model cold start |
| Request/SLO | 16 TTFT regression; 17 TPOT regression; 18 high queue; 19 retry storm; 20 streaming drop |
| Networking | 21 ingress 5xx; 22 DNS outage; 23 TLS failure; 24 upstream timeout; 25 zone routing failure |
| HA/recovery | 26 replica co-location; 27 PDB deadlock; 28 N-1 capacity deficit; 29 cloud GPU scarcity; 30 failback regression |
| Delivery | 31 mutable image; 32 bad canary; 33 rollback artifact missing; 34 config drift; 35 incompatible node upgrade |
| Observability/FinOps | 36 missing metrics; 37 broken page; 38 invoice attribution gap; 39 cost anomaly; 40 incident handoff failure |

For each, capture expected/actual, UTC evidence, user impact, severity, diagnostic path, containment, recovery, rollback and reviewer. Unsafe cases remain tabletop.

## 11. One-hundred-point acceptance rubric

| Area | Points |
|---|---:|
| Architecture, ownership and service inventory | 10 |
| Security, identity, privacy and provenance | 10 |
| GPU capacity, quota and inference performance | 15 |
| HA, N-1 reserve and failure domains | 15 |
| Release, upgrade, drain and rollback | 15 |
| Observability, SLO and paging | 10 |
| Incident command, runbooks and communication | 10 |
| FinOps, budget and cost allocation | 5 |
| Recovery game days, evidence and closure | 10 |
| **Total** | **100** |

**Suggested policy:** GO requires >=90 points, no critical gate failure and required sign-offs. CONDITIONAL GO (80–89) requires explicit risk acceptance and a dated remediation plan; NO-GO is <80 or any critical gate failure. This is a proposed training rubric, **not a vendor certification**. Security/privacy failures, missing rollback, absent N-1 capacity against an agreed requirement or untested critical recovery gates are blockers regardless of score.

## 12. Twelve final acceptance gates

G1 scope/change approval; G2 inventory and ownership; G3 identity/security review; G4 synthetic workload baseline; G5 GPU capacity/quota; G6 N-1 reserve; G7 release/rollback; G8 streaming drain; G9 observability/alert delivery; G10 incident drills and recovery; G11 FinOps budget; G12 signed GO/NO-GO decision.

The [companion acceptance lab](../labs/07-production-readiness-recovery-capstone-check.md) provides a structured evidence template, game days, rubric and reviewer sign-off.

**Part 12 publication complete when both files and indexes are verified. Operational acceptance remains pending until executed and reviewed.**
