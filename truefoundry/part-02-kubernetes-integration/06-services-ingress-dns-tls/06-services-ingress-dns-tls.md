# Part 2.6 --- Services, Ingress, DNS & TLS

## Purpose

This tutorial explains how to investigate the network path for
TrueFoundry workloads running on Kubernetes.

The goal is not to label an incident as a "network issue." The goal is
to prove the request path one transition at a time and identify the
first transition that does not match the expected production path.

The central production question is:

> From the affected client's observation point, where does the request
> path first stop matching the expected path, what evidence proves that
> transition failed, what is the smallest supported blast radius, and
> who owns the authoritative configuration for that layer?

This tutorial focuses on:

-   Kubernetes Services;
-   Service selectors;
-   Service ports and `targetPort`;
-   EndpointSlices;
-   endpoint readiness;
-   Kubernetes Service DNS;
-   external application DNS;
-   load balancers and external entry points;
-   TrueFoundry and Istio routing concepts;
-   Kubernetes Ingress and Gateway concepts;
-   host and path routing;
-   TLS termination;
-   SNI and certificate validation;
-   internal versus external request paths;
-   application listeners;
-   HTTP response interpretation;
-   replica-level network failures;
-   shared-networking blast radius;
-   desired-state ownership;
-   production evidence collection.

The operating principle is:

``` text
DO NOT TROUBLESHOOT "THE NETWORK."

PROVE THE REQUEST PATH
ONE TRANSITION AT A TIME.
```

------------------------------------------------------------------------

## 1. The Production Request Path

A public application request may traverse several independent systems
before reaching the application.

A useful external request model is:

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

An internal request may follow a different path:

``` text
In-Cluster Client
      ↓
Kubernetes DNS
      ↓
Kubernetes Service
      ↓
EndpointSlice
      ↓
Ready Endpoint
      ↓
Pod
      ↓
Application
```

These paths must not be treated as equivalent.

``` text
INTERNAL SUCCESS
≠
EXTERNAL SUCCESS

EXTERNAL SUCCESS
≠
ALL INTERNAL CONSUMERS HEALTHY
```

------------------------------------------------------------------------

## 2. TrueFoundry Networking Context

TrueFoundry provides platform abstractions for deploying and exposing
workloads, but the runtime request path ultimately traverses networking
infrastructure.

Depending on the installed TrueFoundry architecture and cluster
configuration, that infrastructure can include:

-   Kubernetes Services;
-   Istio ingress gateways;
-   cloud load balancers;
-   configured domains;
-   DNS infrastructure;
-   TLS certificates;
-   Kubernetes networking;
-   application Pods.

Current TrueFoundry architectures commonly use Istio-based ingress
infrastructure. However, production troubleshooting must discover the
actual implementation rather than assume that every cluster, deployment
version, or endpoint uses an identical path.

Use:

``` text
DISCOVER THE ACTUAL ROUTING IMPLEMENTATION
BEFORE TROUBLESHOOTING IT
```

Possible routing implementations include:

``` text
Istio
Kubernetes Ingress
Gateway API
Cloud load-balancer routing
Service mesh
Application-level routing
Other
UNKNOWN
```

Do not assume that a Kubernetes `Ingress` object must exist simply
because an application has an externally reachable endpoint.

------------------------------------------------------------------------

## 3. Management Configuration vs Runtime Request Path

A networking configuration can exist successfully in a management system
while the runtime request path is unhealthy.

For example:

``` text
TrueFoundry configuration exists
        ↓
Routing configuration rendered
        ↓
Gateway/controller reconciles configuration
        ↓
Load balancer receives traffic
        ↓
Request reaches Service
        ↓
Service selects endpoint
        ↓
Application receives request
```

Each transition can fail independently.

Therefore:

``` text
PLATFORM CONFIGURATION EXISTS
≠
RUNTIME REQUEST PATH HEALTHY
```

and:

``` text
VISIBLE IN TRUEFOUNDRY
≠
TRUEFOUNDRY OWNS FAILED LAYER
```

------------------------------------------------------------------------

## 4. Four Routing States

For production troubleshooting, distinguish four states:

``` text
DESIRED ROUTE
      ↓
RENDERED ROUTE
      ↓
PROGRAMMED ROUTE
      ↓
OBSERVED REQUEST PATH
```

### Desired Route

What the authoritative configuration says should exist.

Examples:

-   TrueFoundry service exposure configuration;
-   GitOps configuration;
-   Helm values;
-   Terraform/OpenTofu;
-   Istio routing configuration;
-   Gateway configuration;
-   DNS configuration.

### Rendered Route

What Kubernetes or the routing system currently represents.

Examples:

-   Service;
-   Gateway;
-   VirtualService;
-   Ingress;
-   route object;
-   load-balancer configuration.

### Programmed Route

What the controller, gateway, load balancer, or other data-plane
component has actually accepted and programmed.

### Observed Request Path

What a real request actually experiences.

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

## 5. Start With the Observation Point

Before investigating DNS, TLS, Services, Pods, or applications, identify
where the failing request originates.

Possible observation points include:

``` text
External client
Corporate network
VPN
Bastion
Kubernetes Pod
Different Kubernetes namespace
Ingress/Gateway Pod
Backend Pod
Monitoring system
Synthetic probe
```

Record:

``` text
Observation point:
Source:
Timestamp:

Requested scheme:
Requested hostname:
Requested port:
Requested path:

Expected response:
Observed response:
```

Production rule:

``` text
UNKNOWN OBSERVATION POINT
=
INCOMPLETE NETWORK EVIDENCE
```

A successful request from one observation point does not prove that
another path is healthy.

------------------------------------------------------------------------

## 6. Separate Network Transitions

Do not investigate an endpoint as one undifferentiated object.

Break the request into transitions:

``` text
NAME RESOLUTION
      ↓
TCP CONNECTION
      ↓
TLS HANDSHAKE
      ↓
HTTP / GATEWAY ROUTING
      ↓
SERVICE ROUTING
      ↓
ENDPOINT DELIVERY
      ↓
APPLICATION RESPONSE
      ↓
BUSINESS TRANSACTION
```

Important rules:

``` text
DNS SUCCESS
≠
TCP SUCCESS

TCP SUCCESS
≠
TLS SUCCESS

TLS SUCCESS
≠
HTTP ROUTING SUCCESS

HTTP ROUTING SUCCESS
≠
BACKEND SUCCESS

BACKEND RESPONSE
≠
BUSINESS TRANSACTION SUCCESS
```

The investigation should find the first transition where the observed
behavior diverges from the expected behavior.

------------------------------------------------------------------------

# DNS

## 7. Kubernetes Service DNS vs External DNS

Two distinct DNS systems can participate in the same application
architecture.

### Kubernetes Service DNS

Inside the cluster, workloads commonly reach Kubernetes Services using
names associated with Kubernetes DNS.

Conceptually:

``` text
Client Pod
   ↓
Kubernetes DNS
   ↓
Service
```

### External Application DNS

External users normally resolve an application hostname to an external
entry point such as a load balancer.

Conceptually:

``` text
External Client
   ↓
External DNS
   ↓
Load Balancer
```

Do not combine these into one failure domain.

``` text
KUBERNETES SERVICE DNS
≠
EXTERNAL APPLICATION DNS
```

------------------------------------------------------------------------

## 8. DNS Resolution Is Not DNS Correctness

A successful DNS lookup proves that a resolver returned an answer.

It does not prove that the answer is correct.

Record:

``` text
Hostname:
Observation point:
Resolver:
Resolved target:
Expected target:
TTL:
Timestamp:
```

Core rules:

``` text
DNS RECORD EXISTS
≠
DNS RESOLUTION SUCCEEDS

DNS RESOLVES
≠
DNS RESOLVES TO EXPECTED TARGET

DNS RESOLUTION SUCCESS
≠
TCP CONNECTIVITY

DNS FAILURE
≠
AUTOMATICALLY KUBERNETES FAILURE
```

Production DNS problems can include:

-   stale load-balancer target;
-   wrong environment;
-   wrong DNS zone;
-   private/public zone differences;
-   split-horizon DNS;
-   partial propagation;
-   cached records;
-   incorrect CNAME chain;
-   resolver-specific behavior.

------------------------------------------------------------------------

## 9. DNS Observation Point Matters

The same hostname can resolve differently from:

``` text
Corporate DNS
Public resolver
VPN
Private cloud network
Kubernetes Pod
Bastion
```

Therefore:

``` text
DNS RESULT FROM ONE OBSERVATION POINT
≠
DNS RESULT FROM EVERY OBSERVATION POINT
```

Always attach DNS evidence to its source and timestamp.

------------------------------------------------------------------------

# TCP Connectivity

## 10. TCP Is a Separate Failure Boundary

After proving expected DNS resolution, determine whether the expected
host and port are reachable from the affected observation point.

Conceptually:

``` text
DNS
 ↓
TCP
 ↓
TLS
 ↓
HTTP
```

Rules:

``` text
DNS SUCCESS
≠
TCP SUCCESS

TCP CONNECTION SUCCESS
≠
TLS SUCCESS

TCP CONNECTION SUCCESS
≠
APPLICATION TRANSACTION SUCCESS
```

A TCP failure narrows the failure boundary but does not automatically
identify its owner.

Potential causes may exist in:

-   load balancer;
-   firewall;
-   security policy;
-   routing;
-   listener configuration;
-   infrastructure health;
-   network path;
-   destination availability.

Evidence must determine which explanation is supported.

------------------------------------------------------------------------

# TLS

## 11. Discover the TLS Termination Point

Do not assume that TLS terminates at Kubernetes Ingress.

Possible termination points include:

``` text
Cloud load balancer
Istio gateway
Kubernetes Ingress controller
Gateway implementation
Service mesh
Application
Other
UNKNOWN
```

Record:

``` text
TLS termination point:
```

Production rule:

``` text
UNKNOWN TLS TERMINATION POINT
=
DO NOT ASSUME CERTIFICATE OWNER
```

------------------------------------------------------------------------

## 12. TLS Validation Chain

Use:

``` text
Requested hostname
      ↓
SNI
      ↓
TLS termination point
      ↓
Certificate actually presented
      ↓
SAN / hostname validation
      ↓
Validity period
      ↓
Trust chain
      ↓
Next routing layer
```

Record:

``` text
Requested hostname:
SNI hostname:

Presented certificate:
Issuer:
Subject:
SAN coverage:
Valid from:
Valid until:
Trust result:
```

Important rules:

``` text
CERTIFICATE EXISTS
≠
EXPECTED CERTIFICATE PRESENTED

CERTIFICATE SECRET CORRECT
≠
CERTIFICATE PRESENTED TO CLIENT

EXPECTED CERTIFICATE PRESENTED
≠
CERTIFICATE TRUSTED

CERTIFICATE NOT EXPIRED
≠
CERTIFICATE TRUSTED

CERTIFICATE TRUSTED
≠
HOSTNAME VALID

TLS HANDSHAKE SUCCESS
≠
HTTP ROUTING SUCCESS
```

------------------------------------------------------------------------

## 13. Do Not Decode TLS Secrets for Routine Diagnosis

Production troubleshooting normally does not require extracting
private-key or Secret content from Kubernetes.

Use certificate information presented through the network and safe
metadata wherever possible.

``` text
TLS VALIDATION
≠
SECRET EXTRACTION
```

and:

``` text
SECRET EXISTS
≠
SECRET SHOULD BE DECODED
```

Private keys must never be copied into incident evidence.

------------------------------------------------------------------------

## 14. SNI Matters

Multiple hostnames can share the same load balancer or gateway.

The TLS endpoint can select certificate or routing behavior based on the
requested hostname.

Therefore:

``` text
CORRECT IP ADDRESS
≠
CORRECT CERTIFICATE

CORRECT LOAD BALANCER
≠
CORRECT SNI ROUTE
```

A wrong certificate can indicate:

-   wrong SNI hostname;
-   wrong listener;
-   stale gateway configuration;
-   incorrect certificate binding;
-   DNS pointing to an unexpected entry point;
-   shared gateway misconfiguration.

Do not assume that the certificate object itself is defective.

------------------------------------------------------------------------

# Gateway, Ingress, and TrueFoundry Routing

## 15. Ingress Is Not the Controller

A Kubernetes Ingress resource describes desired routing behavior.

An ingress controller is the implementation that acts on that
configuration.

Therefore:

``` text
INGRESS RESOURCE
≠
INGRESS CONTROLLER

INGRESS EXISTS
≠
CONTROLLER ACCEPTED CONFIGURATION

INGRESS CLASS EXISTS
≠
CORRECT CONTROLLER SELECTED

CONTROLLER RUNNING
≠
ROUTE RECONCILED
```

Kubernetes Ingress remains supported, but modern Kubernetes networking
can also use Gateway API implementations.

The tutorial therefore uses the broader concept:

``` text
Ingress / Gateway / Istio routing
```

------------------------------------------------------------------------

## 16. TrueFoundry and Istio Routing

TrueFoundry commonly uses Istio-based ingress infrastructure for
external service exposure.

A conceptual path can be:

``` text
External Client
      ↓
DNS
      ↓
Cloud Load Balancer
      ↓
Istio Ingress Gateway
      ↓
Routing Configuration
      ↓
Kubernetes Service
      ↓
Pod
```

However:

``` text
TRUEFOUNDRY DEPLOYMENT
≠
PROOF OF ONE UNIVERSAL ROUTING IMPLEMENTATION
```

Discover the actual route objects, gateway implementation, controller,
and ownership in the affected environment.

------------------------------------------------------------------------

## 17. Host and Path Routing

A request can successfully reach the gateway and still fail routing.

Validate:

``` text
Requested host:
Configured host:

Requested path:
Configured path:

Expected backend Service:
Configured backend Service:

Expected backend port:
Configured backend port:
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

ROUTE CONFIGURATION
≠
ROUTE PROGRAMMED
```

------------------------------------------------------------------------

## 18. Gateway Health Is Not Route Health

A healthy gateway Pod or controller proves only a limited layer.

``` text
CONTROLLER RUNNING
≠
ROUTE RECONCILED

ROUTE RECONCILED
≠
LOAD BALANCER PROGRAMMED

LOAD BALANCER PROGRAMMED
≠
DNS CORRECT

DNS CORRECT
≠
TLS CORRECT

TLS CORRECT
≠
BACKEND ROUTING CORRECT
```

Do not stop investigation because the ingress controller or gateway Pods
are healthy.

------------------------------------------------------------------------

# Kubernetes Services

## 19. Service Fundamentals

A Kubernetes Service provides a stable network abstraction for a set of
backends.

Common Service types include:

``` text
ClusterIP
NodePort
LoadBalancer
ExternalName
```

For troubleshooting, capture:

``` text
Service name:
Namespace:
Service type:
ClusterIP:
Selector:
Service port:
targetPort:
Protocol:
```

Do not infer the entire exposure architecture from Service type alone.

``` text
SERVICE TYPE
≠
PROOF OF ACTUAL END-TO-END EXPOSURE
```

------------------------------------------------------------------------

## 20. Service Existence Is Only the Beginning

A Service object can exist while no application traffic can successfully
reach the intended workload.

Rules:

``` text
SERVICE EXISTS
≠
SERVICE HAS ENDPOINTS

SERVICE SELECTOR EXISTS
≠
SELECTOR MATCHES INTENDED PODS

SERVICE SELECTOR MATCHES PODS
≠
PODS ARE READY ENDPOINTS

SERVICE SELECTOR MATCHES
≠
TARGET PORT CORRECT
```

------------------------------------------------------------------------

## 21. Validate the Service Selector

Conceptually:

``` text
Service
   ↓
Selector
   ↓
Matching Pods
```

Capture:

``` text
Service selector:
Expected workload labels:
Observed matching Pods:
```

Two major failure modes must be distinguished.

### Zero matching backends

``` text
Service
   ↓
Selector
   ↓
No matching Pods
```

### Wrong matching backends

``` text
Service
   ↓
Selector
   ↓
Unintended Pods
```

Therefore:

``` text
NON-ZERO MATCHING PODS
≠
CORRECT MATCHING PODS
```

------------------------------------------------------------------------

# Service Ports and Application Ports

## 22. `port`, `targetPort`, and Listener Are Different

Example:

``` text
Client
  ↓
Gateway :443
  ↓
Service port :80
  ↓
targetPort :8080
  ↓
Pod IP :8080
  ↓
Application listener :8080
```

Each value has a different purpose.

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

A Kubernetes manifest can be syntactically valid while traffic is
directed to a port on which the application does not listen.

------------------------------------------------------------------------

## 23. Named `targetPort`

A Service can reference a named Pod port rather than a numeric
`targetPort`.

Therefore the investigation must validate the effective mapping rather
than compare only numbers.

Conceptually:

``` text
Service targetPort: http
        ↓
Pod container port named: http
        ↓
Actual numeric port
        ↓
Application listener
```

A name match still does not prove that the process is listening.

------------------------------------------------------------------------

# EndpointSlices

## 24. EndpointSlice Is Primary Backend Evidence

Modern Kubernetes represents Service backends using EndpointSlices.

Conceptually:

``` text
Service
   ↓
EndpointSlice
   ↓
Endpoint
   ↓
Endpoint readiness
   ↓
Pod
```

Capture:

``` text
EndpointSlice:
Service association:
Endpoint addresses:
Endpoint readiness:
Endpoint target references:
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

ENDPOINT READY
≠
APPLICATION TRANSACTION HEALTHY
```

------------------------------------------------------------------------

## 25. Endpoint Readiness

For normal Pod-backed Services, endpoint readiness is closely related to
Pod readiness.

However, configuration such as `publishNotReadyAddresses` can alter
normal assumptions.

Therefore:

``` text
ENDPOINT PRESENT
≠
ENDPOINT READY
```

and:

``` text
NORMAL READINESS ASSUMPTIONS
MUST BE VERIFIED AGAINST SERVICE CONFIGURATION
```

Do not assume every endpoint list contains only ready application
backends.

------------------------------------------------------------------------

## 26. Endpoint Target Identity

Do not validate only the endpoint IP.

Where available, correlate:

``` text
Endpoint IP
   ↓
Target reference
   ↓
Pod
   ↓
Node
   ↓
Revision
   ↓
Application behavior
```

This becomes especially important during intermittent failures.

------------------------------------------------------------------------

# Replica-Level Failures

## 27. One Successful Request Does Not Prove All Backends Healthy

Consider:

``` text
Endpoint A → Pod A → healthy
Endpoint B → Pod B → healthy
Endpoint C → Pod C → failing
```

Clients may observe:

``` text
200
200
503
200
timeout
200
```

Rules:

``` text
ONE SUCCESSFUL REQUEST
≠
ALL BACKENDS HEALTHY

INTERMITTENT FAILURE
=
INVESTIGATE BACKEND DISTRIBUTION
```

Record, when available:

``` text
Endpoint:
Pod:
Node:
Revision:
Readiness:
Restart count:
Observed behavior:
```

Correlation is evidence.

It is not automatically causality.

------------------------------------------------------------------------

## 28. Mixed Revisions

During rollout or partial recovery, EndpointSlices may reference Pods
from different revisions.

Capture:

``` text
Affected endpoint revision:
Healthy endpoint revision:

Same:
YES / NO / UNKNOWN
```

Do not assume:

``` text
DIFFERENT REVISION
=
ROOT CAUSE
```

Instead classify the evidence:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

------------------------------------------------------------------------

# Application Listener

## 29. Pod Health Does Not Prove Listener Health

A Pod can be Running while the application is not listening on the
expected port.

``` text
POD RUNNING
≠
APPLICATION LISTENING
```

A Pod can also be Ready while a specific business operation is
unhealthy.

``` text
POD READY
≠
REQUESTED BUSINESS OPERATION HEALTHY
```

Endpoint readiness is useful routing evidence but is not complete
application validation.

``` text
ENDPOINT READY
≠
APPLICATION TRANSACTION HEALTHY
```

------------------------------------------------------------------------

## 30. Listener Evidence

Determine:

``` text
Expected application port:
Service targetPort:
Container port declaration:
Observed listener:
If safely validated
```

The important transition is:

``` text
Endpoint IP : targetPort
        ↓
Application listener
```

If this transition fails, upstream DNS and routing can still be
perfectly healthy.

------------------------------------------------------------------------

# HTTP and Application Responses

## 31. HTTP Status Codes Are Symptoms

Avoid:

``` text
502 = application problem
503 = Service problem
504 = network problem
404 = application route missing
```

Instead:

``` text
HTTP STATUS
=
SYMPTOM OBSERVED AT A LAYER
```

Record:

``` text
HTTP status:
Relevant response headers:
Responding component:
If identifiable

Observation point:
Timestamp:
```

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

# Internal vs External Paths

## 32. Internal Success and External Failure

Suppose:

``` text
Internal Service request = PASS
External endpoint request = FAIL
```

This supports moving investigation upstream.

Potential remaining layers include:

``` text
External DNS
Load balancer
TLS
Gateway
Host routing
Path routing
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

This is a narrowing rule, not root-cause proof.

Do not restart a healthy backend simply because an external transaction
fails.

------------------------------------------------------------------------

## 33. External Success and Internal Failure

The reverse is also possible.

An external gateway can reach the application while another in-cluster
consumer fails because of:

-   different DNS name;
-   namespace resolution;
-   different Service;
-   different port;
-   NetworkPolicy;
-   different protocol;
-   client-specific configuration.

Therefore:

``` text
EXTERNAL SUCCESS
≠
ALL INTERNAL CONSUMERS HEALTHY
```

------------------------------------------------------------------------

## 34. Test From the Correct Observation Point

A core rule is:

``` text
TEST FROM THE CORRECT OBSERVATION POINT
```

If the customer fails externally, an internal test is useful but cannot
replace external-path evidence.

If only an internal service-to-service call fails, a public endpoint
test does not validate that internal path.

------------------------------------------------------------------------

# HTTP Errors and Failure Domains

## 35. 502

A `502` indicates that some HTTP intermediary observed a bad upstream
response or equivalent implementation-specific condition.

It does not by itself prove:

-   application failure;
-   Service failure;
-   Pod failure;
-   network failure;
-   gateway failure.

Determine which component generated it and inspect the next transition.

------------------------------------------------------------------------

## 36. 503

A `503` can appear for many reasons depending on the responding
component.

Examples can include:

-   no usable backend;
-   application unavailable;
-   route unavailable;
-   gateway-specific condition;
-   maintenance state;
-   overload.

Therefore:

``` text
503
≠
AUTOMATICALLY SERVICE FAILURE
```

------------------------------------------------------------------------

## 37. 504 or Timeout

A timeout does not automatically prove a network failure.

Potential causes can include:

-   slow application;
-   unreachable backend;
-   connection timeout;
-   upstream timeout;
-   dependency latency;
-   gateway timeout;
-   policy;
-   packet loss;
-   routing failure.

Use:

``` text
TIMEOUT
≠
AUTOMATICALLY NETWORK FAILURE
```

Find the first proven transition that did not complete as expected.

------------------------------------------------------------------------

# Desired-State Ownership

## 38. Find the Authoritative Configuration Source

Observed Kubernetes state is not necessarily authoritative desired
state.

Possible owners include:

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
Observed object:
Authoritative configuration source:
Authoritative Configuration Owner:
Ownership confidence:
```

Rules:

``` text
OBSERVED KUBERNETES OBJECT
≠
AUTHORITATIVE CONFIGURATION SOURCE

Observed Drift
≠
Permission to Patch
```

------------------------------------------------------------------------

## 39. Reconciliation Ownership

If another system manages desired state, a manual Kubernetes change may:

-   be reverted;
-   create configuration drift;
-   mask the root cause;
-   produce inconsistent environments;
-   bypass review;
-   expand blast radius.

Therefore:

``` text
DO NOT PATCH THE SYMPTOM
WHEN ANOTHER SYSTEM OWNS DESIRED STATE
```

A temporary manual fix is not necessarily durable remediation.

``` text
MANUAL FIX WORKED
≠
AUTHORITATIVE CONFIGURATION FIXED
```

------------------------------------------------------------------------

# Shared Infrastructure

## 40. Networking Components Can Be Shared

A production route may depend on shared components such as:

``` text
Shared gateway
Shared ingress controller
Shared load balancer
Shared DNS zone
Shared wildcard certificate
Shared TLS Secret
Shared Service
Shared backend
```

Before recommending a change, identify consumers.

------------------------------------------------------------------------

## 41. Load-Balancer Blast Radius

A load-balancer change can affect more than one endpoint.

Use:

``` text
LOAD BALANCER CHANGE
=
SHARED INFRASTRUCTURE + DNS + AVAILABILITY EVENT
```

Do not modify shared ingress or load-balancer infrastructure as a
diagnostic experiment.

``` text
DO NOT MODIFY SHARED INGRESS INFRASTRUCTURE
AS AN INCIDENT DIAGNOSTIC TEST
```

------------------------------------------------------------------------

## 42. TLS Blast Radius

Certificate changes can affect:

-   multiple hostnames;
-   shared gateways;
-   wildcard domains;
-   shared Secrets;
-   load balancers;
-   certificate automation.

Therefore:

``` text
TLS CHANGE
=
SECURITY + ROUTING + AVAILABILITY EVENT
```

Certificate rotation requires ownership and blast-radius validation.

------------------------------------------------------------------------

## 43. Minimum Supported Blast Radius

Use the smallest scope supported by evidence.

Possible scopes include:

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

Record:

``` text
Minimum Supported Blast Radius:
Evidence:
```

------------------------------------------------------------------------

# Evidence Confidence

## 44. Evidence Classification

Use:

``` text
PROVEN
SUPPORTED
UNKNOWN
ASSUMED
```

Example:

``` text
External DNS resolves to LB-A             PROVEN
Expected production LB is LB-B            PROVEN
Gateway route points to Service-X         PROVEN
Service-X has three ready endpoints       PROVEN
Application is healthy on every endpoint  UNKNOWN
DNS mismatch caused the outage            SUPPORTED
```

Rule:

``` text
ASSUMED
≠
ROOT CAUSE
```

------------------------------------------------------------------------

# Failure-Domain Model

## 45. Lowest Proven Healthy Layer

Use the complete chain:

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

Do not mark a layer healthy because a downstream object merely exists.

------------------------------------------------------------------------

## 46. First Failed Transition

Record the first transition that evidence proves is unhealthy.

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
Ready EndpointSlice backends
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

Use:

``` text
First Failed Transition:
```

Do not replace this with:

``` text
Network issue
Ingress issue
Kubernetes issue
TrueFoundry issue
Application issue
```

unless evidence supports that exact boundary.

------------------------------------------------------------------------

# Current Actionable Owner

## 47. Ownership Follows the Failed Transition

Use:

``` text
First Failed Transition
+
Authoritative Configuration Owner
=
Current Actionable Owner
```

Possible ownership domains include:

``` text
Application team
TrueFoundry/platform team
Kubernetes/SRE
Network team
DNS team
Security/certificate team
Cloud infrastructure team
GitOps/IaC owner
External provider
```

Architecture does not define organizational ownership automatically.

Record:

``` text
Authoritative Configuration Owner:

Current Actionable Owner:

Requested Action:
```

------------------------------------------------------------------------

# Evidence Preservation

## 48. Preserve Evidence Before Remediation

Before changing DNS, certificates, Services, routes, gateway
configuration, Pods, or load balancers, preserve:

``` text
Original client error
Timestamp
Observation point
DNS result
Expected DNS target
TCP result
TLS result
Presented certificate metadata
HTTP status
Relevant response headers
Route configuration
Service configuration
EndpointSlices
Endpoint readiness
Endpoint-to-Pod mapping
Pod revision
Relevant events
Relevant controller logs
```

Rules:

``` text
RESTART RESTORED ROUTING
≠
ROOT CAUSE IDENTIFIED

ROUTE CHANGE RESTORED SERVICE
≠
AUTHORITATIVE CONFIGURATION FIXED

SERVICE RESTORED
≠
ROOT CAUSE REMEDIATED
```

------------------------------------------------------------------------

# Production Scenarios

## 49. Scenario A --- DNS Resolves to an Old Load Balancer

Observed:

``` text
Application hostname resolves successfully.
```

But:

``` text
Resolved target:
LB-OLD

Expected target:
LB-CURRENT
```

Classification:

``` text
DNS resolution:
HEALTHY

DNS target correctness:
FAILED
```

Possible First Failed Transition:

``` text
Hostname
→
Expected external entry point
```

Do not investigate Pods first.

------------------------------------------------------------------------

## 50. Scenario B --- DNS and TLS Work but Host/Path Route Is Wrong

Observed:

``` text
DNS:
PASS

TCP:
PASS

TLS:
PASS

HTTP:
404
```

The presented certificate matches the hostname.

Routing evidence shows the requested path does not match the intended
backend route.

Possible First Failed Transition:

``` text
Gateway
→
Expected host/path route
```

The backend application may never receive the request.

------------------------------------------------------------------------

## 51. Scenario C --- Service Has Zero Endpoints

Observed:

``` text
Service exists.
```

But:

``` text
Expected ready endpoints:
> 0

Observed:
0
```

Investigate:

``` text
Service selector
Pod labels
Pod readiness
EndpointSlice
publishNotReadyAddresses behavior
```

Possible First Failed Transition:

``` text
Service selector/readiness
→
Usable Service backend
```

Do not conclude that the Service object itself is broken.

------------------------------------------------------------------------

## 52. Scenario D --- Service Selects Unintended Pods

Observed:

``` text
Service has endpoints.
```

But the target references show they belong to the wrong workload.

Rule:

``` text
NON-ZERO ENDPOINTS
≠
CORRECT ENDPOINTS
```

Possible First Failed Transition:

``` text
Service selector
→
Expected workload
```

------------------------------------------------------------------------

## 53. Scenario E --- `targetPort` Does Not Match Application Listener

Observed:

``` text
Service:
port 80

targetPort:
8080

Application actually listens:
9090
```

Possible First Failed Transition:

``` text
Endpoint : targetPort
→
Application listener
```

DNS, TLS, gateway routing, and Service selection may all be healthy.

------------------------------------------------------------------------

## 54. Scenario F --- One Replica Causes Intermittent Failure

Observed:

``` text
Pod A → healthy
Pod B → healthy
Pod C → failing
```

Client behavior:

``` text
200
200
503
200
timeout
```

Investigate:

``` text
Endpoint IP
Pod
Node
Revision
Readiness
Restart count
Application behavior
```

Minimum Supported Blast Radius may be:

``` text
One endpoint / one Pod
```

Do not classify the entire Service as failed without evidence.

------------------------------------------------------------------------

## 55. Scenario G --- Internal Service Works, External Endpoint Fails

Observed:

``` text
Internal Service path:
PASS

External endpoint:
FAIL
```

Move investigation upstream:

``` text
External DNS
Load balancer
TLS
Gateway
Host/path route
External network
```

Rule:

``` text
INTERNAL BACKEND SUCCESS
+
EXTERNAL FAILURE
=
MOVE INVESTIGATION UPSTREAM
```

This narrows the investigation but does not itself identify root cause.

------------------------------------------------------------------------

## 56. Scenario H --- External Endpoint Works, Internal Consumer Fails

Observed:

``` text
External:
PASS

Internal consumer:
FAIL
```

Investigate the internal path:

``` text
Kubernetes DNS
Namespace resolution
Service name
Service port
NetworkPolicy
Client configuration
Protocol
```

Do not use the successful external path as proof that internal service
discovery is healthy.

------------------------------------------------------------------------

## 57. Scenario I --- Wrong Certificate Presented

Observed:

``` text
DNS:
Expected

TCP:
PASS

TLS endpoint:
Reachable

Certificate:
Wrong hostname
```

A valid certificate may exist in the platform, but the client receives a
different certificate.

Possible First Failed Transition:

``` text
SNI / TLS listener
→
Expected certificate
```

Investigate routing and certificate binding before replacing
certificates.

------------------------------------------------------------------------

## 58. Scenario J --- Certificate/SNI Hostname Mismatch

Observed:

``` text
Certificate valid:
YES

Trusted:
YES

Requested hostname covered by SAN:
NO
```

Possible First Failed Transition:

``` text
Requested hostname
→
Certificate identity
```

A non-expired and trusted certificate can still be invalid for the
requested hostname.

------------------------------------------------------------------------

## 59. Scenario K --- Gateway Healthy but Route Reconciliation Fails

Observed:

``` text
Gateway/controller Pods:
Running

Desired route:
Present

Observed route:
Not programmed / not accepted
```

Rule:

``` text
CONTROLLER RUNNING
≠
ROUTE RECONCILED
```

Investigate controller status, route conditions, events, and
authoritative desired state.

Do not restart the controller as the first diagnostic action.

------------------------------------------------------------------------

## 60. Scenario L --- Shared Gateway Change Expands Blast Radius

Observed:

``` text
One application endpoint fails.
```

A proposed change modifies a gateway or load balancer used by many
applications.

Before any remediation determine:

``` text
Shared consumers:
DNS dependencies:
Certificate dependencies:
Environment scope:
Rollback path:
```

Rule:

``` text
ONE APPLICATION INCIDENT
≠
PERMISSION TO CHANGE SHARED INFRASTRUCTURE
```

------------------------------------------------------------------------

# Production Investigation Workflow

## 61. End-to-End Workflow

Use:

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
REQUEST APPROVED REMEDIATION
        ↓
REGRESSION VALIDATION
```

------------------------------------------------------------------------

# Production Networking Evidence Handoff Contract

## 62. Evidence Template

Use:

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

# Production Safety

## 63. Investigation Is Not Remediation

Production troubleshooting should not mutate networking resources merely
to test a hypothesis.

Do not use incident investigation as justification to:

-   patch a Service;
-   edit selectors;
-   change `targetPort`;
-   modify Ingress;
-   modify Gateway resources;
-   modify Istio routing;
-   change DNS;
-   replace certificates;
-   decode certificate Secrets;
-   restart Pods;
-   restart ingress controllers;
-   restart gateways;
-   recreate load balancers;
-   delete EndpointSlices;
-   change shared infrastructure.

Use:

``` text
OBSERVE
        ↓
PROVE
        ↓
CLASSIFY
        ↓
IDENTIFY OWNER
        ↓
REQUEST APPROVED REMEDIATION
```

------------------------------------------------------------------------

## 64. Production Guardrails

Keep:

``` text
UNKNOWN TARGET
=
NO MUTATION

UNKNOWN ROUTING IMPLEMENTATION
=
NO ROUTE CHANGE

UNKNOWN TLS TERMINATION POINT
=
NO CERTIFICATE CHANGE

UNKNOWN AUTHORITATIVE OWNER
=
NO DIRECT CONFIGURATION CHANGE

Observed Drift
≠
Permission to Patch
```

------------------------------------------------------------------------

# Key Rules

## 65. DNS Rules

``` text
KUBERNETES SERVICE DNS
≠
EXTERNAL APPLICATION DNS

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

## 66. TLS Rules

``` text
CERTIFICATE EXISTS
≠
EXPECTED CERTIFICATE PRESENTED

CERTIFICATE SECRET CORRECT
≠
CERTIFICATE PRESENTED TO CLIENT

EXPECTED CERTIFICATE PRESENTED
≠
CERTIFICATE TRUSTED

CERTIFICATE TRUSTED
≠
HOSTNAME VALID

TLS HANDSHAKE SUCCESS
≠
HTTP ROUTING SUCCESS
```

------------------------------------------------------------------------

## 67. Routing Rules

``` text
INGRESS RESOURCE
≠
INGRESS CONTROLLER

INGRESS EXISTS
≠
CONTROLLER ACCEPTED CONFIGURATION

CONTROLLER RUNNING
≠
ROUTE RECONCILED

ROUTE RECONCILED
≠
LOAD BALANCER PROGRAMMED

GATEWAY REACHABLE
≠
REQUEST ROUTED CORRECTLY

ROUTE EXISTS
≠
ROUTE SELECTS EXPECTED SERVICE
```

------------------------------------------------------------------------

## 68. Service Rules

``` text
SERVICE EXISTS
≠
SERVICE HAS ENDPOINTS

SERVICE SELECTOR EXISTS
≠
SELECTOR MATCHES INTENDED PODS

SERVICE SELECTOR MATCHES
≠
TARGET PORT CORRECT

SERVICE PORT
≠
targetPort

targetPort
≠
PROOF APPLICATION LISTENER EXISTS
```

------------------------------------------------------------------------

## 69. Endpoint Rules

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

ENDPOINT READY
≠
APPLICATION TRANSACTION HEALTHY
```

------------------------------------------------------------------------

## 70. Application Rules

``` text
POD RUNNING
≠
APPLICATION LISTENING

POD READY
≠
REQUESTED BUSINESS OPERATION HEALTHY

HTTP 200
≠
COMPLETE BUSINESS TRANSACTION HEALTH
```

------------------------------------------------------------------------

## 71. Incident Rules

``` text
502 ≠ ROOT CAUSE

503 ≠ ROOT CAUSE

504 ≠ ROOT CAUSE

404 ≠ ROOT CAUSE

TIMEOUT
≠
AUTOMATICALLY NETWORK FAILURE

ASSUMED
≠
ROOT CAUSE

SYMPTOM
≠
FAILURE DOMAIN
≠
OWNER
```

------------------------------------------------------------------------

# Final Production Principle

A production endpoint is not one component.

It is a chain of independently configurable and independently failing
transitions.

The strongest investigation does not ask:

``` text
Is networking working?
```

It asks:

``` text
From the affected observation point:

What is the expected request path?

What transition has been proven healthy?

What is the next transition?

What evidence shows that transition succeeded or failed?

What is the smallest supported blast radius?

What system owns the authoritative configuration?

Who owns the next actionable step?
```

The operating principle remains:

``` text
DO NOT TROUBLESHOOT "THE NETWORK."

PROVE THE REQUEST PATH
ONE TRANSITION AT A TIME.
```
