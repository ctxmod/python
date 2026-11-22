# API Testing

API testing validation workflow covering TestClient usage, async testing with httpx, database fixtures, and contract verification.

Procedure for testing FastAPI applications using TestClient for synchronous tests, httpx for async tests, and isolated database fixtures for integration tests. Run this procedure during development, before commits, and in CI.

## Prerequisites

- FastAPI application with endpoints implemented
- pytest and plugins installed:

```bash
uv add --dev pytest pytest-asyncio httpx pytest-cov
uv add --dev aiosqlite  # For async SQLite test database
```

- `pyproject.toml` configured with `asyncio_mode = "auto"`

## Validation Workflow

### 1. Set Up the Test Client

Use FastAPI's `TestClient` for synchronous endpoint tests. The test client wraps httpx and runs the ASGI application in-process without network overhead.

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

@pytest.fixture
def client() -> TestClient:
    return TestClient(app)
```

For async tests, use `httpx.AsyncClient` with the ASGI transport:

```python
import httpx
import pytest_asyncio

@pytest_asyncio.fixture
async def async_client() -> AsyncGenerator[httpx.AsyncClient, None]:
    async with httpx.AsyncClient(
        transport=httpx.ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        yield client
```

### 2. Write Endpoint Tests

Test each endpoint by exercising the HTTP interface. Validate status codes, response bodies, and error cases.

```python
# tests/routes/test_items.py
def test_create_item(client: TestClient) -> None:
    response = client.post("/items/", json={"name": "Widget", "price": 9.99})
    assert response.status_code == 201
    assert response.json()["name"] == "Widget"

def test_create_item_invalid_price(client: TestClient) -> None:
    response = client.post("/items/", json={"name": "Widget", "price": -1})
    assert response.status_code == 422

def test_get_item_not_found(client: TestClient) -> None:
    response = client.get("/items/99999")
    assert response.status_code == 404
```

### Test Coverage Per Endpoint

| Test Category | What to Verify | Priority |
|---------------|---------------|----------|
| Happy path | Correct status code and response body | Required |
| Validation errors | Pydantic rejects invalid input (422) | Required |
| Not found | Missing resources return 404 | Required |
| Conflict | Duplicate creation returns 409 | When applicable |
| Authentication | Unauthenticated returns 401/403 | When auth exists |
| Pagination | Offset, limit, total count | For list endpoints |

### 3. Configure Database Test Fixtures

Use an isolated test database to prevent tests from affecting production data. Override the session dependency with a test-scoped session that rolls back after each test.

```python
# tests/conftest.py
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from app.models.base import Base
from app.database import get_session

test_engine = create_async_engine("sqlite+aiosqlite:///", echo=True)
test_session_factory = async_sessionmaker(test_engine, expire_on_commit=False)

@pytest_asyncio.fixture(autouse=True)
async def setup_database() -> AsyncGenerator[None, None]:
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

@pytest_asyncio.fixture
async def test_session() -> AsyncGenerator[AsyncSession, None]:
    async with test_session_factory() as session:
        yield session

@pytest.fixture
def client(test_session: AsyncSession) -> TestClient:
    async def override_session() -> AsyncGenerator[AsyncSession, None]:
        yield test_session
    app.dependency_overrides[get_session] = override_session
    with TestClient(app) as client:
        yield client
    app.dependency_overrides.clear()
```

This fixture creates tables before each test and drops them after, providing full isolation. The dependency override routes all database operations through the test session.

- [FastAPI Patterns](../../fastapi/reference/fastapi.md)

### 4. Mock External Dependencies

Replace external services with test doubles using `app.dependency_overrides`. Map the dependency function to a lambda returning a mock. This avoids patching import paths, which is fragile and couples tests to module structure.

| Approach | When to Use | Advantage |
|----------|------------|-----------|
| `dependency_overrides` | FastAPI dependencies | No import path coupling |
| `pytest-mock` / `mocker.patch` | Non-DI code, module-level functions | Works outside FastAPI |
| `respx` / `httpx.MockTransport` | External HTTP calls | Records request history |

### 5. Validate Response Schemas

Parse responses through the declared Pydantic model to catch schema drift between ORM models and response schemas:

```python
def test_response_schema(client: TestClient) -> None:
    response = client.get("/items/1")
    item = ItemResponse.model_validate(response.json())
    assert item.id == 1
```

### 6. Run Tests and Evaluate Results

Execute the full test suite with coverage:

```bash
uv run pytest tests/ --cov=src --cov-report=term-missing -v
```

For focused API testing:

```bash
uv run pytest tests/routes/ -v              # All route tests
uv run pytest tests/routes/test_items.py -v  # Single resource
uv run pytest -k "test_create" -v            # Pattern match
```

- [Testing Validation](../../../../core/packages/testing/reference/testing_procedure.md)

## Thresholds

| Metric | Pass | Warning | Fail |
|--------|------|---------|------|
| Endpoint test coverage | All endpoints tested | Most endpoints tested | Endpoints untested |
| Route handler testing | All meaningful paths tested | Most paths tested | Key paths untested |
| All tests pass | 100% | 99%+ | <99% |
| Test isolation | Pass in any order | Rare order deps | Frequent failures |
| Response schema validation | All endpoints | Most endpoints | Few endpoints |

## Troubleshooting

**TestClient raises 500 with no traceback**: Set `raise_server_exceptions=False` to inspect the error response body instead.

**Async fixtures not recognized**: Verify `asyncio_mode = "auto"` in `pyproject.toml` and use `@pytest_asyncio.fixture` for async generators.

**Database state leaks between tests**: Confirm `setup_database` fixture is `autouse=True` and drops tables in teardown.

**dependency_overrides not taking effect**: Set overrides before creating the TestClient. Clear in teardown.

## Related

- [Request and Response Validation](../../validation/reference/validation.md)
- [Testing Philosophy](../../../../core/packages/testing/reference/testing.md)
- [Endpoint Creation](../../fastapi/reference/endpoint_creation.md)
