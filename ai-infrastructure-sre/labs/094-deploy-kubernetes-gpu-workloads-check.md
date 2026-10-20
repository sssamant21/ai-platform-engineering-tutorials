# Lab 094 — Deploy Kubernetes GPU Workloads Acceptance Gate

**Chapter:** [094 — Deploy Kubernetes GPU Workloads](../part-12-enterprise-operations-and-final-capstones/094-deploy-kubernetes-gpu-workloads.md)

**Status:** NOT EXECUTED. Requires authorized staging Kubernetes cluster, GPU devices, approved images, queue policies and independent review.

## 1. Safety

Do not deploy billable GPU workloads, change live quotas, drain production nodes, alter GPU Operator configuration or run high-load inference without approved staging scope and cost controls. Replace every image placeholder with a reviewed immutable digest before applying. Keep tenant data and secrets out of logs.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | Status |
|---|---|---|---|
| E01 | Staging context and GPU inventory | node/allocatable output | Pending |
| E02 | Namespace security and quota | RBAC/PSA review | Pending |
| E03 | Device plugin health | operator events | Pending |
| E04 | Pinned CUDA Pod | manifest digest | Pending |
| E05 | Device and kernel test | CUDA correctness | Pending |
| E06 | Placement policy | node/taint results | Pending |
| E07 | Topology feasibility | GPU/NIC mapping | Pending |
| E08 | Inference Deployment | readiness/rollout | Pending |
| E09 | Serving load test | p99/TTFT/TPOT | Pending |
| E10 | Training Job | successful completion | Pending |
| E11 | Queue/gang design | policy and admission | Pending |
| E12 | Checkpoint resume | hashes and last step | Pending |
| E13 | Pending/quota fault | root-cause evidence | Pending |
| E14 | Canary rollback | previous version restored | Pending |
| E15 | GPU cost/queue metrics | usage ledger | Pending |
| E16 | Independent acceptance | rubric/COAs | Pending |

## 3. Read-only preflight

\`\`\`bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
kubectl get namespaces --show-labels
kubectl get resourcequota -A
kubectl get priorityclass
kubectl get jobs -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

If the device plugin is not installed or no compatible GPUs exist, mark the test **BLOCKED**, not PASS.

## 4. Manifest validation

Use the chapter's example Pod, Deployment and Job manifests only after substituting approved image digests, namespaces, service accounts, storage and node policy. For an authorized staging context:

\`\`\`bash
kubectl apply --dry-run=server -f gpu-smoke.yaml
kubectl apply --dry-run=server -f inference-staging.yaml
kubectl apply --dry-run=server -f gpu-training-staging.yaml
\`\`\`

Server dry-run is **not** a GPU runtime, admission-policy enforcement or workload SLO test. It validates the target API and admission path without persisting objects.

## 5. Deployment test sequence

In approved staging only:

\`\`\`bash
kubectl apply -f gpu-smoke.yaml
kubectl get pod gpu-smoke -n gpu-validation -o wide
kubectl logs -n gpu-validation gpu-smoke

kubectl apply -f inference-staging.yaml
kubectl rollout status deployment/inference-staging -n gpu-validation --timeout=10m
kubectl get pods -n gpu-validation -o wide

kubectl apply -f gpu-training-staging.yaml
kubectl get jobs -n gpu-validation
kubectl describe job gpu-training-staging -n gpu-validation
\`\`\`

The inference and training examples are templates: they require real images, application endpoints, training code, capacity and access controls. Record failures as failures; do not imply these commands were run.

## 6. Fault-injection matrix

| Test | Expected evidence | Abort if |
|---|---|---|
| GPU quota exceeded | Pending/admission reason | privileged bypass |
| Wrong GPU SKU | scheduling denied | workload placed incorrectly |
| Inference cold start | startup/readiness timings | protected p99 breach |
| Rolling update | SLO and ready replicas | insufficient GPU headroom |
| Training restart | committed checkpoint resumed | lost/invalid optimizer state |
| Queue contention | fair admission and priorities | protected workload starvation |

## 7. Evidence manifest

\`\`\`yaml
chapter: "094"
environment: "GPU_STAGING_REQUIRED"
publication: "PUBLISHED"
execution: "NOT_RUN"
execution_date_utc: null
cluster: null
namespace: "gpu-validation"
gpu_sku: null
gpu_inventory: []
device_plugin_validation: "NOT_RUN"
cuda_smoke: "NOT_RUN"
placement_test: "NOT_RUN"
inference_deployment: "NOT_RUN"
inference_slo_test: "NOT_RUN"
training_job: "NOT_RUN"
queue_admission_test: "NOT_RUN"
checkpoint_resume: "NOT_RUN"
quota_fault_test: "NOT_RUN"
rollback_test: "NOT_RUN"
cost_metrics: "NOT_RUN"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
corrective_actions: []
\`\`\`

## 8. Acceptance

Score **20 + 20 + 20 + 20 + 10 + 10 = 100**. Pass ≥85 only when all critical GPU scheduling, inference SLO, checkpoint integrity, tenant isolation and rollback gates are verified with independent approval.

**Next:** Chapter 095 — Implement Storage and Network Performance Tests.
