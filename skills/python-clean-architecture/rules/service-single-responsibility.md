---
title: Single Responsibility per Service
impact: HIGH
impactDescription: Reduces service bloat
tags: service, single-responsibility, bounded-context
---

## Single Responsibility per Service

Each service class should handle the business logic for exactly one bounded context or subdomain. Avoid "god services" that orchestrate multiple unrelated concerns. When a service grows beyond its context, extract a new bounded context.

**Incorrect (one service handling multiple domains):**

```python
# app/service.py
class AppService:
    def __init__(
        self,
        user_repo: UserRepository,
        order_repo: OrderRepository,
        payment_gateway: PaymentGateway,
        email_client: EmailClient,
    ) -> None:
        self._user_repo = user_repo
        self._order_repo = order_repo
        self._payment_gateway = payment_gateway
        self._email_client = email_client

    def register_user(self, email: str, name: str) -> User: ...
    def place_order(self, user_id: str, items: list[str]) -> Order: ...
    def process_payment(self, order_id: str) -> Payment: ...
    def send_receipt(self, payment_id: str) -> None: ...
```

**Correct (each service owns one context):**

```python
# users/service.py
class UserService:
    def __init__(self, user_repo: UserRepository) -> None:
        self._user_repo = user_repo

    def register(self, email: str, name: str) -> User: ...

# orders/service.py
class OrderService:
    def __init__(self, order_repo: OrderRepository) -> None:
        self._order_repo = order_repo

    def place_order(self, user_id: str, items: list[str]) -> Order: ...

# payments/service.py
class PaymentService:
    def __init__(self, gateway: PaymentGateway) -> None:
        self._gateway = gateway

    def process(self, order_id: str) -> Payment: ...
```

When orchestration across contexts is needed, create a thin application service or use an event-driven approach rather than merging domains.
