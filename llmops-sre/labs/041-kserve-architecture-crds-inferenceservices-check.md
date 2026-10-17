# Chapter 041 — Hands-On Lab: KServe Architecture, CRDs and InferenceServices

**Part 06 | Chapter 041 | Published lab plan; live execution pending**

## Goal

Inventory KServe CRDs and serving mode, design an InferenceService, select a runtime, validate model artifact identity and GPU readiness, and produce an auditable staging deployment/rollback plan.

## Safety

**[SAFE-READ]** authorized Kubernetes inventory. **[LAB-MUTATE]** only separately approved isolated staging with synthetic models and bounded GPU-hours. **[CHANGE-CONTROL]** KServe installation/upgrade, CRDs, runtimes, gateway and production InferenceServices. No PHI, customer prompts, Secrets or public management exposure.

## Lab 0 — Preflight

Record UTC, cluster/context, KServe/controller/CRD version, serving mode (Standard/Knative-based), runtime inventory, namespace, owner, approval, model/image/tokenizer digests, GPU quota, SLO, cost ceiling and rollback.

## Lab 1 — CRD discovery [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get crd | grep -i kserve
kubectl api-resources | grep -Ei 'inferenceservice|servingruntime'
kubectl get inferenceservices -A
kubectl get servingruntimes -A
kubectl get clusterservingruntimes
kubectl get pods -A -o wide
```

If CRDs are missing, record the limitation and stop; do not install KServe without approval.

## Lab 2 — Schema and controller [SAFE-READ]

```bash
kubectl explain inferenceservice --recursive | head -100
kubectl explain servingruntime --recursive | head -100
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Find controller namespace/Deployment through authorized inventory before inspecting logs. Validate admission webhooks, RBAC and reconciliation conditions.

## Lab 3 — InferenceService design [DESIGN]

```yaml
# DESIGN SKELETON — INCOMPLETE, DO NOT APPLY
apiVersion: serving.kserve.io/v1beta1 # verify installed schema
kind: InferenceService
metadata:
  name: synthetic-model
  namespace: approved-staging
spec:
  predictor:
    # Choose installed runtime and version-valid predictor fields.
    # Pin model/tokenizer and image digests; set GPU resources.
```

Complete only using installed CRD schema, runtime inventory and approved staging prerequisites. Do not assume universal field names.

## Lab 4 — Model/runtime compatibility [DESIGN]

Compare ServingRuntime/ClusterServingRuntime supported model formats, protocol (V1/V2/OpenAI-compatible/custom), image provenance, storage initializer behavior, workload identity, GPU/VRAM and cold-start requirements.

## Lab 5 — Deployment and validation plan [DESIGN]

Document GitOps PR, admission validation, InferenceService status, predictor Pod scheduling, model startup, ingress TLS/auth, synthetic golden request, terminal streaming, TTFT/TPOT/E2E, GPU VRAM and rollback.

## Lab 6 — Eight tabletop game days

01 CRD upgrade; 02 runtime removal; 03 model IAM failure; 04 GPU Pending; 05 protocol 404; 06 tokenizer drift; 07 ingress TLS; 08 incomplete rollback.

## Twenty-four failure checklist

F01 CRD absent; F02 version; F03 webhook; F04 controller RBAC; F05 runtime missing; F06 format; F07 storage IAM; F08 checksum; F09 tokenizer; F10 image pull; F11 GPU Pending; F12 plugin; F13 CUDA; F14 VRAM OOM; F15 cold start; F16 ingress; F17 protocol; F18 output; F19 streaming; F20 autoscaling; F21 management; F22 metrics; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | CRD discovery | Pending |
| G03 | Schema/admission | Pending |
| G04 | Controller reconciliation | Pending |
| G05 | Runtime selection | Pending |
| G06 | Model provenance | Pending |
| G07 | Storage IAM | Pending |
| G08 | Image provenance | Pending |
| G09 | GPU scheduling | Pending |
| G10 | VRAM headroom | Pending |
| G11 | Startup/readiness | Pending |
| G12 | Ingress/TLS | Pending |
| G13 | Inference protocol | Pending |
| G14 | Golden output/streaming | Pending |
| G15 | Autoscaling/backpressure | Pending |
| G16 | Security/privacy | Pending |
| G17 | Telemetry/SLO | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
KServe/controller/CRD versions and serving mode:
Runtime inventory/selection:
InferenceService schema/conditions:
Model/tokenizer/image digests:
Storage IAM:
GPU allocation/VRAM:
Ingress/TLS/auth/protocol:
Golden output/terminal streaming:
TTFT/TPOT/E2E:
Security/telemetry/SLO:
GPU-hours/cost:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop approved synthetic load, remove only owned staging resources, restore GitOps source of truth and reconcile GPU-hours. All gates remain pending until independent review.

**Next: Chapter 042 — KServe Runtime Selection and Model Storage.**
