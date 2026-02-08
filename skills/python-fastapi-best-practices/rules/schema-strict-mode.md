---
title: Enable Strict Mode with Field Validators
impact: HIGH
impactDescription: Silent type coercion
tags: pydantic, strict, validation, fields
---

## Enable Strict Mode with Field Validators

Enable strict mode on Pydantic models and use field validators for business rules. Without strict mode, Pydantic silently coerces types (e.g., `"123"` becomes `123`), masking client bugs. Field validators enforce domain constraints at the schema boundary.

**Incorrect (loose coercion, no business validation):**

```python
from pydantic import BaseModel

class OrderCreate(BaseModel):
    quantity: int       # silently accepts "5" as 5
    discount: float     # accepts any float, even negative
    email: str          # accepts any string
```

**Correct (strict mode with field validators):**

```python
from pydantic import BaseModel, EmailStr, field_validator

class OrderCreate(BaseModel):
    model_config = {"strict": True}

    quantity: int
    discount: float
    email: EmailStr

    @field_validator("quantity")
    @classmethod
    def quantity_must_be_positive(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("Quantity must be greater than zero")
        return v

    @field_validator("discount")
    @classmethod
    def discount_in_range(cls, v: float) -> float:
        if not (0.0 <= v <= 1.0):
            raise ValueError("Discount must be between 0 and 1")
        return v
```

With strict mode, sending `{"quantity": "5"}` now returns a 422 error instead of silently coercing. This catches client-side serialization bugs early.

Reference: https://docs.pydantic.dev/latest/concepts/strict_mode/
