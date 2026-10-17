# Chapter 061 — Executable Prompt-Safe Logging and Cardinality Lab

**Status:** Published, not executed. **Safety:** local synthetic data only.

## 1. Environment
Python 3.11+ and pytest. Run in an isolated working directory.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
Windows PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Implement an allowlist-only event builder
Save as `safe_telemetry.py`:

```python
import json
import math
from datetime import datetime, timezone

EVENTS = frozenset({"inference_completed", "inference_failed"})
ENVIRONMENTS = frozenset({"local", "staging", "production"})
MODELS = frozenset({"synthetic", "llama", "mistral", "other"})
STATUS_CLASSES = frozenset({"2xx", "4xx", "5xx"})
SERVICES = frozenset({"llm-gateway", "inference-router"})

def enum(value, allowed, field):
    if not isinstance(value, str) or value not in allowed:
        raise ValueError(f"invalid {field}")
    return value

def count(value, field):
    if isinstance(value, bool) or not isinstance(value, int) or not 0 <= value <= 10_000_000:
        raise ValueError(f"invalid {field}")
    return value

def duration(value):
    if isinstance(value, bool) or not isinstance(value, (int, float)):
        raise ValueError("invalid duration")
    if not math.isfinite(value) or not 0 <= value <= 3_600_000:
        raise ValueError("invalid duration")
    return round(value, 2)

def safe_event(source):
    # Never serialize or merge the original request dictionary.
    if not isinstance(source, dict):
        raise ValueError("source must be dict")
    model = source.get("model_family")
    model = model if isinstance(model, str) and model in MODELS else "other"
    return {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "event": enum(source.get("event"), EVENTS, "event"),
        "service": enum(source.get("service"), SERVICES, "service"),
        "environment": enum(source.get("environment"), ENVIRONMENTS, "environment"),
        "model_family": model,
        "status_class": enum(source.get("status_class"), STATUS_CLASSES, "status_class"),
        "duration_ms": duration(source.get("duration_ms")),
        "input_tokens": count(source.get("input_tokens"), "input_tokens"),
        "output_tokens": count(source.get("output_tokens"), "output_tokens"),
    }

def emit(source):
    return json.dumps(safe_event(source), sort_keys=True, separators=(",", ":"))

def metric_labels(event):
    # A bounded, explicitly approved set; never include correlation IDs.
    return tuple(event[k] for k in ("service", "environment", "model_family", "status_class"))

if __name__ == "__main__":
    print(emit({
        "event": "inference_completed", "service": "llm-gateway",
        "environment": "local", "model_family": "synthetic",
        "status_class": "2xx", "duration_ms": 123.4,
        "input_tokens": 30, "output_tokens": 12,
        "prompt": "SYNTHETIC_PRIVATE_PROMPT",
        "authorization": "Bearer SYNTHETIC_SECRET"
    }))
```

Run `python safe_telemetry.py`. Expected: a single JSON line with safe fields; neither synthetic marker appears.

## 3. Automated regression tests
Save as `test_safe_telemetry.py`:

```python
import json
import math
import pytest
from safe_telemetry import emit, safe_event, metric_labels

BASE = dict(
    event="inference_completed", service="llm-gateway",
    environment="local", model_family="synthetic", status_class="2xx",
    duration_ms=123.4, input_tokens=30, output_tokens=12
)

def test_secret_and_prompt_never_serialized():
    raw = {**BASE, "prompt": "SYNTHETIC_PRIVATE_PROMPT",
           "completion": "SYNTHETIC_PRIVATE_COMPLETION",
           "authorization": "Bearer SYNTHETIC_SECRET",
           "nested": {"cookie": "SYNTHETIC_COOKIE"}}
    out = emit(raw)
    assert all(marker not in out for marker in (
        "SYNTHETIC_PRIVATE_PROMPT", "SYNTHETIC_PRIVATE_COMPLETION",
        "SYNTHETIC_SECRET", "SYNTHETIC_COOKIE"))
    assert "prompt" not in json.loads(out)

def test_unknown_model_bounded():
    event = safe_event({**BASE, "model_family": "user-generated-unique-model-123"})
    assert event["model_family"] == "other"

@pytest.mark.parametrize("key,value", [
    ("input_tokens", -1), ("output_tokens", 1.5),
    ("duration_ms", math.inf), ("duration_ms", float("nan")),
    ("duration_ms", -1), ("input_tokens", True),
    ("environment", "customer@example.com"),
    ("status_class", "503")
])
def test_bad_values_rejected(key, value):
    with pytest.raises(ValueError):
        safe_event({**BASE, key: value})

def test_bounded_metric_labels():
    labels = {metric_labels(safe_event({**BASE, "model_family": f"dynamic-{i}"}))
              for i in range(10000)}
    assert len(labels) == 1
    assert next(iter(labels))[2] == "other"
```

Run `python -m pytest -q`. Expected: **11 passing tests** (1 secret, 1 model, 8 invalid-value, 1 cardinality).

## 4. Failure drills
1. Add `"prompt": source.get("prompt")` to the returned event **in a disposable copy**. Run tests: privacy regression must fail; revert immediately.
2. Replace model catalog mapping with raw model input **in a disposable copy**. Cardinality test must fail; revert.
3. Pass `duration_ms=float("nan")`: builder must raise ValueError without printing source data.
4. Measure safe log bytes and estimate ingestion for 100, 1,000 and 10,000 events/sec with `len(emit(BASE).encode())`; note actual production compression/indexing overhead differs.

## 5. Operational verification
- Inspect emitted keys; confirm no request bodies, exception strings, cookies, JWTs or tenant identifiers.
- Confirm metric dimensions are finite and documented.
- Confirm the logger does not echo input on validation failure.
- Document sample event size, bounded-label estimate and retention policy.
- Review the collector and backend independently; this lab does **not** prove end-to-end collector sanitization.

## 6. Cleanup and evidence
Run `deactivate` and remove only disposable lab files after recording sanitized pytest output, Python version and approved acceptance notes. Never capture real customer content as evidence.

## 7. Acceptance
- [ ] Synthetic JSON emission verified
- [ ] All 11 pytest cases pass
- [ ] Intentional leakage and cardinality mutations cause tests to fail
- [ ] Privacy/security and retention controls reviewed
- [ ] Live execution evidence recorded separately from publication
