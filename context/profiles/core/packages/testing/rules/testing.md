---
paths:
  - "test_*.py"
  - "*/test_*.py"
  - "*/*/test_*.py"
  - "*/*/*/test_*.py"
  - "*/*/*/*/test_*.py"
  - "*/*/*/*/*/test_*.py"
  - "*_test.py"
  - "*/*_test.py"
  - "*/*/*_test.py"
  - "*/*/*/*_test.py"
  - "*/*/*/*/*_test.py"
  - "*/*/*/*/*/*_test.py"
  - "tests/*.py"
  - "tests/*/*.py"
  - "tests/*/*/*.py"
  - "tests/*/*/*/*.py"
  - "tests/*/*/*/*/*.py"
  - "*/tests/*.py"
  - "*/tests/*/*.py"
  - "*/tests/*/*/*.py"
  - "*/tests/*/*/*/*.py"
  - "*/*/tests/*.py"
  - "*/*/tests/*/*.py"
  - "*/*/tests/*/*/*.py"
  - "*/*/*/tests/*.py"
  - "*/*/*/tests/*/*.py"
  - "*/*/*/*/tests/*.py"
  - "conftest.py"
  - "*/conftest.py"
  - "*/*/conftest.py"
  - "*/*/*/conftest.py"
  - "*/*/*/*/conftest.py"
  - "*/*/*/*/*/conftest.py"
---

# Testing

pytest is the configured runner; markers keep tiers selectable, and every mutating operation pairs an idempotency check with a stale-precondition check.

pytest is the configured Python runner for every test the repository writes (`test_*.py`, `conftest.py`); when and where tests run automatically is the repository's run policy, not this rule's. Write checks that verify meaningful behavior, critical edge cases, and the robustness of the software; do not write redundant checks solely for coverage metrics or artificial confidence. Every mutating operation carries both an idempotency check and a stale-precondition check; either alone is incomplete.

## Rationale

pytest's fixture system enables explicit dependency injection, producing isolated and composable setups. The tier gate exists because breadth and cost are independent axes: a narrow check can still reach a paid external service, so tier — what a check may touch — governs selection, not scope. The evaluation tier — anything consuming real credentials, real tokens, or real cost — is additionally excluded from every automatic context by definition. The pairing rule exists because idempotency and precondition staleness fail in opposite directions: an idempotency check alone passes while a stale submission silently overwrites, and a precondition check alone passes while a replay produces a second effect.

- testing layers and tiers, fixture organization and scope selection, mocking strategy, credential/secret and external-service handling, coverage philosophy, property-based testing, tiering obligations, and testability as a design signal — [Testing Philosophy](../reference/testing.md)
- the contract-conformance suite and the idempotency / stale-precondition check pairing — [Contract Testing Validation](../../contracts/reference/contract_testing.md)

## Examples

**Correct**:
```python
# fixture injection with explicit scope
@pytest.fixture(scope="session")
def db_session():
    engine = create_engine("sqlite:///:memory:")
    session = Session(bind=engine)
    yield session
    session.close()

def test_create_user(user_service: UserService, mock_repo: Mock) -> None:
    user = User(id=1, email="test@example.com")
    user_service.create(user)
    mock_repo.save.assert_called_once_with(user)
```

**Incorrect**:
```python
class TestUserService(unittest.TestCase):  # a different runner than the configured pytest runner
    def setUp(self):
        self.repo = FakeRepo()  # No fixture reuse across checks
        self.service = UserService(self.repo)
```

## Exceptions

Performance benchmarks may use pytest-benchmark. Property-based tests via Hypothesis supplement but do not replace example-based pytest checks. Evaluation-tier checks are not exempt from the runner rule — they remain pytest checks, reachable only by explicit selection, never by a default selection.

## Tooling

Run `pytest --cov=src --cov-report=term-missing` for coverage visibility when a run warrants it. Coverage reporting helps identify untested code paths but is not used as a gate. Use `pytest-xdist` (`pytest -n auto`) for parallel execution and `pytest-asyncio` for async support. Register every marker the suite uses under `[tool.pytest.ini_options] markers` — at minimum `contract` and `evaluation` — and deselect the evaluation tier by default with `addopts = "-m 'not evaluation'"`. Under `--strict-markers` an unregistered marker errors; without it, a mistyped marker silently selects nothing, which reads as a passing gate.
