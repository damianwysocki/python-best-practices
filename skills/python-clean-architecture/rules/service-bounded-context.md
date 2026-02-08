---
title: Organize Code by Bounded Context
impact: CRITICAL
impactDescription: Domain alignment clarity
tags: service, architecture, bounded-context, domain-driven-design
---

## Organize Code by Bounded Context

Structure your project around domain bounded contexts (e.g., `users/`, `orders/`, `payments/`) rather than technical layers (e.g., `models/`, `services/`, `repositories/`). Each bounded context encapsulates its own models, services, repositories, and APIs, making the codebase navigable by business capability.

**Incorrect (organized by technical layer):**

```python
# project/models/user.py
class User:
    ...

# project/models/order.py
class Order:
    ...

# project/services/user_service.py
from project.models.user import User

class UserService:
    ...

# project/services/order_service.py
from project.models.order import Order
from project.models.user import User  # cross-cutting concern buried in layers

class OrderService:
    ...
```

**Correct (organized by bounded context):**

```python
# project/users/domain.py
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class User:
    user_id: str
    email: str
    name: str

# project/users/service.py
from project.users.domain import User
from project.users.repository import UserRepository

class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

# project/orders/domain.py
@dataclass(frozen=True, slots=True)
class Order:
    order_id: str
    buyer_id: str
    total: Decimal

# project/orders/service.py
from project.orders.domain import Order
from project.orders.repository import OrderRepository

class OrderService:
    def __init__(self, repo: OrderRepository) -> None:
        self._repo = repo
```

When every bounded context owns its full vertical slice, changes to one domain rarely ripple into another.

Reference: [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
