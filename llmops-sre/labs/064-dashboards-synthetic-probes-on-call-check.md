# Chapter 064 — Executable Dashboards, Synthetic Probes and On-Call Lab

**Status:** Published; live execution pending. **Safety:** local HTTP server, fixed synthetic data, no real model or external API calls.

## 1. Prerequisites
Python 3.11+ and pytest. Create a disposable directory.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell activation: `.venv\Scripts\Activate.ps1`.

## 2. Local synthetic inference server
Save as `fake_inference.py`:

```python
import json
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path != "/v1/infer":
            self.send_error(404)
            return
        size = int(self.headers.get("Content-Length", "0"))
        if size > 4096:
            self.send_error(413)
            return
        try:
            body = json.loads(self.rfile.read(size))
        except (ValueError, UnicodeDecodeError):
            self.send_error(400)
            return
        if body != {"prompt": "synthetic-health-check", "max_tokens": 4}:
            self.send_error(400)
            return
        payload = json.dumps({"result": "SYNTHETIC_OK", "finish_reason": "stop"}).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(payload)))
        self.end_headers()
        self.wfile.write(payload)

    def log_message(self, format, *args):
        # Avoid request bodies and uncontrolled error text in logs.
        pass

if __name__ == "__main__":
    ThreadingHTTPServer(("127.0.0.1", 8765), Handler).serve_forever()
```

Start in one terminal: `python fake_inference.py`.

## 3. Safe synthetic probe
Save as `probe.py`:

```python
import json
import time
from urllib.request import Request, urlopen
from urllib.error import HTTPError, URLError

PAYLOAD = json.dumps({"prompt": "synthetic-health-check", "max_tokens": 4}).encode()

def check(url="http://127.0.0.1:8765/v1/infer", timeout=2.0):
    start = time.monotonic()
    request = Request(url, data=PAYLOAD, headers={"Content-Type": "application/json"}, method="POST")
    try:
        with urlopen(request, timeout=timeout) as response:
            if response.status != 200:
                return {"ok": False, "reason": "bad_status"}
            if response.headers.get("Content-Type", "").split(";")[0] != "application/json":
                return {"ok": False, "reason": "bad_content_type"}
            data = json.loads(response.read(4096))
        if data != {"result": "SYNTHETIC_OK", "finish_reason": "stop"}:
            return {"ok": False, "reason": "bad_response"}
        return {"ok": True, "reason": "success", "duration_ms": round((time.monotonic() - start) * 1000, 2)}
    except HTTPError:
        return {"ok": False, "reason": "http_error"}
    except (URLError, TimeoutError, OSError):
        return {"ok": False, "reason": "network_or_timeout"}
    except (ValueError, UnicodeDecodeError):
        return {"ok": False, "reason": "bad_json"}

if __name__ == "__main__":
    print(json.dumps(check(), sort_keys=True))
```

Run `python probe.py`. Expected `ok: true` and `reason: success`. The probe emits only status and duration, not prompt or completion.

## 4. Automated tests
Save as `test_probe.py`:

```python
import json
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from threading import Thread
from time import sleep
from probe import check

class FixtureHandler(BaseHTTPRequestHandler):
    mode = "ok"

    def do_POST(self):
        if self.mode == "timeout":
            sleep(0.2)
        code = 503 if self.mode == "error" else 200
        data = b'{"result":"SYNTHETIC_OK","finish_reason":"stop"}'
        if self.mode == "malformed":
            data = b'not-json'
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        try:
            self.wfile.write(data)
        except BrokenPipeError:
            pass

    def log_message(self, format, *args):
        pass

def run(mode):
    class LocalHandler(FixtureHandler):
        pass
    LocalHandler.mode = mode
    server = ThreadingHTTPServer(("127.0.0.1", 0), LocalHandler)
    thread = Thread(target=server.serve_forever, daemon=True)
    thread.start()
    return server, f"http://127.0.0.1:{server.server_port}/v1/infer"

def test_success():
    server, url = run("ok")
    try:
        assert check(url)["ok"] is True
    finally:
        server.shutdown()
        server.server_close()

def test_http_error():
    server, url = run("error")
    try:
        assert check(url)["reason"] == "http_error"
    finally:
        server.shutdown()
        server.server_close()

def test_malformed_json():
    server, url = run("malformed")
    try:
        assert check(url)["reason"] == "bad_json"
    finally:
        server.shutdown()
        server.server_close()

def test_timeout():
    server, url = run("timeout")
    try:
        assert check(url, timeout=0.02)["reason"] == "network_or_timeout"
    finally:
        server.shutdown()
        server.server_close()
```

Run `python -m pytest -q`. Expected: **4 passing tests**.

## 5. Dashboard-as-code review checklist
Define and review these panel specifications in your preferred Grafana provisioning repository; the lab does not claim to deploy Grafana:

| Panel | Query source | Units | Failure mode |
|---|---|---|---|
| Availability SLI | approved request/bad counters | percent | idle denominator |
| TTFT p95 | verified TTFT histogram | seconds | low sample volume |
| Queue depth | approved inference gauge | requests | stale scrape |
| Probe success | bounded synthetic result counter | percent | probe identity/region |
| Error budget remaining | rolling SLO rules | percent | missing data |
| GPU saturation | approved GPU exporter | percent | missing worker metrics |

Require deployment annotations, no-data behavior, time-window defaults, on-call owner and runbook links.

## 6. Fault drills
1. Stop the local server: probe should return `network_or_timeout`.
2. Change the expected response token in a disposable copy: probe should report `bad_response`.
3. Increase server delay: timeout test should fail safely.
4. Replace synthetic prompt with arbitrary content: server should return HTTP 400.
5. Review simulated alert routing and escalation paths in documentation only; do not send real pages.

## 7. Acceptance
- [ ] Local server and probe run without external services.
- [ ] All four pytest cases pass.
- [ ] Probe emits no prompt or completion content.
- [ ] Dashboard panels include owner, units, no-data handling and SLO links.
- [ ] Runbook and on-call handoff reviewed.
- [ ] Technical/security review and live execution recorded separately.

Stop the server with Ctrl+C and remove disposable lab artifacts only after retaining sanitized evidence.
