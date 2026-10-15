# 5.5 — Services, Endpoints, Routing and Application Connectivity

**Part:** 05 — Application Deployment · **Chapter:** 5 of 7  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, application and networking teams  
**Safety:** All default diagnostics are **[SAFE-READ]**. NetworkPolicy, Service, Gateway, DNS, TLS and ingress changes require **[CHANGE-CONTROL]**. Active tests are limited to authorized destinations and nonproduction workloads.

## 1. Objectives

Trace application traffic from TrueFoundry endpoint to Kubernetes Pod; distinguish Service, EndpointSlice, DNS, ingress/gateway and network-policy failures; verify HTTP/TLS and application health; and recover connectivity without bypassing security boundaries.

## 2. Traffic path

```text
Client (internal/external)
  ↓ DNS lookup / identity / TLS
Ingress Controller or Gateway API implementation (if configured)
  ↓ Route / backend policy / Service port
Kubernetes Service (ClusterIP / LoadBalancer / NodePort as appropriate)
  ↓ EndpointSlice: ready Pod IPs and target ports
CNI networking / NetworkPolicy / service mesh (if installed)
  ↓ Pod IP : container listener
Application handler → dependency calls → response
```

Not every deployment has ingress, a service mesh or an external endpoint. TrueFoundry may provision routing components based on workspace and version. Inspect actual generated resources before asserting a path.

## 3. Service types and use cases

| Service type | Behavior | Operational note |
|---|---|---|
| ClusterIP | Virtual internal service address | Default; suitable for internal-only application access |
| NodePort | Port exposed on eligible node addresses | Exposure/security depends on network controls |
| LoadBalancer | Requests external/cloud load balancer integration | Provisioning, cost and reachability vary |
| ExternalName | DNS CNAME-like alias | Does not proxy traffic or create Pod endpoints |
| Headless (`clusterIP: None`) | DNS returns backing endpoints | Often used for direct endpoint discovery |

For HTTP ingress, choose the platform-supported Ingress or Gateway API implementation. Ingress resources alone do not provide a controller. Gateway API requires installed CRDs and a supporting controller.

## 4. Service and EndpointSlice mechanics

A Service selects Pods using labels; EndpointSlices represent the backend addresses and readiness/serving/terminating conditions. A Pod that is Running but not Ready generally will not be an eligible normal ready endpoint. Named `targetPort` must match a container port name where used.

**Illustrative Kubernetes manifests — not TrueFoundry API payloads:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-platform
  namespace: tfy-routing-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-platform
  template:
    metadata:
      labels:
        app: hello-platform
    spec:
      containers:
        - name: app
          image: registry.example.com/team/hello@sha256:<approved-digest>
          ports:
            - name: http
              containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            periodSeconds: 5
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: hello-platform
  namespace: tfy-routing-lab
spec:
  type: ClusterIP
  selector:
    app: hello-platform
  ports:
    - name: http
      port: 80
      targetPort: http
```

Replace the image placeholder with a real approved immutable digest. This is not a complete production workload: add security context, limits, identity, startup/liveness policy and other environment controls as required.

## 5. TrueFoundry application endpoint lifecycle

1. Identify the TrueFoundry workspace, deployment, revision, cluster and namespace.
2. Confirm whether exposure is internal-only, authenticated external, or private network.
3. Record the platform-generated hostname/route and backing Kubernetes Service.
4. Verify that the route targets the correct Service name/port and namespace.
5. Check TLS certificate/SNI and application protocol expectations.
6. Trace a request through ingress/gateway logs, Service endpoints and application logs.
7. Verify the authoritative source of routing configuration before changing it.

Do not infer that a URL displayed in TrueFoundry is reachable from every network. Private endpoints require the correct VPC/VNet, DNS, firewall and client identity.

## 6. Kubernetes discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> describe service <service>
kubectl -n <namespace> get pods -l <label-selector> -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Inspect Service selector, `port`, `targetPort`, Pod labels, Pod readiness and EndpointSlice address/port. Avoid assuming a legacy Endpoints object is the authoritative representation for large Services.

### Discover ingress/gateway

```bash
kubectl -n <namespace> get ingress
kubectl get gatewayclasses
kubectl -n <namespace> get gateways,httproutes
```

Gateway commands require installed CRDs; `NotFound` for a resource type is not proof of a routing outage. Identify the actual controller and inspect its logs only with approved access.

### Inspect NetworkPolicy

```bash
kubectl -n <namespace> get networkpolicies
kubectl -n <namespace> describe networkpolicy <policy>
```

NetworkPolicy enforcement depends on the CNI implementation. Kubernetes NetworkPolicy is additive; absence of a policy does not prove reachability, because cloud firewalls, service mesh or platform policy may still deny traffic.

## 7. DNS, TLS and connectivity tests

**Test from the same network path as the affected client.** A successful local port-forward proves application/Service access through the Kubernetes API tunnel, not the normal ingress or NetworkPolicy path.

From an authorized client with DNS tools:

```bash
nslookup <application-hostname>
curl -v --connect-timeout 5 --max-time 15 https://<application-hostname>/health
```

Do not place authentication tokens in shell command lines or shared logs. Use approved secret-handling methods and redact `curl -v` output.

For local Service verification:

```bash
kubectl -n <namespace> port-forward service/<service> 18080:<service-port>
```

In another terminal:

```bash
curl -i http://127.0.0.1:18080/ready
curl -i http://127.0.0.1:18080/health
```

The Service port is the exposed `spec.ports[].port`, not necessarily the container `targetPort`.

### TLS diagnostics

Validate certificate subject alternative names, expiry, chain, SNI, hostname and gateway listener configuration. A certificate may be valid yet the request can fail because of route/backend protocol mismatch. Do not disable TLS verification to claim a fix.

## 8. Layer-by-layer failure matrix

| Symptom | Evidence | First likely failed transition |
|---|---|---|
| DNS NXDOMAIN | Client resolver, private/public DNS zone | Hostname → DNS |
| TLS certificate mismatch | SAN, SNI, listener, hostname | TLS handshake |
| Connection timeout | Network path, firewall, LB, gateway | Client → entrypoint |
| HTTP 404 at gateway | Host/path match, HTTPRoute/Ingress | Gateway → route |
| HTTP 502 | Upstream connection/port/protocol logs | Gateway → backend |
| HTTP 503 | Ready endpoints, gateway health, capacity | Route → available backend |
| Service has no ready endpoints | Selector, Pod readiness, EndpointSlice | Service → Pod |
| Connection refused to Pod | Listener bind/port, container logs | Pod network → process |
| Intermittent failure | Endpoint churn, rollout, readiness | Endpoint lifecycle |
| Internal works, external fails | DNS/TLS/ingress/auth/network path | External entrypoint |
| One namespace fails | NetworkPolicy, service mesh, identity | Source → destination policy |
| Service exists, wrong response | Selector, route, revision labels | Routing → wrong backend |

Avoid restarting ingress controllers, changing network policies to allow all traffic, or exposing internal Services publicly as a diagnostic shortcut.

## 9. Structured incident runbook

1. Capture client location/network, request URL, timestamp, response code, trace/request ID and user impact.
2. Identify TrueFoundry deployment, revision and Kubernetes namespace.
3. Check application Pod readiness and error logs.
4. Validate Service selectors and ready EndpointSlice addresses.
5. Verify Service port/targetPort and application listener.
6. Inspect ingress/Gateway route and backend references, if present.
7. Verify DNS, TLS, network restrictions, authentication and policy from the affected client's path.
8. Compare with last known-good route, image and deployment configuration.
9. Identify **Lowest Proven Healthy Layer**, **First Failed Transition**, actionable owner and approved change.
10. Revalidate normal end-to-end traffic, SLOs, error rate and telemetry after recovery.

## 10. Availability, security and rollback

- Use internal-only exposure by default; require explicit approval for public routes.
- Apply least-privilege network policy and identity, and use TLS/authentication as required.
- Avoid routing to Pods before readiness and verify endpoint churn during rollouts.
- Document dependency on cloud load balancers, DNS zones, certificates, gateway controllers and CNI.
- Roll back via TrueFoundry/GitOps authoritative routing configuration; direct Kubernetes patches may be overwritten.
- Include rollback of hostname, route, backend port, certificate reference and policy when applicable.

### Acceptance gates

| Gate | Evidence |
|---|---|
| G1 | Intended TrueFoundry endpoint and exposure policy recorded |
| G2 | Service selector matches intended Ready Pods |
| G3 | EndpointSlice contains eligible backend addresses/ports |
| G4 | Service port and application listener match |
| G5 | DNS and TLS validated from affected client network |
| G6 | Ingress/Gateway and NetworkPolicy evaluated where applicable |
| G7 | Authorized application request succeeds end-to-end |
| G8 | Rollback, SLO validation and cleanup documented |

## 11. Evidence template

```text
Timestamp / cluster / TrueFoundry workspace:
Application / revision / namespace:
Client network and hostname:
DNS answer and TLS certificate metadata:
Ingress/Gateway route and controller:
Service selector, port, targetPort:
EndpointSlice ready addresses:
Pod labels, readiness and listener:
NetworkPolicy / cloud firewall / service mesh:
HTTP status / request ID / latency:
Lowest Proven Healthy Layer:
First Failed Transition:
Owner / approved remediation / rollback:
Post-change end-to-end test:
```

**Lab:** [5.5 — Application Connectivity Validation](../labs/05-services-endpoints-routing-connectivity-check.md)  
**Next:** Chapter 5.6 — Deployment Revisions, Rollouts, Rollbacks and Troubleshooting.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Kubernetes DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
