# Chapter 13.3 — Hands-on Lab: Kubernetes Scheduling, Pending Pods and GPU Capacity Failures

**Part 13 | Approved staging | Live execution pending**

## Goal and safety

Distinguish unscheduled GPU Pods, admission rejection, device-plugin issues and cloud autoscaling failures using synthetic staging workloads. [SAFE-READ] inspect; [LAB-MUTATE] disposable staging with approved spend cap; [CHANGE-CONTROL] production scheduling, quotas, node groups and GPU pools.

## Lab 0 — Baseline

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get resourcequota,limitrange -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Record cluster, namespace, GPU SKU, `nvidia.com/gpu` allocatable, node labels/taints, autoscaler type, quota, cloud region, model, TrueFoundry revision, SLO and approved budget.

## Lab 1 — Scheduling decision

```bash
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o jsonpath='{.spec.nodeName}{"\n"}'
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
kubectl describe node NODE_NAME
```

Classify: unscheduled Pending, scheduled but not initialized, or Pod creation denied by admission. Identify the exact scheduler reason and evidence timestamp.

## Lab 2 — GPU resource accounting

Compare allocatable GPU resources, requested GPU counts, MIG profile and device plugin status. Check CPU/memory constraints separately. Do not infer free GPU slots from low utilization.

## Lab 3 — Autoscaler/cloud path

Trace matching GPU pool, scale limits, provisioning event, cloud quota, AZ/SKU supply, node bootstrap and resource advertisement. Use cloud read-only commands appropriate to the actual provider; avoid assuming EKS/AKS/GKE.

## Lab 4 — Five game days

1. Synthetic Pod with unsatisfiable node selector.
2. Synthetic Pod with unmatched taint/toleration.
3. GPU quota/admission rejection in disposable namespace.
4. Autoscaler/cloud quota shortage tabletop or approved isolated test.
5. Zone loss/N-1 capacity and SLO tabletop.

Document UTC start/end, blast radius, expected/actual, Events, cost, mitigation, rollback and reviewer.

## Thirty-six scenarios

01 GPU insufficient; 02 CPU insufficient; 03 memory insufficient; 04 invalid GPU request/limit; 05 device plugin; 06 NVIDIA driver; 07 MIG mismatch; 08 GPU sharing; 09 taint; 10 selector; 11 node affinity; 12 Pod affinity; 13 anti-affinity; 14 topology spread; 15 PVC zone; 16 ResourceQuota; 17 LimitRange; 18 priority; 19 autoscaler max; 20 cloud quota; 21 SKU shortage; 22 AZ shortage; 23 subnet IP; 24 bootstrap; 25 Node NotReady; 26 kubelet registration; 27 autoscaler IAM; 28 wrong pool labels; 29 wrong pool taints; 30 slow provisioning; 31 model cold start; 32 PDB maintenance; 33 preemption; 34 Spot interruption; 35 N-1 shortfall; 36 budget cap.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Authorized scope and cost cap | Pending |
| G2 | Pod phase and node assignment classified | Pending |
| G3 | FailedScheduling/admission Events preserved | Pending |
| G4 | GPU allocatable vs requested reconciled | Pending |
| G5 | Taints/affinity/topology analyzed | Pending |
| G6 | Namespace quota/admission checked | Pending |
| G7 | Autoscaler decision traced | Pending |
| G8 | Cloud GPU quota/SKU/AZ checked | Pending |
| G9 | N-1 capacity and warm reserve verified | Pending |
| G10 | Mitigation and rollback safe | Pending |
| G11 | SLO and budget validated | Pending |
| G12 | Reviewer sign-off | Pending |

## Evidence template

```text
Exercise ID / change approval:
UTC start / end:
Cluster / namespace / TrueFoundry revision:
Pod phase / spec.nodeName / scheduler reason:
GPU node SKU / allocatable / requested:
Taints, selectors, affinity, topology:
ResourceQuota and admission:
Autoscaler / cloud quota / SKU / AZ:
Customer SLO proxy / N-1 headroom:
Expected / actual / hypothesis:
Mitigation / rollback / cost:
Executed or tabletop:
Owner / reviewer:
```

## Cleanup

Delete only approved disposable staging objects, verify normal scheduling, confirm GPU nodes and cost return to baseline, and close test alerts. Publication is not proof of lab execution.

**Next:** Chapter 13.4 — Deployment Failures, CrashLoopBackOff and ImagePullBackOff.
