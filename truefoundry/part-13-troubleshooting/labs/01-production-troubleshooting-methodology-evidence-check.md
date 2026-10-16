# Chapter 13.1 — Hands-on Lab: Troubleshooting Methodology and Evidence Collection

**Part 13 | Approved staging | Live execution pending**

## Goal and safety

Use a reproducible triage workflow to isolate synthetic TrueFoundry GPU inference faults without production disruption. [SAFE-READ] diagnostics; [LAB-MUTATE] disposable staging; [CHANGE-CONTROL] production. Use sanitized evidence only.

## Lab 0 — Baseline

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl get deployments -A
kubectl get resourcequota -A
```

Record approved namespace, model/image digest, TrueFoundry agent and vLLM versions, GPU/driver, synthetic workload, SLO, reviewer and change ticket.

## Lab 1 — Create the evidence record

Open a UTC timeline with first symptom, affected region/model, request eligibility, TTFT/TPOT, streaming completion, errors and recent changes. Label observations separately from hypotheses.

## Lab 2 — Five staged diagnostic drills

1. **Pod Pending:** use pre-existing or disposable staging failure; inspect quota, affinity, taints and Events.
2. **Model startup failure:** compare previous logs, probes, CUDA and model artifacts.
3. **Synthetic latency regression:** compare request token mix, queue, TTFT and GPU memory.
4. **Gateway or stream interruption:** inspect ingress path, EndpointSlice and drain timing.
5. **Missing metrics:** distinguish exporter/scrape failure from healthy zero.

Never inject a fault into a shared production service. Tabletop any expensive or unsafe scenario.

## Thirty scenarios

01 agent disconnected; 02 reconciliation drift; 03 Pod Pending; 04 GPU quota; 05 taint mismatch; 06 affinity mismatch; 07 device plugin; 08 Node NotReady; 09 image pull; 10 crash loop; 11 probes; 12 CUDA mismatch; 13 GPU XID/ECC; 14 CUDA OOM; 15 OOMKilled; 16 KV cache; 17 TTFT; 18 TPOT; 19 queue; 20 retries; 21 gateway 5xx; 22 TLS; 23 DNS; 24 stream timeout; 25 endpoint readiness; 26 model artifact IAM; 27 metrics scrape; 28 alert route; 29 autoscaler; 30 rollback.

For each scenario: record expected/actual, UTC evidence, scope, suspected layer, tested hypothesis, safe mitigation, rollback and reviewer. Label tabletop vs executed.

## Twelve acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Authorized scope and budget | Pending |
| G2 | Customer-impact SLI recorded | Pending |
| G3 | UTC timeline established | Pending |
| G4 | Recent changes correlated | Pending |
| G5 | Kubernetes Events and workload evidence | Pending |
| G6 | TrueFoundry agent/platform evidence | Pending |
| G7 | GPU/runtime evidence | Pending |
| G8 | Inference SLO and synthetic evidence | Pending |
| G9 | Privacy and evidence redaction | Pending |
| G10 | Safe mitigation and rollback | Pending |
| G11 | Verified recovery | Pending |
| G12 | Incident report and reviewer sign-off | Pending |

## Evidence template

```text
Incident / exercise ID:
Approval / namespace / cluster:
Start UTC / end UTC:
TrueFoundry, vLLM, Kubernetes, GPU/driver versions:
Model/image digests:
Impact and SLO:
Recent changes:
Observations:
Hypotheses:
Tests and expected/actual:
Sanitized Events/logs/metrics:
Mitigation and rollback:
Recovery validation:
Executed or tabletop:
Open risks / owner / reviewer:
```

## Cleanup

Remove only disposable staging resources, restore baseline, close test alerts and confirm no unexpected GPU charges. Do not mark acceptance complete without executed evidence and sign-off.

**Next:** Chapter 13.2 — TrueFoundry Agent and Control Plane Troubleshooting.
