# Error Handling and Resilience

Error-handling and resilience principles covering the typed application error taxonomy with stable error codes, RFC 9457 problem details at the HTTP boundary, adapter-owned error normalization and the never-infer-success rule, the three distinct idempotency levels with their identity formulas, and retry discipline with the typed stale-precondition conflict.

Errors are part of the contract. A service's callers — clients, agents,
adapters, tests — make decisions from error responses, so the error
surface is designed with the same discipline as the success surface:
typed failures, stable codes, one wire shape, and explicit retry
semantics. This reference owns the Python-side error-handling law: the
typed application error taxonomy, the problem-details error contract at
the HTTP boundary, adapter-owned error normalization, the three
idempotency levels, and the retry and conflict discipline built on them.

## Typed Application Errors

Failures inside the application are typed values, not strings. The
taxonomy is enumerated code — a closed set of typed error classes shared
across the codebase — and every class carries a **stable application
error code**: a machine-readable string that names the failure class and
outlives refactors. Renaming an exception class never renames its code;
codes are part of the public contract (they appear in the contract's
error model) and are retired deliberately, never silently redefined.

The taxonomy partitions failures by what the caller can do about them:

| Failure class | The caller's situation |
|---|---|
| Validation failure | The request is malformed or violates constraints; fix the request |
| Authorization failure | The principal may not perform the operation |
| Not found | The addressed entity does not exist |
| Stale precondition | The revision precondition no longer holds; reload state and resubmit |
| Dependency failure | An external system failed; may be retryable |
| Internal failure | An invariant or implementation error; not the caller's to fix |

Two rules hold across the taxonomy:

- An error crossing a component boundary is always a typed instance
  carrying the stable code and structured context — which
  `workflow_revision` was expected, which key was replayed, which
  dependency failed — never a bare string or an untyped raise. This is
  the error-side face of the typed-interfaces law in the
  model-boundaries reference.
- The error class determines retry semantics. Retry policy is a
  decision about a type, never a match on a message substring.

## Problem Details at the HTTP Boundary

The HTTP error contract shape is RFC 9457 problem details (the IETF
problem-details standard), carrying the stable application error code as
an extension member. One shape serves every failure — request
validation, authorization, stale precondition, dependency failure — so
clients parse a single error schema:

| Member | Meaning |
|---|---|
| `type` | A stable URI identifying the failure class, resolvable to its documentation |
| `title` | A short human-readable summary, stable per `type` |
| `status` | The HTTP status code |
| `detail` | A human-readable explanation of this occurrence |
| `instance` | The operation path this occurrence happened on |
| `code` (extension) | The stable application error code from the taxonomy |

A stale-precondition conflict renders as:

```text
HTTP/1.1 409 Conflict
type:     https://<app>.example/errors/stale-precondition
title:    Stale precondition
status:   409
detail:   workflow_revision 12 no longer current; current is 14
instance: /workflows/{id}/approvals
code:     STALE_PRECONDITION
```

The shape is owned by the contract: the error model is one common schema
in the contract bundle, and the code set evolves under the contract's
additive compatibility rules (the contract-first architecture reference
owns that discipline). Status mapping is deliberate and small — 5xx is
reserved for genuine internal failure, and a stale precondition is a 409
Conflict with the typed body, never a generic 500. Registering exception
handlers is mechanics owned by the api routing reference; normalizing
Pydantic's violation output into this shape is owned by the api
validation reference.

## Adapter-Owned Error Normalization

Every external side effect — a call into a task system, a notification
system, any system the application changes — goes through a typed
application adapter, and the adapter owns the full responsibility list:

- **request construction** — the adapter is the only code that knows the
  target system's request shape;
- **authentication** — credentials are resolved and applied inside the
  adapter, never threaded through domain code;
- **external idempotency where supported** — where the target system
  offers an idempotency mechanism, the adapter supplies it;
- **response validation** — the raw response is validated into a typed
  result before anything downstream sees it;
- **receipt creation** — a validated outcome becomes a durable receipt;
- **error normalization** — foreign failures are mapped into the typed
  taxonomy.

The adapter contract, as a flow:

```text
adapter.perform(effect) -> receipt:
    request  = build(effect)            # request construction
    response = send(request)            # authentication + external idempotency
    result   = validate(response)       # response validation; never inferred
    receipt  = to_receipt(result)       # receipt creation
    return receipt                      # caller reconciles into Workflow State
```

Three laws complete the contract:

- **No raw foreign error crosses the adapter boundary.** Foreign status
  codes, response bodies, and exception types are mapped into the typed
  taxonomy inside the adapter; domain code sees typed failures only.
  Whether a foreign outcome is a validation failure or a dependency
  failure is the adapter's judgment, encoded once — never re-derived at
  each call site.
- **Never infer success.** The component issuing a call never infers
  success from having issued it: success is established by response
  validation and recorded as a durable receipt that reconciles into
  Workflow State and the Audit Record history. A model never infers
  success merely because it issued a call — the workflow owns when a
  side effect is allowed, the agent may propose it, and the adapter
  reports whether it actually occurred.
- **The side effect never runs inside the transition that authorizes
  it.** The authorizing state transition and a durable intent record
  commit together; a dispatcher delivers the intent; the adapter
  executes and creates the receipt; the receipt reconciles back. The
  pipeline mechanics — durable intent records, dispatch, deduplication
  record storage — are owned by the cloud-platform doctrine's state and
  reliability references; this profile consumes the invariant and owns
  the adapter contract above.

## Three Idempotency Levels

Idempotency is three distinct rules with three distinct identity
formulas — not one bundled concept. The levels complement one another
and are never interchangeable; each guards a different boundary.

**1. Command idempotency — deduplicating externally issued Commands.**
Identity formula: **principal + operation + idempotency key**. Every
mutating endpoint accepts an `Idempotency-Key` header; matched against
this identity, repeated submission returns the original semantic outcome.
A submission replayed with the same identity — principal, operation,
key — receives the stored outcome of the first execution, success or
failure as it originally was, and produces no second effect. A key
reused with a different operation or payload is a validation failure,
not a replay.

**2. Domain Event delivery deduplication — deduplicating deliveries.**
Identity formula: **the immutable `domain_event_id`, scoped to its
consumer**. A redelivered Domain Event hits the consumer's
deduplication record and produces no repeated business effect. The
consumer records the identifier durably before performing any side
effect, so a crash between delivery and processing cannot turn a
redelivery into a duplicate effect.

**3. Action idempotency — preventing double execution.**
Identity formula: **`action_id`, valid against its `workflow_revision`
precondition**. A consequential action never executes twice merely
because an agent retries or a Domain Event is redelivered: execution is
keyed on `action_id`, and the precondition fails loudly when the state
the action was proposed against has moved.

The delivery guarantee these rules build is **application-level
effectively-once side-effect semantics where required, implemented
through durable deduplication records and idempotent adapters**. The
transport is never described as providing exactly-once delivery — no
message transport provides it, and treating one as if it did removes the
very deduplication the application depends on.

Workflow concurrency is not a fourth idempotency level: the
`workflow_revision` is an optimistic-concurrency precondition that the
levels above consume, and revision identifiers act as preconditions in
idempotency contexts — never as deduplication keys. The deduplication
keys are `domain_event_id` and `action_id`. Where the durable records
live and how they are stored is state-store mechanics owned by the cloud-platform
doctrine; this profile owns the contract expression above.

## Stale Preconditions and the Typed Conflict

- Mutating requests carry their `workflow_revision` precondition, and
  the `action_revision` in addition when the request approves or
  executes a specific proposed action. The `action_revision` is the
  identity of one specific proposed action; where it is computed from
  the action's content, that computation is the generic **precondition
  hash** — a mechanism, not a separate identifier.
- On mismatch, the operation fails with a **409 Conflict** carrying the
  typed problem-details body — the expected and current
  `workflow_revision` — never a generic 500 and never a best-effort
  overwrite. A stale submission fails loudly rather than risking an
  incorrect approval.
- The approval lifecycle — what a stale approval may and may not
  authorize, and that a stale approval never authorizes a newer action —
  is consumed from the locked approval decision record, whose canonical
  statement and state machine live in the cloud-platform doctrine's state
  reference; this profile does not restate the machine.
- The client's remedy is prescribed: reload current state, then
  resubmit against a fresh `workflow_revision` precondition — and the
  action's fresh `action_revision` when the request targets a specific
  proposed action. A stale precondition is never retried unchanged — it
  is an answer, not a transient failure. Rendering staleness is a
  separate concern on `projection_revision` and protects rendering,
  never command correctness.
- Clients and UIs handle connection, loading, and error states without
  implying an action occurred when it did not — the never-infer-success
  rule on the consumption side.

## Retry Discipline

- Retry is a decision about a typed error class. Transport timeouts and
  transient dependency failures are retryable; validation,
  authorization, not-found, and stale-precondition failures are answers
  and are never retried as-is.
- A retry preserves identity. A retried Command reuses its idempotency
  key — a new key is a new Command; a redelivered Domain Event is caught
  by the `domain_event_id` deduplication record; a retried action
  re-checks its `action_id` precondition. Retry safety comes from these
  identity formulas, not from chance.
- Retries are bounded. Exhaustion lands as a typed dependency failure —
  never a silent drop, never an unbounded loop.
- Backoff policy and delivery-pipeline dispatch are owned by the cloud-platform
  doctrine's reliability reference; this profile owns the semantic
  rules above.
- Obligation, with mechanics owned by this profile's testing reference:
  every mutating API operation carries idempotency handling and
  stale-precondition handling, and both are exercised by tests.

## Rule Companion: Rationale, Examples, Exceptions, and Tooling

*Relocated verbatim from the `errors` rule (during an earlier rule-slimming pass). Heading levels demoted one level; rule-local link pointers removed. No other change.*

### Rationale

Callers — clients, adapters, tests — make decisions from error responses, so the error surface needs the same discipline as the success surface. A stable code outlives the class rename that would otherwise break every consumer branching on it, which is why the code, not the class name, is the contract. The code set and the problem-details error model are a common schema in the contract source, so applying this rule without the contract-first reference produces codes that live only in Python and are invisible to the consumers that must branch on them. One wire shape means clients parse one schema instead of a per-endpoint dialect. Message-substring matching turns retry policy into an accident of wording; the typed class makes it a decision about a kind of failure. The three idempotency levels guard three different boundaries with three different identity formulas — collapsing them into one "idempotency" concept leaves at least two boundaries unguarded, because the identity that deduplicates a submitted Command cannot deduplicate a redelivered Domain Event. Where a web-framework sub-profile is in scope, it owns the mechanics of normalizing framework validation output into this shape.

### Examples

**Correct**:

```python
class AppError(Exception):
    code: str            # stable; survives class renames
    status: int

class StalePrecondition(AppError):
    code, status = "STALE_PRECONDITION", 409

@app.exception_handler(AppError)
async def handle(request: Request, exc: AppError) -> JSONResponse:
    return JSONResponse(
        status_code=exc.status,
        media_type="application/problem+json",
        content={
            "type": f"https://<app>.example/errors/{exc.code.lower()}",
            "title": exc.title,
            "status": exc.status,
            "detail": str(exc),
            "instance": request.url.path,
            "code": exc.code,
        },
    )

def should_retry(exc: AppError) -> bool:
    return isinstance(exc, DependencyFailure)   # class, not message
```

**Incorrect**:

```python
# Untyped raise, no stable code, ad-hoc envelope, wrong status.
raise Exception("workflow revision mismatch")

return JSONResponse(status_code=500, content={"error": "conflict"})

def should_retry(exc: Exception) -> bool:
    return "timeout" in str(exc)   # message matching
```

### Exceptions

CLI tools and script tools have no HTTP boundary: they map the same typed taxonomy onto per-class exit codes and structured stderr diagnostics instead of problem details. Third-party exceptions are normalized into the taxonomy inside the adapter that raised them; no foreign error class crosses an adapter boundary. Framework-raised validation failures are normalized into the same shape rather than re-raised as typed application errors at their origin.

### Tooling

Register one exception handler per typed base class rather than per endpoint. Every mutating operation carries both an idempotency test and a stale-precondition test; the contract-conformance suite fails when either half of the pair is missing. Emit the stable code as a structured field on the error record so failure classes aggregate without parsing messages.

## Where the Rest Lives

- The **contract-first architecture reference** owns the error model as
  a common contract schema and its `contract_revision` evolution.
- The **api validation reference** owns normalizing Pydantic's violation
  output into the problem-details shape.
- The **api routing reference** owns exception-handler registration and
  the handler-to-response mechanics.
- The **observability reference** owns the boundary between Telemetry
  error records and the Audit Record history, and the correlation
  identifier that ties a retry chain to one business transaction.
- The **cloud-platform doctrine** owns the deduplication-record mechanics,
  the outbox/inbox pipeline, the approval state machine's canonical
  home, and retry/backoff dispatch. This profile links; it does not
  restate them.
- The **testing reference** owns the idempotency and stale-precondition
  test tiers that prove these rules hold.

## Related

- [Contract-First Architecture](../../contracts/reference/contracts.md)
- [Model Boundaries](../../typing/reference/model_boundaries.md)
- [Observability Layers](../../observability/reference/observability.md)
- the api profile's *Request and Response Validation* reference (`profiles/api/packages/validation/reference/validation.md`)
- the api profile's *Routing and Handler Design* reference (`profiles/api/packages/fastapi/reference/routing.md`)
