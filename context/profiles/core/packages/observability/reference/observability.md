# Observability Layers

Observability layer principles covering the three layers (business audit, technical telemetry, agent-execution telemetry) with per-layer ownership, the non-substitutability rule, and shared-correlation-identifier propagation across requests, the Domain Event transport, and the Audit Record history.

A production system is observed through three distinct record layers:
business audit, technical telemetry, and agent-execution telemetry. Each
layer has exactly one owner, each serves a concern the others do not,
and none substitutes for another.

Observability records execution; it does not define business truth.

## The Three Layers

### Business Audit — the Audit Record History

The business audit layer is the Audit Record history: an append-only
record of what the system committed or attempted to commit — transition
identity, actor, timestamp, correlation and revision identifiers,
receipts, and approval references.

**Owner.** The application code path that commits or attempts a
business transition writes the Audit Record, in the same discipline as
the authoritative state store it transitions. The history's authority
is confined to being the immutable record of committed transitions: it
is never Workflow State, never the operational source for concurrency
or queries, and never rebuilt or sampled.

### Technical Telemetry

Technical telemetry is Telemetry (technical records): observational
records of execution behavior — latency, retries, errors, queue depth,
resource use — emitted as structured records (implementation instances:
structured log entries, tracing spans, metric series).

**Owner.** Every runtime emits its own technical Telemetry. The
application API runtime emits request-path records (authentication
outcomes, validation failures, response latency); the workflow
execution runtime emits async-processing records (retry outcomes, queue
depth, background-orchestration progress). Its authority is
observational only: Telemetry describes execution behavior and never
carries business state.

### Agent-Execution Telemetry

Agent-execution telemetry is Telemetry (agent-level records):
observational records of agent execution — model and tool invocation,
steps, evaluation results, decision outcomes.

**Owner.** The agent-execution layer (owned by agent-framework
doctrine (the ai module)) owns this layer's records and their emission; this profile
neither defines nor duplicates that content. A Python service that
hosts agent execution exposes the shared correlation identifier so the
agent-execution layer can attribute its records to the same business
transaction. Agent-level records are observational only — they never
determine whether a business transition committed.

## Non-Substitutability

The three layers serve distinct concerns, and none substitutes for
another:

- **Telemetry never defines business truth.** Whether a transition
  committed is answered only by the Audit Record history and the
  authoritative state store. No Telemetry record — technical or
  agent-level — is evidence of commitment.
- **Audit Records never serve as Telemetry.** The history is not
  sampled, not queried for operational health, and not a source of
  latency or error-rate signals; its evidentiary value depends on
  remaining append-only and complete.
- **Agent-execution Telemetry never substitutes for audit.** An agent
  must never be the sole authority for whether a business transition
  occurred.

## Position of the Structured-Logging Reference

Structured logging, tracing, and metrics as documented in this
profile's structured-logging reference constitute the
technical-telemetry layer — its Python implementation instances. The
structured-logging reference owns emission mechanics (processors,
correlation binding, context propagation, PII redaction); this
reference owns the layer model and the correlation discipline that
gives the records a business-transaction join key.

## Shared Correlation Identifier

One correlation identifier per business transaction flows across all
three observation surfaces, and it is the only key that joins them:

1. **Request propagation.** Ingress middleware accepts the correlation
   identifier from the inbound request or generates one, and binds it
   to the request context; every technical-Telemetry record in the
   request scope carries it.
2. **Message propagation.** When processing publishes a Domain Event to
   the Domain Event transport, the identifier travels in the payload;
   the consuming runtime binds it before any processing, so Telemetry
   and any further publications on the consumer side carry the same
   identifier.
3. **Audit propagation.** Every Audit Record write carries the
   identifier, tying the committed transition back to the request and
   the async chain that caused it.

The identifier is a typed common schema in the contract bundle — a
first-class contract type, not an ad-hoc string — so every boundary
that carries it agrees on its shape.

```text
on request(headers):
    cid = headers.correlation_id or generate()
    bind(cid)                               # request-scope records carry cid

on publish Domain Event(payload):
    payload.correlation_id = current(cid)   # the transport carries it forward

on consumer delivery(payload):
    bind(payload.correlation_id)            # async scope re-binds before work

on record Audit Record(transition):
    record.correlation_id = current(cid)    # history ties back to the request
```

The identifier must survive runtime boundaries: the application API
runtime and the workflow execution runtime are separately deployed,
and the correlation identifier is preserved through Domain Event
publication and Audit Record writes across both. Losing it at the
transport boundary splits one business transaction into unjoinable
halves. Bind it in the consumer before any work runs — not lazily at
first use — so no record is emitted without it.

## Where the Rest Lives

- The **cloud-platform doctrine's observability references** own the
  three-layer model as stack architecture — the canonical statement of
  the layering and the platform correlation plumbing. This profile
  links; it does not restate.
- The **agent-framework doctrine (the ai module)** owns agent-execution Telemetry —
  what an agent run records, where, and with what semantics.
- This profile owns the Python emission mechanics: the
  structured-logging reference (processors, correlation binding, PII
  redaction) and the correlation-identifier propagation discipline
  above.

## Related

- [Structured Logging](logging.md)
- [Contract-First Architecture](../../contracts/reference/contracts.md)
- [Logging Setup](logging_setup.md)
