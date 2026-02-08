---
title: Define Domain-Specific Exception Hierarchy
impact: MEDIUM
impactDescription: Enables precise error handling
tags: clean-code, exceptions, domain, error-handling
---

## Define Domain-Specific Exception Hierarchy

Define a custom exception hierarchy for each bounded context. Never use bare `except:` or catch `Exception` broadly. Domain exceptions make error handling precise, self-documenting, and allow callers to handle specific failure modes.

**Incorrect (generic exceptions and bare except):**

```python
class OrderService:
    def place_order(self, user_id: str, amount: Decimal) -> Order:
        try:
            user = self._user_service.get_user(user_id)
            if user.credit < amount:
                raise Exception("Not enough credit")  # generic
            return self._repo.save(Order(user_id=user_id, amount=amount))
        except:  # bare except catches SystemExit, KeyboardInterrupt
            print("Something went wrong")
            return None
```

**Correct (domain exception hierarchy):**

```python
# orders/exceptions.py
class OrderError(Exception):
    """Base exception for the orders bounded context."""

class InsufficientCreditError(OrderError):
    def __init__(self, user_id: str, required: Decimal, available: Decimal) -> None:
        self.user_id = user_id
        self.required = required
        self.available = available
        super().__init__(
            f"User {user_id} needs {required} but has {available}"
        )

class OrderNotFoundError(OrderError):
    def __init__(self, order_id: str) -> None:
        self.order_id = order_id
        super().__init__(f"Order {order_id} not found")

# orders/service.py
class OrderService:
    def place_order(self, user_id: str, amount: Decimal) -> Order:
        user = self._user_service.get_user(user_id)
        if user.credit < amount:
            raise InsufficientCreditError(user_id, amount, user.credit)
        return self._repo.save(Order(user_id=user_id, amount=amount))

# api/routes.py
def create_order_endpoint(request: Request) -> Response:
    try:
        order = order_service.place_order(request.user_id, request.amount)
        return Response(status=201, body=order)
    except InsufficientCreditError as exc:
        return Response(status=402, body={"error": str(exc)})
    except OrderError as exc:
        return Response(status=400, body={"error": str(exc)})
```

Domain exceptions carry context, enable precise error handling, and never swallow unexpected errors.
