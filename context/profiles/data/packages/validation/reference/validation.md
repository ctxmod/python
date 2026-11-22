# Data Validation

Data validation principles covering Pydantic v2, msgspec, schema evolution, and boundary validation strategy.

Data validation ensures that information entering a system conforms to expected shapes, types, and constraints before processing begins. In data pipelines, validation failures detected early prevent cascading errors through downstream transformation stages. Two primary tools serve different points on the performance-expressiveness spectrum: Pydantic v2 for rich validation and msgspec for high-throughput deserialization.

## Validation at Boundaries

External data enters the system through well-defined boundaries: file ingestion, API responses, database queries, message queue consumers, and user uploads. Each boundary represents a trust transition where data moves from uncontrolled to controlled space.

| Boundary | Validation Tool | Rationale |
|----------|----------------|-----------|
| File ingestion (CSV, JSON, Parquet) | Pydantic per-row or Polars schema | Row-level type and constraint checking |
| API responses | Pydantic models | Structured deserialization with error detail |
| Message queue payloads | msgspec (high-throughput) or Pydantic | Throughput requirements determine choice |
| Configuration files | Pydantic Settings | Environment variable integration, nested validation |
| Database query results | Pydantic or dataclass mapping | Shape verification after query |

Internal function calls between pipeline stages do not need runtime validation. Static type checking through basedpyright handles internal contracts between trusted modules.

This file specializes the base profile's boundary law for pipeline work; it does not restate it. The model-class ladder, the mapping-not-mirroring discipline, and the prohibition on passing an untyped mapping as a component interface are the parent's and hold here unchanged — a decoded record becomes a typed model at the boundary and travels as one afterwards. What this file adds is the throughput specialization: on high-volume decode paths, msgspec may stand in for Pydantic as the boundary validator. That is a choice of validator, never a licence to skip validation or to hand a raw mapping onward.

- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)
- [Model Boundaries](../../../../core/packages/typing/reference/model_boundaries.md)

## Pydantic v2 Patterns

Pydantic v2 (built on pydantic-core in Rust) provides validation with strong error reporting and custom validators. Enable strict mode to prevent implicit type coercion.

```python
from pydantic import BaseModel, ConfigDict, field_validator
from datetime import datetime

class Transaction(BaseModel):
    model_config = ConfigDict(strict=True, frozen=True)
    id: str
    amount: float
    currency: str
    timestamp: datetime

    @field_validator("currency")
    @classmethod
    def valid_currency(cls, v: str) -> str:
        allowed = {"USD", "EUR", "GBP", "SEK"}
        if v not in allowed:
            raise ValueError(f"Currency must be one of {allowed}")
        return v
```

### Validation Patterns

| Pattern | Use When |
|---------|----------|
| `ConfigDict(strict=True)` | Prevent silent coercion (`"42"` to `42`) |
| `ConfigDict(frozen=True)` | Immutable validated records |
| `field_validator` | Per-field custom constraints |
| `model_validator(mode="after")` | Cross-field validation |
| `TypeAdapter` | Validate without defining a model class |

### Batch Validation

For processing files with thousands of records, collect validation errors rather than failing on the first:

```python
from pydantic import ValidationError

valid_records: list[Transaction] = []
errors: list[tuple[int, str]] = []

for idx, raw in enumerate(raw_records):
    try:
        valid_records.append(Transaction.model_validate(raw))
    except ValidationError as e:
        errors.append((idx, str(e)))

if errors:
    log.warning(f"{len(errors)} validation failures in {len(raw_records)} records")
```

This approach provides a complete error inventory rather than failing at the first invalid record.

## msgspec for Performance-Critical Paths

msgspec delivers 2-10x faster deserialization than Pydantic through compile-time schema analysis and zero-copy decoding. Use it when throughput dominates over validation expressiveness.

```python
import msgspec

class Event(msgspec.Struct, frozen=True):
    event_id: str
    payload: dict[str, object]
    timestamp: float

decoder = msgspec.json.Decoder(list[Event])
events = decoder.decode(raw_bytes)
```

### Tool Selection

| Criterion | Pydantic v2 | msgspec |
|-----------|------------|---------|
| Custom validators | Rich support | Limited |
| Error messages | Detailed, structured | Minimal |
| Deserialization speed | Fast (Rust core) | Faster (2-10x) |
| Serialization speed | Fast | Faster |
| Schema documentation | Automatic JSON Schema | Manual |
| Ecosystem integration | FastAPI, SQLAlchemy, Django | Standalone |

Use Pydantic when validation logic requires custom field validators, cross-field checks, or rich error messages for user-facing APIs. Use msgspec when processing high-volume streams where deserialization speed is the bottleneck and basic type checking suffices.

## Schema Evolution

Data schemas change over time. Handle evolution without breaking existing consumers:

```python
class TransactionV2(BaseModel):
    model_config = ConfigDict(strict=True)
    id: str
    amount: float
    currency: str
    timestamp: datetime
    metadata: dict[str, str] = {}  # New field with default

    @classmethod
    def from_v1(cls, v1: Transaction) -> "TransactionV2":
        return cls(**v1.model_dump(), metadata={})
```

### Evolution Guidelines

| Change Type | Backward Compatible | Strategy |
|-------------|-------------------|----------|
| Add optional field with default | Yes | Add field, existing data validates |
| Remove field | No | Deprecate first, remove in next major |
| Change field type | No | New model version, migration function |
| Add constraint to existing field | No | Validate with warning first, enforce later |

Test schema evolution with compatibility checks in CI. Verify that old data validates against new schemas and that new data degrades gracefully with old schemas.

## Data Quality Checks

Beyond type validation, data quality checks verify statistical and semantic properties:

| Check | Tool | Example |
|-------|------|---------|
| Null presence | Polars `null_count()` | No nulls in required columns |
| Value ranges | Pydantic validators | Temperature between -100 and 100 |
| Uniqueness | Polars `is_unique()` | No duplicate IDs |
| Referential integrity | Custom checks | All foreign keys exist |
| Freshness | Timestamp comparison | Data within expected time window |

Embed quality checks at pipeline stage boundaries. Log failures with structured metadata (row count, failure rate, sample errors) for observability.

## Evidence

Pydantic v2 benchmarks show 5-50x improvement over v1 through the Rust-based pydantic-core. msgspec benchmarks demonstrate 2-10x speedup over Pydantic v2 for pure deserialization. Research on data pipeline reliability (2024-2026) confirms that boundary validation reduces downstream failure rates by 60-80%.

## Related

- [Data Processing](../../processing/reference/processing.md)
- [Streaming and Checkpointing](../../processing/reference/streaming.md)
