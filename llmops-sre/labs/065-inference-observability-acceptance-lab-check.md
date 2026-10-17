# Chapter 065 — Executable Inference Observability Acceptance Lab

**Status:** Published; execution and review pending. **Scope:** synthetic local acceptance only.

## 1. Prerequisites
Python 3.11+; pytest for tests. No cloud credentials, GPUs, model weights or production endpoints required.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
Windows PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Synthetic acceptance engine
Save as `acceptance.py`:

```python
from decimal import Decimal
from dataclasses import dataclass
import json
import math

SERVICES = {"gateway", "router", "worker"}
ENVIRONMENTS = {"local", "staging"}
OUTCOMES = {"ok", "server_error", "partial_stream", "timeout"}
MODELS = {"synthetic", "llama", "mistral", "other"}

@dataclass(frozen=True)
class Journey:
    service: str
    environment: str
    outcome: str
    model_family: str
    ttft_ms: float
    queue_depth: int

def validate(j):
    if j.service not in SERVICES or j.environment not in ENVIRONMENTS:
        raise ValueError("unapproved service or environment")
    if j.outcome not in OUTCOMES:
        raise ValueError("unapproved outcome")
    if isinstance(j.ttft_ms, bool) or not isinstance(j.ttft_ms, (int, float)):
        raise ValueError("invalid ttft")
    if not math.isfinite(j.ttft_ms) or not 0 <= j.ttft_ms <= 3_600_000:
        raise ValueError("invalid ttft")
    if isinstance(j.queue_depth, bool) or not isinstance(j.queue_depth, int) or not 0 <= j.queue_depth <= 100000:
        raise ValueError("invalid queue depth")

def safe_record(j, raw=None):
    validate(j)
    # Deliberately ignore all raw prompt, response, header and token content.
    return {
        "service": j.service,
        "environment": j.environment,
        "outcome": j.outcome,
        "model_family": j.model_family if j.model_family in MODELS else "other",
        "ttft_ms": round(j.ttft_ms, 2),
        "queue_depth": j.queue_depth,
        "eligible": 1,
        "bad": int(j.outcome != "ok"),
    }

def metric_labels(record):
    return tuple(record[k] for k in ("service", "environment", "model_family"))

def error_budget(target, eligible, bad):
    target = Decimal(str(target))
    if not Decimal("0") < target < Decimal("1"):
        raise ValueError("invalid target")
    if type(eligible) is not int or type(bad) is not int:
        raise ValueError("counts must be integers")
    if eligible <= 0 or not 0 <= bad <= eligible:
        raise ValueError("invalid counters")
    allowance = Decimal(eligible) * (1 - target)
    return {
        "allowed": allowance,
        "consumed": Decimal(bad) / allowance,
        "remaining": 1 - Decimal(bad) / allowance,
        "burn": (Decimal(bad) / Decimal(eligible)) / (1 - target),
    }

def probe_result(status, complete, elapsed_ms, timeout_ms=2000):
    if status != 200:
        return {"ok": False, "reason": "http_error"}
    if not complete:
        return {"ok": False, "reason": "incomplete_stream"}
    if elapsed_ms > timeout_ms:
        return {"ok": False, "reason": "timeout"}
    return {"ok": True, "reason": "success"}

def validate_spans(spans):
    # Span identifiers are synthetic opaque IDs; no content attributes allowed.
    by_id = {s["id"]: s for s in spans}
    if len(by_id) != len(spans):
        return False
    if any(set(s) != {"id", "parent", "stage"} for s in spans):
        return False
    if any(s["stage"] not in {"gateway", "router", "worker"} for s in spans):
        return False
    roots = [s for s in spans if s["parent"] is None]
    if len(roots) != 1 or roots[0]["stage"] != "gateway":
        return False
    for s in spans:
        seen = {s["id"]}
        parent = s["parent"]
        while parent is not None:
            if parent not in by_id or parent in seen:
                return False
            seen.add(parent)
            parent = by_id[parent]["parent"]
    return True

if __name__ == "__main__":
    event = Journey("gateway", "local", "ok", "synthetic", 140.0, 0)
    print(json.dumps(safe_record(event, {"prompt": "SYNTHETIC_SECRET"}), sort_keys=True))
    print({k: str(v) for k, v in error_budget("0.999", 1000000, 1500).items()})
```

## 3. Automated acceptance tests
Save as `test_acceptance.py`:

```python
from decimal import Decimal
import json
import pytest
from acceptance import Journey, safe_record, metric_labels, error_budget, probe_result, validate_spans

BASE = Journey("gateway", "local", "ok", "synthetic", 140.0, 0)

def test_healthy_journey():
    assert safe_record(BASE)["bad"] == 0

def test_server_failure():
    assert safe_record(Journey("gateway", "local", "server_error", "synthetic", 140, 0))["bad"] == 1

def test_partial_stream():
    assert safe_record(Journey("gateway", "local", "partial_stream", "synthetic", 140, 0))["bad"] == 1

def test_ttft_observed():
    assert safe_record(Journey("gateway", "local", "ok", "synthetic", 2500, 0))["ttft_ms"] > 2000

def test_queue_pressure():
    assert safe_record(Journey("worker", "staging", "ok", "synthetic", 140, 30))["queue_depth"] > 20

def test_prompt_secret_exclusion():
    result = json.dumps(safe_record(BASE, {"prompt": "SYNTHETIC_SECRET", "authorization": "Bearer FAKE_KEY"}))
    assert "SYNTHETIC_SECRET" not in result and "FAKE_KEY" not in result

def test_bounded_labels():
    labels = {metric_labels(safe_record(Journey("gateway", "local", "ok", f"dynamic-{i}", 1, 0)))
              for i in range(1000)}
    assert len(labels) == 1 and next(iter(labels))[2] == "other"

def test_trace_parentage():
    assert validate_spans([
        {"id": "a", "parent": None, "stage": "gateway"},
        {"id": "b", "parent": "a", "stage": "router"},
        {"id": "c", "parent": "b", "stage": "worker"},
    ])

def test_trace_rejects_content():
    assert not validate_spans([{"id": "a", "parent": None, "stage": "gateway", "prompt": "SYNTHETIC_SECRET"}])

def test_probe_http_error():
    assert probe_result(503, False, 100)["reason"] == "http_error"

def test_probe_partial_stream():
    assert probe_result(200, False, 100)["reason"] == "incomplete_stream"

def test_probe_timeout():
    assert probe_result(200, True, 3000)["reason"] == "timeout"

def test_budget():
    result = error_budget("0.999", 1000000, 1500)
    assert result["allowed"] == Decimal("1000.000")
    assert result["consumed"] == Decimal("1.5")
    assert result["remaining"] == Decimal("-0.5")
    assert result["burn"] == Decimal("1.5")

@pytest.mark.parametrize("eligible,bad", [(0, 0), (100, 101), (100, -1), (True, 0)])
def test_invalid_budget(eligible, bad):
    with pytest.raises(ValueError):
        error_budget("0.999", eligible, bad)

def test_invalid_ttft():
    with pytest.raises(ValueError):
        safe_record(Journey("gateway", "local", "ok", "synthetic", float("nan"), 0))

def test_invalid_queue():
    with pytest.raises(ValueError):
        safe_record(Journey("gateway", "local", "ok", "synthetic", 100, -1))
```

Run:
```bash
python acceptance.py
python -m pytest -q
```
Expected: **19 passing pytest cases** (13 individual tests, 4 parametrized invalid-budget cases, and 2 invalid-field tests). If your pytest reports a different count, treat the actual collected count as authoritative and investigate.

## 4. Failure injection
1. **Leakage:** add `"prompt": raw.get("prompt")` to `safe_record` in a disposable copy; privacy test must fail.
2. **Cardinality:** export raw `model_family`; bounded-label test must fail.
3. **Partial stream:** classify `partial_stream` as success; test must fail.
4. **Missing telemetry:** simulate no events; reject denominator zero in budget calculation.
5. **Invalid trace:** create a parent cycle; `validate_spans` must return False.
6. **Probe timeout:** increase elapsed time beyond the deadline; probe must fail closed.

## 5. Production-like integration checklist
This local test suite is **not** a substitute for:
- deployed collector filtering and RBAC verification;
- actual Prometheus `promtool` rules validation from Chapters 062–063;
- Grafana dashboard provisioning and scrape freshness;
- independent-region probes and streaming transport validation;
- Alertmanager routing, grouping, inhibition and on-call acknowledgment;
- GPU exporter/queue metrics and end-to-end tracing across real inference;
- security review, load testing, rollback drill and change approval.

## 6. 100-point assessment worksheet
| Dimension | Max | Score |
|---|---:|---|
| SLI taxonomy and journey coverage | 15 | ___ |
| Metrics and histogram correctness | 15 | ___ |
| Logging and trace privacy | 15 | ___ |
| Burn-rate and error-budget logic | 15 | ___ |
| Synthetic probes | 10 | ___ |
| Dashboards and freshness | 10 | ___ |
| Incident response and on-call | 10 | ___ |
| Security, FinOps and evidence | 10 | ___ |
| **Total** | **100** | **___** |

**Pass recommendation:** >=90/100 with zero critical security/SLI gaps and reviewed production integration evidence. Do not mark accepted based on synthetic tests alone.

## 7. Evidence and cleanup
Record Python/pytest versions, sanitized command output, test counts, failed mutation tests, reviewer names, SLO taxonomy, alert routing, and unresolved risks. Stop any disposable processes and delete only lab-local artifacts after preserving approved evidence. No real prompts, customer tokens or production secrets are required.
