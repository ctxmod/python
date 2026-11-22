# Pipeline Testing

Pipeline testing workflow covering unit tests for transformations, integration tests for stage composition, and data quality validation.

Procedure for testing data pipelines across three layers: unit tests for individual transformations, integration tests for stage composition, and data quality validation for output correctness. Pipeline testing verifies that transformations produce expected results, handle edge cases, and maintain data quality invariants.

## Prerequisites

- pytest and plugins installed (`uv add --dev pytest pytest-cov hypothesis`)
- Test data fixtures representing valid, invalid, and edge-case inputs
- Pipeline modules with testable transformation functions
- Data quality expectations defined for pipeline output

## Validation Workflow

### 1. Create Test Data Fixtures

Build reusable test data that covers normal, boundary, and error cases:

```python
import polars as pl
import pytest

@pytest.fixture
def sample_events() -> pl.DataFrame:
    return pl.DataFrame({
        "event_id": ["e1", "e2", "e3", "e4"],
        "user_id": ["u1", "u1", "u2", "u2"],
        "action": ["click", "purchase", "view", "invalid"],
        "value": [1.0, 50.0, 0.5, -1.0],
        "timestamp": [
            "2026-01-15T10:00:00",
            "2026-01-15T11:00:00",
            "2026-01-15T12:00:00",
            "2026-01-15T13:00:00",
        ],
    })

@pytest.fixture
def empty_events() -> pl.DataFrame:
    return pl.DataFrame(
        schema={"event_id": pl.Utf8, "user_id": pl.Utf8, "action": pl.Utf8, "value": pl.Float64}
    )
```

Use typed fixtures with explicit schemas. Avoid loading test data from external files unless the file format itself is under test.

- [Testing Philosophy](../../../../core/packages/testing/reference/testing.md)

### 2. Test Individual Transformations (Unit)

Test each transformation function in isolation with controlled input:

```python
def test_filter_valid_removes_invalid_actions(sample_events: pl.DataFrame) -> None:
    result = filter_valid(sample_events.lazy()).collect()
    assert result["action"].to_list() == ["click", "purchase", "view"]
    assert len(result) == 3

def test_filter_valid_handles_empty_input(empty_events: pl.DataFrame) -> None:
    result = filter_valid(empty_events.lazy()).collect()
    assert len(result) == 0
    assert result.schema == empty_events.schema

def test_aggregate_by_user_sums_correctly(sample_events: pl.DataFrame) -> None:
    result = aggregate_by_user(sample_events.lazy()).collect()
    u1_count = result.filter(pl.col("user_id") == "u1")["event_count"][0]
    assert u1_count == 2
```

Each test verifies one transformation's behavior. Test both the expected output and structural properties (schema preservation, row counts, column presence).

### 3. Test Stage Composition (Integration)

Test multiple stages composed together. Mark with `@pytest.mark.integration` for selective execution:

```python
@pytest.mark.integration
def test_full_pipeline_produces_expected_output(sample_events: pl.DataFrame) -> None:
    result = aggregate_by_user(enrich_dates(filter_valid(sample_events.lazy()))).collect()
    assert "date" in result.columns
    assert result["event_count"].sum() == 3

@pytest.mark.integration
def test_pipeline_with_file_io(tmp_path: Path) -> None:
    input_file = tmp_path / "input.jsonl"
    input_file.write_text('{"event_id":"e1","user_id":"u1","action":"click","value":1.0,"timestamp":"2026-01-15T10:00:00"}\n')
    run_pipeline(str(input_file), str(tmp_path / "output"))
    output = pl.read_parquet(tmp_path / "output" / "aggregated.parquet")
    assert len(output) > 0
```

Use `tmp_path` for file I/O to ensure test isolation.

### 4. Test Edge Cases

| Edge Case | Test Approach |
|-----------|--------------|
| Empty input | Pass empty DataFrame, verify empty output |
| Single record | Verify aggregation works with one row |
| All records invalid | Verify filtering produces empty result without error |
| Null values | Verify null handling (filter, coalesce, or error) |
| Duplicate records | Verify deduplication or correct aggregation |
| Unicode strings | Verify string operations handle multi-byte characters |

### 5. Property-Based Testing

Use Hypothesis to generate diverse inputs and verify pipeline invariants:

```python
from hypothesis import given, settings
from hypothesis import strategies as st

@given(
    values=st.lists(st.floats(allow_nan=False, allow_infinity=False), min_size=1),
)
@settings(max_examples=100)
def test_transformation_preserves_row_count(values: list[float]) -> None:
    df = pl.DataFrame({"id": [f"i{i}" for i in range(len(values))], "value": values})
    result = transform(df.lazy()).collect()
    assert len(result) == len(values)
```

Focus on invariants: row count preservation, sum conservation, ordering stability, no-null introduction. Use `@settings(max_examples=100)` for CI and higher values for nightly runs.

- [Data Validation](../../validation/reference/validation.md)

### 6. Test Data Quality and Checkpointing

Validate output quality with assertion-based checks covering null presence, value ranges, key uniqueness, and schema conformance:

```python
def validate_output_quality(df: pl.DataFrame) -> None:
    for col in ["event_id", "user_id", "date"]:
        assert df[col].null_count() == 0, f"Nulls found in {col}"
    assert (df["event_count"] > 0).all(), "Event count must be positive"

def test_pipeline_output_quality(sample_events: pl.DataFrame) -> None:
    result = run_pipeline_in_memory(sample_events)
    validate_output_quality(result)
```

For pipelines with checkpointing, verify save/load roundtrips and resume behavior. Test that loading a checkpoint and restarting the pipeline skips already-processed items.

## Thresholds

| Metric | Pass | Warning | Fail |
|--------|------|---------|------|
| Transformation testing | All meaningful paths and edge cases tested | Most paths tested | Key transformations untested |
| Integration test pass rate | 100% | 99%+ | <99% |
| Property-based test examples | >=100 per test | 50-100 | <50 |
| Data quality checks | All pass | <5% failures | >=5% failures |
| Edge case coverage | All defined cases tested | Most cases tested | Key edge cases untested |

## Troubleshooting

**Flaky tests with floating-point comparisons**: Use `abs(a - b) < epsilon` or `pytest.approx()` instead of exact equality for float comparisons.

**Slow property-based tests**: Reduce `max_examples` for CI, keep higher values for nightly runs. Use `@settings(deadline=None)` for slow operations.

**Test data drift from production**: Periodically refresh test fixtures to reflect current production schemas. Add schema evolution tests when models change.

**Memory issues in integration tests**: Use `tmp_path` for file I/O and clean up DataFrames between tests. Avoid session-scoped fixtures for large DataFrames.

## Related

- [Data Processing](processing.md)
- [Testing Validation](../../../../core/packages/testing/reference/testing_procedure.md)
