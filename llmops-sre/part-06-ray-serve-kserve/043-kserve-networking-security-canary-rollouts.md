# Chapter 043 — KServe Networking, Security and Canary Rollouts

**Part 06 — Ray Serve and KServe | Chapter 043/100**

> **Status:** Written tutorial published. Live labs, independent technical/security review and operational acceptance pending. KServe networking, canary and rollout support depends on installed version, serving mode (Standard or Knative-based), gateway and runtime; inspect the actual CRDs before using any manifests.

## Learning objectives

Design secure KServe inference networking, compare Standard and Knative-based ingress, enforce TLS and workload identity, isolate tenants, protect inference and control APIs, implement validated canary/blue-green promotion, and recover from routing or model regressions with complete rollback evidence.

## 1. Reference architecture

```text
Client → TLS/API Gateway/WAF/authn/authz/rate limit
              │
          DNS + ingress / Gateway API / mesh
              │
        KServe InferenceService endpoint
              │
       ┌──────┴────────┐
       │               │
 Stable model A    Canary model B (only if supported)
       │               │
   Predictor Pod     Predictor Pod
       │               │
 GPU / runtime / immutable model artifacts
              │
 KServe controller + CRDs + Kubernetes Service
              │
 NetworkPolicies, workload identity, Prometheus/OTel
```

The exact resource graph varies. Standard mode commonly uses Kubernetes workload resources; Knative-based mode uses Knative serving resources and their routing/autoscaling machinery. Never assume Istio, Knative, Gateway API or percentage-based traffic splitting is present.

## 2. Networking inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get crd | grep -Ei 'kserve|knative|gateway'
kubectl get inferenceservices -A
kubectl get ingress -A
kubectl get gateways.gateway.networking.k8s.io -A
kubectl get httproutes.gateway.networking.k8s.io -A
kubectl get services -A
kubectl get endpointslices -A
```

Missing optional CRDs may produce `the server doesn't have a resource type`; record that and continue only with resources actually installed. Do not install networking controllers or mutate routes during discovery.

## 3. InferenceService endpoint and DNS

Validate the InferenceService status conditions and published URL against the actual gateway. Test DNS resolution, TLS certificate chain, SNI, Host header and upstream Service/EndpointSlice mappings. A healthy predictor Pod does not guarantee the public route is functional.

```bash
kubectl -n APPROVED_NS describe inferenceservice APPROVED_MODEL
kubectl -n APPROVED_NS get svc,endpointslices
```

Use synthetic probes with valid authentication; never print production tokens or real prompts.

## 4. Serving mode decision matrix

| Concern | Standard mode | Knative-based mode |
|---|---|---|
| workload controller | Kubernetes-oriented | Knative Serving |
| route objects | cluster networking dependent | Knative route/revision concepts |
| scale-to-zero | not assumed | may be supported/configured |
| canary weighting | not universally built in | revision routing may support it |
| cold starts | runtime and Kubernetes | plus Knative activation path |
| network dependencies | ingress/Gateway/mesh choice | Knative networking layer |

Confirm installed KServe version, controller mode, actual CRD schema and route implementation. A KServe canary field in one release does not imply the same semantics in another.

## 5. TLS and authentication

Terminate TLS at an approved gateway and verify encryption for any sensitive internal hop. Enforce OIDC/JWT or enterprise authentication before model inference; apply model-level authorization and per-tenant quotas. Network isolation alone is not user authentication.

Document TLS expiry, certificate rotation, trust chain, upstream SNI and authentication failures. Restrict management APIs and avoid publicly exposing KServe controllers or model server admin endpoints.

## 6. NetworkPolicies and east-west traffic

Adopt namespace isolation and least-privilege ingress/egress for predictor Pods. Explicitly allow DNS, artifact storage/VPC endpoints, metrics scraping, gateway traffic and required control-plane connections. NetworkPolicy behavior depends on CNI enforcement; verify it with approved synthetic connectivity tests.

## 7. Workload identity and model storage

Use read-only cloud workload identity for model artifact retrieval. Do not embed AWS keys, Azure secrets or GCP service account JSON in manifests. Pin artifact digests and storage object versions. Verify KMS decrypt and network endpoint access before promoting canary traffic.

## 8. Canary release strategy

```text
immutable model + tokenizer + image + runtime + gateway bundle
 → admission/schema checks
 → staging synthetic golden test
 → deploy candidate without production traffic
 → validate readiness, GPU VRAM and model output
 → route limited approved canary cohort (if supported)
 → compare terminal validity, TTFT, TPOT, p99, error budget
 → promote or route back to stable
 → confirm old revision drained and rollback evidence
```

If the serving mode lacks native weighted traffic splitting, implement a separately governed gateway/mesh route or blue-green switch; never invent a KServe spec field.

## 9. Blue-green versus weighted canary

| Method | Benefit | Risk |
|---|---|---|
| blue-green switch | simple rollback target | traffic cutover spike |
| weighted canary | limited blast radius | complex route semantics |
| header-based cohort | deterministic synthetic tests | auth/header trust |
| shadow traffic | compare responses | double GPU cost, privacy concerns |

Never mirror real user prompts to an unapproved model. Synthetic shadow tests are preferred.

## 10. Promotion and rollback gates

Require valid terminal completions, stable TTFT/TPOT/E2E, bounded error-budget burn, GPU VRAM headroom, correct tenant isolation, zero secrets leakage, verified image/model/tokenizer digests, known-good rollback and change approval. Canary sample size must be adequate; a few successful probes do not establish p99 performance.

## 11. Streaming and gateway timeouts

LLM streaming may last much longer than ordinary HTTP requests. Align gateway idle/read timeouts, request deadlines, cancellation propagation and terminal response semantics. Distinguish client disconnect from predictor crash and record incomplete streams separately.

## 12. Security threats

Protect against unauthenticated inference, cross-tenant model access, prompt logging, data exfiltration via artifact endpoints, SSRF-like model fetch patterns, public admin APIs, untrusted runtime images and overly broad Kubernetes RBAC. Apply signed artifacts, least-privilege admission, egress restrictions and sanitized telemetry.

## 13. Observability

Collect gateway 4xx/5xx/429, DNS/TLS errors, upstream reset, route revision, InferenceService conditions, predictor readiness, terminal-validity SLI, TTFT/TPOT/E2E, per-cohort canary results, GPU VRAM and GPU-hours per valid completion. Avoid high-cardinality tenant IDs in metric labels.

## 14. Safe-read troubleshooting

```bash
kubectl config current-context
kubectl get inferenceservices -A
kubectl get ingress,svc,endpointslices -A
kubectl get networkpolicies -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Use `kubectl get httproutes.gateway.networking.k8s.io -A` only if the Gateway API CRD exists. Inspect controller/gateway logs from discovered namespaces; redact credentials and prompt content.

## 15. Troubleshooting decision tree

```text
KServe endpoint fails?
 ├─ DNS failure → record/zone/resolver
 ├─ TLS error → cert/SNI/trust
 ├─ 401/403 → authn/authz/identity
 ├─ 404 → host/path/protocol route
 ├─ 429 → quota/admission
 ├─ 502/503 → upstream Service/EndpointSlice/readiness
 ├─ timeout → gateway idle/deadline/streaming
 ├─ canary wrong revision → route weights/labels
 ├─ model mismatch → image/model/tokenizer digests
 ├─ GPU OOM → candidate VRAM
 ├─ NetworkPolicy drop → CNI enforcement
 └─ rollback fails → incomplete release bundle
```

## 16. Twenty-four failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | incorrect DNS record | resolver |
| F02 | expired gateway certificate | TLS chain |
| F03 | upstream SNI mismatch | handshake |
| F04 | missing gateway authentication | security audit |
| F05 | authorization denies valid cohort | 403 |
| F06 | route Host header mismatch | 404 |
| F07 | protocol path mismatch | 404 |
| F08 | endpoint slice empty | Kubernetes state |
| F09 | predictor not ready | Pod status |
| F10 | ingress timeout too short | 504 |
| F11 | streaming interrupted | terminal validator |
| F12 | NetworkPolicy blocks gateway | CNI flow |
| F13 | egress blocks model storage | initializer log |
| F14 | workload identity too broad | IAM audit |
| F15 | management endpoint public | exposure scan |
| F16 | unsupported canary field | schema rejection |
| F17 | canary route weight wrong | gateway config |
| F18 | stale route targets old model | revision |
| F19 | tokenizer drift in candidate | golden test |
| F20 | candidate GPU OOM | VRAM |
| F21 | shadow traffic leaks prompts | privacy review |
| F22 | telemetry missing for cohort | metric audit |
| F23 | rollback misses gateway route | manifest diff |
| F24 | doubled GPU cost during rollout | FinOps |

## 17. Sixteen operational runbooks

1. Diagnose DNS and endpoint resolution.
2. Recover TLS certificate, trust and SNI.
3. Restore gateway authentication and authorization.
4. Diagnose Host/path/protocol route mismatch.
5. Recover Service/EndpointSlice and predictor readiness.
6. Diagnose streaming gateway timeouts.
7. Recover NetworkPolicy/CNI connectivity.
8. Restore model storage egress and workload identity.
9. Contain management endpoint exposure.
10. Validate installed canary and traffic-splitting support.
11. Correct canary route weights and cohort selection.
12. Diagnose candidate model/tokenizer drift.
13. Recover GPU OOM during dual-revision rollout.
14. Restore canary telemetry, SLO and privacy controls.
15. Reconcile rollout GPU-hour cost and quotas.
16. Execute complete gateway/model/runtime rollback and RCA.

## 18. Eight tabletop game days

1. DNS changes point inference to wrong gateway.
2. TLS certificate expires during model rollout.
3. NetworkPolicy blocks gateway-to-predictor traffic.
4. KServe canary field is rejected by installed CRD.
5. Weighted route sends most traffic to unready candidate.
6. Streaming responses truncate after gateway timeout.
7. Shadow testing exposes sensitive prompts.
8. Rollback restores model but leaves canary gateway weights.

Tabletop by default; live fault injection requires separately approved isolated staging.

## 19. Twenty acceptance gates

G01 approved scope/versions; G02 DNS resolution; G03 TLS/SNI; G04 gateway authn/authz; G05 model-level tenant isolation; G06 Service/EndpointSlice; G07 protocol/route; G08 NetworkPolicy/CNI; G09 storage egress/identity; G10 management isolation; G11 canary capability/schema; G12 route weights/cohort; G13 model/tokenizer provenance; G14 golden output/streaming; G15 TTFT/TPOT/error budget; G16 GPU VRAM/quota; G17 privacy/security; G18 complete rollback; G19 FinOps/telemetry; G20 independent review.

See [companion lab](../labs/043-kserve-networking-security-canary-rollouts-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 044 — KServe Autoscaling and GPU Scheduling.**
