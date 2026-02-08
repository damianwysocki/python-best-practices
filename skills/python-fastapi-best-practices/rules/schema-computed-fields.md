---
title: Use Computed Fields for Derived Data
impact: MEDIUM
impactDescription: Redundant stored data
tags: pydantic, computed, fields, response
---

## Use Computed Fields for Derived Data

Use `@computed_field` for derived data in response models instead of storing computed values in the database or calculating them in the endpoint. Computed fields keep derivation logic co-located with the schema definition and are included in serialization and OpenAPI docs automatically.

**Incorrect (computing derived values in the endpoint):**

```python
@router.get("/orders/{order_id}")
async def get_order(order_id: int, service=Depends(get_order_service)):
    order = await service.get(order_id)
    return {
        **order.dict(),
        "total_price": order.quantity * order.unit_price,
        "display_name": f"Order #{order.id}",
    }
```

**Correct (computed fields on the response model):**

```python
from pydantic import BaseModel, computed_field

class OrderResponse(BaseModel):
    model_config = {"from_attributes": True}

    id: int
    quantity: int
    unit_price: float
    status: str

    @computed_field
    @property
    def total_price(self) -> float:
        return self.quantity * self.unit_price

    @computed_field
    @property
    def display_name(self) -> str:
        return f"Order #{self.id}"
```

The `total_price` and `display_name` fields appear in the JSON output and OpenAPI schema without being stored or computed outside the model.

Reference: https://docs.pydantic.dev/latest/concepts/fields/#computed-fields
