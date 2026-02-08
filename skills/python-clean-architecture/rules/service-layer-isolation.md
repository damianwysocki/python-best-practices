---
title: Isolate Service Communication Through Interfaces
impact: CRITICAL
impactDescription: Prevents coupling leaks
tags: service, isolation, interface, architecture
---

## Isolate Service Communication Through Interfaces

Services must communicate only through well-defined service interfaces (Protocol or ABC). A service should never directly import or call another bounded context's repository, database client, or internal implementation detail.

**Incorrect (service directly uses another context's repository):**

```python
# orders/service.py
from users.repository import UserRepository  # direct repo import

class OrderService:
    def __init__(self, order_repo: OrderRepository, user_repo: UserRepository) -> None:
        self._order_repo = order_repo
        self._user_repo = user_repo  # leaking internal detail

    def place_order(self, user_id: str, items: list[str]) -> Order:
        user = self._user_repo.get_by_id(user_id)  # reaching into users internals
        return self._order_repo.create(user.email, items)
```

**Correct (service communicates through a service interface):**

```python
# users/interface.py
from typing import Protocol

class UserServiceInterface(Protocol):
    def get_user_email(self, user_id: str) -> str: ...

# orders/service.py
from users.interface import UserServiceInterface

class OrderService:
    def __init__(
        self,
        order_repo: OrderRepository,
        user_service: UserServiceInterface,
    ) -> None:
        self._order_repo = order_repo
        self._user_service = user_service

    def place_order(self, user_id: str, items: list[str]) -> Order:
        email = self._user_service.get_user_email(user_id)
        return self._order_repo.create(email, items)
```

This ensures that internal implementation changes in one bounded context do not force changes in another.
