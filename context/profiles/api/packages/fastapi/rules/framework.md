# Framework

Use FastAPI with dependency injection and APIRouter for all API applications.

Use FastAPI as the API framework. Organize endpoints with `APIRouter` grouped by domain. Inject services and database sessions through FastAPI's `Depends` system, declared as `Annotated[T, Depends(...)]` aliases, rather than module-level globals or manual wiring. A `yield` dependency whose exit code can still fail the request (a commit, a flush) declares `scope="function"`, because the default scope runs that code after the response is sent. Keep route handlers thin: a handler maps the validated request into a typed application input, calls one application service, and maps the typed result back — it holds no business logic.

## Rationale

FastAPI's dependency injection integrates with Python's type system, providing automatic request parsing, validation, and OpenAPI documentation from type hints. Router-based organization keeps endpoint definitions modular and testable. The `Depends` system extends the base profile's constructor injection pattern with request-scoped lifecycle management, enabling clean separation between HTTP handling and business logic. The scope rule exists because the default `"request"` scope sends the status code before the exit code runs: a commit that fails there leaves the client holding a 2xx for a write that never landed. The thin-handler rule is what makes the HTTP/business-logic separation observable: once a handler constructs storage queries or invokes a model directly, the injected service boundary exists on paper only.

- [FastAPI Patterns](../reference/fastapi.md)
- [Contract-First Architecture](../../../../core/packages/contracts/reference/contracts.md)

## Examples

**Correct**:
```python
from fastapi import APIRouter
from app.dependencies import UserServiceDep  # Annotated[UserService, Depends(get_user_service)]
from app.schemas import UserResponse

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/{user_id}")
async def get_user(user_id: int, service: UserServiceDep) -> UserResponse:
    return await service.get(user_id)
```

**Incorrect**:
```python
from fastapi import FastAPI

app = FastAPI()
service = UserService()  # Global instance, untestable

@app.get("/users/{user_id}")
async def get_user(user_id: int):
    return service.get(user_id)  # No DI, no response model
```

## Exceptions

Background workers and CLI entry points outside the request cycle may use manual dependency wiring instead of `Depends`. Internal health check endpoints may omit dependency injection when they require no services.
