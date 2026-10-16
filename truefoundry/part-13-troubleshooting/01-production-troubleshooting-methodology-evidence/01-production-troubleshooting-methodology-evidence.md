# Chapter 13.1 — Production Troubleshooting Methodology and Evidence Collection

**Part 13 — Troubleshooting | TrueFoundry SRE canonical tutorial**

## Learning outcomes

Build a systematic, evidence-driven approach to troubleshooting TrueFoundry on Kubernetes: distinguish platform, GPU, model, gateway and application failures; scope customer impact; establish a UTC timeline; correlate signals; isolate hypotheses; recover safely; and write actionable incident findings.

**[SAFE-READ]** inventory, logs, events, metrics and traces within approved RBAC scope. **[LAB-MUTATE]** isolated staging only. **[CHANGE-CONTROL]** production deployment changes, GPU node drains, IAM, traffic shifts, restart or rollback. Never copy customer prompts, PHI, API keys, credentials or model weights into incident evidence.

## 1. End-to-end diagnostic model

```text
Client and synthetic probes
  → DNS / TLS / ingress / load balancer
  → TrueFoundry routing and deployment state
  → Kubernetes Service / EndpointSlice / Pod
  → vLLM engine / model / tokenizer / KV cache
  → GPU driver / device plugin / CUDA / node
  → model registry, IAM, cloud quota and storage
```

Investigate from observed impact toward the failing dependency. Multiple errors may be symptoms of one shared failure domain. The TrueFoundry control plane, installed agent, Kubernetes and model server each expose different views; confirm actual product/version and authoritative desired state before changing resources.

## 2. Five-stage incident method

1. **Detect and classify:** customer-visible SLI, error ratio, TTFT/TPOT, stream completion, affected models/regions and severity.
2. **Establish timeline:** first symptom, recent deployments, scaling, node events, IAM changes, model updates and external dependencies.
3. **Correlate evidence:** gateway status, Kubernetes Events, Pod state, GPU health, inference metrics, logs, traces and independent synthetic probes.
4. **Test hypotheses safely:** one controlled change at a time in staging; record expected vs actual and reject unsupported explanations.
5. **Mitigate, verify and learn:** follow approved runbook/rollback, prove SLO recovery, preserve sanitized evidence, document contributing factors and owners.

Keep observation, hypothesis and confirmed root cause in separate fields. A Pod in Running state does not prove inference health; a missing metric is not zero.

## 3. Safe triage commands

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
kubectl get deployments,statefulsets -A
kubectl get svc,endpointslices -A
kubectl get resourcequota,limitrange -A
kubectl get pdb,hpa -A
```

Within an approved namespace:

```bash
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --all-containers=true --since=30m --tail=200
kubectl -n MODEL_NAMESPACE logs MODEL_POD --all-containers=true --previous --tail=200
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o wide
```

`--previous` works only when prior container logs exist. Log content may contain sensitive material: use access controls and redaction. Never run blanket cluster-wide log collection without approval.

## 4. Symptom-to-layer decision table

| Symptom | Primary checks | Common confounder |
|---|---|---|
| HTTP 5xx | gateway, upstream endpoints, Pod readiness, logs | ingress error masks model crash |
| High TTFT | prefill tokens, queue, cold starts, GPU memory | traffic mix changed |
| High TPOT | decode throughput, batching, GPU contention | client network slowness |
| Pending Pods | Events, quota, GPU allocatable, taints, affinity | cloud GPU supply exhausted |
| CrashLoopBackOff | prior logs, probes, CUDA/model load, OOM | probe too aggressive |
| ImagePullBackOff | registry, digest, auth, egress | cloud network policy |
| CUDA unavailable | driver, device plugin, runtime, GPU allocation | wrong node pool |
| GPU OOM | model fit, KV cache, context, quantization | CPU memory OOM is distinct |
| Streams interrupted | gateway timeouts, drain, Pod termination | client cancellation |
| No metrics | scrape config, exporter, relabeling, RBAC | absence falsely shown healthy |

## 5. GPU and vLLM evidence

Record GPU model/MIG mode, driver, CUDA compatibility, GPU allocatable/requested, node/zone, vLLM version, model digest, context length, quantization and tensor parallelism. If approved and installed, collect `nvidia-smi` inside an appropriate GPU diagnostic environment; it is not guaranteed in the model container. Inspect NVIDIA XID/ECC alerts via approved telemetry.

Measure request arrival, input/output token distributions, waiting/running requests, queue time, TTFT/TPOT, errors, GPU memory and model cold start. Verify actual Prometheus metric names against deployed exporters and engine versions.

## 6. Changes and causal confidence

A recent release is a useful hypothesis, not proof. Compare affected vs unaffected models, zones and revisions; assess traffic shifts, GPU quota, IAM, driver upgrades and platform reconciliation. Distinguish correlation from root cause. Use immutable image/model digests and UTC change records.

## 7. Evidence bundle schema

```yaml
incident_id: "INC-EXAMPLE"
environment: "staging"
service: "synthetic-inference"
start_utc: "YYYY-MM-DDTHH:MM:SSZ"
affected_scope: "model / region / tenants (redacted)"
sli_impact: "measured; denominator and window"
recent_changes: []
observations: []
hypotheses: []
confirmed_findings: []
evidence_refs: []
mitigations: []
rollback_status: "not started"
recovery_validation: "pending"
owner: "assigned"
review_status: "pending"
```

Avoid storing raw token payloads, credentials or identifying account metadata in evidence.

## 8. Twelve troubleshooting runbooks

1. Establish incident severity and customer impact.
2. Build UTC timeline and release correlation.
3. Inspect TrueFoundry agent/reconciliation health.
4. Diagnose Pod Pending and GPU placement.
5. Diagnose CrashLoopBackOff and probe failures.
6. Diagnose ImagePullBackOff and artifact fetch.
7. Diagnose NVIDIA/CUDA and GPU device health.
8. Diagnose model load and KV cache pressure.
9. Diagnose TTFT/TPOT and queue saturation.
10. Diagnose ingress 5xx, DNS/TLS and streaming interruption.
11. Diagnose missing telemetry and alert delivery.
12. Verify recovery, rollback, handoff and RCA.

## 9. Thirty troubleshooting scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | Agent disconnected | 16 | KV cache exhaustion |
| 02 | Reconciliation drift | 17 | TTFT spike |
| 03 | Pod Pending | 18 | TPOT regression |
| 04 | GPU quota exceeded | 19 | Request queue surge |
| 05 | Taint mismatch | 20 | Retry storm |
| 06 | Node affinity mismatch | 21 | Gateway 5xx |
| 07 | Device plugin missing | 22 | TLS expiry |
| 08 | Node NotReady | 23 | DNS failure |
| 09 | ImagePullBackOff | 24 | Streaming timeout |
| 10 | CrashLoopBackOff | 25 | Endpoint not ready |
| 11 | Probe misconfiguration | 26 | Model artifact access denied |
| 12 | CUDA driver mismatch | 27 | Metrics scrape failure |
| 13 | GPU XID/ECC | 28 | Alert route broken |
| 14 | CUDA OOM | 29 | Autoscaler stalled |
| 15 | Container OOMKilled | 30 | Rollback incomplete |

## 10. Twelve acceptance gates

G1 scope and authorization; G2 customer impact; G3 timeline; G4 recent changes; G5 Kubernetes evidence; G6 TrueFoundry control-plane evidence; G7 GPU/runtime evidence; G8 inference SLO evidence; G9 security redaction; G10 safe mitigation and rollback; G11 recovery verification; G12 reviewed incident report.

See the [hands-on lab](../labs/01-production-troubleshooting-methodology-evidence-check.md). Live execution and technical/security review remain pending.

**Next:** Chapter 13.2 — TrueFoundry Agent and Control Plane Troubleshooting.
