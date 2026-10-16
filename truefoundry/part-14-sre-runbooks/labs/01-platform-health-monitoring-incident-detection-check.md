# Chapter 14.1 — Hands-on Lab: Platform Health Monitoring and Incident Detection

**Part 14 — SRE Runbooks | Approved staging | Live execution pending**

## Goal

Build a platform health dashboard, verify customer-path synthetic inference, map alert ownership, diagnose missing telemetry and conduct bounded incident detection game days. Publication is not proof of execution or acceptance.

**Safety:** [SAFE-READ] approved queries; [LAB-MUTATE] disposable staging; [CHANGE-CONTROL] production alert routes and policies. Synthetic traffic only, no secrets, PHI or real prompts in artifacts.

## Lab 0 — Baseline inventory

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl get svc,endpointslices -A
```

Record cluster, TrueFoundry agent identity, model/image revision, GPU SKU, ingress, vLLM version, exporters, synthetic endpoint, SLOs, on-call ownership and spend cap.

## Lab 1 — Validate health signal coverage

Verify independently: TrueFoundry desired/observed revision; Kubernetes Deployment readiness; GPU allocatable and DCGM exporter; vLLM queue, TTFT/TPOT; ingress 5xx/429; DNS/TLS; synthetic request success and SSE completion. Record actual exporter metric names/labels; do not assume they exist.

## Lab 2 — Alert design and routing

Design model-specific SLO burn alerts with minimum-traffic gates and telemetry freshness checks. Review grouping/inhibition and escalation. Test a **staging-only** notification route; do not page production responders without approval.

## Lab 3 — Five game days

1. Stale agent/reconciliation with independent serving probe.
2. GPU node/device-plugin degradation with customer SLO baseline.
3. vLLM queue and TTFT synthetic degradation (bounded).
4. Gateway 503/stream truncation with EndpointSlice and TLS evidence.
5. Prometheus/DCGM scrape outage and alert-routing verification.

Tabletop unsafe fault injections. Capture UTC timeline, expected/actual, detection delay, page route, SLO, mitigation, rollback and reviewer.

## Thirty-six scenarios

01 synthetic unavailable; 02 stream truncated; 03 gateway 5xx; 04 429; 05 TLS; 06 DNS; 07 agent; 08 reconcile; 09 UI stale; 10 API throttle; 11 deployment; 12 GPU Pending; 13 image pull; 14 crash loop; 15 EndpointSlice; 16 readiness; 17 device plugin; 18 GPU allocatable; 19 CUDA OOM; 20 host OOM; 21 queue; 22 TTFT; 23 TPOT; 24 KV cache; 25 cold start; 26 XID; 27 ECC; 28 thermal; 29 autoscaler max; 30 cloud quota; 31 DCGM missing; 32 Prometheus stale; 33 paging; 34 alert storm; 35 zone; 36 UI-only recovery.

## Twelve acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope and budget | Pending |
| G2 | Owner/severity matrix | Pending |
| G3 | Synthetic inference and stream SLO | Pending |
| G4 | Agent/reconciliation telemetry | Pending |
| G5 | Kubernetes workload health | Pending |
| G6 | GPU/DCGM telemetry | Pending |
| G7 | vLLM queue/TTFT/TPOT telemetry | Pending |
| G8 | Ingress/DNS/TLS health | Pending |
| G9 | Scrape freshness/observability alerts | Pending |
| G10 | Staging page/escalation verified | Pending |
| G11 | Rollback and recovery evidence | Pending |
| G12 | Reviewer sign-off | Pending |

## Evidence record

```text
Exercise / change approval / UTC start-end:
TrueFoundry, Kubernetes, GPU, vLLM, ingress:
Model revision / synthetic request profile:
SLIs/SLOs, telemetry freshness:
Agent, Pod, EndpointSlice, GPU signals:
Alert fired / owner / escalation time:
Expected vs actual, mitigation, rollback:
Customer-path recovery / cost:
Executed or tabletop:
Owner, reviewer, open risk:
```

## Cleanup

Remove only disposable staging resources, restore alert routes, confirm exporter freshness and healthy synthetic streams, and close test alerts. Formal technical/security review and live execution remain pending.

**Next:** Chapter 14.2 — TrueFoundry Agent Health and Reconciliation Runbook.
