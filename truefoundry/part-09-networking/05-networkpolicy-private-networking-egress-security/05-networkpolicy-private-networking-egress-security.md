# Chapter 9.5 — NetworkPolicy, Private Networking and Egress Security

**Part 09 — Networking | Production-oriented canonical tutorial**

## Learning outcomes and safety

Design and verify default-deny network segmentation for TrueFoundry workloads; protect DNS, ingress, inference backends, platform agents, model artifact downloads and observability; distinguish Kubernetes NetworkPolicy from cloud firewall, private endpoint and service-mesh controls; implement least-privilege egress; rehearse failure containment and rollback.

**Execution labels:** **[SAFE-READ]** inspect only; **[LAB-MUTATE]** isolated staging namespace only; **[CHANGE-CONTROL]** approved production change with owner, rollback and monitoring. Never apply sample policies to production namespaces. Verify actual TrueFoundry version, agent topology, cloud, CNI and workload paths rather than assuming a particular implementation.

## 1. Architecture and trust boundaries

```text
Developer/CI → TrueFoundry control plane
                       ↕ management/agent channel (implementation dependent)
Authorized client → private/public LB → gateway/ingress → Service
                       → model-serving Pod → internal data stores
                                         → artifact registry / object storage
                                         → approved model APIs
                                         → telemetry collectors
Kubernetes Pod → CoreDNS → approved DNS resolver
```

A management/control-plane route is distinct from inference data-plane traffic. Build an explicit flow inventory before enforcing restrictions. A working application endpoint does **not** prove all required startup, rollout, image pull, metrics or streaming paths are allowed.

| Traffic class | Source → destination | Required controls |
|---|---|---|
| Inference ingress | gateway → serving Pod | authenticated route, namespace/pod identity, allowed port |
| Control-plane management | agent → configured endpoints | documented TLS, identity, approved destination |
| DNS | workload → cluster DNS | UDP/TCP 53 where needed, resolver policy |
| Model initialization | workload → artifact/object store | approved destination, IAM, private endpoint where available |
| Telemetry | workload → collector | allowed OTLP/metrics ports, least-privilege |
| Dependencies | workload → internal DB/cache | service identity, restricted port, TLS |
| External model API | workload → approved provider | egress proxy/gateway, FQDN and credentials controls |
| Node-originated flows | kubelet/runtime → registry/cloud | cloud routing/firewall; often not governed by Pod NetworkPolicy |

## 2. NetworkPolicy semantics and limitations

Kubernetes NetworkPolicy is a **Layer 3/4** policy. It selects Pods with `podSelector` in its own namespace. Ingress and egress isolation are **independent**: a Pod becomes isolated in a direction when any policy selecting it includes that direction. Allowed connections are the **union** of applicable policies, not a sequential firewall rule list. For Pod-to-Pod traffic, both source egress and destination ingress must permit the flow if both directions are isolated.

NetworkPolicy enforcement requires a compatible CNI/network policy engine. A policy API object may exist without effective enforcement. Standard NetworkPolicy does not offer portable FQDN, HTTP path, identity-aware L7 or explicit deny rules. DNS-based policy, global policies and host endpoint enforcement are CNI-specific; verify supported features and precedence. Node traffic, hostNetwork Pods, service NAT ordering and return-path behavior vary by CNI/cloud implementation.

**Never infer safety from policy YAML alone.** Validate connectivity from actual source Pods, destination Pods and representative cloud networks.

## 3. Discover platform and CNI [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl -n kube-system get ds,deploy | grep -Ei 'cilium|calico|aws-node|azure|antrea|weave' || true
kubectl get networkpolicy -A
kubectl get namespaces --show-labels
kubectl get pods -A -o wide
kubectl get svc,endpointslice -A
kubectl get ingressclass,ingress -A
kubectl get gatewayclass,gateway,httproute -A 2>/dev/null || true
```

Check controller documentation and operational configuration for enforcement; for example, Cilium, Calico and cloud-specific CNIs can have different defaults. Record the effective CNI, version, network policy mode, DNS setup, ingress controller namespace, telemetry namespace, service mesh sidecars, `hostNetwork` workloads, and TrueFoundry agent namespaces.

## 4. Label strategy and staged rollout

Create stable labels for workload role, environment and namespace purpose; prefer Kubernetes `kubernetes.io/metadata.name` namespace selector only when verified. Never use a broad namespace selector as a substitute for identifying trusted gateway Pods. A selector that matches **zero** Pods silently blocks traffic; a selector that matches **too many** Pods can widen access.

Rollout sequence:
1. Inventory required flows using application traces, gateway metrics, DNS queries, CNI flow logs and owner confirmation.
2. Record pre-change health, deployment rollout state, request latency, error rate and model load time.
3. Create an isolated lab namespace with disposable Pods and a working connectivity baseline.
4. Apply allow policies first; prove critical flows.
5. Add default-deny ingress and egress in the lab; rerun full acceptance.
6. Promote through GitOps in staging using canary workloads/namespace where possible.
7. Request production change approval; set rollback trigger thresholds and watch during rollout.
8. Revalidate startup, autoscaling, upgrades, telemetry, inference streaming and recovery—not only one HTTP request.

## 5. Policy examples [LAB-MUTATE ONLY]

These manifests use the namespace `tfy-net-lab` and **illustrative labels**. Confirm all Pod labels and the gateway namespace before reuse.

### 5.1 Default deny ingress and egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: tfy-net-lab
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
```

This isolates selected Pods in both directions; it can immediately break DNS, application access, dependencies and observability until explicit allows exist.

### 5.2 Allow cluster DNS

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cluster-dns
  namespace: tfy-net-lab
spec:
  podSelector:
    matchLabels:
      app: inference-echo
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**Caution:** DNS Pod labels and DNS topology differ across installations. NodeLocal DNSCache may run on node-local IPs and not match this selector. Validate actual resolver IP, CNI behavior and both UDP/TCP DNS before enforcing.

### 5.3 Allow only the trusted ingress gateway

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-gateway-to-inference
  namespace: tfy-net-lab
spec:
  podSelector:
    matchLabels:
      app: inference-echo
  policyTypes: [Ingress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: gateway-system
          podSelector:
            matchLabels:
              app.kubernetes.io/name: approved-gateway
      ports:
        - protocol: TCP
          port: 8080
```

The gateway's source egress policy must also allow the destination. If an external LB connects directly to NodePort/Pods or source IP is SNATed, actual source identity may differ. Verify datapath before applying.

### 5.4 Allow internal telemetry collector

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-otlp-egress
  namespace: tfy-net-lab
spec:
  podSelector:
    matchLabels:
      app: inference-echo
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: observability
          podSelector:
            matchLabels:
              app.kubernetes.io/name: otel-collector
      ports:
        - protocol: TCP
          port: 4317
        - protocol: TCP
          port: 4318
```

Verify collector labels and whether telemetry is pushed, scraped or sidecar-mediated. Scraping requires collector-to-workload ingress instead.

## 6. Egress controls and private networking

Standard NetworkPolicy `ipBlock` is IP/CIDR-based, not domain-aware. Public SaaS, artifact stores and managed services can change IPs; do not hard-code transient IP addresses as a long-term security design. Use supported egress gateway/proxy, CNI FQDN controls (if explicitly supported), private endpoints or firewall policy and inspect DNS-to-IP translation behavior. DNS allowlisting alone does not enforce egress restrictions.

**AWS EKS:** evaluate private subnets, NAT, VPC endpoints/PrivateLink for ECR/S3/STS/Secrets Manager as applicable, security groups for Pods where enabled, route tables, NACLs, VPC DNS and VPC Flow Logs. AWS VPC CNI NetworkPolicy support depends on version/configuration; confirm it is enabled and functioning. An EKS private API endpoint is not proof that inference traffic is private.

**Azure AKS:** evaluate VNet/subnets, Private Link/Private Endpoints for ACR/Storage/Key Vault as applicable, Azure Firewall, NSGs, UDRs, private DNS zone links, managed identity and chosen CNI/network-policy engine. AKS private cluster API endpoint is distinct from private workload ingress.

**Model and container artifact pulls:** kubelet/container runtime image pulls generally occur at node level, not via a Pod egress NetworkPolicy. Model files fetched by an init container or serving process are Pod traffic and can be blocked. Separate both in diagnostics.

## 7. Zero-trust controls beyond NetworkPolicy

Use identity-aware authorization at gateway/app, short-lived credentials, least-privilege cloud IAM, TLS to backends, encrypted secrets, and audit trails. mTLS in a supported mesh provides workload authentication but does not automatically authorize application operations. Protect outbound model/API access against data exfiltration using allowlisted destinations, approved proxies and request-level policies. Do not log prompts, tokens, secrets or PHI in flow evidence.

## 8. Validation commands and negative tests [SAFE-READ]

```bash
kubectl -n tfy-net-lab get netpol -o yaml
kubectl -n tfy-net-lab get pods --show-labels
kubectl -n gateway-system get pods --show-labels
kubectl -n tfy-net-lab get svc,endpointslice
kubectl -n tfy-net-lab get events --sort-by=.lastTimestamp | tail -25
```

For an approved test Pod with appropriate tools, compare expected-allowed DNS, gateway→backend, telemetry, model download and expected-denied cross-namespace/Internet paths. Timeouts are not definitive evidence of a NetworkPolicy drop: investigate DNS, endpoint readiness, routing, security groups, TLS and application failures. Prefer CNI policy verdicts/flow logs to confirm enforcement.

## 9. Production troubleshooting runbooks

### RB-01 — Inference 503 after policy change
Check gateway route, Service/EndpointSlice, readiness, policy selector and CNI drops. Restore previous known-good GitOps policy under change control; validate end-to-end inference.

### RB-02 — DNS failures after default deny
Verify resolver type, UDP/TCP 53, DNS labels and NodeLocal DNSCache; allow only the actual resolver path and retest.

### RB-03 — Model download stuck at startup
Differentiate node image pull from Pod model artifact download; inspect VPC/VNet endpoint, DNS, IAM, proxy and egress logs. Restore least-privilege path.

### RB-04 — Telemetry disappears
Confirm OTLP push vs metrics scrape, collector Pod labels, ingress and egress on both sides, and collector health; restore observation before declaring recovery.

### RB-05 — Cross-namespace lateral access unexpectedly succeeds
Check whether source/destination policies actually select Pods, union-of-allows, CNI enforcement, hostNetwork, mesh and cloud rules; treat verified unauthorized access as a security incident.

### RB-06 — Private endpoint reachable from public network
Inspect cloud LB scheme, firewall/SG/NSG, routes, private DNS and alternate hostnames; contain under security incident procedure, preserve evidence.

### RB-07 — Egress to external API fails intermittently
Compare DNS resolution, changing provider IPs, proxy logs, SNAT exhaustion, NAT/Firewall quotas and TLS; avoid brittle fixed IP allows.

### RB-08 — NetworkPolicy resource exists but no enforcement
Identify CNI/version/configuration and policy status; verify negative connectivity test; escalate CNI implementation gap rather than trusting object existence.

### RB-09 — Streaming inference resets
Correlate gateway idle timeouts, egress gateway/proxy drain, connection tracking, Pod restarts and policy updates; validate SSE/WebSocket end-to-end.

### RB-10 — Agent/control-plane communication broken
Map actual TrueFoundry agent endpoints and authentication, cluster egress path, proxy/DNS/TLS, cloud firewall and policy ownership. Avoid opening broad Internet egress.

### RB-11 — Cross-zone dependency failure
Inspect network routing, endpoint placement, availability zone/subnet restrictions and health; restore approved multi-zone paths.

### RB-12 — Rollback did not restore connectivity
Check GitOps reconciliation, policy union, stale DNS, CNI datapath convergence, cloud firewall rules and route changes; restore full prior known-good state and document all changes.

## 10. SLO, change control and rollback

Measure: successful authorized inference, p95/p99 latency, error budget burn, DNS errors, policy deny verdicts, model initialization success, egress gateway health, telemetry delivery, and unauthorized-access negative-test success. Capture before/after baselines. A rollback must restore the **previous policy set and dependent cloud routes**, not blindly remove all security controls.

Production change checklist: source/destination flow matrix; labels and namespace owners; CNI capability proof; private endpoint/egress design; security sign-off; Git commit; staged tests; canary and abort thresholds; on-call; known-good rollback; post-change synthetic and negative tests; evidence archive.

## 11. Acceptance and next steps

Complete the [Chapter 9.5 hands-on lab](../labs/05-networkpolicy-private-networking-egress-security-check.md), including 26 failure scenarios and 10 acceptance gates. Formal review and live execution remain pending until independently completed.

**Next:** Chapter 9.6 — Inference Streaming, Timeouts and Networking Troubleshooting.
