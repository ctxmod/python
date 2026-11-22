# Routing

Structure endpoints with consistent paths, HTTP methods, and async handlers.

Define all route handlers as `async def`. Group endpoints into `APIRouter` instances by domain, each in its own module. Use plural resource names in paths and standard HTTP methods for CRUD operations. Return typed response models from every endpoint. Register global exception handlers so every failure leaves the boundary as one problem-details error shape rather than a per-endpoint envelope.

## Rationale

Async handlers prevent blocking the event loop during I/O operations (database queries, external API calls), which is the primary bottleneck in API applications. Consistent resource naming and HTTP method mapping produce predictable APIs that align with REST conventions. Typed response models generate accurate OpenAPI documentation and validate outgoing data, catching serialization errors before they reach clients. Centralized exception handlers keep the error shape uniform for the same reason response models keep the success shape uniform: a client that must learn a second schema per endpoint cannot handle failures generically.

- [Routing and Handler Design](../reference/routing.md)
- [FastAPI Patterns](../reference/fastapi.md)
- [Error Handling and Resilience](../../../../core/packages/errors/reference/errors.md)

## Examples

**Correct**:
```python
from fastapi import APIRouter, status
from app.schemas import ItemCreate, ItemResponse

router = APIRouter(prefix="/items", tags=["items"])

@router.post("/", status_code=status.HTTP_201_CREATED, response_model=ItemResponse)
async def create_item(payload: ItemCreate) -> ItemResponse:
    item = await item_service.create(payload)
    return item

@router.get("/{item_id}", response_model=ItemResponse)
async def get_item(item_id: int) -> ItemResponse:
    return await item_service.get(item_id)
```

**Incorrect**:
```python
@app.post("/createItem")  # Verb in path, no prefix grouping
def create_item(data: dict):  # Sync handler, untyped input/output
    return item_service.create(data)
```

## Exceptions

WebSocket endpoints use `async def` with the `@router.websocket` decorator and do not return response models. Server-sent event streams return `StreamingResponse` instead of a typed model. Both cases still require async handlers.
