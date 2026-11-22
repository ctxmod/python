# Contract Testing Validation

Contract testing validation workflow covering contract lint and bundle, regeneration, the generated-output drift check, the conformance suite, and generated-mock currency.

Procedure for proving that contracts, generated output, and service behavior stay in sync: lint and bundle the contract source, regenerate every output, reject drift, run the conformance suite, and verify generated mocks are current. The lint, regeneration, and drift legs are deterministic generation checks and run as part of the change itself on every contract change and before any commit that touches `gen/`. The conformance suite is test execution: it runs when a contract question or acceptance criterion needs it, or as a CI contract gate where the repository's run policy places one. The procedure is entirely fast-offline tier: the service runs in-process and nothing reaches a network or a credential.

## Prerequisites

- Contract source initialized and the codegen tools pinned in the lockfile (the setup procedure, steps 7-8)
- Contract recipes wired in the justfile (the setup procedure, steps 9-10): `contract-lint`, `contract-generate`, `contract-drift-check`
- pytest with the `contract` marker registered in `pyproject.toml`; conformance tests live under `tests/contract/` and carry `@pytest.mark.contract`
- The service importable in-process

[Contract-First Architecture](contracts.md)

## Validation Workflow

### 1. Lint and Bundle the Contract Source

Validate the modular contract source and resolve it into the single reviewed bundle:

```bash
uv run redocly lint contracts/api/openapi.yaml
uv run buf lint contracts/domain-events
uv run redocly bundle contracts/api/openapi.yaml --output gen/api/bundle/openapi.yaml
```

For the asynchronous contracts, run the breaking-change check against the previous committed revision:

```bash
uv run buf breaking contracts/domain-events --against <baseline>
```

A lint or breaking-change finding is a contract failure: fix the contract source, never the check's configuration. The toolchain reference owns tool selection and pinning; if a tool is not yet a lockfile dependency, pin it there first.

### 2. Regenerate Every Output

Regenerate all derived artifacts from the freshly bundled contract — the bundle, the Pydantic DTOs, the Domain Event bindings:

```bash
just contract-generate
```

The recipes invoke the pinned tools through `uv run`, so the lockfile's exact versions execute.

### 3. Diff-Check for Drift

Regenerate into a temporary location and compare against the committed `gen/` tree:

```bash
just contract-drift-check
```

A non-empty diff is a failure, not a warning. It means one of two things, and both are fixed forward, never by silencing the check:

| Diff cause | Remedy |
|------------|--------|
| Generated output was hand-edited | Revert the hand edit; regenerate |
| A contract change was not followed by regeneration | Regenerate and commit the result together with the contract change |

Repeat the regenerate-and-compare pair for every generation target — the bundle, the DTOs, and the bindings. The drift-check recipe must cover each.

### 4. Run the Conformance Suite

Run the real service in-process against the bundled contract artifact when a contract question or acceptance criterion needs it, or in the contract gate:

```bash
uv run pytest tests/contract/
```

The `contract` marker lets the suite be selected by marker inside a combined run (`uv run pytest -m contract`). The suite proves three things:

- Request and response behavior conforms to the bundled contract — schemas, status codes, and the problem-details error shape match the contract's error model.
- The pairing rule holds: every mutating operation in the contract carries BOTH an idempotency test AND a stale-precondition test. A mutating operation missing either half of the pair is a failure of this step even when every present test passes.
- The three idempotency levels behave as the error-handling reference specifies: a repeated Command submission returns the original semantic outcome; a redelivered Domain Event produces no repeated business effect; a retried action does not execute twice; and a stale precondition fails with the typed 409 conflict.

[Error Handling and Resilience](../../errors/reference/errors.md)

### 5. Verify Generated Mocks Are Current

Mocks and fixtures of contract surfaces are generation targets under the same drift discipline:

- Where a mock or fixture generation target exists, add it to the drift-check recipe; committed mocks must equal fresh regeneration.
- The guard is absolute: no hand-shaped payload object exists outside fixtures generated from the contracts. A hand-shaped mock drifting from the contract is the failure mode this step exists to catch — generated mocks stay current, or the check fails.

[Testing Philosophy](../../testing/reference/testing.md)

## Thresholds

| Check | Pass | Fail |
|-------|------|------|
| Contract lint and breaking check | Zero findings | Any finding |
| Drift check | Empty diff on every target | Any difference |
| Conformance suite | All tests pass; pairing complete | Any failure, or a mutating operation missing its pair |
| Mock and fixture currency | Committed equals regenerated; no hand-shaped payloads | Any stale or hand-shaped mock |

## Troubleshooting

**Drift fails with no contract change**: someone hand-edited `gen/` output. Revert the edit and regenerate; never commit a hand-mended generated file.

**Conformance fails while generated types line up**: generated types can agree while behavior diverges — the conformance suite, not the type check, is the authority. Extend the suite to the affected response shapes.

**Bundle differs on an untouched contract**: the toolchain version moved. Re-pin the lockfile; the lockfile, not the local environment, selects versions.

**The suite selects nothing**: register the `contract` marker in `[tool.pytest.ini_options]` markers — under `--strict-markers`, an unregistered marker errors instead of silently selecting no tests.

[Python Toolchain](../../toolchain/reference/tooling.md)
[Project Setup](../../toolchain/reference/project_setup.md)
