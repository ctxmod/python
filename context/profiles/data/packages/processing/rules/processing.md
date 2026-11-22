# Data Processing

Use Polars for DataFrame operations with lazy evaluation; avoid eager computation on large datasets.

Use Polars as the primary DataFrame library. Prefer lazy evaluation (`LazyFrame`) over eager execution to enable query optimization. Chain transformations declaratively and call `.collect()` only when results are needed. Reserve Pandas for compatibility requirements with libraries that lack Polars support.

## Rationale

Polars executes queries 5-30x faster than Pandas through Rust-backed columnar processing, automatic parallelization, and query plan optimization. Lazy evaluation enables Polars to reorder, prune, and parallelize operations before execution begins. Eager computation on large datasets risks unnecessary memory allocation and redundant passes over data. Declarative chaining produces readable transformation pipelines that the query optimizer can improve without manual tuning.

- [Data Processing](../reference/processing.md)

## Examples

**Correct**:
```python
import polars as pl

result = (
    pl.scan_csv("events.csv")
    .filter(pl.col("status") == "active")
    .group_by("category")
    .agg(
        pl.col("value").mean().alias("avg_value"),
        pl.col("value").count().alias("count"),
    )
    .sort("avg_value", descending=True)
    .collect()
)
```

**Incorrect**:
```python
import pandas as pd

df = pd.read_csv("events.csv")           # Loads entire file eagerly
df = df[df["status"] == "active"]         # No query optimization
result = df.groupby("category")["value"].mean()  # Sequential execution
```

## Exceptions

Libraries requiring Pandas DataFrames (scikit-learn, some plotting libraries) justify Pandas usage at integration boundaries. Convert with `df.to_pandas()` at the boundary, keeping Polars for internal processing. Small datasets under 10,000 rows where Polars' optimization overhead exceeds the processing time may use either library.

## Tooling

Add Polars as a project dependency (`uv add polars`). Use `pl.scan_csv()`, `pl.scan_parquet()`, and `pl.scan_ndjson()` for lazy file reading. Profile memory usage with `polars.Config.set_verbose(True)` during development.
