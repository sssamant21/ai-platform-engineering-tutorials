# Chapter 14.4 — Hands-on Lab: Model Deployment and Rollback

**Part 14 — SRE Runbooks | Approved staging | Live execution pending**

## Goal and safety

Practice reproducible model release preflight, staging canary, rollback and streaming recovery through TrueFoundry-managed desired state. **[SAFE-READ]** inspect; **[LAB-MUTATE]** disposable staging models; **[CHANGE-CONTROL]** production deployment/traffic shifts. Never use real prompts, PHI, production tokens or uncontrolled GPU load.

## Lab 0 — Inventory and baseline

```bash
kubectl config current-context
kubectl -n MODEL_NAMESPACE get deployments,replicasets,pods -o wide
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
kubectl -n MODEL_NAMESPACE get svc,endpointslices
```

Record TrueFoundry revision, source commit, immutable image/model digests, tokenizer, vLLM/CUDA/driver, GPU SKU, Pod requests, N-1 reserve, quota, cost cap, synthetic TTFT/TPOT, HTTP error rate and streaming completion.

## Lab 1 — Staging preflight

Validate image registry, model artifact identity, least-privilege secret references, startup probes, GPU allocation, model warmup and rollback availability. Record compatible release unit including config, routing and autoscaler settings.

## Lab 2 — Bounded canary

Use approved TrueFoundry release workflow on an isolated staging model. Capture before/after image and model digests, Pod Events, synthetic response contract, TTFT/TPOT, queue, GPU memory, HTTP errors, SSE completion and spend. Do not infer production readiness from a small sample.

## Lab 3 — Rollback rehearsal

Trigger a safe **staging-only** configuration regression or use a tabletop simulation. Roll back through authoritative platform desired state, not manual Deployment patches. Verify old artifact accessibility, desired/observed revision agreement, gateway route, healthy streams and alert reset.

## Lab 4 — Six game days

1. Registry/image digest failure.
2. GPU Pending/quota during rolling overlap.
3. Model artifact or secret reference failure.
4. CUDA incompatibility or vLLM startup error.
5. Canary TTFT/TPOT and SSE regression.
6. Rollback artifact missing, manual drift or controller reconciliation failure (tabletop if unsafe).

Each requires UTC timeline, approved scope, expected/actual, customer SLO, evidence, mitigation, rollback, cost and reviewer.

## Forty scenarios

01 image digest; 02 registry auth; 03 ImagePull; 04 agent; 05 reconcile; 06 admission; 07 Pending; 08 taint; 09 GPU quota; 10 autoscaler; 11 cloud capacity; 12 init; 13 artifact IAM; 14 secret; 15 crash loop; 16 host OOM; 17 startup probe; 18 false readiness; 19 driver/CUDA; 20 GPU missing; 21 weights; 22 tokenizer; 23 model load; 24 CUDA OOM; 25 KV cache; 26 TTFT; 27 TPOT; 28 queue; 29 502; 30 503; 31 504; 32 EndpointSlice; 33 TLS; 34 SSE; 35 retries; 36 canary sample; 37 cost; 38 old artifact; 39 drift; 40 rollback.

## Twelve acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope/change | Pending |
| G2 | Immutable image/model/config provenance | Pending |
| G3 | Identity/security/artifact access | Pending |
| G4 | GPU quota/N-1/cost | Pending |
| G5 | Startup/readiness/streaming | Pending |
| G6 | Synthetic baseline | Pending |
| G7 | Canary SLO | Pending |
| G8 | TTFT/TPOT/GPU | Pending |
| G9 | Abort/escalation procedure | Pending |
| G10 | Compatible rollback | Pending |
| G11 | Post-change inference/alerts | Pending |
| G12 | Reviewer/audit sign-off | Pending |

## Evidence template

```text
Exercise / change ticket / UTC start-end:
Cluster, model, TrueFoundry revision:
Source commit, image/model/tokenizer digests:
GPU SKU, driver, CUDA, vLLM, capacity:
Baseline synthetic traffic and SLO:
Canary traffic, errors, TTFT, TPOT, SSE:
Expected/actual and abort conditions:
Rollback revision and observed state:
Post-rollback SLO and cost:
Executed/tabletop, owner, reviewer:
```

## Cleanup

Restore approved staging desired state, remove disposable canary traffic and models, validate stream completion, GPU reserve, telemetry, cost and no orphaned routes. Formal review and live execution remain pending.

**Next:** Chapter 14.5 — vLLM Serving Health and Performance Runbook.
