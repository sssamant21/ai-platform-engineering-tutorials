# Chapter 082 — AI Gateway Rate Limits and Denial-of-Wallet Protection

**Part 10 — AI Security, Governance and FinOps · Chapter 082 of 100**  
**Status:** Published chapter and companion lab. Live staging execution, load testing, and independent technical/security review pending.

## Learning objectives
Design layered AI gateway admission that prevents request floods, token exhaustion, retry storms and **denial-of-wallet** (unexpected inference spend), while preserving fair tenant access and SLOs. Implement deterministic quotas and budgets, inspect failure modes and define rollback/incident runbooks.

## 1. Reference architecture
```text
client / agent
    |
identity validation + tenant binding
    |
API gateway: WAF / RPS / burst token bucket
    |
tenant policy: request / input / output token budgets
    |
cost admission: reserved worst-case tokens / currency ceiling
    |
concurrency semaphore + bounded queue + deadline
    |
model router -> GPU inference worker -> metered completion
    |
usage ledger (idempotent reconciliation)
    |
SRE metrics + budget alerts + audit
```
The gateway is not a substitute for authentication, authorization or workload isolation. Derive tenant identity from verified claims, not a client-controlled header. Enforce quotas at the **last shared choke point before GPU allocation** and at relevant upstream ingress layers.

## 2. Denial-of-wallet threat model
| Attack/failure | Cost or SLO impact | Mitigation |
|---|---|---|
| high-RPS request flood | queue saturation and request charges | per-principal/tenant token buckets |
| long-context prompts | expensive prefill and KV cache | input token ceilings, context validation |
| unbounded output | prolonged decode GPU use | max output tokens and timeouts |
| retry amplification | repeated billed inference | bounded retries, jitter, idempotency |
| many parallel streams | GPU contention and runaway spend | concurrency permits, fair queue |
| forged tenant identity | quota bypass | verified tenant membership |
| cost accounting lag | overspend beyond budget | pessimistic pre-admission reservation |
| multi-model routing | expensive model selected unexpectedly | policy-bound model price tier |
| distributed gateway race | double-admission | atomic shared counters or reservation service |
| abusive automated agents | recursive tool/inference loops | per-workflow budgets, max hops |

## 3. Layered limits and units
Define limits for:
- requests per second and burst capacity (token bucket);
- requests/minute and tokens/minute per tenant, principal and model;
- maximum input/output tokens per request;
- concurrent in-flight requests and bounded queue wait;
- daily/monthly budget in a documented currency and billing unit;
- agent workflow steps, retries and tool execution cost.

**Avoid double-counting:** request token estimates are admission inputs; actual billed usage must be reconciled from trusted runtime/provider metering. A completion that fails or is canceled may still consume resources. Reserve a conservative maximum amount before scheduling and release unused reservation after trusted reconciliation.

## 4. Cost and admission equations
For an illustrative model:
```text
estimated_cost =
  (input_tokens / 1_000_000) * input_price_per_million
  + (max_output_tokens / 1_000_000) * output_price_per_million

admit only if:
  authenticated AND authorized
  AND token/rate/concurrency quota permits
  AND available_budget >= estimated_cost
```
These prices are illustrative; real pricing may vary by provider, region, cache discount, priority and provisioned capacity. For self-hosted GPUs, account for GPU-seconds, amortization, idle cost, power and shared overhead rather than using API list price as a substitute.

## 5. Gateway implementation pattern
1. Verify caller and bind tenant membership.
2. Validate model allowlist, token counts and request body size before expensive work.
3. Apply per-identity rate limit and bounded burst.
4. Atomically reserve token/cost budget and concurrency slot; reject when exhausted.
5. Route to an allowed model tier with deadline and maximum output.
6. On completion, reconcile actual metered cost using an idempotent request ID.
7. On timeout/cancel, reconcile known usage or retain reservation pending bounded recovery.
8. Emit sanitized metrics and audit decisions without raw prompts or tokens.
9. Run canary load tests for retries, gateway restarts and multiple replicas.
10. Tune limits against tenant fairness, goodput, latency and error budgets.

## 6. HTTP behavior
| Outcome | Suggested status | Operational meaning |
|---|---|---|
| invalid credentials | 401 | authenticate again |
| valid identity lacks scope | 403 | policy denial |
| rate/token/concurrency limit | 429 | retry only after bounded guidance |
| budget exhausted | 429 or policy-specific 403 | stop automated retries; request budget approval |
| overload / backend unavailable | 503 | bounded retry/backoff when safe |
| malformed request / token size | 400 or 413 | correct client request |

Return `Retry-After` only when meaningful; avoid leaking another tenant's quota state. Ensure clients do not retry 4xx policy denials indefinitely.

## 7. Metrics, SLOs and alerts
Track admitted/rejected requests, 429/503 by reason, estimated vs actual cost, outstanding reservations, token throughput, queue wait, GPU utilization, per-tenant fairness, p95 latency and budget consumption rate. Use low-cardinality tenant classes in shared dashboards. Alerts should distinguish benign burst traffic from sustained abuse and should have documented false-positive review.

Example thresholds (illustrative): alert when budget consumption reaches 70%, 85% and 95%, or when projected spend exceeds approved budget before the period ends. Production thresholds require historical calibration.

## 8. Troubleshooting
| Symptom | Check | Safe response |
|---|---|---|
| legitimate requests receive 429 | bucket refill, concurrency, tokens, tenant policy | review entitlement, adjust approved limit |
| spending rises despite limits | retries, multi-gateway races, reconciliation lag | tighten atomic admission and reservations |
| GPU overloaded while RPS low | long context, output length, KV cache | enforce token and concurrency budgets |
| budget never releases | canceled jobs, missing finalizers, ledger errors | reconcile with trusted usage |
| tenants starve each other | global vs per-tenant queue | weighted fair scheduling |
| gateway outage bypasses policy | fallback route and routing rules | fail closed or approved degraded mode |
| cost mismatch | cached token billing, provider metering | investigate ledger and pricing model |

## 9. Production runbooks

### RB-082-A — Denial-of-wallet alert
Validate spend velocity, request origins, principal/tenant usage, token sizes and retry loops. Apply approved rate/token/cost caps, disable compromised identities, preserve sanitized evidence and notify FinOps/security owners.

### RB-082-B — Quota false-positive
Inspect policy version, recent workload shift and trusted usage. Grant temporary approved exception with expiry and audit; never disable all tenant limits.

### RB-082-C — Ledger/reservation outage
Fail closed on new high-cost admission or use an explicitly approved conservative emergency budget. Reconcile reservations before restoring normal limits; avoid double charging.

### RB-082-D — GPU overload and retry storm
Apply queue caps and concurrency shedding, stop recursive agent retries, honor client deadlines and restore capacity after validating SLOs and cost controls.

## 10. 100-point acceptance
| Domain | Points |
|---|---:|
| Identity-bound rate and token limits | 20 |
| Atomic cost reservations and reconciliation design | 20 |
| Concurrency, fairness and retry controls | 20 |
| FinOps metrics, alerting and incident response | 20 |
| Automated tests, staging plan and audit | 20 |
| **Total** | **100** |

Critical release blockers: unauthenticated GPU admission, cross-tenant budget charging, overspend from unbounded reservations, or a bypassable policy enforcement point. Educational target >=90/100 with no critical failures.

## 11. Companion lab
[Chapter 082 — Executable AI Gateway Rate Limits and Denial-of-Wallet Protection Lab](../labs/082-ai-gateway-rate-limits-denial-of-wallet-protection-check.md). It models a single-process deterministic budget ledger; it is **not** a distributed atomic reservation service.

## References
- [OWASP API Security](https://owasp.org/API-Security/)
- [OWASP LLM Top 10](https://genai.owasp.org/llm-top-10/)
- [IETF HTTP 429](https://www.rfc-editor.org/rfc/rfc6585.html)
- [OpenTelemetry Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)

**Next:** Chapter 083 — GPU FinOps, Budget Alerts and Rightsizing.
