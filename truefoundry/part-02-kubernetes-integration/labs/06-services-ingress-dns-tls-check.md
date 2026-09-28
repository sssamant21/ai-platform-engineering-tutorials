# Part 2.6 Lab --- Services, Ingress, DNS & TLS Check

> **\[SAFE-READ\] Production Validation Lab**

## Purpose

This lab validates the end-to-end request path for a TrueFoundry
workload on Kubernetes without changing production state.

The investigation model is:

``` text
External Client
      ↓
External DNS
      ↓
Load Balancer / Entry Point
      ↓
TLS Termination
      ↓
Istio Gateway / Ingress / Gateway
      ↓
Host + Path Routing
      ↓
Kubernetes Service
      ↓
EndpointSlice
      ↓
Ready Endpoint
      ↓
Pod IP : targetPort
      ↓
Application Listener
      ↓
Application Response
      ↓
Business Transaction
```

For internal traffic:

``` text
In-Cluster Client
      ↓
Kubernetes DNS
      ↓
Service
      ↓
EndpointSlice
      ↓
Pod
      ↓
Application
```

The goal is to determine:

``` text
WHAT observation point is failing?

WHAT request path was expected?

WHERE is the Lowest Proven Healthy Layer?

WHERE is the First Failed Transition?

WHAT is the Minimum Supported Blast Radius?

WHO owns authoritative configuration?

WHO owns the next actionable step?
```

The operating principle is:

``` text
DO NOT TROUBLESHOOT "THE NETWORK."

PROVE THE REQUEST PATH
ONE TRANSITION AT A TIME.
```

------------------------------------------------------------------------

# 1. Safety Contract

This is a read-only production validation lab.

## Allowed Operations

Use safe inspection commands such as:

``` bash
kubectl config current-context
kubectl cluster-info
kubectl get
kubectl describe
kubectl logs
```

Safe external observations may include organization-approved tools for:

``` text
DNS lookup
TCP connectivity check
TLS certificate inspection
HTTP HEAD/GET request
```

Use only approved endpoints and approved client locations.

## Prohibited Operations

Do **not** run:

``` text
kubectl create
kubectl apply
kubectl delete
kubectl edit
kubectl patch
kubectl replace
kubectl scale
kubectl rollout restart
helm install
helm upgrade
helm uninstall
```

Do not:

-   modify Services;
-   modify Service selectors;
-   change `port` or `targetPort`;
-   modify EndpointSlices;
-   modify Ingress resources;
-   modify Gateway resources;
-   modify Istio routing;
-   change DNS records;
-   replace certificates;
-   decode TLS Secrets;
-   extract private keys;
-   restart Pods;
-   delete Pods;
-   restart ingress controllers;
-   restart gateway Pods;
-   recreate load balancers;
-   create diagnostic Pods in production;
-   modify NetworkPolicies;
-   change firewall/security-group rules;
-   change shared ingress infrastructure;
-   perform mutation "just to test."

Production guardrails:

``` text
UNKNOWN TARGET = NO MUTATION

UNKNOWN ROUTING IMPLEMENTATION = NO ROUTE CHANGE

UNKNOWN TLS TERMINATION POINT = NO CERTIFICATE CHANGE

UNKNOWN AUTHORITATIVE OWNER = NO DIRECT CONFIGURATION CHANGE

Observed Drift ≠ Permission to Patch
```

------------------------------------------------------------------------

# 2. Lab Variables

Identify the target before starting.

``` text
ENVIRONMENT=<environment>
WORKSPACE=<truefoundry-workspace>

CONTEXT=<kubernetes-context>
NAMESPACE=<namespace>

HOSTNAME=<application-hostname>
PORT=<port>
PROTOCOL=<http|https>
PATH=<request-path>

SERVICE=<service>
POD=<affected-pod>
CONTAINER=<container>
```

Optional:

``` text
INGRESS=<ingress>
GATEWAY=<gateway>
ROUTE=<route>
ENDPOINTSLICE=<endpointslice>
```

Do not continue if the environment, cluster, namespace, or endpoint is
uncertain.

------------------------------------------------------------------------

# 3. Capture the Original Symptom

Before checking Kubernetes objects, record the original client-visible
failure.

``` text
Observation point:

Source:

Observation timestamp:

Requested scheme:

Requested hostname:

Requested port:

Requested path:

Expected response:

Observed response:

HTTP status:
If applicable

Original error:
```

Rule:

``` text
UNKNOWN OBSERVATION POINT
=
INCOMPLETE NETWORK EVIDENCE
```

------------------------------------------------------------------------

# 4. Verify Kubernetes Context

Run:

``` bash
kubectl config current-context
```

Record:

``` text
Expected context:

Observed context:

Match:
YES / NO
```

If the context is wrong:

``` text
STOP
```

Do not continue against an unexpected cluster.

------------------------------------------------------------------------

# 5. Verify Cluster Connectivity

Run:

``` bash
kubectl cluster-info
```

Record:

``` text
Kubernetes API reachable:
YES / NO
```

Remember:

``` text
KUBERNETES API REACHABLE
≠
APPLICATION NETWORK PATH HEALTHY
```

------------------------------------------------------------------------

# 6. Verify Namespace

Run:

``` bash
kubectl get namespace <namespace>
```

Record:

``` text
Namespace:

Exists:
YES / NO
```

Namespace existence proves only that the namespace object exists.

------------------------------------------------------------------------

# 7. Discover the Actual Routing Implementation

Do not assume that an externally exposed TrueFoundry workload always
uses a Kubernetes `Ingress` object.

Safely inspect the resources that exist in the environment.

Examples:

``` bash
kubectl get ingress -n <namespace>
```

If Gateway API is installed and relevant:

``` bash
kubectl get gateway -n <namespace>
```

``` bash
kubectl get httproute -n <namespace>
```

If Istio CRDs are installed and relevant, inspect the approved routing
resources used by the environment.

Record:

``` text
Routing implementation:

Istio / Ingress / Gateway / Other / UNKNOWN

Routing resource:

Controller / gateway:
If known

Authoritative configuration source:
If known
```

Rule:

``` text
DISCOVER THE ACTUAL ROUTING IMPLEMENTATION
BEFORE TROUBLESHOOTING IT
```

------------------------------------------------------------------------

# 8. Validate External DNS

From the affected or equivalent approved observation point, perform an
approved DNS lookup.

Examples of tools may include:

``` text
nslookup
dig
Resolve-DnsName
```

Record:

``` text
Hostname:

Observation point:

Resolver:

Resolved target:

Expected target:

TTL:
If available

Timestamp:
```

Classify:

``` text
DNS resolution:
PASS / FAIL / UNKNOWN

DNS target correctness:
PASS / FAIL / UNKNOWN
```

Rules:

``` text
DNS RECORD EXISTS
≠
DNS RESOLUTION SUCCEEDS

DNS RESOLVES
≠
DNS RESOLVES TO EXPECTED TARGET

DNS SUCCESS
≠
TCP SUCCESS
```

------------------------------------------------------------------------

# 9. Compare DNS From Different Observation Points

Only when relevant and approved, compare resolution from:

``` text
Affected client
Corporate network
VPN
Bastion
Kubernetes environment
Public resolver
```

Record:

``` text
Observation point A:
Result:

Observation point B:
Result:

Same:
YES / NO / UNKNOWN
```

Potential evidence includes:

``` text
Split-horizon DNS
Private/public DNS differences
Resolver caching
Old load-balancer target
Wrong environment
```

Do not modify DNS during this lab.

------------------------------------------------------------------------

# 10. Validate TCP Connectivity

Using an approved non-mutating connectivity check from the affected
observation point, test the expected destination and port.

Record:

``` text
Destination:

Port:

Observation point:

TCP connectivity:
PASS / FAIL / UNKNOWN

Timestamp:
```

Rules:

``` text
DNS SUCCESS
≠
TCP SUCCESS

TCP SUCCESS
≠
TLS SUCCESS

TCP SUCCESS
≠
APPLICATION TRANSACTION SUCCESS
```

Do not treat a TCP failure as proof of the responsible team or
component.

------------------------------------------------------------------------

# 11. Identify TLS Termination Point

For HTTPS endpoints, determine the supported termination layer.

Possible values:

``` text
Cloud load balancer
Istio gateway
Ingress controller
Gateway
Service mesh
Application
Other
UNKNOWN
```

Record:

``` text
TLS termination point:

Evidence:

Confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Rule:

``` text
UNKNOWN TLS TERMINATION POINT
=
DO NOT ASSUME CERTIFICATE OWNER
```

------------------------------------------------------------------------

# 12. Inspect the Presented Certificate

Use an approved TLS inspection method against the endpoint.

Capture only certificate metadata.

Record:

``` text
Requested hostname:

SNI hostname:

Presented certificate subject:

Issuer:

SAN covers requested hostname:
YES / NO / UNKNOWN

Valid from:

Valid until:

Trust result:
PASS / FAIL / UNKNOWN

TLS handshake:
PASS / FAIL / UNKNOWN
```

Do not extract private keys.

Do not decode Kubernetes TLS Secrets.

Rules:

``` text
CERTIFICATE EXISTS
≠
EXPECTED CERTIFICATE PRESENTED

CERTIFICATE SECRET CORRECT
≠
CERTIFICATE PRESENTED TO CLIENT

CERTIFICATE NOT EXPIRED
≠
CERTIFICATE TRUSTED

CERTIFICATE TRUSTED
≠
HOSTNAME VALID

TLS SUCCESS
≠
HTTP ROUTING SUCCESS
```

------------------------------------------------------------------------

# 13. Capture HTTP Evidence

Using an approved read-only HTTP request, record:

``` text
Requested URL:

Observation point:

HTTP status:

Relevant response headers:

Responding component:
If identifiable

Response timestamp:
```

Do not send state-changing application requests.

Prefer a safe endpoint or method approved for production validation.

Rules:

``` text
502 ≠ ROOT CAUSE

503 ≠ ROOT CAUSE

504 ≠ ROOT CAUSE

404 ≠ ROOT CAUSE

404 ≠ PROOF APPLICATION GENERATED RESPONSE

HTTP 200
≠
COMPLETE BUSINESS TRANSACTION HEALTH
```

------------------------------------------------------------------------

# 14. Inspect Kubernetes Ingress When Present

If Kubernetes Ingress is actually used:

``` bash
kubectl get ingress <ingress> -n <namespace>
```

``` bash
kubectl describe ingress <ingress> -n <namespace>
```

Record:

``` text
Ingress:

Ingress class:

Observed address:

Host:

Path:

Backend Service:

Backend port:

Relevant conditions/events:
```

Rules:

``` text
INGRESS RESOURCE
≠
INGRESS CONTROLLER

INGRESS EXISTS
≠
CONTROLLER ACCEPTED CONFIGURATION

INGRESS ADDRESS ASSIGNED
≠
ENDPOINT HEALTHY
```

------------------------------------------------------------------------

# 15. Inspect Gateway API Resources When Present

If Gateway API is used and the relevant resource types are installed:

``` bash
kubectl get gateway <gateway> -n <namespace>
```

``` bash
kubectl describe gateway <gateway> -n <namespace>
```

For an HTTPRoute:

``` bash
kubectl get httproute <route> -n <namespace>
```

``` bash
kubectl describe httproute <route> -n <namespace>
```

Record:

``` text
Gateway:

Route:

Hostnames:

Path match:

Backend reference:

Backend port:

Accepted:
YES / NO / UNKNOWN

Programmed:
YES / NO / UNKNOWN

Relevant conditions:
```

Do not modify route resources.

------------------------------------------------------------------------

# 16. Inspect Istio Routing When Present

If the environment uses Istio, identify the actual routing resources
used by the platform and inspect them read-only.

Record:

``` text
Istio ingress gateway:

Gateway configuration:

Routing object:

Requested host:

Configured host:

Requested path:

Configured path:

Backend Service:

Backend port:

Reconciliation/status evidence:
If available
```

Do not assume that object names are identical across TrueFoundry
installations.

Do not edit Istio resources during diagnosis.

------------------------------------------------------------------------

# 17. Validate Host and Path Routing

Compare:

``` text
Requested host:

Configured host:

Match:
YES / NO / UNKNOWN

Requested path:

Configured path:

Match:
YES / NO / UNKNOWN

Expected backend Service:

Configured backend Service:

Match:
YES / NO / UNKNOWN

Expected backend port:

Configured backend port:

Match:
YES / NO / UNKNOWN
```

Rules:

``` text
GATEWAY REACHABLE
≠
REQUEST ROUTED CORRECTLY

HOST MATCH
≠
PATH MATCH

ROUTE EXISTS
≠
ROUTE SELECTS EXPECTED SERVICE
```

------------------------------------------------------------------------

# 18. Inspect the Kubernetes Service

Run:

``` bash
kubectl get service <service> -n <namespace>
```

Then:

``` bash
kubectl describe service <service> -n <namespace>
```

Record:

``` text
Service:

Namespace:

Type:

ClusterIP:

Selector:

Port:

targetPort:

Protocol:
```

Rules:

``` text
SERVICE EXISTS
≠
SERVICE HAS ENDPOINTS

SERVICE TYPE
≠
PROOF OF ACTUAL END-TO-END EXPOSURE
```

------------------------------------------------------------------------

# 19. Validate Service Selector

Use the selector reported by the Service.

List matching Pods using the actual selector.

Example:

``` bash
kubectl get pods -n <namespace> -l <label-key>=<label-value> -o wide
```

Do not guess labels.

Record:

``` text
Service selector:

Expected workload:

Observed matching Pods:

Matching Pod count:
```

Classify:

``` text
Selector matches intended workload:
YES / NO / UNKNOWN
```

Rules:

``` text
SERVICE SELECTOR EXISTS
≠
SELECTOR MATCHES INTENDED PODS

NON-ZERO MATCHING PODS
≠
CORRECT MATCHING PODS
```

------------------------------------------------------------------------

# 20. Inspect EndpointSlices

List EndpointSlices in the namespace:

``` bash
kubectl get endpointslices -n <namespace>
```

When the Service is known, use its standard Service-name label:

``` bash
kubectl get endpointslices -n <namespace> -l kubernetes.io/service-name=<service>
```

Record:

``` text
EndpointSlice:

Endpoint count:

Ready endpoint count:

Expected endpoint present:
YES / NO / UNKNOWN
```

Rules:

``` text
ENDPOINTSLICE EXISTS
≠
EXPECTED ENDPOINT PRESENT

ENDPOINT PRESENT
≠
ENDPOINT READY

NON-ZERO ENDPOINTS
≠
CORRECT ENDPOINTS
```

------------------------------------------------------------------------

# 21. Describe Relevant EndpointSlice

For a relevant slice:

``` bash
kubectl describe endpointslice <endpointslice> -n <namespace>
```

Inspect:

``` text
Addresses
Conditions
Target references
Ports
```

Record:

``` text
Endpoint address:

Ready:

Serving:
If available

Terminating:
If available

Target reference:

Endpoint port:
```

Do not edit EndpointSlices.

------------------------------------------------------------------------

# 22. Check `publishNotReadyAddresses`

Inspect the Service safely:

``` bash
kubectl get service <service> -n <namespace> -o yaml
```

Locate:

``` text
publishNotReadyAddresses
```

Record:

``` text
publishNotReadyAddresses:
true / false / not set
```

Remember:

``` text
NORMAL ENDPOINT READINESS ASSUMPTIONS
MUST BE VERIFIED AGAINST SERVICE CONFIGURATION
```

------------------------------------------------------------------------

# 23. Validate `port` → `targetPort`

Record:

``` text
Service port:

targetPort:

targetPort type:
Numeric / Named

Protocol:
```

If `targetPort` is named, identify the corresponding Pod container-port
name from the workload specification.

Record:

``` text
Named targetPort:

Matching container port name:

Resolved numeric port:
If supported by evidence
```

Rules:

``` text
SERVICE PORT
≠
targetPort

targetPort
≠
PROOF APPLICATION LISTENER EXISTS

containerPort
≠
PROOF PROCESS IS LISTENING
```

------------------------------------------------------------------------

# 24. Identify the Workload Controller

Identify the controller associated with the endpoint Pod.

Examples:

``` bash
kubectl get deployment -n <namespace>
```

``` bash
kubectl get statefulset -n <namespace>
```

``` bash
kubectl get daemonset -n <namespace>
```

Inspect the known controller:

``` bash
kubectl describe deployment <workload> -n <namespace>
```

or the corresponding controller type.

Record:

``` text
Controller:

Controller type:

Revision:
If available

Expected Pod labels:

Container ports:
```

------------------------------------------------------------------------

# 25. Correlate Endpoint to Pod

Use EndpointSlice target references and Pod information.

Run:

``` bash
kubectl get pod <pod> -n <namespace> -o wide
```

Record:

``` text
Endpoint IP:

Target Pod:

Pod IP:

Node:

Pod phase:

Ready:

Revision:
If known
```

Classify:

``` text
Endpoint maps to expected Pod:
YES / NO / UNKNOWN
```

------------------------------------------------------------------------

# 26. Check Pod Readiness and Restarts

Run:

``` bash
kubectl get pod <pod> -n <namespace>
```

Then:

``` bash
kubectl describe pod <pod> -n <namespace>
```

Record:

``` text
Pod phase:

Ready condition:

Container:

Restart count:

Relevant readiness failure:

Relevant event:
```

Rules:

``` text
POD RUNNING
≠
POD READY

POD READY
≠
REQUESTED BUSINESS OPERATION HEALTHY
```

------------------------------------------------------------------------

# 27. Compare All Backends

For intermittent incidents, correlate each endpoint with its Pod.

Evidence table:

``` text
Endpoint | Pod | Node | Revision | Ready | Restarts | Behavior
---------|-----|------|----------|-------|----------|---------
         |     |      |          |       |          |
         |     |      |          |       |          |
         |     |      |          |       |          |
```

Rule:

``` text
ONE SUCCESSFUL REQUEST
≠
ALL BACKENDS HEALTHY
```

If only one endpoint is implicated, do not automatically expand the
incident to the entire Service.

------------------------------------------------------------------------

# 28. Compare Revisions

When multiple ReplicaSets or revisions exist:

``` bash
kubectl get replicasets -n <namespace>
```

Inspect only relevant controllers.

Record:

``` text
Healthy Pod revision:

Affected Pod revision:

Same revision:
YES / NO / UNKNOWN
```

Rule:

``` text
DIFFERENT REVISION
≠
ROOT CAUSE
```

Treat revision difference as evidence requiring further validation.

------------------------------------------------------------------------

# 29. Validate Application Listener Evidence

Determine:

``` text
Expected application port:

Service targetPort:

Declared container port:

Application listener:
PROVEN / SUPPORTED / UNKNOWN
```

Use approved application documentation, startup logs, safe metrics, or
other non-mutating evidence.

Do not use intrusive production commands solely to prove a hypothesis.

Rule:

``` text
POD RUNNING
≠
APPLICATION LISTENING
```

------------------------------------------------------------------------

# 30. Review Application Logs Safely

When application logs are necessary:

``` bash
kubectl logs <pod> -n <namespace> -c <container>
```

If the container restarted and previous logs are relevant:

``` bash
kubectl logs <pod> -n <namespace> -c <container> --previous
```

Use only the minimum relevant time range/output supported by operational
policy.

Do not copy:

-   credentials;
-   tokens;
-   authorization headers;
-   private data;
-   Secret values.

Record:

``` text
Relevant timestamp:

Relevant listener/startup evidence:

Relevant request/error evidence:
```

------------------------------------------------------------------------

# 31. Review Kubernetes Events

Run:

``` bash
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

Correlate event timestamps with the incident.

Record:

``` text
Relevant event:

Object:

Timestamp:

Relationship to incident:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Rule:

``` text
TIMING CORRELATION
≠
ROOT CAUSE PROOF
```

------------------------------------------------------------------------

# 32. Validate Internal Request Path

Use existing approved monitoring, application telemetry, or an
already-authorized in-cluster observation point.

Do not create a diagnostic Pod for this production lab.

Record:

``` text
Internal observation point:

Kubernetes DNS:
PASS / FAIL / UNKNOWN

Service reachability:
PASS / FAIL / UNKNOWN

Application response:
PASS / FAIL / UNKNOWN
```

Rule:

``` text
INTERNAL SUCCESS
≠
EXTERNAL SUCCESS
```

------------------------------------------------------------------------

# 33. Compare Internal and External Results

Record:

``` text
Internal path:
PASS / FAIL / UNKNOWN

External path:
PASS / FAIL / UNKNOWN
```

If:

``` text
Internal = PASS
External = FAIL
```

move investigation upstream toward:

``` text
External DNS
Load balancer
TLS
Gateway
Host/path routing
External network
```

Use:

``` text
INTERNAL BACKEND SUCCESS
+
EXTERNAL FAILURE
=
MOVE INVESTIGATION UPSTREAM
```

This narrows the failure domain but does not prove root cause.

------------------------------------------------------------------------

# 34. External Success Does Not Prove Internal Health

If:

``` text
External = PASS
Internal consumer = FAIL
```

investigate:

``` text
Kubernetes DNS
Namespace resolution
Service name
Service port
NetworkPolicy
Client configuration
Protocol
```

Rule:

``` text
EXTERNAL SUCCESS
≠
ALL INTERNAL CONSUMERS HEALTHY
```

------------------------------------------------------------------------

# 35. Identify the Responding Layer

For HTTP failures, determine whether evidence supports the response
originating from:

``` text
Load balancer
Gateway
Ingress
Service mesh
Application
External dependency
UNKNOWN
```

Record:

``` text
HTTP status:

Responding layer:

Evidence:

Confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Do not infer ownership from status code alone.

------------------------------------------------------------------------

# 36. Check Desired vs Rendered vs Programmed Route

Record:

``` text
Desired route:

Rendered route:

Programmed route:

Observed request path:
```

Classify each:

``` text
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Central rule:

``` text
DESIRED ROUTE
≠
RENDERED ROUTE
≠
PROGRAMMED ROUTE
≠
OBSERVED REQUEST PATH
```

------------------------------------------------------------------------

# 37. Identify Authoritative Routing Owner

Determine the actual desired-state source.

Possible values:

``` text
TrueFoundry
GitOps
Helm
Istio
Gateway API
Kubernetes manifests
External DNS
Certificate manager
Cloud certificate service
Cloud load balancer
Terraform / OpenTofu
Application repository
Operator
Other
UNKNOWN
```

Record:

``` text
Observed routing object:

Authoritative configuration source:

Authoritative Configuration Owner:

Ownership confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED
```

Rule:

``` text
OBSERVED KUBERNETES OBJECT
≠
AUTHORITATIVE CONFIGURATION SOURCE
```

------------------------------------------------------------------------

# 38. Identify Shared Infrastructure

Determine whether the affected path uses:

``` text
Shared gateway
Shared ingress controller
Shared load balancer
Shared DNS zone
Shared certificate
Wildcard certificate
Shared TLS Secret
Shared Service
Shared backend
```

Record:

``` text
Shared component:

Known consumers:

Consumer scope complete:
YES / NO / UNKNOWN
```

Rules:

``` text
LOAD BALANCER CHANGE
=
SHARED INFRASTRUCTURE + DNS + AVAILABILITY EVENT

TLS CHANGE
=
SECURITY + ROUTING + AVAILABILITY EVENT
```

This lab performs no such change.

------------------------------------------------------------------------

# 39. Build the Incident Timeline

Record:

``` text
T1 Last known healthy:

T2 DNS/routing/certificate change:
If known

T3 Deployment/revision change:
If known

T4 Symptom began:

T5 Alert fired:

T6 Investigation began:
```

Use `UNKNOWN` when evidence is unavailable.

Rule:

``` text
TIMING CORRELATION
≠
ROOT CAUSE PROOF
```

------------------------------------------------------------------------

# 40. Determine Lowest Proven Healthy Layer

Use:

``` text
Client
  ↓
DNS
  ↓
TCP
  ↓
TLS
  ↓
Load Balancer
  ↓
Gateway / Ingress
  ↓
Host / Path Route
  ↓
Service
  ↓
EndpointSlice
  ↓
Endpoint
  ↓
Pod
  ↓
Application Listener
  ↓
Application Response
  ↓
Business Transaction
```

Record:

``` text
Lowest Proven Healthy Layer:
```

Examples:

``` text
Expected DNS target
TCP connection
TLS handshake
Gateway listener
Host/path route
Service
Ready EndpointSlice backend
Application listener
Application response
```

Do not claim a layer healthy without evidence.

------------------------------------------------------------------------

# 41. Determine First Failed Transition

Examples:

``` text
Hostname
→
Expected DNS target
```

``` text
TCP connection
→
TLS handshake
```

``` text
SNI hostname
→
Expected certificate
```

``` text
Gateway
→
Expected route
```

``` text
Route
→
Backend Service
```

``` text
Service selector
→
Expected Pods
```

``` text
Service
→
Ready EndpointSlice backend
```

``` text
Endpoint
→
Application listener
```

``` text
Application response
→
Business transaction
```

Record:

``` text
First Failed Transition:
```

Do not write only:

``` text
Network issue
Ingress issue
Kubernetes issue
TrueFoundry issue
Application issue
```

unless the evidence supports that exact boundary.

------------------------------------------------------------------------

# 42. Determine Minimum Supported Blast Radius

Choose the smallest evidence-supported scope:

``` text
One client
One hostname
One path
One route
One Service
One EndpointSlice
One endpoint
One Pod
One revision
One namespace
One gateway
One load balancer
One DNS zone
One environment
Multiple environments
```

Record:

``` text
Minimum Supported Blast Radius:

Evidence:
```

Rules:

``` text
ONE HOSTNAME FAILURE
≠
ENTIRE GATEWAY FAILURE

ONE SERVICE FAILURE
≠
CLUSTER NETWORKING FAILURE

ONE ENDPOINT FAILURE
≠
ENTIRE SERVICE FAILURE
```

------------------------------------------------------------------------

# 43. Assign Evidence Confidence

Use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

``` text
DNS resolves to LB-A                      PROVEN
Expected target is LB-B                   PROVEN
TCP to LB-A succeeds                      PROVEN
Wrong DNS target caused outage            SUPPORTED
Backend application is unhealthy          UNKNOWN
```

Rule:

``` text
ASSUMED ≠ ROOT CAUSE
```

------------------------------------------------------------------------

# 44. Identify Current Actionable Owner

Use:

``` text
First Failed Transition
+
Authoritative Configuration Owner
=
Current Actionable Owner
```

Record:

``` text
Authoritative Configuration Owner:

Current Actionable Owner:

Requested Action:
```

Possible requested actions:

``` text
Validate expected DNS target.

Validate gateway route reconciliation.

Validate expected certificate binding.

Validate Service selector.

Validate targetPort/application listener mapping.

Validate approved desired-state configuration.

Validate shared-infrastructure change history.
```

Do not request:

``` text
Patch it and see.
Restart it and see.
Replace the certificate and see.
Change DNS and see.
```

------------------------------------------------------------------------

# 45. Preserve Evidence Before Remediation

Before any future approved change, ensure the incident record contains:

``` text
Original client error
Observation point
Timestamp
DNS result
Expected DNS target
TCP result
TLS result
Presented certificate metadata
HTTP status
Relevant response headers
Routing configuration
Service configuration
EndpointSlices
Endpoint readiness
Endpoint-to-Pod mapping
Pod revision
Relevant events
Relevant safe logs
Lowest Proven Healthy Layer
First Failed Transition
Minimum Supported Blast Radius
Authoritative Configuration Owner
Current Actionable Owner
```

This lab itself performs no remediation.

------------------------------------------------------------------------

# 46. Production Networking Evidence Handoff

Complete:

``` text
Environment:
TrueFoundry Workspace:

Cluster:
Kubernetes context:
Namespace:

Observation point:
Observation timestamp:

Requested scheme:
Requested hostname:
Requested port:
Requested path:

Original symptom:

Expected request path:

DNS resolver:
Resolved target:
Expected DNS target:

TCP connectivity:
PASS / FAIL / UNKNOWN

TLS termination point:

TLS handshake:
PASS / FAIL / UNKNOWN

Presented certificate:
Issuer:
SAN match:
Validity:
Trust:
SNI hostname:

Routing implementation:
Istio / Ingress / Gateway / Other / UNKNOWN

Route:
Host:
Path:

Backend Service:
Service type:
Service port:
targetPort:
Selector:

EndpointSlice:
Endpoint count:
Ready endpoint count:

Affected endpoint:
Affected Pod:
Container:
Application listener:

Internal path:
PASS / FAIL / UNKNOWN

External path:
PASS / FAIL / UNKNOWN

HTTP status:
Responding layer:
If known

Lowest Proven Healthy Layer:

First Failed Transition:

Minimum Supported Blast Radius:

Evidence Confidence:
PROVEN / SUPPORTED / UNKNOWN / ASSUMED

Authoritative Configuration Owner:

Current Actionable Owner:

Requested Action:

DNS modified:
NO

Service modified:
NO

Route modified:
NO

Certificate modified:
NO

Secret decoded:
NO

Pod restarted:
NO

Shared infrastructure modified:
NO
```

------------------------------------------------------------------------

# 47. Acceptance Checklist

The lab passes when the following are answered with evidence or
explicitly recorded as `UNKNOWN`.

``` text
[ ] Correct Kubernetes context proven.

[ ] Namespace proven.

[ ] Original client symptom captured.

[ ] Observation point captured.

[ ] Hostname, port, protocol, and path captured.

[ ] Expected request path documented.

[ ] External DNS result captured when applicable.

[ ] Expected DNS target compared with observed target.

[ ] TCP result captured when applicable.

[ ] TLS termination point identified or marked UNKNOWN.

[ ] TLS handshake result captured when applicable.

[ ] Presented certificate metadata captured when applicable.

[ ] SNI/hostname relationship checked.

[ ] Actual routing implementation discovered.

[ ] Host/path routing checked.

[ ] Backend Service identified.

[ ] Service type captured.

[ ] Service port captured.

[ ] targetPort captured.

[ ] Service selector validated.

[ ] EndpointSlices inspected.

[ ] Expected endpoints validated.

[ ] Endpoint readiness checked.

[ ] publishNotReadyAddresses checked when relevant.

[ ] Endpoint-to-Pod mapping validated.

[ ] Pod readiness checked.

[ ] Backend revisions compared when relevant.

[ ] Application listener evidence evaluated.

[ ] Internal path classified.

[ ] External path classified.

[ ] HTTP status treated as evidence, not root cause.

[ ] Desired/rendered/programmed/observed route states considered.

[ ] Shared infrastructure identified.

[ ] Authoritative configuration source identified or marked UNKNOWN.

[ ] Incident timeline recorded.

[ ] Lowest Proven Healthy Layer identified.

[ ] First Failed Transition identified.

[ ] Minimum Supported Blast Radius identified.

[ ] Evidence Confidence assigned.

[ ] Current Actionable Owner identified.

[ ] Requested Action recorded.

[ ] DNS modified = NO.

[ ] Service modified = NO.

[ ] Route modified = NO.

[ ] Certificate modified = NO.

[ ] Secret decoded = NO.

[ ] Pod restarted = NO.

[ ] Shared infrastructure modified = NO.
```

------------------------------------------------------------------------

# 48. Production Investigation Flow

``` text
CAPTURE ORIGINAL CLIENT SYMPTOM
        ↓
IDENTIFY OBSERVATION POINT
        ↓
CAPTURE HOST / PORT / PROTOCOL / PATH
        ↓
VERIFY DNS RESOLUTION
        ↓
VERIFY EXPECTED DNS TARGET
        ↓
VERIFY TCP CONNECTIVITY
        ↓
IDENTIFY TLS TERMINATION POINT
        ↓
VERIFY TLS HANDSHAKE
        ↓
VERIFY PRESENTED CERTIFICATE
        ↓
DISCOVER ACTUAL ROUTING IMPLEMENTATION
        ↓
VERIFY ROUTE RECONCILIATION
        ↓
VERIFY HOST / PATH
        ↓
VERIFY BACKEND SERVICE
        ↓
VERIFY SERVICE PORT / targetPort
        ↓
VERIFY SERVICE SELECTOR
        ↓
VERIFY ENDPOINTSLICE
        ↓
VERIFY EXPECTED ENDPOINTS
        ↓
VERIFY ENDPOINT READINESS
        ↓
CORRELATE ENDPOINT → POD → REVISION
        ↓
VERIFY APPLICATION LISTENER
        ↓
VERIFY APPLICATION RESPONSE
        ↓
VERIFY BUSINESS TRANSACTION
        ↓
COMPARE INTERNAL / EXTERNAL PATHS
        ↓
IDENTIFY LOWEST PROVEN HEALTHY LAYER
        ↓
IDENTIFY FIRST FAILED TRANSITION
        ↓
PROVE MINIMUM SUPPORTED BLAST RADIUS
        ↓
IDENTIFY AUTHORITATIVE CONFIGURATION OWNER
        ↓
IDENTIFY CURRENT ACTIONABLE OWNER
        ↓
HAND OFF APPROVED REMEDIATION
        ↓
REGRESSION VALIDATION
```

------------------------------------------------------------------------

# 49. Lab Guardrails

The following must remain true throughout the lab:

``` text
UNKNOWN TARGET = NO MUTATION

UNKNOWN ROUTING IMPLEMENTATION = NO ROUTE CHANGE

UNKNOWN TLS TERMINATION POINT = NO CERTIFICATE CHANGE

UNKNOWN AUTHORITATIVE OWNER = NO DIRECT CONFIGURATION CHANGE

KUBERNETES SERVICE DNS
≠
EXTERNAL APPLICATION DNS

DNS RESOLVES
≠
DNS RESOLVES TO EXPECTED TARGET

DNS SUCCESS
≠
TCP SUCCESS

TCP SUCCESS
≠
TLS SUCCESS

TLS SUCCESS
≠
HTTP ROUTING SUCCESS

CERTIFICATE EXISTS
≠
EXPECTED CERTIFICATE PRESENTED

CERTIFICATE SECRET CORRECT
≠
CERTIFICATE PRESENTED TO CLIENT

TLS VALIDATION
≠
SECRET EXTRACTION

INGRESS RESOURCE
≠
INGRESS CONTROLLER

CONTROLLER RUNNING
≠
ROUTE RECONCILED

ROUTE EXISTS
≠
ROUTE SELECTS EXPECTED SERVICE

SERVICE EXISTS
≠
SERVICE HAS ENDPOINTS

SERVICE SELECTOR EXISTS
≠
SELECTOR MATCHES INTENDED PODS

NON-ZERO ENDPOINTS
≠
CORRECT ENDPOINTS

ENDPOINT PRESENT
≠
ENDPOINT READY

ENDPOINT READY
≠
APPLICATION TRANSACTION HEALTHY

SERVICE PORT
≠
targetPort

targetPort
≠
PROOF APPLICATION LISTENER EXISTS

POD RUNNING
≠
APPLICATION LISTENING

ONE SUCCESSFUL REQUEST
≠
ALL BACKENDS HEALTHY

INTERNAL SUCCESS
≠
EXTERNAL SUCCESS

EXTERNAL SUCCESS
≠
ALL INTERNAL CONSUMERS HEALTHY

DESIRED ROUTE
≠
RENDERED ROUTE
≠
PROGRAMMED ROUTE
≠
OBSERVED REQUEST PATH

502 ≠ ROOT CAUSE

503 ≠ ROOT CAUSE

504 ≠ ROOT CAUSE

404 ≠ ROOT CAUSE

TIMEOUT
≠
AUTOMATICALLY NETWORK FAILURE

Observed Drift
≠
Permission to Patch

ASSUMED ≠ ROOT CAUSE

SYMPTOM ≠ FAILURE DOMAIN ≠ OWNER
```

------------------------------------------------------------------------

# 50. Completion Standard

This lab is complete when the engineer can answer:

``` text
WHERE did the failing request originate?

WHAT hostname, port, protocol, and path were requested?

WHAT request path was expected?

DID DNS resolve?

DID DNS resolve to the expected target?

DID TCP connectivity succeed?

WHERE does TLS terminate?

WHAT certificate was actually presented?

DID TLS validation succeed?

WHAT routing implementation is actually used?

DID host/path routing select the expected Service?

DOES the Service select the intended workload?

WHAT are the Service port and targetPort?

DOES the EndpointSlice contain the expected backends?

ARE the expected endpoints ready?

IS one backend behaving differently?

IS the application listener supported by evidence?

DO internal and external paths behave differently?

WHAT is the Lowest Proven Healthy Layer?

WHAT is the First Failed Transition?

WHAT is the Minimum Supported Blast Radius?

WHO owns authoritative configuration?

WHO is the Current Actionable Owner?

WHAT action is requested next?

WAS DNS changed?
NO

WAS the Service changed?
NO

WAS routing changed?
NO

WAS a certificate changed?
NO

WAS a Secret decoded?
NO

WAS a Pod restarted?
NO

WAS shared infrastructure changed?
NO
```

The purpose is not to repair production by experimentation. The purpose
is to produce a safe, evidence-backed handoff that identifies the first
proven failure boundary and the owner of the next approved action.
