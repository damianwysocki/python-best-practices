---
title: Validate Data at System Boundaries
impact: MEDIUM
impactDescription: Trusts internal data flow
tags: dto, validation, boundary, input, pydantic
---

## Validate Data at System Boundaries

Validate and sanitize data at the system boundaries (API endpoints, message consumers, file readers) where external input enters your system. Once data passes boundary validation and is converted to internal DTOs, trust those DTOs within the domain layer without re-validating.

**Incorrect (validation scattered everywhere):**

```python
class OrderService:
    def create_order(self, user_id: str, amount: float) -> Order:
        if not user_id:  # redundant if boundary already validated
            raise ValueError("user_id required")
        if amount <= 0:  # redundant validation
            raise ValueError("amount must be positive")
        if not isinstance(amount, (int, float)):  # redundant type check
            raise TypeError("amount must be numeric")
        return self._repo.save(Order(user_id=user_id, amount=amount))
```

**Correct (validate at boundary, trust internally):**

```python
# api/routes.py  (system boundary)
from pydantic import BaseModel, Field

class CreateOrderRequest(BaseModel):
    user_id: str = Field(min_length=1)
    amount: Decimal = Field(gt=0)

def create_order_endpoint(request: Request) -> OrderResponse:
    validated = CreateOrderRequest.model_validate(request.json())
    dto = CreateOrderInput(user_id=validated.user_id, amount=validated.amount)
    return order_service.create_order(dto)

# orders/service.py  (trusts validated DTO)
@dataclass(frozen=True, slots=True)
class CreateOrderInput:
    user_id: str
    amount: Decimal

class OrderService:
    def create_order(self, input: CreateOrderInput) -> Order:
        # No re-validation needed -- boundary already guaranteed correctness
        return self._repo.save(Order(user_id=input.user_id, amount=input.amount))
```

Boundary validation eliminates defensive checks throughout the domain, keeping business logic clean and focused.
