# Chapter 14.6 — Hands-on Lab: Incident Response, Escalation and Communications

**Part 14 — SRE Runbooks | Approved staging/tabletop | Live execution pending**

## Objective

Practice incident command, SEV classification, multi-layer diagnosis, escalation, safe mitigation, communication cadence, shift handoff and customer-path recovery for TrueFoundry GPU inference incidents.

**Safety:** staging-only fault injection with approval, synthetic traffic, no PHI/prompts/secrets in incident records. Production mutations and vendor data sharing require explicit change/security approval.

## Lab 0 — Incident preparation

```bash
kubectl config current-context
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl get svc,endpointslices -A
```

Record on-call roster, incident commander, operations lead, communications lead, scribe, security lead, change approver, TrueFoundry/cloud/NVIDIA escalation, region/GPU SKU, SLOs, known-good model revision and cost cap.

## Lab 1 — Declare and triage

Simulate a synthetic inference availability/streaming failure. Capture UTC detection, acknowledgment, severity, affected model/region, error budget, HTTP errors, TTFT/TPOT, queue, Pod/GPU/agent status and known changes. Assign roles and create a decision log.

## Lab 2 — Escalation and communications

Produce two approved mock internal updates and one customer-safe status update with measured impact, known facts, uncertainty and next update time. Practice a vendor escalation with sanitized logs and revision details. Never send unapproved customer communications or real support cases during a lab.

## Lab 3 — Mitigation, rollback and verification

Choose reversible mitigation and define owner, approver, abort condition and rollback. Use authoritative TrueFoundry desired state in staging or tabletop if unsafe. Verify customer-path synthetic inference, SSE completion, TTFT/TPOT, error rate, agent reconciliation, GPU capacity and alert re-arming.

## Lab 4 — Six game days

1. Gateway 503 + SSE truncation, SEV1 comms.
2. Agent disconnected but inference healthy.
3. GPU quota + Pending Pod and cloud escalation.
4. Bad vLLM rollout with CUDA OOM/TTFT regression.
5. Credential/privacy alert with restricted evidence.
6. Compound outage with missing telemetry and shift handoff.

For each: UTC timeline, role ownership, customer SLO, expected/actual, updates, escalation, mitigation, rollback, reviewer, executed/tabletop classification.

## Forty-eight scenario checklist

01 no page; 02 alert storm; 03 stale dashboard; 04 missing probe; 05 low traffic; 06 no acknowledgment; 07 no IC; 08 conflicting operators; 09 severity; 10 approver; 11 log missing; 12 handoff; 13 agent; 14 reconcile; 15 UI stale; 16 registry; 17 model revision; 18 artifact; 19 quota; 20 autoscaler; 21 Pending; 22 device plugin; 23 XID/ECC; 24 zone; 25 CUDA OOM; 26 host OOM; 27 KV; 28 TTFT; 29 TPOT; 30 queue; 31 502; 32 503; 33 504; 34 TLS; 35 SSE; 36 retries; 37 late update; 38 speculative RCA; 39 tenant leak; 40 wrong audience; 41 missed update; 42 unredacted vendor evidence; 43 no rollback; 44 worsened SLO; 45 Pod-only recovery; 46 muted alerts; 47 no stability window; 48 no RCA owner.

## Fourteen acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope | Pending |
| G2 | SEV and roles assigned | Pending |
| G3 | Customer-path SLO measured | Pending |
| G4 | UTC evidence and decision log | Pending |
| G5 | Cross-layer diagnosis | Pending |
| G6 | Escalation contacts validated | Pending |
| G7 | Security/privacy controls | Pending |
| G8 | Change approval | Pending |
| G9 | Rollback readiness | Pending |
| G10 | Internal updates | Pending |
| G11 | Approved external wording | Pending |
| G12 | Shift handoff | Pending |
| G13 | Synthetic recovery and alert re-arm | Pending |
| G14 | Reviewer sign-off | Pending |

## Evidence template

```text
Incident ID / exercise / approval:
UTC detection, acknowledgment, declaration:
SEV, IC, ops, comms, security, scribe:
Customer scope / SLO / synthetic traffic:
TrueFoundry/Kubernetes/GPU/vLLM/gateway evidence:
Fact vs hypothesis / change correlation:
Escalation, vendor case (if simulated):
Update timestamps and next update:
Mitigation, approver, abort, rollback:
Recovery SLO / stability / alerts:
Executed/tabletop, owner, reviewer, RCA action:
```

## Cleanup and review

Restore staging state, close mock pages and incident bridge, verify telemetry and customer-path synthetic SLO, archive sanitized evidence and assign action owners. Publication is not operational acceptance.

**Next:** Chapter 14.7 — Post-Incident Validation and Reliability Acceptance Runbook.
