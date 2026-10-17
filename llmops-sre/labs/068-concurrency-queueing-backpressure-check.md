# Chapter 068 — Executable Concurrency, Queueing and Backpressure Lab

**Status:** Published; execution pending. **Safety:** offline deterministic queue simulation only; no model endpoint or external network.

## 1. Prerequisites
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
Windows PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Queue simulator
Save as `queue_lab.py`:

```python
from dataclasses import dataclass
import heapq
import json
import math

@dataclass(frozen=True)
class Config:
    workers: int = 2
    queue_limit: int = 4
    service_s: float = 1.0
    deadline_s: float = 5.0

@dataclass(frozen=True)
class Event:
    arrival: float
    accepted: bool
    reason: str
    wait_s: float | None
    finish: float | None

def simulate(arrivals, config=Config()):
    if type(config.workers) is not int or config.workers < 1:
        raise ValueError("workers must be positive")
    if type(config.queue_limit) is not int or config.queue_limit < 0:
        raise ValueError("queue limit must be nonnegative")
    if not math.isfinite(config.service_s) or config.service_s <= 0:
        raise ValueError("service time must be positive")
    if not math.isfinite(config.deadline_s) or config.deadline_s <= 0:
        raise ValueError("deadline must be positive")
    if any(not isinstance(t, (int, float)) or not math.isfinite(t) or t < 0 for t in arrivals):
        raise ValueError("invalid arrival")
    if list(arrivals) != sorted(arrivals):
        raise ValueError("arrivals must be sorted")
    # A finite-server FIFO model: each worker's next free time is tracked.
    # Waiting count at arrival includes only previously accepted requests
    # whose service start time is still in the future.
    free = [0.0] * config.workers
    heapq.heapify(free)
    scheduled_starts = []
    events = []
    for arrival in arrivals:
        scheduled_starts = [start for start in scheduled_starts if start > arrival]
        next_free = free[0]
        start = max(arrival, next_free)
        wait = start - arrival
        if start > arrival and len(scheduled_starts) >= config.queue_limit:
            events.append(Event(arrival, False, "queue_full", None, None))
            continue
        if wait + config.service_s > config.deadline_s:
            events.append(Event(arrival, False, "deadline_unachievable", None, None))
            continue
        heapq.heapreplace(free, start + config.service_s)
        if start > arrival:
            scheduled_starts.append(start)
        events.append(Event(arrival, True, "admitted", wait, start + config.service_s))
    return events

def report(events, window_s=None):
    if not events:
        raise ValueError("no events")
    horizon = window_s if window_s is not None else max(
        max(e.finish or e.arrival for e in events) - min(e.arrival for e in events), 1e-9)
    if not math.isfinite(horizon) or horizon <= 0:
        raise ValueError("invalid window")
    accepted = [e for e in events if e.accepted]
    rejected = [e for e in events if not e.accepted]
    waits = sorted(e.wait_s for e in accepted)
    return {
        "offered": len(events),
        "admitted": len(accepted),
        "rejected": len(rejected),
        "rejection_ratio": len(rejected) / len(events),
        "admitted_rps": len(accepted) / horizon,
        "wait_p95_s": waits[math.ceil(0.95 * len(waits)) - 1] if waits else None,
        "max_wait_s": max(waits) if waits else None,
    }

if __name__ == "__main__":
    scenarios = {
        "healthy": [i * 0.7 for i in range(20)],
        "overload": [i * 0.05 for i in range(40)],
        "burst": [0.0] * 30,
    }
    for name, arrivals in scenarios.items():
        print(name, json.dumps(report(simulate(arrivals)), sort_keys=True))
```

**Important limitation:** This is a FIFO finite-worker approximation with fixed service time. It does not model vLLM continuous batching, prefill/decode interference, cancellations after admission or token-dependent KV consumption. `queue_limit` applies to scheduled waiting jobs, not total in-flight requests.

## 3. Automated tests
Save as `test_queue_lab.py`:

```python
import pytest
from queue_lab import Config, simulate, report

def test_all_healthy_accepted():
    events = simulate([0, 2, 4, 6], Config(workers=1, queue_limit=1))
    assert all(e.accepted for e in events)

def test_overload_rejects():
    events = simulate([0] * 20, Config(workers=1, queue_limit=2, deadline_s=10))
    assert sum(not e.accepted for e in events) > 0

def test_queue_limit_zero():
    events = simulate([0, 0, 0], Config(workers=1, queue_limit=0))
    assert sum(e.accepted for e in events) == 1

def test_deadline_admission():
    events = simulate([0, 0], Config(workers=1, queue_limit=2, service_s=2, deadline_s=2))
    assert events[0].accepted
    assert events[1].reason == "deadline_unachievable"

def test_wait_increases_under_burst():
    events = simulate([0, 0, 0], Config(workers=1, queue_limit=3, deadline_s=10))
    waits = [e.wait_s for e in events if e.accepted]
    assert waits == [0, 1, 2]

def test_more_workers_reduce_rejections():
    arrivals = [0] * 20
    one = report(simulate(arrivals, Config(workers=1, queue_limit=2, deadline_s=10)))
    four = report(simulate(arrivals, Config(workers=4, queue_limit=2, deadline_s=10)))
    assert four["rejected"] < one["rejected"]

def test_accounting():
    result = report(simulate([0] * 10))
    assert result["offered"] == result["admitted"] + result["rejected"]

@pytest.mark.parametrize("cfg", [
    Config(workers=0), Config(queue_limit=-1),
    Config(service_s=0), Config(deadline_s=0),
])
def test_invalid_config(cfg):
    with pytest.raises(ValueError):
        simulate([0], cfg)

def test_unsorted_arrivals():
    with pytest.raises(ValueError):
        simulate([1, 0])

def test_empty_report():
    with pytest.raises(ValueError):
        report([])
```

Run:
```bash
python queue_lab.py
python -m pytest -q
```
Expected: **13 passing pytest cases** (9 individual tests and 4 parameterized cases).

## 4. Failure drills
1. Raise offered load from 1 to 20 RPS with fixed service rate; observe queue age and rejection.
2. Set `queue_limit=0`; verify overload rejects before waiting.
3. Increase workers from 1 to 4; compare admitted work and latency.
4. Reduce deadline below service time; verify early rejection.
5. Simulate a burst of 100 arrivals at the same timestamp; record bounded queue behavior.
6. Remove the queue cap in a disposable copy and explain the resulting memory/latency risk.
7. Discuss how retries can amplify offered load even when original client traffic is unchanged.

## 5. Evidence and acceptance
Capture Python/pytest versions, configs, offered/admitted/rejected counts, wait p95 and failure drills. Do not interpret synthetic RPS as production throughput.

| Criterion | Points |
|---|---:|
| Traffic accounting and metrics | 20 |
| Bounded queue and overload response | 20 |
| Deadline-aware admission | 20 |
| Deterministic tests and drills | 20 |
| Security, safety and runbook readiness | 20 |
| **Total** | **100** |

Recommended educational acceptance >=90/100; production capacity acceptance requires real staging traffic, on-call review and approved rollback plan. Remove lab artifacts only after preserving sanitized evidence.
