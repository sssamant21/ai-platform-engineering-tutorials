# Part 2.2 — Understanding `tfy-agent` Deployment

## 1. Purpose

When a Kubernetes cluster is connected to TrueFoundry, the platform requires an integration path between the TrueFoundry Control Plane and the Kubernetes Compute Plane.

`tfy-agent` is a key component of that integration.

For SREs, the critical distinction is:

```text
tfy-agent runs in the Compute Plane.

tfy-agent ≠ Compute Plane
tfy-agent ≠ Kubernetes cluster
tfy-agent ≠ application workload
tfy-agent ≠ TrueFoundry Control Plane
```

A useful conceptual architecture is:

```text
TrueFoundry Control Plane
          │
          │ secure agent-initiated
          │ outbound connection
          ▼
      tfy-agent
          │
          ▼
Cluster-side controllers /
reconciliation components
          │
          ▼
     Kubernetes API
          │
          ▼
       Workloads
```

The exact deployed components can vary with TrueFoundry version, provider, and configuration. Always discover the actual topology.

## 2. Control Path vs Runtime Path

Two paths must be investigated independently.

### Control / integration path

```text
TrueFoundry Control Plane
          │
          ▼
      tfy-agent
          │
          ▼
Cluster-side reconciliation
          │
          ▼
     Kubernetes API
```

This path participates in platform management and deployment operations.

### Application runtime path

```text
Client
  │
  ▼
DNS / Load Balancer / Ingress
  │
  ▼
Service
  │
  ▼
Pod
  │
  ▼
Application / Model
```

Therefore:

```text
Management Path Healthy ≠ Runtime Path Healthy

Runtime Path Healthy ≠ Management Path Healthy
```

A `tfy-agent` problem does not by itself prove an application outage.

## 3. Agent-Initiated Connectivity

The cluster-side agent initiates its connection toward the TrueFoundry Control Plane.

Operationally, investigate the outbound path:

```text
tfy-agent
    │
    ▼
DNS
    │
    ▼
Routing
    │
    ▼
Network policy
    │
    ▼
Firewall / proxy
    │
    ▼
TCP
    │
    ▼
TLS
    │
    ▼
Authentication
    │
    ▼
Control-plane session
```

Do not assume that opening inbound access into the Kubernetes cluster is the appropriate remediation for an agent-control-connection problem.

Avoid hard-coding one transport protocol as universally applicable across every TrueFoundry deployment or version.

## 4. Deployment Identity

Before troubleshooting, establish exactly what is being investigated.

Record:

```text
Environment
Cluster
Kubernetes context
Namespace
Controller
Pod
Container
Image/version
ServiceAccount
Deployment method
Installation/reconciliation owner
Observation timestamp
```

The mandatory production rule remains:

```text
UNKNOWN TARGET = NO MUTATION
```

Never modify a resource until the target has been proven.

## 5. Verify the Kubernetes Target

Begin with read-only checks:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

Confirm:

```text
Expected environment
        =
Expected cluster
        =
Current Kubernetes target
```

A familiar resource name is not sufficient proof.

## 6. Discover the Actual Deployment

Do not assume the agent namespace or controller type.

Start with discovery:

```bash
kubectl get pods -A | grep -i tfy
kubectl get deployments -A | grep -i tfy
kubectl get daemonsets -A | grep -i tfy
kubectl get statefulsets -A | grep -i tfy
```

Once identified:

```bash
kubectl get pods -n <verified-namespace> -o wide
```

The deployed environment is the source of truth.

## 7. Controller Before Pod

A Pod is normally an instance controlled by another resource.

Determine the ownership chain:

```text
Pod
 ↑
ReplicaSet?
 ↑
Deployment?
 ↑
Helm release?
 ↑
GitOps/controller?
 ↑
Platform-generated configuration?
```

Operational rule:

```text
POD ≠ AUTHORITATIVE CONFIGURATION
```

Deleting or editing a Pod may address a symptom without correcting the desired state.

## 8. Establish a Healthy Baseline

Before incidents, capture a known-good baseline:

```text
Namespace
Controller type
Expected replicas
ServiceAccount
Image/version
CPU/memory requests
CPU/memory limits
Normal readiness
Normal restart behavior
Expected connectivity
Relevant network policy
Deployment method
Reconciliation owner
```

During an incident compare:

```text
Known Good
    vs
Observed Now
```

This is more reliable than judging configuration in isolation.

## 9. Layered Agent Health Model

Do not define agent health as `Pod = Running`.

Use:

```text
Kubernetes placement
       ↓
Container execution
       ↓
Process/readiness
       ↓
Identity/authorization
       ↓
Outbound connectivity
       ↓
TLS/authentication
       ↓
Control-plane session
       ↓
Reconciliation path
       ↓
Requested operation succeeds
```

Therefore:

```text
Pod Running ≠ Agent Functional

Agent Functional ≠ Reconciliation Functional

Reconciliation Functional ≠ Application Healthy
```

## 10. Pod and Container State

Inspect:

```bash
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
```

Important evidence includes:

```text
PodScheduled
Initialized
ContainersReady
Ready
Container state
Restart count
Last termination reason
Exit code
Probe failures
Events
Node
```

`Running + repeated restarts` should not automatically be classified as healthy.

## 11. Replica Health

If multiple replicas exist, inspect every replica.

Example:

```text
tfy-agent-a    Running
tfy-agent-b    CrashLoopBackOff
```

This does not automatically prove either complete integration outage or integration health.

Determine whether the condition represents:

```text
Redundancy degradation

or

Functional degradation
```

and prove the resulting blast radius.

## 12. Logs

After proving the correct target:

```bash
kubectl logs <pod> -n <namespace>
```

If the container restarted:

```bash
kubectl logs <pod> -n <namespace> --previous
```

Investigate evidence related to startup, authentication, authorization, DNS, connectivity, TLS, timeouts, Kubernetes API access, reconciliation, resource exhaustion, and rate limiting.

```text
Error visible in tfy-agent log ≠ tfy-agent root cause
```

Logs may expose failures originating from another dependency.

## 13. Kubernetes Events

Inspect timestamped events:

```bash
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

Events may expose:

```text
FailedScheduling
FailedMount
FailedCreate
ImagePullBackOff
BackOff
Unhealthy
Evicted
OOM
Admission failures
```

Correlate event timestamps with incident timestamps.

## 14. Resource Pressure

Where metrics are available:

```bash
kubectl top pod <pod> -n <namespace>
```

Inspect configured resources:

```bash
kubectl get pod <pod> -n <namespace> -o yaml
```

Investigate CPU saturation, CPU throttling, memory pressure, OOMKilled, eviction, and insufficient resources.

A connectivity symptom may be secondary to resource starvation.

## 15. Node Correlation

Determine the node:

```bash
kubectl get pod <pod> -n <namespace> -o wide
```

Then inspect:

```bash
kubectl describe node <node>
```

Check:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
Recent node events
```

Ask whether only `tfy-agent` Pods are affected or whether multiple workloads on the node are affected.

A node failure should not be mislabeled as an agent failure.

## 16. ServiceAccount

Determine the actual identity:

```bash
kubectl get pod <pod> -n <namespace> -o jsonpath="{.spec.serviceAccountName}"
```

Then inspect metadata:

```bash
kubectl get serviceaccount <service-account> -n <namespace>
```

Conceptually:

```text
tfy-agent Pod
      │
      ▼
Kubernetes ServiceAccount
      │
      ├── Kubernetes RBAC
      │
      └── Cloud identity/IAM
```

Do not collapse these layers into a generic permission issue.

## 17. Operation-Specific RBAC

Do not ask only whether `tfy-agent` has RBAC.

Determine:

```text
Which operation failed?
Which verb?
Which resource?
Which namespace?
Which ServiceAccount?
```

Then test the specific permission:

```bash
kubectl auth can-i <verb> <resource> \
  --as=system:serviceaccount:<agent-namespace>:<service-account> \
  -n <target-namespace>
```

Preserve this distinction:

```text
TrueFoundry Authorization
          ≠
Kubernetes RBAC
          ≠
Cloud IAM
```

## 18. Configuration and Secret Safety

Possible configuration sources include:

```text
Pod specification
ConfigMaps
Environment configuration
Secrets
Mounted configuration
ServiceAccount
Installation values
Provider-specific identity
```

Safe metadata discovery can include:

```bash
kubectl get configmap -n <namespace>
kubectl get secret -n <namespace>
```

But:

```text
SECRET EXISTS ≠ SECRET SHOULD BE DECODED
```

Do not place tokens, credentials, private certificates, secret values, or sensitive proxy credentials into troubleshooting evidence.

## 19. Network Troubleshooting

Investigate network health progressively:

```text
DNS
 ↓
Route
 ↓
Network policy
 ↓
Firewall
 ↓
Proxy
 ↓
TCP
 ↓
TLS
 ↓
Authentication
 ↓
Application protocol/session
```

Do not jump from `Agent cannot connect` to `Firewall issue` without evidence.

## 20. Proxy Awareness

Enterprise environments may use:

```text
HTTP_PROXY
HTTPS_PROXY
NO_PROXY
Egress proxies
TLS inspection
Corporate certificate authorities
```

Proxy configuration can affect the control connection. Inspect configuration safely and sanitize URLs containing credentials.

## 21. TLS Is a Separate Failure Domain

Possible TLS failures include:

```text
Expired certificate
Unknown authority
Trust-chain failure
Hostname mismatch
Handshake timeout
```

Remember:

```text
DNS Healthy ≠ TCP Healthy

TCP Healthy ≠ TLS Healthy

TLS Healthy ≠ Authentication Healthy
```

Each transition requires separate evidence.

## 22. Lowest Proven Healthy Layer

Suppose investigation establishes:

```text
Pod Running                 PROVEN
DNS                         PROVEN
TCP                         PROVEN
TLS                         FAILED
Authentication              NOT TESTED
Control session             NOT TESTED
Reconciliation              NOT TESTED
```

Do not claim `Agent connectivity healthy`.

The last proven healthy transition is TCP connectivity.

## 23. First Failed Transition

For the previous example:

```text
DNS
 ↓
TCP
 ↓
TLS
 X
```

Record:

```text
First Failed Transition:
TCP connectivity → TLS establishment
```

This is more actionable than `TrueFoundry is down`.

## 24. Evidence Confidence

Use:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

```text
Pod Running                 PROVEN
DNS resolution              PROVEN
TCP connectivity            PROVEN
TLS configuration           UNKNOWN
Credentials valid           UNKNOWN
Control Plane outage        ASSUMED
```

The investigation should convert `UNKNOWN / ASSUMED` into `SUPPORTED / PROVEN` before root cause is assigned.

## 25. Reconciliation and Desired-State Ownership

TrueFoundry environments can involve multiple controllers or configuration systems.

Examples may include:

```text
TrueFoundry
Helm
Argo CD
GitOps
OpenTofu/Terraform
Kubernetes operators
Other controllers
```

Before remediation determine:

```text
Observed resource
       ↓
Controller
       ↓
Desired-state owner
       ↓
Authoritative source
```

Production rule:

```text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

Preferred operational sequence:

```text
OBSERVE
   ↓
IDENTIFY OWNER
   ↓
IDENTIFY AUTHORITATIVE SOURCE
   ↓
APPROVE CHANGE
   ↓
CHANGE SOURCE OF TRUTH
   ↓
RECONCILE
   ↓
VALIDATE
```

## 26. Configuration Drift

Compare:

```text
Expected Configuration
          vs
Observed Configuration
```

Potential drift areas:

```text
Image/version
ServiceAccount
RBAC
Replica count
Resources
ConfigMaps
Network policy
Installation values
Controller configuration
```

Do not automatically repair drift using `kubectl edit` or `kubectl patch`. First identify the authoritative source.

## 27. Change and Upgrade Correlation

During investigation determine whether the incident followed:

```text
tfy-agent upgrade
TrueFoundry platform change
Kubernetes upgrade
Helm change
GitOps synchronization
RBAC change
Network policy change
Certificate rotation
Proxy change
Cluster maintenance
```

But:

```text
CHANGE BEFORE FAILURE ≠ ROOT CAUSE
```

Timeline correlation narrows investigation; it does not prove causation.

## 28. Minimum Supported Blast Radius

Determine only what the evidence supports.

Potential scopes include:

```text
One container
One Pod
One replica
Agent redundancy
One cluster integration
One deployment operation
Multiple management operations
Runtime workload
Multiple clusters
```

Do not infer:

```text
One tfy-agent Pod failing
            =
Entire TrueFoundry platform unavailable
```

Prove the scope.

## 29. Current Actionable Owner

Use:

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

Example:

```text
Symptom:
tfy-agent reports Forbidden

Evidence:
ServiceAccount             PROVEN
Requested Kubernetes verb  PROVEN
Required RBAC missing      PROVEN

Failure domain:
Kubernetes authorization

Current Actionable Owner:
Team controlling authoritative RBAC configuration
```

The component reporting an error does not automatically own its remediation.

## 30. Detection vs Remediation

Production troubleshooting should follow:

```text
Detection
    ↓
Evidence collection
    ↓
Failure-domain isolation
    ↓
Ownership determination
    ↓
Approved remediation
    ↓
Post-change validation
```

Avoid reflexive production actions such as:

```text
kubectl delete pod
kubectl edit
kubectl patch
kubectl rollout restart
helm upgrade
```

until ownership, impact, and change authorization are established.

## 31. Failure Scenario — CrashLoopBackOff

```text
Symptom:
tfy-agent CrashLoopBackOff
```

Investigate:

```text
Previous container logs
Termination reason
Exit code
Configuration
Resource pressure
Events
Node state
Recent changes
```

Do not immediately conclude `TrueFoundry Control Plane outage`.

## 32. Failure Scenario — Forbidden

```text
tfy-agent
   │
   ▼
Kubernetes API
   │
   X Forbidden
```

Investigate:

```text
ServiceAccount
Verb
Resource
Namespace
Role / ClusterRole
RoleBinding / ClusterRoleBinding
```

Based on proven evidence, the likely failure domain may be the Kubernetes authorization path.

## 33. Failure Scenario — TLS

Evidence:

```text
Pod Running       PROVEN
DNS               PROVEN
TCP               PROVEN
TLS               FAILED
```

Then:

```text
Lowest Proven Healthy Layer:
TCP connectivity

First Failed Transition:
TCP → TLS
```

Do not yet make claims about authentication or reconciliation.

## 34. Failure Scenario — One Replica Failing

```text
Replica A    Running
Replica B    CrashLoopBackOff
```

Determine whether this represents redundancy degradation or functional integration failure before declaring an outage.

## 35. Failure Scenario — Agent Healthy, Deployment Fails

Suppose:

```text
Agent Pod stable              PROVEN
Control connection healthy    PROVEN
Deployment fails              PROVEN
```

Continue down the stack:

```text
Cluster-side controller
Kubernetes authorization
Admission policy
Scheduling
Image pull
Storage
Networking
Application configuration
```

Do not stop because `tfy-agent` appears healthy.

## 36. Post-Remediation Validation

After an approved remediation validate progressively:

```text
Pod stable
    ↓
Readiness stable
    ↓
Agent connectivity stable
    ↓
Control session functional
    ↓
Reconciliation functional
    ↓
Requested operation succeeds
    ↓
Existing workloads unaffected
    ↓
Runtime traffic healthy
```

Remember:

```text
Pod Recovered ≠ Service Restored

Service Restored ≠ Root Cause Remediated
```

## 37. Evidence Handoff Contract

When another team must act, provide:

```text
Environment:
Cluster:
Context:
Namespace:

Agent controller:
Agent Pod(s):
Image/version:
ServiceAccount:

Observation timestamp:

Symptom:

Replica state:
Restart history:
Node state:

Control-path status:
Runtime-path status:

Relevant events:
Sanitized log evidence:
Authorization evidence:

Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Evidence Confidence:

Authoritative Configuration Owner:

Current Actionable Owner:

Requested Action:

Sensitive information removed:
YES
```

Avoid vague escalation such as `tfy-agent broken. Please check.`

## 38. Production Investigation Flow

```text
VERIFY TARGET
      ↓
DISCOVER DEPLOYMENT
      ↓
ESTABLISH BASELINE
      ↓
IDENTIFY CONTROLLER
      ↓
CHECK REPLICAS
      ↓
CHECK POD / CONTAINER
      ↓
CHECK EVENTS
      ↓
CHECK NODE / RESOURCES
      ↓
CHECK IDENTITY
      ↓
CHECK OPERATION-SPECIFIC RBAC
      ↓
CHECK OUTBOUND NETWORK
      ↓
CHECK PROXY / TLS
      ↓
CHECK AUTHENTICATION
      ↓
CHECK CONTROL SESSION
      ↓
CHECK RECONCILIATION
      ↓
CHECK REQUESTED OPERATION
      ↓
DETERMINE LOWEST PROVEN HEALTHY LAYER
      ↓
DETERMINE FIRST FAILED TRANSITION
      ↓
PROVE BLAST RADIUS
      ↓
IDENTIFY AUTHORITATIVE OWNER
      ↓
APPROVED REMEDIATION
      ↓
END-TO-END VALIDATION
```

## 39. Core Production Rules

1. `tfy-agent` runs in the Compute Plane but is not the Compute Plane.
2. Discover actual topology instead of assuming it.
3. `UNKNOWN TARGET = NO MUTATION`.
4. The agent initiates its control connection outbound.
5. Do not hard-code one transport as universally applicable.
6. Separate control/integration health from runtime health.
7. `Pod Running ≠ Agent Functional`.
8. `Agent Functional ≠ Reconciliation Functional`.
9. `Reconciliation Functional ≠ Application Healthy`.
10. `POD ≠ AUTHORITATIVE CONFIGURATION`.
11. Inspect every replica.
12. Correlate agent health with node health.
13. Preserve previous-container logs after restarts.
14. Treat events as timestamped evidence.
15. Validate RBAC by verb + resource + namespace + identity.
16. `TrueFoundry Authorization ≠ Kubernetes RBAC ≠ Cloud IAM`.
17. `SECRET EXISTS ≠ SECRET SHOULD BE DECODED`.
18. Separate DNS, TCP, TLS, and authentication.
19. Account for enterprise proxies.
20. Determine the Lowest Proven Healthy Layer.
21. Determine the First Failed Transition.
22. Record evidence confidence.
23. Determine the Minimum Supported Blast Radius.
24. `SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER`.
25. Determine desired-state ownership before remediation.
26. Change the authoritative source rather than patching symptoms.
27. Treat timeline correlation as evidence, not proof.
28. Separate detection from remediation.
29. Validate management and runtime paths after remediation.
30. `Service Restored ≠ Root Cause Remediated`.
31. Escalate using an Evidence Handoff Contract with an explicit Requested Action.

## Key Takeaway

`tfy-agent` is a critical cluster-side integration component, but its health must be evaluated as part of a larger control and reconciliation path.

Production troubleshooting should prove each transition, determine the smallest evidence-supported blast radius, identify the authoritative configuration owner, and separate management-path failures from application runtime failures before remediation.
