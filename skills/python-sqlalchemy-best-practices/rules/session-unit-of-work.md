---
title: One Session Equals One Unit of Work
impact: HIGH
impactDescription: Ensures transactional integrity
tags: session, unit-of-work, transaction, commit
---

## One Session Equals One Unit of Work

A session should represent a single logical operation. Commit once at the end
of the operation. Multiple commits within one session break atomicity -- if the
second commit fails, the first is already persisted, leaving data in an
inconsistent state.

**Incorrect (multiple commits in one session):**

```python
def transfer_funds(session: Session, from_id: int, to_id: int, amount: Decimal) -> None:
    sender = session.get(Account, from_id)
    sender.balance -= amount
    session.commit()  # Persisted even if next step fails

    receiver = session.get(Account, to_id)
    receiver.balance += amount
    session.commit()  # If this fails, money vanished
```

**Correct (single commit at end of unit of work):**

```python
def transfer_funds(session: Session, from_id: int, to_id: int, amount: Decimal) -> None:
    sender = session.get(Account, from_id)
    receiver = session.get(Account, to_id)

    if sender is None or receiver is None:
        raise ValueError("Account not found")

    sender.balance -= amount
    receiver.balance += amount

    session.commit()  # Atomic: both changes persist or neither does
```

If you need to perform multiple independent operations, use separate sessions
for each. The session's identity map tracks all loaded objects, so a long-lived
session with many commits accumulates stale state.

For read-only operations that do not need transactions, you can still use a
single session without committing -- the session will roll back on close.

Reference: https://docs.sqlalchemy.org/en/20/orm/session_basics.html#framing-out-a-begin-commit-rollback-block
