---
title: Never Use Global or Module-Level Sessions
impact: CRITICAL
impactDescription: Prevents thread-safety bugs
tags: session, global, dependency-injection, concurrency
---

## Never Use Global or Module-Level Sessions

A global or module-level session is shared across threads and requests, leading
to race conditions, stale data, and transaction cross-contamination. Sessions are
not thread-safe. Always create sessions per-request or per-unit-of-work and
inject them via dependency injection.

**Incorrect (global module-level session):**

```python
from sqlalchemy.orm import Session

engine = create_engine("postgresql://localhost/mydb")
db = Session(engine)  # Global session shared everywhere

class UserService:
    def get_user(self, user_id: int) -> User | None:
        return db.get(User, user_id)  # Thread-unsafe

    def create_user(self, name: str) -> User:
        user = User(name=name)
        db.add(user)
        db.commit()  # Commits other threads' pending work too
        return user
```

**Correct (inject session via dependency injection):**

```python
from sqlalchemy.orm import Session

class UserService:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get_user(self, user_id: int) -> User | None:
        return self._session.get(User, user_id)

    def create_user(self, name: str) -> User:
        user = User(name=name)
        self._session.add(user)
        self._session.commit()
        return user

# Usage: create session per request/unit-of-work
with Session(engine) as session:
    service = UserService(session)
    user = service.create_user("Alice")
```

This pattern makes testing trivial because you can inject a test session backed
by an in-memory SQLite database or a rolled-back transaction.

Reference: https://docs.sqlalchemy.org/en/20/orm/session_basics.html#is-the-session-thread-safe
