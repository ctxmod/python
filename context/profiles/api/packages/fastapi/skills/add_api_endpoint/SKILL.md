---
name: add_api_endpoint
description: "Add an HTTP operation contract-first — change the contract source, lint and bundle, regenerate every output, write the thin route adapter, and prove the operation with conformance tests including the idempotency and stale-precondition pair. Use when: add endpoint; new api endpoint; add an operation; contract-first endpoint; extend the api; new route."
---

# Add an API Endpoint

Add an HTTP operation in the order the contract-first discipline requires: contract source first, generated output second, handwritten adapter third, conformance tests last.

## Purpose

Use when a new HTTP operation — a query, a Command, an approval-style action — must exist. The sequence below is the operational form of rules already stated elsewhere; this skill sequences them and does not restate them. Every "why" belongs to the reference material linked at the end, and every step is a pointer into it rather than a copy of it.

The one thing this skill exists to prevent is the reversed order: writing the handler first and retrofitting the contract afterwards. That order produces a contract describing what was built rather than a contract deciding what will be built, and the compatibility decision is then made silently, by an implementation detail.

## Workflow

**1. Change the contract source.** Add the operation to the modular contract source under `contracts/api/` — a path item, a stable operation identifier, request and response schemas, the error responses it can return. Reuse the common schema set (identifiers, pagination, timestamps, correlation identifiers, the error model, revision-precondition objects) instead of introducing near-duplicates. A mutating operation accepts an idempotency key; an operation that acts on a specific proposed action carries its revision preconditions. Advance the `contract_revision` for the evolution step. Review this change before writing implementation code — that review is the whole point of the ordering.

**2. Lint and bundle.** Run the contract lint, then resolve the modular source into the single reviewed bundle. A lint finding is a contract failure: fix the contract source, never the checker's configuration.

**3. Regenerate every output.** Run the pinned generation recipes so the request and response DTOs — and any bindings the change touches — come from the freshly bundled contract. Never hand-write the transport types and never hand-edit what the generator produced. Commit the regenerated output together with the contract change, in one change set.

**4. Write the thin route adapter.** The handler is an adapter, not a place for logic: it accepts the generated DTO, maps it to a typed application input, calls one application service, maps the typed result back to the generated response DTO, and translates typed failures into the problem-details error shape. Nothing else belongs there — no storage query construction, no model invocation, no prompt text. The route-contract detail and the framework binding are owned by the framework sub-profile's reference material; consult it there rather than improvising the handler shape here.

**5. Prove it with conformance tests.** Add the conformance tests for the new operation: request and response behavior against the bundled contract, and — for a mutating operation — both halves of the pair, an idempotency test and a stale-precondition test. Either half alone leaves the operation half-guarded. Run the contract-testing procedure end to end before landing.

## Definition of Done

- [ ] The contract source carries the operation, and the contract change was reviewed before the implementation.
- [ ] No handwritten request or response type exists for the operation outside generated output.
- [ ] The generated-output drift check is clean, and contract plus generated output land in one change set.
- [ ] The handler maps and delegates only; typed failures render as problem details.
- [ ] The operation has conformance tests, and a mutating operation has both an idempotency test and a stale-precondition test.

## Context

### Contract-first architecture and generated-code authority

[Contract-First Architecture](../../../../../core/packages/contracts/reference/contracts.md) (required reading)

### Model classes, mapping, and typed boundaries

[Model Boundaries](../../../../../core/packages/typing/reference/model_boundaries.md) (required reading)

### Typed errors, problem details, and the three idempotency levels

[Error Handling and Resilience](../../../../../core/packages/errors/reference/errors.md) (required reading)

### The contract gate

[Contract Testing Validation](../../../../../core/packages/contracts/reference/contract_testing.md) (required reading)

### Supporting material

[Project Setup](../../../../../core/packages/toolchain/reference/project_setup.md)
[Testing Philosophy](../../../../../core/packages/testing/reference/testing.md)
[Type Safety](../../../../../core/packages/typing/reference/type_safety.md)
