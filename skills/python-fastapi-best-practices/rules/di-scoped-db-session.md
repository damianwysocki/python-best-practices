---
title: Scoped Database Session via Depends
impact: CRITICAL
impactDescription: Leaked database connections
tags: dependency-injection, database, session, lifecycle
---

## Scoped Database Session via Depends

Use a `Depends()` generator (with `yield`) for database session lifecycle management. The generator ensures the session is properly closed after each request, even if an exception occurs. Without this, connections leak and eventually exhaust the pool.

**Incorrect (manual session management in endpoint):**

```python
from app.db import SessionLocal

@router.get("/users")
async def list_users():
    session = SessionLocal()
    try:
        users = session.query(User).all()
        return users
    finally:
        session.close()  # duplicated everywhere
```

**Correct (generator dependency manages session lifecycle):**

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(DATABASE_URL, pool_size=20)
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

@router.get("/users")
async def list_users(
    db: AsyncSession = Depends(get_db),
) -> list[UserResponse]:
    result = await db.execute(select(User))
    return result.scalars().all()
```

The `async with` context manager and the `yield` pattern guarantee cleanup runs regardless of success or failure.

Reference: https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/
