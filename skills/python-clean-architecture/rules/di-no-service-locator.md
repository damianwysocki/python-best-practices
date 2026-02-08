---
title: Never Use Service Locator Pattern
impact: HIGH
impactDescription: Hides dependency graph
tags: dependency-injection, service-locator, anti-pattern
---

## Never Use Service Locator Pattern

Avoid the service locator pattern where classes look up their dependencies from a global registry at runtime. This hides the dependency graph, makes testing harder, and couples code to the locator.

**Incorrect (service locator pattern):**

```python
# locator.py
class ServiceLocator:
    _services: dict[type, object] = {}

    @classmethod
    def register(cls, interface: type, impl: object) -> None:
        cls._services[interface] = impl

    @classmethod
    def get(cls, interface: type) -> object:
        return cls._services[interface]

# orders/service.py
from locator import ServiceLocator

class OrderService:
    def place_order(self, user_id: str) -> Order:
        repo = ServiceLocator.get(OrderRepository)  # hidden lookup
        notifier = ServiceLocator.get(Notifier)  # hidden lookup
        order = repo.create(user_id)
        notifier.notify(order)
        return order
```

**Correct (explicit constructor injection):**

```python
class OrderService:
    def __init__(
        self,
        repo: OrderRepository,
        notifier: Notifier,
    ) -> None:
        self._repo = repo
        self._notifier = notifier

    def place_order(self, user_id: str) -> Order:
        order = self._repo.create(user_id)
        self._notifier.notify(order)
        return order
```

Explicit injection keeps the dependency graph visible at the call site and in the constructor signature, making both testing and reasoning straightforward.
