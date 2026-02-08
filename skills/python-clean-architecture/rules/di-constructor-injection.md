---
title: Inject Dependencies via Constructor
impact: CRITICAL
impactDescription: Explicit dependency graph
tags: dependency-injection, constructor, init, testability
---

## Inject Dependencies via Constructor

Always inject dependencies through `__init__` parameters. Never instantiate collaborators inside methods or at class level. Constructor injection makes the dependency graph explicit, simplifies testing, and enables the composition root to control wiring.

**Incorrect (instantiating dependencies inside methods):**

```python
class OrderService:
    def place_order(self, user_id: str, items: list[str]) -> Order:
        repo = PostgresOrderRepository()  # hidden dependency
        notifier = SmtpNotifier()  # hidden dependency
        order = repo.create(user_id, items)
        notifier.notify(user_id, f"Order {order.id} placed")
        return order
```

**Correct (injecting via __init__):**

```python
class OrderService:
    def __init__(
        self,
        repo: OrderRepository,
        notifier: Notifier,
    ) -> None:
        self._repo = repo
        self._notifier = notifier

    def place_order(self, user_id: str, items: list[str]) -> Order:
        order = self._repo.create(user_id, items)
        self._notifier.notify(user_id, f"Order {order.id} placed")
        return order
```

Constructor injection ensures every dependency is visible in the signature, making it trivial to supply test doubles and reason about what a class needs.
