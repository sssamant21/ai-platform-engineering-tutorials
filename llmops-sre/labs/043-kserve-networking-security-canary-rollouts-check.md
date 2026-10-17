# Chapter 043 — Hands-On Lab: KServe Networking, Security and Canary Rollouts

**Part 06 | Chapter 043 | Written lab plan; live execution pending**

## Goal

Inventory KServe networking and serving mode, design authenticated inference routing, validate NetworkPolicy and storage egress, create a safe canary/blue-green release plan and document full rollback evidence.

## Safety

**[SAFE-READ]** authorized Kubernetes discovery. **[LAB-MUTATE]** approved isolated staging with synthetic requests and bounded GPU-hours. **[CHANGE-CONTROL]** production gateways, TLS, policies, routes, CRDs and traffic splits. No PHI, customer prompts, Secrets or public management APIs.

## Lab 0 — Preflight

Record UTC, cluster/context, KServe/controller version, Standard/Knative mode, ingress/Gateway/mesh, namespace, immutable image/model/tokenizer digests, GPU quota, SLO, change ID, reviewer and rollback route.

## Lab 1 — Networking discovery [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get crd | grep -Ei 'kserve|knative|gateway'
kubectl get inferenceservices -A
kubectl get ingress -A
kubectl get svc,endpointslices -A
kubectl get networkpolicies -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Only query Gateway API resources if their CRDs are installed. Record absent resources rather than installing anything.

## Lab 2 — Route and TLS map [DESIGN]

Draw client → DNS → gateway TLS/auth → ingress route → KServe predictor → model runtime. Record Host header, SNI, port, path, protocol, readiness and timeout ownership. Design synthetic authenticated probes with no real customer data.

## Lab 3 — Security matrix [DESIGN]

| Boundary | Required policy | Evidence |
|---|---|---|
| client → gateway | TLS, authentication, quota | pending |
| gateway → predictor | allowed network path, identity | pending |
| predictor → model storage | read-only workload identity | pending |
| predictor → metrics | restricted scrape | pending |
| operator/control plane | least-privilege RBAC | pending |
| tenant A → tenant B | denied by policy | pending |

Never use permissive wildcard IAM or log tokens for diagnostics.

## Lab 4 — Canary design [DESIGN]

```text
Release A stable: pinned image/model/tokenizer/runtime/gateway
Release B candidate: pinned image/model/tokenizer/runtime/gateway
Capability: validate weighted routing or choose blue-green
Synthetic golden checks: output + terminal streaming
Promotion gates: p95/p99, TTFT, TPOT, valid completions, VRAM
Rollback: restore complete route and release A bundle
```

Use actual installed CRD and gateway semantics; do not assume a universal canary YAML field.

## Lab 5 — Rollback and incident simulation [TABLETOP]

Simulate incorrect canary weight, TLS failure, 503 upstream, gateway timeout and candidate GPU OOM on paper. Record blast radius, detection, approval, mitigation, rollback and terminal validation.

## Lab 6 — Eight game days

01 DNS drift; 02 certificate expiry; 03 NetworkPolicy deny; 04 unsupported canary schema; 05 wrong traffic weight; 06 streaming timeout; 07 privacy leak; 08 route rollback drift.

## Twenty-four failure checklist

F01 DNS; F02 TLS; F03 SNI; F04 auth missing; F05 authorization; F06 Host; F07 protocol; F08 EndpointSlice; F09 readiness; F10 timeout; F11 stream; F12 NetworkPolicy; F13 storage egress; F14 IAM; F15 management exposure; F16 canary schema; F17 weight; F18 stale route; F19 tokenizer; F20 GPU OOM; F21 shadow privacy; F22 metrics; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | DNS | Pending |
| G03 | TLS/SNI | Pending |
| G04 | Gateway authn/authz | Pending |
| G05 | Tenant isolation | Pending |
| G06 | Service/EndpointSlice | Pending |
| G07 | Route/protocol | Pending |
| G08 | NetworkPolicy/CNI | Pending |
| G09 | Storage egress/identity | Pending |
| G10 | Management isolation | Pending |
| G11 | Canary schema/capability | Pending |
| G12 | Weights/cohort | Pending |
| G13 | Model provenance | Pending |
| G14 | Golden output/stream | Pending |
| G15 | Latency/error budget | Pending |
| G16 | GPU VRAM/quota | Pending |
| G17 | Privacy/security | Pending |
| G18 | Complete rollback | Pending |
| G19 | FinOps/telemetry | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/change approval:
KServe mode/controller/CRDs:
DNS/TLS/SNI/Host/path:
Gateway authn/authz and quotas:
Service/EndpointSlice:
NetworkPolicy and storage identity:
Canary/blue-green capability and route:
Model/tokenizer/image digests:
Golden output and streaming:
TTFT/TPOT/E2E and valid completion:
GPU VRAM/quota/cost:
Security and telemetry:
Game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop activities create no resources. Approved staging experiments must restore GitOps routes, remove only owned test resources, stop synthetic load, verify no orphaned GPU replicas and reconcile GPU-hours. Do not mark gates complete without independently reviewed evidence.

**Next: Chapter 044 — KServe Autoscaling and GPU Scheduling.**
