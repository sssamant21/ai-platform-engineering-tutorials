# Chapter 081 — Executable Privacy-Safe Telemetry and Data Retention Lab

**Status:** Published; execution and independent privacy/security review pending. Uses synthetic telemetry only.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Offline telemetry export gate
Save as `privacy_gate.py`:
```python
from dataclasses import dataclass
from datetime import datetime, timezone, timedelta
import json
import math

ALLOWED = {
    "model_revision": str,
    "operation": str,
    "status_class": str,
    "input_tokens": int,
    "output_tokens": int,
    "duration_ms": float,
}
FORBIDDEN_MARKERS = ("demo_secret_", "synthetic_canary_")
RETENTION_DAYS = {"metric": 30, "trace": 14, "audit": 90}

@dataclass(frozen=True)
class Event:
    kind: str
    created_at: datetime
    attributes: dict

def export_decision(event, now):
    if event.kind not in RETENTION_DAYS:
        return {"allow": False, "reason": "unknown_kind"}
    if not isinstance(event.created_at, datetime) or event.created_at.tzinfo is None:
        return {"allow": False, "reason": "invalid_timestamp"}
    if not isinstance(now, datetime) or now.tzinfo is None:
        return {"allow": False, "reason": "invalid_now"}
    if event.created_at > now:
        return {"allow": False, "reason": "future_event"}
    if now - event.created_at > timedelta(days=RETENTION_DAYS[event.kind]):
        return {"allow": False, "reason": "retention_expired"}
    if not isinstance(event.attributes, dict):
        return {"allow": False, "reason": "invalid_attributes"}
    safe = {}
    for key, value in event.attributes.items():
        if key not in ALLOWED:
            return {"allow": False, "reason": "unapproved_attribute"}
        typ = ALLOWED[key]
        if type(value) is not typ:
            return {"allow": False, "reason": "invalid_attribute_type"}
        if isinstance(value, str) and any(marker in value.lower() for marker in FORBIDDEN_MARKERS):
            return {"allow": False, "reason": "synthetic_canary"}
        if isinstance(value, float) and not math.isfinite(value):
            return {"allow": False, "reason": "invalid_numeric"}
        if isinstance(value, int) and value < 0:
            return {"allow": False, "reason": "invalid_numeric"}
        if isinstance(value, float) and value < 0:
            return {"allow": False, "reason": "invalid_numeric"}
        safe[key] = value
    return {"allow": True, "reason": "safe_schema", "attributes": safe}

def sample(now=None):
    now = now or datetime(2026, 1, 1, tzinfo=timezone.utc)
    return Event("trace", now, {
        "model_revision": "approved-rev",
        "operation": "inference",
        "status_class": "2xx",
        "input_tokens": 512,
        "output_tokens": 128,
        "duration_ms": 120.5,
    }), now

if __name__ == "__main__":
    event, now = sample()
    print(json.dumps(export_decision(event, now), indent=2))
```

**Limitations:** This strict allowlist rejects unknown fields but does not discover sensitive values hidden in an allowed string, encoded data, distributed storage, third-party SDKs or backups. The retention decision does not actually delete stored records. Production telemetry must use approved collectors, storage lifecycle and privacy review.

## 3. Automated tests
Save as `test_privacy_gate.py`:
```python
from dataclasses import replace
from datetime import timedelta, datetime, timezone
import pytest
from privacy_gate import Event, export_decision, sample

def test_safe_event():
    e, now = sample()
    assert export_decision(e, now)["allow"]

def test_prompt_rejected():
    e, now = sample()
    assert export_decision(replace(e, attributes={**e.attributes, "prompt": "private"}), now)["reason"] == "unapproved_attribute"

def test_auth_header_rejected():
    e, now = sample()
    assert not export_decision(replace(e, attributes={"authorization": "Bearer synthetic"}), now)["allow"]

def test_nested_payload_rejected():
    e, now = sample()
    assert not export_decision(replace(e, attributes={"operation": {"raw": "payload"}}), now)["allow"]

def test_canary_rejected():
    e, now = sample()
    assert export_decision(replace(e, attributes={"operation": "synthetic_canary_123"}), now)["reason"] == "synthetic_canary"

def test_expired_trace():
    e, now = sample()
    assert export_decision(replace(e, created_at=now - timedelta(days=15)), now)["reason"] == "retention_expired"

def test_at_retention_boundary():
    e, now = sample()
    assert export_decision(replace(e, created_at=now - timedelta(days=14)), now)["allow"]

def test_future_timestamp():
    e, now = sample()
    assert not export_decision(replace(e, created_at=now + timedelta(seconds=1)), now)["allow"]

def test_naive_timestamp():
    e, now = sample()
    assert not export_decision(replace(e, created_at=datetime(2026, 1, 1)), now)["allow"]

def test_unknown_kind():
    e, now = sample()
    assert not export_decision(replace(e, kind="prompt"), now)["allow"]

@pytest.mark.parametrize("attrs", [
    {"input_tokens": -1},
    {"input_tokens": True},
    {"duration_ms": float("nan")},
    {"output_tokens": "128"},
    {"model_revision": ["rev"]},
])
def test_invalid_attribute_values(attrs):
    e, now = sample()
    assert not export_decision(replace(e, attributes=attrs), now)["allow"]
```

Run:
```bash
python privacy_gate.py
python -m pytest -q
```
Expected: **15 pytest cases** (10 standalone and 5 parameterized). Live execution has not been performed.

## 4. Failure drills
1. Add a `prompt`, `authorization` or arbitrary nested attribute; verify rejection.
2. Inject a synthetic canary into an approved string attribute; verify rejection.
3. Test trace expiry at 14 and 15 days.
4. Reject timezone-naive and future timestamps.
5. Reject invalid numeric and non-scalar values.
6. Verify the safe schema still preserves timing, status and token counts.
7. Plan actual collector/exporter integration tests with synthetic canaries, access controls and retention TTL.
8. Design a deletion-propagation drill across primary storage, backups and dashboards with privacy-owner approval.

## 5. 100-point acceptance
| Area | Points |
|---|---:|
| Data inventory and minimization | 20 |
| Schema, exporter and access controls | 20 |
| Retention/deletion design | 20 |
| Regression tests and canary evidence | 20 |
| Runbooks and independent review | 20 |
| **Total** | **100** |

Educational target >=90/100 with no critical privacy failures. Production acceptance requires collector/storage integration, retention enforcement, deletion verification and independent privacy/security sign-off.
