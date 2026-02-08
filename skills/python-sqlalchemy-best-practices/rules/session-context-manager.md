---
title: Always Use Context Manager for Session Lifecycle
impact: CRITICAL
impactDescription: Prevents session leaks
tags: session, context-manager, lifecycle, resource-management
---

## Always Use Context Manager for Session Lifecycle

SQLAlchemy sessions hold database connections and transactional state. Failing to
properly close sessions leads to connection pool exhaustion, leaked transactions,
and data corruption. Always use `with Session(engine) as session:` or a
`sessionmaker` combined with a context manager to guarantee cleanup on both
success and failure paths.

**Incorrect (manual session without cleanup):**

```python
from sqlalchemy.orm import Session

def create_user(engine, name: str) -> User:
    session = Session(engine)
    user = User(name=name)
    session.add(user)
    session.commit()  # If this raises, session is never closed
    return user
```

**Correct (context manager guarantees cleanup):**

```python
from sqlalchemy.orm import Session

def create_user(engine, name: str) -> User:
    with Session(engine) as session:
        user = User(name=name)
        session.add(user)
        session.commit()
        return user
```

For web frameworks, use a dependency that yields the session:

```python
from collections.abc import Generator
from sqlalchemy.orm import Session, sessionmaker

SessionLocal = sessionmaker(bind=engine)

def get_session() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        yield session
```

The context manager calls `session.close()` automatically when the block exits,
whether by normal return or exception. This returns the underlying connection
back to the pool and rolls back any uncommitted transaction.

Reference: https://docs.sqlalchemy.org/en/20/orm/session_basics.html#opening-and-closing-a-session
