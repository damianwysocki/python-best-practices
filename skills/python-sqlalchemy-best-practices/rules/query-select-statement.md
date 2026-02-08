---
title: Use select() Statement Style (2.0 API)
impact: CRITICAL
impactDescription: Future-proof query pattern
tags: query, select, 2.0-api, session-execute
---

## Use select() Statement Style (2.0 API)

The legacy `session.query(Model)` API is soft-deprecated in SQLAlchemy 2.0.
Use `select()` statements executed via `session.execute()` instead. The new
API is composable, supports subqueries naturally, works identically in sync
and async contexts, and produces better type inference.

**Incorrect (legacy session.query API):**

```python
def get_active_users(session: Session) -> list[User]:
    return (
        session.query(User)
        .filter(User.is_active == True)
        .order_by(User.name)
        .all()
    )

def get_user_by_email(session: Session, email: str) -> User | None:
    return session.query(User).filter_by(email=email).first()
```

**Correct (select() with session.execute):**

```python
from sqlalchemy import select

def get_active_users(session: Session) -> Sequence[User]:
    stmt = select(User).where(User.is_active == True).order_by(User.name)
    return session.execute(stmt).scalars().all()

def get_user_by_email(session: Session, email: str) -> User | None:
    stmt = select(User).where(User.email == email)
    return session.execute(stmt).scalars().first()
```

The `select()` API is also required for `AsyncSession`:

```python
async def get_active_users(session: AsyncSession) -> Sequence[User]:
    stmt = select(User).where(User.is_active == True)
    result = await session.execute(stmt)
    return result.scalars().all()
```

The statement can be built incrementally and passed around, making it easy
to compose complex queries from reusable fragments.

Reference: https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html
