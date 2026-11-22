# Routing and Handler Design

API routing principles covering RESTful conventions, handler design, path parameters, status codes, and RFC 9457 problem-details error responses.

API routes map HTTP methods and paths to async handler functions. Each handler accepts validated input through type-annotated parameters, delegates to a service layer, and returns a typed response model. Handlers contain no business logic -- they translate between HTTP semantics and application operations.

## RESTful Path Conventions

Use plural nouns for resource collections. Map CRUD operations to standard HTTP methods. Nest sub-resources under their parent when a clear ownership relationship exists.

| Operation | Method | Path | Status Code |
|-----------|--------|------|-------------|
| List | GET | `/items` | 200 |
| Create | POST | `/items` | 201 |
| Read | GET | `/items/{id}` | 200 |
| Update | PUT | `/items/{id}` | 200 |
| Partial update | PATCH | `/items/{id}` | 200 |
| Delete | DELETE | `/items/{id}` | 204 |
| Sub-resource | GET | `/items/{id}/comments` | 200 |

Avoid verbs in paths (`/createItem`, `/deleteUser`). The HTTP method already communicates the action. Use query parameters for filtering, sorting, and pagination on collection endpoints.

## Async Handlers

Define all route handlers as `async def`. Sync handlers (`def`) block the event loop during I/O operations, degrading throughput under concurrent load. FastAPI runs sync handlers in a thread pool, adding overhead without the benefits of native async.

```python
@router.get("/{item_id}", response_model=ItemResponse)
async def get_item(
    item_id: int,
    service: ItemService = Depends(get_item_service),
) -> ItemResponse:
    item = await service.get(item_id)
    if item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return item
```

When calling synchronous third-party libraries from async handlers, use `asyncio.to_thread()` to avoid blocking:

```python
import asyncio

@router.get("/report")
async def generate_report() -> ReportResponse:
    result = await asyncio.to_thread(sync_pdf_library.generate, data)
    return ReportResponse(url=result.url)
```

## Path and Query Parameters

FastAPI infers parameter sources from type annotations. Path parameters come from the URL template. Query parameters come from function arguments not present in the path.

```python
@router.get("/items")
async def list_items(
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=20, ge=1, le=100),
    category: str | None = Query(default=None),
) -> PaginatedResponse[ItemResponse]:
    return await service.list(skip=skip, limit=limit, category=category)
```

### Parameter Guidelines

| Parameter Type | Source | Annotation |
|---------------|--------|------------|
| Path parameter | URL path segment | `item_id: int` in path `/{item_id}` |
| Query parameter | URL query string | `Query(default=..., ge=..., le=...)` |
| Request body | JSON payload | Pydantic `BaseModel` subclass |
| Header | HTTP headers | `Header(default=None)` |
| Cookie | HTTP cookies | `Cookie(default=None)` |

Use `Query()` with constraints for query parameters that need validation. Plain type annotations (`skip: int = 0`) work for parameters without constraints.

## Response Models

Declare `response_model` on every endpoint. This serves three purposes: validating outgoing data against the schema, filtering out fields not present in the response model, and generating accurate OpenAPI documentation.

```python
@router.post(
    "/",
    response_model=ItemResponse,
    status_code=status.HTTP_201_CREATED,
    responses={409: {"model": ErrorResponse}},
)
async def create_item(payload: ItemCreate) -> ItemResponse:
    return await service.create(payload)
```

The `responses` parameter documents additional status codes in OpenAPI. Declare error responses for status codes that handlers raise through `HTTPException` or exception handlers.

## Structured Error Responses

Define application-specific exception classes and register global exception handlers. This produces consistent error response shapes across all endpoints. The shape itself is the RFC 9457 problem-details object carrying a stable application error code as an extension member; the taxonomy behind the classes, the code registry, and the status mapping are owned by the base profile's error-handling reference. This file owns the registration mechanics.

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class AppError(Exception):
    def __init__(self, code: str, title: str, message: str, status_code: int = 400):
        self.code = code
        self.title = title
        self.message = message
        self.status_code = status_code

@app.exception_handler(AppError)
async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
    return JSONResponse(
        status_code=exc.status_code,
        media_type="application/problem+json",
        content={
            "type": f"https://<app>.example/errors/{exc.code.lower()}",
            "title": exc.title,
            "status": exc.status_code,
            "detail": exc.message,
            "instance": request.url.path,
            "code": exc.code,
        },
    )

class NotFoundError(AppError):
    def __init__(self, resource: str, id: int | str):
        super().__init__(
            code="NOT_FOUND",
            title="Not found",
            message=f"{resource} {id} not found",
            status_code=404,
        )
```

Service-layer code raises `NotFoundError("Item", item_id)`. The exception handler converts it to the problem-details response. Handlers stay clean of error formatting logic.

- [Error Handling and Resilience](../../../../core/packages/errors/reference/errors.md)

## Pagination

Return paginated responses for collection endpoints with total count, offset, and limit. Offset-based pagination works well for datasets under 100,000 rows. For larger datasets, use cursor-based pagination (opaque token instead of offset) to avoid performance degradation at high offsets.

## Handler Responsibility Boundaries

| Layer | Responsibility |
|-------|---------------|
| Handler | Parse HTTP input, call service, return HTTP response |
| Service | Business logic, orchestrate repositories, enforce invariants |
| Repository | Database queries, data access |

Handlers should not contain SQL queries, complex conditionals, or business rules. A handler that exceeds 15 lines (excluding parameter declarations) likely contains logic that belongs in the service layer.

## Evidence

RESTful conventions follow established HTTP specification (RFC 9110) and API design guidelines from Google, Microsoft, and Stripe. Async handler performance advantages are documented in FastAPI benchmarks showing 2-5x throughput improvement over sync handlers under concurrent load. Structured error handling follows RFC 9457 Problem Details for HTTP APIs, which obsoletes the earlier RFC 7807 while keeping the same member set.

## Related

- [FastAPI Patterns](fastapi.md)
- [Request and Response Validation](../../validation/reference/validation.md)
- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)
