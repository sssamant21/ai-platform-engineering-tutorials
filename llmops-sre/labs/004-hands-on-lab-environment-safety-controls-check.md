# Chapter 004 — Hands-On Lab: Environment and Safety Controls

**Part 01 | Chapter 004 | Execution pending**

## Scope

Validate an approved environment safely and produce an auditable lab execution plan. **Default is read-only**; no namespace, GPU workload, cloud infrastructure or model is created by these instructions.

## Lab 0 — Authorization record

```text
UTC window:
Owner / reviewer / on-call:
Cloud account and cluster (secure evidence reference):
Approved Kubernetes context:
Allowed namespace:
Allowed operations: SAFE-READ / LAB-MUTATE / GPU-LAB:
Approved model/image/digest:
Maximum hours and budget:
Abort and cleanup owner:
```

## Lab 1 — Tool and context baseline [SAFE-READ]

```bash
git --version
kubectl version --client
helm version --short
python3 --version
kubectl config current-context
kubectl cluster-info
kubectl auth can-i get pods --all-namespaces
```

Expected: tool versions and confirmed authorized context. Stop if identity, environment or permissions are unclear.

## Lab 2 — Kubernetes inventory [SAFE-READ]

```bash
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get namespaces
kubectl get deploy,sts,ds,pods -A -o wide
kubectl get svc,endpointslices -A
kubectl get resourcequota,limitrange -A
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

Expected: read-only topology, quotas, workload and event inventory. Redact output before committing any evidence.

## Lab 3 — GPU prerequisites [SAFE-READ]

```bash
kubectl describe nodes
kubectl get daemonsets -A
kubectl get resourcequota -A
```

Record `nvidia.com/gpu` allocatable, driver/device plugin evidence if available, GPU pool labels/taints, model/VRAM sizing, approved image and runtime compatibility. If no GPU exists, mark GPU tasks **blocked** rather than fabricating results.

## Lab 4 — Review isolation manifests [DESIGN ONLY]

Review the namespace and quota YAML in the chapter. Validate proposed Pod security, namespace-scoped RBAC, network policy support, artifact access, secrets and egress against your policies. Do not apply manifests without separately approved staging change.

## Lab 5 — Budget and cleanup design [DESIGN ONLY]

Fill:

```text
Estimated GPU-hours:
GPU SKU / hourly estimate source:
Maximum spend:
Cloud quota:
TTL and cleanup deadline:
Synthetic traffic request/token ceiling:
Explicit cleanup resource inventory:
Rollback/abort trigger:
```

Cloud cost estimates are not spending controls by themselves.

## Lab 6 — Six game days [TABLETOP]

| Scenario | Required evidence |
|---|---|
| Wrong context | stop/identity validation |
| RBAC denial | least-privilege escalation |
| GPU Pending | scheduler/taint/quota path |
| CUDA mismatch | compatibility matrix |
| Sensitive logs | redaction and security escalation |
| Budget overrun | stop, inventory, cost reconciliation |

## Sixteen failure checklist

F01 context; F02 SSO; F03 RBAC; F04 admission; F05 quota; F06 CPU Pending; F07 GPU Pending; F08 CUDA missing; F09 OOM; F10 image pull; F11 model 403; F12 DNS; F13 metrics; F14 sensitive logs; F15 GPU spend; F16 leftover cloud resources.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Authorization and named owner | Pending |
| G02 | Verified context | Pending |
| G03 | Toolchain versions | Pending |
| G04 | Scoped RBAC | Pending |
| G05 | Admission/network plan | Pending |
| G06 | Synthetic-only data policy | Pending |
| G07 | GPU compatibility checklist | Pending |
| G08 | Quota/budget/TTL | Pending |
| G09 | Observability/privacy | Pending |
| G10 | Cleanup and rollback plan | Pending |
| G11 | Game-day tabletop | Pending |
| G12 | Independent reviewer | Pending |

## Exit and cleanup

For read-only execution, confirm no resources were created. For separately authorized staging mutations, verify explicit resource removal and external cloud cleanup. Record each gate as pending/passed/failed/blocked with evidence and reviewer.

**Next: Chapter 005 — Fundamentals Acceptance Lab and Incident Walkthrough.**
