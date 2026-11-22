# Validation

Validate all request and response data with Pydantic models in strict mode.

Define Pydantic models with `ConfigDict(strict=True)` for all request bodies and response shapes. Separate request schemas from response schemas, even when fields overlap. Use `Field()` for constraints (min/max, regex, descriptions) that flow into OpenAPI documentation. Where a contract source exists, transport schemas are generated from it rather than hand-authored, and hand-authored models stay on the internal side of the boundary.

## Rationale

Strict mode rejects type coercion (e.g., string "1" silently becoming integer 1), catching malformed input at the API boundary before it reaches business logic. Separating request and response schemas prevents leaking internal fields (database IDs, timestamps, soft-delete flags) through input models and allows each to evolve independently. Field constraints embedded in Pydantic models generate OpenAPI validation documentation without additional configuration. The generated-versus-hand-authored split matters for the same reason: a hand-authored transport model is a second source of truth for a shape the contract already owns, and the two diverge quietly.

[Request and Response Validation](../reference/validation.md)
[FastAPI Patterns](../../fastapi/reference/fastapi.md)
[Type Safety](../../../../core/packages/typing/reference/type_safety.md)
[Model Boundaries](../../../../core/packages/typing/reference/model_boundaries.md)
[Error Handling and Resilience](../../../../core/packages/errors/reference/errors.md)

## Examples

**Correct**:
```python
from pydantic import BaseModel, ConfigDict, Field

class ItemCreate(BaseModel):
    model_config = ConfigDict(strict=True)
    name: str = Field(min_length=1, max_length=255)
    price: float = Field(gt=0, description="Price in USD")

class ItemResponse(BaseModel):
    model_config = ConfigDict(strict=True)
    id: int
    name: str
    price: float
    created_at: datetime
```

**Incorrect**:
```python
class Item(BaseModel):  # Single model for request and response
    id: int | None = None  # Optional ID leaks into create requests
    name: str
    price: float  # No constraints, no strict mode
```

## Exceptions

File upload endpoints may accept `UploadFile` parameters directly rather than wrapping them in Pydantic models. Query parameter filtering with many optional fields may use dataclasses or `Depends` callables when Pydantic overhead adds no validation benefit.
