---
title: Disable Autoflush and Flush Explicitly
impact: MEDIUM
impactDescription: Prevents surprising query results
tags: query, autoflush, flush, session-config
---

## Disable Autoflush and Flush Explicitly

By default, SQLAlchemy autoflushes pending changes before every query. This can
cause unexpected `IntegrityError` exceptions during reads and makes debugging
difficult because the flush happens implicitly. Disable autoflush and call
`session.flush()` explicitly when needed.

**Incorrect (relying on autoflush, surprising errors on read):**

```python
SessionLocal = sessionmaker(bind=engine)  # autoflush=True by default

def add_and_check(session: Session) -> bool:
    user = User(name="Alice", email="invalid")  # Bad data
    session.add(user)

    # This SELECT triggers autoflush, raising IntegrityError unexpectedly
    stmt = select(User).where(User.name == "Bob")
    bob = session.execute(stmt).scalars().first()
    return bob is not None
```

**Correct (autoflush disabled, explicit flush):**

```python
SessionLocal = sessionmaker(bind=engine, autoflush=False)

def add_and_check(session: Session) -> bool:
    user = User(name="Alice", email="alice@example.com")
    session.add(user)

    # No autoflush: this SELECT runs cleanly
    stmt = select(User).where(User.name == "Bob")
    bob = session.execute(stmt).scalars().first()

    # Flush explicitly when you want pending changes sent to DB
    session.flush()
    return bob is not None
```

When autoflush is disabled, use `session.flush()` to send pending changes to
the database before any read that depends on those writes. This gives you
precise control over when SQL is emitted.

Note: `flush()` does not commit. It only writes changes to the database within
the current transaction. Call `session.commit()` to make them permanent.

Reference: https://docs.sqlalchemy.org/en/20/orm/session_api.html#sqlalchemy.orm.Session.params.autoflush
