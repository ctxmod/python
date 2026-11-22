# Logging Setup

Step-by-step procedure for configuring structured logging with structlog and OpenTelemetry.

Procedure for configuring structured logging using structlog with OpenTelemetry integration, request context propagation, and PII redaction.

## Prerequisites

- **Project structure**: Existing Python project with `pyproject.toml` and `src/` layout.
- **OpenTelemetry status**: Whether the project uses distributed tracing.

## Setup Workflow

### 1. Install Dependencies

```bash
uv add structlog
uv add opentelemetry-api opentelemetry-sdk  # If using distributed tracing
```

### 2. Create the Logging Configuration Module

Create `src/{package}/logging.py`:

```python
"""Structured logging configuration."""

from __future__ import annotations

import logging
import sys

import structlog

_REDACT_FIELDS = frozenset({
    "password", "token", "secret", "authorization",
    "email", "phone", "ssn", "credit_card",
    "api_key", "access_token", "refresh_token",
})

def configure_logging(
    log_level: str = "INFO",
    json_output: bool = True,
    enable_otel: bool = False,
) -> None:
    processors: list[structlog.types.Processor] = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        _redact_pii,
    ]
    if enable_otel:
        processors.insert(1, _add_otel_context)
    if json_output:
        processors.append(structlog.processors.JSONRenderer())
    else:
        processors.append(structlog.dev.ConsoleRenderer())

    structlog.configure(
        processors=processors,
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, log_level.upper())
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(file=sys.stdout),
        cache_logger_on_first_use=True,
    )

def _redact_pii(logger, method_name, event_dict):
    for key in event_dict:
        if key.lower() in _REDACT_FIELDS:
            event_dict[key] = "[REDACTED]"
    return event_dict

def _add_otel_context(logger, method_name, event_dict):
    from opentelemetry import trace
    span = trace.get_current_span()
    ctx = span.get_span_context()
    if ctx.is_valid:
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
    return event_dict
```

[Structured Logging](logging.md)

### 3. Create Request Context Middleware

For web applications, bind request-scoped context in middleware:

```python
import uuid
import structlog

async def logging_middleware(request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    structlog.contextvars.bind_contextvars(
        request_id=request_id, method=request.method, path=request.url.path,
    )
    logger = structlog.get_logger()
    logger.info("request_started")
    try:
        response = await call_next(request)
        logger.info("request_completed", status=response.status_code)
        response.headers["X-Request-ID"] = request_id
        return response
    except Exception:
        logger.exception("request_failed")
        raise
    finally:
        structlog.contextvars.unbind_contextvars("request_id", "method", "path")
```

### 4. Initialize at Application Startup

Call configuration before any log calls. Development uses `json_output=False` for readable console output; production uses JSON for machine parsing.

```python
from my_package.logging import configure_logging

settings = get_settings()
configure_logging(
    log_level=settings.log_level,
    json_output=settings.environment != "development",
    enable_otel=settings.otel_enabled,
)
```

[Configuration Setup](../../configuration/reference/configuration_setup.md)

### 5. Use Throughout the Codebase

Obtain loggers with `structlog.get_logger()`. Bind operation-specific context with `logger.bind()`:

```python
logger = structlog.get_logger()

async def process_order(order_id: str, items: list[dict]) -> None:
    log = logger.bind(order_id=order_id, item_count=len(items))
    log.info("processing_started")
    log.info("processing_completed")
```

### 6. Verify the Setup

| Verification | Method |
|-------------|--------|
| JSON format | Run with `json_output=True`, inspect stdout |
| Context propagation | Send request, verify `request_id` in all log lines |
| PII redaction | Log with a `password` field, verify `[REDACTED]` |
| OpenTelemetry | Send traced request, verify `trace_id` in output |

## Troubleshooting

**Duplicate log lines**: Use `structlog.stdlib.LoggerFactory()` when bridging third-party library output through structlog to avoid dual writes to stdout.

**Missing context in async tasks**: `asyncio.create_task()` copies context automatically. For thread pool executors, use `contextvars.copy_context()`.

## Related

- [Python Toolchain](../../toolchain/reference/tooling.md)
- [Project Setup](../../toolchain/reference/project_setup.md)
