# Error Handling

Raise typed application errors with stable codes, return RFC 9457 problem details at the HTTP boundary, and keep the three idempotency levels distinct.

Raise typed application errors, each carrying a stable application error code — never a bare string and never an untyped raise across a component boundary. Serialize every failure at the HTTP boundary as one RFC 9457 problem-details object with the stable code as an extension member. Decide retry from the error class, never from a message substring. Keep the three idempotency levels distinct: Command idempotency, Domain Event delivery deduplication, and action idempotency. A stale revision precondition fails with a typed 409 Conflict, never a generic 500 and never a best-effort overwrite.

The stable code, not the class name, is the contract. CLI tools and script tools have no HTTP boundary and map the same typed taxonomy onto per-class exit codes and structured stderr diagnostics instead. Third-party exceptions are normalized into the taxonomy inside the adapter that raised them — no foreign error class crosses an adapter boundary.

- the typed-error and problem-details worked examples, the three idempotency levels and their identity formulas, stale-precondition conflicts, retry discipline, adapter-owned normalization, and the exception-handler and idempotency/precondition test-pairing tooling — [Error Handling and Resilience](../reference/errors.md)
- the contract source that owns the error-code set and the problem-details error model as a common schema — [Contract-First Architecture](../../contracts/reference/contracts.md)
