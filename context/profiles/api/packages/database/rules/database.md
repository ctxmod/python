# Database

Use SQLAlchemy 2.0 async engine with session-per-request lifecycle.

Use SQLAlchemy 2.0 with `AsyncSession` for all database access. Provide sessions through a FastAPI dependency that creates one session per request and commits or rolls back on completion. Implement the repository pattern to isolate query logic from route handlers.

## Rationale

SQLAlchemy 2.0's async API avoids blocking the event loop during database operations, matching FastAPI's async-first design. Session-per-request via `Depends` ensures each request gets an isolated transaction that commits on success or rolls back on failure, preventing partial writes and connection leaks. The repository pattern keeps SQL queries in dedicated classes, making business logic testable with mock repositories rather than requiring a live database.

- [Database Integration](../reference/database_integration.md)
- [FastAPI Patterns](../../fastapi/reference/fastapi.md)

## Examples

**Correct**:
```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from fastapi import Depends

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        yield session

@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    session: AsyncSession = Depends(get_session),
) -> UserResponse:
    repo = UserRepository(session)
    return await repo.get(user_id)
```

**Incorrect**:
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

engine = create_engine(DATABASE_URL)  # Sync engine blocks event loop

@router.get("/users/{user_id}")
def get_user(user_id: int):
    with Session(engine) as session:  # Manual session, sync access
        return session.query(User).get(user_id)
```

## Exceptions

Read-only analytics queries on large datasets may use raw async connections with `connection.execute()` to bypass ORM overhead. Migration scripts (Alembic) use the sync engine since they run outside the request cycle.
