# Chapter 12.6 — Hands-on Lab: Operational Runbooks, Maintenance and Incident Command

**Part 12 | Approved staging | Live execution pending**

## Mission and safeguards

Practice on-call handoff, maintenance preflight, incident declaration, SRE triage, communications, safe recovery and blameless postmortem for TrueFoundry GPU inference. **[SAFE-READ]** inspect; **[LAB-MUTATE]** disposable staging; **[CHANGE-CONTROL]** production. Do not page real production responders for test alerts without explicit coordination. No raw prompts, PHI or credentials in evidence.

## Lab 0 — Read-only baseline

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get deployments -A
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Record environment, TrueFoundry/vLLM versions, service owners, escalation paths, incident channel, approved SLOs, model/GPU inventory and safe rollback contact.

## Lab 1 — Shift handoff

Write a one-page UTC handoff with customer SLIs, pending risks, recent changes, GPU headroom, alerts, incident status and owners. Verify backup on-call coverage and unresolved tasks.

## Lab 2 — Maintenance rehearsal

Prepare a staging change with risk, affected services, PDB and GPU reserve, active stream handling, preflight checks, rollback and post-change verification. Execute only approved disposable changes; otherwise tabletop.

## Lab 3 — Five incident game days

1. **GPU Pod Pending:** correlate quota, taints, node capacity and autoscaler.
2. **vLLM queue saturation:** use bounded synthetic requests; inspect TTFT/TPOT and GPU/KV.
3. **Gateway streaming interruption:** investigate routing/drain behavior in staging.
4. **Alert routing failure:** use non-production receiver and document no-data handling.
5. **Bad release rollback:** verify immutable model/image/config and restored SLO.

Record UTC timeline, severity, incident commander, technical lead, scribe, detection/acknowledgment/mitigation/recovery times, communications and rollback.

## Thirty-six troubleshooting scenarios

| # | Scenario | Primary response |
|---|---|---|
| 01 | On-call owner missing | Escalation |
| 02 | Backup responder unreachable | Handoff |
| 03 | Alert route broken | Alertmanager |
| 04 | Alert storm | Grouping |
| 05 | Metrics absent | Telemetry |
| 06 | No-data shown healthy | SLO |
| 07 | TrueFoundry agent unhealthy | Reconciliation |
| 08 | Pod Pending | Scheduling |
| 09 | GPU quota exhausted | ResourceQuota |
| 10 | Cloud GPU quota exhausted | Provider |
| 11 | Node NotReady | Kubernetes |
| 12 | Device plugin unhealthy | GPU |
| 13 | CUDA runtime unavailable | Driver |
| 14 | GPU XID/ECC | Hardware |
| 15 | ImagePullBackOff | Registry |
| 16 | CrashLoopBackOff | Logs/probes |
| 17 | Model artifact unavailable | Storage/IAM |
| 18 | Model load timeout | Cold start |
| 19 | CUDA OOM | Device memory |
| 20 | Pod OOMKilled | Container memory |
| 21 | KV cache exhaustion | vLLM |
| 22 | TTFT regression | Prefill/queue |
| 23 | TPOT regression | Decode |
| 24 | Gateway 5xx | Request path |
| 25 | TLS certificate expired | Security |
| 26 | DNS resolution failure | Networking |
| 27 | Streaming interrupted | Drain |
| 28 | PDB blocks node maintenance | Availability |
| 29 | Autoscaler stalled | Capacity |
| 30 | Zone capacity lost | N-1 |
| 31 | Rollback artifact missing | Release |
| 32 | TrueFoundry overwrites manual change | Source of truth |
| 33 | Customer update inaccurate | Communications |
| 34 | Incident timeline incomplete | Evidence |
| 35 | Sensitive data in logs | Security incident |
| 36 | Postmortem actions lack owners | Follow-up |

For each scenario document observed versus expected, UTC evidence, impact, hypothesis, mitigation, rollback, escalation and reviewer. Unsafe cases are tabletop.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved scope and safe drill limits | Pending |
| G2 | Service ownership and on-call roster current | Pending |
| G3 | Severity and incident roles defined | Pending |
| G4 | Runbook templates and review dates complete | Pending |
| G5 | Maintenance preflight and PDB/GPU reserve checked | Pending |
| G6 | Alert delivery and escalation rehearsed | Pending |
| G7 | Customer SLO/impact triage validated | Pending |
| G8 | Five game days documented | Pending |
| G9 | Communications and handoff exercised | Pending |
| G10 | Rollback and recovery verified | Pending |
| G11 | Postmortem and action tracking completed | Pending |
| G12 | SRE/platform/security/service-owner sign-off | Pending |

Pass requires timestamped evidence and reviewer sign-off. Tabletop-only recovery must not be recorded as executed.

## Cleanup and evidence

Restore disposable staging configuration, close test alerts, confirm healthy baseline, collect sanitized runbook links, incident timeline, handoff, communications, postmortem and sign-off.

**Next:** Chapter 12.7 — Production Readiness and Recovery Capstone.
