# Database Patterns

SQLAlchemy 2.0 async database patterns covering session management, repository design, connection pooling, transactions, and migrations.

SQLAlchemy 2.0's async API provides non-blocking database access that aligns with FastAPI's async-first architecture. Session-per-request lifecycle through `Depends` gives each request an isolated transaction. The repository pattern separates query logic from business logic, making both independently testable.

## Async Engine and Session Factory

Create the async engine once during application startup using the lifespan context manager. Share the session factory across the application through dependency injection.

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine(
    settings.database_url,
    pool_size=20,
    max_overflow=10,
    pool_timeout=30,
    pool_recycle=3600,
    echo=settings.debug,
)

async_session_factory = async_sessionmaker(engine, expire_on_commit=False)
```

Set `expire_on_commit=False` to prevent lazy-load attempts on committed objects. Without this setting, accessing attributes on a committed model triggers a synchronous load that raises `MissingGreenlet` in async contexts.

### Engine Configuration

| Parameter | Production Value | Rationale |
|-----------|-----------------|-----------|
| `pool_size` | 10-50 | Match expected concurrent requests |
| `max_overflow` | 5-20 | Allow burst capacity |
| `pool_recycle` | 3600 | Prevent stale connections |
| `pool_pre_ping` | True | Detect dead connections before use |

Start with `pool_size` equal to the number of uvicorn workers and adjust based on monitoring.

## Session-Per-Request

Provide database sessions through a FastAPI dependency that creates one session per request. The session commits on success and rolls back on failure:

```python
from collections.abc import AsyncGenerator

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

This pattern ensures each request operates within a single transaction boundary. The session opens when the dependency resolves and closes after the response completes, preventing connection leaks.

## Repository Pattern

Repositories encapsulate database queries behind a typed interface. Each repository handles one aggregate root (one primary table with its related tables):

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

class UserRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def get(self, user_id: int) -> User | None:
        result = await self._session.execute(
            select(User).where(User.id == user_id)
        )
        return result.scalar_one_or_none()

    async def create(self, user: User) -> User:
        self._session.add(user)
        await self._session.flush()
        return user
```

### Repository Guidelines

| Guideline | Rationale |
|-----------|-----------|
| One repository per aggregate root | Keeps query logic focused on a single domain |
| Accept session through constructor | Enables testing with mock sessions |
| Use `flush()` not `commit()` | Let the session dependency handle commit/rollback |
| Keep queries in the repository | Services remain free of SQL |

## Model Definition

Define SQLAlchemy models using the 2.0 declarative style with `Mapped` and `mapped_column` for type checker integration:

```python
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    name: Mapped[str] = mapped_column(String(100))
    is_active: Mapped[bool] = mapped_column(default=True)
```

`Mapped[]` provides type checker annotations while `mapped_column()` configures database columns.

## Relationships and Eager Loading

Load relationships explicitly in queries to avoid the N+1 problem. Lazy loading (the default) fails in async contexts because it triggers synchronous queries that raise `MissingGreenlet`.

```python
async def get_user_with_orders(self, user_id: int) -> User | None:
    result = await self._session.execute(
        select(User).options(selectinload(User.orders)).where(User.id == user_id)
    )
    return result.scalar_one_or_none()
```

Use `selectinload` for one-to-many relationships and `joinedload` for many-to-one. Define relationships with `back_populates` for bidirectional navigation.

## Migrations with Alembic

Use Alembic for schema migrations. Configure `env.py` with the async migration runner for compatibility with the async engine. Generate migrations with `alembic revision --autogenerate -m "description"` and review every autogenerated migration before applying -- autogenerate misses index renames, data migrations, and constraint modifications.

| Practice | Rationale |
|----------|-----------|
| Review autogenerated migrations | Autogenerate is a starting point, not a final product |
| One logical change per migration | Simplifies rollback and debugging |
| Never edit applied migrations | Create new migrations to correct issues |
| Include both upgrade and downgrade | Enable rollback without manual intervention |

## Transaction Patterns

All repository operations within a single request share the same session and transaction through session-per-request. For explicit savepoints, use `session.begin_nested()` to create rollback points that preserve the outer transaction on failure.

## Evidence

SQLAlchemy 2.0 async patterns follow the library's official documentation. Session-per-request is recommended by both SQLAlchemy and FastAPI. Connection pool sizing comes from PostgreSQL operational best practices.

## Related

- [FastAPI Patterns](../../fastapi/reference/fastapi.md)
- [Type Safety](../../../../core/packages/typing/reference/type_safety.md)
- [Project Structure](../../../../core/packages/structure/reference/structure.md)
