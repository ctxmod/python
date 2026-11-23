# Job Repository Setup

Job repository setup procedure covering database schema creation, state machine implementation, checkpoint integration, and testing workflows.

Procedure for implementing a PostgreSQL-backed job repository with state machine lifecycle management. The workflow creates the database schema, implements the state machine, integrates checkpointing with the job payload, and establishes testing patterns. The result is a job processing system that persists state through crashes, supports concurrent workers, and resumes from checkpoints on restart.

## Prerequisites

Gather before starting:

- **PostgreSQL instance**: Connection string for the target database. Local Docker or native PostgreSQL for development.
- **SQLAlchemy 2.0 async configured**: Async engine and session factory established per the database integration procedure.
- **Schema design**: Job types, payload structures, and dependency relationships defined in outline form.
- **Checkpoint requirements**: Which job types need checkpointing, checkpoint interval targets.

Install required dependencies:

```bash
uv add "sqlalchemy[asyncio]" asyncpg python-statemachine pgqueuer   # [asyncio] carries greenlet from SQLAlchemy 2.1
uv add --dev aiosqlite pytest-asyncio
```

## Setup Workflow

### 1. Create the Job Schema

Define the jobs table with status tracking, optimistic locking, and JSONB payload storage. Create an Alembic migration for the schema:

```python
# models/job.py
from sqlalchemy import String, Integer, DateTime, JSON, ARRAY
from sqlalchemy.orm import Mapped, mapped_column
from datetime import datetime

class Job(Base):
    __tablename__ = "jobs"
    id: Mapped[int] = mapped_column(primary_key=True)
    status: Mapped[str] = mapped_column(String(20), default="pending")
    job_type: Mapped[str] = mapped_column(String(100), index=True)
    priority: Mapped[int] = mapped_column(Integer, default=0)
    payload: Mapped[dict] = mapped_column(JSON, default=dict)
    current_state: Mapped[str] = mapped_column(String(50), default="initialized")
    version: Mapped[int] = mapped_column(Integer, default=1)  # Optimistic locking
    retry_count: Mapped[int] = mapped_column(Integer, default=0)
    max_retries: Mapped[int] = mapped_column(Integer, default=3)
    worker_id: Mapped[str | None] = mapped_column(String(100), nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    visible_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    depends_on: Mapped[list[int]] = mapped_column(ARRAY(Integer), default=list)
    last_error: Mapped[str | None] = mapped_column(nullable=True)
```

Create a `JobHistory` table alongside for audit trail tracking (job_id, from_state, to_state, trigger, metadata, timestamp).

- [Job Repositories and State Machines](jobs.md)

### 2. Implement the State Machine

Define the job lifecycle using `python-statemachine` with declarative states and transitions:

```python
# state/job_machine.py
from statemachine import StateMachine, State

class JobStateMachine(StateMachine):
    initialized = State(initial=True)
    pending = State()
    queued = State()
    running = State()
    paused = State()
    completed = State(final=True)
    failed = State()
    cancelled = State(final=True)
    dead_letter = State(final=True)

    submit = initialized.to(pending)
    enqueue = pending.to(queued)
    start = queued.to(running)
    complete = running.to(completed)
    fail = running.to(failed)
    pause = running.to(paused)
    resume = paused.to(running)
    cancel = pending.to(cancelled) | queued.to(cancelled) | running.to(cancelled)
    retry = failed.to(queued)
    abandon = failed.to(dead_letter)
```

Hook `after_transition` to persist state changes to the database and record entries in the job history table.

### 3. Implement the Job Repository

Create a repository class encapsulating job queries with atomic dequeue using SKIP LOCKED:

```python
# repositories/jobs.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text

class JobRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def dequeue(self, job_type: str | None = None, worker_id: str = "") -> Job | None:
        result = await self._session.execute(text("""
            UPDATE jobs SET status = 'running', worker_id = :worker_id,
                started_at = now(), visible_at = now() + interval '30 minutes',
                updated_at = now(), version = version + 1
            WHERE id = (
                SELECT id FROM jobs
                WHERE status = 'queued' AND visible_at <= now()
                    AND (:job_type IS NULL OR job_type = :job_type)
                ORDER BY priority DESC, created_at ASC
                LIMIT 1 FOR UPDATE SKIP LOCKED
            ) RETURNING *
        """), {"job_type": job_type, "worker_id": worker_id})
        row = result.fetchone()
        return Job(**row._mapping) if row else None
```

Add methods for `enqueue`, `update_state` (with optimistic locking via `WHERE version = :expected_version`), `save_checkpoint`, and `check_dependencies`.

### 4. Integrate Checkpointing

Store checkpoint data in the job payload under a `checkpoint` key. Implement save and load methods that operate on the JSONB payload:

```python
# processors/checkpointing.py
from dataclasses import dataclass, field

@dataclass
class JobCheckpoint:
    step: int = 0
    processed_ids: set[str] = field(default_factory=set)
    metrics: dict[str, float] = field(default_factory=dict)
    state: dict = field(default_factory=dict)

class CheckpointingProcessor:
    def __init__(self, job_id: int, repo: JobRepository, interval: int = 1000):
        self.job_id = job_id
        self.repo = repo
        self.interval = interval

    async def run(self) -> None:
        checkpoint = await self.repo.load_checkpoint(self.job_id)
        async for item in self.stream_from(checkpoint.step):
            if item.id in checkpoint.processed_ids:
                continue
            await self.process(item)
            checkpoint.step += 1
            checkpoint.processed_ids.add(item.id)
            if checkpoint.step % self.interval == 0:
                await self.repo.save_checkpoint(self.job_id, checkpoint)
        await self.repo.save_checkpoint(self.job_id, checkpoint)
```

This pattern matches the streaming checkpoint structure, enabling jobs to resume from the last saved position after failures.

- [Streaming and Checkpointing](../../processing/reference/streaming.md)

### 5. Implement Worker with Concurrency Control

Create a worker that polls for jobs with bounded concurrency using `asyncio.Semaphore(max_concurrent)`. The worker loop dequeues a job, spawns a processing task within the semaphore boundary, and sleeps when no jobs are available. Each processing task creates its own session, instantiates a `CheckpointingProcessor`, runs the job, and transitions to `completed` or handles failure on exception.

### 6. Test the Job Repository

Write tests at three levels:

| Level | Database | Tests |
|-------|----------|-------|
| Unit (state machine) | None | Valid transitions succeed, invalid transitions raise `TransitionNotAllowed` |
| Integration | Local PostgreSQL | Dequeue atomicity, optimistic locking conflicts, checkpoint save/load |
| End-to-end | Local PostgreSQL | Full lifecycle: enqueue with dependencies, resolve, dequeue, checkpoint, complete |

State machine unit tests verify transition logic without a database. Integration tests confirm SKIP LOCKED prevents duplicate claims, stale version updates fail, and checkpoint data round-trips correctly. End-to-end tests exercise the full lifecycle and verify the job history table contains all expected transitions.

## Decision Support

### PgQueuer vs Custom Implementation

| Factor | PgQueuer | Custom |
|--------|----------|--------|
| Setup time | Minutes | Hours |
| LISTEN/NOTIFY | Built-in | Manual |
| Customization | Limited | Full control |
| Checkpoint integration | Via payload | Native |
| State machine | External | Integrated |

Use PgQueuer for straightforward job queues. Build custom when the job lifecycle requires integrated state machines, complex dependency resolution, or tightly coupled checkpointing.

## Troubleshooting

**Jobs stuck in running state**: Check `visible_at` values. Verify the recovery background process runs and returns stalled jobs to `queued`. Increase visibility timeout if jobs legitimately run longer than the timeout.

**Duplicate job processing**: Verify SKIP LOCKED is present in the dequeue query. Check that the dequeue runs inside a transaction. Confirm workers use unique `worker_id` values.

**Checkpoint data too large**: Move large checkpoint state (processed ID sets exceeding 100K entries) to a separate table indexed by job ID. Keep only summary metrics in the job payload.

- [Migration Workflow](../../migrations/reference/migration_workflow.md)

## Related

- [Database API Principles](../../database/reference/database_principles.md)

## Sources

- https://docs.sqlalchemy.org/en/21/changelog/migration_21.html — greenlet moved to the `sqlalchemy[asyncio]` extra in 2.1 (accessed 2026-11-22)
