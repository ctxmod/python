# Structured Logging

Use structlog with JSON rendering, OpenTelemetry correlation, and PII redaction.

Use structlog with JSON rendering for all application logging. Include standard correlation identifiers (trace_id, request_id, correlation_id) in every log record. Propagate request-scoped context via contextvars. Redact PII fields before serialization. Carry the same correlation identifier past the request scope — into Domain Event publication and consumption, and into Audit Record writes — so one business transaction is reconstructable across separately deployed runtimes.

This rule governs one layer only: the technical Telemetry layer. A log line is never the record of what the system committed. CLI tools and short-lived scripts may use `ConsoleRenderer` instead of JSON during development, and third-party loggers emitting standard `logging` records integrate through structlog's `ProcessorFormatter` without modification.

- the startup processor chain and its ordering, OpenTelemetry integration, the standard identifier set, contextvars propagation, PII redaction, performance budgets, and the worked correct/incorrect examples — [Structured Logging](../reference/logging.md)
- the Telemetry / Audit Record boundary and which layer owns which record — [Observability Layers](../reference/observability.md)
