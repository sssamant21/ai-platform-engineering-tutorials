# Chapter 061 — Prompt-Safe Logging, Redaction and Cardinality

**Part 08 — LLM Observability, Metrics and Tracing**  
**Status:** Published documentation and companion lab; technical/security review and live execution pending.

## Learning objectives
- Distinguish application logs, traces, metrics and audit records for LLM inference.
- Design an **allowlist-first** structured logging schema that excludes prompts, completions, tokens, API keys and tenant identifiers.
- Apply defense in depth: data minimization, deterministic sanitization, transport controls, collector filtering and retention.
- Control metric and log cardinality and estimate ingestion/storage cost.
- Implement and test a synthetic Python lab with leakage and cardinality regression gates.
- Operate runbooks for accidental disclosure, noisy labels and missing telemetry.

## 1. Production architecture

```text
client -> gateway -> model router -> serving backend (vLLM/Triton/Ray)
             |            |                |
       structured logs / trace spans / bounded metrics
             \____________|________________/
                          |
            local allowlist + value validation
                          |
          OTLP / log agent / collector pipeline
                          |
        filter/redact + RBAC + TLS + retention
                          |
          central observability backend
```

**Trust boundary:** prompts, model outputs, retrieved documents, tool arguments, headers and exception messages are potentially sensitive. An observability pipeline is not a safe place for raw content by default. Even a trace ID or opaque correlation ID can become identifying when joined to other datasets. Use random per-request IDs for short-lived correlation, enforce access restrictions, and avoid exporting tenant IDs.

### Recommended structured event contract
| Field | Example | Policy |
|---|---|---|
| `event` | `inference_completed` | fixed enumeration |
| `service` | `llm-gateway` | fixed deployment value |
| `environment` | `staging` | controlled enumeration |
| `model_family` | `synthetic` | bounded, mapped catalog |
| `status_class` | `2xx` | fixed enumeration |
| `duration_ms` | `247` | numeric, nonnegative, bounded |
| `input_tokens` / `output_tokens` | `120` / `64` | counts only, never token strings |
| `request_correlation` | random opaque ID | logs/traces only; short retention |
| `prompt`, `completion`, `authorization` | — | **never exported** |

Never log raw `request.body`, `response.body`, `messages`, `tool_calls`, `system_prompt`, `Authorization`, cookies or exception stringifications without independent review. Redacting with regex alone is insufficient for unknown PII, encoded secrets, or structured prompt content.

## 2. Privacy-safe instrumentation principles
1. **Minimize at source:** construct a new safe event from approved fields, rather than logging then removing forbidden keys.
2. **Validate values:** map dynamic strings into bounded enums; clamp/validate numeric ranges.
3. **Fail closed:** unknown keys are ignored, unknown model names map to `other`, malformed values fail validation.
4. **Keep content out of spans:** trace span names and attributes use route templates and controlled labels; no prompt snippets or user IDs.
5. **Use layered controls:** app allowlist, collector filtering, backend RBAC, encryption, retention and audit trails.
6. **Separate audit from telemetry:** access/decision audit events may require different schemas, retention and approval; do not blend into general application logs.
7. **Document exceptions:** any content capture requires explicit data-owner/security/privacy approval, legal basis, access controls and expiration; never silently enable in production.

## 3. Cardinality and operational costs
Cardinality is the number of distinct time series produced by combinations of metric labels. A metric with 5 environments × 12 model families × 4 status classes × 20 instances can produce **4,800** series before histogram buckets or additional dimensions. Adding per-request IDs makes series growth effectively unbounded. Histogram buckets multiply series count; native histograms have different storage characteristics.

**Never use:** prompt text, request IDs, session IDs, user IDs, tenant IDs, raw URL paths or free-form exception messages as metric labels. Instead use bounded route templates, outcome classes, deployment tier and a reviewed model catalog.

Estimate telemetry cost with:
```text
daily_ingest_GiB ≈ events_per_second × average_bytes_per_event × 86400 / 2^30
monthly_storage_GiB ≈ daily_ingest_GiB × retention_days × replication_factor
```
This is a rough logical-volume model; indexing, compression, sampling and backend overhead vary. Measure actual ingest and series counts in staging before rollout.

## 4. Executable lab
See [Chapter 061 companion lab](../labs/061-prompt-safe-logging-redaction-cardinality-check.md). The lab implements an allowlist-only JSON event builder, rejects invalid numeric fields, maps arbitrary model names to `other`, and tests that synthetic secrets and prompts cannot leak through unexpected input fields.

## 5. Operational SRE scenarios
| Symptom | Investigation | Safe response |
|---|---|---|
| Sensitive content in log index | identify source, field and exposure window | stop emission; invoke security/privacy incident process; preserve restricted evidence |
| Sudden metric series explosion | compare labels and active series before/after deploy | drop offending dynamic label; roll back instrumentation under change control |
| Logs disappear after policy update | check collector filtering, ingest errors and drop counters | restore approved schema, replay only safe synthetic data |
| Missing request correlation | inspect trace context, sampled spans and log correlation policy | use opaque log/trace correlation, never put IDs in metric labels |
| High observability bill | inspect ingest bytes, cardinality, retention and sampling | reduce volume, cap labels, shorten approved retention |
| Debug mode exposes body | review middleware, HTTP libraries and error handlers | disable body capture and add regression tests |

## 6. Production runbooks

### RB-061-A — Suspected prompt or secret leakage
**[SAFE-READ]** identify affected logger/exporter, time range, indices, viewers and data classes without reproducing sensitive text in tickets. **[CHANGE-CONTROL]** stop the emission at source, restrict access, involve incident commander/security/privacy, rotate exposed credentials where relevant, follow approved purge and evidence preservation policies, and verify with canary synthetic payloads. Record containment, remediation and prevention.

### RB-061-B — Cardinality explosion
Check active series and top label values, compare deployment revision, determine which label introduced unbounded values, remove or map it, verify query and scrape health, and observe ingestion after rollback. Do not indiscriminately delete historical metrics during an incident.

### RB-061-C — Collector saturation or missing logs
Check exporter/collector queue lengths, dropped/refused records, CPU, memory, OTLP connectivity, authentication, filtering and backend quotas. Keep inference request serving independent of logging availability. Restore with bounded retries and a validated safe schema.

### RB-061-D — Retention and access review
Validate least-privilege roles, encryption, region/residency, retention, deletion workflows, access audit and incident response ownership. Test retention using synthetic records.

## 7. Acceptance criteria
- [ ] No prompts, completions, bearer tokens or free-form user content emitted.
- [ ] Unknown keys ignored and invalid values rejected.
- [ ] Metrics use only bounded dimensions; no per-request labels.
- [ ] Synthetic leakage tests pass.
- [ ] Cardinality budget and retention documented.
- [ ] Collector/backend controls reviewed.
- [ ] Technical review, security review and live lab execution tracked independently.

## 8. References
- [OpenTelemetry Logs](https://opentelemetry.io/docs/specs/otel/logs/)
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [Prometheus instrumentation best practices](https://prometheus.io/docs/practices/instrumentation/)
- [Prometheus naming and labels](https://prometheus.io/docs/practices/naming/)

**Next:** Chapter 062 — Prometheus Recording Rules and Alert Design.
