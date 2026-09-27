# Lab 1.4 — TrueFoundry and Kubernetes Correlation & Failure Isolation

**Classification:** `[SAFE-READ]`
**Associated Tutorial:** Part 1.4 — TrueFoundry and Kubernetes: How They Work Together
**Audience:** SRE / Platform Engineering
**Mode:** Observation only

## 1. Purpose

Safely correlate a TrueFoundry workload to Kubernetes evidence and identify where its lifecycle stops progressing.

```text
TrueFoundry Workload
        ↓
Compute Plane
        ↓
Kubernetes Resource
        ↓
Pod
        ↓
Container
        ↓
Application
        ↓
Endpoint / Dependency
```

Production techniques:
- Lowest Proven Healthy Layer
- First Failed Transition

No cluster state will be intentionally modified.

## 2. Safety Gate

Allowed:

```text
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
```

Forbidden:

```text
kubectl create
kubectl apply
kubectl delete
kubectl edit
kubectl patch
kubectl scale
kubectl rollout restart
helm install
helm upgrade
helm uninstall
```

```text
OBSERVE → CORRELATE → PROVE → CLASSIFY
```

## 3. Verify Kubernetes Context

```bash
kubectl config current-context
```

Record expected and observed context. If they do not match, stop.

## 4. Verify API and Node Health

```bash
kubectl get nodes
kubectl get nodes -o wide
```

Record API reachability and node status. API reachability does not prove workload health.

## 5. Discover TrueFoundry Components

```bash
kubectl get pods -A | grep -Ei 'truefoundry|tfy'
kubectl get deployments -A | grep -Ei 'truefoundry|tfy'
kubectl get pods -A | grep -i 'tfy-agent'
```

Where present:

```bash
kubectl describe pod <tfy-agent-pod> -n <namespace>
kubectl logs <tfy-agent-pod> -n <namespace> --tail=100
```

Do not assume identical component names across installations.

## 6. Discover Workload Types

```bash
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get jobs -A
kubectl get cronjobs -A
```

## 7. Select an Approved Workload

Record:

```text
TrueFoundry workspace:
TrueFoundry workload:
Workload type:
Compute Plane / cluster:
Kubernetes namespace:
Kubernetes workload:
```

Use `Not established from current evidence` rather than guessing.

## 8. Inspect Desired and Observed State

For a Deployment:

```bash
kubectl get deployment <deployment> -n <namespace>
kubectl describe deployment <deployment> -n <namespace>
```

Record desired, current, updated, ready, and available replicas.

Classify:

```text
Desired state matches observed state:
YES / NO / PARTIAL
```

## 9. Correlate Pods and Nodes

```bash
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
```

Record Pod, Ready, Status, Restarts, Age, and Node.

Inspect conditions, container state, readiness, events, volumes, requests, and limits.

## 10. Inspect Container State

Classify:
- Waiting
- Running
- Terminated

Record reason, exit code, restart count, and last termination reason where applicable.

## 11. Inspect Events

```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl get events -A --sort-by='.lastTimestamp'
```

Look for `FailedScheduling`, `FailedMount`, `FailedAttachVolume`, `BackOff`, `Unhealthy`, `Evicted`, and image-pull failures.

Events are evidence, not automatically root cause.

## 12. Inspect Runtime Logs

```bash
kubectl logs <pod> -n <namespace> --tail=100
kubectl get pod <pod> -n <namespace> -o jsonpath="{.spec.containers[*].name}"
kubectl logs <pod> -n <namespace> -c <container> --tail=100
```

Do not include credentials, tokens, PII, or sensitive payloads in evidence.

## 13. Lowest Proven Healthy Layer

Complete:

```text
Platform request accepted       YES / NO / UNKNOWN
Kubernetes object exists        YES / NO
Pod created                     YES / NO
Pod scheduled                   YES / NO
Container started               YES / NO
Application Ready               YES / NO
Endpoint reachable              YES / NO / NOT TESTED
Dependency healthy              YES / NO / NOT TESTED

Lowest proven healthy layer:
```

## 14. First Failed Transition

```text
Platform Request
       ↓
Kubernetes Object
       ↓
Pod Created
       ↓
Pod Scheduled
       ↓
Container Started
       ↓
Application Ready
       ↓
Endpoint Reachable
```

Record the first failed transition and focus investigation there.

## 15. Scheduling Evidence

For a Pending Pod:

```bash
kubectl describe pod <pod> -n <namespace>
```

Look for insufficient CPU/memory/GPU, node selector or affinity mismatch, untolerated taints, and PVC dependencies.

For node capacity:

```bash
kubectl describe node <node>
```

Do not modify requests or limits.

## 16. GPU Evidence

Where applicable:

```bash
kubectl get nodes -o custom-columns="NODE:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"
kubectl describe pod <pod> -n <namespace>
kubectl get pods -A | grep -Ei 'nvidia|gpu'
```

Record:

```text
GPU requested:
GPU allocatable:
Pod scheduled:
GPU node:
GPU failure stage:
```

Classify the stage:
1. Scheduling
2. Device access
3. CUDA / application runtime

## 17. Network Objects

```bash
kubectl get svc -n <namespace>
kubectl describe svc <service> -n <namespace>
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress> -n <namespace>
```

Remember: TrueFoundry Service is not the same as Kubernetes Service.

## 18. Management vs Runtime Path

Management:

```text
TrueFoundry Control Plane
        ↓
Cluster Integration
        ↓
Kubernetes API
```

Runtime:

```text
Client
  ↓
Gateway / Ingress
  ↓
Kubernetes Service
  ↓
Pod
  ↓
Application
```

Record each as `HEALTHY`, `UNHEALTHY`, or `UNKNOWN`. Do not infer one solely from the other.

## 19. Configuration and Secret Safety

```bash
kubectl get configmaps -n <namespace>
kubectl describe configmap <configmap> -n <namespace>
kubectl get secrets -n <namespace>
```

Validate Secret object/reference existence only. Do not dump or decode Secret values.

## 20. ServiceAccount

```bash
kubectl get serviceaccounts -n <namespace>
kubectl get pod <pod> -n <namespace> -o jsonpath="{.spec.serviceAccountName}"
```

Record the ServiceAccount without modifying permissions.

## 21. Storage

Where applicable:

```bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc> -n <namespace>
```

Record PVC status, StorageClass, and capacity.

## 22. Dependency Boundary

Record known dependencies where appropriate:

```text
Database:
Redis:
Object storage:
External API:
Model storage:
Other:
```

Do not retrieve credentials to test them.

## 23. Authoritative Configuration and Drift

Record:

```text
Authoritative configuration source:
TrueFoundry / Git / GitOps / Other / Unknown
```

If unknown, stop before manual mutation.

Example:

```text
Git desired replicas         2
Kubernetes observed replicas 5
```

Treat this as possible drift. Determine the authoritative source, reconciliation owner, and reason before changing anything.

## 24. Failure-Domain Classification

Choose the evidence-supported investigation domain:

```text
TrueFoundry Control Plane
Cluster Integration
GitOps / Reconciliation
Kubernetes API
Scheduling
Node / Container Runtime
Application
Model Server
GPU Runtime
Network
Storage
IAM / Authorization
External Dependency
Unknown
```

Evidence first; classification second.

## 25. Production Incident Snapshot

```text
Timestamp:
Kubernetes context:
Cluster:
TrueFoundry workspace:
TrueFoundry workload:
Workload type:
Kubernetes namespace:
Kubernetes workload:
Desired replicas:
Ready replicas:
Pod:
Pod status:
Node:
Restart count:
Recent events:
Container state:
Relevant runtime evidence:
Kubernetes Service:
Ingress / Gateway:
ServiceAccount:
PVC:
GPU requested:
GPU capacity:
GPU failure stage:
Lowest proven healthy layer:
First failed transition:
Management-path status:
Runtime-path status:
Authoritative configuration source:
Suspected failure domain:
Evidence supporting classification:
```

Never include credentials, tokens, Secret values, or sensitive payloads.

## 26. Incident Communication Exercise

Prefer precise evidence.

Instead of `TrueFoundry deployment is broken`, state that the expected Kubernetes workload exists but its Pod remains Pending and cite the scheduling evidence.

Instead of `GPU is broken`, state whether scheduling, device access, or CUDA/model-runtime initialization is the failing stage.

Instead of `Kubernetes is down`, state exactly which API or runtime evidence is unavailable and what remains unverified.

## 27. Acceptance Checklist

```text
[ ] Correct Kubernetes context verified
[ ] Kubernetes API access established
[ ] Node health inspected
[ ] TrueFoundry cluster components inspected
[ ] tfy-agent identified where present
[ ] Approved workload selected
[ ] Platform workload correlated to Kubernetes resource
[ ] Desired and observed state compared
[ ] Pod correlated to node
[ ] Container state inspected
[ ] Events inspected
[ ] Logs inspected where permitted
[ ] Lowest proven healthy layer identified
[ ] First failed transition identified
[ ] Scheduling evidence checked
[ ] GPU stage classified where applicable
[ ] Kubernetes Service inspected where applicable
[ ] Ingress inspected where applicable
[ ] Management and runtime paths considered separately
[ ] Secret values were not exposed
[ ] ServiceAccount identified where applicable
[ ] Storage checked where applicable
[ ] Authoritative configuration considered
[ ] Failure domain classified using evidence
[ ] No cluster state modified
```

## 28. Completion Criteria

The lab passes when the engineer can demonstrate:

```text
TrueFoundry Workload
        ↓
Kubernetes Representation
        ↓
Desired vs Observed State
        ↓
Pod / Container Evidence
        ↓
Lowest Proven Healthy Layer
        ↓
First Failed Transition
        ↓
Failure Domain
        ↓
Authoritative Remediation Path
```

without changing cluster state or exposing sensitive information.
