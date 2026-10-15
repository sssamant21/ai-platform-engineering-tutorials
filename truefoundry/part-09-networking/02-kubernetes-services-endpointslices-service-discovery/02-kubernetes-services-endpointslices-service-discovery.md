# 9.2 — Kubernetes Services, EndpointSlices and Service Discovery

**Part 09 — Networking | Chapter 2 of 7**  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, networking engineers and model-serving operators  
**Execution:** **[SAFE-READ]** inspection; **[LAB-MUTATE]** approved isolated staging; **[CHANGE-CONTROL]** production.

## 1. Objectives and prerequisites

Explain Service types, DNS naming, selectors, EndpointSlices, readiness and termination; trace a TrueFoundry/vLLM request to a specific Pod; distinguish Service discovery, proxying and CNI dataplane behavior; diagnose routing and cold-start failures; validate stream-safe rollouts and recovery.

Prerequisites: Chapter 9.1, read-only Kubernetes access, authorized namespace and endpoint, deployed model service, documented port mapping, approved metrics and rollback. Actual TrueFoundry gateway and Kubernetes network implementations must be discovered, not presumed.

## 2. Service architecture

```text
Client / Gateway / Ingress
       │
       ▼
Service DNS name → Service virtual IP (when applicable)
       │
       ▼
Service routing implementation (kube-proxy iptables/IPVS,
eBPF replacement, or cloud dataplane — deployment dependent)
       │
       ▼
EndpointSlice → Pod IP:targetPort → inference listener
       │
       ▼
Model Ready → response / streaming tokens
```

**A Service does not run the application or make a Pod Ready.** It provides a stable discovery/routing abstraction. DNS may return a virtual IP, individual endpoint IPs for a headless Service, or a provider-specific result. An EndpointSlice is not itself a load balancer.

## 3. Service types and when to use them

| Type | Exposure | Common use | Caveat |
|---|---|---|---|
| ClusterIP | Internal cluster address | Private inference backend | Requires permitted in-cluster route |
| NodePort | Port on nodes | Provider LB integration, controlled labs | Direct node exposure needs strict network review |
| LoadBalancer | Cloud/provider load balancer | Internal or external endpoint | Annotation/class and exposure depend on provider |
| ExternalName | DNS alias | External dependency discovery | No native proxying; TLS Host/SNI can differ |
| Headless (`clusterIP: None`) | DNS endpoints | Stateful/direct endpoint discovery | Clients must handle endpoint changes |

Avoid publishing model endpoints with NodePort or external LoadBalancer merely for debugging. Prefer the deployment's approved TrueFoundry gateway/exposure workflow.

## 4. Port and protocol contract

```yaml
# Illustrative only; apply exclusively to an isolated approved staging namespace.
apiVersion: v1
kind: Service
metadata:
  name: llm-backend
  namespace: tfy-network-lab
spec:
  type: ClusterIP
  selector:
    app: llm-backend
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8000
```

- `port`: Service listener port; `targetPort`: container listener port or named Pod port.
- `selector`: must match intended Pods; check rollout labels and namespace.
- `appProtocol`: optional hint; support depends on controller.
- HTTP route host/path, TLS termination and auth are **not** supplied by this Service.
- If Pods use a named port, verify name resolution for all selected Pods.

## 5. EndpointSlices and Pod readiness

EndpointSlices describe addresses, ports and endpoint conditions for a Service. A selector-based Service normally gets managed EndpointSlices from Kubernetes controllers; manually managed endpoints require special care and can be dangerous if mixed with automatic ownership.

Key conditions:

- `ready`: broadly indicates eligible routing, with exceptions such as `publishNotReadyAddresses`.
- `serving`: can distinguish an endpoint still serving while terminating.
- `terminating`: endpoint is being removed/drained.

Interpret conditions with the Kubernetes release and dataplane/controller behavior; not every proxy uses every field identically.

For GPU-backed models, Pod `Running` can precede image pull, model load, CUDA initialization, warmup and actual readiness. A model-aware readiness probe should not return success until inference can serve valid requests. Avoid changing readiness thresholds just to hide cold starts.

```bash
kubectl -n <namespace> get svc <service> -o yaml
kubectl -n <namespace> get endpointslices -l kubernetes.io/service-name=<service> -o wide
kubectl -n <namespace> get endpointslices -l kubernetes.io/service-name=<service> -o yaml
kubectl -n <namespace> get pods -l app=<expected-label> -o wide
kubectl -n <namespace> describe pod <pod>
```

**Security:** Service/EndpointSlice YAML may disclose internal IPs and topology; keep evidence in authorized channels.

## 6. Service DNS discovery

Typical Kubernetes DNS names:

```text
llm-backend
llm-backend.tfy-network-lab
llm-backend.tfy-network-lab.svc
llm-backend.tfy-network-lab.svc.cluster.local
```

The actual cluster DNS suffix can differ. Short names depend on Pod namespace and DNS search paths. Test resolution from an **authorized existing** client Pod, not an automatically privileged debug container.

```bash
# Inside an approved existing diagnostic Pod, when tools are present:
nslookup llm-backend.tfy-network-lab.svc
getent hosts llm-backend.tfy-network-lab.svc
```

DNS resolution is only the first gate. A record can resolve while endpoints are absent or the application is unhealthy.

## 7. kube-proxy, eBPF and dataplane variants

Many clusters use kube-proxy with iptables/IPVS/nftables depending on version and configuration; others replace Service handling with an eBPF dataplane. Verify the installed implementation before inspecting rules. Do not assume that kube-proxy must be present.

Failure signatures differ: missing Service rules, endpoint sync lag, node-specific routing, CNI route failure, NetworkPolicy drop, or cloud security-group/NSG denial. A Service IP timeout is not sufficient to identify the responsible layer.

## 8. Readiness, termination and long-lived streams

LLM streaming creates a drain problem:

```text
rollout/scale-down → Pod terminating
 → endpoint readiness changes → no new sessions
 → existing streams finish within drain/termination window
 → Pod exits
```

Set termination grace, application shutdown behavior, gateway drain timeouts, PodDisruptionBudget and autoscaler policy coherently. PDBs constrain some voluntary evictions, but do not prevent every form of termination or rollout; confirm actual workload-controller behavior.

Avoid assuming a successful TCP connection implies a completed inference stream. Record first token, subsequent chunks and final completion.

## 9. Service debugging runbook [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get svc,deploy,pods -o wide
kubectl -n <namespace> describe svc <service>
kubectl -n <namespace> get endpointslices -l kubernetes.io/service-name=<service> -o yaml
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get networkpolicy
kubectl -n <namespace> get pdb
kubectl -n <namespace> describe pod <pod>
```

Then establish:

1. Correct cluster/namespace and Service name.
2. DNS record and namespace search behavior.
3. Correct Service type, selector, port and targetPort.
4. Expected EndpointSlice addresses and `ready/serving/terminating` conditions.
5. Pod Ready status, listener, model initialization and revision.
6. NetworkPolicy, CNI, node/LB route and gateway backend reference.
7. End-to-end request with valid TLS/auth and stream completion.

Use `kubectl port-forward` only when approved: it bypasses the normal gateway path and cannot prove public ingress behavior.

## 10. Troubleshooting decision tree

```text
Client cannot reach model
 ├─ DNS fails → service name / namespace / CoreDNS / search path
 ├─ DNS resolves, connection fails
 │    ├─ Service selector wrong → no EndpointSlices
 │    ├─ Endpoints exist but not Ready → model startup/probes
 │    ├─ Port mismatch → Service targetPort / listener
 │    └─ Correct endpoints → dataplane / policy / gateway
 ├─ HTTP 404 → gateway Host/path or app route
 ├─ HTTP 503 → no healthy backend or gateway upstream failure
 ├─ HTTP 504 → timeout, queue, cold start or network stall
 └─ Stream drops → drain, idle timeout or backend crash
```

Record **Lowest Proven Healthy Layer (LPHL)** and **First Failed Transition (FFT)** with timestamps, not assumptions.

## 11. Failure scenarios and first checks

| ID | Symptom | FFT hypothesis | First evidence |
|---|---|---|---|
| S1 | Service DNS NXDOMAIN | Name → DNS | Resolver query |
| S2 | Short-name wrong namespace | Search path → DNS | Pod resolv.conf |
| S3 | Service selector mismatch | Service → Pods | Selector/labels |
| S4 | Empty EndpointSlice set | Controller → endpoints | Service selector |
| S5 | Endpoints not Ready | Pod → endpoint | Readiness reason |
| S6 | Named targetPort unresolved | Service → listener | Pod port names |
| S7 | Wrong numeric targetPort | Service → listener | Container ports |
| S8 | Pod listens only on loopback | Pod IP → listener | Bind address |
| S9 | NetworkPolicy blocks client | Client → Pod | Policy/flow |
| S10 | CNI Pod route failure | Dataplane → Pod | Node/CNI events |
| S11 | Node-specific Service failure | Node → dataplane | Node comparison |
| S12 | kube-proxy/eBPF rules stale | Endpoint → forwarding | Dataplane status |
| S13 | Headless client caches old IP | DNS → client | TTL/cache |
| S14 | ExternalName TLS mismatch | Alias → TLS | SNI/SAN |
| S15 | Internal LB misconfigured | Cloud LB → Service | Scheme/targets |
| S16 | Accidental public exposure | LB → internet | Cloud inventory |
| S17 | Gateway backend wrong port | Gateway → Service | BackendRef |
| S18 | Model cold start | Pod → Ready | Startup timeline |
| S19 | EndpointSlice terminating state mishandled | Drain → routing | Conditions |
| S20 | Rolling update loses all Ready Pods | Rollout → availability | Replica timeline |
| S21 | Scale-down truncates stream | Pod drain → client | Completion |
| S22 | PDB prevents planned drain | Eviction → disruption | PDB conditions |
| S23 | Dual-stack family mismatch | Service IP → route | IP families |
| S24 | Wrong model revision behind Service | Selector → Pod | Image/model digest |

Classify each as actual fault, expected behavior or not applicable. A blocked drain by PDB can be protective rather than a failure; verify the intended disruption budget.

## 12. Production recovery

1. Declare scope: Service, namespace, model, error codes, TTFT/ITL and stream completion.
2. Capture immutable revision, Service/EndpointSlice state and recent rollout events.
3. Prove DNS → Service → endpoint readiness → dataplane → model.
4. Contain via approved gateway routing or known-good deployment; avoid creating public NodePorts.
5. Repair the first proven failed transition using authoritative TrueFoundry/GitOps controls.
6. Validate Ready endpoints, correct model, TLS/auth, successful complete streams and SLO recovery.
7. Document LPHL/FFT, affected traffic, time to recovery and preventive alerts.

## 13. Acceptance gates

| Gate | Evidence |
|---|---|
| A1 | Service type and exposure documented |
| A2 | Selector, port and targetPort verified |
| A3 | EndpointSlice Ready/serving/terminating understood |
| A4 | Service DNS and namespace resolution verified |
| A5 | Dataplane and NetworkPolicy ownership identified |
| A6 | Stream drain and readiness contract reviewed |
| A7 | S1–S24 classified with LPHL/FFT |
| A8 | Safe staging validation and rollback documented or Not run |

## 14. Evidence template

```text
Date/time / owner / approval:
TrueFoundry workspace / cluster / namespace:
Service type/name/selector/port/targetPort:
Service DNS and cluster suffix:
EndpointSlice ready/serving/terminating:
Pod listener/readiness/model digest:
kube-proxy/eBPF/CNI and NetworkPolicy:
Gateway/LB backend and TLS/auth:
Streaming drain/termination grace/PDB:
LPHL/FFT and S1–S24:
A1–A8 Pass/Fail/Not run, rollback and SLO:
```

**Hands-on lab:** [9.2 — Services, EndpointSlices and Service Discovery Check](../labs/02-kubernetes-services-endpointslices-service-discovery-check.md)

**Next:** Chapter 9.3 — Ingress, Gateway API and Cloud Load Balancers.

## References

- [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
