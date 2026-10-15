# Lab 6.5 — Model Loading, Warmup, Health Probes and Readiness Check

**Canonical:** [Chapter 6.5](../05-model-loading-warmup-health-readiness/05-model-loading-warmup-health-readiness.md)  
**Mode:** **[SAFE-READ]** by default. Optional probe changes, rollouts, fault injection and rollback are **[LAB-MUTATE]** in an approved disposable nonproduction TrueFoundry workspace.

## 1. Objectives

Measure model cold-start stages; validate startup/readiness/liveness semantics; prove the intended model is loaded before traffic; inspect EndpointSlice readiness; classify 16 failure cases; and document a safe recovery path.

## 2. Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record TrueFoundry version/workspace, namespace, approved deployment, image digest, model ID/version, owner, baseline SLO and rollback artifact. No real customer data or unapproved production changes.

## 3. Startup state timeline

| Stage | Start | End | Evidence |
|---|---|---|---|
| Scheduling | | | |
| Image pull | | | |
| Container start | | | |
| Model fetch/checksum | | | |
| Deserialize/runtime init | | | |
| Host/GPU memory allocation | | | |
| Warmup/golden inference | | | |
| Startup probe success | | | |
| Readiness and endpoints | | | |

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=200
kubectl -n <namespace> logs <pod> --previous --tail=200
kubectl -n <namespace> get endpointslices -o wide
```

Capture sanitized logs and monitoring timestamps; account for clock skew and cached downloads.

## 4. Probe contract inspection [SAFE-READ]

```bash
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> get pod <pod> -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" "}{.reason}{"\n"}{end}'
kubectl -n <namespace> get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" ready="}{.ready}{" restarts="}{.restartCount}{" lastReason="}{.lastState.terminated.reason}{"\n"}{end}'
```

Record startup failure budget, readiness path/period/timeout, liveness path and observed failures. Confirm the app actually implements these endpoints. **Running is not Ready; Ready is not proof of golden inference correctness.**

## 5. Optional controlled rollout [LAB-MUTATE]

1. In an isolated nonproduction workspace, deploy a vetted model image at immutable digest A.
2. Record artifact fetch, checksum, model load and warmup timing.
3. Verify readiness remains false until the model is loaded and a golden inference passes.
4. Confirm Service EndpointSlice membership only after readiness.
5. Deploy candidate B with approved startup budget and surge capacity.
6. Monitor Pod readiness, endpoints, request errors and P95/P99 latency.
7. Restore known-good A using the **authoritative TrueFoundry/GitOps workflow** and verify recovery.

Do not intentionally kill shared Pods, clear shared caches or change production probes.

## 6. Optional graceful shutdown test [LAB-MUTATE]

On a disposable deployment, generate a bounded synthetic request, perform an approved rollout or termination, and measure whether the request completes or is explicitly canceled. Compare application SIGTERM handling, gateway drain behavior and grace period. Never assume a fixed `preStop` sleep guarantees safety.

## 7. Failure tabletop

| ID | Scenario | First failed transition |
|---|---|---|
| P1 | Pod Pending | Scheduler → node |
| P2 | ImagePullBackOff | Registry → container |
| P3 | Model 403/404 | Identity/reference → storage |
| P4 | Artifact checksum mismatch | Download → integrity |
| P5 | Init container blocked | Init → app |
| P6 | Framework import error | Image → runtime |
| P7 | CUDA initialization failure | Device → framework |
| P8 | Host OOMKilled | Process → cgroup |
| P9 | GPU VRAM OOM | Model → GPU memory |
| P10 | Startup probe restart loop | Init duration → startup budget |
| P11 | Running but not Ready | Model → readiness |
| P12 | Ready before warmup | Readiness → traffic |
| P13 | Expensive probe overload | Probe → process |
| P14 | No ready EndpointSlices | Pod → Service |
| P15 | Rollout stalls without GPU headroom | Surge → scheduling |
| P16 | Streaming request interrupted on termination | Drain → request |

For each: lowest proven healthy layer, evidence, owner, minimal approved remediation, rollback and validation.

## 8. Acceptance scorecard

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G1 Model/version ownership | | |
| G2 Cold-start stage timing | | |
| G3 Probe contract validated | | |
| G4 Warmup before readiness | | |
| G5 RAM/VRAM/cache capacity | | |
| G6 Endpoint/drain behavior | | |
| G7 Failure isolation and recovery | | |
| G8 Rollback and cleanup | | |

## 9. Evidence report

```text
Date/time / approval:
TrueFoundry version/workspace/deployment:
Image digest and model version:
Pod/node/GPU resources:
Scheduling/pull/fetch/load/warmup timestamps:
Startup/readiness/liveness settings:
Ready conditions and EndpointSlices:
Host memory / VRAM / restarts:
Golden inference and first-request P99:
Graceful termination observations:
P1–P16 classifications:
Lowest Proven Healthy Layer:
First Failed Transition:
Remediation and rollback owner:
Post-recovery SLO and cleanup:
Not run steps and follow-ups:
```

**Next:** Chapter 6.6 — Model-Serving Observability, Performance and Troubleshooting.
