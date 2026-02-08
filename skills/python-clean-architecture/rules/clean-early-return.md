---
title: Use Guard Clauses and Early Returns
impact: MEDIUM
impactDescription: Reduces nesting depth
tags: clean-code, early-return, guard-clause, readability
---

## Use Guard Clauses and Early Returns

Use guard clauses to handle error conditions and edge cases at the top of a function, then return or raise early. This eliminates deep nesting and keeps the "happy path" at the lowest indentation level, improving readability.

**Incorrect (deeply nested conditionals):**

```python
def process_order(order: Order | None, user: User | None) -> OrderResult:
    if order is not None:
        if user is not None:
            if order.status == OrderStatus.PENDING:
                if user.is_active:
                    if order.total > Decimal("0"):
                        result = payment_service.charge(user, order.total)
                        if result.success:
                            return OrderResult(status="completed")
                        else:
                            return OrderResult(status="payment_failed")
                    else:
                        return OrderResult(status="invalid_total")
                else:
                    return OrderResult(status="inactive_user")
            else:
                return OrderResult(status="not_pending")
        else:
            return OrderResult(status="no_user")
    else:
        return OrderResult(status="no_order")
```

**Correct (guard clauses with early return):**

```python
def process_order(order: Order | None, user: User | None) -> OrderResult:
    if order is None:
        return OrderResult(status="no_order")
    if user is None:
        return OrderResult(status="no_user")
    if order.status != OrderStatus.PENDING:
        return OrderResult(status="not_pending")
    if not user.is_active:
        return OrderResult(status="inactive_user")
    if order.total <= Decimal("0"):
        return OrderResult(status="invalid_total")

    result = payment_service.charge(user, order.total)
    if not result.success:
        return OrderResult(status="payment_failed")

    return OrderResult(status="completed")
```

The guard clause version reads top-to-bottom with no nesting, making the logic immediately clear.
