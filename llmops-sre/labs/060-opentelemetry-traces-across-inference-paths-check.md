# Chapter 060 — Executable OpenTelemetry Tracing Lab

**Status:** Lab instructions published; execution and review pending.  
**Scope:** synthetic local workload only; no model or GPU required.

## 1. Setup [LAB-MUTATE]
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install "opentelemetry-api>=1.25,<2" "opentelemetry-sdk>=1.25,<2" "opentelemetry-exporter-otlp-proto-http>=1.25,<2" pytest
```
On PowerShell activate with `.venv\Scripts\Activate.ps1`.

## 2. Instrument a synthetic inference request
Save as `trace_demo.py`:

```python
import os
import time
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.trace import Status, StatusCode

def build_provider(exporter=None):
    provider = TracerProvider(resource=Resource.create({
        "service.name": "synthetic-llm-gateway",
        "deployment.environment.name": "local-lab"
    }))
    provider.add_span_processor(BatchSpanProcessor(exporter or ConsoleSpanExporter()))
    return provider

def synthetic_request(tracer):
    with tracer.start_as_current_span("POST /v1/chat/completions", kind=trace.SpanKind.SERVER) as root:
        root.set_attribute("http.route", "/v1/chat/completions")
        with tracer.start_as_current_span("authorize_request") as span:
            span.set_attribute("auth.outcome", "allowed")
        with tracer.start_as_current_span("route_inference") as span:
            span.set_attribute("model.family", "synthetic")
        with tracer.start_as_current_span("inference_queue"):
            time.sleep(0.01)
        with tracer.start_as_current_span("synthetic_inference") as span:
            span.set_attribute("simulation", True)
            time.sleep(0.02)
        with tracer.start_as_current_span("stream_response") as span:
            span.add_event("first_token")
            time.sleep(0.01)
            span.add_event("stream_complete")
        root.set_status(Status(StatusCode.OK))

if __name__ == "__main__":
    provider = build_provider()
    tracer = provider.get_tracer("llmops.chapter060")
    try:
        synthetic_request(tracer)
    finally:
        provider.force_flush(timeout_millis=5000)
        provider.shutdown()
```

Run: `python trace_demo.py`. Expected: one root and five child spans with matching trace IDs; streaming span has two ordered events. Timings are simulated, **not** measured model latency.

## 3. Automated span-tree and privacy tests
Save as `test_trace_demo.py`:

```python
from opentelemetry.sdk.trace.export import InMemorySpanExporter, SimpleSpanProcessor
from opentelemetry.sdk.trace import TracerProvider
from trace_demo import synthetic_request

def test_tree_and_redaction():
    exporter = InMemorySpanExporter()
    provider = TracerProvider()
    provider.add_span_processor(SimpleSpanProcessor(exporter))
    synthetic_request(provider.get_tracer("test"))
    spans = exporter.get_finished_spans()
    assert len(spans) == 6
    roots = [s for s in spans if s.parent is None]
    assert len(roots) == 1
    root = roots[0]
    assert all(s.context.trace_id == root.context.trace_id for s in spans)
    assert all(s.parent is not None and s.parent.span_id == root.context.span_id for s in spans if s is not root)
    stream = next(s for s in spans if s.name == "stream_response")
    assert [e.name for e in stream.events] == ["first_token", "stream_complete"]
    forbidden = ("prompt", "authorization", "api_key", "tenant_id", "completion_text")
    assert not any(any(k in key.lower() for k in forbidden) for s in spans for key in s.attributes)
    provider.shutdown()
```

Run `python -m pytest -q`. Expected: 1 passed. If your installed SDK does not export `InMemorySpanExporter` from `opentelemetry.sdk.trace.export`, use `from opentelemetry.sdk.trace.export.in_memory_span_exporter import InMemorySpanExporter`.

## 4. Optional OTLP Collector
Create `collector.yaml`:

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
processors:
  batch: {}
exporters:
  debug:
    verbosity: detailed
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

Start local collector (pin a reviewed release before team reuse):

```bash
docker run --rm --name otel-060 -p 127.0.0.1:4318:4318 -v "$PWD/collector.yaml:/etc/otelcol/config.yaml:ro" otel/opentelemetry-collector:0.113.0 --config=/etc/otelcol/config.yaml
```

In another terminal replace the exporter initialization in `build_provider` with:

```python
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
provider.add_span_processor(BatchSpanProcessor(
    exporter or OTLPSpanExporter(endpoint="http://127.0.0.1:4318/v1/traces")
))
```

Remove the original `provider.add_span_processor(...)` line first, then run `python trace_demo.py`. Expected: collector debug output contains the six spans. Use localhost only; do not expose unauthenticated OTLP to shared networks.

## 5. Failure drills and evidence
- Stop collector; verify the synthetic request still completes and exporter failures are logged, not propagated to request handling.
- Change OTLP port; inspect export errors and restore endpoint.
- Intentionally omit a child span in a disposable copy; confirm tree test fails.
- Add a forbidden attribute to a disposable copy; confirm privacy test fails.
- Record sanitized test output, timestamps, collector logs, version pins and cleanup evidence. Never store real prompts or tokens.

## 6. Cleanup
```bash
docker stop otel-060
deactivate
```
Delete disposable virtual environment and lab files only after saving sanitized evidence.

## 7. Acceptance
- [ ] Python smoke run
- [ ] pytest tree/privacy checks
- [ ] collector receives six spans
- [ ] failure drills complete
- [ ] security review complete
- [ ] live execution evidence attached

**Publication does not imply execution.**
