# Endpoint Creation

Endpoint creation workflow covering route definition, schema design, service wiring, error handling, and verification.

Procedure for adding new API endpoints to a FastAPI application. Each endpoint follows a layered structure: the route handler accepts validated input through Pydantic schemas, delegates to a service, and returns a typed response. Dependencies resolve through FastAPI's `Depends` system.

## Prerequisites

Gather before starting:

- **Resource name**: The domain entity this endpoint operates on (e.g., `users`, `orders`).
- **HTTP method and path**: Following RESTful conventions (GET, POST, PUT, PATCH, DELETE).
- **Request/response shape**: Fields, types, and constraints for input and output.
- **Service dependencies**: Repositories, external clients, or other services the handler requires.
- **Authentication requirements**: Public, authenticated, or role-restricted access.

## Creation Workflow

### 1. Define Request and Response Schemas

Create Pydantic models in the `schemas/` directory grouped by domain. Maintain separate models for creation, update, and response.

```python
# app/schemas/items.py
from pydantic import BaseModel, ConfigDict, Field

class ItemCreate(BaseModel):
    model_config = ConfigDict(strict=True)
    name: str = Field(min_length=1, max_length=255)
    price: float = Field(gt=0)

class ItemResponse(BaseModel):
    model_config = ConfigDict(strict=True, from_attributes=True)
    id: int
    name: str
    price: float
    created_at: datetime
```

Enable `from_attributes=True` on response models for direct conversion from SQLAlchemy model instances.

- [Request and Response Validation](../../validation/reference/validation.md)

### 2. Create or Extend the Router Module

Place the router in `app/routes/{resource}.py`. If the router module already exists, add the new handler to it. New routers register with the application in `app/main.py`.

```python
# app/routes/items.py
from fastapi import APIRouter, status
from app.dependencies import ItemServiceDep

router = APIRouter(prefix="/items", tags=["items"])

@router.post("/", status_code=status.HTTP_201_CREATED)
async def create_item(payload: ItemCreate, service: ItemServiceDep) -> ItemResponse:
    return await service.create(payload)
```

Register new routers in `app/main.py` with `app.include_router(items.router)`.

- [Routing and Handler Design](routing.md)

### 3. Implement the Service Method

Services contain business logic and orchestrate repositories. The service receives dependencies through constructor injection and exposes typed methods matching the endpoint operations.

```python
# app/services/items.py
class ItemService:
    def __init__(self, repository: ItemRepository) -> None:
        self._repository = repository

    async def create(self, payload: ItemCreate) -> Item:
        item = Item(**payload.model_dump())
        return await self._repository.create(item)
```

### 4. Wire Dependencies

Create a dependency function that assembles the service with its repository. FastAPI resolves the dependency chain automatically per request.

```python
# app/dependencies.py
from typing import Annotated

from fastapi import Depends
from app.database import SessionDep  # Annotated[AsyncSession, Depends(get_session, scope="function")]

async def get_item_service(session: SessionDep) -> ItemService:
    return ItemService(ItemRepository(session))

ItemServiceDep = Annotated[ItemService, Depends(get_item_service)]
```

The session dependency commits in its exit code, so it carries `scope="function"`: the commit finishes before the 201 is sent, and a failed commit surfaces as an error response rather than after a success the client already holds.

- [FastAPI Patterns](fastapi.md)

### 5. Add Error Handling

Raise application-specific exceptions from the service layer. The global exception handler converts them to structured JSON responses.

```python
# In the service
async def get(self, item_id: int) -> Item:
    item = await self._repository.get(item_id)
    if item is None:
        raise NotFoundError("Item", item_id)
    return item
```

Document error responses on the route decorator:

```python
@router.get(
    "/{item_id}",
    response_model=ItemResponse,
    responses={404: {"model": ErrorResponse}},
)
```

### 6. Verify the Endpoint

Run type checking and tests to confirm the endpoint works:

```bash
uv run basedpyright src/app/routes/items.py
uv run pytest tests/routes/test_items.py -v
```

Start the development server and test manually with curl or the OpenAPI docs at `/docs`:

```bash
uv run uvicorn app.main:app --reload
```

## Decision Support

### HTTP Method Selection

| Operation | Method | Request Body | Idempotent |
|-----------|--------|-------------|------------|
| Create resource | POST | Yes | No |
| Read resource | GET | No | Yes |
| Full replace | PUT | Yes | Yes |
| Partial update | PATCH | Yes | No |
| Delete resource | DELETE | No | Yes |
| List collection | GET | No | Yes |

### Response Status Codes

| Scenario | Status Code | When |
|----------|-------------|------|
| Successful read | 200 | GET returning data |
| Created | 201 | POST creating a resource |
| No content | 204 | DELETE with no response body |
| Validation error | 422 | Pydantic rejects input |
| Not found | 404 | Resource does not exist |
| Conflict | 409 | Duplicate or state conflict |

## Troubleshooting

**Circular imports between routes and dependencies**: Move dependency functions to a separate `dependencies.py` module. Avoid importing route modules from dependency modules.

**Client got 2xx but the row is missing**: the session dependency commits after the response — it lacks `scope="function"` (FastAPI ≥ 0.121.0). See [FastAPI Patterns](fastapi.md).

**Response model validation fails**: Verify `from_attributes=True` on response schemas when returning SQLAlchemy models. Check that all response model fields have corresponding attributes on the ORM model.

**Dependency not found in tests**: Use `app.dependency_overrides` to replace dependencies with test doubles. Clear overrides in fixture teardown.

- [API Testing](../../testing/reference/api_testing.md)

## Related

- [Module Creation](../../../../core/packages/structure/reference/module_creation.md)

## Sources

- https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/ — dependency `scope` and exit-code timing, FastAPI 0.121.0+ (accessed 2026-11-22)
- https://fastapi.tiangolo.com/tutorial/response-model/ — return annotation as response model (accessed 2026-11-22)
