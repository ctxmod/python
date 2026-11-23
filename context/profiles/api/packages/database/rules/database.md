# Database

Use the SQLAlchemy 2.x async engine with session-per-request lifecycle.

Use SQLAlchemy 2.x with `AsyncSession` for all database access, installed as `sqlalchemy[asyncio]` (from 2.1, greenlet ships only with that extra). Provide sessions through a FastAPI dependency that creates one session per request and commits or rolls back on completion, declared with `scope="function"` so the commit finishes before the response is sent. Load relationships explicitly; never rely on lazy loading under asyncio. Implement the repository pattern to isolate query logic from route handlers.

## Rationale

SQLAlchemy 2.x's async API avoids blocking the event loop during database operations, matching FastAPI's async-first design. Session-per-request via `Depends` ensures each request gets an isolated transaction that commits on success or rolls back on failure, preventing partial writes and connection leaks; under FastAPI's default dependency scope that commit would run after the status code is already sent, so a failed commit could not reach the client. The repository pattern keeps SQL queries in dedicated classes, making business logic testable with mock repositories rather than requiring a live database.

- [Database Integration](../reference/database_integration.md)
- [FastAPI Patterns](../../fastapi/reference/fastapi.md)

## Examples

**Correct**:
```python
from collections.abc import AsyncIterator
from typing import Annotated

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_session() -> AsyncIterator[AsyncSession]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

SessionDep = Annotated[AsyncSession, Depends(get_session, scope="function")]

@router.get("/users/{user_id}")
async def get_user(user_id: int, session: SessionDep) -> UserResponse:
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

Read-only analytics queries on large datasets may use raw async connections with `connection.execute()` to bypass ORM overhead. Migration scripts (Alembic) run outside the request cycle and do not use the request session; with an async driver they run through the `async` template's `run_sync` bridge (data profile, migrations package).
