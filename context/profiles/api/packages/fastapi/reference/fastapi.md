# FastAPI Patterns

FastAPI framework principles covering dependency injection, router organization, middleware, lifespan events, and background tasks.

FastAPI applications organize around three mechanisms: dependency injection through `Depends`, route grouping through `APIRouter`, and request lifecycle management through lifespan events. These mechanisms compose to produce applications where HTTP handling, business logic, and infrastructure remain separated and independently testable.

## Dependency Injection with Depends

FastAPI's `Depends` system extends constructor injection with request-scoped lifecycle management. Dependencies declared in function signatures resolve automatically per request, enabling clean teardown of resources like database sessions and HTTP clients.

```python
from collections.abc import AsyncIterator
from typing import Annotated

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_session() -> AsyncIterator[AsyncSession]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# scope="function": the commit runs before the response is sent, so a failed
# commit becomes a 500 instead of a 2xx the client has already received.
SessionDep = Annotated[AsyncSession, Depends(get_session, scope="function")]

async def get_user_service(session: SessionDep) -> UserService:
    return UserService(UserRepository(session))

UserServiceDep = Annotated[UserService, Depends(get_user_service)]
```

Declare dependencies with `Annotated[T, Depends(...)]` — FastAPI's documented preference — and name the alias once (`SessionDep`, `UserServiceDep`) so every handler reuses the same declaration and the type checker sees the real parameter type rather than a `Depends(...)` default value.

**When the exit code runs.** Since FastAPI 0.121.0, `Depends(..., scope=...)` selects when a `yield` dependency's code after `yield` runs. The default for a `yield` dependency is `scope="request"`: the exit code runs **after the response has been sent**. That is right for pure cleanup (closing a client) and wrong for anything that can still fail the request — a commit, a flush, an outbox write. With the default scope, a commit that raises leaves the client holding a success response for a write that never landed (verified with FastAPI 0.165.0: default scope returned 200 while the commit raised; `scope="function"` returned 500). Use `scope="function"` for every dependency whose exit code can fail the operation, or commit inside the application service before the handler returns. A `"request"`-scoped dependency may only depend on other `"request"`-scoped dependencies; a `"function"`-scoped one may depend on either.

Nest dependencies to build a graph: `get_user_service` depends on `get_session`, and FastAPI resolves the chain automatically.

### Dependency Guidelines

| Guideline | Rationale |
|-----------|-----------|
| One responsibility per dependency | Keeps the dependency graph shallow and debuggable |
| Use `yield` for resources needing cleanup | Sessions, connections, and file handles close reliably |
| `scope="function"` when exit code can fail the request | Commits and flushes must finish before the status code is sent |
| Declare with `Annotated[T, Depends(...)]` aliases | One declaration per dependency; the checker sees `T`, not a default value |
| Type-annotate dependency return values | basedpyright validates the full injection chain |
| Avoid module-level state in dependencies | Request-scoped dependencies prevent cross-request contamination |
| Test by calling dependency functions directly | Dependencies are plain async generators, testable without FastAPI |

### Overriding for Tests

Use `app.dependency_overrides` to replace dependencies during testing. Map the dependency function to a lambda returning a mock. This avoids `mocker.patch` path string fragility -- the dependency function itself serves as the override key. Call `app.dependency_overrides.clear()` in teardown.

## Router Organization

Group endpoints by domain into `APIRouter` instances. Each router lives in its own module and registers with the application through `include_router`:

```python
# app/routes/users.py
from fastapi import APIRouter

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/{user_id}")
async def get_user(user_id: int, service: UserServiceDep) -> UserResponse:
    return await service.get(user_id)

# app/main.py
from fastapi import FastAPI
from app.routes import users, items, health

app = FastAPI()
app.include_router(users.router)
app.include_router(items.router)
app.include_router(health.router)
```

The return annotation is the response model: FastAPI validates, filters, and documents the response from it, so `response_model=` is needed only when the declared return type must differ from the serialized shape (for example, a handler that returns an ORM object filtered through a narrower DTO).

Routers accept their own middleware, dependencies, and response classes. Apply shared authentication at the router level rather than repeating it on each endpoint:

```python
router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(require_admin)],
)
```

## The Route Contract: Thin Adapters

Every route handler is a thin adapter between the HTTP contract and the application services. Request and response types come from the generated contract DTOs (the contract-first architecture reference at [Contract-First Architecture](../../../../core/packages/contracts/reference/contracts.md) owns the generation discipline). The handler body has exactly five responsibilities:

1. **Receive the framework-validated request DTO.** The route signature declares the generated transport types; validation runs before the handler body; the body never parses raw JSON.
2. **Obtain the authenticated principal and map the request DTO to a typed application Command.** The principal comes from the shared router-level authentication dependency (see Router Organization above); carry it, the idempotency key, and any revision preconditions (`workflow_revision`, `action_revision`) into the Command. The mapping is mechanical — no domain logic lives here.
3. **Delegate to the handwritten command/query application service.** The service makes authorization decisions against the carried principal and owns idempotency, state transitions, and side-effect publication.
4. **Accept the typed result.** The service returns an application or domain object, never an untyped dict.
5. **Map the result to the generated response DTO or the typed error mapping, and return it.** The route declares the response model and status code; when the service returns a typed error, the handler maps it to the HTTP error contract (the error-handling reference owns the error taxonomy). Serialization belongs to the framework.

```
command route: approve a proposed action
  input:  generated ApproveRequest (framework-validated) + principal from router auth
  step 1: cmd = Command(principal, action_revision, workflow_revision, idempotency_key)
  step 2: result = approval_service.approve(cmd)   # decides authorization + owns effects
  step 3: return generated Approval(result)         # or the typed error response
```

Where a project has not adopted contract generation, the same five steps hold with hand-authored transport schemas standing in for generated DTOs; the separation between transport schemas and internal models still applies (see the validation reference and the model-boundaries reference).

This composes with router organization above: routers group endpoints by domain and apply shared dependencies; the five-step contract governs each handler body.

### Prohibited in Routes

| Prohibition | Rationale |
|-------------|-----------|
| Prompt text or agent instructions | The HTTP layer is an interaction boundary, not an agent; instruction text belongs to the component that owns agent execution, never the route. |
| Storage query logic | Routes never touch stores or repositories directly; reads go through services, writes through Commands. |
| Model or tool invocations | Model and tool calls belong behind application services, where authorization, idempotency, and Telemetry apply uniformly. |

## Lifespan Events

Use the `lifespan` context manager (introduced in FastAPI 0.93+) for startup and shutdown logic. This replaces the deprecated `on_event` decorators:

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from typing import TypedDict

from fastapi import FastAPI

class AppState(TypedDict):
    engine: AsyncEngine
    session_factory: async_sessionmaker[AsyncSession]

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[AppState]:
    engine = create_async_engine(settings.database_url, pool_pre_ping=True)
    yield {"engine": engine, "session_factory": async_sessionmaker(engine, expire_on_commit=False)}
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

Yielding a mapping is the lifespan-state protocol: each request receives a shallow copy as `request.state`, so dependencies read `request.state.session_factory` instead of a module-level global, and tests that run the app's lifespan get fresh resources per app instance. Assigning to `app.state` still works but is untyped and shared mutable state.

Lifespan manages database engines, connection pools, HTTP client sessions, and cache connections. Resources initialized before `yield` are available throughout the application lifetime. Cleanup after `yield` runs during shutdown.

## OpenTelemetry

FastAPI 0.142.0 added native OpenTelemetry support; since 0.143.0 automatic exporter setup from `OTEL_*` variables is opt-in (`FASTAPI_OTEL_AUTO_CONFIGURE=true` or `FastAPI(telemetry={"auto_configure": True})`). A service that configures its own tracer and exporter providers — the observability reference's three-layer model — needs no change; do not enable auto-configuration on top of an explicit provider setup, or two exporter pipelines run.

## Middleware

Apply middleware for cross-cutting concerns: CORS headers, request timing, structured logging, and request ID propagation. Middleware executes in reverse registration order (last registered runs first). Keep middleware thin -- delegate complex logic to dependencies instead, since reading the request body in middleware prevents FastAPI from parsing it for route handlers.

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

## Background Tasks

Use `BackgroundTasks` for fire-and-forget work that should not delay the response: sending emails, writing audit logs, triggering webhooks. For longer-running or failure-sensitive tasks, use a task queue (Celery, arq, or Dramatiq) instead -- background tasks share the process and have no retry mechanism.

```python
@router.post("/users/")
async def create_user(
    payload: UserCreate,
    background: BackgroundTasks,
    service: UserServiceDep,
) -> UserResponse:
    user = await service.create(payload)
    background.add_task(send_welcome_email, user.email)
    return user
```

## Evidence

FastAPI dependency injection patterns follow the framework's official documentation and production usage at companies processing millions of requests daily. The `Depends` system has remained stable since FastAPI 0.1.0; `scope` (0.121.0) is the one change to when `yield` dependencies exit. The lifespan context manager replaced event hooks based on community feedback about resource management reliability. FastAPI 0.162.6 dropped Python 3.10 and 0.128.0 dropped `pydantic.v1`; the 0.14x–0.16x series (2026-10) tightened request-parameter semantics — explicit JSON `null` versus an omitted value (0.151.0), defaults no longer validated unless `validate_default=True` (0.148.0), `extra` model config applied per parameter location (0.160.0) — so read the release notes before bumping across those versions.

## Related

- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)
- [Project Structure](../../../../core/packages/structure/reference/structure.md)
- [Routing and Handler Design](routing.md)
- [Request and Response Validation](../../validation/reference/validation.md)

## Sources

- https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/ — `scope="function"` versus the default `"request"` exit timing; Annotated preference (accessed 2026-11-22; timing probed with FastAPI 0.165.0)
- https://fastapi.tiangolo.com/release-notes/ — 0.121.0 dependency scopes, 0.128.0 `pydantic.v1` removal, 0.142.0/0.143.0 OpenTelemetry, 0.148.0–0.162.6 parameter-semantics changes and Python 3.10 drop (accessed 2026-11-22)
- https://fastapi.tiangolo.com/tutorial/response-model/ — return annotation as the response model (accessed 2026-11-22)
- https://starlette.dev/lifespan/ — lifespan state yielded as a mapping and copied into `request.state` (accessed 2026-11-22)
