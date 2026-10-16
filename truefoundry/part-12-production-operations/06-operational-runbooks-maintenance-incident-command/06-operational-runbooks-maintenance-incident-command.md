# Chapter 12.6 — Operational Runbooks, Maintenance and Incident Command

**Part 12 — Production Operations | TrueFoundry SRE canonical tutorial**

## Objectives and safety

Establish an enterprise operating system for TrueFoundry-hosted GPU inference: shift handoffs, maintenance planning, runbook standards, on-call escalation, incident command, customer communication, change freeze, evidence preservation and post-incident learning.

**[SAFE-READ]** inspect status, metrics, logs, Events and configuration. **[LAB-MUTATE]** isolated staging drills with synthetic inference. **[CHANGE-CONTROL]** production drains, failovers, GPU pool changes, routing, credentials and model deployments. Never execute destructive commands on production without approved incident authority. Do not include prompts, PHI, credentials or model weights in tickets or shared logs.

## 1. Production operating model

```text
User impact / SLO burn / operational risk
             │
      Alertmanager + on-call
             │
   Triage → severity → commander
             │
 ┌───────────┼─────────────┐
 │           │             │
Kubernetes  GPU/vLLM     TrueFoundry/edge
 │           │             │
 └───── mitigation + communications ─────┘
             │
  verify recovery and SLO
             │
  incident record → RCA → actions
```

Separate service owner, incident commander, technical lead, communications lead and scribe. Small incidents may combine roles, but accountability must remain explicit. Maintain primary/backup on-call, platform/vendor escalation and a secure incident channel.

## 2. Shift handoff and daily checks

Each shift reviews customer-facing availability, streaming completion, TTFT/TPOT, active incidents, error-budget burn, pending Pods, GPU health, capacity/quota, recent releases, expiring credentials/certificates, backups/artifact availability and planned changes. Record UTC timestamp, owner, unresolved risk, action deadline and next escalation.

**Read-only preflight:**
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get deployments -A
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl top nodes 2>/dev/null || true
```

Identify the actual TrueFoundry agent namespace/deployment from inventory; do not assume its resource name. Confirm RBAC and approved scope before cluster-wide inspection.

## 3. Maintenance lifecycle

**Plan:** risk, affected models/tenants, node/zone, customer impact, dependencies, GPU headroom, PDB, drain policy, maintenance window, owners, rollback and communication.

**Preflight:** confirm healthy replicas, SLOs, synthetic smoke tests, immutable artifacts, model cache/registry, spare GPU nodes, quota, cloud SKU availability and incident freeze.

**Execute:** use version-controlled changes, approved platform controls and one failure domain at a time. For node work, inspect `kubectl drain --help` and current PDB behavior; never use `--force` or `--disable-eviction` as a routine workaround. Preserve long-running inference streams.

**Validate:** restore readiness, streaming completion, TTFT/TPOT, GPU memory, queue, error rate, alerts and desired replica spread. Verify no lingering cordoned nodes or orphan GPU spend.

**Close:** document actual change, timeline, deviations, rollback outcome and handoff.

## 4. Severity and incident command

| Severity | Example | Response |
|---|---|---|
| SEV1 | Broad inference outage, sensitive-data exposure or critical security compromise | Immediate commander, security where applicable, frequent stakeholder updates |
| SEV2 | Significant model/region degradation, sustained SLO burn | Incident bridge, technical lead and mitigation |
| SEV3 | Limited tenant impact, reduced redundancy, capacity risk | Assigned owner, escalation if impact grows |
| SEV4 | Nonurgent defect or observability improvement | Planned backlog |

These are example classifications; align thresholds and communication cadence with organization policy. Severity is based on **impact and risk**, not GPU utilization alone.

Incident lifecycle: detect → declare → assign roles → scope impact → stabilize → mitigate → verify recovery → communicate → close → blameless review. Use UTC timeline and distinguish observation, hypothesis, mitigation and confirmed cause.

## 5. Triage decision tree

```text
Customer-facing errors/latency/streaming issue?
 ├─ YES → gateway status and trace/log correlation
 │        ├─ Pods not ready → Events, probes, scheduling
 │        ├─ CUDA/GPU failure → driver, device plugin, XID, memory
 │        ├─ queue/TTFT → prefill, tokens, autoscaling, KV cache
 │        ├─ network → DNS, TLS, ingress, upstream timeout
 │        └─ change-linked → canary comparison and rollback
 └─ NO → telemetry absent or redundancy/capacity risk?
          ├─ YES → monitoring incident or preventive maintenance
          └─ NO → trend analysis, record evidence, watch
```

A missing metric is not a healthy zero. Use independent synthetic checks when telemetry is impaired.

## 6. Standard runbook contract

Every production runbook must specify: title, service, severity, owner, last review, permissions, scope, symptoms, impact/SLIs, safe read-only commands, diagnostic branching, mitigation, change-control requirements, rollback, verification, escalation, evidence redaction and follow-up. Label potentially destructive commands and avoid hardcoded production namespaces.

## 7. Sixteen operational runbooks

**RB-01 — TrueFoundry agent unhealthy:** inspect actual agent workload, identity, network reachability, reconciliation errors and platform support.

**RB-02 — Deployment Pending:** inspect Events, GPU allocatable, quotas, affinity, taints and autoscaler.

**RB-03 — CrashLoopBackOff:** inspect `kubectl logs --previous`, probes, model artifacts, OOM and CUDA startup.

**RB-04 — ImagePullBackOff:** inspect registry auth, digest, egress, architecture and rate limits.

**RB-05 — CUDA unavailable:** validate NVIDIA device plugin, driver/runtime compatibility and Pod GPU allocation.

**RB-06 — GPU XID/ECC incident:** preserve node evidence, assess device health and coordinate approved replacement.

**RB-07 — vLLM model load failure:** inspect tokenizer/model revision, permissions, quantization and memory fit.

**RB-08 — GPU memory pressure:** distinguish CUDA OOM, KV cache exhaustion and Kubernetes OOMKilled.

**RB-09 — TTFT regression:** inspect token mix, prefill, queue, warm capacity and recent changes.

**RB-10 — Streaming interruption:** inspect gateway timeout, connection draining, Pod termination and client cancellation.

**RB-11 — Gateway 5xx or TLS/DNS:** trace route, certificate, endpoint readiness and upstream errors.

**RB-12 — Autoscaling stalled:** inspect HPA/KEDA metrics, pending Pods, cloud quota and GPU node provisioning.

**RB-13 — PDB blocks maintenance:** inspect available replicas, N-1 reserve and voluntary eviction safety.

**RB-14 — SLO alert storm or silence:** validate recording rules, no-data policy, Alertmanager routes and silences.

**RB-15 — Rollback fails:** inspect immutable image/model/config, TrueFoundry reconciliation and traffic route.

**RB-16 — Regional incident/failover:** assess independent capacity, identity, model artifacts, DNS/routing and approved RTO/RPO.

For every runbook, record detection, scope, evidence, owner, containment, rollback, verification and escalation. Never equate mitigation with confirmed root cause.

## 8. Communications and handoff templates

**Initial update:** severity, UTC start, affected service/region, user impact, current mitigation, next update and incident commander. Avoid unsupported root-cause claims.

**Recovery update:** customer-visible SLI, restored endpoints, remaining degraded functions, monitoring window and owner.

**Handoff:** open actions, current hypotheses, mitigations, rollback state, pending approvals, alert silences and next escalation.

**Postmortem:** timeline, contributing factors, customer impact, detection gaps, what worked, action owners, deadlines and verification.

## 9. Game-day and operational readiness

Rehearse paging, on-call acknowledgment, safe mitigation, recovery, handoff and vendor escalation in staging. Test an alert that cannot deliver, a GPU Pod that cannot schedule, a vLLM queue saturation incident, a streaming drain regression and a change rollback. Measure detection time, acknowledgment time, mitigation time and verified recovery separately.

## 10. Acceptance

Require current ownership/on-call matrix, maintained runbooks, tested alert routing, maintenance procedure, safe rollback, five incident drills, customer communication, sanitized evidence and review sign-off. Publication is not live validation.

Complete the [companion lab](../labs/06-operational-runbooks-maintenance-incident-command-check.md): **36 scenarios, five game days and 12 acceptance gates**. Formal review and live execution pending.

**Next:** Chapter 12.7 — Production Readiness and Recovery Capstone.
