# Chapter 097 — Build a Secure Multi-Environment GPU Serving Platform

**Part 12 — Final Production AI SRE Capstone · Chapter 097 of 100**  
**Status:** Published with offline executable lab. Live infrastructure provisioning, GPU workload validation, security review and production acceptance pending.

## Learning objectives
Implement reproducible dev/staging/prod GPU inference environments with GitOps, workload identity, immutable model and container provenance, Kubernetes isolation, policy-as-code, network restrictions, secret management, capacity controls and promotion gates. Treat infrastructure and model release as a single audited change process.

## 1. Platform architecture
```text
                   Protected Git branches
                          |
               CI tests / policy checks
                          |
                IaC plan + approvals
                          |
              GitOps desired state
                 /        |        \
              dev       staging     prod
              |            |          |
          isolated IAM / namespaces / network policy
              |            |          |
            GPU node pools + device plugin
              |            |          |
          KServe / Ray Serve / TrueFoundry
              |            |          |
          signed image + model + tokenizer
              |            |          |
            AI gateway + quotas + mTLS
                          |
             traces / SLO / audit / cost
```
Isolation must be enforced by credentials, network and policy boundaries—not environment labels alone. In high-assurance deployments, use separate clusters/accounts/subscriptions and keys for production.

## 2. Repository structure
```text
gpu-platform/
  terraform/
    modules/{network,identity,cluster,gpu,registry,observability}/
    environments/{dev,staging,prod}/
  kubernetes/
    base/
    overlays/{dev,staging,prod}/
    policies/
  gitops/
    applications/{dev,staging,prod}/
  releases/
    manifests/
    signatures/
  tests/
    policy/
    security/
    smoke/
    load/
  runbooks/
    gpu-scheduling.md
    deployment-rollback.md
    access-revocation.md
```
Do not commit cloud credentials, Kubeconfigs, registry tokens, customer prompts or secrets.

## 3. Identity and tenant isolation
- Use cloud workload identity (AWS IRSA/EKS Pod Identity, Azure Workload Identity or equivalent) with narrowly scoped artifact access.
- Separate dev, staging and prod service principals, keys, KMS grants, namespaces and deployment approvals.
- Enable Kubernetes RBAC, Pod Security Admission and network policies backed by a supporting CNI.
- Default-deny ingress/egress, then explicitly allow DNS, gateway, telemetry and approved artifact endpoints.
- Enforce tenant authorization at gateway, retrieval and tool execution; namespaces alone are not a complete tenant security boundary.
- Pin runtime images by digest, verify model provenance and restrict who can update release manifests.

## 4. Kubernetes baseline — illustrative
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: llm-staging
  labels:
    pod-security.kubernetes.io/enforce: restricted
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: llm-staging
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-quota
  namespace: llm-staging
spec:
  hard:
    requests.nvidia.com/gpu: "4"
    limits.nvidia.com/gpu: "4"
```
The quota keys are illustrative and must be validated against the installed Kubernetes/device-plugin configuration; some environments enforce GPU quotas using `requests.nvidia.com/gpu` only. A default-deny policy blocks DNS, telemetry, model downloads and application traffic until explicit allow rules are installed. **Do not apply this fragment to a live namespace without a tested connectivity plan.**

## 5. GPU workload example — staging only
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-serving
  namespace: llm-staging
spec:
  replicas: 3
  selector:
    matchLabels:
      app: model-serving
  template:
    metadata:
      labels:
        app: model-serving
    spec:
      serviceAccountName: model-serving
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: model-serving
      containers:
        - name: inference
          image: registry.example.invalid/approved/serving@sha256:REPLACE_WITH_APPROVED_DIGEST
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "2"
              memory: 8Gi
              nvidia.com/gpu: "1"
            limits:
              memory: 8Gi
              nvidia.com/gpu: "1"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 10
```
This example is **not directly deployable** until you create the least-privilege service account, choose an approved digest and model runtime, verify the image runs as non-root and supports the GPU device plugin, and establish required network policies and secrets. GPU workloads can require runtime-specific security constraints; validate those rather than weakening the entire namespace.

## 6. GitOps promotion gates
1. Commit IaC and manifest changes to protected branches.
2. Run format, static checks, policy-as-code, dependency and image scans.
3. Generate and review a plan; require separate production approval.
4. Deploy dev and verify model download, signed artifacts, runtime identity and functional smoke tests.
5. Promote the **same immutable artifact digests** to staging; run load, safety, quality and tenant-isolation tests.
6. Validate N-1 GPU capacity, zone distribution, SLOs, cost budgets and rollback readiness.
7. Approve production canary; measure real user-impacting metrics.
8. Roll back on critical regression, record release provenance and reconcile drift.
9. Block promotion if policy checks, security gates or required evidence are missing.

## 7. Operational verification — read-only
```bash
kubectl config current-context
kubectl get namespaces --show-labels
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -n llm-staging -o wide
kubectl get serviceaccount,role,rolebinding,networkpolicy,resourcequota -n llm-staging
kubectl get events -n llm-staging --sort-by=.lastTimestamp
kubectl get deploy -n llm-staging -o yaml
```
For live GPU health, use approved node-level telemetry (DCGM/device-plugin metrics) and vendor-specific checks. Never assume `nvidia-smi` is present in a nonprivileged application container.

## 8. Troubleshooting
| Symptom | Diagnosis | Safe remediation |
|---|---|---|
| Pending GPU pod | insufficient allocatable GPUs, taints, affinity, quota | inspect scheduler and node pool |
| ImagePullBackOff | invalid digest or registry identity | verify immutable release and scoped credentials |
| Model download 403 | workload identity or bucket policy | repair least-privilege binding |
| DNS broken after policy | missing explicit egress allow | add tested DNS rule |
| Pod rejected | Pod Security Admission or policy | fix image/runtime compatibility |
| Readiness never succeeds | model warmup, port/path, artifact load | inspect logs and initialization |
| cross-env access | IAM/network trust misconfiguration | block promotion and security review |
| high standby cost | idle GPUs, fragmentation, excessive replicas | capacity/FinOps review |
| zone skew | topology constraints or GPU scarcity | validate eligible nodes and N-1 |

## 9. Runbooks
**RB-097-A — Environment bootstrap:** verify cloud account, identity, encryption, network/CNI, GPU quota, audit and GitOps ownership before workloads.

**RB-097-B — GPU scheduling failure:** inspect events, allocatable GPUs, device plugin, taints, topology, quota and driver compatibility; do not bypass security to schedule.

**RB-097-C — Registry/model access failure:** validate workload identity, signed artifact and least-privilege object access; avoid sharing static keys.

**RB-097-D — Security policy violation:** halt rollout, preserve audit, repair configuration, rerun isolation tests and require approval.

**RB-097-E — Promotion rollback:** pin previous approved image/model/prompt revisions, restore routing, verify quality/SLO and record RCA/COA.

## 10. Acceptance rubric — 100 points
| Domain | Points |
|---|---:|
| Environment isolation and identity | 20 |
| GPU Kubernetes runtime and topology | 20 |
| Provenance, secrets and network policy | 20 |
| GitOps promotion, rollback and tests | 20 |
| Observability, FinOps, runbooks and evidence | 20 |
| **Total** | **100** |

Educational pass >=90/100 with zero critical tenant isolation, unsigned model, secret exposure or unauthorized production promotion failures. A manifest linter cannot replace live policy enforcement tests.

## 11. Companion lab
[Chapter 097 — Executable Multi-Environment GPU Platform Acceptance Lab](../labs/097-secure-multi-environment-gpu-serving-platform-check.md) evaluates synthetic environment isolation, provenance, GPU capacity and promotion gates with pytest.

## References
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes Topology Spread](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [SLSA](https://slsa.dev/)

**Next:** Chapter 098 — Implement SLOs, Load Testing and Capacity Governance.
