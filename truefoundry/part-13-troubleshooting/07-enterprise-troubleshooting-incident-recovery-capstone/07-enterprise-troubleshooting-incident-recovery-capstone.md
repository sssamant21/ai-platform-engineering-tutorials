# Chapter 13.7 — Enterprise Troubleshooting and Incident Recovery Capstone

**Part 13 — Troubleshooting | Final enterprise capstone**

## Mission

Integrate Chapters 13.1–13.6 into an evidence-driven, multi-layer TrueFoundry GPU inference incident response and recovery program. Diagnose failures across agent/control plane, Kubernetes admission/scheduling, deployment startup, networking/streaming and NVIDIA/CUDA/vLLM. Practice incident command, safe rollback, customer-path verification, postmortem and measurable acceptance.

**Publication is not operational certification.** Formal technical/security review, live lab execution and acceptance remain pending.

## Safety and prerequisites

**[SAFE-READ]** inspect approved Kubernetes/TrueFoundry/cloud telemetry and synthetic probes. **[LAB-MUTATE]** only disposable staging with change approval, rollback, blast-radius boundary and GPU cost cap. **[CHANGE-CONTROL]** all production fault injection, identity, GPU resets, node drains, routing, deployment rollback and cloud quota. Do not expose prompts, completions, PHI, registry tokens, kubeconfigs, credentials or proprietary model artifacts.

Inventory TrueFoundry/agent, Kubernetes, cloud provider, CNI, ingress/gateway, GPU SKU, NVIDIA driver/device plugin, vLLM, model/image digests, autoscaler, metrics, tracing, incident paging, SLOs, owners and recovery dependencies.

## 1. Multi-layer fault model

```text
Synthetic customer request / client SDK
     │ DNS, TLS, auth, rate limits
     ▼
Gateway / ingress / routing
     │ Service / EndpointSlice / readiness
     ▼
TrueFoundry desired state + agent reconciliation
     │ Kubernetes Deployment / ReplicaSet / Pod
     │ admission / scheduler / GPU capacity
     ▼
GPU node / NVIDIA driver / device plugin / CUDA
     │ vLLM / model artifacts / KV cache / batching
     ▼
Streaming tokens / TTFT / TPOT / completion
     │
Prometheus, logs, traces, DCGM, alerts, on-call
```

Control-plane availability, Kubernetes Pod readiness and customer-serving inference are **different** signals. Diagnose each independently. Avoid attributing every 5xx to vLLM or every Pending Pod to GPU quota without Events.

## 2. Severity and customer impact

Define SEV from measured eligible-request availability, p95 TTFT/TPOT, streaming completion, affected tenants/models/regions, duration and security implications. Record incident commander, technical lead, communications lead and scribe. Use UTC timestamps, clear escalation and a decision log. Missing telemetry is a visibility failure, not a zero-error result.

## 3. Unified read-only triage

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments,statefulsets,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
kubectl get resourcequota,limitrange -A
kubectl get pdb,hpa -A
kubectl get svc,endpointslices -A
```

Use namespace-scoped commands where RBAC/sensitivity requires. For a discovered authorized model Pod:

```bash
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --all-containers=true --since=30m --tail=200
kubectl -n MODEL_NAMESPACE logs MODEL_POD --all-containers=true --previous --tail=200
```

`--previous` may be unavailable. Logs can contain secrets or sensitive inference data; sanitize before evidence storage.

## 4. Cross-layer diagnosis decision tree

```text
Customer SLO violated?
  ├─ No → verify independent synthetic probe and observability
  └─ Yes
      ├─ DNS/TLS/connect failure → network path and certificate
      ├─ 401/403/429 → identity, policy, rate limits
      ├─ 502/503/504 → gateway, Service, EndpointSlice, upstream
      └─ Slow/truncated inference
          ├─ Pod absent → admission, scheduler, quota, autoscaler
          ├─ Pod failing → image, init, probes, CUDA/model startup
          ├─ Pod healthy but slow → queue, prefill, decode, GPU, traffic mix
          └─ stream drops → proxy timeout, buffering, drain, client
```

Investigate agent/control-plane separately if desired and observed revisions differ, cluster registration is stale or deployment operations are blocked. Confirm the actual platform version and resource ownership.

## 5. End-to-end evidence standard

Every exercise must capture: UTC incident timeline; customer-impact proxy; request/model revision; cloud region and zone; synthetic workload token distribution; Pod state and Events; gateway status; TrueFoundry desired vs Kubernetes observed state; GPU allocatable/health; vLLM queue/TTFT/TPOT; streaming completion; mitigations; rollback; cost; owner; reviewer; executed vs tabletop label.

Separate **observed fact**, **hypothesis**, **confirmed contributing factor**, and **root cause**. Do not call a correlated change root cause without evidence.

## 6. Six integrated game days

1. **Agent/reconciliation degradation:** new deployments fail while existing synthetic inference is independently checked.
2. **GPU scheduling and quota:** Pending model Pod, device plugin, autoscaler and cloud GPU supply investigation.
3. **Bad model rollout:** ImagePullBackOff/CrashLoopBackOff or incompatible model revision, safe staging rollback.
4. **Networking/streaming:** DNS/TLS/gateway failure, EndpointSlice/readiness and SSE completion investigation.
5. **GPU memory and vLLM saturation:** distinguish host OOM, CUDA OOM, KV cache and TTFT/TPOT regression under bounded synthetic load.
6. **Compound incident:** gateway 503 + GPU shortage + stale control-plane status + missing telemetry; incident command, staged recovery, customer update and postmortem.

Each requires an approved scenario, blast radius, preflight, abort conditions, synthetic probes, detection, diagnosis, rollback, verified SLO recovery, cost impact and reviewer. Tabletop destructive or unsafe injections.

## 7. Recovery and rollback hierarchy

Prefer reversible, low-blast-radius mitigations:
1. Validate issue and freeze unrelated changes.
2. Reduce harmful retry amplification or route only through approved healthy capacity.
3. Roll back known-bad image/model/config through authoritative TrueFoundry workflow.
4. Restore GPU capacity with cloud quota, AZ availability, device plugin and node health checks.
5. Restore gateway/EndpointSlice/streaming path.
6. Restore control-plane reconciliation, alerts and redundancy.
7. Confirm end-to-end synthetic SLO and stream completion, not merely Pod Running.

A rollback must include image **and** model/config compatibility; avoid manual Kubernetes hotfixes that the platform controller will overwrite.

## 8. Sixteen operational runbooks

1. Declare SEV and measure impact.
2. Establish UTC timeline and change correlation.
3. TrueFoundry agent disconnected.
4. Reconciliation and desired-state drift.
5. GPU Pod Pending / scheduling filters.
6. Cloud GPU quota and autoscaler stalled.
7. Device plugin/driver GPU absent.
8. ImagePullBackOff / registry access.
9. CrashLoopBackOff / probes / init.
10. CUDA/model startup incompatibility.
11. Host OOM vs GPU CUDA OOM.
12. vLLM KV cache/TTFT/TPOT regression.
13. Gateway 5xx, Service and EndpointSlice.
14. DNS/TLS/mTLS and SSE stream truncation.
15. Alert blind spot and incident communications.
16. Rollback, recovery verification and postmortem.

Each runbook specifies trigger, affected scope, safe diagnostics, hypothesis tests, mitigation, rollback, verification, escalation and owner.

## 9. Forty-eight failure scenarios

| Domain | Scenarios |
|---|---|
| Control plane | 01 agent unavailable; 02 cluster registration stale; 03 identity expired; 04 RBAC denied; 05 reconcile queue stalled; 06 UI status stale |
| Scheduling | 07 GPU quota; 08 taint mismatch; 09 affinity mismatch; 10 topology spread blocked; 11 device plugin missing; 12 cloud SKU unavailable |
| Deployment | 13 image digest missing; 14 registry auth; 15 init failure; 16 config key absent; 17 startup probe too short; 18 CrashLoopBackOff |
| GPU/runtime | 19 CUDA driver mismatch; 20 no GPU injected; 21 XID/ECC; 22 host OOMKilled; 23 CUDA OOM; 24 NCCL initialization |
| Model | 25 artifact IAM denied; 26 model weight load timeout; 27 tokenizer mismatch; 28 KV cache pressure; 29 cold start; 30 tensor parallel mismatch |
| Performance | 31 TTFT regression; 32 TPOT regression; 33 request queue; 34 retry storm; 35 streaming incomplete; 36 autoscale too slow |
| Network | 37 DNS NXDOMAIN; 38 TLS expiry; 39 gateway 502; 40 gateway 503; 41 gateway 504; 42 empty EndpointSlice |
| Reliability/operations | 43 zone loss; 44 N-1 reserve shortfall; 45 missing metrics; 46 paging failure; 47 rollback artifact absent; 48 incident handoff gap |

Document each scenario with expected/actual, UTC evidence, blast radius, diagnosis, mitigation, rollback, SLO verification and reviewer.

## 10. One-hundred-point acceptance rubric

| Domain | Points |
|---|---:|
| Incident command, impact and timeline | 10 |
| TrueFoundry agent and reconciliation | 10 |
| Kubernetes scheduling, GPU quota and autoscaling | 15 |
| Deployment, image pull and model startup | 15 |
| Networking, TLS, gateway and streaming | 15 |
| NVIDIA/CUDA/vLLM performance and OOM | 15 |
| Observability and evidence integrity | 10 |
| Rollback, recovery and postmortem | 10 |
| **Total** | **100** |

**Training policy:** GO >=90 with no critical gate failures; CONDITIONAL GO 80–89 with documented risk acceptance and remediation owner; NO-GO <80 or critical gate failure. These are proposed training thresholds, not vendor certification. Unverified customer-path recovery, security exposure or missing rollback evidence blocks GO regardless of score.

## 11. Fourteen final acceptance gates

G1 approved scope and cost cap; G2 incident severity/ownership; G3 UTC timeline and evidence; G4 agent/control-plane investigation; G5 scheduler/GPU capacity; G6 deployment/image/startup; G7 network/TLS/streaming; G8 CUDA/vLLM diagnosis; G9 independent synthetic SLO; G10 observability and paging; G11 safe rollback; G12 redundancy and budget; G13 RCA/action owners; G14 reviewer-signed GO/NO-GO.

Use the [capstone acceptance lab](../labs/07-enterprise-troubleshooting-incident-recovery-capstone-check.md) to collect evidence and sign-off.

**Part 13 publication completes when this chapter, lab and indexes are verified. Live execution, technical/security review and acceptance remain separate pending work.**

**Next planned curriculum:** Part 14 — SRE Runbooks.
