# Chapter 086 — Executable LLM Incident Command, SEV and Escalation Lab

**Status:** Published; execution and operational review pending. Synthetic incidents only.

## 1. Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 2. Deterministic incident command simulator
Save as `incident_command.py`:
```python
from dataclasses import dataclass, field
from datetime import datetime, timezone
import json

@dataclass(frozen=True)
class Signals:
    widespread_outage: bool = False
    cross_tenant_exposure: bool = False
    uncontrolled_spend: bool = False
    major_degradation: bool = False
    partial_degradation: bool = False

def classify(signals):
    if signals.widespread_outage or signals.cross_tenant_exposure or signals.uncontrolled_spend:
        return "SEV1"
    if signals.major_degradation:
        return "SEV2"
    if signals.partial_degradation:
        return "SEV3"
    return "SEV4"

def escalation(signals):
    owners = {"incident_commander", "operations_oncall"}
    if signals.cross_tenant_exposure:
        owners.add("security_privacy")
    if signals.uncontrolled_spend:
        owners.add("finops")
    if signals.widespread_outage or signals.major_degradation:
        owners.add("communications")
    return sorted(owners)

@dataclass
class Incident:
    incident_id: str
    started_at: datetime
    commander: str | None = None
    timeline: list[dict] = field(default_factory=list)
    mitigated: bool = False
    slo_verified: bool = False
    security_signoff: bool = False

    def add_event(self, when, event):
        if not isinstance(when, datetime) or when.tzinfo is None:
            raise ValueError("UTC-aware timestamp required")
        if when < self.started_at:
            raise ValueError("event precedes incident")
        if self.timeline and when < self.timeline[-1]["at"]:
            raise ValueError("timeline must be chronological")
        if not event or any(x in event.lower() for x in ("demo_secret_", "synthetic_canary_")):
            raise ValueError("unsafe or empty event")
        self.timeline.append({"at": when, "event": event})

    def can_close(self, signals):
        return bool(
            self.commander and self.mitigated and self.slo_verified
            and (not signals.cross_tenant_exposure or self.security_signoff)
            and self.timeline
        )

def sample():
    now = datetime(2026, 1, 1, tzinfo=timezone.utc)
    incident = Incident("INC-086", now, commander="oncall-ic")
    incident.add_event(now, "Incident declared")
    return incident, Signals(major_degradation=True)

if __name__ == "__main__":
    incident, signals = sample()
    print(json.dumps({
        "severity": classify(signals),
        "escalation": escalation(signals),
        "can_close": incident.can_close(signals),
    }, indent=2))
```

**Limitations:** This lab does not contact PagerDuty, Opsgenie, Slack, incident bridges, Kubernetes or GPU systems. It uses simplified severity rules and a string-based synthetic canary guard, not production PII detection. Real closure must follow organizational incident/security policy.

## 3. Automated tests
Save as `test_incident_command.py`:
```python
from datetime import datetime, timedelta, timezone
import pytest
from incident_command import Signals, Incident, classify, escalation, sample

@pytest.mark.parametrize("signals,expected", [
    (Signals(widespread_outage=True), "SEV1"),
    (Signals(cross_tenant_exposure=True), "SEV1"),
    (Signals(uncontrolled_spend=True), "SEV1"),
    (Signals(major_degradation=True), "SEV2"),
    (Signals(partial_degradation=True), "SEV3"),
    (Signals(), "SEV4"),
])
def test_severity(signals, expected):
    assert classify(signals) == expected

def test_security_escalation():
    assert "security_privacy" in escalation(Signals(cross_tenant_exposure=True))

def test_finops_escalation():
    assert "finops" in escalation(Signals(uncontrolled_spend=True))

def test_comms_escalation():
    assert "communications" in escalation(Signals(major_degradation=True))

def test_open_incident_not_closable():
    incident, signals = sample()
    assert not incident.can_close(signals)

def test_recovered_incident_closable():
    incident, signals = sample()
    incident.mitigated = True
    incident.slo_verified = True
    assert incident.can_close(signals)

def test_exposure_requires_security_signoff():
    incident, _ = sample()
    incident.mitigated = True
    incident.slo_verified = True
    assert not incident.can_close(Signals(cross_tenant_exposure=True))
    incident.security_signoff = True
    assert incident.can_close(Signals(cross_tenant_exposure=True))

def test_no_commander_no_close():
    incident, signals = sample()
    incident.commander = None
    incident.mitigated = True
    incident.slo_verified = True
    assert not incident.can_close(signals)

def test_timeline_rejects_backdating():
    incident, _ = sample()
    with pytest.raises(ValueError):
        incident.add_event(incident.started_at - timedelta(seconds=1), "Backdated")

def test_timeline_rejects_naive_datetime():
    incident, _ = sample()
    with pytest.raises(ValueError):
        incident.add_event(datetime(2026, 1, 1), "Naive")

def test_timeline_rejects_synthetic_secret():
    incident, _ = sample()
    with pytest.raises(ValueError):
        incident.add_event(incident.started_at, "demo_secret_123")
```

Run:
```bash
python incident_command.py
python -m pytest -q
```
Expected: **16 pytest cases** (10 standalone and 6 parameterized). Execution has not been performed.

## 4. Game-day checklist
1. Declare a SEV2 model-serving degradation and assign incident command.
2. Escalate simulated cross-tenant exposure to security/privacy as SEV1.
3. Escalate uncontrolled spend to FinOps.
4. Record a chronological, privacy-safe incident timeline.
5. Reject closure before mitigation and SLO recovery verification.
6. Require specialist sign-off for exposure cases.
7. Simulate a failed rollback and escalate decision authority.
8. Write a five-sentence customer-safe status update and RCA/COA outline.

## 5. 100-point assessment
| Domain | Points |
|---|---:|
| Severity classification | 20 |
| Command roles and escalation | 20 |
| Timeline and evidence discipline | 20 |
| Recovery and closure gates | 20 |
| Communications and postmortem | 20 |
| **Total** | **100** |

Educational target >=90/100 with no critical missed escalation. Production acceptance requires a real game day, alert routing validation and independent operations/security review.
