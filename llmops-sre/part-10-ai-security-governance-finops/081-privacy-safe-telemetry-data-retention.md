# Chapter 081 — Privacy-Safe Telemetry and Data Retention

**Part 10 — AI Security, Governance and FinOps · Chapter 081 of 100**  
**Publication:** Chapter and executable companion lab. **Pending:** independent privacy/security review, live staging execution and production approval.

## Learning objectives
Design telemetry that supports LLM SRE observability without unnecessarily collecting prompts, responses, personal data, credentials or customer content. Apply data classification, minimization, pseudonymization, retention, deletion, access control and incident procedures across traces, logs, metrics and evaluation artifacts.

## 1. Reference architecture
```text
inference gateway / model runtime / tool broker
                    |
           telemetry instrumentation
                    |
    in-process allowlist + sensitive-field drop
                    |
         OpenTelemetry Collector
       processors: filter / transform / batch
                    |
       restricted transport + authentication
                    |
    metrics | sanitized traces | restricted audit
                    |
      access control + TTL + deletion workflow
```
**Privacy boundary:** Observability payloads are a new data processing surface. A request ID or trace ID can still become linkable to a person when joined with other systems. Treat pseudonymization as a risk reduction, not anonymization. Production policy must reflect organizational requirements and applicable laws; this tutorial is not legal advice.

## 2. Threat and data inventory
| Signal | Useful SRE fields | Avoid by default | Risk |
|---|---|---|---|
| Metrics | model revision, status class, token count, duration, region | prompt text, raw tenant IDs, user IDs | cardinality and re-identification |
| Traces | trace ID, span timing, sanitized operation, outcome | raw prompts, completions, auth headers | sensitive-content leakage |
| Logs | structured error code, safe correlation ID, component | request/response body, credentials, cookies | broad access and retention |
| Audit | principal reference, decision, action, policy version | secret value, model-generated raw text | privileged record exposure |
| Evaluations | synthetic dataset ID, aggregate score | production user content without approval | unintended training/data reuse |

Inventory upstream SDKs, reverse proxies, tracing exporters, queue dead-letter topics, dashboards, object stores, support tickets and third-party telemetry vendors; sensitive payloads may escape through any of them.

## 3. Data classification and minimization
Define public, internal, confidential and restricted classes with an accountable data owner. Permit only necessary attributes by **explicit allowlist**. Treat arbitrary exception strings, HTTP query parameters, baggage and tool arguments as potentially sensitive. Prefer counts and bucketed token sizes over raw values. Disable raw prompt/completion capture unless explicitly approved with a lawful purpose, retention period, access controls and deletion mechanism.

### Safe instrumentation pattern
```python
# Illustrative OpenTelemetry usage; install opentelemetry-api separately.
from opentelemetry import trace
tracer = trace.get_tracer("llm-inference")
with tracer.start_as_current_span("inference.request") as span:
    span.set_attribute("llm.model_revision", "approved-digest-prefix")
    span.set_attribute("llm.input_tokens", 512)
    span.set_attribute("llm.output_tokens", 128)
    span.set_attribute("http.response.status_code", 200)
    # Do not set prompt, completion, Authorization header or user identity.
```
A collector can enforce an additional denylist, but dropping sensitive fields **at the source** is preferable. Unknown nested fields, stack traces and exception messages can bypass simplistic redaction.

## 4. Retention, deletion and access
Define retention by data class and purpose, not one global number. Example **illustrative**, requiring policy approval:
| Store | Example retention | Access |
|---|---|---|
| Aggregate operational metrics | 30–90 days | SRE read-only |
| Sanitized distributed traces | 7–14 days | restricted SRE |
| Security audit events | per security/legal schedule | audited security role |
| Temporary synthetic evaluation artifacts | 7 days | evaluation team |
| Raw user content | not collected by default | exceptional approved workflow only |

TTL does not guarantee immediate deletion from backups, replicas, exports or immutable archives. Define deletion propagation, backup expiry, legal holds, deletion evidence and vendor contracts. A legal hold may override normal expiry only through documented authority.

## 5. Telemetry privacy controls
1. Review data flows and assign data owners and retention classes.
2. Configure SDK instrumentation to emit approved field names only.
3. Block request/response body, token, cookie, tool payload and untrusted exception attributes.
4. Apply collector-side filter/transform as defense-in-depth, including nested fields.
5. Restrict exporter endpoints, mTLS credentials, tenant boundaries and access roles.
6. Avoid high-cardinality raw user/tenant IDs in metric labels.
7. Configure storage TTL, backup lifecycle and deletion requests with audit evidence.
8. Use synthetic canary values in staging to verify no secret reaches exports.
9. Monitor privacy-control failures and vendor ingestion drift.
10. Review new telemetry attributes during each model/gateway deployment.

## 6. Testing matrix
| Scenario | Expected |
|---|---|
| Prompt included in incoming event | field dropped before export |
| Authorization header included | dropped, not masked partially |
| Exception contains synthetic canary | quarantined/dropped |
| Nested arbitrary payload | rejected unless schema-approved |
| Tenant identifier in metrics label | replaced by approved cohort label |
| Missing retention metadata | fail-closed export or quarantine |
| Authorized deletion request | primary and derivative stores covered |
| Vendor exporter disabled | no unexpected fallback export |

## 7. Troubleshooting
| Symptom | Diagnose | Containment |
|---|---|---|
| Synthetic canary in traces | SDK instrumentation, collector processors | stop export, purge approved scope |
| Prompt in error logs | middleware body logging, exception formatting | disable logger, restrict access |
| Cardinality explosion | user IDs in metric labels | remove labels, control series |
| Deleted record reappears | replica, archive, dashboard cache | propagate deletion and verify |
| Retention not enforced | TTL configuration, object lock, backup policy | correct lifecycle, escalate |
| Trace loss after filtering | overbroad processor rules | validate sanitized allowlist and sampling |

## 8. Production runbooks

### RB-081-A — Sensitive telemetry disclosure
Stop affected exporter, restrict access to evidence, identify data classes and exposure window, initiate privacy/security incident process, rotate exposed credentials and document authorized deletion/remediation. Never copy leaked content into a ticket.

### RB-081-B — Retention/deletion failure
Identify all copies and processors, verify holds and backups, repair lifecycle rules, execute approved deletion, retain only minimal proof and obtain privacy-owner sign-off.

### RB-081-C — Observability degradation after hardening
Compare safe schema and drop counters, ensure required latency/error/token metrics survive, restore only approved attributes and run synthetic canary checks before re-enabling export.

### RB-081-D — Third-party telemetry drift
Disable unauthorized vendor fields/routes, review processor changes, data processing agreements, retention and regional storage, then reapprove.

## 9. 100-point acceptance
| Domain | Points |
|---|---:|
| Data inventory, classification and purpose limitation | 20 |
| Source/collector minimization and export controls | 20 |
| Retention, deletion, backups and access controls | 20 |
| Synthetic privacy regression and observability quality | 20 |
| Incident runbooks, evidence and review | 20 |
| **Total** | **100** |

Any unauthorized raw prompt export, real secret exposure or cross-tenant access is a critical release blocker. Educational target >=90/100 with zero critical gate failures. Publication is not privacy/legal sign-off.

## 10. Companion lab
[Chapter 081 — Executable Privacy-Safe Telemetry and Data Retention Lab](../labs/081-privacy-safe-telemetry-data-retention-check.md). Offline Python policy engine tests allowlisted attributes, nested payload rejection, retention metadata and synthetic canary detection. It is not a production DLP or storage-deletion system.

## References
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [NIST SP 800-53](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)

**Next:** Chapter 082 — AI Gateway Rate Limits and Denial-of-Wallet Protection.
