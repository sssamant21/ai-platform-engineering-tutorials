# Chapter 042 — Hands-On Lab: KServe Runtime Selection and Model Storage

**Part 06 | Chapter 042 | Published lab plan; live execution pending**

## Objective

Discover KServe runtimes and schemas, compare runtime/model compatibility, validate immutable model storage and workload identity, design GPU cold-start tests and create a complete rollback evidence package.

## Guardrails

**[SAFE-READ]** approved Kubernetes resource inspection and sanitized logs. **[LAB-MUTATE]** isolated staging only with explicit approval, synthetic models and bounded GPU-hours. **[CHANGE-CONTROL]** CRD/runtime changes, IAM, production storage and InferenceService rollout. No secrets, PHI or customer prompts in artifacts.

## Lab 0 — Preflight

Capture UTC, context, KServe version, serving mode, runtime name/scope, model/tokenizer/image digests, storage URI version, GPU SKU/quota, owner, change ID, cost ceiling and rollback release.

## Lab 1 — Runtime and schema inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl api-resources | grep -Ei 'servingruntime|inferenceservice'
kubectl get servingruntimes -A
kubectl get clusterservingruntimes
kubectl get inferenceservices -A
kubectl explain servingruntime.spec --recursive | head -100
kubectl explain inferenceservice.spec.predictor --recursive | head -100
```

Stop if CRDs are absent. Never install KServe or modify shared ClusterServingRuntime resources during discovery.

## Lab 2 — Runtime compatibility matrix [DESIGN]

| Runtime candidate | Model format | Protocol | GPU/TP | Storage method | Evidence |
|---|---|---|---|---|---|
| Installed runtime A | verify | verify | verify | verify | pending |
| Installed runtime B | verify | verify | verify | verify | pending |
| Approved custom | verify | verify | verify | verify | pending |

Only record actual installed or approved candidates. Verify container digest and model-server readiness semantics.

## Lab 3 — Immutable artifact contract [DESIGN]

```yaml
# WORKSHEET, NOT A KUBERNETES MANIFEST
image_digest: REPLACE
model_uri_and_version: REPLACE
model_file_manifest_sha256: REPLACE
tokenizer_sha256: REPLACE
runtime_name: VERIFIED
storage_service_account: APPROVED_IDENTITY
protocol: VERIFIED
gpu_count: APPROVED
rollback_bundle: KNOWN_GOOD
```

Validate all model files via manifest hashes; a single config checksum is insufficient.

## Lab 4 — Storage/IAM diagnostic plan [SAFE-READ/DESIGN]

For an approved object, inspect only metadata, identity bindings and sanitized startup errors. Map 403 to service account/role/bucket/KMS, 404 to immutable object version, timeout to DNS/TLS/VPC endpoint, checksum mismatch to corruption, and eviction to ephemeral storage. Do not grant wildcard storage access to solve an unexplained 403.

## Lab 5 — GPU cold-start and correctness [DESIGN]

Define bounded synthetic tests measuring image pull, artifact fetch, checksum, CUDA init, model load, warmup, readiness, golden inference, TTFT/TPOT/E2E, terminal streaming, per-device VRAM and GPU-hours. Include abort conditions and clean rollback.

## Lab 6 — Eight tabletop game days

01 runtime changed; 02 mutable artifact drift; 03 identity read denied; 04 KMS denied; 05 ephemeral storage exhaustion; 06 tokenizer mismatch; 07 CUDA OOM; 08 incomplete artifact rollback.

## Twenty-four failure checklist

F01 CRD; F02 runtime ambiguity; F03 model format; F04 image drift; F05 registry; F06 mutable URI; F07 initializer; F08 identity; F09 storage 403; F10 KMS; F11 artifact 404; F12 network; F13 disk; F14 stale cache; F15 checksum; F16 tokenizer; F17 unsafe code; F18 GPU Pending; F19 driver; F20 VRAM; F21 startup probe; F22 protocol; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/version | Pending |
| G02 | Runtime inventory | Pending |
| G03 | Model format/protocol | Pending |
| G04 | Image provenance | Pending |
| G05 | Immutable model | Pending |
| G06 | Tokenizer integrity | Pending |
| G07 | Workload identity | Pending |
| G08 | Storage/KMS/TLS | Pending |
| G09 | Network/download | Pending |
| G10 | Cache/disk capacity | Pending |
| G11 | GPU scheduling | Pending |
| G12 | CUDA/VRAM | Pending |
| G13 | Cold-start readiness | Pending |
| G14 | Golden inference | Pending |
| G15 | Streaming completion | Pending |
| G16 | Security/privacy | Pending |
| G17 | Telemetry/SLO | Pending |
| G18 | Full rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
KServe/controller/CRD version and mode:
ServingRuntime/ClusterServingRuntime:
Image/model/tokenizer digests:
Object version and manifest checksum:
Service account, bucket, KMS and network:
Download/cache/ephemeral storage:
GPU/TP/VRAM and cold-start stages:
Inference protocol/golden output:
TTFT/TPOT/E2E/terminal streaming:
Security, telemetry and SLO:
GPU-hours/cost:
Game-day findings and rollback:
Reviewer/acceptance gates:
```

## Cleanup

Read-only/tabletop work creates no resources. For approved staging, stop synthetic traffic, delete only owned resources, restore GitOps source of truth, verify caches and reconcile GPU-hours. Do not mark gates complete without evidence.

**Next: Chapter 043 — KServe Networking, Security and Canary Rollouts.**
