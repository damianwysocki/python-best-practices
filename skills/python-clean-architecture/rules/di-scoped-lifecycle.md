---
title: Match Dependency Lifecycle to Scope
impact: MEDIUM
impactDescription: Prevents resource leaks
tags: dependency-injection, lifecycle, scope, singleton, transient
---

## Match Dependency Lifecycle to Scope

Ensure each dependency's lifecycle matches its intended scope. Singletons (e.g., connection pools) live for the application lifetime. Request-scoped dependencies (e.g., database sessions) are created and destroyed per request. Transient dependencies are created fresh each time.

**Incorrect (singleton session causes request leakage):**

```python
# container.py
db_session = SessionLocal()  # one session for entire app lifetime

def create_user_service() -> UserService:
    return UserService(session=db_session)  # all requests share one session

# This causes data leaks between requests and connection exhaustion
```

**Correct (request-scoped session):**

```python
# container.py
from contextlib import contextmanager
from collections.abc import Generator

engine = create_engine(settings.database_url)  # singleton: app lifetime
SessionLocal = sessionmaker(bind=engine)  # singleton: factory

@contextmanager
def request_session() -> Generator[Session, None, None]:
    session = SessionLocal()  # transient: per-request
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

def create_user_service(session: Session) -> UserService:
    repo = SqlAlchemyUserRepository(session=session)  # request-scoped
    return UserService(repo=repo)
```

Mismatched lifecycles lead to subtle bugs: shared sessions leak state between requests, and transient connection pools waste resources.
