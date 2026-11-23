# Data Processing

Data processing principles covering Polars lazy evaluation, chunked processing, generator pipelines, and memory management.

Data processing transforms raw input into structured output through a sequence of operations: filtering, mapping, aggregating, and joining. Polars provides the primary engine for tabular processing through lazy evaluation and automatic parallelization. Generator-based pipelines handle record-by-record processing with constant memory. The choice between approaches depends on data shape, volume, and processing complexity.

## Polars Lazy Evaluation

Polars' `LazyFrame` defers execution until `.collect()`. This allows the query optimizer to reorder operations, prune unused columns, and push predicates closer to the data source. Eager execution (`DataFrame`) processes each operation immediately, preventing optimization.

```python
import polars as pl

# Lazy: optimizer eliminates unused columns, pushes filter before groupby
result = (
    pl.scan_parquet("data/*.parquet")
    .filter(pl.col("status") == "complete")
    .group_by("region")
    .agg(pl.col("revenue").sum())
    .sort("revenue", descending=True)
    .collect()
)
```

### Lazy vs Eager Selection

| Scenario | Approach | Rationale |
|----------|----------|-----------|
| Multi-step transformations | Lazy | Optimizer improves execution plan |
| File scanning (CSV, Parquet, NDJSON) | Lazy (`scan_*`) | Predicate pushdown, projection pushdown |
| Interactive exploration | Eager | Immediate feedback during prototyping |
| Small in-memory datasets (<10K rows) | Either | Optimization overhead exceeds benefit |
| Streaming large files | Lazy; `.collect()` or `.sink_*()` | Streaming engine (default for lazy queries since Polars 2.0), bounded memory |

Since Polars 2.0, `LazyFrame.collect()` keeps `engine="auto"`, which now resolves to the **streaming engine** for lazy queries; eager `DataFrame` operations still run in memory. `collect(streaming=True)` was deprecated in 1.25 and removed in 2.0 — it now raises `TypeError` with a pointer to `engine="streaming"`. For results that do not fit in memory, write them out with `sink_parquet()` / `sink_ipc()` rather than collecting.

**Row order is not guaranteed by the streaming engine** for operations that do not require it — `group_by`, joins and `unpivot` among them. A pipeline that depended on the old in-memory engine's incidental ordering changes output silently on upgrade: pass `maintain_order=True` to `group_by`, `maintain_order=` on joins where supported, or end with an explicit `.sort(...)` wherever downstream consumers, checksums, or golden files depend on order. `pl.Config.set_engine_affinity("in-memory")` (or `POLARS_ENGINE_AFFINITY=in-memory`) restores the old default as a migration aid, not a target state.

## Polars Expression Patterns

Polars expressions compose naturally for complex transformations. Prefer expression-based operations over row-wise iteration.

```python
# Expression-based: parallel, vectorized
df = df.with_columns(
    (pl.col("price") * pl.col("quantity")).alias("total"),
    pl.col("timestamp").dt.date().alias("date"),
    pl.when(pl.col("status") == "returned")
    .then(pl.lit(0))
    .otherwise(pl.col("total"))
    .alias("net_total"),
)
```

### Common Expression Patterns

| Pattern | Expression | Use Case |
|---------|-----------|----------|
| Conditional | `pl.when().then().otherwise()` | Derived columns with logic |
| Window | `pl.col("x").over("group")` | Group-aware calculations |
| String | `pl.col("name").str.to_lowercase()` | Text normalization |
| Temporal | `pl.col("ts").dt.truncate("1h")` | Time bucketing |
| Struct | `pl.col("nested").struct.field("key")` | Nested data access |
| List | `pl.col("tags").list.contains("x")` | Array column operations |

Avoid `.apply()` and `.map_elements()` for row-wise Python functions. These operations bypass Polars' parallelization and fall back to single-threaded Python execution. If custom logic requires Python, batch-process through generators instead.

## Generator-Based Processing

Generators enable constant-memory processing of arbitrarily large datasets. Each stage yields one record at a time, composing into multi-stage ETL pipelines.

```python
import json
from collections.abc import Iterator

def extract(path: str) -> Iterator[dict]:
    with open(path) as f:
        for line in f:
            yield json.loads(line)

def transform(records: Iterator[dict]) -> Iterator[dict]:
    for record in records:
        if record["status"] == "valid":
            yield {
                "id": record["id"],
                "value": record["value"] * 2,
            }

def load(records: Iterator[dict], output: str) -> int:
    count = 0
    with open(output, "w") as f:
        for record in records:
            f.write(json.dumps(record) + "\n")
            count += 1
    return count

# Compose: constant memory regardless of input size
pipeline = transform(extract("input.jsonl"))
written = load(pipeline, "output.jsonl")
```

### When to Use Generators vs Polars

| Criterion | Generators | Polars |
|-----------|-----------|--------|
| Record-by-record logic | Preferred | Use `.map_elements()` as last resort |
| Tabular aggregation | Inefficient | Preferred |
| Memory constraint | Constant memory | Configurable via streaming |
| Custom Python logic per record | Natural | Slow (breaks vectorization) |
| Joins and group-by | Manual | Built-in, optimized |
| File format diversity | Any format | CSV, Parquet, JSON, NDJSON, IPC |

## Chunked File Processing

For files too large to load entirely, use Polars streaming or manual batching:

```python
import polars as pl

# Polars streaming: automatic chunking with bounded memory
result = (
    pl.scan_csv("large.csv")
    .filter(pl.col("value") > 0)
    .group_by("category")
    .agg(pl.col("value").mean())
    .sort("category")          # streaming group_by does not preserve order
    .collect()                 # engine="auto" resolves to streaming for lazy queries (Polars 2)
)
```

For custom per-batch logic, iterate `pl.scan_csv(path).collect_batches(chunk_size=N)` and process each `DataFrame` batch; `pl.read_csv_batched` was removed in Polars 2.

## File-Based Intermediate Storage

When pipeline stages produce intermediate results too large for memory, spill to disk using efficient formats:

| Format | Read Speed | Write Speed | Compression | Schema Preservation |
|--------|-----------|-------------|-------------|-------------------|
| Parquet | Fast | Fast | Excellent | Full type information |
| IPC/Arrow | Fastest | Fastest | Good | Full type information |
| NDJSON | Medium | Medium | None (use gzip) | Partial |
| CSV | Slow | Medium | None | None |

Prefer Parquet for intermediate storage. It preserves types, compresses well, and supports predicate pushdown when scanned lazily.

```python
# Spill intermediate results to Parquet
intermediate = stage_1_transform(input_data)
intermediate.write_parquet("tmp/stage_1.parquet")

# Resume from intermediate
stage_2_input = pl.scan_parquet("tmp/stage_1.parquet")
result = stage_2_transform(stage_2_input).collect()
```

## DuckDB and Arrow Interoperability

Polars, DuckDB and PyArrow share the Arrow columnar format, so handing a frame between them is a zero- or low-copy boundary, not a serialization step. Use DuckDB where SQL is the clearer expression (window-heavy reporting, ad hoc joins over Parquet directories) and keep the pipeline's typed boundary in Polars:

```python
import duckdb
import polars as pl

events = pl.scan_parquet("data/*.parquet").collect()
summary = duckdb.sql("select category, sum(value) as total from events group by category order by category").pl()
table = summary.to_arrow()   # pyarrow.Table for Arrow-native consumers
```

DuckDB resolves `events` from the enclosing Python scope (a replacement scan). Check the result schema at that boundary: DuckDB's `sum` over an integer column returns `HUGEINT`, which arrives in Polars as `Decimal(38, 0)` rather than `Int64` (verified with DuckDB 2.0.1 and Polars 2.1.0) — cast explicitly where the downstream model expects an integer.

## Pandas Interoperability

Some libraries require Pandas DataFrames. Convert at the boundary and return to Polars immediately:

```python
# Convert at library boundary
pandas_df = polars_df.to_pandas()
result = sklearn_function(pandas_df)

# Return to Polars
polars_result = pl.from_pandas(result)
```

Avoid keeping data in Pandas format longer than the library call requires. Polars-to-Pandas conversion copies data; minimize these transitions.

## Evidence

Polars benchmarks (2024-2026) demonstrate 5-30x speedups over Pandas for typical analytical queries. The Polars 2.0 behavior above (streaming default, `streaming=` removal, `read_csv_batched` removal) was re-verified 2026-11-22 against Polars 2.1.0. The TPC-H benchmark shows Polars outperforming Pandas, DuckDB, and Spark for datasets fitting in single-node memory. Generator-based processing maintains O(1) memory usage regardless of input size, confirmed through memory profiling of production ETL pipelines.

## Related

- [Data Validation](../../validation/reference/validation.md)
- [Streaming and Checkpointing](streaming.md)
- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)

## Sources

- https://docs.pola.rs/releases/upgrade/2/ — Polars 2 upgrade guide: streaming engine default for lazy `collect`, row-order caveat, `streaming=` removal, engine affinity (accessed 2026-11-22)
- https://pola.rs/posts/announcing-polars-2/ — Polars 2.0 announcement (accessed 2026-11-22)
- https://duckdb.org/docs/stable/guides/python/polars — DuckDB and Polars interchange via Arrow (accessed 2026-11-22)
