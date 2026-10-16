# Chapter 009 — Hands-On Lab: Streaming, Backpressure and API Gateway Behavior

**Part 02 | Chapter 009 | Written lab published; execution pending**

## Goal

Design and validate a synthetic SSE customer journey, distinguish TTFB/TTFT, identify proxy buffering and partial streams, and create a safe gateway incident response plan.

## Safety

**[SAFE-READ]** approved gateway/Kubernetes telemetry. **[LAB-MUTATE]** authorized staging-only synthetic streams, bounded request/token budgets. **[CHANGE-CONTROL]** production gateway, timeout, deployment or rate-limit changes. No real prompts, PHI or credentials in evidence.

## Lab 0 — Preflight

Record UTC, approved account/context, gateway/runtime revisions, API streaming contract, model, synthetic endpoint, owner, token/traffic ceiling, deadline, spend budget and reviewer.

```bash
kubectl config current-context
kubectl get ingress,svc,endpointslices -A
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

If the environment uses Gateway API or a managed gateway, substitute verified resource types. Stop if unauthorized.

## Lab 1 — Streaming protocol worksheet

Document expected `Content-Type`, SSE field grammar, JSON event schema, first **generated token** signal, finish reason, final sentinel/terminal event (if required), and clean-close behavior. Confirm which details are vendor-specific.

## Lab 2 — Parser edge cases [DESIGN]

Create synthetic parser fixtures for:
1. Two events in one network chunk.
2. One event split across chunks.
3. Multiline `data:` fields.
4. UTF-8 character split across reads.
5. Keepalive comment before first token.
6. HTTP 200 with missing completion marker.
7. Valid terminal event and clean close.
8. Server error after headers.

Classify complete vs incomplete; never count heartbeat as first output token.

## Lab 3 — Timing worksheet [DESIGN]

Hypothetical timeline:

```text
t=0 ms request sent
t=100 ms first HTTP byte
t=750 ms first generated token
t=900 ms second generated token
t=2300 ms final valid completion
TTFB = 100 ms
TTFT = 750 ms
first observed ITL = 150 ms
E2E = 2300 ms
```

If a proxy buffers the first token until 1,200 ms, client TTFT becomes 1,200 ms even when runtime first-token generation remains at 750 ms.

## Lab 4 — Gateway/runtime observations [SAFE-READ]

```bash
kubectl -n GATEWAY_NAMESPACE describe deployment GATEWAY_DEPLOYMENT
kubectl -n GATEWAY_NAMESPACE logs deployment/GATEWAY_DEPLOYMENT --since=15m --tail=100
```

Compare gateway status, upstream timings, client TTFT, complete-stream rate, runtime queue, active streams and cancellation. Redact logs and use approved dashboards.

## Lab 5 — Timeout and backpressure design

Build a table of client overall deadline, gateway request/idle timeout, upstream connect timeout, runtime queue/generation timeout, shutdown drain, slow-consumer buffer limit and cancellation policy. Include ownership, version-specific settings, abort conditions and rollback. Do not apply configuration changes without approval.

## Lab 6 — Eight game days

01 proxy buffering; 02 incomplete 200 stream; 03 slow client; 04 cancellation leak; 05 idle timeout; 06 rollout drain; 07 retry storm; 08 green HTTP metrics/failed SSE. Default **tabletop**.

## Twenty-scenario checklist

F01 DNS; F02 TLS; F03 auth; F04 429; F05 503; F06 504; F07 buffering; F08 compression; F09 heartbeat; F10 UTF-8; F11 SSE chunking; F12 terminal event; F13 truncated 200; F14 timeout; F15 slow consumer; F16 cancellation; F17 retry; F18 rollout; F19 privacy; F20 telemetry.

## Sixteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Streaming path diagram | Pending |
| G02 | API completion contract | Pending |
| G03 | SSE parser fixtures | Pending |
| G04 | TTFB/TTFT/ITL | Pending |
| G05 | Proxy buffering analysis | Pending |
| G06 | Backpressure controls | Pending |
| G07 | Cancellation propagation | Pending |
| G08 | Timeout hierarchy | Pending |
| G09 | Retry/idempotency policy | Pending |
| G10 | 429 classification | Pending |
| G11 | Privacy/auth | Pending |
| G12 | Synthetic completion probe | Pending |
| G13 | Telemetry freshness | Pending |
| G14 | Authorized diagnostics | Pending |
| G15 | Rollback/runbook | Pending |
| G16 | Reviewer signoff | Pending |

## Evidence and cleanup

```text
UTC/context/approval:
Gateway/runtime/API revision:
SSE protocol and completion:
TTFB/TTFT/ITL/E2E:
Proxy buffering/timeout/backpressure:
Cancellation and retries:
Synthetic tests executed/tabletop/blocked:
Security/privacy/cost:
Findings, owners, reviewer:
```

Read-only and tabletop exercises create no resources. Stop separately authorized staging synthetic traffic, reconcile cost and keep gates pending until reviewed.

**Next: Chapter 010 — Inference Internals Performance and Reliability Acceptance Project.**
