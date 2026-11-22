# Job Repositories and State Machines

Job repository and state machine principles covering PostgreSQL-backed queues, python-statemachine lifecycle, checkpointing integration, optimistic locking, and DAG dependencies.

Long-running data processes require durable state tracking beyond what in-memory variables provide. A PostgreSQL-backed job repository persists job metadata, progress, and checkpoints through ACID transactions. Explicit state machines enforce valid lifecycle transitions and prevent jobs from entering inconsistent states. Together, these patterns enable crash recovery, concurrent execution, and operational visibility across distributed workers.

## Job Repository Pattern

The job repository stores each job as a row in PostgreSQL with status, payload, checkpoint data, and a `version` column for optimistic locking. JSONB payload columns hold both job configuration and checkpoint state, avoiding separate storage mechanisms.

### Core Schema Design

A `jobs` table tracks status, execution metadata, and checkpoint state. A `job_history` table records every state transition for audit and debugging. Use a PostgreSQL `ENUM` type for status values to enforce valid states at the database level. Include a `version` column (integer, starting at 1) for optimistic locking, a `visible_at` timestamp for crash recovery, and a `depends_on` integer array for dependency tracking.

Index the `jobs` table on `(status, visible_at)` with a partial index filtering active statuses (`pending`, `queued`, `running`). Add a secondary index on `(job_type, status)` for type-filtered queries.

### Job Dequeuing with SKIP LOCKED

Use `SELECT ... FOR UPDATE SKIP LOCKED` inside an `UPDATE ... RETURNING` statement to atomically claim the next available job. This prevents multiple workers from grabbing the same job without explicit distributed locks. Order by priority (descending) and creation time (ascending) to process high-priority jobs first while maintaining FIFO ordering within the same priority level.

The dequeue query should atomically set status to `running`, assign the worker ID, record `started_at`, set `visible_at` to a future timestamp (the visibility timeout), and increment the `version` column.

### Visibility Timeout and Crash Recovery

Set `visible_at` to `now() + interval` when a worker claims a job. A background process periodically scans for jobs where `status = 'running'` and `visible_at < now()`, returning them to `queued` status. This handles worker crashes without manual intervention. Typical visibility timeouts range from 15 to 60 minutes depending on expected job duration.

## State Machine Lifecycle

Jobs follow a deterministic state machine with these states and transitions:

| State | Description | Valid Transitions |
|-------|-------------|-------------------|
| `initialized` | Created, not yet submitted | `pending` |
| `pending` | Awaiting dependency completion | `queued`, `cancelled` |
| `queued` | Ready for worker pickup | `running`, `cancelled` |
| `running` | Actively executing | `completed`, `failed`, `paused`, `cancelled` |
| `paused` | Suspended with checkpoint preserved | `running` |
| `completed` | Finished successfully | (final) |
| `failed` | Execution error | `queued` (retry), `dead_letter` |
| `cancelled` | Externally cancelled | (final) |
| `dead_letter` | Exceeded retry limit | (final) |

Use `python-statemachine` for declarative state definitions. Define states as class attributes and transitions as directed edges between them. Hook `before_transition` for validation and `after_transition` for persistence. The library's native async support integrates with async job processors without blocking the event loop.

### Retry and Dead Letter Handling

Track `retry_count` and `max_retries` per job. When a job fails, increment `retry_count`. If `retry_count < max_retries`, transition back to `queued` with exponential backoff applied through `visible_at`. If retries are exhausted, transition to `dead_letter` for manual investigation. Record every failure in `job_history` with the error message and stack trace.

## Checkpointing Integration

Store checkpoint data inside the job's JSONB `payload` under a `checkpoint` key. Checkpoints follow the same structure used in streaming pipelines: step counter, set of processed IDs, metrics dictionary, and arbitrary state. Save checkpoints at regular intervals (every N items or M seconds) using `jsonb_set()` to update only the checkpoint portion of the payload.

On job resume, load the checkpoint from the payload and skip already-processed items. This avoids separate checkpoint storage while maintaining compatibility with the streaming checkpoint pattern.

[Streaming and Checkpointing](../../processing/reference/streaming.md)

### Checkpoint Frequency Guidelines

| Job Duration | Checkpoint Interval | Acceptable Lost Progress |
|-------------|--------------------|-----------------------|
| Under 30 minutes | Every 1,000 items | 1-2 minutes |
| 30 minutes to 2 hours | Every 5,000 items | 5-10 minutes |
| Over 2 hours | Every 10,000 items or 15 minutes | 10-15 minutes |

Balance checkpoint frequency against write overhead. Each checkpoint updates the job row, consuming a database round trip.

## Optimistic Locking

Every state-changing operation includes a `WHERE version = $expected` clause and increments `version` on success. If the update affects zero rows, another worker modified the job concurrently. Handle version conflicts with retry logic using exponential backoff and jitter (base delay of 100ms, maximum 3 attempts). This eliminates distributed locks while preventing lost updates.

The `version` column is the row-level realization of an optimistic-concurrency precondition — the same shape the base profile names `workflow_revision` at the application layer. It is a precondition, never a deduplication key: a mismatch means reload and re-decide, not retry the same write unchanged. Retrying a lost update is safe here precisely because the retry re-reads state first.

### Relation to the Base Profile's Idempotency Levels

Job-level idempotency is one application of a three-level model this profile does not restate. The base profile's error-handling reference owns the three levels and their identity formulas; the mapping for job work is:

| Job-side mechanism | Level it realizes |
|---|---|
| An idempotency key carried on the job's submission, scoped to submitter and operation | Command idempotency — a resubmitted job returns the original outcome rather than enqueuing a second one |
| The dequeue claim plus the processed-item set in the checkpoint | Delivery deduplication — a redelivered or re-claimed item produces no repeated effect |
| The `WHERE version = $expected` precondition on each state transition | The optimistic-concurrency precondition the levels above consume |

Read the base profile's reference for the rules; this table exists only so the job vocabulary and the application vocabulary are recognizably the same thing.

[Error Handling and Resilience](../../../../core/packages/errors/reference/errors.md)

## Job Dependencies and DAGs

Jobs can declare dependencies through a `depends_on` integer array referencing other job IDs. Jobs with unresolved dependencies start in `pending` status. A dependency resolver -- triggered after each job completion -- checks whether pending jobs have all dependencies satisfied and transitions them to `queued`.

For workflows requiring complex DAG orchestration with conditional branching, parallel fan-out, or dynamic task generation, use a dedicated orchestration platform (Temporal for mission-critical durability, Prefect for Python-native pipelines, Dagster for data asset management) instead of manual dependency tracking.

## Concurrent Execution

Control worker concurrency with `asyncio.Semaphore` to limit the number of simultaneously running jobs per worker process. Each worker polls for available jobs, claims them through the SKIP LOCKED mechanism, and processes them within the semaphore boundary. Multiple worker processes across hosts can safely share the same job queue through PostgreSQL's row-level locking.

## Testing Long-Running Processes

Test each concern independently:

| Concern | Test Strategy |
|---------|--------------|
| State machine transitions | Unit tests: verify valid transitions succeed, invalid transitions raise errors |
| Optimistic locking | Integration tests: simulate concurrent version conflicts |
| Checkpoint persistence | Integration tests: save, reload, and verify checkpoint data |
| Dequeue atomicity | Integration tests: verify SKIP LOCKED prevents duplicate claims |
| End-to-end lifecycle | Integration tests: enqueue, dequeue, checkpoint, complete |

Use a local PostgreSQL instance for integration tests. State machine unit tests run without a database by testing the `python-statemachine` instance directly.

## Anti-Patterns

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| In-memory job state | Lost on crash | Persist to PostgreSQL |
| Polling without SKIP LOCKED | Duplicate claims | Atomic dequeue with `FOR UPDATE SKIP LOCKED` |
| Unbounded retries | Queue starvation | `max_retries` with dead letter transition |
| Non-idempotent operations | Duplicate side effects on retry | Idempotency key per submitted job, recorded durably before the side effect runs |
| Missing visibility timeout | Stuck jobs after worker crash | `visible_at` with background recovery |
| Tight coupling to orchestration platform | Vendor lock-in | Repository pattern with pluggable executors |

## Evidence

PostgreSQL `SKIP LOCKED` (introduced in 9.5) provides non-blocking concurrent access confirmed through production benchmarks handling 10,000+ dequeue operations per second. The `python-statemachine` library (v2.3+) provides native async support with built-in persistence adapters. PgQueuer benchmarks demonstrate reliable job processing with LISTEN/NOTIFY for real-time dequeuing at scale. Production pipeline analysis (2024-2026) shows that integrated checkpointing reduces mean recovery time from hours to minutes for long-running jobs.

## Related

- [Data Processing](../../processing/reference/processing.md)
- [Database API Principles](../../database/reference/database_principles.md)
