# Chapter 001 — Hands-on Lab: Introduction to LLMOps and AI Inference SRE

**Part 01 | Companion lab | Execution pending**

## Goal and boundaries

Trace the inference request path and capture a read-only reliability baseline for an approved Kubernetes GPU-serving environment. Identify model, gateway, runtime, GPU, telemetry and on-call owners. Produce a safe incident worksheet.

**[SAFE-READ]** authorized inspection only. **[LAB-MUTATE]** isolated staging synthetic probe if separately approved. **[CHANGE-CONTROL]** production changes, GPU allocation and failure injection. Do not paste credentials, PHI, prompts or private output into evidence.

## Lab 0 — Preflight

Record cloud, cluster/context, namespace, inference gateway, approved synthetic model and token budget, monitoring endpoint, cost ceiling, reviewer and UTC window. Confirm permissions and redact outputs.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
kubectl get deployments,pods -A -o wide
kubectl get svc,endpointslices -A
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

Expected: inventory of schedulable nodes, workloads, endpoints and recent events. No mutations.

## Lab 1 — Map the service

Document the synthetic client → TLS/auth gateway → Kubernetes Service/EndpointSlice → inference Pod → GPU runtime → model artifact and telemetry path. Mark ownership boundaries and expected control-plane dependencies. Do not assume a TrueFoundry agent is in the customer data path.

## Lab 2 — Runtime and GPU diagnosis

With approved namespace/deployment values:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Check image digest, Pod readiness, resource requests, GPU allocatable, device plugin, model revision and runtime errors. Redact logs. Use approved DCGM dashboard for GPU telemetry; do not run privileged diagnostic containers.

## Lab 3 — SLI and SLO worksheet

For a synthetic-only approved endpoint, define eligible requests, success (including complete SSE), error classes, TTFT, TPOT, E2E, sample window and telemetry freshness. Calculate a hypothetical 99.9% request SLO error budget from a hypothetical 10,000 eligible requests: 10 allowed failures. This is **not** a production SLO.

## Lab 4 — Incident tabletop

Choose one scenario: Pending GPU Pod, CUDA runtime mismatch, high TTFT, gateway 503 or truncated SSE. Record first customer symptom, read-only commands, probable fault domain, safe mitigation, escalation, rollback owner and recovery evidence. Do not actually trigger failure.

## Lab 5 — Six game days

01 rate-limit versus saturation; 02 GPU quota Pending; 03 CUDA mismatch; 04 TTFT regression; 05 SSE timeout; 06 monitoring blind spot. Mark each **tabletop**, **executed**, or **not attempted** and record review.

## Twelve failure checklist

F01 Pending; F02 CUDA missing; F03 OOM; F04 model ACL; F05 429; F06 503; F07 TTFT; F08 TPOT; F09 SSE; F10 quality; F11 metrics; F12 cost.

## Ten acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | LLMOps/SRE responsibility matrix | Pending |
| G02 | Inference request diagram | Pending |
| G03 | TTFT/TPOT/E2E definitions | Pending |
| G04 | Eligible SLI worksheet | Pending |
| G05 | Gateway/SSE fault classification | Pending |
| G06 | Kubernetes/GPU read-only inventory | Pending |
| G07 | CUDA/driver/model compatibility worksheet | Pending |
| G08 | Security and GPU cost controls | Pending |
| G09 | Incident triage tabletop | Pending |
| G10 | Reviewer acceptance | Pending |

## Evidence template

```text
Lab / UTC / approved context:
Cluster / namespace / synthetic model:
Request path and owner:
Image/model/tokenizer/runtime/GPU:
SLI eligibility, TTFT, TPOT, SSE:
Read-only findings and redactions:
Scenario, diagnosis and mitigation:
Rollback and customer-path recovery:
Security/FinOps controls:
Executed/tabletop, blockers, reviewer:
```

## Cleanup

No resources should be created by the read-only lab. Confirm synthetic requests (if approved) are stopped, retain sanitized evidence and leave gates pending until reviewed.

**Next:** Chapter 002 — Production LLM Service Architecture and SRE Responsibilities.
