# Database API Principles

Database API principles for connection management, query patterns, resilience, and zero-downtime operations.

Principles governing how applications interact with relational databases. These principles apply across database engines and programming languages, covering connection lifecycle, query design, transaction safety, resilience, and schema evolution support.

## Connection Management

Applications share database connections through pools. A pool bounds the number of open connections, reuses them across requests, and reclaims idle resources. Pool sizing depends on workload characteristics: CPU-bound applications need fewer connections than I/O-bound ones. A useful starting point is `2 * CPU_cores + disk_spindles` for synchronous workloads, adjusted downward when using async drivers that multiplex work across fewer connections.

**Acquisition and release.** Acquire connections as late as possible and release them as early as possible. Hold a connection only for the duration of its transaction. Long-held connections starve other consumers and inflate pool pressure. Lazy acquisition defers the cost until the connection is actually needed; eager acquisition wastes resources when request paths do not require database access.

**Timeouts.** Configure three timeout boundaries:

| Timeout | Purpose | Typical Range |
|---------|---------|---------------|
| Connection acquisition | Fail fast when the pool is saturated | 1-5 seconds |
| Query execution | Prevent runaway queries | 5-30 seconds |
| Idle connection | Reclaim unused resources | 5-10 minutes |

Without acquisition timeouts, saturated pools cause cascading latency across the application.

**Read replicas.** Route read-only queries to replicas when available. This distributes load and reduces contention on the primary. The API layer should make replica routing explicit so callers understand consistency trade-offs: replica reads may return slightly stale data.

## Query Patterns

### Streaming Large Results

Fetching millions of rows into memory fails at scale. Server-side cursors or batched fetches bound memory by processing results incrementally. The application reads a fixed-size batch, processes it, then fetches the next. This pattern keeps memory constant regardless of result set size.

### Batched Writes

Individual inserts create round-trip overhead. Batch writes group multiple operations into a single database call. Batch size balances throughput against transaction log pressure. Typical batch sizes range from 500 to 5,000 rows, tuned by measuring throughput at different sizes.

### Row-Level Locking

Explicit locking coordinates concurrent access. `SELECT FOR UPDATE` acquires row-level locks for modification. The `SKIP LOCKED` variant enables work queue patterns by allowing concurrent consumers to claim different rows without blocking each other.

Lock only the rows you intend to modify. Broad locks cause contention; targeted locks enable concurrency.

### Parameterized Queries

All queries use parameterized placeholders. String interpolation into SQL creates injection vulnerabilities regardless of input sanitization. Parameterized queries separate data from logic at the protocol level.

## Transaction Design

Transactions group operations into atomic units. The choice of isolation level determines what concurrent transactions can observe.

| Isolation Level | Prevents | Allows | Use When |
|-----------------|----------|--------|----------|
| Read Committed | Dirty reads | Non-repeatable reads, phantoms | Most OLTP operations |
| Repeatable Read | Dirty reads, non-repeatable reads | Phantoms (in some engines) | Consistent snapshots needed |
| Serializable | All anomalies | Nothing | Correctness outweighs throughput |

Select the weakest isolation level that guarantees correctness. Stronger isolation reduces throughput through increased locking or serialization conflicts.

**Lock duration.** Transactions determine lock lifetime. Short transactions release locks quickly, improving concurrency. Move non-database work (API calls, file I/O, computation) outside the transaction boundary.

**Deadlock prevention.** Deadlocks occur when transactions acquire locks in different orders. Prevent them by establishing a consistent lock ordering: always lock tables and rows in the same sequence. When deadlocks remain possible, design operations to retry on deadlock detection.

**Savepoints.** Savepoints allow partial rollback within a transaction. They enable error recovery without abandoning all preceding work, which is useful in batch operations where individual items may fail independently.

## Error Handling and Resilience

Database connections fail. Networks partition. Queries time out. The API layer must handle these failures gracefully.

**Retry strategy.** Retry transient failures (connection resets, lock timeouts, serialization conflicts) with exponential backoff and jitter. Jitter prevents thundering herd effects when multiple clients retry simultaneously. Cap retries at 3-5 attempts before surfacing the error.

**Error classification.** Distinguish retryable errors from permanent failures:

- **Retryable**: Connection lost, lock timeout, serialization failure, temporary resource exhaustion
- **Permanent**: Constraint violation, syntax error, authentication failure, missing table

Retrying permanent errors wastes resources and delays error reporting.

**Health checks.** Validate connections before use. Stale connections from the pool may reference closed server-side state. A lightweight validation query (`SELECT 1`) confirms the connection is alive. Run validation on checkout or at periodic intervals, balancing reliability against validation overhead.

**Circuit breakers.** When the database is unreachable, stop sending requests. A circuit breaker tracks failure rates and opens when failures exceed a threshold, allowing the database time to recover. The breaker closes again after a probe request succeeds.

## Schema Evolution Support

Production databases evolve. The API layer must tolerate schema changes without downtime.

**Expand-contract pattern.** Breaking changes split across releases. First, expand: add the new column or table alongside the old. Then migrate data. Finally, contract: remove the old structure once all consumers have migrated. The API supports both old and new schemas during the transition period.

- [Database Migration Management](../../migrations/reference/migrations.md)

**Query resilience.** Avoid `SELECT *` in production code. Explicit column lists prevent breakage when new columns appear and avoid transferring unnecessary data. Queries that name their columns tolerate additive schema changes without modification.

**Index creation.** Create indexes concurrently in production to avoid blocking writes. Standard index creation acquires a table-level lock that blocks all inserts and updates for the duration of the build. Concurrent index creation trades build speed for availability.

**Backward compatibility.** Design queries to tolerate both pre-migration and post-migration schemas during deployment windows. Column renames, type changes, and constraint additions each require transition strategies that keep the application functional throughout the migration.

## Multi-Database Compatibility

Applications sometimes run against multiple database engines: one for production, another for testing or local development.

**Standard SQL first.** Prefer SQL standard syntax over dialect-specific extensions. Standard SQL works across engines without modification. Reserve dialect-specific features for cases where they provide substantial benefits (advisory locks, `SKIP LOCKED`, JSONB operations).

**Feature detection.** When dialect-specific features are necessary, detect feature availability rather than checking database version strings. Feature detection adapts to forks, patches, and backports that version checking misses.

**Abstraction boundaries.** Isolate dialect-specific code behind interfaces. The application interacts with the abstraction; the implementation handles dialect differences. This confines migration effort when adding or removing database engine support.

**Testing coverage.** Test against all supported database engines. Behavior differences between engines surface only through execution. SQL that parses correctly on one engine may fail or produce different results on another. Automated test suites running against each engine catch these discrepancies before deployment.

## Anti-Patterns

| Anti-Pattern | Problem | Alternative |
|--------------|---------|-------------|
| Connection per query | Pool exhaustion under load, connection overhead per operation | Pool connections at the application level |
| `SELECT *` | Schema coupling, wasted network and memory I/O | Name columns explicitly |
| Unbounded transactions | Extended lock holding, deadlock risk, replication lag | Scope transactions tightly around data operations |
| N+1 queries | Latency multiplied by row count | Batch fetches or joins |
| Ignoring isolation levels | Phantom reads, lost updates, non-repeatable reads | Choose isolation level deliberately per operation |
| String-concatenated SQL | Injection vulnerabilities | Parameterized queries always |
| Retrying permanent errors | Wasted resources, delayed error reporting | Classify errors before retry |

## Related

- [Job Repositories and State Machines](../../jobs/reference/jobs.md)
