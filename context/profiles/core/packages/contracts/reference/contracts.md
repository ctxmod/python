# Contract-First Architecture

Contract-first architecture principles covering contracts as source, contract technology per boundary, generated-code authority, and the handwritten-versus-generated division.

Write durable business logic once; generate the types, clients, request
validation, mocks, and Domain Event bindings from the contracts. The API
contract and the Domain Event contracts are source code: they are reviewed
before implementation code, and a change flows contract first, then
regeneration, then implementation. The contract source is the single
source of truth for its boundary; no component deserializes untyped JSON
at a process boundary.

## Contracts Are Source Code

- API and Domain Event contracts live in the repository, are code-reviewed
  like code, and change through the same review path as implementation.
- A needed behavior change starts in the contract: edit the contract,
  regenerate the derived artifacts, then write the small amount of
  handwritten code the change requires.
- Every API and Domain Event contract carries a `contract_revision` — the
  compatibility and schema-evolution marker that consumers and Audit
  Records reference.
- A component may not deserialize untyped JSON at a process boundary:
  payloads cross boundaries as generated types or validated models, never
  as raw dictionaries.

## Contract Technology per Boundary

Use the contract technology best suited to each explicit integration
boundary. Never force one technology onto every boundary for uniformity.

| Boundary | Canonical contract | Generated for Python | Serialization | Owner |
|---|---|---|---|---|
| HTTP command/query API (clients to `<app>`) | OpenAPI 3.2 | Pydantic v2 request/response DTOs | JSON over HTTPS | the API domain |
| Asynchronous Domain Events crossing process or service boundaries | Protobuf, namespaced `<repo>.<domain>.v<N>` | message classes for producers and consumers | binary Protobuf | the domain that publishes the Domain Event |
| Internal typed interfaces — domain Commands, layer-internal Domain Events, persistence DTOs, tool and configuration schemas | the defining layer's own typed interfaces (Protocols, Pydantic models) | nothing; owned in place | in-process typed invocation | the layer defining the responsibility |

Three rules govern the table:

1. **Per-boundary suitability.** OpenAPI is not forced onto the
   asynchronous boundary merely for uniformity, and Protobuf is not forced
   onto a client-facing JSON API. Pick the technology that is strongest
   at each boundary.
2. **Internal interfaces stay internal.** Internal typed interfaces are
   owned by the layer that defines the responsibility and are not
   automatically expressed as OpenAPI or Protobuf models.
3. **Canonical-to-deployable.** The canonical Domain Event contract
   generates the exact deployable message-bus topic schema, which
   generates the language bindings. A hand-transformed deployable schema
   with no explicit generation mechanism is prohibited — that is two
   contracts claiming one boundary. The envelope, topic-schema, and
   evolution specifics are owned by the cloud-platform doctrine's contracts references; this profile links to them and does not restate them.

## Boundary Ownership, Python Side

Each contract has exactly one owner, and the owner decides its evolution:

- The **API domain** owns the OpenAPI 3.2 contract: endpoint definitions,
  operation identifiers, request/response schemas, the error model,
  compatibility rules, and security requirements.
- The **publishing domain** owns each Protobuf Domain Event contract:
  message identity, payload structure, field compatibility, and evolution
  rules.
- The **defining layer** owns its internal typed interfaces.
- Cross-language duplication is resolved by generation, not by parallel
  authorship: this profile consumes the generated Python bindings, and
  the browser-client doctrine (the typescript module) owns the generated clients, query hooks,
  and mocks generated from the same contract.

## Generated-Code Authority

Generated artifacts carry exactly four properties:

- **Reproducible.** Every generated artifact is regenerable from its
  contract source by a deterministic generation step; the same inputs
  produce the same output.
- **Never authoritative.** The contract source is the single source of
  truth; a generated artifact never becomes one.
- **Committable.** Generated output may be committed for ergonomic and
  build reasons. Committing it does not make it source.
- **Always regenerable and drift-checked.** CI regenerates the output and
  rejects drift: committed generated output must equal fresh
  regeneration.

Two prohibitions follow. Generated code is never manually edited. And the
path from generated code through manual modification to a new contract is
prohibited — a needed change goes into the contract source and is
regenerated. ("Disposable" is deliberately not the word for generated
output: committable and regenerable are the operative properties, and
calling committed output disposable misstates both.)

The generation pipeline, in semantics: the modular contract sources are
linted, bundled into one portable reviewed artifact, and every
language output is generated from that same bundle. The toolchain is
pinned in lockfiles and invoked through `uv run` or `uvx`. The resolved
bundle is retained as a reviewed build artifact. Concrete tool selection
and pinning live in this profile's packaging and codegen-toolchain
reference.

```
contracts/api/            (modular OpenAPI 3.2 source)
  -> lint
  -> bundle   -> gen/api/bundle/openapi.yaml   (reviewed artifact)
  -> generate -> gen/api/python/               (Pydantic v2 DTOs)
CI: regenerate into a temporary directory; fail if it differs from gen/
```

## Handwritten versus Generated

Keep handwritten code intentionally small.

**Handwritten:**

- the state-transition guard and idempotency service;
- a small set of application command handlers;
- a few side-effect adapters, all implementing the same shared receipt
  model (receipt semantics are owned by the cloud-platform doctrine's state
  reference; this profile consumes the shape);
- read-model projection functions that materialize Projections;
- composition code that assembles services and configuration.

**Never handwritten:**

- transport request/response types (HTTP and message-bus);
- interfaces duplicated across languages;
- JSON parsing at HTTP or message-bus boundaries;
- mock data shapes;
- Domain Event serialization and deserialization;
- API reference documentation.

The no-handwritten rule is precise, not blanket: never hand-author models
that duplicate an external transport contract — but view models, local
UI or form state, and composition models are not contract duplication
and remain legitimate handwritten code. Generation belongs to contract
boundaries; internal models are shared across layers only where their
semantics genuinely coincide. The model-class ladder and the
generated-DTO discipline are owned by this profile's model-boundaries
reference.

## Contract-First Repository Layout

The layout below composes with the src/ layout, which stays authoritative
for every Python package and application (see the project-structure
reference). It applies when a repository hosts multiple deployables or a
shared contract surface; a single-package project remains a plain src/
project.

```
<repo>/
  contracts/
    api/                          # OpenAPI 3.2 source graph (modular)
      openapi.yaml                # small root; imports path items
      paths/<domain>/
      schemas/common/
      schemas/<domain>/
      parameters/
      responses/
      security/
    domain-events/                # Protobuf sources + tool configuration
      <repo>/<domain>/v<N>/       # one .proto file per Domain Event definition
  gen/                            # generated output; never hand-edit
    api/
      bundle/openapi.yaml         # bundled, reviewed contract artifact
      python/                     # Pydantic v2 request/response DTOs
    domain-events/
      python/                     # message classes
  apps/
    <service>/                    # one deployable application per entry
  packages/
    one per package               # shared source packages (domain, projections)
  tests/
```

Each application and source package keeps the src/ layout internally:

```
apps/<service>/src/  # holds the application package
  main.py            # application entrypoint (framework wiring)
  routers/           # thin adapters only; see the FastAPI route contract
  services/          # application services: state, authorization, side effects
  config/            # settings and logging setup
```

Deployment, infrastructure, and platform wiring around these directories
are owned by the cloud-platform doctrine; this profile stops at the
application boundary.

## Modular Contract Source

Organize the OpenAPI source so review scales with the contract:

- a small root document that imports path items; every reusable schema
  is a single file; only local relative references;
- every operation carries a stable operation identifier;
- resource-oriented paths, with explicit custom actions reserved for
  Commands;
- a common schema set shared across domains: resource identifiers,
  pagination tokens and limits, timestamps, correlation identifiers, the
  error model (problem-details shaped, with a stable application error
  code), revision-precondition objects, and references to Audit Records
  and receipts;
- `additionalProperties: false` on Command payloads;
- evolution defaults: additive optional fields are the default compatible
  path; never change a field's meaning; never reuse a removed field
  name; each evolution step advances the `contract_revision`.

Every mutating operation accepts an idempotency key, and approval-style
Commands carry their `workflow_revision` and `action_revision`
preconditions so a stale action fails with a typed conflict response
instead of producing an incorrect effect. The typed error contracts and
idempotency semantics are owned by this profile's error-handling
reference.

## Drift and Generated-Diff Semantics

A contract change must flow through the whole chain before it lands:

```
contract source change
  -> lint + bundle
  -> breaking-change check (asynchronous contracts)
  -> regenerate every language output
  -> generated-diff check
  -> contract-conformance tests
```

The semantics, not the job mechanics:

- **The generated-diff check proves committed generated output is
  current.** Regenerate into a temporary location and compare against
  the committed `gen/` tree; any difference is a build failure, not a
  warning.
- **Conformance tests prove the service matches the contract.** The
  application's request/response behavior is tested against the bundled
  artifact, so an implementation cannot drift from the contract even
  when the generated types happen to line up.
- **Breaking a consumer is a contract decision.** A change that breaks
  compatibility is resolved through the evolution rules and
  `contract_revision`, not by patching consumer code around it.

The CI gate list and test-tier mechanics are owned by this profile's
testing reference.

## Definition of Done

A contract-first codebase is not done until:

- [ ] No handwritten HTTP request or response type exists outside
      generated output.
- [ ] No hand-shaped message payload object exists outside fixtures
      generated from the Protobuf contracts.
- [ ] Every mutating API operation has an idempotency test and a
      stale-precondition test.
- [ ] Generated output matches the pinned contract toolchain in CI —
      regeneration is byte-stable and drift-free.

## Rule Companion: Rationale, Examples, Exceptions, and Tooling

*Relocated verbatim from the `contracts` rule (during an earlier rule-slimming pass). Heading levels demoted one level; rule-local link pointers removed. No other change.*

### Rationale

Generated artifacts carry exactly four properties: reproducible from their contract source, never authoritative, committable for build ergonomics, and always regenerable under a CI drift check. A hand edit inside generated output breaks all four at once — the next regeneration silently discards it, and until then the committed artifact and the contract disagree about the same boundary. Reviewing the contract before the implementation places the compatibility decision where it is cheapest to make, and the `contract_revision` records that decision for consumers and Audit Records. Untyped JSON at a process boundary reintroduces exactly the unvalidated shape the contract exists to eliminate, and it does so where no static check can see it.

### Examples

**Correct**:

```python
# Generated DTOs are consumed as-is; shapes change only in the contract source.
from gen.api.python.models import CreateItemRequest, ItemResponse

async def create_item(payload: CreateItemRequest) -> ItemResponse:
    return await service.create(payload)
```

```just
# Regeneration is a recipe; drift is a build failure, not a warning.
contract-generate:
    uv run redocly bundle contracts/api/openapi.yaml --output gen/api/bundle/openapi.yaml
    uv run datamodel-codegen --input gen/api/bundle/openapi.yaml --input-file-type openapi --output gen/api/python/models.py --output-model-type pydantic_v2.BaseModel
```

**Incorrect**:

```python
# gen/api/python/models.py -- hand-edited generated output.
class ItemResponse(BaseModel):
    id: int
    note: str = ""   # added by hand because the contract was not updated

# Untyped decode at a process boundary: no generated type, no validation.
async def handle(raw: bytes) -> None:
    payload = json.loads(raw)
    await service.apply(payload["item_id"])
```

### Exceptions

A single-package project with no external contract surface has no `contracts/` tree, and this rule does not apply to it. Generated output may be excluded from formatters and linters — exclusion is configuration, not editing. A generator defect is resolved by re-pinning or replacing the generator in the lockfile, never by patching its output. View models, local form state, and composition models are not transport-contract duplicates and remain legitimate handwritten code.

### Tooling

Run the gate through the pinned toolchain: `just contract-lint`, `just contract-generate`, `just contract-drift-check`. CI regenerates into a temporary directory and fails on any difference from the committed `gen/` tree. Never add `gen/` to the ignore file — the drift check needs the committed tree to compare against.

## Where the Rest Lives

- The **cloud-platform doctrine's contracts references** owns the canonical
  contract-first statement, contract technology per boundary at platform
  level, the OpenAPI HTTP boundary detail, Protobuf Domain Event
  contracts (envelope, topic schemas, evolution rules), and the
  platform code-generation pipeline. This profile links; it does not
  restate.
- The **browser-client doctrine (the typescript module)** owns generated browser clients,
  query hooks, and mocks.
- This profile owns the Python-side specializations: generated Pydantic
  DTOs, the thin-adapter route contract (FastAPI reference), the
  repository layout, and the drift discipline above.

## Related

- SSOT (engineering module, `principles` profile: `design/reference/ssot.md`)
- [Project Structure](../../structure/reference/structure.md)
- [Type Safety](../../typing/reference/type_safety.md)
- [Python Toolchain](../../toolchain/reference/tooling.md)
- the api profile's *FastAPI Patterns* reference (`profiles/api/packages/fastapi/reference/fastapi.md`)
