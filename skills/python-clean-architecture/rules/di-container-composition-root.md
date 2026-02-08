---
title: Wire Dependencies at the Composition Root
impact: HIGH
impactDescription: Centralizes object creation
tags: dependency-injection, composition-root, wiring, container
---

## Wire Dependencies at the Composition Root

All dependency wiring should happen in a single composition root (typically `main.py`, `container.py`, or a factory function). Business logic modules should never create their own dependencies or reference the DI container.

**Incorrect (wiring scattered across modules):**

```python
# orders/service.py
from infrastructure.postgres_repo import PostgresOrderRepository
from infrastructure.smtp_notifier import SmtpNotifier

class OrderService:
    def __init__(self) -> None:
        self._repo = PostgresOrderRepository(dsn="postgres://...")
        self._notifier = SmtpNotifier(host="smtp.example.com")
```

**Correct (wiring at composition root):**

```python
# orders/service.py
class OrderService:
    def __init__(self, repo: OrderRepository, notifier: Notifier) -> None:
        self._repo = repo
        self._notifier = notifier

# container.py  (composition root)
from orders.service import OrderService
from infrastructure.postgres_repo import PostgresOrderRepository
from infrastructure.smtp_notifier import SmtpNotifier

def create_order_service(settings: Settings) -> OrderService:
    repo = PostgresOrderRepository(dsn=settings.database_url)
    notifier = SmtpNotifier(host=settings.smtp_host)
    return OrderService(repo=repo, notifier=notifier)

# main.py
def main() -> None:
    settings = Settings.from_env()
    order_service = create_order_service(settings)
    app = create_app(order_service=order_service)
    app.run()
```

Centralizing wiring makes it easy to see the full dependency graph and swap implementations for different environments.
