# Chapter 9.7 — Hands-on Lab: Production Networking and Recovery Capstone

**Part 09 final capstone | Staging-only fault injection | Formal execution pending**

## Mission and safety

Build and validate a production-like TrueFoundry inference networking path, then diagnose and recover from multi-layer faults without weakening authentication, TLS or isolation. Use an authorized disposable namespace `tfy-net-capstone`, synthetic model responses and an approved lab hostname.

**[SAFE-READ]** read-only inspection; **[LAB-MUTATE]** disposable staging resources only; **[CHANGE-CONTROL]** production requires approved change, on-call and rollback. Never modify shared DNS zones, production LB listeners, certificates, NetworkPolicies or customer-serving Pods for this lab.

## Step 0 — Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl get ns --show-labels
kubectl get ingressclass,ingress -A
kubectl get gatewayclass,gateway,httproute -A 2>/dev/null || true
kubectl get svc,endpointslice -A
kubectl get networkpolicy -A
kubectl get certificate,issuer,clusterissuer -A 2>/dev/null || true
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

Record cluster/context, CNI policy enforcement, controller and version, cloud LB and DNS owners, lab certificate issuer, TrueFoundry management/agent path, approved fault boundaries, security reviewer and abort thresholds. If ownership is unclear, stop.

## Step 1 — Reference deployment [LAB-MUTATE]

Deploy a non-sensitive streaming echo/mock model service with two replicas, explicit readiness and a ClusterIP Service. Create a scoped gateway/Ingress route with approved lab DNS and TLS. Apply least-privilege NetworkPolicy and only required private/approved egress paths. Configure per-hop metrics and sanitized request correlation. Store manifests in a reviewed GitOps branch and capture rollback commit.

## Step 2 — Establish baseline [SAFE-READ]

From approved client vantage points record: DNS answer and TTL; TCP and TLS handshake; SNI/SAN/chain; authorized HTTP response; unauthorized rejection; backend EndpointSlice readiness; first byte; first model token; event spacing; full completion marker; p95/p99 over an approved bounded test; DNS and CNI verdicts; telemetry and model artifact reachability.

## Step 3 — Security and isolation tests [SAFE-READ]

Verify external/public and internal/private reachability independently. Demonstrate no unintended public route, no auth bypass via alternate hostname, and denied cross-namespace and unapproved egress flows. Verify allowed DNS UDP/TCP, gateway-to-Pod, telemetry and private endpoint paths. If a test requires a security-sensitive mutation, use a tabletop instead.

## Step 4 — Fault injection and recovery [LAB-MUTATE]

For each approved scenario: declare start time; record LPHL/FFT; inject a **lab-owned** failure; observe alerts and application impact; identify the failed hop; apply the smallest safe recovery; validate full stream completion and security controls; roll back; capture elapsed detection/recovery times. Stop on unexpected cross-namespace impact, security exposure, resource exhaustion or shared-service degradation.

## Step 5 — Staged deployment and stream draining [LAB-MUTATE]

Run synthetic long-lived streams during a controlled rollout of the lab backend. Observe readiness withdrawal, EndpointSlice update, termination grace, LB/gateway drain and completion/cancellation semantics. Restore known-good version; prove no orphaned connections or unhealthy targets.

## Step 6 — Final rollback and cleanup [LAB-MUTATE]

Revert the lab GitOps commit and restore any lab-owned DNS/certificate/route/policy objects. Verify that shared DNS, LB and cloud resources were not modified. Re-run authorized and denied traffic checks, SLO probes and telemetry checks. Record remaining exceptions and owner sign-off.

## Failure and recovery matrix — 32 scenarios

| # | Scenario | Evidence / recovery focus |
|---|---|---|
| 01 | Authoritative DNS record missing | Zone ownership and record restore |
| 02 | Split-horizon DNS wrong | Private zone association |
| 03 | DNS TTL masks rollback | Resolver propagation |
| 04 | CoreDNS upstream failure | DNS metrics and forwarding |
| 05 | Certificate expired | Live listener certificate |
| 06 | SNI/SAN mismatch | Correct hostname and certificate |
| 07 | Missing certificate intermediate | Served chain |
| 08 | Renewal controller stalled | Issuer/challenge status |
| 09 | Wrong LB target | Target group/backend health |
| 10 | Cloud LB listener missing | Listener/route ownership |
| 11 | Ingress host mismatch | Host/path routing |
| 12 | Gateway HTTPRoute rejected | Accepted/ResolvedRefs conditions |
| 13 | Service selector mismatch | Ready EndpointSlice |
| 14 | Readiness flapping | Pod health and rollout |
| 15 | Gateway-to-backend ingress denied | NetworkPolicy selectors |
| 16 | Source Pod egress denied | Source policy verdict |
| 17 | DNS blocked by egress | UDP/TCP 53, NodeLocal DNS |
| 18 | Untrusted namespace unexpectedly allowed | Union-of-allows and CNI |
| 19 | Private endpoint publicly reachable | LB scheme and cloud ACL |
| 20 | Private DNS resolves to public address | Zone links and resolver |
| 21 | Model artifact download fails | Pod egress, DNS and IAM |
| 22 | Node image pull fails | Node runtime network path |
| 23 | Telemetry collector unreachable | OTLP or scrape policy |
| 24 | TrueFoundry agent connectivity loss | Management egress and identity |
| 25 | 502 upstream reset | Pod/proxy logs |
| 26 | 503 no healthy backend | Service/EndpointSlice |
| 27 | 504 timeout | Limiting timeout layer |
| 28 | SSE events buffered | Proxy buffering/flush |
| 29 | Fixed-time stream disconnect | Idle/absolute deadline |
| 30 | High TTFT under queue pressure | Queue/prefill/GPU metrics |
| 31 | Rolling deployment drops streams | Drain and termination grace |
| 32 | GitOps restores faulty state after manual fix | Desired-state revert |

**Each scenario record:** expected result, fault owner, timestamp, observed symptoms, affected SLO, alert, request ID, evidence, root-cause hypothesis, recovery, rollback, verification and follow-up. Scenarios unsafe to inject are tabletop-only and marked accordingly.

## Twelve acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved staging scope, owners and versioned architecture | Pending |
| G2 | Full DNS→LB→gateway→Service→Pod trace proven | Pending |
| G3 | TLS certificate/SNI/chain and auth validated | Pending |
| G4 | EndpointSlice readiness, health and routing validated | Pending |
| G5 | NetworkPolicy enforcement and negative tests proven | Pending |
| G6 | Private/public exposure and cloud egress restrictions verified | Pending |
| G7 | DNS, artifact, telemetry and management dependencies validated | Pending |
| G8 | First byte, TTFT, inter-token and completion metrics captured | Pending |
| G9 | Timeout/buffering and active-stream drain tested | Pending |
| G10 | Fault detection, incident command and recovery rehearsed | Pending |
| G11 | GitOps rollback and complete resource cleanup verified | Pending |
| G12 | SLO/security evidence, exceptions and formal reviewer sign-off | Pending |

**Pass:** every applicable gate has timestamped evidence and reviewer approval; unexecuted/blocked gates cannot count as passed. A published lab is not a passed lab.

## Evidence and sign-off template

- UTC start/end, environment, context, region and change ID
- Architecture version, Git commit, DNS and LB owner, CNI/controller versions
- Certificate fingerprint (never private key), policy and route selectors
- Request correlation IDs and redacted DNS/TLS/gateway/CNI/model metrics
- Baseline and post-recovery SLO measurements, LPHL/FFT and recovery duration
- Failure scenario outcomes, residual risks, rollback proof and cleanup evidence
- Application owner, SRE, cloud networking and security reviewers; sign-off date

## Part 09 publication closeout

All seven chapters and seven companion labs are published when this capstone is merged and indexed. Formal technical reviews, hands-on execution and operational certification remain pending until performed and recorded.

**Next:** Part 10 — Security, beginning with the chapter defined by its existing master layout.
