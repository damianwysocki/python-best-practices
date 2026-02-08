---
title: Extract Reusable Field Patterns
impact: MEDIUM
impactDescription: Duplicated field definitions
tags: pydantic, reuse, base, schema
---

## Extract Reusable Field Patterns

Extract common field patterns into base schemas to eliminate duplication. When multiple models share the same fields (timestamps, pagination, audit trails), define them once in a base class. This ensures consistent naming, types, and validation across the entire API.

**Incorrect (duplicated fields across models):**

```python
class UserResponse(BaseModel):
    id: int
    created_at: datetime
    updated_at: datetime
    email: str

class OrderResponse(BaseModel):
    id: int
    created_at: datetime
    updated_at: datetime  # duplicated fields
    total: float
```

**Correct (shared base schemas):**

```python
from pydantic import BaseModel, Field

class TimestampMixin(BaseModel):
    created_at: datetime
    updated_at: datetime

class IdentifiedModel(TimestampMixin):
    id: int

    model_config = {"from_attributes": True}

class UserResponse(IdentifiedModel):
    email: str
    full_name: str

class OrderResponse(IdentifiedModel):
    total: float
    status: str

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int = Field(ge=1)
    page_size: int = Field(ge=1, le=100)
    has_next: bool
```

The `PaginatedResponse` generic lets you wrap any response type:

```python
@router.get("/users", response_model=PaginatedResponse[UserResponse])
async def list_users(page: int = 1, page_size: int = 20):
    ...
```

Reference: https://docs.pydantic.dev/latest/concepts/models/#model-inheritance
