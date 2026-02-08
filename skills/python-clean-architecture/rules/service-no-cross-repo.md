---
title: No Cross-Context Repository Access
impact: CRITICAL
impactDescription: Enforces domain boundaries
tags: service, repository, bounded-context, isolation
---

## No Cross-Context Repository Access

A service must never import or access another bounded context's repository. Repositories are internal implementation details of their owning context. Cross-context data needs must flow through service interfaces.

**Incorrect (order service imports user repository):**

```python
# orders/service.py
from orders.repository import OrderRepository
from users.repository import UserRepository  # violation: cross-context repo access

class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository,
        user_repo: UserRepository,
    ) -> None:
        self._order_repo = order_repo
        self._user_repo = user_repo

    def create_order(self, user_id: str, amount: Decimal) -> Order:
        user = self._user_repo.find_by_id(user_id)
        if user.credit < amount:
            raise InsufficientCredit()
        return self._order_repo.save(Order(user_id=user_id, amount=amount))
```

**Correct (order service uses user service interface):**

```python
# users/interface.py
from typing import Protocol
from decimal import Decimal

class UserCreditChecker(Protocol):
    def has_sufficient_credit(self, user_id: str, amount: Decimal) -> bool: ...

# orders/service.py
from orders.repository import OrderRepository
from users.interface import UserCreditChecker

class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository,
        credit_checker: UserCreditChecker,
    ) -> None:
        self._order_repo = order_repo
        self._credit_checker = credit_checker

    def create_order(self, user_id: str, amount: Decimal) -> Order:
        if not self._credit_checker.has_sufficient_credit(user_id, amount):
            raise InsufficientCredit()
        return self._order_repo.save(Order(user_id=user_id, amount=amount))
```

This preserves the encapsulation of each bounded context's persistence layer.
