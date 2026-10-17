# Chapter 045 — Capstone Lab: Ray Serve and KServe Operations Project

**Part 06 final project | Written lab plan published | Live execution pending**

## Mission

Produce an evidence-backed Ray Serve versus KServe production-readiness comparison, with approved isolated staging, immutable releases, synthetic golden inference, GPU capacity, secure routing, autoscaling, fault recovery, rollback, SLO and FinOps.

## Safety classification

**[SAFE-READ]** authorized inventory and sanitized logs. **[LAB-MUTATE]** only approved isolated staging, synthetic prompts and strict GPU-hour/concurrency caps. **[CHANGE-CONTROL]** production CRDs/operators, gateway routing, node pools, IAM, faults and rollbacks. **[TABLETOP]** all failures by default. Do not log PHI, credentials or real customer prompts.

## Lab 0 — Charter and version matrix

Record UTC, change ID, cluster/context, Ray/KubeRay/KServe/controller versions, KServe serving mode, GPU SKU/quota, namespace, owner, reviewer, immutable model/tokenizer/image digests, SLO, budget, abort triggers and rollback.

## Lab 1 — Cluster inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get crd | grep -Ei 'ray|kserve|knative'
kubectl get rayservices,rayclusters -A
kubectl get inferenceservices -A
kubectl get servingruntimes -A
kubectl get clusterservingruntimes
kubectl get pods -A -o wide
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Missing resource types will return errors; record prerequisite gaps and do not install operators without approval.

## Lab 2 — Immutable release contract [DESIGN]

```yaml
# WORKSHEET, NOT A DEPLOYABLE MANIFEST
ray_version: PINNED
kuberay_version: PINNED
kserve_version: PINNED
kserve_serving_mode: VERIFIED
serving_runtime: VERIFIED
model_sha256_manifest: VERIFIED
tokenizer_sha256_manifest: VERIFIED
image_digest: VERIFIED
gateway_route_revision: PINNED
gpu_sku_and_count: APPROVED
rollback_bundle: KNOWN_GOOD
```

Verify exact CRD schema, runtime name, inference protocol and model storage identity.

## Lab 3 — Security and network plan [DESIGN]

Document TLS/auth, gateway route, tenant quota, workload identity, read-only artifact storage, KMS, signed image, NetworkPolicy, control-plane RBAC and private dashboards. Use synthetic prompts only.

## Lab 4 — SLO parity matrix [DESIGN]

| SLI | Ray Serve | KServe | Target |
|---|---|---|---|
| terminal-valid completion | pending | pending | approved |
| TTFT p95/p99 | pending | pending | approved |
| TPOT/ITL | pending | pending | approved |
| E2E p95/p99 | pending | pending | approved |
| queue wait/rejection | pending | pending | approved |
| GPU VRAM peak | pending | pending | approved |
| cold-start duration | pending | pending | approved |
| GPU-hours/valid completion | pending | pending | approved |

Use comparable models, token budgets, hardware and load; document deviations.

## Lab 5 — Bounded staging test plan [LAB-MUTATE only if approved]

Plan baseline, step-load, burst, long-context, streaming cancellation, scale-out, scale-down and rollback. Set duration, RPS/concurrency, max tokens, abort thresholds, cost ceiling and cleanup. Do not execute from this written publication.

## Lab 6 — Failure/recovery evidence [TABLETOP]

Run eight tabletop game days: actor crash; head outage; runtime missing; device plugin failure; model IAM failure; wrong canary route; KV OOM/queue saturation; incomplete rollback. Capture detection, containment, recovery steps, RTO and sign-off.

## Twenty-four failure checklist

F01 Ray CRDs; F02 KServe CRDs; F03 API mismatch; F04 runtime; F05 artifact drift; F06 IAM; F07 image; F08 GPU Pending; F09 plugin; F10 CUDA; F11 VRAM; F12 KV; F13 Ray actor; F14 Ray head; F15 KServe controller; F16 runtime missing; F17 gateway; F18 protocol; F19 autoscaling; F20 retries; F21 canary; F22 privacy; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Charter/versions/owners | Pending |
| G02 | Ray/KServe CRDs | Pending |
| G03 | Runtime/protocol | Pending |
| G04 | Immutable artifacts | Pending |
| G05 | Storage IAM/KMS | Pending |
| G06 | GPU scheduling/topology | Pending |
| G07 | VRAM/KV | Pending |
| G08 | Ray readiness | Pending |
| G09 | KServe readiness | Pending |
| G10 | TLS/auth/tenant isolation | Pending |
| G11 | Golden output/stream | Pending |
| G12 | TTFT/TPOT/E2E | Pending |
| G13 | Queue/retry | Pending |
| G14 | Autoscaler lead time | Pending |
| G15 | Failure recovery/RTO | Pending |
| G16 | Security/privacy | Pending |
| G17 | Telemetry/alerts | Pending |
| G18 | Full rollback | Pending |
| G19 | FinOps/ADR | Pending |
| G20 | Independent sign-off | Pending |

## Evidence template

```text
UTC/context/change ID/approval:
Ray/KubeRay/KServe/CRD versions:
GPU SKU/quota/topology:
Immutable model/tokenizer/image/runtime:
Ray Serve deployment and readiness:
KServe InferenceService/runtime readiness:
Gateway/TLS/auth/tenant isolation:
Synthetic golden/terminal streaming:
TTFT/TPOT/E2E/queue/retries:
Autoscaler desired/ready and cold starts:
VRAM/KV/device-plugin evidence:
Security/privacy/telemetry:
GPU-hours/cost per valid completion:
Eight game-day findings:
Full rollback evidence:
ADR recommendation and reviewer:
Acceptance gates:
```

## Cleanup and sign-off

Read-only/tabletop work creates no resources. Approved staging experiments must stop synthetic load, restore GitOps source of truth, remove only owned test resources, verify no orphaned GPU nodes, reconcile GPU-hours and retain sanitized evidence. Mark each gate complete only after independent review.

**Part 06 written capstone complete; live acceptance pending. Next: Chapter 046 — Kubernetes GPU Scheduling Fundamentals.**
