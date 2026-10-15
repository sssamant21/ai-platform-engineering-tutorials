# Chapter 9.5 — Hands-on Lab: NetworkPolicy, Private Networking and Egress Security

**Part 09 | Staging-only lab | No production policy changes**

## Objectives and prerequisites

Validate CNI enforcement, ingress/egress isolation, cluster DNS, gateway-to-inference access, private connectivity, outbound dependencies, telemetry, negative tests and rollback. Prerequisites: approved isolated cluster or namespace, non-sensitive echo workload, test client Pods, test DNS name, gateway/observability namespaces, CNI flow visibility if available, cloud read access and approved change owner.

**[SAFE-READ]** observation; **[LAB-MUTATE]** only isolated disposable staging; **[CHANGE-CONTROL]** production changes require approval. If the environment is unavailable, record **not executed**; do not claim a pass.

## Lab 0 — Preflight and topology [SAFE-READ]

```bash
kubectl config current-context
kubectl get ns --show-labels
kubectl -n kube-system get ds,deploy | grep -Ei 'cilium|calico|aws-node|azure|antrea' || true
kubectl get networkpolicy -A
kubectl get svc,endpointslice -A
kubectl get pods -A -o wide
```

Record: cluster/context, CNI and policy enforcement status, Pod labels, actual DNS implementation, TrueFoundry agent path, ingress/gateway namespace, approved private endpoints, artifact registry and telemetry destination. Stop if CNI enforcement is unknown.

## Lab 1 — Baseline positive and negative flows [SAFE-READ]

Prepare a table of expected connections: authorized gateway→echo TCP 8080; echo→DNS UDP/TCP 53; echo→OTLP 4317/4318 if configured; echo→approved artifact endpoint; blocked untrusted namespace→echo; blocked echo→unapproved external host. Verify DNS, TCP connectivity, HTTP status and CNI verdicts from dedicated test Pods. Do not probe customer networks.

## Lab 2 — Apply explicit allows then default deny [LAB-MUTATE]

1. Create disposable namespace `tfy-net-lab` and labeled echo Pods using approved staging manifests.
2. Deploy a disposable test client in a separate namespace and verify baseline reachability.
3. Adapt the canonical tutorial's DNS, gateway and telemetry allow examples to **actual** labels/ports; inspect the rendered selectors.
4. Apply allow policies using GitOps or `kubectl apply -f` in the lab only.
5. Apply the canonical default-deny policy in the lab.
6. Retest each authorized flow and denied flow. Capture CNI drop evidence if supported.
7. Confirm that allowed return traffic behaves correctly and neither app startup nor telemetry regresses.

**Important:** Do not apply the tutorial's `gateway-system` or `k8s-app: kube-dns` example selectors without verifying they match the real Pods. NodeLocal DNSCache requires separate validation.

## Lab 3 — Verify selectors and union-of-allows [LAB-MUTATE]

Demonstrate that a policy selecting no Pods does not provide intended protection. In a disposable namespace, add a deliberately over-broad allow policy and observe that it can override the effective default-deny restriction through union-of-allows. Remove the test policy immediately; retain only the reviewed allow set. Record source and destination policy selections separately.

## Lab 4 — Private cloud networking [SAFE-READ]

Inspect private/public LB scheme, VPC/VNet route tables, private DNS, endpoint attachment, security groups/NSGs and egress NAT/firewall topology. Test from approved private and public vantage points. Do not change live subnets, routes, firewall policies or endpoint settings.

## Lab 5 — Artifact, inference and telemetry checks [SAFE-READ]

Test workload model download separately from node image pull. Validate inference readiness and streaming behavior, OTLP delivery, internal datastore connectivity and agent control-plane reachability. Collect redacted logs, traces, CNI flow verdicts and timestamps. Never send sensitive model inputs.

## Lab 6 — Staged rollback [LAB-MUTATE]

Revert only the lab policy changes to the last known-good state. Repeat positive and negative tests. Confirm no accidental shared resources or DNS records were removed. For a production rehearsal, document GitOps revert commit, owner, abort thresholds and approval requirements without executing it.

## Failure investigation matrix — 26 scenarios

| # | Scenario | Evidence and diagnostic focus |
|---|---|---|
| 01 | Default-deny blocks gateway | Destination ingress selection |
| 02 | Gateway egress blocked | Source egress selection |
| 03 | Namespace selector typo | Namespace labels |
| 04 | Pod selector matches zero Pods | Pod labels |
| 05 | Over-broad allow defeats segmentation | Union of allowed policies |
| 06 | UDP DNS blocked | Port 53 and CNI verdict |
| 07 | TCP DNS blocked | Truncated DNS/large responses |
| 08 | NodeLocal DNSCache bypasses DNS selector | Actual resolver IP/path |
| 09 | CoreDNS upstream unreachable | Resolver forwarding/cloud rules |
| 10 | Model file download denied | Pod egress and object storage |
| 11 | Image pull fails on node | Node runtime network, not Pod policy |
| 12 | OTLP export times out | Collector egress and ingress |
| 13 | Metrics scrape denied | Scraper source and workload ingress |
| 14 | Private endpoint DNS wrong | Zone link and split-horizon |
| 15 | Private endpoint route absent | VPC/VNet routes and endpoint |
| 16 | External API IP changes | Egress proxy/FQDN controls |
| 17 | Egress gateway unhealthy | Proxy/gateway health and NAT |
| 18 | SNAT exhaustion | NAT capacity/connection metrics |
| 19 | HostNetwork traffic bypass assumption | CNI and node enforcement |
| 20 | NetworkPolicy not enforced | CNI mode/version and negative test |
| 21 | Mesh sidecar port path differs | Proxy interception and mTLS |
| 22 | Public LB exposes private service | LB scheme, firewall and test |
| 23 | Cross-namespace unauthorized reachability | Policy selectors/other allows |
| 24 | TrueFoundry agent cannot reach control plane | Agent destination, DNS/TLS/proxy |
| 25 | Streaming connection reset | Connection tracking and gateway drain |
| 26 | GitOps reverts manual emergency change | Desired state and change ownership |

For each: capture expected vs actual, impact, root-cause hypothesis, safe remediation, rollback, and retest. Reproduce only where staging isolation and authorization permit.

## Production runbook tabletop

Rehearse RB-01 through RB-12 from the canonical tutorial, including incident commander, SRE/security owner, communication cadence, evidence preservation, restoration validation and follow-up actions.

## Ten acceptance gates

| Gate | Evidence required | Status |
|---|---|---|
| G1 | Correct staging context, approved scope and owner | Pending |
| G2 | CNI policy enforcement verified by negative test | Pending |
| G3 | Accurate source/destination flow inventory | Pending |
| G4 | Default deny plus least-privilege allows verified | Pending |
| G5 | DNS UDP/TCP and actual resolver topology validated | Pending |
| G6 | Gateway-to-backend inference and health verified | Pending |
| G7 | Artifact download, dependencies and telemetry work | Pending |
| G8 | Private ingress and approved egress verified from distinct vantage points | Pending |
| G9 | Unauthorized cross-namespace and Internet flows denied | Pending |
| G10 | Staging rollback, metrics and evidence reviewed | Pending |

**Pass criteria:** all applicable gates verified with timestamped evidence and owner sign-off. A blocked/not-executed gate is not a pass. Publication is not proof of lab completion.

## Evidence template

Record UTC timestamp, cluster/context, namespace, CNI/version, policy name, source/destination labels, protocol/port, expected result, actual result, policy verdict or cloud flow evidence, SLO signal, rollback result, exception, reviewer and ticket.

**Next:** Chapter 9.6 — Inference Streaming, Timeouts and Networking Troubleshooting.
