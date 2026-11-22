# Job Repository and State Machine

Use PostgreSQL SKIP LOCKED for job queues with explicit state machines; persist state before execution.

Use PostgreSQL with `SELECT ... FOR UPDATE SKIP LOCKED` for job queue dequeuing. Define job lifecycles as explicit state machines using `python-statemachine`. Persist job state to the database before beginning execution, and integrate checkpoints into the job payload for crash recovery.

## Rationale

In-memory job state vanishes on worker crashes, leaving pipelines in an unrecoverable position. PostgreSQL row-level locking with SKIP LOCKED provides atomic job claiming without distributed lock infrastructure. Explicit state machines prevent invalid transitions (such as moving directly from `pending` to `completed`) that produce inconsistent data. Persisting state before execution guarantees that crashes leave a recoverable record rather than silent data loss. Optimistic locking through a version column eliminates distributed locks while preventing concurrent workers from overwriting each other's progress.

[Job Repositories and State Machines](../reference/jobs.md)

## Examples

**Correct**:
```python
from sqlalchemy import update, select

# Atomic dequeue with SKIP LOCKED
stmt = (
    select(Job)
    .where(Job.status == "queued", Job.visible_at <= now())
    .order_by(Job.priority.desc(), Job.created_at)
    .limit(1)
    .with_for_update(skip_locked=True)
)
job = await session.scalar(stmt)
if job:
    job.status = "running"
    job.version += 1
    job.started_at = now()
    await session.commit()  # State persisted before execution begins
```

**Incorrect**:
```python
# No SKIP LOCKED: multiple workers claim the same job
job = await session.scalar(select(Job).where(Job.status == "queued").limit(1))
job.status = "running"
await session.commit()
# No version check: concurrent updates overwrite each other
```

## Exceptions

Lightweight task dispatching for sub-second operations where crash recovery adds unnecessary overhead may use in-memory queues. Jobs processed within a single request-response cycle do not require the full state machine pattern.

## Tooling

Use `python-statemachine` for state definitions (`uv add python-statemachine`). Store checkpoints as JSONB within the job row using `jsonb_set()` for partial updates. Use `asyncio.Semaphore` for per-worker concurrency limits.
