# Migration Workflow

Migration workflow procedure covering Alembic initialization, multi-location directory setup, SQLite and PostgreSQL testing, and production deployment checklists.

Procedure for managing database migrations with Alembic in SQLAlchemy 2.0 async projects. The workflow covers initial setup, multi-location configuration for monorepo packages, testing against both SQLite and PostgreSQL, and production deployment. The result is a migration system that supports independent package development while maintaining a unified schema history.

## Prerequisites

Gather before starting:

- **SQLAlchemy 2.0 async engine**: Configured with `asyncpg` for PostgreSQL.
- **Model definitions**: ORM models using `Mapped` and `mapped_column` declarative style.
- **Database instances**: Local PostgreSQL for integration testing, production PostgreSQL connection details.
- **Package structure**: For multi-location setups, identify which packages own which tables.

Install required dependencies:

```bash
uv add alembic "sqlalchemy[asyncio]" asyncpg   # [asyncio] carries greenlet from SQLAlchemy 2.1
uv add --dev aiosqlite pytest-asyncio
```

## Setup Workflow

### 1. Initialize Alembic

Create the Alembic configuration at the project root:

```bash
uv run alembic init -t async alembic
```

This creates `alembic.ini` and an async `alembic/env.py` whose online runner already uses an async engine and `connection.run_sync`; there is no need to hand-replace the generic template's runner. `-t pyproject_async` is the alternative when the project keeps Alembic settings in `[tool.alembic]` of `pyproject.toml` (pyproject configuration arrived in Alembic 1.16; `alembic list_templates` shows `generic`, `async`, `pyproject`, `pyproject_async` and `multidb` on 1.20.1), with `alembic.ini` reduced to URL and logging.

Set `target_metadata = Base.metadata` after importing all model modules explicitly. Missing imports cause autogenerate to skip tables.

- [Database Migration Management](migrations.md)

### 2. Configure the Database URL

Set the database URL in `alembic.ini` or override it through environment variables in `env.py`:

```python
# alembic/env.py
import os

def get_url() -> str:
    return os.environ.get("DATABASE_URL", "postgresql+asyncpg://localhost/mydb")
```

For testing, override the URL to point to SQLite (`sqlite+aiosqlite://`) or a test PostgreSQL database.

### 3. Set Up Multi-Location Migrations

For monorepo projects where multiple packages contribute tables, configure `version_locations` in `alembic.ini`:

```ini
[alembic]
script_location = alembic
path_separator = ;   # without it, the semicolon list collapses into one bogus directory
version_locations = alembic/versions;packages/core/migrations;packages/jobs/migrations;packages/analytics/migrations
```

Create each migration directory with an empty `__init__.py`:

```
project/
  alembic/
    env.py
    versions/         # Shared migrations (base tables, cross-cutting)
  packages/
    core/
      migrations/     # Core domain migrations
      models.py
    jobs/
      migrations/     # Job repository migrations
      models.py
    analytics/
      migrations/     # Analytics migrations
      models.py
```

Assign branch labels per location to prevent cross-location dependency confusion. When generating a migration for a specific package, use:

```bash
uv run alembic revision --autogenerate -m "add job history table" --version-path packages/jobs/migrations
```

### 4. Generate the Initial Migration

Import all models and generate the first migration:

```bash
uv run alembic revision --autogenerate -m "initial schema"
```

Review the generated migration file. Verify that all expected tables appear. Add dialect-specific operations (PostgreSQL ENUM types, partial indexes) manually where autogenerate omits them:

```python
def upgrade():
    dialect = op.get_context().dialect.name
    if dialect == "postgresql":
        op.execute("CREATE TYPE job_status AS ENUM ('pending', 'queued', 'running', ...)")

    op.create_table("jobs", ...)
```

### 5. Apply and Verify

Apply the migration and verify the schema:

```bash
uv run alembic upgrade head
uv run alembic check  # Verify no pending changes
```

## Testing Workflow

### Fast Feedback: SQLite In-Memory

Create a pytest fixture that sets up an `aiosqlite` in-memory engine, overrides `sqlalchemy.url` in the Alembic config, runs `upgrade(config, "head")`, yields the engine, then runs `downgrade(config, "base")` and disposes. This catches syntax errors and basic schema issues in seconds.

### Full Compatibility: Local PostgreSQL

Run a Docker PostgreSQL instance matching the production version (`docker run -d --name test-postgres -e POSTGRES_PASSWORD=test -p 5433:5432 postgres:16`). Test with `DATABASE_URL=postgresql+asyncpg://postgres:test@localhost:5433/test`. Verify ENUM types, JSONB operations, partial indexes, and lock behavior.

### Migration Test Checklist

| Test | Database | Validates |
|------|----------|-----------|
| `upgrade head` succeeds | Both | Forward migration chain intact |
| `downgrade base` succeeds | Both | Rollback chain intact |
| Round-trip (`up -> down -> up`) | Both | Idempotent migration execution |
| Autogenerate produces empty diff after `upgrade head` | Both | Models match applied schema |
| Data migration transforms records correctly | PostgreSQL | Data integrity during backfills |
| Concurrent access during migration | PostgreSQL | Lock behavior under load |

## Production Deployment

### Pre-Deployment

| Step | Command | Purpose |
|------|---------|---------|
| Check current revision | `alembic current` | Confirm starting point |
| Review migration SQL | `alembic upgrade head --sql` | Inspect exact DDL |
| Test downgrade | `alembic downgrade -1` (staging) | Verify rollback works |
| Back up database | `pg_dump -Fc dbname > backup.dump` | Recovery safety net |

### Deployment Sequence

Apply migrations before deploying application code. The application expects the new schema to exist.

```
1. Back up database
2. Run alembic upgrade head
3. Verify alembic current matches expected revision
4. Deploy application code
5. Verify application health checks pass
6. Monitor error rates for 15 minutes
```

### Rollback Sequence

If issues arise after migration, revert application code first, then run `alembic downgrade -1` to the previous revision, verify `alembic current` matches, and confirm application health.

### Zero-Downtime Operations

For breaking schema changes, split into separate migration files across multiple releases: first add the nullable column (brief lock), then backfill data in batches (no lock), then set NOT NULL (brief lock). This expand-contract pattern minimizes lock duration.

- [Database API Principles](../../database/reference/database_principles.md)

## Decision Support

### Single vs Multi-Location

| Factor | Single Location | Multi-Location |
|--------|----------------|----------------|
| Project size | Single package | Monorepo with 2+ packages |
| Team structure | One team | Multiple teams owning different domains |
| Migration conflicts | Rare | Frequent without branch labels |
| Setup complexity | Minimal | Moderate (branch labels, merge migrations) |

### When to Write Manual Migrations

| Scenario | Autogenerate | Manual |
|----------|-------------|--------|
| Add/drop table | Yes | No |
| Add/drop column | Yes | No |
| Data backfill | No | Yes |
| Index rename | No | Yes |
| ENUM value addition (PostgreSQL) | No | Yes |
| Conditional dialect logic | No | Yes |

## Troubleshooting

**"Multiple head revisions" error**: Two migrations were created without a common ancestor. Generate a merge: `alembic merge heads -m "merge branches"`.

**Autogenerate produces no changes**: Verify all model modules import before `target_metadata` assignment. Check that models inherit from the correct `Base` class.

**Migration fails on SQLite but passes on PostgreSQL**: The migration uses PostgreSQL-specific features without dialect branching. Add `if dialect == "postgresql"` guards for unsupported operations.

**Lock timeout during production migration**: The migration holds an `ACCESS EXCLUSIVE` lock too long. Split into smaller operations. Add `SET lock_timeout = '5s'` at the start of the migration to fail fast rather than block.

## Related

- [Job Repository Setup](../../jobs/reference/job_repository_setup.md)

## Sources

- https://alembic.sqlalchemy.org/en/latest/tutorial.html — `alembic init -t async` / `-t pyproject` templates and `path_separator` (accessed 2026-11-22)
