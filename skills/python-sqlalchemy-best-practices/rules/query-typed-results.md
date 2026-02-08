---
title: Type Query Results Properly
impact: HIGH
impactDescription: Enables static result validation
tags: query, typing, sequence, scalar-result, return-types
---

## Type Query Results Properly

Always annotate query return types with the correct container type. Use
`Sequence[Model]` for `.all()` results and `Model | None` for `.first()`
or `.one_or_none()`. This enables type checkers to catch errors when callers
assume a result is non-None or iterate over a scalar.

**Incorrect (untyped or incorrectly typed results):**

```python
def get_users(session: Session):  # No return type
    stmt = select(User)
    return session.execute(stmt).scalars().all()

def get_user(session: Session, user_id: int):  # No return type
    return session.get(User, user_id)

def get_emails(session: Session) -> list:  # Too broad
    stmt = select(User.email)
    return session.execute(stmt).scalars().all()
```

**Correct (properly typed query results):**

```python
from collections.abc import Sequence

def get_users(session: Session) -> Sequence[User]:
    stmt = select(User)
    return session.execute(stmt).scalars().all()

def get_user(session: Session, user_id: int) -> User | None:
    return session.get(User, user_id)

def get_user_strict(session: Session, user_id: int) -> User:
    stmt = select(User).where(User.id == user_id)
    result = session.execute(stmt).scalars().one()  # Raises if not found
    return result

def get_emails(session: Session) -> Sequence[str]:
    stmt = select(User.email)
    return session.execute(stmt).scalars().all()
```

Use `Sequence` from `collections.abc` instead of `list` because `.all()`
returns a `Sequence`, and `Sequence` is covariant, which is more compatible
with generic code. Use `.one()` when exactly one result is expected, and
`.one_or_none()` when zero or one is acceptable.

Reference: https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html#selecting-orm-entities
