# Chapter 039 — Hands-On Lab: Ray Serve Autoscaling and Fault Tolerance

**Part 06 | Chapter 039 | Published lab plan; live execution pending**

## Goal

Design and evaluate Serve replica autoscaling, Ray worker/GPU capacity, cold starts, bounded queues, scale-down draining, fault recovery, SLO and FinOps. No live tests are claimed.

## Safety

**[SAFE-READ]** authorized Ray/Kubernetes inventory and sanitized telemetry. **[LAB-MUTATE]** separately approved isolated staging with synthetic workloads, bounded concurrency, duration and GPU-hours. **[CHANGE-CONTROL]** production autoscaler, worker groups, GPU nodes, routing and fault injection. No PHI, customer prompts, Secrets or unapproved cluster modifications.

## Lab 0 — Preflight

Record UTC, change ID, Ray/Serve/KubeRay versions, cluster/context, model/engine digests, GPU SKU/quota, warm replicas, SLO, owner, reviewer, cost ceiling and rollback.

```bash
date -u
kubectl config current-context
kubectl get rayservices,rayclusters -A
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

If CRDs are unavailable, record the finding and stop; do not install KubeRay without approval.

## Lab 1 — Three-layer scaling map [DESIGN]

Draw Serve desired replicas → Ray actor placement → KubeRay worker Pods → Kubernetes GPU node provisioning. Record owner, scaling signal, time to provision, quota and failure mode for each.

## Lab 2 — Autoscaling worksheet [DESIGN]

```yaml
# NOT A DEPLOYABLE RAY CONFIG — version-validated worksheet
min_replicas: APPROVED_WARM_CAPACITY
max_replicas: APPROVED_GPU_QUOTA
target_ongoing_requests: APPROVED_VALUE
upscale_delay: VERSION_VALIDATED
downscale_delay: VERSION_VALIDATED
max_pending_requests: APPROVED_BOUND
gpu_hour_ceiling: APPROVED_LIMIT
```

Check the installed Serve schema for actual keys/defaults. Define capacity for worst-case prompt/output and VRAM, not only request count.

## Lab 3 — Synthetic load matrix [DESIGN]

| Cohort | Goal | Evidence |
|---|---|---|
| steady | warm baseline | ready replicas, TTFT |
| ramp | scale-up response | desired/ready lag |
| burst | overload protection | queue/429/503 |
| long-context | KV/VRAM | GPU peak |
| quota ceiling | safe rejection | bounded backlog |
| scale-down | graceful drain | terminal completion |
| replica crash | restart/retry | recovery timing |
| worker loss | distributed recovery | rank readiness |

Execute only with explicit staging approval, bounded synthetic traffic and abort conditions.

## Lab 4 — Failure and rollback [DESIGN]

Define replica restart, worker replacement, Ray head recovery, immutable model bundle rollback, synthetic golden checks and SLO restoration. Distinguish actor restart from durable state recovery.

## Lab 5 — Observability and FinOps [DESIGN]

Record Serve desired/ready, pending actors, GPU worker Pods, node autoscaler decisions, model cold-start time, queue, TTFT/TPOT/E2E, valid terminal completion, per-GPU VRAM, GPU-hours and cost per valid completion. Validate exporter metric names/units.

## Lab 6 — Eight tabletop game days

01 burst before GPU provisioning; 02 placement infeasible; 03 cloud quota; 04 slow model download; 05 stream interrupted on scale-down; 06 worker loss; 07 head failure; 08 scaling oscillation.

## Twenty-four failure checklist

F01 schema; F02 warm SLO; F03 max quota; F04 stale signal; F05 decision lag; F06 pending actors; F07 placement; F08 GPU Pod; F09 cloud quota; F10 plugin; F11 image; F12 weights; F13 warmup; F14 OOM; F15 queue; F16 retries; F17 oscillation; F18 stream; F19 state; F20 ranks; F21 head; F22 metrics; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | Autoscaling schema | Pending |
| G03 | Min/max capacity | Pending |
| G04 | Demand signals | Pending |
| G05 | Desired/ready reconciliation | Pending |
| G06 | Actor scheduling | Pending |
| G07 | GPU worker scaling | Pending |
| G08 | Node/cloud quota | Pending |
| G09 | Model cold start | Pending |
| G10 | VRAM headroom | Pending |
| G11 | Bounded queue | Pending |
| G12 | Retry/deadline | Pending |
| G13 | Scale-up latency | Pending |
| G14 | Scale-down draining | Pending |
| G15 | Replica/worker recovery | Pending |
| G16 | Ray head/TP health | Pending |
| G17 | Security/privacy | Pending |
| G18 | Telemetry/SLO | Pending |
| G19 | Rollback/FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/change approval:
Ray/Serve/KubeRay versions:
Model/GPU/worker quota:
Autoscaling policy and signal:
Desired/ready replica timeline:
Pending actors/GPU workers/node autoscaler:
Cold-start and VRAM:
Queue/TTFT/TPOT/E2E:
Terminal completion/draining:
Replica/worker/head recovery:
Security/telemetry/SLO:
GPU-hours/cost:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop approved synthetic load, restore GitOps source of truth, remove only owned staging resources, verify no orphaned GPU workers and reconcile GPU-hours. All acceptance gates remain pending until independently reviewed.

**Next: Chapter 040 — Ray Serve Observability and Recovery.**
