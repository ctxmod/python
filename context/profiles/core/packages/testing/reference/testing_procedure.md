# Testing Validation

Testing validation workflow covering pytest execution, fixture setup, coverage collection, and test organization.

Procedure for running the Python test toolchain, organizing fixtures, collecting coverage, and validating test quality. Testing validates that the codebase behaves as expected across unit, integration, and end-to-end layers. Run this procedure when a question, defect, or acceptance criterion needs it during development, and wherever the repository's run policy places it (hooks, CI, or explicit invocation); the policy itself is a repository decision.

## Prerequisites

- pytest and plugins installed (`uv add --dev pytest pytest-cov pytest-asyncio`)
- `pyproject.toml` configured with `[tool.pytest.ini_options]`
- Test directory structure mirroring source layout
- Source code with testable modules

## Validation Workflow

### 1. Run the Test Suite

Execute all tests with coverage:

```bash
uv run pytest --cov=src --cov-report=term-missing
```

For focused testing during development:

```bash
uv run pytest tests/test_client.py              # Single file
uv run pytest tests/test_client.py::TestClient   # Single class
uv run pytest -k "test_get"                      # Pattern match
uv run pytest --lf                               # Last failed only
```

### 2. Evaluate Test Results

Analyze test output across three dimensions:

| Dimension | Pass Criteria | Action on Failure |
|-----------|--------------|-------------------|
| All tests pass | Zero failures, zero errors | Fix failing tests before other work |
| No warnings | Zero pytest warnings | Resolve deprecations and configuration issues |
| Coverage reviewed | Uncovered paths investigated | Add tests for meaningful untested behavior |

### 3. Review Coverage Report

The `--cov-report=term-missing` flag shows uncovered line numbers per module. Use coverage as a discovery tool to find untested code paths, not as a target to maximize. Prioritize testing effort by risk:

| Risk Level | Testing Expectation | Examples |
|------------|---------------------|----------|
| Critical | Thorough — all meaningful paths and edge cases | Authentication, payment, data integrity |
| Standard | Solid coverage of primary behaviors and error paths | Business logic, API handlers, services |
| Low | Minimal or excluded | Generated code, configuration wiring |

Exclude non-meaningful paths from coverage:

```toml
[tool.coverage.run]
source = ["src"]
omit = [
    "src/*/_internal/generated/*",
    "src/*/py.typed",
]

[tool.coverage.report]
exclude_lines = [
    "if TYPE_CHECKING:",
    "pragma: no cover",
    "@overload",
]
```

Coverage reporting provides visibility into untested code paths. Do not use coverage as a gate; focus on writing tests for meaningful behavior and edge cases instead.

[Testing Philosophy](testing.md)

### 4. Organize Fixtures

Structure fixtures using a hybrid model. Shared fixtures live in `conftest.py` at directory boundaries. Test-specific fixtures stay near their usage.

```
tests/
  conftest.py                # Session-scoped shared fixtures
  fixtures/
    database.py              # Database fixtures (imported via pytest_plugins)
    api_client.py            # HTTP client fixtures
  services/
    conftest.py              # Service-specific fixtures
    test_user_service.py
  models/
    conftest.py              # Model-specific fixtures
    test_user.py
```

Register fixture modules in the root `conftest.py`:

```python
# tests/conftest.py
pytest_plugins = [
    "tests.fixtures.database",
    "tests.fixtures.api_client",
]
```

### Fixture Scope Selection

| Scope | Use When | Example |
|-------|----------|---------|
| `function` | Each test needs isolation (default) | Mock repositories, test data |
| `class` | Tests in a class share state | Service instance with config |
| `module` | Expensive per-module setup | Database connection |
| `session` | One-time setup for full suite | Database engine, API client |

Prefer `function` scope for test isolation. Broader scopes introduce shared state that can cause test interference. Use broader scopes only when per-test creation measurably impacts suite duration.

### 5. Validate Test Isolation

Tests must pass in any order and in parallel. Verify isolation:

```bash
uv run pytest --randomly-seed=12345    # Randomized order (pytest-randomly)
uv run pytest -n auto                   # Parallel execution (pytest-xdist)
```

Common isolation failures:

| Symptom | Cause | Resolution |
|---------|-------|------------|
| Tests pass alone, fail together | Shared mutable state | Use function-scoped fixtures |
| Order-dependent failures | Test A modifies global state | Remove global state, inject dependencies |
| Parallel failures | File system or port conflicts | Use `tmp_path` fixture, random ports |

### 6. Fast Feedback Strategies

| Strategy | Command | When |
|----------|---------|------|
| Last failed | `uv run pytest --lf` | After fixing a failure |
| Affected only | `uv run pytest --testmon` | During active development |
| Skip slow | `uv run pytest -m "not slow"` | Quick local validation |
| Parallel | `uv run pytest -n auto` | Full suite runs |

For async tests, set `asyncio_mode = "auto"` in `pyproject.toml` and use `@pytest_asyncio.fixture` for async fixtures. Mark slow tests with `@pytest.mark.slow` for selective exclusion.

### 7. Respect the Tier Gate

The suite this procedure runs covers the fast-offline and integration tiers only. Evaluation-tier tests — anything consuming real tokens, real credentials, or incurring real cost — are marker-deselected by default so no automatic run selects them. Register every marker the suite uses in the same block: an unregistered marker errors under `--strict-markers` and, without it, silently selects nothing — which reads as a passing gate.

```toml
[tool.pytest.ini_options]
markers = [
    "contract: contract-conformance tests; fast-offline tier",
    "evaluation: evaluation tier; manual invocation only, never selected in CI",
]
addopts = "-m 'not evaluation'"
```

They run only through the explicit manual invocation documented in the testing principles reference, and never via a widened CI command. See the contract-testing procedure for the contract-gate side of the suite.

## Thresholds

| Metric | Pass | Warning | Fail |
|--------|------|---------|------|
| Test pass rate | 100% | 99%+ | <99% |
| Critical path testing | All meaningful paths tested | Most paths tested | Key paths untested |
| Test suite duration (unit) | <5 min | 5-10 min | >10 min |
| Flaky test rate | 0% | <2% | >=2% |

## Troubleshooting

**pytest cannot find tests**: Verify `testpaths` in `pyproject.toml` and that test files follow `test_*.py` naming.

**Fixtures not found**: Check `conftest.py` placement and `pytest_plugins` dotted module paths.

**Async tests hang**: Check event loop scope. Use `loop_scope="session"` for cross-test fixtures. Avoid mixing sync and async fixtures at the same scope.

**Coverage lower than expected**: Verify `source` in `[tool.coverage.run]` points to `src/`.

[Python Toolchain](../../toolchain/reference/tooling.md)

## Related

- [Type Safety](../../typing/reference/type_safety.md)
- [Module Creation](../../structure/reference/module_creation.md)
