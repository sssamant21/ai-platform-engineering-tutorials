# Chapter 044 — Hands-On Lab: KServe Autoscaling and GPU Scheduling

**Part 06 | Chapter 044 | Published lab plan; live execution pending**

## Objective

Discover installed KServe autoscaling mode and GPU resources, design bounded scaling and placement, evaluate cold starts, VRAM, streaming drain, quotas and cost, and produce operational acceptance evidence.

## Guardrails

**[SAFE-READ]** authorized Kubernetes inspection and sanitized metrics. **[LAB-MUTATE]** separately approved isolated staging with synthetic traffic and bounded GPU-hours. **[CHANGE-CONTROL]** production HPA/KPA, node pools, GPU quotas, model replicas and fault injection. No PHI, customer prompts, Secrets or uncontrolled load.

## Lab 0 — Preflight

Record UTC, cluster/context, KServe version and serving mode, autoscaler/controller, GPU SKU, node pools, ResourceQuota, model/tokenizer/image digests, SLO, approval, cost ceiling and rollback.

## Lab 1 — Scaling controller discovery [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get crd | grep -Ei 'kserve|knative'
kubectl get inferenceservices -A
kubectl get hpa -A
kubectl get pods -A -o wide
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Do not assume HPA or KPA is configured simply because KServe exists.

## Lab 2 — GPU scheduling evidence [SAFE-READ]

```bash
kubectl -n APPROVED_NS describe inferenceservice APPROVED_MODEL
kubectl -n APPROVED_NS describe pod APPROVED_POD
kubectl describe node APPROVED_GPU_NODE
```

Record GPU allocatable/allocated, taints, selectors, affinity, topology, device-plugin health, Pending reasons and cloud quota. Do not mutate nodes or GPU pools during inventory.

## Lab 3 — Autoscaling policy worksheet [DESIGN]

```yaml
# DESIGN WORKSHEET — NOT A DEPLOYABLE KUBERNETES OBJECT
serving_mode: VERIFIED_STANDARD_OR_KNATIVE
controller: VERIFIED_AUTOSCALER
min_warm_replicas: APPROVED
max_replicas: WITHIN_GPU_QUOTA
scale_signal: VERIFIED_INSTALLED_METRIC
queue_bound: APPROVED
request_deadline: APPROVED
gpu_hour_ceiling: APPROVED
rollback_bundle: IMMUTABLE_KNOWN_GOOD
```

Validate controller-specific fields against installed CRD schemas before applying any staging manifest.

## Lab 4 — Synthetic capacity matrix [DESIGN]

| Test | Goal | Evidence |
|---|---|---|
| steady | warm baseline | TTFT, ready replicas |
| burst | scale-up latency | desired/ready timeline |
| quota ceiling | safe overload | bounded queue |
| long context | KV/VRAM safety | per-GPU peak |
| scale-down | stream drain | terminal validity |
| GPU Pending | fault classification | scheduler events |
| cold model | startup budget | stage timings |
| idle | cost optimization | GPU-hours |

Execute only under approved staging scope, concurrency and time limits.

## Lab 5 — Recovery and rollback [DESIGN]

Prepare immutable InferenceService, runtime, model/tokenizer, gateway and autoscaler configuration. Define freeze, restore, synthetic golden inference, terminal streaming, SLO observation, GPU quota reconciliation and incident notes.

## Lab 6 — Eight tabletop game days

01 missing scaling controller; 02 GPU taints; 03 cloud quota; 04 device plugin; 05 slow model download; 06 KV OOM; 07 stream truncation; 08 oscillation/cost.

## Twenty-four failure checklist

F01 mode; F02 controller; F03 metric; F04 stale signal; F05 min warm; F06 max quota; F07 desired; F08 Pending; F09 taints; F10 topology; F11 plugin; F12 driver; F13 cloud quota; F14 image; F15 artifact; F16 KMS; F17 CUDA OOM; F18 KV; F19 queue; F20 cold start; F21 drain; F22 oscillation; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/version/mode | Pending |
| G02 | Scaling controller | Pending |
| G03 | Metric freshness | Pending |
| G04 | Min/max/warm pool | Pending |
| G05 | Desired/ready reconciliation | Pending |
| G06 | GPU scheduling | Pending |
| G07 | Taints/topology | Pending |
| G08 | Device plugin/driver | Pending |
| G09 | Cloud quota | Pending |
| G10 | Image/model startup | Pending |
| G11 | Workload identity/KMS | Pending |
| G12 | VRAM/KV | Pending |
| G13 | Queue/retry | Pending |
| G14 | Cold-start SLO | Pending |
| G15 | Streaming drain | Pending |
| G16 | Tenant/security | Pending |
| G17 | Telemetry/alerting | Pending |
| G18 | Full rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
KServe mode/controller/CRD versions:
InferenceService/runtime/model digests:
GPU SKU/nodes/taints/quota:
Scaling signals and desired/ready timeline:
Pending reasons/device plugin/driver:
Image/model startup stages:
VRAM/KV and terminal streaming:
Queue/retries/TTFT/TPOT/E2E:
Security/telemetry/alerting:
GPU-hours/cost:
Game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. For approved staging, stop synthetic traffic, restore GitOps source of truth, delete only owned test resources, verify no orphaned GPU nodes and reconcile GPU-hours. Do not mark gates complete without independent evidence.

**Next: Chapter 045 — Ray Serve and KServe Operations Project.**
