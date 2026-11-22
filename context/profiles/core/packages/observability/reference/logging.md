# Structured Logging

Structured logging with structlog, OpenTelemetry correlation, context propagation, and PII redaction, positioned as the technical-telemetry layer of the observability model.

Python logging in production systems requires machine-readable output, distributed trace correlation, and strict control over sensitive data. structlog provides the structured logging foundation, OpenTelemetry supplies trace and span identifiers for cross-service correlation, and contextvars propagate request-scoped metadata without threading it through function signatures.

## The Technical-Telemetry Layer

The practices in this reference — structured logging, trace correlation, and standard identifiers — constitute the technical-telemetry layer of the three-layer observability model: Telemetry (technical records) covering latency, retries, errors, and resource use. The layer model, the business-audit layer (the Audit Record history), and the shared-correlation-identifier discipline are defined in this profile's observability-layers reference. The layers are non-substitutable: these records describe execution and never define business truth.

## structlog as Foundation

structlog processes log events through a configurable processor chain, producing structured JSON output suitable for log aggregation systems (Datadog, a log-aggregation backend, Elasticsearch). It wraps the standard library `logging` module, maintaining compatibility with third-party libraries that emit standard log records.

```python
import structlog

logger = structlog.get_logger()
logger.info("order_processed", order_id="abc-123", total=99.50, items=3)
# {"event": "order_processed", "order_id": "abc-123", "total": 99.5, "items": 3, "timestamp": "...", "level": "info"}
```

Key-value pairs replace interpolated strings. Each log event is a dictionary, enabling filtering and alerting without regex parsing.

### Processor Chain

| Processor | Purpose |
|-----------|---------|
| `structlog.contextvars.merge_contextvars` | Injects request-scoped context |
| `structlog.processors.add_log_level` | Adds `level` field |
| `structlog.processors.TimeStamper(fmt="iso")` | Adds ISO 8601 timestamp |
| `structlog.processors.StackInfoRenderer()` | Renders stack traces on exceptions |
| Custom PII filter | Redacts sensitive fields |
| `structlog.processors.JSONRenderer()` | Final JSON serialization |

Processor order matters. Context merging runs first so downstream processors operate on the complete event. JSON rendering runs last as the terminal serializer.

## OpenTelemetry Integration

Distributed systems require correlation identifiers to trace requests across service boundaries. OpenTelemetry provides `trace_id` and `span_id` values that link log entries to distributed traces.

```python
from opentelemetry import trace

def add_otel_context(logger, method_name, event_dict):
    span = trace.get_current_span()
    ctx = span.get_span_context()
    if ctx.is_valid:
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
    return event_dict
```

Insert this processor after context merging but before rendering. Log entries then carry the same trace identifiers visible in tracing backends (Jaeger, Tempo, Honeycomb).

## Standard Identifiers

Adopt OpenTelemetry semantic conventions for consistent cross-service log queries:

| Field | Source | Purpose |
|-------|--------|---------|
| `service.name` | Configuration | Identifies the emitting service |
| `service.version` | Build metadata | Correlates behavior with deployments |
| `trace_id` | OpenTelemetry span context | Distributed trace correlation |
| `span_id` | OpenTelemetry span context | Identifies specific operation |
| `request_id` | HTTP header or generated UUID | Per-request tracking |
| `correlation_id` | Propagated header | Business transaction tracking across services |
| `environment` | Configuration | Distinguishes production, staging, development |

Bind service-level identifiers once at startup. Bind request-level identifiers in middleware per request.

## Context Propagation with contextvars

`contextvars` provides task-local storage that works across async boundaries. Bind request-scoped metadata in middleware; all downstream log calls automatically include it.

| Scope | Mechanism | Example Fields |
|-------|-----------|---------------|
| Process | `structlog.configure()` bound values | `service.name`, `environment` |
| Request | `structlog.contextvars.bind_contextvars()` | `request_id`, `user_id`, `path` |
| Operation | `logger.bind()` local instance | `order_id`, `batch_size` |

For asyncio applications, contextvars correctly isolate values between concurrent tasks. Each request handler receives its own context copy, preventing cross-request contamination.

## Correlation Beyond the Request Scope

The correlation identifier does not stop at the HTTP request boundary. A business transaction continues as async processing — a Domain Event published to the Domain Event transport and consumed by a separately deployed runtime — and ends as a committed transition recorded in the Audit Record history. The same `correlation_id` propagates across all three points:

| Propagation point | Mechanism |
|-------------------|-----------|
| Request | Ingress middleware binds `correlation_id` from the inbound header or generates one; every log entry in the request scope carries it |
| Message | Published Domain Event payloads carry `correlation_id`; the consumer binds it before any processing, so records on both sides of the transport boundary join one transaction |
| Audit | Every Audit Record write carries `correlation_id`, tying the committed transition back to the request and the async chain that caused it |

Because the application API runtime and the workflow execution runtime are separately deployed, the identifier must survive the transport boundary. Bind it in the consumer before any work runs — not lazily at first use — so no log entry is emitted without it. The layer model and the non-substitutability rule this propagation serves are defined in this profile's observability-layers reference.

## PII Redaction

Log events must not contain personally identifiable information in production. Implement redaction as a structlog processor that runs before serialization.

```python
REDACT_FIELDS = frozenset({
    "password", "token", "secret", "authorization",
    "email", "phone", "ssn", "credit_card",
    "api_key", "access_token", "refresh_token",
})

def redact_pii(logger, method_name, event_dict):
    for key in event_dict:
        if key.lower() in REDACT_FIELDS:
            event_dict[key] = "[REDACTED]"
    return event_dict
```

| Category | Fields | Treatment |
|----------|--------|-----------|
| Authentication | `password`, `token`, `api_key`, `secret` | Always redact |
| Personal data | `email`, `phone`, `ssn`, `date_of_birth` | Always redact in production |
| Financial | `credit_card`, `account_number` | Always redact |
| Internal identifiers | `user_id`, `order_id` | Retain (not PII) |

## Performance Considerations

Logging can become a bottleneck under high throughput. Guard expensive computations behind level checks, sample high-frequency events (health checks, heartbeats), and use `logging.handlers.QueueHandler` for async output in high-throughput services.

| Metric | Target | Warning |
|--------|--------|---------|
| Log call overhead | <5 microseconds | >10 microseconds |
| JSON serialization | <20 microseconds per event | >50 microseconds |
| Queue depth | <1,000 pending events | >5,000 pending |
| Dropped events | 0 (except sampled) | Any unintentional drops |

## Rule Companion: Rationale, Examples, Exceptions, and Tooling

*Relocated verbatim from the `logging` rule (during an earlier rule-slimming pass). Heading levels demoted one level; rule-local link pointers removed. No other change.*

### Rationale

Structured JSON logs enable machine parsing for aggregation systems (Datadog, a log-aggregation backend, Elasticsearch) without regex extraction. Correlation identifiers link log entries to distributed traces across service boundaries. Context propagation through contextvars eliminates manual threading of request metadata through function signatures while remaining async-safe. PII redaction as a processor prevents sensitive data from reaching log storage. This rule governs one layer only — the technical Telemetry layer. Applying it without knowing which layer owns which record produces the failure it is meant to prevent: a log line treated as the record of what the system committed. The observability reference draws that boundary and is mandatory here.

### Examples

**Correct**:
```python
import structlog

logger = structlog.get_logger()

# Structured key-value pairs, not interpolated strings
logger.info("order_processed", order_id="abc-123", total=99.50, items=3)

# Context propagation via contextvars in middleware
structlog.contextvars.bind_contextvars(request_id=request_id, user_id=user_id)

# PII redaction processor in the chain
def redact_pii(logger, method_name, event_dict):
    for key in event_dict:
        if key.lower() in REDACT_FIELDS:
            event_dict[key] = "[REDACTED]"
    return event_dict
```

**Incorrect**:
```python
import logging

logger = logging.getLogger(__name__)

# Interpolated strings lose structure
logger.info(f"Order {order_id} processed for user {email}, total: {total}")

# Manual context threading through every function
def process_order(order_id: str, request_id: str, user_id: str) -> None:
    logger.info(f"Processing order {order_id}", extra={"request_id": request_id})
```

### Exceptions

CLI tools and short-lived scripts may use `structlog` with console rendering (`ConsoleRenderer`) instead of JSON during development. Third-party library loggers that emit standard `logging` records integrate through structlog's `ProcessorFormatter` without modification.

### Tooling

Configure structlog at application startup with the processor chain: `merge_contextvars`, `add_log_level`, `TimeStamper(fmt="iso")`, OpenTelemetry context processor, PII redaction processor, `JSONRenderer`. Processor order matters: context merging first, serialization last.

## Evidence

structlog adoption across FastAPI, Celery, and Django production deployments demonstrates stability for Python structured logging. OpenTelemetry reached GA status for Python tracing and logging signals. contextvars (PEP 567) provides the standard mechanism for async-safe context propagation since Python 3.7, with structlog's native integration since v21.1.

## Related

- [Observability Layers](observability.md)
- [Type Safety](../../typing/reference/type_safety.md)
- [Python Toolchain](../../toolchain/reference/tooling.md)
- [Logging Setup](logging_setup.md)
