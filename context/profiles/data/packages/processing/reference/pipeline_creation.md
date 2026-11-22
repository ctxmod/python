# Pipeline Creation

Pipeline creation workflow covering source identification, stage design, processing pattern selection, and checkpointing setup.

Procedure for designing and implementing data pipelines that transform raw input into validated output. Pipelines combine validation at ingestion, transformation through Polars or generators, and output with checkpointing for fault tolerance. The result is a pipeline that processes data reliably with bounded memory and supports resume after failures.

## Prerequisites

Gather before starting:

- **Data sources**: File paths, API endpoints, database queries, or message queue topics providing input data.
- **Transformation requirements**: Operations to apply (filtering, mapping, aggregation, joining, enrichment).
- **Output format**: Target format (Parquet, NDJSON, database table, API call) and destination.
- **Volume estimate**: Expected record count and size to determine processing pattern.
- **Reliability requirements**: Whether the pipeline requires checkpointing, retry logic, or idempotent writes.

## Creation Workflow

### 1. Define Data Models

Create Pydantic models for input validation and typed output representation:

```python
from pydantic import BaseModel, ConfigDict
from datetime import datetime

class RawEvent(BaseModel):
    model_config = ConfigDict(strict=True)
    event_id: str
    user_id: str
    action: str
    timestamp: datetime
    payload: dict[str, object]

class ProcessedEvent(BaseModel):
    model_config = ConfigDict(strict=True, frozen=True)
    event_id: str
    user_id: str
    action: str
    date: str
    value: float
```

Define models before writing transformation logic. Models serve as executable documentation of data contracts between pipeline stages.

- [Data Validation](../../validation/reference/validation.md)

### 2. Choose Processing Pattern

Select the processing approach based on data volume and transformation complexity:

| Pattern | Volume | Transformations | Memory |
|---------|--------|----------------|--------|
| Polars lazy | Medium to large | Tabular (filter, join, aggregate) | Bounded via streaming |
| Generator pipeline | Any size | Record-by-record custom logic | Constant |
| Polars batched | Very large | Tabular with custom per-batch logic | Controlled chunks |
| Hybrid (generator + Polars) | Large | Mix of custom and tabular | Bounded |

For most pipelines, start with Polars lazy evaluation. Switch to generators when per-record Python logic is unavoidable. Use hybrid approaches when pipelines combine record-level validation with tabular aggregation.

- [Data Processing](processing.md)

### 3. Implement Extraction

Create the data source reader. Use lazy scanning for file sources and generators for streaming sources:

```python
import polars as pl
from collections.abc import Iterator

# File source with Polars lazy scanning
def scan_events(path: str) -> pl.LazyFrame:
    return pl.scan_ndjson(path)

# Streaming source with generator
def stream_events(path: str) -> Iterator[RawEvent]:
    with open(path) as f:
        for line in f:
            yield RawEvent.model_validate_json(line)
```

Validate data at the extraction boundary. For Polars pipelines, validate schemas after scanning. For generator pipelines, validate each record during extraction.

### 4. Implement Transformation

Chain transformation operations. Keep each function focused on a single concern:

```python
def filter_valid(lf: pl.LazyFrame) -> pl.LazyFrame:
    return lf.filter(pl.col("action").is_in(["click", "purchase", "view"]))

def enrich_dates(lf: pl.LazyFrame) -> pl.LazyFrame:
    return lf.with_columns(pl.col("timestamp").dt.date().cast(pl.Utf8).alias("date"))

def aggregate_by_user(lf: pl.LazyFrame) -> pl.LazyFrame:
    return lf.group_by("user_id", "date").agg(pl.col("event_id").count().alias("event_count"))
```

Compose transformations through function chaining. For generator pipelines, apply the same pattern with `Iterator` input/output types instead of `LazyFrame`.

### 5. Implement Loading

Write output using atomic writes (temp file + rename) to prevent partial results:

```python
from pathlib import Path

def write_parquet(df: pl.DataFrame, output: Path) -> None:
    temp = output.with_suffix(".tmp.parquet")
    df.write_parquet(temp)
    temp.rename(output)
```

### 6. Add Checkpointing and Progress

For pipelines running longer than 5 minutes, add checkpointing at stage boundaries or regular intervals. Create a dataclass holding step counter and records processed, with `save()` using atomic writes (temp file + rename) and a `load()` classmethod for resume. Use `tqdm` for CLI progress bars and structured logging for headless services.

- [Streaming and Checkpointing](streaming.md)

### 7. Create Entry Point

Wire the pipeline stages into an executable entry point:

```python
from pathlib import Path

def run_pipeline(input_path: str, output_dir: str) -> None:
    output = Path(output_dir)
    output.mkdir(parents=True, exist_ok=True)

    result = aggregate_by_user(enrich_dates(filter_valid(scan_events(input_path)))).collect()
    write_parquet(result, output / "aggregated.parquet")
```

## Decision Support

### Batch vs Streaming

| Factor | Batch | Streaming |
|--------|-------|-----------|
| Data arrives | All at once (files, dumps) | Continuously (queues, APIs) |
| Latency requirement | Minutes to hours acceptable | Seconds to minutes required |
| Processing model | Polars lazy, full-dataset ops | Async generators, per-record |
| Checkpointing | Per-stage | Per-interval |

### Single File vs Partitioned Output

| Factor | Single File | Partitioned |
|--------|-------------|-------------|
| Output size | <1 GB | >1 GB |
| Query pattern | Full scan | Filtered by partition key |
| Write pattern | Atomic replace | Append per partition |
| Format | Parquet, CSV | Parquet with partition directories |

## Troubleshooting

**OOM during collection**: Switch from eager to lazy evaluation. Use `collect(streaming=True)` for large results. Profile with `pl.Config.set_verbose(True)`.

**Slow per-record processing**: Move logic from generators to Polars expressions. `.apply()` is a last resort; prefer vectorized operations.

**Schema mismatch between stages**: Validate intermediate schemas with explicit column selection and type casting between stages.

## Related

- [Project Setup](../../../../core/packages/toolchain/reference/project_setup.md)
