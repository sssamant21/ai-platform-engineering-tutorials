# Chapter 087 — Executable Inference 429, 5xx, Timeout and SSE Incident Lab

**Status:** Published; execution and staging review pending. Uses synthetic request observations only.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Offline triage and stream validation
Save as `inference_incident.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class Observation:
    status: int
    reason: str
    queue_ms: int
    ttft_ms: int
    backend_ready: bool
    retry_count: int

def triage(o):
    if type(o.status) is not int or not 100 <= o.status <= 599:
        return "invalid_observation"
    if any(type(x) is not int or x < 0 for x in (o.queue_ms, o.ttft_ms, o.retry_count)):
        return "invalid_observation"
    if o.retry_count > 3:
        return "retry_amplification"
    if o.status == 429:
        return "quota_limited" if o.reason == "quota" else "capacity_limited"
    if o.status in (502, 503) and not o.backend_ready:
        return "backend_unavailable"
    if o.status in (504, 408) or o.ttft_ms > 10000:
        return "timeout_or_slow_first_token"
    if 500 <= o.status <= 599:
        return "server_error"
    if o.queue_ms > 5000:
        return "queue_pressure"
    return "healthy"

def parse_sse(stream):
    if not isinstance(stream, str):
        raise TypeError("stream must be text")
    events = []
    for block in stream.replace("\r\n", "\n").split("\n\n"):
        if not block.strip():
            continue
        data_lines = []
        for line in block.split("\n"):
            if line.startswith("data:"):
                data_lines.append(line[5:].lstrip(" "))
            elif line.startswith(":") or line.startswith("event:") or line.startswith("id:") or line.startswith("retry:"):
                continue
            else:
                raise ValueError("unsupported SSE line")
        if data_lines:
            events.append("\n".join(data_lines))
    return events

def completed_stream(stream, terminal_marker="[DONE]"):
    events = parse_sse(stream)
    return bool(events and events[-1] == terminal_marker)

if __name__ == "__main__":
    observation = Observation(429, "quota", 0, 0, True, 1)
    stream = "data: synthetic-token\n\ndata: [DONE]\n\n"
    print(json.dumps({"triage": triage(observation), "stream_complete": completed_stream(stream)}, indent=2))
```

**Limitations:** The SSE parser intentionally supports only a teaching subset of framing, not all EventSource semantics, multiline edge cases or partial network chunks. `[DONE]` is a configurable sample marker, not a universal SSE completion standard. No live model, proxy, network or billing system is exercised.

## 3. Automated tests
Save as `test_inference_incident.py`:
```python
import pytest
from inference_incident import Observation, triage, parse_sse, completed_stream

@pytest.mark.parametrize("obs,expected", [
    (Observation(429, "quota", 0, 0, True, 0), "quota_limited"),
    (Observation(429, "overload", 0, 0, True, 0), "capacity_limited"),
    (Observation(503, "", 0, 0, False, 0), "backend_unavailable"),
    (Observation(504, "", 9000, 0, True, 0), "timeout_or_slow_first_token"),
    (Observation(500, "", 0, 0, True, 0), "server_error"),
    (Observation(200, "", 6000, 500, True, 0), "queue_pressure"),
    (Observation(200, "", 10, 100, True, 0), "healthy"),
    (Observation(200, "", 0, 0, True, 4), "retry_amplification"),
    (Observation(700, "", 0, 0, True, 0), "invalid_observation"),
])
def test_triage(obs, expected):
    assert triage(obs) == expected

def test_sse_complete():
    assert completed_stream("data: synthetic\n\ndata: [DONE]\n\n")

def test_sse_missing_terminal():
    assert not completed_stream("data: synthetic\n\n")

def test_sse_comment():
    assert parse_sse(": heartbeat\n\ndata: synthetic\n\n") == ["synthetic"]

def test_sse_multiline_data():
    assert parse_sse("data: first\ndata: second\n\n") == ["first\nsecond"]

def test_sse_crlf():
    assert completed_stream("data: synthetic\r\n\r\ndata: [DONE]\r\n\r\n")

def test_sse_bad_line():
    with pytest.raises(ValueError):
        parse_sse("Authorization: demo\n\n")

def test_sse_nontext():
    with pytest.raises(TypeError):
        parse_sse(None)

def test_negative_latency_invalid():
    assert triage(Observation(200, "", -1, 0, True, 0)) == "invalid_observation"

def test_ttft_timeout():
    assert triage(Observation(200, "", 0, 12000, True, 0)) == "timeout_or_slow_first_token"
```

Run:
```bash
python inference_incident.py
python -m pytest -q
```
Expected: **18 pytest cases** (9 parameterized and 9 standalone). Execution has not been performed.

## 4. SRE game-day checklist
1. Separate quota-limited and overload-limited 429.
2. Identify backend unavailable vs generic 5xx.
3. Simulate queue timeout and slow first token.
4. Verify SSE framing, heartbeat, multiline data and terminal marker.
5. Simulate incomplete stream and client disconnect handling in staging.
6. Detect retry amplification and estimate additional token/GPU cost.
7. Compare gateway, proxy, scheduler and model server deadlines.
8. Document rollback and incident communication under Chapter 086.

## 5. 100-point acceptance
| Area | Points |
|---|---:|
| Status and reason diagnosis | 20 |
| Timeout and queue triage | 20 |
| SSE framing and completion checks | 20 |
| Retry/cost and recovery controls | 20 |
| Automated tests and incident evidence | 20 |
| **Total** | **100** |

Educational target >=90/100 with no unbounded retry or policy bypass. Production approval requires real SSE traffic, client disconnect tests, gateway metrics and independent operational review.
