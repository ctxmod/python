# Model Boundaries

Model boundary principles covering the model-class ladder, per-class authorship rules, mapping-not-mirroring between classes, the validated-DTO boundary flow, and typed interfaces at every untrusted boundary.

A codebase represents one set of business concepts through several distinct
model classes. The classes are frequently structurally similar, but
similarity does not make them the same model: each class serves a different
responsibility, evolves under different forces, and is owned by a different
layer. This reference owns the ladder of classes, the per-class rules, the
mapping discipline between classes, the normative boundary flow, and the
typed-interface prohibition that holds at every crossing.

## The Model-Class Ladder

The ladder, stated once:

```
External Contract DTO  ->  Application DTO  ->  Domain Model  ->  Persistence Model

Agent Decision Model  <->  Domain Model      (model-facing, both directions)
```

Reading the ladder:

- Each arrow is a **mapping** owned by the layer that receives the data.
  The mapping is explicit, typed, reviewable code — never an implicit
  field-copy, and never a class shared across two positions.
- The main chain runs from the wire inward to storage.
- The **Agent Decision Model** stands in a bidirectional relationship with
  the Domain Model: domain information is condensed into a bounded
  model-facing schema on the way in, and the structured decision is
  returned on the way out.

## Per-Class Rules

| Class | Represents | Rule |
|---|---|---|
| External Contract DTO | HTTP requests and responses on the wire | Generated from the contract where practical, with backward compatibility. Never hand-authored as a duplicate of the transport contract. |
| Application DTO | The application layer's internal request and result shapes | Hand-authored. The external contract is authoritative at the process boundary and is not forced to become the internal model merely to avoid duplication. |
| Domain Model | Business invariants and meaningful domain concepts | Handwritten. Never generated from the transport contract merely because fields match. |
| Persistence Model | Storage requirements: indexes, denormalized fields, projection metadata, serialization optimizations | Must not leak storage concerns into domain semantics. |
| Agent Decision Model | Information presented to a model, and the structured decision it returns | Bounded, typed, minimal, validated, and explicit about uncertainty and permitted actions. |

Three of the rules carry obligations beyond the table row.

**External contract boundaries are authoritative for process and
integration boundaries.** The internal layers keep whatever distinct models
their responsibilities need; the contract is not collapsed into the
internal model to avoid duplication, and internal shapes are not projected
onto the contract. At the HTTP surface, route handlers receive the
generated DTOs and map them to typed Commands — the five-step route
contract is owned by the FastAPI reference in the api profile.

**Persistence serves the domain, never the reverse.** Storage-driven
fields — index hints, denormalized copies, projection metadata,
serialization shortcuts — live in Persistence Models and in the
persistence/read-model store, not in Domain Model definitions. The same
discipline bounds the read side: a Projection is a derived,
non-authoritative representation, and what a projection function writes is
validated at the boundary (below).

**The model-facing class is effective only after application validation.**
A model proposes a typed proposal/decision model; it never directly owns
permissions, state transitions, or irreversible effects. The proposal
becomes effective only after the application layer has validated
authorization, idempotency, and the `workflow_revision` and
`action_revision` preconditions. The design of agent-facing schemas — the
presentation shape, the permitted-action vocabulary, uncertainty signaling
— is owned by the agent-framework doctrine (part of the cloud-platform doctrine); this
profile owns the boundary rule, not the schema design.

## Mapping, Not Mirroring

- **Similarity is not identity.** The classes may be structurally similar,
  but similarity does not make them the same model. Two positions share a
  model only where their semantics genuinely coincide — the test is
  semantic, never a field-list comparison.
- **Mirroring is the failure mode the ladder exists to prevent.** Copying
  the transport schema layer by layer into the internal classes couples
  every internal evolution decision to the wire contract: a
  compatibility-motivated contract change would silently become a domain
  semantics change. The schema-coupling warning cuts both directions —
  internal storage or presentation shapes are not mirrored outward onto
  the contract either.
- **Generation belongs to contract boundaries.** Code generation produces
  transport types — HTTP request/response DTOs and message-bus message
  classes — and nothing else. The generated-code authority properties and
  the handwritten-versus-generated division are owned by the
  contract-first architecture reference.
- **The no-handwritten rule is scoped by the view-model allowance, not
  blanket.** Never hand-author a model that duplicates an external
  transport contract. The allowance: view models, local UI or form state,
  and composition models are not contract duplication — they present or
  assemble information rather than restating the wire shape — and remain
  legitimate handwritten code.

## The Boundary Flow

Data entering the process follows one normative flow:

```
raw inbound record        # bytes, or a parsed-but-unvalidated structure,
                          # at a process boundary
  -> validated DTO        # generated transport DTO at external boundaries;
                          # hand-authored internal DTO elsewhere. Validation
                          # runs here — strict, constrained, total.
  -> domain object        # mapped by the application layer; invariants
                          # hold from this point on
```

- No stage is skipped: a raw inbound record never reaches domain logic, a
  validated DTO is never re-used as the domain object, and a domain object
  never becomes an outbound payload without a DTO mapping step.
- The outbound direction reverses the flow: domain object → application
  DTO → transport DTO → serialized payload.
- Where the inbound boundary is the HTTP API, the first stage is the
  framework's: validation runs against the generated DTOs before any
  handler body executes, and the body never parses raw JSON (the route
  contract in the FastAPI reference).

## No Untyped Dicts Between Components

`dict[str, Any]` — or any untyped dictionary or `Any`-annotated mapping —
never serves as the interface between components:

- A component never returns an untyped dict to another component; it
  returns a typed model: a generated DTO, an Application DTO, a domain
  object, or a typed Command.
- A component never accepts an untyped dict in its signature.
- Where a library hands back untyped data — parsed JSON, a driver row
  mapping, a payload from a legacy integration — the untyped value is
  confined to the adapter that immediately validates it into a typed
  model, and never escapes that adapter.
- JSON parsing at HTTP or message-bus boundaries is never handwritten; it
  is the framework's job against the generated DTOs (the never-handwritten
  list in the contract-first reference).

The prohibition is about the shape of interfaces, not about where
validation runs. Internal calls between trusted modules need no runtime
validation — that distinction is owned by the type-safety reference — but
their interfaces are still typed models, not dictionaries.

## Pydantic at Every Untrusted Boundary

Pydantic models validate every boundary at which data of unproven shape
enters:

| Boundary | What is validated |
|---|---|
| HTTP requests and responses | Transport DTOs at the API surface |
| message transport payload decoding | Incoming message and Domain Event payloads, into typed message classes |
| read-model/Projection writes | The shape a projection function materializes, before the read-model store accepts it |
| model-facing tool I/O | A tool's argument and result schemas |
| configuration loading | Settings models at startup |

Boundary models are strict (`ConfigDict(strict=True)`) and constrained,
however they were produced: hand-authored schemas carry strict mode
directly, and generated DTOs inherit it from the contract's generation
configuration. The language-level validation stack — basedpyright strict
static checking, Pydantic boundary validation, TypeGuard runtime guards —
is owned by the type-safety reference; the API-side schema disciplines
(strict-mode detail, schema separation, constraints, validators) by the
api validation reference.

## Where the Rest Lives

- The **contract-first architecture reference** owns contract technology
  per boundary, generated-code authority, the handwritten-versus-generated
  division, and the drift discipline.
- The **api validation reference** owns the generated-transport-versus-
  hand-authored-internal schema discipline on the API surface.
- The **type-safety reference** owns the enforcement stack that backs
  these boundaries.
- The **cloud-platform doctrine** owns agent-facing schema design and platform
  contract details; this profile links, and does not restate them.

## Related

- [Contract-First Architecture](../../contracts/reference/contracts.md)
- [Type Safety](type_safety.md)
- the api profile's *FastAPI Patterns* reference (`profiles/api/packages/fastapi/reference/fastapi.md`)
- the api profile's *Request and Response Validation* reference (`profiles/api/packages/validation/reference/validation.md`)
