# Chapter 13.4 — Hands-on Lab: Deployment Failures, CrashLoopBackOff and ImagePullBackOff

**Part 13 | Approved staging | Live execution pending**

## Mission and safeguards

Diagnose the full deployment lifecycle of a TrueFoundry-managed GPU inference service: admission, image pull, init, model startup, probes, rollout and rollback. Use synthetic staging traffic. **[SAFE-READ]** inspect. **[LAB-MUTATE]** disposable staging only. **[CHANGE-CONTROL]** production rollouts, credentials, image, model and routing changes. No secrets, real prompts, PHI or proprietary weights in evidence.

## Lab 0 — Baseline

```bash
kubectl config current-context
kubectl -n MODEL_NAMESPACE get deploy,rs,pods -o wide
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
kubectl -n MODEL_NAMESPACE get svc,endpointslices
kubectl -n MODEL_NAMESPACE get resourcequota
```

Record TrueFoundry desired revision, Kubernetes observed image digest, model digest, vLLM/driver/GPU versions, registry and artifact-store dependency, cold-start p95, synthetic TTFT/TPOT, streaming completion and approval.

## Lab 1 — Failed deployment diagnosis

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --all-containers=true --tail=200 --since=30m
kubectl -n MODEL_NAMESPACE logs MODEL_POD --all-containers=true --previous --tail=200
```

Identify whether failure occurred before Pod creation, while Pending, image pull, init, startup or readiness. `--previous` may have no retained container. Sanitize logs.

## Lab 2 — Image pull and configuration

Using **pre-existing failure evidence or disposable staging manifests**, investigate nonexistent image digest, missing registry authorization, rate limit, architecture mismatch, missing Secret/ConfigMap key and volume mount errors. Do not generate a real credential failure on shared production identities.

## Lab 3 — Model startup and probes

Compare CPU OOMKilled vs CUDA OOM, slow model warmup vs liveness restart, and model artifact download vs image pull. Inspect startup/readiness/liveness thresholds against measured cold-start time. Keep GPU spend bounded.

## Lab 4 — Five game days

1. Missing image digest or isolated registry access failure.
2. Synthetic process exit / CrashLoopBackOff.
3. Missing disposable ConfigMap key / init failure.
4. Startup probe too aggressive for simulated model warmup.
5. Bad staging release with supported TrueFoundry rollback and streaming-safe validation.

Each game day records UTC timeline, impact, Events, sanitized logs, hypothesis, approved change, rollback, TTFT/TPOT, streaming completion, cost and reviewer. Unsafe injections are tabletop.

## Thirty-six failure scenarios

01 rollout deadline; 02 ReplicaSet admission; 03 ResourceQuota; 04 PodSecurity; 05 GPU Pending; 06 volume mount; 07 image digest; 08 registry auth; 09 rate limit; 10 registry DNS; 11 registry TLS; 12 architecture; 13 pull secret; 14 CreateContainerConfigError; 15 missing Secret key; 16 missing ConfigMap; 17 init crash; 18 process exit; 19 startup probe; 20 liveness loop; 21 readiness; 22 OOMKilled; 23 CUDA OOM; 24 driver mismatch; 25 model IAM; 26 model download; 27 tokenizer; 28 tensor parallel; 29 warmup; 30 GPU surge; 31 maxUnavailable; 32 stream drain; 33 reconciliation drift; 34 rollback artifact; 35 missing metrics; 36 Ready but inference fails.

## Twelve acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope and GPU spend cap | Pending |
| G2 | Deployment/ReplicaSet/Pod classified | Pending |
| G3 | Events and termination reason preserved | Pending |
| G4 | Immutable image digest and registry checked | Pending |
| G5 | Admission/config/init diagnosis | Pending |
| G6 | Probe timing and warmup validated | Pending |
| G7 | GPU/CUDA/model startup evidence | Pending |
| G8 | TrueFoundry desired vs observed revision | Pending |
| G9 | Streaming-safe rollout verified | Pending |
| G10 | Approved rollback rehearsed | Pending |
| G11 | Synthetic SLO and readiness recovered | Pending |
| G12 | Sanitized evidence and reviewer sign-off | Pending |

## Evidence record

```text
Exercise ID / approval:
Environment / namespace / UTC start-end:
TrueFoundry desired revision / observed Deployment:
Image digest / model digest / GPU and runtime versions:
Pod phase, conditions, Events, termination reason:
Registry/artifact/config/probe findings:
Customer-path synthetic SLO baseline and outcome:
Expected vs observed:
Mitigation / rollback / cost:
Executed or tabletop:
Owner / reviewer / outstanding risks:
```

## Cleanup

Remove only disposable staging resources, restore approved model/image/config, confirm no stuck rollout, validate inference SLO and streaming completion, and reconcile GPU cost. Do not mark the lab executed merely because it was published.

**Next:** Chapter 13.5 — Networking, DNS, TLS, Ingress and Streaming Failures.
