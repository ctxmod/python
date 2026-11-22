# FastAPI Patterns

FastAPI framework principles covering dependency injection, router organization, middleware, lifespan events, and background tasks.

FastAPI applications organize around three mechanisms: dependency injection through `Depends`, route grouping through `APIRouter`, and request lifecycle management through lifespan events. These mechanisms compose to produce applications where HTTP handling, business logic, and infrastructure remain separated and independently testable.

## Dependency Injection with Depends

FastAPI's `Depends` system extends constructor injection with request-scoped lifecycle management. Dependencies declared in function signatures resolve automatically per request, enabling clean teardown of resources like database sessions and HTTP clients.

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

async def get_user_service(
    session: AsyncSession = Depends(get_session),
) -> UserService:
    return UserService(UserRepository(session))
```

Dependencies that `yield` execute their cleanup code after the response completes, providing predictable resource management without manual `try/finally` blocks in handlers. Nest dependencies to build a graph: `get_user_service` depends on `get_session`, and FastAPI resolves the chain automatically.

### Dependency Guidelines

| Guideline | Rationale |
|-----------|-----------|
| One responsibility per dependency | Keeps the dependency graph shallow and debuggable |
| Use `yield` for resources needing cleanup | Sessions, connections, and file handles close reliably |
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

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, service: UserService = Depends(get_user_service)) -> UserResponse:
    return await service.get(user_id)

# app/main.py
from fastapi import FastAPI
from app.routes import users, items, health

app = FastAPI()
app.include_router(users.router)
app.include_router(items.router)
app.include_router(health.router)
```

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

Use the `lifespan` context manager (introduced in FastAPI 0.93+) for startup and shutdown logic. This replaces the deprecated `on_event` decorators and provides type-safe resource sharing through the application state:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    engine = create_async_engine(settings.database_url)
    app.state.engine = engine
    yield
    await engine.dispose()

app = FastAPI(lifespan=lifespan)
```

Lifespan manages database engines, connection pools, HTTP client sessions, and cache connections. Resources initialized before `yield` are available throughout the application lifetime. Cleanup after `yield` runs during shutdown.

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
@router.post("/users/", response_model=UserResponse)
async def create_user(
    payload: UserCreate,
    background: BackgroundTasks,
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    user = await service.create(payload)
    background.add_task(send_welcome_email, user.email)
    return user
```

## Evidence

FastAPI dependency injection patterns follow the framework's official documentation and production usage at companies processing millions of requests daily. The `Depends` system has remained stable since FastAPI 0.1.0. The lifespan context manager replaced event hooks based on community feedback about resource management reliability.

## Related

- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)
- [Project Structure](../../../../core/packages/structure/reference/structure.md)
- [Routing and Handler Design](routing.md)
- [Request and Response Validation](../../validation/reference/validation.md)
