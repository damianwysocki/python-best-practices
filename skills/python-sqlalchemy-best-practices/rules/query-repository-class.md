---
title: Encapsulate Queries in Typed Repository Classes
impact: HIGH
impactDescription: Improves testability and reuse
tags: query, repository, protocol, pattern, abstraction
---

## Encapsulate Queries in Typed Repository Classes

Scattering raw `select()` calls across service code makes queries hard to find,
test, and optimize. Encapsulate data access in Repository classes with a
`Protocol` interface so services depend on an abstraction, not SQLAlchemy
directly.

**Incorrect (queries scattered in business logic):**

```python
class OrderService:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get_pending_orders(self) -> Sequence[Order]:
        stmt = select(Order).where(Order.status == "pending")
        return self._session.execute(stmt).scalars().all()

    def cancel_expired(self) -> int:
        stmt = select(Order).where(
            Order.status == "pending",
            Order.created_at < datetime.now() - timedelta(days=7),
        )
        orders = self._session.execute(stmt).scalars().all()
        for order in orders:
            order.status = "cancelled"
        return len(orders)
```

**Correct (repository with Protocol interface):**

```python
from typing import Protocol

class OrderRepository(Protocol):
    def get_pending(self) -> Sequence[Order]: ...
    def get_expired(self, cutoff: datetime) -> Sequence[Order]: ...

class SqlAlchemyOrderRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get_pending(self) -> Sequence[Order]:
        stmt = select(Order).where(Order.status == "pending")
        return self._session.execute(stmt).scalars().all()

    def get_expired(self, cutoff: datetime) -> Sequence[Order]:
        stmt = select(Order).where(
            Order.status == "pending",
            Order.created_at < cutoff,
        )
        return self._session.execute(stmt).scalars().all()

class OrderService:
    def __init__(self, repo: OrderRepository) -> None:
        self._repo = repo

    def cancel_expired(self) -> int:
        cutoff = datetime.now() - timedelta(days=7)
        orders = self._repo.get_expired(cutoff)
        for order in orders:
            order.status = "cancelled"
        return len(orders)
```

The `Protocol` interface allows injecting a fake repository in tests without
touching the database at all.

Reference: https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html
