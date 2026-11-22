# Streaming and Checkpointing

Use bounded queues and generators for streaming; checkpoint long-running pipelines for fault tolerance.

Long-running data pipelines require streaming patterns with backpressure control and periodic checkpointing. Use bounded `asyncio.Queue` for producer-consumer flows, generators for memory-efficient sequential processing, and atomic checkpoints for fault tolerance. Every pipeline running longer than 5 minutes must support resume from the last checkpoint.

## Rationale

Unbounded queues and eager loading cause memory exhaustion on large datasets. Backpressure through bounded queues prevents producers from overwhelming consumers. Generator-based pipelines process data with constant memory regardless of input size. Checkpointing enables recovery from crashes without reprocessing completed work, reducing wasted compute in pipelines that run for hours. Atomic writes (write to temp file, then rename) prevent checkpoint corruption from partial writes during failures.

- [Streaming and Checkpointing](../reference/streaming.md)

## Examples

**Correct**:
```python
import asyncio
from pathlib import Path

# Bounded queue with backpressure
queue: asyncio.Queue[dict | None] = asyncio.Queue(maxsize=1000)

# Checkpoint with atomic write
def save_checkpoint(state: dict, path: Path) -> None:
    temp = path.with_suffix(".tmp")
    temp.write_text(json.dumps(state))
    temp.rename(path)  # Atomic on POSIX
```

**Incorrect**:
```python
# Unbounded queue risks OOM
queue: asyncio.Queue[dict] = asyncio.Queue()  # No maxsize

# Non-atomic checkpoint risks corruption
with open("checkpoint.json", "w") as f:
    json.dump(state, f)  # Partial write on crash corrupts file
```

## Exceptions

Short-lived pipelines under 5 minutes of runtime may omit checkpointing when reprocessing from scratch is acceptable. In-memory-only processing of datasets that fit comfortably in available RAM may use eager loading instead of streaming.

## Tooling

Use `asyncio.Queue(maxsize=N)` for async backpressure, `asyncio.Semaphore` for concurrency limiting, and `tqdm` for progress reporting. Store checkpoints as JSON for inspectability. Use `pathlib.Path` for atomic file operations.
