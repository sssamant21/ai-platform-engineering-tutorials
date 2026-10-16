# Chapter 12.7 — Production Readiness and Recovery Capstone: Acceptance Lab

**Part 12 | Approved staging | Live execution pending**

## Goal and rules

Perform an integrated TrueFoundry GPU inference readiness assessment with synthetic load, bounded staging fault drills, incident command, rollback and documented **GO / CONDITIONAL GO / NO-GO**. [SAFE-READ] inspection; [LAB-MUTATE] approved staging; [CHANGE-CONTROL] production. No destructive production testing, real prompts, PHI, credentials or uncontrolled GPU costs.

## Lab 0 — Readiness inventory

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get resourcequota,limitrange -A
kubectl get pdb -A
kubectl get hpa -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Record approved cluster/namespace, TrueFoundry/agent, vLLM, model digest, GPU SKU, driver, Kubernetes, ingress, autoscaler, cloud quota, SLO, RTO/RPO, incident owner, change ticket and spend cap.

## Lab 1 — Architecture and security

Map all shared dependencies and failure domains; validate least privilege, network isolation, TLS, image/model integrity, secret handling, logging privacy and recovery artifact access. Redact all evidence.

## Lab 2 — Capacity and SLO baseline

Run representative bounded synthetic requests. Measure output tokens/sec, input/output distributions, p95 TTFT/TPOT, streaming completion, queue, GPU memory and error rate. Calculate remaining **ready** capacity after largest zone/node loss and compare with priority peak demand.

## Lab 3 — Release and rollback

Verify immutable image/model/config and version matrix. Execute an authorized disposable staging canary, record SLO and GPU changes, validate streaming drain and restore known-good revision using supported TrueFoundry controls.

## Lab 4 — Six game days

1. GPU scheduling/quota failure.
2. Model/CUDA runtime failure.
3. Bounded inference queue saturation.
4. Streaming drain interruption.
5. Zone/regional loss **tabletop** unless separately approved.
6. Missing telemetry or broken nonproduction alert route.

Record preconditions, blast radius, commander, UTC timeline, detection, mitigation, recovery, rollback, SLO, cost, evidence and reviewer.

## Lab 5 — FinOps and operational review

Reconcile billed/allocated GPU cost, unallocated cost, warm HA reserve, cost per million output tokens, budget alerts and rightsizing safeguards. Test handoff, severity, communications and postmortem ownership.

## Forty scenario checklist

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | Workload identity missing | 21 | Ingress 5xx |
| 02 | Credential expired | 22 | DNS outage |
| 03 | RBAC overprivileged | 23 | TLS failure |
| 04 | Sensitive logging | 24 | Upstream timeout |
| 05 | Model digest mismatch | 25 | Zone routing failure |
| 06 | GPU quota exhausted | 26 | Replica co-location |
| 07 | Taint mismatch | 27 | PDB deadlock |
| 08 | MIG mismatch | 28 | N-1 capacity deficit |
| 09 | Device plugin failure | 29 | Cloud GPU scarcity |
| 10 | GPU node NotReady | 30 | Failback regression |
| 11 | CUDA mismatch | 31 | Mutable image |
| 12 | Model OOM | 32 | Bad canary |
| 13 | KV cache pressure | 33 | Missing rollback artifact |
| 14 | Tokenizer mismatch | 34 | Configuration drift |
| 15 | Model cold start | 35 | Incompatible node upgrade |
| 16 | TTFT regression | 36 | Missing metrics |
| 17 | TPOT regression | 37 | Broken paging |
| 18 | High queue | 38 | Invoice attribution gap |
| 19 | Retry storm | 39 | GPU cost anomaly |
| 20 | Streaming drop | 40 | Incident handoff failure |

Each scenario requires expected/actual, safe UTC evidence, owner, impact, mitigation, rollback, reviewer and an **executed vs tabletop** label.

## Twelve acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope, environment and cost cap | Pending |
| G2 | Architecture inventory and ownership | Pending |
| G3 | Security/identity/privacy review | Pending |
| G4 | Representative synthetic SLO baseline | Pending |
| G5 | GPU capacity and quotas measured | Pending |
| G6 | N-1 failure-domain reserve verified | Pending |
| G7 | Release compatibility and rollback tested | Pending |
| G8 | Streaming-safe drain tested | Pending |
| G9 | Alerts, no-data and paging verified | Pending |
| G10 | Recovery game days and incident command | Pending |
| G11 | FinOps budget and cost attribution | Pending |
| G12 | Signed readiness decision | Pending |

## One-hundred-point scorecard

| Area | Maximum | Earned |
|---|---:|---|
| Architecture/ownership | 10 | Pending |
| Security/privacy | 10 | Pending |
| Capacity/performance | 15 | Pending |
| HA/N-1 | 15 | Pending |
| Release/rollback | 15 | Pending |
| Observability/paging | 10 | Pending |
| Incident command/runbooks | 10 | Pending |
| FinOps | 5 | Pending |
| Recovery/evidence | 10 | Pending |
| **Total** | **100** | **Not assessed** |

Training policy: GO >=90 with no critical failures; CONDITIONAL GO 80–89 with documented risk acceptance; NO-GO <80 or critical gate failure. No readiness status can be assigned until actual evidence and reviewer sign-off exist.

## Evidence template

```text
Exercise ID:
Change ticket / approvals:
Environment / region / cluster:
TrueFoundry / vLLM / GPU / driver versions:
Model and image digests:
SLO and RTO/RPO targets:
Start UTC / end UTC:
Synthetic load profile and sample size:
Expected / observed:
Metrics, logs and Events (sanitized):
Incident commander / technical lead:
Detection / acknowledgment / mitigation / recovery:
Customer-impact proxy:
Rollback performed and validated:
GPU cost / budget impact:
Executed or tabletop:
Open risks and action owners:
Reviewer and date:
```

## Cleanup and sign-off

Restore staging routes/config, remove disposable workloads, confirm baseline SLO and GPU capacity, close test alerts and verify no unexpected charges. Collect SRE, platform, security, service owner and FinOps decisions. Mark **pending** until reviews and live drills are complete.

**Part 12 publication capstone complete; production certification is not implied.**
