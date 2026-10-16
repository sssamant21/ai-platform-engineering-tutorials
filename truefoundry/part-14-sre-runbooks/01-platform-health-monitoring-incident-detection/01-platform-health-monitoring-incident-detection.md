# Chapter 14.1 — TrueFoundry Platform Health Monitoring and Incident Detection Runbook

**Part 14 — SRE Runbooks | Canonical operational guide**

## Mission

Establish an executable, evidence-based platform health runbook covering TrueFoundry control plane and `tfy-agent`, Kubernetes workloads, GPU pools, NVIDIA/DCGM, vLLM inference, ingress and synthetic customer-path SLOs. Detect incidents before broad customer impact, classify severity, route alerts and verify recovery.

**Scope:** SRE/DBRE/platform engineering on-call; EKS, AKS or GKE implementations. Discover installed component names and vendor/version-specific interfaces. No command below assumes universal TrueFoundry namespaces or metrics.

**Safety:** [SAFE-READ] approved inspections and synthetic probes; [LAB-MUTATE] disposable staging; [CHANGE-CONTROL] production alert rules, routing, deployment rollback, node drain and capacity changes. Never place API tokens, customer prompts, PHI or private model data in logs, alerts or incident tickets.

## 1. Service dependency map

```text
TrueFoundry control plane ↔ tfy-agent / cluster registration
       │ desired state / reconciliation
       ▼
Kubernetes API / scheduler / GPU nodes / autoscaler
       │ NVIDIA driver / device plugin / DCGM
       ▼
Model Deployment / Service / EndpointSlice / vLLM
       │ gateway / TLS / DNS / authentication
       ▼
Customer synthetic inference / SSE streaming
       │
Prometheus + logs + traces + alerts + incident management
```

Control-plane health, workload readiness and customer-path health are distinct. Healthy Pods can still return errors; a disconnected agent can coexist with currently functioning inference.

## 2. Define SLIs, SLOs and alert ownership

| Signal | SLI | Initial alert approach | Primary owner |
|---|---|---|---|
| Inference availability | eligible successful requests / eligible requests | multi-window burn-rate; low-traffic guard | Model SRE |
| TTFT | p95 time to first generated token | sustained threshold against model-specific SLO | Model SRE |
| TPOT | p95 per-output-token latency | workload-normalized threshold | Model SRE |
| Streaming completion | completed streams / eligible streams | sustained drop | Gateway + model SRE |
| Platform reconciliation | age of last successful reconcile | age exceeds expected interval | Platform SRE |
| Agent connection | heartbeat/connection age if exposed | confirmed stale with independent corroboration | Platform SRE |
| GPU capacity | allocatable minus requested GPU slots | headroom below N-1 reserve | GPU platform |
| GPU hardware | XID/ECC/temperature/health | hardware error, not utilization alone | GPU platform |
| Kubernetes | unavailable replicas / Pending duration | sustained service-impacting condition | K8s SRE |
| Observability | scrape freshness, probe success | absent/stale data; independent alert | Observability SRE |

Use organization-approved numeric thresholds from measured baselines and contractual SLOs; the table does not assert universal thresholds. Monitor both short and long error-budget windows to reduce noise and detect fast burns. Do not page on raw GPU utilization without customer impact/capacity context.

## 3. Five-minute on-call triage

1. Acknowledge page and establish UTC start time, affected models/regions/tenants, SEV and incident commander.
2. Check **independent synthetic customer-path** request, including streaming completion.
3. Check ingress 5xx, 429, 401/403, p95 TTFT/TPOT, queue, error budget and request volume.
4. Check TrueFoundry desired vs observed state and discovered agent health.
5. Check Kubernetes Pods, Events, EndpointSlices, node health and GPU resource allocatable.
6. Check NVIDIA/DCGM, vLLM model loading and queue metrics.
7. Correlate releases, certificates, network policies, node provisioning and cloud quota changes.
8. Declare mitigation owner, escalation and next update time; preserve sanitized evidence.

## 4. Read-only command pack

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
kubectl get svc,endpointslices -A
kubectl get resourcequota,pdb,hpa -A
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --since=20m --tail=150
```

Use namespace-scoped alternatives when cluster-wide RBAC is not granted. Identify `tfy-agent` from installed inventory; do not assume Deployment name or namespace. Logs may contain sensitive fields—redact.

## 5. Prometheus and telemetry design

Use version-verified metrics from Prometheus, kube-state-metrics, node exporter, DCGM exporter, ingress controller and vLLM `/metrics`. Verify labels and names against deployed exporters before adopting example PromQL.

Example **illustrative** request error ratio (replace labels/metric names with actual series):

```promql
sum(rate(http_requests_total{service="model-gateway",status=~"5.."}[5m]))
/
clamp_min(sum(rate(http_requests_total{service="model-gateway"}[5m])), 1)
```

A clamp denominator of 1 can distort low-traffic error ratios; for production burn-rate alerts use the actual eligible-request denominator, minimum-volume gating and measured SLO/error budget. Do not copy illustrative queries into production unchanged.

For inference, track queue duration, running/waiting requests, TTFT/TPOT, input/output token mix, KV cache and GPU memory. Correlate metrics with model revision, GPU SKU, zone and synthetic workload. Missing metrics require a dedicated telemetry freshness alert.

## 6. Synthetic inference and streaming health

Use a dedicated approved staging/production synthetic model, safe prompt, scoped identity, low rate and documented endpoint. Record HTTP result, TTFT, total latency, streamed event count, completion marker where protocol defines one and response termination. A successful HTTP 200 with truncated stream is **not** success. Never use real user prompts.

## 7. Alert routing and severity

- **SEV1:** widespread customer-serving failure, major security concern or rapidly exhausted error budget; immediate page, incident commander, communications and vendor escalation.
- **SEV2:** material model/region degradation or sustained SLO breach; page relevant owning team and establish mitigation.
- **SEV3:** bounded degradation without immediate user impact; ticket or business-hours workflow according to policy.

These are suggested definitions; use the organization's approved severity matrix. Group related alerts by service, cluster, region and incident to prevent page storms. Inhibit derivative alerts only when parent signal is reliable; never silence independent customer-path probes during a control-plane outage.

## 8. Diagnostic decision table

| Symptom | First evidence | Next runbook |
|---|---|---|
| Agent stale but inference healthy | agent logs, reconciliation age, synthetic probe | Chapter 13.2 / Part 14 agent |
| Pending GPU Pod | FailedScheduling, allocatable, autoscaler | Chapter 13.3 |
| CrashLoop/ImagePull | Pod Events, previous logs, digest | Chapter 13.4 |
| 502/503/504 or stream truncation | gateway, EndpointSlice, TLS, stream trace | Chapter 13.5 |
| CUDA OOM or TTFT spike | Pod state, DCGM, vLLM queue, traffic mix | Chapter 13.6 |
| Multiple simultaneous failures | customer SLO, UTC timeline, command structure | Chapter 13.7 |

## 9. Twelve operational runbooks

1. Platform health dashboard and telemetry freshness.
2. Synthetic inference availability and SSE completion.
3. SLO error-budget burn detection.
4. TrueFoundry agent heartbeat and reconciliation.
5. Kubernetes deployment readiness and Pod failures.
6. GPU node/device-plugin and allocatable capacity.
7. NVIDIA hardware XID/ECC/thermal alarms.
8. vLLM queue, TTFT and TPOT regression.
9. Ingress HTTP 5xx/429 and DNS/TLS.
10. Observability blind spot or exporter outage.
11. Incident paging, escalation and communications.
12. Recovery, alert re-arming and post-incident validation.

For each runbook capture trigger, UTC timeline, customer impact, authorized diagnostics, hypothesis, mitigation, rollback, SLO validation and owner.

## 10. Thirty-six failure scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | Synthetic probe failing | 19 | CUDA OOM |
| 02 | Synthetic stream truncated | 20 | Host OOMKilled |
| 03 | High gateway 5xx | 21 | vLLM queue spike |
| 04 | High 429 | 22 | TTFT regression |
| 05 | TLS expiry | 23 | TPOT regression |
| 06 | DNS resolution error | 24 | KV cache pressure |
| 07 | Agent disconnected | 25 | Cold-start delay |
| 08 | Reconciliation stale | 26 | GPU XID |
| 09 | Platform UI stale | 27 | GPU ECC |
| 10 | Kubernetes API throttling | 28 | GPU thermal throttle |
| 11 | Deployment unavailable | 29 | Autoscaler max reached |
| 12 | Pending GPU Pod | 30 | Cloud GPU quota |
| 13 | ImagePullBackOff | 31 | DCGM exporter down |
| 14 | CrashLoopBackOff | 32 | Prometheus scrape stale |
| 15 | Empty EndpointSlice | 33 | Alert routing failure |
| 16 | Pod readiness failure | 34 | Alert storm |
| 17 | GPU device plugin down | 35 | Zone failure |
| 18 | GPU allocatable zero | 36 | Recovery only checked in UI |

## 11. Twelve acceptance gates

G1 authorized scope; G2 owner and severity matrix; G3 synthetic inference/stream SLO; G4 agent/control-plane signal; G5 Kubernetes workload signal; G6 GPU/DCGM signal; G7 vLLM performance signal; G8 ingress/DNS/TLS signal; G9 telemetry freshness; G10 paging and escalation; G11 rollback and recovery proof; G12 reviewer sign-off.

Complete the [companion lab](../labs/01-platform-health-monitoring-incident-detection-check.md). Live execution and formal review pending.

**Next:** Chapter 14.2 — TrueFoundry Agent Health and Reconciliation Runbook.
