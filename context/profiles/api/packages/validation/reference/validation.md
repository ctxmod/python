# Request and Response Validation

Request/response validation principles with Pydantic v2 covering strict mode, schema separation, generated versus hand-authored schemas, custom validators, OpenAPI integration, and the RFC 9457 problem-details error response shape.

Pydantic v2 validates all data crossing the API boundary. Request models parse and constrain incoming payloads. Response models filter and shape outgoing data. Strict mode prevents silent type coercion. Separate schemas for input and output enable independent evolution and prevent internal field leakage.

## Strict Mode

Enable `ConfigDict(strict=True)` on all API schemas. Without strict mode, Pydantic silently converts `"42"` to `42` for integer fields and `"true"` to `True` for boolean fields. This coercion masks client-side type errors that should be caught and reported.

```python
from pydantic import BaseModel, ConfigDict, Field

class ItemCreate(BaseModel):
    model_config = ConfigDict(strict=True)
    name: str = Field(min_length=1, max_length=255)
    price: float = Field(gt=0, description="Price in USD")
    tags: list[str] = Field(default_factory=list, max_length=10)
```

Strict mode applies to the model and all nested models. A single non-strict model in a nested hierarchy creates a coercion gap. Apply strict mode at the base class level when using shared base models.

## Schema Separation

Maintain separate Pydantic models for creation requests, update requests, and responses. Sharing a single model between input and output leads to one of two problems: either internal fields (IDs, timestamps, soft-delete flags) appear in creation requests, or clients cannot receive computed fields.

```python
class ItemCreate(BaseModel):
    model_config = ConfigDict(strict=True)
    name: str = Field(min_length=1, max_length=255)
    price: float = Field(gt=0)

class ItemUpdate(BaseModel):
    model_config = ConfigDict(strict=True)
    name: str | None = None
    price: float | None = Field(default=None, gt=0)

class ItemResponse(BaseModel):
    model_config = ConfigDict(strict=True, from_attributes=True)
    id: int
    name: str
    price: float
    created_at: datetime
    updated_at: datetime
```

### Schema Naming Convention

| Suffix | Purpose | Example |
|--------|---------|---------|
| `Create` | POST request body | `UserCreate` |
| `Update` | PUT/PATCH request body | `UserUpdate` |
| `Response` | Response payload | `UserResponse` |
| `Filter` | Query parameter group | `UserFilter` |
| `Detail` | Extended response with relations | `UserDetail` |

The `from_attributes=True` config allows Pydantic to read from SQLAlchemy model attributes directly, eliminating manual dictionary conversion.

## Field Constraints

Use `Field()` to declare constraints that flow into OpenAPI documentation. Constraints validated at the Pydantic layer reject invalid data before it reaches business logic.

```python
from pydantic import Field, field_validator

class OrderCreate(BaseModel):
    model_config = ConfigDict(strict=True)
    quantity: int = Field(ge=1, le=1000, description="Number of items")
    email: str = Field(pattern=r"^[^@]+@[^@]+\.[^@]+$")
    notes: str | None = Field(default=None, max_length=2000)
```

### Common Constraint Patterns

| Constraint | Field Argument | Use Case |
|-----------|---------------|----------|
| Minimum/maximum | `ge`, `le`, `gt`, `lt` | Numeric ranges |
| String length | `min_length`, `max_length` | Text inputs |
| Pattern | `pattern` | Email, phone, slug formats |
| Allowed values | `Literal["a", "b"]` type | Enumerations |
| List size | `min_length`, `max_length` | Collection bounds |

Prefer Pydantic constraints over manual validation in handlers. Constraints produce automatic error messages with field names and violation details.

## Custom Validators

Use `@field_validator` for validation logic that exceeds simple constraints: cross-field validation, database-dependent checks, or business rule enforcement.

```python
from pydantic import field_validator, model_validator

class DateRangeFilter(BaseModel):
    model_config = ConfigDict(strict=True)
    start_date: date
    end_date: date

    @model_validator(mode="after")
    def validate_date_range(self) -> "DateRangeFilter":
        if self.end_date < self.start_date:
            raise ValueError("end_date must be after start_date")
        if (self.end_date - self.start_date).days > 365:
            raise ValueError("date range cannot exceed 365 days")
        return self
```

`@field_validator` validates individual fields. `@model_validator` validates relationships between fields. Use `mode="after"` for validators that need parsed values and `mode="before"` for validators that operate on raw input.

## Nested Models

Compose Pydantic models for structured payloads. Nested models validate recursively, and strict mode propagates through the hierarchy. Reuse nested models across parent schemas to maintain consistent validation for shared structures like addresses, monetary amounts, or date ranges.

## Error Response Format

Customize FastAPI's default validation error format by registering a `RequestValidationError` exception handler. Normalize Pydantic's error output into a consistent shape with field paths and messages that clients can parse programmatically.

The normalized shape is the RFC 9457 problem-details object: `type`, `title`, `status`, `detail`, and `instance` members, plus the stable application error code as an extension member. Every application error — validation failures, authorization failures, stale-precondition conflicts — shares this one wire shape, so clients parse a single error schema. Request-validation failures carry Pydantic's field paths and violation messages as problem-detail entries within that shape.

The typed error taxonomy, the stable-code registry, the full member semantics, and the status-code mapping (including the 409 Conflict a stale precondition produces) are owned by the error-handling reference ([Error Handling and Resilience](../../../../core/packages/errors/reference/errors.md)). This file owns only the schema-level normalization of Pydantic's violation output into that shape.

## Schema Organization

Place schemas in a `schemas/` directory grouped by domain (`users.py`, `items.py`). Shared schemas (pagination wrappers, error responses) live in `common.py`. Domain schemas import from `common.py` when needed. This structure prevents circular imports and keeps schema files focused.

## Transport Schemas versus Internal Schemas

Transport schemas — the request and response models that appear on the wire — are generated from the OpenAPI contract wherever contract generation is adopted. The contract-first architecture reference ([Contract-First Architecture](../../../../core/packages/contracts/reference/contracts.md)) owns the generation discipline and the authority semantics: the contract is the single source of truth for the wire shape, and the generated Pydantic DTOs are its output, never an independent schema set.

Internal schemas — application DTOs, domain-facing models, service request and result types — are hand-authored. They evolve with the domain, not with the wire: a transport change regenerates the transport DTOs but does not silently redefine internal semantics.

The separation is deliberate, and it cuts both ways:

- Generating internal schemas from the transport contract couples internal semantics to the wire shape: a compatibility-driven contract change would become an unreviewed domain change.
- Hand-authoring transport schemas duplicates the contract and creates two sources of truth for one boundary.
- Where a project has not adopted contract generation, hand-authored transport schemas stand in for generated DTOs — but they remain transport schemas, and the separation from internal models still holds.

The disciplines in this file apply at both sites: strict mode, schema separation, field constraints, and custom validators govern hand-authored schemas directly, while generated transport DTOs carry them through the contract's generation configuration. The model-class ladder and the mapping-not-mirroring rule behind this separation are owned by the model-boundaries reference ([Model Boundaries](../../../../core/packages/typing/reference/model_boundaries.md)).

## Evidence

Pydantic v2 strict mode and validation patterns follow the library's official documentation. Schema separation aligns with API design practices documented by FastAPI's author and production API teams. The `from_attributes` configuration enables direct SQLAlchemy model serialization, reducing boilerplate in the response path. Validation error customization follows FastAPI's exception handler documentation.

## Related

- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)
- [FastAPI Patterns](../../fastapi/reference/fastapi.md)
- [Routing and Handler Design](../../fastapi/reference/routing.md)
