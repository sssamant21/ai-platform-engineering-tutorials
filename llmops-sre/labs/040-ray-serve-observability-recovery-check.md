# Chapter 040 — Hands-On Lab: Ray Serve Observability and Recovery

**Part 06 | Chapter 040 | Published execution plan; live execution pending**

## Mission

Design and validate an auditable Ray Serve incident evidence package covering client SLIs, gateway/proxy, Serve replicas, Ray Core, KubeRay, GPU telemetry, security, full-stack recovery and FinOps.

## Safety

**[SAFE-READ]** authorized cluster inspection and sanitized logs. **[LAB-MUTATE]** separately approved isolated staging with synthetic traffic, bounded time and GPU-hours. **[CHANGE-CONTROL]** production restart, rollout, traffic changes, node fault injection and model control. No PHI, customer prompts, Secrets or public Ray management exposure.

## Lab 0 — Charter

Record UTC, change/incident ID, cluster/context, Ray/KubeRay/Serve versions, model/tokenizer/image digests, GPU SKU/quota, SLO, owner, reviewer, cost ceiling, abort conditions and known-good rollback.

## Lab 1 — Read-only infrastructure inventory

```bash
date -u
kubectl config current-context
kubectl get crd | grep -i ray
kubectl get rayservices,rayclusters -A
kubectl get pods,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For authorized namespace:

```bash
kubectl -n RAY_NAMESPACE describe rayservice APPROVED_RAYSERVICE
kubectl -n RAY_NAMESPACE get pods -o wide
kubectl -n RAY_NAMESPACE get endpointslices
```

If CRDs are absent, record and stop; no unapproved operator installation.

## Lab 2 — Metrics contract [DESIGN]

Define custom instrumentation counters for eligible requests and valid terminal completions; histograms for E2E, TTFT and TPOT; gauges for queue depth and ready replicas; GPU telemetry for per-device VRAM. Verify HELP/TYPE and units. Do not infer p99 from cumulative duration counters.

Example custom metric query:

```promql
histogram_quantile(0.99,
  sum by (le, model) (rate(inference_e2e_seconds_bucket[5m]))
)
```

This is not a claim that Ray exports `inference_e2e_seconds_bucket` by default.

## Lab 3 — Client synthetic contract [DESIGN]

Define synthetic prompts and expected output invariants. Capture first token, final terminal marker, E2E, TTFT, TPOT, errors, cancellation and retry classification. Do not record real prompts or tenant IDs.

## Lab 4 — GPU and Ray fault isolation [SAFE-READ/DESIGN]

On authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Compare Ray logical GPU reservations with Kubernetes device allocation, replica startup, placement groups, KV/VRAM and queue pressure. Map symptom → first failing layer → evidence → mitigation → rollback.

## Lab 5 — Recovery worksheet [DESIGN]

Prepare complete immutable release bundle (Ray/KubeRay/Serve, model/tokenizer, backend, gateway policy), approval, draining, synthetic golden validation, terminal streaming, SLO and cost observation. Tabletop head/worker failures unless separately approved for staging.

## Lab 6 — Eight tabletop game days

01 telemetry outage during TTFT incident; 02 healthy RayService but invalid model; 03 mutable artifact drift; 04 head outage; 05 GPU worker lost during streaming; 06 retry storm; 07 exposed dashboard; 08 incomplete rollback.

## Twenty-four failure checklist

F01 terminal telemetry; F02 histogram units; F03 stale scrape; F04 cardinality; F05 privacy leak; F06 route; F07 proxy; F08 controller; F09 actor crash; F10 artifact; F11 tokenizer; F12 head; F13 GPU Pending; F14 placement; F15 CUDA OOM; F16 KV; F17 queue; F18 retries; F19 stream; F20 state loss; F21 NCCL; F22 management exposure; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions/owners | Pending |
| G02 | Terminal-validity SLI | Pending |
| G03 | TTFT/TPOT/E2E definitions | Pending |
| G04 | Metric units/histograms | Pending |
| G05 | Telemetry freshness | Pending |
| G06 | Privacy-safe traces/logs | Pending |
| G07 | Gateway/proxy health | Pending |
| G08 | Serve app/replica readiness | Pending |
| G09 | Ray head/worker health | Pending |
| G10 | GPU/VRAM | Pending |
| G11 | Model/tokenizer provenance | Pending |
| G12 | Queue/admission/retry | Pending |
| G13 | Streaming/cancellation | Pending |
| G14 | Actor/worker recovery | Pending |
| G15 | Distributed ranks | Pending |
| G16 | Management isolation | Pending |
| G17 | SLO alerts/escalation | Pending |
| G18 | Full-stack rollback | Pending |
| G19 | FinOps/DR | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Ray/KubeRay/Serve/model release digests:
Client valid completion and TTFT/TPOT/E2E:
Gateway/proxy status:
Serve application/replica health:
Ray head/worker/actor state:
GPU/VRAM/KV/placement:
Telemetry freshness and units:
Trace/log privacy:
SLO/error budget/alerts:
Security and management isolation:
Rollback and recovery timing:
GPU-hours/cost:
Game-day mode and findings:
Reviewer/gate evidence:
```

## Cleanup

Read-only and tabletop activities create no resources. Stop approved staging load, restore GitOps source of truth, remove only owned test resources, verify no orphaned GPU workers and reconcile GPU-hours. All gates remain pending until independent review.

**Next: Chapter 041 — KServe Architecture, CRDs and InferenceServices.**
