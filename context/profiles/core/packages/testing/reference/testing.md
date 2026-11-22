# Testing Philosophy

Testing principles covering fixture organization, mocking strategies, coverage philosophy, property-based testing, the three test tiers with the manual-evaluation gatekeeping rule, fake-config provider injection, same-helper local and production parity, and contract-conformance obligations.

Testable Python code results from applying design principles -- dependency injection through constructors, Protocol-based interfaces, and explicit dependencies -- rather than from testing-specific techniques. When code requires extensive mocking or complex test setup, the remedy is design correction. Sound architecture provides the testability.

Run policy — whether hooks, pre-commit, or CI execute tests automatically, and where temporary checks written during active work live — is a repository decision this reference does not make. Whatever the policy, the layers and tiers below classify what a test covers and what it may touch, and the evaluation tier never runs automatically. The pytest suite is the configured Python runner.

## Testing Layers

Each layer serves a distinct purpose with different speed and scope characteristics:

| Layer | Focus | Dependencies | Speed | When to run |
|-------|-------|-------------|-------|----------------|
| Unit | Single function or class | All mocked | Fast (<1s) | On explicit selection during active work |
| Integration | Component interactions | Real deps, isolated | Medium (1-10s) | On explicit selection when a boundary question needs it |
| End-to-end | Full workflows | All real | Slow (10-60s) | On explicit selection for release or behavior claims |

Unit tests form the majority of authored checks. Integration checks validate component boundaries. End-to-end checks cover critical user paths. This distribution prioritizes fast feedback while still verifying system-level behavior.

## Test Tiers

The scope layers above classify tests by breadth; a second, orthogonal axis classifies them by what a test is allowed to touch — network, credentials, external services, real cost. Three tiers:

| Tier | May touch | Where it runs |
|------|-----------|---------------|
| 1. Fast offline | Nothing external — no network, no credentials, no external services; providers are fakes and services run in-process | Every local run and every CI run |
| 2. Integration | Real client code against a dev/staging environment tier or local emulators (service profiles document specifics) | CI, marked and documented as externally dependent |
| 3. Manually-triggered evaluation | Real external services, real credentials and tokens, production-style data — real cost | Explicit manual invocation only |

Tier 1 covers both unit and offline-integration scope: an in-process database or a fake provider keeps the test offline regardless of its breadth. Tier 2 exercises real client behavior against an environment that is cheap and isolated — a dev/staging environment tier with least-privilege credentials of its own (service profiles own specifics), or a local emulator where the service provides one. Such tests carry a marker so their external dependency is visible, and their documentation states the environment access they require.

**The evaluation tier is guarded: it never runs in any automated pipeline.** Any test that consumes real tokens or real credentials, or incurs real cost, belongs to this tier by definition. It runs only through an explicit command guarded by an environment-flag opt-in plus a confirmation step, and CI never sets the flag:

```bash
# the evaluation harness -- manual invocation only
if [[ "$RUN_EVALUATION_TESTS" != "1" ]]; then
    echo "evaluation tier is manual-only; set RUN_EVALUATION_TESTS=1 to run"
    exit 1
fi
uv run scripts/run_evaluation.py --env=dev --dataset=quick
```

A scheduled or manually-dispatched workflow that runs these tests is a separate, explicit pipeline, never part of the routine gate; the tiers classify what a check may touch.

The tiers compose with the scope layers rather than replacing them: an end-to-end check exercises the application's own stack under an environment the invoking agent controls, which is tier 2 at most. Whatever reaches past that — real external model services, production-style data volumes — is tier 3 and manual, whatever its scope.

## Fixture Organization

pytest fixtures serve as explicit dependency injection for tests. Organize them in a hybrid model: centralized shared fixtures in `conftest.py` at directory boundaries, localized test-specific fixtures near their usage.

```python
# tests/conftest.py -- centralized shared fixtures
pytest_plugins = ["tests.fixtures.database", "tests.fixtures.api_client"]

# tests/fixtures/database.py
@pytest.fixture(scope="session")
def db_engine():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()

# tests/services/conftest.py -- localized fixtures
@pytest.fixture
def user_service(mock_user_repo, mock_notifier):
    return UserService(mock_user_repo, mock_notifier)
```

### Scope Selection

| Scope | Use When | Teardown |
|-------|----------|----------|
| `function` (default) | Each test needs fresh state | After each test |
| `class` | Tests in a class share setup | After class completes |
| `module` | Module-level shared resource | After module completes |
| `session` | Expensive one-time setup (db engine, client) | After full suite |

Prefer `function` scope for test isolation. Use broader scopes only for genuinely expensive resources where per-test creation measurably slows the suite.

## Mocking Strategies

Mock external dependencies at the boundary, not deep within the call chain. Constructor injection makes mock substitution straightforward -- pass `Mock(spec=Protocol)` instances through constructor parameters rather than patching import paths.

### Mock Guidelines

| Guideline | Rationale |
|-----------|-----------|
| Mock at the boundary | Internal mocking couples tests to implementation |
| Use `spec=Protocol` | Catches attribute errors at test time |
| Prefer constructor injection | Avoids `mocker.patch` path string fragility |
| Mock behaviors, not data | `return_value` and `side_effect` over attribute setting |
| Use `AsyncMock` for async | Correct await behavior in async tests |

`mocker.patch` (from pytest-mock) is appropriate when constructor injection is unavailable -- for example, when testing code that calls module-level functions from third-party libraries:

```python
def test_api_failure(mocker: MockerFixture) -> None:
    mocker.patch("httpx.get", side_effect=ConnectionError("timeout"))
    with pytest.raises(ApiError):
        client.fetch_data()
```

## Testing Credentials, Secrets, and External Services

Code that, in production, reads a credential, a secret, or an external service is tested with one pattern: **swap the config file, inject a fake provider.** The test then exercises the production code path — config parsing, resolution, caching — with only the inputs swapped. This is the boundary-mocking rule above applied to the credential boundary, with one addition: the provider seam is a Protocol the application owns, so the fake replaces an interface rather than patching a module.

### Fake Config and Provider Injection

- The application config is loaded from a path the test can point elsewhere: write a fake config file with the same structure and test values, and point the loader at it.
- Every credential or secret lookup goes through a small Protocol — a credential/secret provider — accepted as a constructor argument. Tests inject a fake provider; production injects the real one. The Protocol is the seam: never construct a provider client inside business code, and never monkeypatch a client module.
- Exercise the caching behavior too. The fake provider counts its calls; after the first resolution, further lookups must be served from the cache and must not reach the provider. A test that only asserts the returned value leaves the cache path untested.

```python
class SecretProvider(Protocol):
    def get_secret(self, name: str) -> bytes: ...

def test_secret_resolution_uses_config_and_cache(tmp_path):
    config = write_fake_config(tmp_path, env="test")       # swapped config file
    provider = FakeSecretProvider({"db_password": b"fake-password"})
    resolver = SecretResolver(config, provider)            # production code path

    assert resolver.get_secret("db_password") == b"fake-password"
    assert resolver.get_secret("db_password") == b"fake-password"  # cache hit
    assert provider.calls == 1
```

The result is a test that is fast, deterministic, and low-memory while staying close to production behavior: the code under test is the code that runs in production.

### Same-Helper Local and Production Parity

Three properties keep local runs and production honest to each other, stated explicitly:

- **Same helper.** One resolution helper serves production and local runs alike; local runs swap the config file and the provider, never the helper. A parallel test-only resolution path is prohibited — it stops testing the code that runs in production.
- **Same code path.** Config parsing, resolution, and caching are identical in every environment; only the inputs differ.
- **Same config shape.** Configuration files are identical in structure across environments; only the environment tier and the resource identifiers differ.

All client construction stays behind the shared helpers — never inline constructor calls scattered through the codebase — so wiring changes in one place and a wiring defect surfaces in tests before it reaches production. Integration tests against the dev/staging tier or local emulators call the exact same helper functions; anything that differs per environment is a config value, not a code branch.

### Keep External I/O out of Hot Paths

- Resolve at startup or on first access, then cache in memory for the process lifetime with a bounded cache — a least-recently-used bound, so a long-running process neither re-fetches nor grows without bound.
- Never resolve inside a tight loop or a high-frequency path. A framework callback or request handler must not resolve a credential on every invocation — resolve once when the handler or tool is constructed, and read from the cache thereafter.
- The hot-path rule is what keeps tier 1 fast: when resolution happens once and is cached, faking the provider once fakes every subsequent use.

## Coverage Philosophy

Coverage measures execution paths, not correctness. Write tests that verify meaningful behavior, critical edge cases, and the robustness of the software. Do not write redundant tests solely for coverage metrics or artificial confidence. Prefer fewer high-value tests that catch real bugs over many low-value tests that duplicate coverage. Use coverage reports to identify untested code paths, not as a target to maximize.

Focus testing effort by risk, not by percentage targets:

| Path | Testing Expectation | Rationale |
|------|---------------------|-----------|
| Critical business logic | Thorough — all meaningful paths and edge cases | High-impact code warrants exhaustive behavioral testing |
| Standard application code | Solid coverage of primary behaviors and error paths | Adequate verification without diminishing returns |
| Generated or boilerplate code | Excluded | Coverage of generated code provides no signal |

Use coverage reporting for visibility:

```bash
pytest --cov=src --cov-report=term-missing
```

A test that increases coverage without asserting behavior is worse than no test: it creates false confidence. Every test should verify a specific behavioral expectation.

## Property-Based Testing

Hypothesis generates diverse inputs to find edge cases that example-based tests miss. Research shows Hypothesis catches 50x more mutations than traditional unit tests, with 76% of mutations caught in the first 20 generated inputs.

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers(min_value=0, max_value=100), min_size=1))
def test_normalize_preserves_ordering(values: list[int]) -> None:
    result = normalize(values)
    assert all(0 <= v <= 1 for v in result)
    for i in range(len(result) - 1):
        if values[i] < values[i + 1]:
            assert result[i] <= result[i + 1]
```

Use property-based testing for pure functions with well-defined invariants, data transformation pipelines, and serialization roundtrips. Avoid it for integration tests with real services or tests where generating valid input is harder than writing examples.

## Contract Conformance Obligations

The contract-first architecture reference owns the drift and generated-diff semantics; this reference owns the test obligations that prove them.

- **The pairing rule.** Every mutating API operation carries BOTH an idempotency test AND a stale-precondition test. The pairing is mandatory — either test alone is incomplete coverage of the operation.
- The idempotency side of the pair mirrors the three idempotency levels of the error-handling reference:

| Idempotency level | The test proves |
|-------------------|-----------------|
| Command idempotency — identity is principal + operation + idempotency key | Repeating the same submission returns the original semantic outcome and produces no second effect |
| Domain Event delivery deduplication — identity is the immutable `domain_event_id` scoped to its consumer | Redelivering the same Domain Event hits the deduplication record and produces no repeated business effect |
| Action idempotency — identity is `action_id`, valid against its `workflow_revision` precondition | A retry does not execute the action twice; the precondition fails loudly once the state it was proposed against has moved |

- The stale-precondition side of the pair proves the conflict path: a mutating request carrying a superseded `workflow_revision` — and its `action_revision` when the request targets a specific proposed action — fails with the typed 409 conflict response, never a best-effort overwrite.
- **Generated mocks stay current.** Mocks and fixtures of contract surfaces are generated from the contract, never hand-shaped; committed mocks must equal fresh regeneration. A hand-shaped payload object outside generated fixtures is exactly the drift the check exists to catch.
- **API behavior conforms to the bundled contract.** Conformance tests run the real service in-process against the bundled contract artifact, so an implementation cannot drift from the contract even where generated types happen to line up.
- Scenario coverage lands at tier 2: an integration run against the dev/staging tier or local emulators verifies the success, retry, rejection, and duplicate Domain Event delivery flows.

The runnable sequence — lint and bundle, regenerate, drift-check, conformance suite, mock currency — is the contract-testing procedure (`reference/procedures/validation/python/contract-testing.md`).

## Fast Feedback Strategies

| Strategy | Tool | Impact |
|----------|------|--------|
| Parallel execution | `pytest-xdist` (`-n auto`) | Linear speedup with core count |
| Incremental testing | `pytest-testmon` (`--testmon`) | Run only affected tests |
| Failing-first ordering | `pytest --lf` (last failed) | Fastest signal on regressions |
| Marker-based selection | `pytest -m "not slow"` | Skip expensive tests locally |

### CI Tiering

```
PR checks:     Unit tests (2-5 min)
Merge to main: Unit + Integration (5-15 min)
Release:       Full suite including E2E (15-60 min)
```

Where a repository runs its committed suite in CI, tier it this way: fast unit checks on every pull request, integration on merge, and the full suite including end-to-end before release.

## CI Gate Obligations

The contract-first architecture reference defers the gate list here. Generalized, these are the claims a contract-first Python service must be able to prove — through a CI gate or through checks executed when the corresponding claim is made, whichever the repository's run policy prescribes:

- **Contract currency** — the API contract lints, bundles, and its generated outputs are current.
- **Domain Event contract currency** — the Protobuf sources lint, contain no breaking changes, and their generated bindings are current.
- **Conformance** — the service's request and response behavior conforms to the bundled contract.
- **Scenario coverage** — a tier-2 run verifies success, retry, rejection, and duplicate Domain Event delivery.
- **Adversarial inputs** — adversarial-input tests against exposed boundaries pass.
- **Generated-output currency** — every committed generated artifact (DTOs, bindings, mocks) equals fresh regeneration under the pinned toolchain.
- **Evaluation-tier absence** — CI never sets the evaluation opt-in flag, so the manual tier is absent from every automatic gate by construction.

Deployment gates belong to the cloud-platform doctrine, and browser-side mock-driven UI states to the browser-client doctrine (the typescript module); this list stops at the application boundary.

## Testability as Design Signal

Tests that require deep mocking, complex setup, or access to private internals indicate design problems. Common barriers and their resolutions:

| Barrier | Symptom | Resolution |
|---------|---------|------------|
| Hidden dependencies | `mocker.patch` deep in call chain | Accept dependencies through constructors |
| Wide interfaces | Test doubles with many methods | Use smaller Protocol-based interfaces |
| Tight coupling | Tests break when implementation changes | Mock at boundaries, test behavior |
| Global state | Tests interfere with each other | Remove globals, inject state |

Each barrier has a design-level resolution. Testing workarounds (monkey-patching, module-level mocking) treat symptoms while preserving the underlying coupling.

## Related

- [Type Safety](../../typing/reference/type_safety.md)
- [Project Structure](../../structure/reference/structure.md)
- [Python Toolchain](../../toolchain/reference/tooling.md)
- [Contract-First Architecture](../../contracts/reference/contracts.md)
- [Error Handling and Resilience](../../errors/reference/errors.md)
- [Contract Testing Validation](../../contracts/reference/contract_testing.md)
