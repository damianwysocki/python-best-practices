---
title: Use Async Engine and AsyncSession for Async Applications
impact: HIGH
impactDescription: Prevents event-loop blocking
tags: session, async, asyncio, engine, asyncsession
---

## Use Async Engine and AsyncSession for Async Applications

In async applications (FastAPI, Starlette, aiohttp), using synchronous
`create_engine` and `Session` blocks the event loop on every database call.
Use `create_async_engine` with `AsyncSession` to keep the event loop free and
maintain concurrency.

**Incorrect (synchronous engine in async application):**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

engine = create_engine("postgresql://localhost/mydb")

async def get_user(user_id: int) -> User | None:
    with Session(engine) as session:  # Blocks the event loop
        return session.get(User, user_id)
```

**Correct (async engine with AsyncSession):**

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.ext.asyncio import async_sessionmaker

engine = create_async_engine("postgresql+asyncpg://localhost/mydb")
AsyncSessionLocal = async_sessionmaker(engine, class_=AsyncSession)

async def get_user(user_id: int) -> User | None:
    async with AsyncSessionLocal() as session:
        return await session.get(User, user_id)
```

For FastAPI dependency injection:

```python
from collections.abc import AsyncGenerator

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

Note that `AsyncSession` requires an async-compatible driver such as `asyncpg`
for PostgreSQL or `aiosqlite` for SQLite. The connection URL scheme must
include the async driver (e.g., `postgresql+asyncpg://`).

Reference: https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html
