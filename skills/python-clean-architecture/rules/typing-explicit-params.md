---
title: Always Annotate All Function Parameters
impact: CRITICAL
impactDescription: Enables static analysis
tags: typing, parameters, annotation, function
---

## Always Annotate All Function Parameters

Every function parameter must have a type annotation. Unannotated parameters default to `Any` in most type checkers, which silently disables type checking for all code that touches that parameter.

**Incorrect (unannotated parameters):**

```python
def create_order(user_id, items, discount=0.0):
    total = sum(item.price for item in items)
    return Order(user_id=user_id, total=total * (1 - discount))

def send_notification(user, message, urgent=False):
    if urgent:
        user.send_sms(message)
    else:
        user.send_email(message)
```

**Correct (fully annotated parameters):**

```python
from decimal import Decimal

def create_order(
    user_id: str,
    items: list[OrderItem],
    discount: Decimal = Decimal("0.0"),
) -> Order:
    total = sum(item.price for item in items)
    return Order(user_id=user_id, total=total * (1 - discount))

def send_notification(
    user: User,
    message: str,
    urgent: bool = False,
) -> None:
    if urgent:
        user.send_sms(message)
    else:
        user.send_email(message)
```

Full parameter annotations let the type checker verify every call site, catching type mismatches before runtime.
