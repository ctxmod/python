# Python Module Overview

Base profile for Python development with tooling, testing, and packaging standards.

The python core profile is the contract-first Python authoring tier of this module, and every other python profile specializes this tier rather than restating it.

## Purpose and scope

Applies to Python source, project manifests, and contract sources in any repository that writes Python — services, packages, script tools, and data or model work alike. This is the module's root profile: it extends nothing and carries the language-level law that the api, data, ml, and compute profiles specialize.

The profile stops at the application boundary. Deployment, infrastructure, and platform wiring are owned by cloud-platform doctrine (the infra and ai modules, `github.com/ctxmod/infra/context` and `github.com/ctxmod/ai/context`); browser clients generated from the same contracts are owned by the typescript module (`github.com/ctxmod/typescript/context`). This profile points to both and restates neither. The architecture vocabulary used throughout — Command, Domain Event, Workflow State, Projection, Audit Record, Telemetry, the three idempotency levels — is defined in the [glossary](glossary.md).

## Architectural defaults

| Concern | Default |
|---|---|
| Toolchain | uv, basedpyright strict, ruff, pytest — all configured through `pyproject.toml`, dev tooling in one canonical surface |
| Composition | `src/` layout inside every package and application; the contract-first top level composes around it in multi-deployable repositories |
| Contracts | API and Domain Event contracts are reviewed source; generated output is never authoritative and never hand-edited |
| Typing | Strict static checking plus validation at every untrusted boundary; no untyped mapping as a component interface |
| Errors | Typed application errors with stable codes; RFC 9457 problem details at the HTTP boundary; a stale precondition is a typed 409 |
| Idempotency | Three distinct levels — Command, Domain Event delivery, action — with three identity formulas, never collapsed into one |
| Observability | Three layers — the Audit Record history, technical Telemetry, agent-level Telemetry — none substituting for another |
| Testing | Three tiers by what a test may touch; the evaluation tier never runs automatically in CI |

## Inheritance and overrides

A descendant adds constraints, specializes behavior, introduces implementation-specific guidance, and adds skills and references. A descendant does not copy this tier's reference material, and does not silently redefine it — a real disagreement is declared as an override in the descendant's own profile, not left to be discovered by a reader comparing two files.

The descendant profiles today, and what each specializes:

- **api** — the web framework binding: dependency injection, router organization, the thin-adapter route contract, and request/response schema separation at the HTTP boundary.
- **data** — pipeline and job concerns: durable job state, migrations, dataframe processing, streaming. Its high-throughput deserialization allowance is a specialization of the boundary-validation default, not a replacement for it.
- **ml** — experiment configuration, training-loop coordination, and checkpoint discipline.
- **compute** — rented GPU/CPU session operations: the colab CLI and runpodctl, spend gating, and teardown discipline for workloads run off-workstation (bulk inference, batch generation).

## Normative references

The reference layer is the source of architectural truth; the rules layer enforces it; the skills layer operationalizes it.

- **Architecture and contracts** — [contract-first architecture](../../contracts/reference/contracts.md) · [model boundaries](../../typing/reference/model_boundaries.md) · [project structure](../../structure/reference/structure.md)
- **Correctness at the boundary** — [type safety](../../typing/reference/type_safety.md) · [error handling and resilience](../../errors/reference/errors.md) · [configuration](../../configuration/reference/configuration.md)
- **Operability** — [observability layers](../../observability/reference/observability.md) · [structured logging](../../observability/reference/logging.md)
- **Build and prove** — [packaging](../../toolchain/reference/packaging.md) · [toolchain](../../toolchain/reference/tooling.md) · [testing philosophy](../../testing/reference/testing.md)
- **Procedures** — [project setup](../../toolchain/reference/project_setup.md) · [module creation](../../structure/reference/module_creation.md) · [testing validation](../../testing/reference/testing_procedure.md) · [contract testing](../../contracts/reference/contract_testing.md) · [type checking](../../typing/reference/type_checking.md)
- **Skills** — the api profile's *add an API endpoint* skill (`profiles/api/packages/fastapi/skills/add_api_endpoint/SKILL.md`) · [regenerate contracts](../../contracts/skills/regenerate_contracts/SKILL.md)
