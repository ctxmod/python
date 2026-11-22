# Data Validation

Validate data at boundaries with Pydantic v2 or msgspec; use strict mode for type coercion safety.

Validate all external data at system boundaries using Pydantic v2 or msgspec. Use Pydantic for complex validation with custom validators and field constraints. Use msgspec for performance-critical deserialization paths where throughput matters more than validation expressiveness.

## Rationale

Data pipelines ingest data from sources outside the system's control: files, APIs, databases, message queues. Without boundary validation, malformed records propagate through transformation stages and produce corrupt output or silent failures. Strict validation at ingestion catches problems early, before compute resources are spent on invalid data. Pydantic v2 provides expressive validation with a strict mode that prevents implicit type coercion, while msgspec delivers 2-10x faster deserialization for high-throughput scenarios.

- [Data Validation](../reference/validation.md)

## Examples

**Correct**:
```python
from pydantic import BaseModel, ConfigDict, field_validator

class SensorReading(BaseModel):
    model_config = ConfigDict(strict=True)
    sensor_id: str
    timestamp: datetime
    value: float

    @field_validator("value")
    @classmethod
    def value_in_range(cls, v: float) -> float:
        if not -1000 <= v <= 1000:
            raise ValueError(f"Sensor value {v} outside valid range")
        return v

def ingest(raw: dict[str, object]) -> SensorReading:
    return SensorReading.model_validate(raw)
```

**Incorrect**:
```python
def ingest(raw: dict) -> dict:
    return {
        "sensor_id": raw["sensor_id"],       # No type validation
        "value": float(raw.get("value", 0)), # Silent coercion, default hides missing data
    }
```

## Exceptions

Internal data transfers between trusted pipeline stages do not require runtime validation. Static type checking covers internal contracts. Exploratory data analysis scripts may relax validation during prototyping, but production pipelines always validate at boundaries.

## Tooling

Add Pydantic (`uv add pydantic`) or msgspec (`uv add msgspec`) as project dependencies. Configure basedpyright strict mode to catch type mismatches at boundaries.
