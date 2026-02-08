---
title: Use NewType for Domain Identifiers
impact: MEDIUM
impactDescription: Prevents ID mix-ups
tags: typing, newtype, identifiers, domain, type-safety
---

## Use NewType for Domain Identifiers

Use `NewType` to create distinct types for domain identifiers (user IDs, order IDs, etc.). This prevents accidentally passing a user ID where an order ID is expected, even though both are strings at runtime.

**Incorrect (raw strings for all IDs):**

```python
def get_order(order_id: str) -> Order: ...
def get_user(user_id: str) -> User: ...

# Bug: accidentally swapped arguments -- no type error
order = get_order(user_id)  # passes type check, fails at runtime
user = get_user(order_id)   # passes type check, fails at runtime
```

**Correct (NewType for distinct IDs):**

```python
from typing import NewType

UserId = NewType("UserId", str)
OrderId = NewType("OrderId", str)

def get_order(order_id: OrderId) -> Order: ...
def get_user(user_id: UserId) -> User: ...

user_id = UserId("usr_123")
order_id = OrderId("ord_456")

order = get_order(user_id)   # type error: expected OrderId, got UserId
user = get_user(order_id)    # type error: expected UserId, got OrderId
order = get_order(order_id)  # correct
```

`NewType` has zero runtime overhead -- it is an identity function at runtime but provides full compile-time type distinction.
