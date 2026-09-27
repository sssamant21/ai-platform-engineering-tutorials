# TrueFoundry Part 2 — Kubernetes Integration

## Purpose

Part 2 moves from the TrueFoundry architecture and SRE foundations established in Part 1 into the Kubernetes integration layer.

The goal is to teach how TrueFoundry integrates with a customer Kubernetes environment, how platform intent maps to Kubernetes resources, and how an SRE validates and troubleshoots that integration safely in production.

## Audience

This part is intended for:

- Platform Engineers
- Site Reliability Engineers (SREs)
- DevOps Engineers
- Kubernetes Administrators
- AI/ML Platform Engineers
- Infrastructure Engineers supporting TrueFoundry

## Prerequisites

Readers should understand the concepts established in Part 1, especially:

- TrueFoundry Control Plane vs Compute Plane
- `tfy-agent` as a cluster-integration component
- TrueFoundry Workspace vs Kubernetes namespace
- management path vs runtime path
- authoritative, desired, and observed state
- Lowest Proven Healthy Layer
- First Failed Transition
- failure domains and Current Actionable Owner
- `UNKNOWN TARGET = NO MUTATION`

Basic Kubernetes familiarity is expected.

## Part 2 Learning Path

| # | Tutorial | Production Focus |
|---|---|---|
| 2.1 | Connecting a Kubernetes Cluster to TrueFoundry | Compute Plane onboarding, prerequisites, integration, and validation |
| 2.2 | Understanding `tfy-agent` Deployment | Agent architecture, lifecycle, connectivity, health, and troubleshooting |
| 2.3 | Namespaces & Workload Placement | Namespace strategy, placement, isolation, and workload correlation |
| 2.4 | ServiceAccounts, RBAC & Permissions | Kubernetes identities, authorization, least privilege, and failure analysis |
| 2.5 | ConfigMaps, Secrets & Runtime Configuration | Runtime configuration, secret-safe operations, ownership, and reconciliation |
| 2.6 | Services, Ingress, DNS & TLS | Service networking, endpoint paths, DNS, certificates, and network failure domains |
| 2.7 | Storage, PVCs & Persistent Workloads | StorageClasses, PV/PVC lifecycle, attachment, persistence, and storage failures |

## Repository Structure

```text
part-02-kubernetes-integration/
├── README.md
├── PART-02-LAYOUT.md
├── 01-connecting-kubernetes-cluster/
│   └── 01-connecting-kubernetes-cluster.md
├── 02-understanding-tfy-agent-deployment/
│   └── 02-understanding-tfy-agent-deployment.md
├── 03-namespaces-workload-placement/
│   └── 03-namespaces-workload-placement.md
├── 04-serviceaccounts-rbac-permissions/
│   └── 04-serviceaccounts-rbac-permissions.md
├── 05-configmaps-secrets-runtime-configuration/
│   └── 05-configmaps-secrets-runtime-configuration.md
├── 06-services-ingress-dns-tls/
│   └── 06-services-ingress-dns-tls.md
├── 07-storage-pvcs-persistent-workloads/
│   └── 07-storage-pvcs-persistent-workloads.md
└── labs/
    ├── 01-connecting-kubernetes-cluster-check.md
    ├── 02-understanding-tfy-agent-deployment-check.md
    ├── 03-namespaces-workload-placement-check.md
    ├── 04-serviceaccounts-rbac-permissions-check.md
    ├── 05-configmaps-secrets-runtime-configuration-check.md
    ├── 06-services-ingress-dns-tls-check.md
    └── 07-storage-pvcs-persistent-workloads-check.md
```

Tutorial directories are created as part of the Part 2 skeleton. Canonical tutorial and lab files are added only when their workflow reaches the appropriate stage.

## Review Workflow

Every tutorial follows the same stage-gated workflow:

```text
Draft
  ↓
Technical/Vendor Validation
  ↓
Production/SRE Review
  ↓
Revised Final
  ↓
Canonical
  ↓
[SAFE-READ] Hands-On Lab
  ↓
Repository Validation
```

Only one stage is advanced at a time.

Technical/vendor validation should use current official TrueFoundry documentation and relevant upstream Kubernetes documentation where required.

## Production/SRE Principles

Part 2 carries forward the operational rules established in Part 1:

```text
UNKNOWN TARGET = NO MUTATION
```

```text
Symptom Location ≠ Failure Location
```

```text
Error visible in TrueFoundry ≠ TrueFoundry root cause
```

```text
TrueFoundry Authorization ≠ Kubernetes RBAC ≠ Cloud IAM
```

```text
Workspace ≠ Kubernetes Namespace
```

Troubleshooting should establish:

```text
Evidence
  ↓
Lowest Proven Healthy Layer
  ↓
First Failed Transition
  ↓
Failure Domain
  ↓
Next Required Action
  ↓
Current Actionable Owner
```

## Hands-On Lab Policy

Production-oriented validation labs are marked:

```text
[SAFE-READ]
```

Unless a lab explicitly states otherwise, production validation uses observational commands such as:

```bash
kubectl config current-context
kubectl get
kubectl describe
kubectl logs
```

Read-only JSONPath queries may be used where needed.

Production validation labs must not instruct the reader to perform unapproved mutations such as:

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

Labs must not expose Kubernetes Secret values or other credentials.

Installation or configuration exercises that inherently require mutation must be clearly separated from `[SAFE-READ]` production validation and must identify the intended non-production or approved change context.

## Completion Criteria

Part 2 is complete only when all seven tutorials have completed:

- Draft
- Technical/Vendor Validation
- Production/SRE Review
- Revised Final
- Canonical
- Hands-On Lab
- Repository Validation

`truefoundry/STATUS-TRACKER.md` is updated once at the end of Part 2 after all seven tutorials and labs have passed repository validation.

## Outcome

After completing Part 2, the reader should be able to trace a TrueFoundry-managed workload through the Kubernetes integration layer, distinguish platform state from Kubernetes runtime state, collect safe production evidence, identify the first failed transition, and route remediation to the correct operational owner.
