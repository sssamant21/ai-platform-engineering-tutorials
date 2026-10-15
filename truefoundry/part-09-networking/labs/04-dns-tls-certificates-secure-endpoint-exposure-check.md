# Chapter 9.4 — Hands-on Lab: DNS, TLS, Certificates and Secure Endpoint Exposure

**Part 09 | Isolated staging only | No production changes**

## Objectives, prerequisites and safety

Prove authoritative and private DNS behavior, ingress/Gateway routing, TLS identity, certificate lifecycle, backend reachability, secure exposure and rollback. Have a disposable namespace, approved test hostname, known ingress/Gateway controller, staging certificate issuer, `kubectl`, `dig` or `Resolve-DnsName`, `openssl`, `curl`, access to a private-network test host and a public test host if applicable. Never use production domains, live private keys or real customer traffic.

**Labels:** [SAFE-READ] inspection; [LAB-MUTATE] disposable staging only; [CHANGE-CONTROL] production requires approval. Store evidence with secrets redacted.

## Lab 0 — Inventory and baseline [SAFE-READ]

```bash
kubectl config current-context
kubectl get ingressclass,ingress -A
kubectl get gatewayclass,gateway,httproute -A 2>/dev/null || true
kubectl get certificate,issuer,clusterissuer -A 2>/dev/null || true
kubectl get svc,endpointslice -A
kubectl -n kube-system get pods -o wide | grep -Ei 'dns|coredns' || true
```

Record controller type/version, zone owner, hostname, expected DNS targets, TLS termination and backend encryption. Stop if the context or ownership is uncertain.

## Lab 1 — DNS resolution matrix [SAFE-READ]

```bash
export HOST=inference.lab.example.com # replace with an approved real lab hostname
dig "$HOST" A +noall +answer
dig "$HOST" CNAME +noall +answer
dig "$HOST" TXT +noall +answer
```

Repeat from an internal resolver, authorized VPC/VNet host and an external host. Record expected and actual addresses, TTL and public/private reachability. Do not interpret a missing public record as sufficient proof of isolation.

## Lab 2 — Route and backend verification [SAFE-READ]

```bash
kubectl -n tfy-net-lab get ingress,svc,pods,endpointslice
kubectl -n tfy-net-lab describe ingress inference-https
kubectl -n tfy-net-lab get events --sort-by=.lastTimestamp | tail -25
```

If using Gateway API, inspect Gateway listener `Accepted/Programmed` and HTTPRoute `Accepted/ResolvedRefs` conditions instead. Trace the selected hostname and path to a ready EndpointSlice.

## Lab 3 — TLS identity and redirect [SAFE-READ]

```bash
openssl s_client -connect "$HOST:443" -servername "$HOST" -showcerts </dev/null
curl -Iv "https://$HOST/"
curl -I "http://$HOST/"  # only where HTTP listener is intentionally exposed
```

Record SAN, issuer, expiry, chain, negotiated TLS version, redirect policy and application status. A 401/403 is not proof of app failure if authorization is expected. Never use `curl -k` to mark TLS accepted.

## Lab 4 — Isolated route and certificate change [LAB-MUTATE]

1. Create a disposable namespace and non-sensitive echo workload using organization-approved manifests.
2. Create a test hostname and routing object with explicit controller ownership.
3. Issue a lab-only certificate using the approved issuer (DNS-01 for private-only hostnames when appropriate).
4. Validate the new route, SNI, SAN, chain and authorized response.
5. Rotate to a second valid staging certificate or certificate version; observe listener reload and stream behavior.
6. Roll back to the prior **still-valid and uncompromised** certificate reference and verify recovery.
7. Remove only lab-owned resources, ensuring shared DNS zones and controller-owned records are not deleted.

No secret/key PEM is committed to Git or copied into incident notes. If no approved issuer or disposable environment exists, document the steps as **not executed**, rather than simulating success.

## Lab 5 — Certificate automation [SAFE-READ]

```bash
kubectl get certificate,certificaterequest,order,challenge -A 2>/dev/null || true
kubectl -n tfy-net-lab describe certificate "$CERT_NAME"
```

Inspect issuer readiness, challenge type, rate limits, renewal schedule, controller permissions, owner and alert routing. If cert-manager is absent, inspect the actual cloud-managed certificate system instead.

## Lab 6 — Exposure and authentication [SAFE-READ]

From approved internal and external vantage points, check DNS, TCP/TLS reachability and authorization responses. Verify private LB scheme/firewall restrictions and required application identity checks. Do not run unauthenticated scans or send production payloads.

## Failure injection matrix — 24 scenarios

Perform only those safely reproducible in isolated staging; otherwise use read-only evidence review. For each scenario record symptom, detection signal, isolation, recovery action, validation and rollback.

| # | Scenario | Primary investigation |
|---|---|---|
| 01 | Missing A/alias record | Authoritative zone and ownership |
| 02 | Wrong CNAME target | GitOps desired state and LB target |
| 03 | DNS TTL delays cutover | Resolver caching and propagation |
| 04 | Public/private split-horizon mismatch | Zone associations and resolver vantage |
| 05 | SERVFAIL from upstream resolver | CoreDNS and forwarding |
| 06 | NXDOMAIN due to wrong search suffix | Namespace and cluster domain |
| 07 | ExternalDNS ownership collision | TXT registry and zone filters |
| 08 | DNS record deleted with route | Reconciliation policy and Git history |
| 09 | Expired leaf certificate | Live listener expiry and issuer |
| 10 | Missing intermediate CA | Presented chain and trust store |
| 11 | Wrong certificate SAN | Hostname and SAN |
| 12 | Missing SNI | Default listener certificate |
| 13 | Wrong wildcard depth | SAN wildcard matching |
| 14 | TLS Secret missing | Namespace and controller events |
| 15 | Issuer not Ready | Issuer permissions and CA reachability |
| 16 | ACME HTTP-01 blocked | Port 80 and challenge path |
| 17 | ACME DNS-01 TXT not propagated | DNS provider IAM and TXT TTL |
| 18 | Cloud certificate not bound to listener | LB certificate reference |
| 19 | Gateway listener rejects certificateRefs | Gateway conditions and namespace scope |
| 20 | Backend TLS verification fails | Backend trust and hostname |
| 21 | Private LB accidentally public | Scheme, ACLs and external test |
| 22 | TLS success but 503 | EndpointSlice readiness and backend health |
| 23 | Auth bypass on alternate hostname | Host routes and authorization policies |
| 24 | Long-lived inference stream reset on rotation | Drain timeout and connection metrics |

## Production runbook rehearsal

Rehearse the canonical tutorial's RB-01 through RB-10 with a tabletop owner, escalation path, rollback and evidence capture. Do not cause production outages.

## Nine acceptance gates

| Gate | Required evidence | Result |
|---|---|---|
| G1 | Correct context, owners and documented exposure classification | Pending |
| G2 | Public/private DNS resolution matrix validated | Pending |
| G3 | DNS ownership and TTL/cutover documented | Pending |
| G4 | TLS SAN, SNI, chain and expiry verified from live listener | Pending |
| G5 | Certificate renewal/rotation owner and alerting verified | Pending |
| G6 | Gateway/Ingress route → Service → ready backend traced | Pending |
| G7 | Authentication, network restrictions and HTTPS policy tested | Pending |
| G8 | Staging rollback and streaming behavior validated | Pending |
| G9 | Evidence, exceptions, incident contacts and review sign-off recorded | Pending |

**Pass condition:** all applicable gates verified with evidence; blocked/not-executed gates are explicitly marked and cannot be counted as passed. Publication does not certify lab execution.

## Evidence worksheet

Record UTC time, cluster/context, sanitized hostname, zone owner, controller and version, certificate fingerprint (not key), expected vs actual DNS answer, TLS outcome, request status, LB health, test vantage point, gate result, owner and remediation ticket.

**Next:** Chapter 9.5 — NetworkPolicy, Private Networking and Egress Security.
