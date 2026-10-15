# Chapter 9.4 — DNS, TLS, Certificates and Secure Endpoint Exposure

**Part 09 — Networking | Canonical production tutorial**

## Learning objectives and operating contract

Trace DNS and TLS from client to inference workload; distinguish public and private exposure; inspect Kubernetes DNS, ingress/gateway certificate configuration and renewal; detect mismatched SNI, SAN and routing; execute staged rotation and safe rollback. All commands are examples: identify the actual TrueFoundry runtime, controller, cloud account and namespace before applying. TrueFoundry product-specific capabilities must be verified against the installed version, not assumed.

**Safety legend:** `[SAFE-READ]` non-mutating inspection; `[LAB-MUTATE]` only in an isolated disposable staging cluster; `[CHANGE-CONTROL]` requires approvals, tested rollback, owner and maintenance window.

## 1. End-to-end request and trust boundaries

```text
Client → resolver → authoritative DNS (public/private)
       → cloud load balancer → TLS listener/SNI selection
       → ingress or Gateway API HTTPRoute → Service → EndpointSlice
       → ready application/model Pod
```

A TrueFoundry management/control-plane endpoint is not automatically the same endpoint as a workload's runtime inference route. Inventory both independently. Capture hostname, exposure type, DNS zone owner, CNAME/alias/A record, ingress/gateway class, TLS termination point, backend encryption, identity enforcement, target Service, health checks, and streaming behavior.

| Boundary | Security decision | Operational proof |
|---|---|---|
| Public vs private DNS | Who can resolve the name | Public authoritative lookup and VPC/VNet resolver lookup |
| Load balancer | Who can connect | Scheme, firewall/security groups, listener, source restrictions |
| TLS edge | Which certificate and key | SAN, validity, chain, SNI, negotiated protocol |
| Gateway → backend | Is traffic re-encrypted | Backend TLS policy and observed connection |
| Application | Who can invoke inference | Authentication/authorization and audit evidence |
| Cluster DNS | How Services resolve | CoreDNS health and Service/EndpointSlice records |

**Threat model:** DNS visibility is not authorization. A private DNS name can still point to a public listener, and a private listener can be reached through unintended network paths. TLS provides encryption and endpoint authentication but does not replace workload authorization.

## 2. Discover actual infrastructure [SAFE-READ]

```bash
kubectl config current-context
kubectl get namespaces
kubectl get ingressclass,ingress -A
kubectl get gatewayclass,gateway,httproute -A 2>/dev/null || true
kubectl get svc,endpointslice -A
kubectl -n kube-system get deploy,svc | grep -Ei 'dns|coredns' || true
kubectl get certificate,issuer,clusterissuer -A 2>/dev/null || true
kubectl get pods -A | grep -Ei 'external-dns|cert-manager|ingress|gateway' || true
```

Do not install a controller merely because these APIs are absent. Record controller ownership, version, cloud LB annotations and desired-state Git repository. In multi-cluster installations, repeat discovery in each approved context.

## 3. DNS design and diagnosis

Public authoritative DNS is often managed by Route 53 or Azure DNS; internal resolution may use Route 53 private hosted zones, Azure Private DNS, corporate conditional forwarders or custom resolvers. Split-horizon names can return different answers inside and outside the network. Negative caching, TTL and resolver propagation affect cutover time; do not assume immediate global convergence.

```bash
# [SAFE-READ] Replace with a non-sensitive, approved hostname
export HOST=api.example.com
dig +short "$HOST"
dig "$HOST" A +noall +answer
dig "$HOST" CNAME +noall +answer
dig "$HOST" NS +noall +answer
dig +trace "$HOST"  # may be blocked or incomplete for private zones
```

On Windows use `Resolve-DnsName api.example.com`. Test separately from a trusted private-network host and an external vantage point; never infer public exposure solely from an internal DNS answer.

Inside Kubernetes, `service.namespace.svc.cluster.local` depends on cluster domain and namespace. Validate the configured search domains, CoreDNS availability, NetworkPolicy and upstream forwarding. For headless Services and endpoint readiness, examine EndpointSlices rather than assuming a Service DNS answer proves a ready backend.

ExternalDNS, if installed, reconciles DNS from Kubernetes resources. Scope it by zone, ownership TXT registry, policy, sources and domain filters; two controllers without distinct ownership can overwrite records. Deleting a Kubernetes route may delete its managed DNS record, depending on policy.

## 4. TLS certificate chain, SNI and termination

A valid endpoint requires: certificate not expired; matching DNS SAN; complete trusted intermediate chain; appropriate private key; supported protocol/cipher; and correct SNI routing. Wildcards only match one label level (`*.example.com` does not match `a.b.example.com`). A certificate can be valid yet served from the wrong listener.

```bash
# [SAFE-READ] Review served certificate and negotiated TLS
openssl s_client -connect "$HOST:443" -servername "$HOST" -showcerts </dev/null
curl -vI "https://$HOST/"    # status 401/403 can prove TLS succeeds; don't treat as app success
curl --resolve "$HOST:443:203.0.113.10" -I "https://$HOST/" # TEST-NET-3 example IP only
```

Use `openssl x509 -noout -dates -subject -issuer -ext subjectAltName` against an approved PEM copy. Never paste private keys, token-bearing URLs or full certificate Secrets into tickets. `kubectl describe secret` may expose metadata; `kubectl get secret -o yaml` exposes key material in base64 and is not a routine diagnostic.

TLS patterns:
1. **Edge termination:** HTTPS client → LB/gateway; backend HTTP inside a controlled boundary.
2. **Re-encryption:** HTTPS client → edge; separate verified TLS connection edge → backend.
3. **Passthrough:** TLS negotiated by workload; routing and observability constraints differ.
4. **mTLS:** mutual identity validation at supported gateway/service-mesh boundary; requires certificate lifecycle and revocation planning.

Choose a documented pattern per workload and data classification. Avoid claiming end-to-end encryption when the backend hop is HTTP.

## 5. Certificate lifecycle and automation

Common deployment-specific options: cloud-managed certificates (AWS ACM, Azure Key Vault integration where supported), cert-manager with ACME or private CA, or enterprise PKI. AWS ACM certificates attached to supported AWS listeners can renew automatically subject to eligibility and validation. Azure Key Vault certificates are not automatically hot-reloaded by every controller; validate integration. cert-manager depends on issuer, solver, permissions, DNS/HTTP reachability and controller readiness.

```bash
# [SAFE-READ]
kubectl get certificate,certificaterequest,order,challenge -A 2>/dev/null || true
kubectl describe certificate -n "$NAMESPACE" "$CERT_NAME"
kubectl get events -n "$NAMESPACE" --sort-by=.lastTimestamp | tail -30
kubectl describe ingress -n "$NAMESPACE" "$INGRESS_NAME"
```

**Certificate rotation [CHANGE-CONTROL]:**
1. Confirm owner, current issuer, DNS names, expiry and live listener certificate fingerprint.
2. Check issuance rate limits, domain validation ownership, automation health, staging environment and alert coverage.
3. Provision the replacement with overlapping validity; keep previous version recoverable according to key-management policy.
4. Validate SAN/chain/SNI and backend routing in staging; verify auth and long-running streaming connections.
5. Roll out via GitOps or approved cloud change; watch error rate, TLS handshake failures, latency and LB health.
6. Verify the new certificate from every relevant ingress/LB endpoint, not merely the Kubernetes Secret.
7. Roll back listener/certificate reference if validation fails; do not roll back to expired or compromised keys.
8. Revoke old material if required by incident policy, document evidence and monitor renewal horizon.

**ACME challenge selection:** HTTP-01 needs reachable port 80/path handling; DNS-01 needs authorized TXT updates and propagation. Private-only endpoints commonly require DNS-01 or private CA, depending on trust requirements. Avoid exposing private applications publicly just to pass a challenge.

## 6. Example Kubernetes configuration [LAB-MUTATE ONLY]

These examples are illustrative, not a recommendation to install or switch production controllers.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: inference-https
  namespace: tfy-net-lab
spec:
  ingressClassName: example-controller
  tls:
    - hosts: [inference.lab.example.com]
      secretName: inference-lab-tls
  rules:
    - host: inference.lab.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: inference-echo
                port:
                  number: 8080
```

The TLS Secret must already be provisioned by the approved issuer; never commit private keys to Git. Some cloud load balancers use certificate identifiers/annotations rather than Kubernetes TLS Secrets. Gateway API uses a Gateway listener with `certificateRefs` and attached HTTPRoutes, subject to supported controller and ReferenceGrant behavior. Do not combine two active controllers on the same production hostname without explicit ownership.

## 7. Cloud-specific considerations

**AWS EKS:** inspect Route 53 public/private hosted-zone association, ALB/NLB scheme, target health, ACM certificate ARN and validation state, listener rules, security groups, AWS Load Balancer Controller permissions and ExternalDNS TXT ownership. TLS termination at ALB does not guarantee TLS to Pods.

**Azure AKS:** inspect Azure DNS vs Private DNS zone links, Application Gateway or other ingress architecture, managed identity/RBAC, Key Vault access and certificate version selection, backend health and NSG rules. AKS clusters can use several ingress models; inspect before applying controller-specific commands.

## 8. Observability, SLOs and secure exposure acceptance

Capture DNS lookup failures, NXDOMAIN/SERVFAIL, cert days-to-expiry, issuance/renewal errors, TLS handshake failures, HTTP 4xx vs 5xx, LB unhealthy targets, gateway route rejection, and request/stream interruption. Synthetic probes should cover external/public and private-network vantage points. Alert before renewal safety margin, with owner and escalation; use a policy-defined threshold (for example 30/14/7 days) rather than treating a single value as universal.

Acceptance evidence: correct public/private reachability, TLS identity/chain, issuer ownership, automation/renewal path, HTTPS redirect where required, least-privilege network access, authn/authz, healthy backends, streaming behavior and tested rollback.

## 9. Production runbooks

### RB-01 — NXDOMAIN or unexpected DNS answer
[SAFE-READ] Compare authoritative records, private zone links, resolver vantage points, TTL, ExternalDNS ownership and recent Git changes. [CHANGE-CONTROL] Restore the intended record only after identifying authoritative owner; avoid blind edits to records managed by GitOps/ExternalDNS.

### RB-02 — SERVFAIL or intermittent DNS
[SAFE-READ] Check CoreDNS health, upstream resolvers, zone delegation, query volume and NetworkPolicy. [CHANGE-CONTROL] Escalate upstream failures or adjust approved resolver configuration; measure post-change error rates.

### RB-03 — Certificate expired or near expiry
[SAFE-READ] Inspect served certificate, issuer status, DNS validation and renewal controller events. [CHANGE-CONTROL] Replace through approved issuer; verify the actual LB listener and every SNI hostname.

### RB-04 — Certificate name mismatch
[SAFE-READ] Confirm client hostname, SNI, SAN, ingress host rule, listener default certificate and DNS destination. [CHANGE-CONTROL] Correct routing/certificate reference; never disable certificate verification as a fix.

### RB-05 — Incomplete trust chain or untrusted CA
[SAFE-READ] Compare served intermediates and client trust store. [CHANGE-CONTROL] Repair chain or deploy approved enterprise CA trust; do not bypass TLS verification.

### RB-06 — TLS works but inference returns 404/502/503
[SAFE-READ] Follow hostname → gateway route → Service → EndpointSlice → ready Pod, then check application logs and auth. [CHANGE-CONTROL] Restore known-good routing/backend release with GitOps and confirm health.

### RB-07 — Private endpoint accidentally public
[SAFE-READ] Capture DNS answers, LB scheme, reachable listener and source ACLs. [CHANGE-CONTROL / SECURITY INCIDENT] Apply approved containment (LB/firewall/network restriction), preserve evidence, rotate exposed credentials if indicated and verify from outside network.

### RB-08 — Certificate rotation causes streaming disconnects
[SAFE-READ] Correlate listener reloads, connection resets, gateway draining and SSE/WebSocket duration. [CHANGE-CONTROL] Use staged listener rollout and graceful draining; restore known-good certificate reference if still valid and safe.

### RB-09 — ACME validation stuck
[SAFE-READ] Inspect Order/Challenge, DNS TXT or HTTP-01 path, solver permissions and propagation. [CHANGE-CONTROL] Fix owner-controlled configuration, respect CA rate limits and verify issuance before switching traffic.

### RB-10 — DNS cutover points to wrong cluster
[SAFE-READ] Compare desired Git state, zone ownership, CNAME/alias targets, LB health and split-horizon answers. [CHANGE-CONTROL] Restore previous healthy record, account for TTL and validate auth/tenant isolation.

## 10. Change review and rollback checklist

Record: change ticket; hostname/zone; public/private classification; LB/controller and owner; certificate fingerprint/expiry/issuer; old/new routing targets; DNS TTL; backup certificate reference; test vantage points; on-call owner; metrics and abort thresholds; recovery steps; timestamps and evidence. Never remove a valid fallback until new certificate and endpoint are independently verified.

## 11. Completion criteria

Complete the companion [hands-on lab](../labs/04-dns-tls-certificates-secure-endpoint-exposure-check.md), capture all nine acceptance gates and sign off any production change through the normal security/SRE process. Publication is not proof of execution or vendor certification.
