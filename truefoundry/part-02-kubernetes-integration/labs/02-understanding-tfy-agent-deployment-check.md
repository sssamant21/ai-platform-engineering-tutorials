# Part 2.2 Hands-On Lab — Understanding `tfy-agent` Deployment

> **[SAFE-READ]**
>
> Production-safe validation only. Do not mutate resources or decode Kubernetes Secrets.

## Objectives

Validate the deployed `tfy-agent` topology and collect evidence sufficient to distinguish agent, Kubernetes, networking, authorization, reconciliation, and runtime failure domains.

Core rules:

```text
UNKNOWN TARGET = NO MUTATION
POD ≠ AUTHORITATIVE CONFIGURATION
SECRET EXISTS ≠ SECRET SHOULD BE DECODED
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## 1. Record the observation time

```bash
date -u
```

Record the timestamp used for event, restart, and log correlation.

## 2. Verify the Kubernetes target

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

Record the expected environment, cluster, and context. If the target cannot be proven, stop.

## 3. Inspect node health

```bash
kubectl get nodes -o wide
kubectl describe node <node>
```

Check `Ready`, `MemoryPressure`, `DiskPressure`, `PIDPressure`, and `NetworkUnavailable`.

## 4. Discover the actual `tfy-agent` deployment

Do not assume the namespace or controller type.

```bash
kubectl get pods -A | grep -i tfy
kubectl get deployments -A | grep -i tfy
kubectl get daemonsets -A | grep -i tfy
kubectl get statefulsets -A | grep -i tfy
```

Windows CMD equivalents:

```bat
kubectl get pods -A | findstr /I tfy
kubectl get deployments -A | findstr /I tfy
kubectl get daemonsets -A | findstr /I tfy
kubectl get statefulsets -A | findstr /I tfy
```

Record:

```text
Verified namespace:
Controller type:
Controller name:
Agent Pod(s):
```

## 5. Inspect replicas and placement

```bash
kubectl get pods -n <verified-namespace> -o wide
```

Record all replicas and distinguish redundancy degradation from functional failure.

## 6. Determine Pod ownership

```bash
kubectl get pod <pod> -n <verified-namespace> -o jsonpath='{.metadata.ownerReferences[*].kind}{" "}{.metadata.ownerReferences[*].name}{"
"}'
```

If the immediate owner is a ReplicaSet, inspect it read-only and identify the higher-level controller.

```text
POD ≠ AUTHORITATIVE CONFIGURATION
```

## 7. Inspect the controller

Use the command matching the discovered controller:

```bash
kubectl get deployment <controller> -n <verified-namespace>
kubectl describe deployment <controller> -n <verified-namespace>
```

or the corresponding `daemonset` / `statefulset` commands.

Record desired, available, ready, and updated replicas plus controller conditions.

## 8. Inspect Pod and container state

```bash
kubectl describe pod <pod> -n <verified-namespace>
kubectl get pod <pod> -n <verified-namespace> -o yaml
```

Capture:

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
Node
```

```text
Pod Running ≠ Agent Functional
```

## 9. Inspect current and previous logs

```bash
kubectl logs <pod> -n <verified-namespace>
```

For restarted containers:

```bash
kubectl logs <pod> -n <verified-namespace> --previous
```

For multi-container Pods, identify the container first and use `-c <container>`.

Look for startup, authorization, DNS, connection, TLS, authentication, Kubernetes API, reconciliation, timeout, rate-limit, and resource-pressure evidence.

Sanitize all evidence before sharing it.

## 10. Inspect timestamped events

```bash
kubectl get events -n <verified-namespace> --sort-by=.metadata.creationTimestamp
```

Look for scheduling, image pull, mount, admission, readiness, eviction, OOM, and backoff events. Correlate timestamps with the incident.

## 11. Inspect resources and runtime pressure

```bash
kubectl get pod <pod> -n <verified-namespace> -o yaml
kubectl top pod <pod> -n <verified-namespace>
```

If metrics are unavailable, record that fact rather than treating it as an agent failure.

Check requests, limits, CPU pressure, memory pressure, OOMKilled, and eviction evidence.

## 12. Correlate with the node

```bash
kubectl get pod <pod> -n <verified-namespace> -o wide
kubectl describe node <node>
```

Determine whether the evidence is isolated to `tfy-agent` or affects unrelated workloads on the same node.

## 13. Identify the ServiceAccount

```bash
kubectl get pod <pod> -n <verified-namespace> -o jsonpath='{.spec.serviceAccountName}{"
"}'
kubectl get serviceaccount <service-account> -n <verified-namespace>
```

Do not retrieve Secret values.

## 14. Inspect RBAC bindings

```bash
kubectl get rolebindings -A
kubectl get clusterrolebindings
```

Where needed, search the output for the verified ServiceAccount.

Presence of a binding does not prove the required operation is authorized.

## 15. Validate operation-specific RBAC

If an actual failed operation is known, identify its verb, resource, target namespace, and ServiceAccount.

```bash
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<agent-namespace>:<service-account> -n <target-namespace>
```

Record the result exactly.

```text
TrueFoundry Authorization ≠ Kubernetes RBAC ≠ Cloud IAM
```

## 16. Inspect configuration metadata safely

```bash
kubectl get configmap -n <verified-namespace>
kubectl get secret -n <verified-namespace>
```

Inspect only approved non-secret configuration when needed.

```text
SECRET EXISTS ≠ SECRET SHOULD BE DECODED
```

Do not use this lab to decode or extract Secret values.

## 17. Inspect environment references

Read the Pod specification and identify references such as:

```text
env
envFrom
configMapRef
secretRef
configMapKeyRef
secretKeyRef
```

Record references only. Do not retrieve referenced Secret values.

## 18. Inspect NetworkPolicy

```bash
kubectl get networkpolicy -A
kubectl get networkpolicy -n <verified-namespace>
```

When relevant:

```bash
kubectl describe networkpolicy <policy> -n <verified-namespace>
```

Do not call a policy the cause until its selectors and egress rules are proven applicable.

## 19. Check proxy references

Inspect the workload specification for:

```text
HTTP_PROXY
HTTPS_PROXY
NO_PROXY
http_proxy
https_proxy
no_proxy
```

Do not expose proxy credentials.

## 20. Search logs for connectivity evidence

Linux/macOS:

```bash
kubectl logs <pod> -n <verified-namespace> | grep -Ei "dns|timeout|connect|tls|certificate|x509|auth|forbidden|unauthorized"
```

Windows CMD:

```bat
kubectl logs <pod> -n <verified-namespace> | findstr /I "dns timeout connect tls certificate x509 auth forbidden unauthorized"
```

Matches are investigation leads, not automatic root cause.

## 21. Classify the control path

Complete using only evidence:

```text
Kubernetes placement:       PROVEN / FAILED / UNKNOWN
Container execution:        PROVEN / FAILED / UNKNOWN
Readiness:                  PROVEN / FAILED / UNKNOWN
ServiceAccount identity:    PROVEN / FAILED / UNKNOWN
Kubernetes authorization:   PROVEN / FAILED / UNKNOWN
DNS:                        PROVEN / FAILED / UNKNOWN
TCP connectivity:           PROVEN / FAILED / UNKNOWN
TLS:                        PROVEN / FAILED / UNKNOWN
Authentication:             PROVEN / FAILED / UNKNOWN
Control-plane session:      PROVEN / FAILED / UNKNOWN
Reconciliation:             PROVEN / FAILED / UNKNOWN
Requested operation:        PROVEN / FAILED / UNKNOWN
```

Do not infer a later layer from an earlier healthy layer.

## 22. Determine the Lowest Proven Healthy Layer

Example:

```text
Pod Running       PROVEN
DNS               PROVEN
TCP               PROVEN
TLS               FAILED
Authentication    UNKNOWN
```

Record:

```text
Lowest/last proven healthy transition:
TCP connectivity
```

## 23. Determine the First Failed Transition

For the example above:

```text
First Failed Transition:
TCP connectivity → TLS establishment
```

If no transition has been proven failed, record `UNKNOWN`.

## 24. Record evidence confidence

Use:

```text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Never present an `ASSUMED` item as root cause.

## 25. Determine the Minimum Supported Blast Radius

Choose only the smallest scope supported by evidence:

```text
One container
One Pod
One replica
Agent redundancy
One cluster integration
One deployment operation
Multiple management operations
One runtime workload
Multiple workloads
Multiple clusters
UNKNOWN
```

Record:

```text
Minimum Supported Blast Radius:
```

## 26. Separate control health from runtime health

Record independently:

```text
Control/integration path:
HEALTHY / DEGRADED / FAILED / UNKNOWN

Application runtime path:
HEALTHY / DEGRADED / FAILED / UNKNOWN
```

```text
Management Path Healthy ≠ Runtime Path Healthy
Runtime Path Healthy ≠ Management Path Healthy
```

## 27. Determine desired-state ownership

Inspect labels, annotations, owner references, and approved platform documentation to determine whether the resource is managed by TrueFoundry, Helm, Argo CD/GitOps, OpenTofu/Terraform, an operator, another controller, or is still unknown.

```bash
kubectl get pod <pod> -n <verified-namespace> -o jsonpath='{.metadata.labels}{"
"}{.metadata.annotations}{"
"}'
```

Record:

```text
Observed controller:
Potential desired-state owner:
Authoritative source confirmed: YES / NO
```

If the authoritative source is unknown:

```text
NO MUTATION
```

## 28. Check for configuration drift

Compare the observed image/version, replicas, ServiceAccount, resource settings, configuration references, network policy, and controller metadata with the approved baseline/source of truth.

Record:

```text
Drift observed:
YES / NO / UNKNOWN
```

Do not repair drift in this lab.

## 29. Correlate recent changes

Check approved change history for preceding agent/platform upgrades, Kubernetes upgrades, Helm/GitOps changes, RBAC changes, NetworkPolicy changes, certificate rotation, proxy changes, or cluster maintenance.

```text
CHANGE BEFORE FAILURE ≠ ROOT CAUSE
```

Record correlation as evidence, not proof.

## 30. Identify the Current Actionable Owner

Based on the first proven failure domain, record:

```text
Current Actionable Owner:
Evidence:
Requested Action:
```

```text
SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

## 31. Evidence Handoff Contract

Complete:

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
Last termination:
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

Do not share the evidence package until sensitive information has been removed.

## 32. Acceptance Checklist

```text
[ ] Correct Kubernetes target verified
[ ] Kubernetes API reachable
[ ] Node health inspected
[ ] tfy-agent resources discovered
[ ] Namespace verified
[ ] Controller identified
[ ] All replicas inspected
[ ] Pod/container state inspected
[ ] Restart history inspected
[ ] Current logs inspected
[ ] Previous logs inspected when applicable
[ ] Timestamped events inspected
[ ] Resource configuration inspected
[ ] Node correlation completed
[ ] ServiceAccount identified
[ ] Operation-specific RBAC checked when applicable
[ ] Secret values were NOT exposed
[ ] NetworkPolicy reviewed when applicable
[ ] Proxy references reviewed when applicable
[ ] Control-path layers classified
[ ] Runtime path classified independently
[ ] Lowest Proven Healthy Layer recorded
[ ] First Failed Transition recorded
[ ] Evidence confidence recorded
[ ] Minimum Supported Blast Radius recorded
[ ] Desired-state owner investigated
[ ] Current Actionable Owner identified
[ ] Evidence Handoff Contract completed
```

## 33. Completion Criteria

You should be able to answer:

```text
WHAT is affected?
WHERE is the first proven failure?
WHAT is the Minimum Supported Blast Radius?
WHAT is the Lowest Proven Healthy Layer?
WHAT is the First Failed Transition?
WHO owns the authoritative configuration?
WHO is the Current Actionable Owner?
WHAT action is required next?
```

## 34. Final Safety Gate

Verify:

```text
NO MUTATION
NO SECRET DECODING
NO CREDENTIAL EXPOSURE
NO ASSUMPTION PRESENTED AS FACT
```

## Key Takeaway

A production investigation of `tfy-agent` begins by proving the target and collecting read-only evidence—not by restarting Pods or changing Kubernetes resources.

Separate control-path health from runtime-path health, prove the first failed transition and smallest supported blast radius, and identify the authoritative owner before remediation.
