# Database Migrations

Use Alembic with SQLAlchemy 2.0 async for migrations; support both SQLite and PostgreSQL with expand-contract deployments.

Use Alembic with SQLAlchemy 2.0 async engines for all schema migrations. Configure multi-location migration sourcing for packages with independent migration directories. Write dialect-aware migrations that run against both SQLite and PostgreSQL. Apply the expand-contract pattern for zero-downtime production deployments.

## Rationale

Schema changes without migration tooling produce unreproducible database states across environments. Alembic's autogeneration detects model-to-schema drift, while explicit async engine integration keeps migrations compatible with `asyncpg` and `aiosqlite` drivers. Multi-location sourcing enables independent packages to own their migrations without merge conflicts in a single directory. The expand-contract pattern decomposes breaking changes into backward-compatible steps, preventing downtime during deployment. Testing against both SQLite (for speed) and PostgreSQL (for fidelity) catches dialect-specific issues before they reach production.

- [Database Migration Management](../reference/migrations.md)

## Examples

**Correct**:
```python
from alembic import op
import sqlalchemy as sa

def upgrade():
    dialect = op.get_context().dialect.name
    # Expand: add nullable column first
    op.add_column("jobs", sa.Column("priority", sa.Integer(), nullable=True))
    if dialect == "postgresql":
        op.execute("UPDATE jobs SET priority = 0 WHERE priority IS NULL")
        op.alter_column("jobs", "priority", nullable=False)

def downgrade():
    op.drop_column("jobs", "priority")
```

**Incorrect**:
```python
def upgrade():
    # Non-nullable column without default blocks writes during deployment
    op.add_column("jobs", sa.Column("priority", sa.Integer(), nullable=False))
    # No downgrade function: rollback impossible
```

## Exceptions

Prototype databases used exclusively for local experimentation may skip multi-location configuration. Initial project setup before production deployment may use destructive migrations (drop and recreate) until the schema stabilizes.

## Tooling

Configure `version_locations` in `alembic.ini` for multi-package support. Use `aiosqlite` with in-memory databases for fast migration round-trip tests (`upgrade head`, `downgrade base`, `upgrade head`). Run integration tests against local PostgreSQL matching the production version. Set `lock_timeout = '5s'` in PostgreSQL migration scripts to fail fast on lock contention.
