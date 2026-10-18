# Chapter 082 — Executable AI Gateway Rate Limits and Denial-of-Wallet Protection Lab

**Status:** Published; execution and independent review pending. Offline single-process simulator with synthetic pricing.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic admission and budget reservation
Save as `gateway_budget.py`:
```python
from dataclasses import dataclass, field
from decimal import Decimal
import json

INPUT_PER_M = Decimal("2.00")    # synthetic USD / million input tokens
OUTPUT_PER_M = Decimal("8.00")   # synthetic USD / million output tokens
MILLION = Decimal(1_000_000)

@dataclass(frozen=True)
class Request:
    request_id: str
    tenant: str
    verified_tenant: str
    input_tokens: int
    max_output_tokens: int

@dataclass
class Ledger:
    budget: Decimal
    reserved: dict[str, Decimal] = field(default_factory=dict)
    completed: dict[str, Decimal] = field(default_factory=dict)
    rate_tokens: int = 3
    concurrency: int = 0

    def used(self):
        return sum(self.reserved.values(), Decimal(0)) + sum(self.completed.values(), Decimal(0))

def cost(input_tokens, output_tokens):
    return (Decimal(input_tokens) * INPUT_PER_M + Decimal(output_tokens) * OUTPUT_PER_M) / MILLION

def admit(ledger, request):
    if not request.tenant or request.tenant != request.verified_tenant:
        return {"allow": False, "reason": "tenant_denied"}
    if not request.request_id:
        return {"allow": False, "reason": "invalid_request_id"}
    if request.request_id in ledger.reserved or request.request_id in ledger.completed:
        return {"allow": False, "reason": "duplicate_request"}
    counts = (request.input_tokens, request.max_output_tokens)
    if any(type(x) is not int or x < 0 for x in counts):
        return {"allow": False, "reason": "invalid_tokens"}
    if request.input_tokens > 8192 or request.max_output_tokens > 1024:
        return {"allow": False, "reason": "token_limit"}
    if ledger.rate_tokens <= 0:
        return {"allow": False, "reason": "rate_limit"}
    if ledger.concurrency >= 2:
        return {"allow": False, "reason": "concurrency_limit"}
    estimate = cost(*counts)
    if ledger.used() + estimate > ledger.budget:
        return {"allow": False, "reason": "budget_exceeded"}
    ledger.reserved[request.request_id] = estimate
    ledger.rate_tokens -= 1
    ledger.concurrency += 1
    return {"allow": True, "reason": "admitted", "reserved_usd": str(estimate)}

def finish(ledger, request, actual_output_tokens):
    if request.request_id not in ledger.reserved:
        return {"ok": False, "reason": "unknown_reservation"}
    if type(actual_output_tokens) is not int or not 0 <= actual_output_tokens <= request.max_output_tokens:
        return {"ok": False, "reason": "invalid_actual_usage"}
    actual = cost(request.input_tokens, actual_output_tokens)
    ledger.reserved.pop(request.request_id)
    ledger.completed[request.request_id] = actual
    ledger.concurrency -= 1
    return {"ok": True, "charged_usd": str(actual)}

if __name__ == "__main__":
    ledger = Ledger(Decimal("0.10"))
    req = Request("req-001", "alpha", "alpha", 1000, 100)
    print(json.dumps(admit(ledger, req), indent=2))
    print(json.dumps(finish(ledger, req, 40), indent=2))
    print("spent_usd", ledger.used())
```

**Security limitations:** `verified_tenant` is a simulated gateway assertion, not cryptographic identity. The ledger is in-memory and **not safe across multiple processes or replicas**. No clock-driven token bucket, persistent journal, idempotent external metering or crash recovery is implemented. A production service must use atomic durable reservations, verified identity and trusted usage.

## 3. Automated tests
Save as `test_gateway_budget.py`:
```python
from decimal import Decimal
import pytest
from gateway_budget import Ledger, Request, admit, finish, cost

R = Request("r1", "alpha", "alpha", 1000, 100)

def test_admission():
    assert admit(Ledger(Decimal("1")), R)["allow"]

def test_tenant_spoof():
    assert admit(Ledger(Decimal("1")), Request("r1", "beta", "alpha", 100, 10))["reason"] == "tenant_denied"

def test_duplicate():
    ledger = Ledger(Decimal("1"))
    admit(ledger, R)
    assert admit(ledger, R)["reason"] == "duplicate_request"

def test_budget_exceeded():
    assert admit(Ledger(Decimal("0.0001")), R)["reason"] == "budget_exceeded"

def test_rate_limit():
    assert admit(Ledger(Decimal("1"), rate_tokens=0), R)["reason"] == "rate_limit"

def test_concurrency_limit():
    assert admit(Ledger(Decimal("1"), concurrency=2), R)["reason"] == "concurrency_limit"

def test_reconciliation():
    ledger = Ledger(Decimal("1"))
    admit(ledger, R)
    assert finish(ledger, R, 50)["ok"]
    assert ledger.used() == cost(1000, 50)
    assert ledger.concurrency == 0

def test_duplicate_finish():
    ledger = Ledger(Decimal("1"))
    admit(ledger, R)
    finish(ledger, R, 50)
    assert finish(ledger, R, 50)["reason"] == "unknown_reservation"

def test_invalid_actual_usage():
    ledger = Ledger(Decimal("1"))
    admit(ledger, R)
    assert not finish(ledger, R, 101)["ok"]
    assert "r1" in ledger.reserved

@pytest.mark.parametrize("input_tokens,output_tokens", [
    (-1, 10),
    (100, -1),
    (8193, 100),
    (100, 1025),
    (True, 100),
])
def test_invalid_or_excessive_tokens(input_tokens, output_tokens):
    result = admit(Ledger(Decimal("1")), Request("r1", "alpha", "alpha", input_tokens, output_tokens))
    assert not result["allow"]

def test_reserved_budget_released():
    ledger = Ledger(Decimal("1"))
    admit(ledger, R)
    reserved = ledger.used()
    finish(ledger, R, 0)
    assert ledger.used() <= reserved

def test_completed_idempotency_key():
    ledger = Ledger(Decimal("1"))
    admit(ledger, R)
    finish(ledger, R, 10)
    assert admit(ledger, R)["reason"] == "duplicate_request"
```

Run:
```bash
python gateway_budget.py
python -m pytest -q
```
Expected: **16 pytest cases** (11 standalone plus 5 parameterized). Execution has not been performed.

## 4. Failure drills
1. Submit a request with mismatched verified tenant and observe denial.
2. Submit the same request ID twice and confirm no double reservation.
3. Exhaust request-rate tokens and concurrent permits independently.
4. Exhaust synthetic budget and confirm fail-closed admission.
5. Exceed input/output token ceilings.
6. Reconcile actual usage below the reservation and verify unused budget is released.
7. Simulate a worker crash between reservation and reconciliation; document missing durability and recovery.
8. Design staging tests for distributed atomic counters, retries, cancellation, provider metering and model-specific prices.

## 5. Acceptance rubric
| Domain | Points |
|---|---:|
| Identity-bound admission | 20 |
| Token, rate and concurrency quotas | 20 |
| Cost reservation and reconciliation | 20 |
| Adversarial/incident tests | 20 |
| Production architecture and audit evidence | 20 |
| **Total** | **100** |

Educational target >=90/100 with no critical overspend/tenant-isolation failure. Production approval requires distributed concurrency tests, durable metering, alerting and independent security/FinOps review.
