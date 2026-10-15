# Chapter 9.7 — Production Networking and Recovery Capstone

**Part 09 — Networking | Final integrated production-operations project**

## 1. Mission, scope and safety

Design, deploy, observe, deliberately test and recover a **production-like TrueFoundry inference network** in an authorized isolated staging environment. Integrate Chapter 9.1 end-to-end request paths, 9.2 Services and EndpointSlices, 9.3 ingress/Gateway API and cloud load balancers, 9.4 DNS/TLS, 9.5 network isolation and private egress, and 9.6 streaming inference/timeouts.

The outcome is an evidence-backed **network reliability acceptance dossier**, not a claim that a live production cluster was tested.

**Safety labels:** `[SAFE-READ]` read-only inspection; `[LAB-MUTATE]` disposable staging resources only; `[CHANGE-CONTROL]` approved production changes with SRE/security sign-off. No customer traffic, credentials, PHI, real model prompts or unapproved network scans. Verify actual TrueFoundry topology, Kubernetes CNI, gateway/controller, cloud and application protocol; do not assume any TrueFoundry feature is universally available.

## 2. Architecture and system boundaries

```text
Authorized client / synthetic load generator
    ↓ DNS (public or private zone) → resolver and zone owner
    ↓ TLS / SNI / certificate chain
    ↓ Cloud LB + firewall/SG/NSG + health check
    ↓ Ingress/Gateway listener + auth + route + timeout policy
    ↓ Kubernetes Service → EndpointSlice → ready inference Pod
    ↓ model queue/prefill/decode → streamed response
    ↘ DNS / private endpoints / approved model artifact store
    ↘ internal telemetry collector / metrics / logs
TrueFoundry management plane ↔ installed agent (separate trust path)
```

Document each hop with owner, identity, protocol, port, DNS name, certificate boundary, encryption state, exposure type, dependencies, health signals and rollback. Management-plane access and data-plane inference must be evaluated separately.

## 3. Capstone deliverables

1. Architecture diagram and annotated traffic-flow matrix.
2. Versioned staging manifests/GitOps commit for test namespace, echo or mock inference backend, Service, ingress/Gateway route, DNS/TLS and NetworkPolicies.
3. Security evidence for authentication, private/public exposure, least-privilege egress and negative access tests.
4. Baseline metrics for availability, HTTP errors, DNS/TCP/TLS setup, first byte, TTFT, inter-token latency, p95/p99, concurrent streams, backend readiness and model load.
5. Fault-injection evidence for at least **30 scenarios**, with impact, detection, recovery and rollback.
6. Recovery playbook, communication timeline, abort thresholds, ownership/RACI and SLO validation.
7. Acceptance report with **12 gates**, reviewer, timestamps, exceptions and sign-off.

## 4. Baseline and readiness [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces --show-labels
kubectl get ingressclass,ingress -A
kubectl get gatewayclass,gateway,httproute -A 2>/dev/null || true
kubectl get svc,endpointslice -A
kubectl get networkpolicy -A
kubectl get certificate,issuer,clusterissuer -A 2>/dev/null || true
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

Record CNI and enforcement mode, TrueFoundry agent placement, LB/controller ownership, cloud private endpoint design, GitOps reconciler, synthetic load limits, on-call contacts and recovery owner. If any of these are unknown, pause mutation and complete inventory.

## 5. Build the isolated reference deployment [LAB-MUTATE]

Use a disposable namespace `tfy-net-capstone`, synthetic echo or mock token-stream backend, two or more replicas and explicit readiness/startup/liveness probes as appropriate. Publish only through an approved staging gateway/ingress class and hostname. Use an approved lab-only certificate and DNS ownership; no private keys in Git.

- **Service:** ClusterIP on the actual backend port, selector matches ready Pods; verify EndpointSlice readiness.
- **Gateway/Ingress:** host/path match, TLS listener, authenticated access, correct health check and streaming-compatible buffering/timeouts where supported.
- **NetworkPolicy:** default-deny with narrowly scoped ingress from gateway and egress to actual DNS, telemetry and approved dependencies. Confirm CNI enforcement with negative tests.
- **Private egress:** use cloud private endpoints or an approved proxy/firewall; validate cloud IAM separately from network reachability.
- **Observability:** correlation ID, request completion marker, per-hop latency, proxy status, policy drop verdicts, TLS expiry and model-server queue metrics.
- **GitOps:** stage each change in a reviewable commit, prove desired-state convergence and capture the revert procedure.

**No example manifest is universally deployable:** copy the reviewed, CNI/controller-specific configurations from Chapters 9.2–9.6 and adapt selectors, DNS and certificate references to your real staging architecture.

## 6. Define SLOs, error budget and abort thresholds

Choose **measured, service-owned** targets rather than inventing universal TrueFoundry SLOs. Suggested dimensions:
- Availability: fraction of authorized inference requests that complete at the application protocol level.
- Latency: p95/p99 TTFT, first-byte latency, inter-token latency and total completion duration, segmented by model, input/output token count and concurrency.
- Reliability: 502/503/504, client cancellations, midstream resets, healthy EndpointSlice count and successful deployment drains.
- Security: unauthorized ingress and unapproved egress blocked, certificate validity, auth enforced, private endpoint not publicly reachable.
- Dependencies: DNS success, model artifact availability, telemetry delivery and agent connectivity.

Establish baseline, alert thresholds, rollback triggers, observation period and an explicit **stop-test** limit for saturation. A response with HTTP 200 but truncated SSE stream is **not** a successful completion.

## 7. Integrated test plan

### Phase A — Connectivity and identity
Trace DNS → TCP → TLS/SNI → gateway → Service → EndpointSlice → Pod. Verify split-horizon answers and cloud LB scheme. Confirm SAN/chain, auth rejection without credentials, authorized success, and no unintended alternate host/path bypass.

### Phase B — Policy and dependencies
Demonstrate gateway ingress and approved DNS/telemetry/model artifact egress while blocking untrusted namespaces and external destinations. Distinguish Pod model download from node image pull. Prove private endpoint route, resolver and cloud firewall/identity.

### Phase C — Streaming and capacity
Measure real event timestamps, first byte vs first model token, queue/prefill, concurrent streams, response buffering and slow-reader backpressure. Test bounded load only; stop on GPU memory, queue, proxy connection or error-budget limits.

### Phase D — Failure and recovery
Inject only authorized staging faults: wrong route, missing ready endpoints, bad DNS record in **lab-owned zone**, invalid **lab-only** certificate reference, missing policy allow, gateway timeout, buffering, Pod restart and staged rollout. Capture first failure, impact, mitigation, restore and post-recovery metrics.

### Phase E — Security and rollback
Revert GitOps changes to known-good state, verify negative access tests still deny unapproved flows, and confirm no orphaned LB, certificate, DNS, private endpoint or namespace resources.

## 8. Incident command and recovery sequence

1. **Declare incident:** record UTC first failure timestamp (FFT), last known healthy/progress point (LPHL), customer-visible impact, affected route/model/region and severity.
2. **Stabilize:** stop unsafe changes and load, preserve evidence, assign incident commander, SRE, networking/cloud, security and application owners.
3. **Find the failing hop:** DNS → LB/TLS → gateway → Service/EndpointSlice → Pod → model → egress dependency. Correlate sanitized request IDs and CNI verdicts.
4. **Choose recovery:** rollback route/policy/certificate, restore healthy backends, reduce load or use an approved failover target. Do not bypass TLS/auth or broadly open egress.
5. **Validate:** authorized request success and full stream completion, TTFT/ITL baseline, backend readiness, private exposure and negative access tests.
6. **Communicate:** update impact and recovery status with evidence, uncertainty and next checkpoint.
7. **Close:** document timeline, root cause/contributing factors, permanent corrective actions, monitoring gaps and next drill.

## 9. Twelve production runbooks

### RB-01 — DNS outage or wrong target
Compare authoritative and private-zone records, TTL, ExternalDNS ownership and resolver paths. Restore lab-owned known-good record through approved change, validate inside/outside network.

### RB-02 — TLS certificate failure
Check live listener certificate, SNI/SAN, chain, issuer and renewal. Rebind valid approved certificate; do not disable verification or expose keys.

### RB-03 — LB/gateway route mismatch
Inspect listener, hostname, HTTPRoute/Ingress conditions, target health and Service backend. Revert last bad route change and verify correct tenant/auth boundary.

### RB-04 — No ready endpoints
Inspect EndpointSlices, readiness probes, rollout and capacity. Restore known-good healthy workload; verify probe and route.

### RB-05 — Default-deny breaks inference
Trace source/destination selectors and CNI verdicts; add only the verified allow path or revert the policy commit. Preserve deny posture for unauthorized sources.

### RB-06 — Private endpoint unexpectedly public
Treat as security incident; inspect LB scheme, cloud firewall/SG/NSG, DNS and alternate hostnames. Contain exposure with approved controls and preserve evidence.

### RB-07 — Model initialization fails
Separate node image pull from Pod artifact download; check DNS, cloud IAM, private endpoint, egress policy and storage. Restore approved dependency.

### RB-08 — Telemetry disappears
Validate collector reachability, scraping vs push, namespace labels, agent and gateway metrics. Restore observability and avoid declaring success while blind.

### RB-09 — High TTFT and queue saturation
Compare first byte, first token, model queue, GPU load, cold starts and request mix. Reduce load or scale within approved capacity, validate completion.

### RB-10 — Fixed-duration stream resets
Inventory client/LB/gateway/mesh/server timeouts, heartbeats, proxy buffering and connection drains. Correct the limiting layer in staging before promotion.

### RB-11 — Rollout interrupts long-lived streams
Check readiness withdrawal, preStop, termination grace, LB deregistration delay and active connections. Revert deployment or implement tested graceful draining.

### RB-12 — Recovery regression / incomplete rollback
Check GitOps reconciliation, DNS TTL, cloud firewall changes, certificate bindings, policy union and backend readiness. Restore complete last-known-good state and retest security and SLO gates.

## 10. Failure scenarios to rehearse

The companion lab provides **32 scenario checks** across DNS, TLS, ingress/Gateway, cloud LB, Service/EndpointSlice, NetworkPolicy, private egress, agent dependencies, streaming, GPU load, observability and rollback. Every scenario needs a detection signal, diagnostic proof, corrective action, rollback and post-recovery verification. Simulate only in isolated staging; use tabletop evidence review where mutation would risk shared infrastructure.

## 11. Change control, rollback and evidence

Use a reviewed GitOps commit, defined maintenance window and abort criteria. Capture pre/post config hashes and sanitized logs. Roll back **all related layers**—DNS, LB, gateway, policy, workload and cloud routes—as required; do not assume a `kubectl rollout undo` restores DNS or cloud infrastructure. Recheck security negative tests after rollback.

**Acceptance:** all 12 gates in the [companion lab](../labs/07-production-networking-recovery-capstone-check.md) have timestamped evidence and owner sign-off. Publication alone is not proof of exercise completion.

## 12. Part 09 closeout

With Chapter 9.7 published, all **seven canonical chapters and seven companion labs** in Part 09 are available. **Formal vendor/SRE review, actual lab execution, and production readiness approval remain separate pending activities.**

**Next curriculum area:** Part 10 — Security. Verify the existing Part 10 layout before starting its first chapter.
