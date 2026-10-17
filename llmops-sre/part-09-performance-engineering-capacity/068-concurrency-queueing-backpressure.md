# Chapter 068 — Concurrency, Queueing and Backpressure

**Part 09 — Performance Engineering and Capacity · Chapter 068 of 100**  
**Publication status:** Published tutorial and companion lab. **Execution and technical/security reviews:** pending.

## Learning objectives
1. Distinguish offered load, admitted load, in-flight concurrency, queue length, throughput and goodput.
2. Apply Little's Law `L = λW` with its stability assumptions.
3. Identify queue saturation, head-of-line blocking, retry storms and coordinated omission.
4. Design bounded admission, deadline propagation, overload rejection and safe backpressure.
5. Run deterministic experiments and automated tests without production traffic.
6. Write incident runbooks for latency spikes and queue overload.

## 1. Request path and queueing boundaries
```text
external arrivals (offered RPS)
           |
     gateway quotas / deadline
           |
      admission controller ----> 429/503 + Retry-After policy
           |
       bounded queue  <------ queue depth / oldest age / wait p95
           |
       model scheduler <---- max active sequences / batch capacity
           |
      prefill + decode
           |
      streaming response
           |
       client completion
```

**Concurrency** is the number of in-flight requests, not requests per second. **Queue depth** is waiting work, not total concurrency. **Throughput** is successfully completed work per unit time. **Goodput** is completed work meeting explicit SLO thresholds per unit time. An overloaded service can report high GPU utilization while user goodput collapses.

## 2. Queueing theory for SRE
For a stable system over a representative interval, **Little's Law** states `L = λW`, where L is average work in system, λ is effective arrival/throughput rate, and W is average time in system. Example: 20 requests/sec × 0.5 sec = 10 requests in flight on average. This does not imply a specific p95 latency and cannot safely predict an unstable, unbounded queue.

Utilization `ρ = λ/μ` in a simple single-server model, with service rate μ. As ρ approaches 1, queueing delay becomes highly sensitive to burstiness and service-time variance. LLM serving is more complex: continuous batching, variable prompt/output lengths, KV-cache constraints and prefill/decode interference make service rate workload-dependent.

### Workload dimensions
| Variable | Example | Failure mechanism |
|---|---|---|
| Offered arrival rate | 10–100 RPS | admission overload |
| Prompt length | 128–16,384 tokens | prefill and KV demand |
| Output length | 32–1,024 tokens | decode occupancy |
| In-flight requests | 8–256 | scheduler/KV saturation |
| Burst size | 10–1,000 | transient queue spike |
| Client retry policy | jittered bounded retry | synchronized retry amplification |
| Request deadline | 2–60 sec | abandoned work and tail latency |

All numbers are illustrative; choose service-specific budgets and respect actual model context limits.

## 3. Backpressure patterns
- **Bounded queues:** enforce a maximum waiting count and maximum queue age; do not allow unbounded memory growth.
- **Admission control:** reject early when the service cannot meet a deadline, preferably before expensive prefill.
- **Deadline propagation:** client, gateway, scheduler and worker should share a consistent remaining-time budget; stop abandoned work.
- **Concurrency limits:** cap active sequences and in-flight tokens; request count alone ignores variable KV-cache demand.
- **Fair scheduling:** prevent long-context requests or noisy tenants from starving short interactive traffic; do not put tenant IDs into metrics labels.
- **Overload signaling:** distinguish policy quota `429` from temporary capacity `503`; document whether either counts as a bad SLI event. Use `Retry-After` only when a meaningful retry delay can be estimated.
- **Retry discipline:** exponential backoff with jitter, bounded attempts, per-request deadlines and circuit breakers. Never retry non-idempotent operations without an approved semantic contract.

## 4. Observability and SLOs
Required metrics:
- `llm_inference_offered_requests_total`, `llm_inference_admitted_requests_total`, `llm_inference_rejected_requests_total{reason}`;
- `llm_inference_queue_depth`, `llm_inference_queue_wait_seconds_bucket`, `llm_inference_oldest_queue_age_seconds`;
- `llm_inference_active_sequences`, `llm_inference_inflight_tokens`, `llm_inference_request_deadline_exceeded_total`;
- user-facing TTFT, TPOT, stream completion, error ratio, goodput and burn-rate alerts.

These are **proposed metric names**, not exporter guarantees. Labels must be bounded: service, environment, region and controlled reason codes. Monitor scrape freshness separately. Queue-depth dashboards without rejection rate and oldest queue age can miss overload.

## 5. Load-test methodology
1. Pin model, runtime, tokenizer, hardware, autoscaler and traffic cohort distribution.
2. Define open-loop arrivals to avoid coordinated omission; retain attempted arrivals even if rejected.
3. Warm up, sweep load, repeat steady-state intervals, then run bounded burst tests.
4. Compare **unbounded hypothetical queue**, bounded queue with rejection, and deadline-aware admission.
5. Measure offered/admitted/completed/rejected counts, queue p95, E2E p95, goodput and recovery time.
6. Establish stop conditions for error budget, latency, queue age, memory and benchmark spend.
7. Run only in approved staging; synthetic lab is a conceptual queue simulator.

## 6. Failure scenarios and troubleshooting
| Symptom | Hypotheses | Checks |
|---|---|---|
| TTFT spikes, TPOT stable | queue buildup or prefill contention | queue wait, oldest age, prompt cohort |
| GPU idle, queue grows | routing/admission bottleneck | worker assignment, dispatch lag |
| 429 surge | quota or concurrency gate | rejection reason, tenant fairness |
| 503 surge | capacity admission | offered load, service time, autoscaler |
| High tail latency after recovery | stale queue / abandoned requests | deadlines, cancellations, oldest age |
| Retry storm | synchronized clients | retry headers, attempts, jitter |
| Short requests starved | long-context head-of-line blocking | scheduler fairness, workload mix |
| OOM under modest RPS | high in-flight token footprint | KV cache, active sequences |

## 7. Production runbooks

### RB-068-A — Queue overload
**[SAFE-READ]** identify offered vs admitted load, queue depth/age, active sequences, TTFT, completion and error budget burn. Correlate traffic bursts and recent rollout. **[CHANGE-CONTROL]** activate approved admission limits, traffic shift or capacity changes. Confirm recovery by falling queue age and restored goodput.

### RB-068-B — Retry amplification
Inspect client attempt counts, `Retry-After`, backoff behavior, gateway 429/503 and downstream load. Coordinate client owners; avoid emergency blanket retries. Use reviewed jitter/backoff configuration and track user-visible impact.

### RB-068-C — Deadline and cancellation leak
Compare client disconnects to active worker sequences and wasted tokens. Ensure cancellation propagates through gateway/router/worker. Drain stale queue entries through approved operational controls; preserve sanitized incident evidence.

### RB-068-D — Capacity acceptance
Document throughput/goodput curves, p95 TTFT/E2E, queue-age bound, rejection behavior, failover margin, autoscaling lag and cost. Sign off only with real staging measurements and service-owner approval.

## 8. Security and acceptance
- **[SAFE-READ]** inspect telemetry; **[LAB-MUTATE]** local synthetic simulator; **[CHANGE-CONTROL]** production limits.
- No prompt text, response text, credentials or tenant IDs in metric labels.
- Protect against denial-of-service via bounded queues and request/token quotas.
- [ ] Offered, admitted, completed and rejected traffic accounted for.
- [ ] Queue is bounded; overload rejects instead of consuming unbounded memory.
- [ ] Deadline propagation and retry behavior documented.
- [ ] Automated tests cover overload, recovery and fairness assumptions.
- [ ] Runbooks, privacy and cost controls reviewed.
- [ ] Live execution and technical/security reviews independently recorded.

## References
- [Google SRE Workbook — Handling Overload](https://sre.google/workbook/handling-overload/)
- [Google SRE Book — Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [Prometheus histogram practices](https://prometheus.io/docs/practices/histograms/)
- [vLLM documentation](https://docs.vllm.ai/)

**Next:** Chapter 069 — Batching, KV Cache and Memory Tuning.
