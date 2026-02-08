---
title: Separate Schemas per Operation
impact: HIGH
impactDescription: Over-exposed data fields
tags: pydantic, schema, crud, separation
---

## Separate Schemas per Operation

Create separate `CreateSchema`, `UpdateSchema`, and `ResponseSchema` for each resource. A single shared model either exposes internal fields to clients or prevents required fields on creation. Separation ensures each operation accepts and returns exactly the right fields.

**Incorrect (single model for all operations):**

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    email: str
    password: str       # leaked in responses
    is_active: bool
    created_at: datetime  # client shouldn't set this
```

**Correct (separate schemas per operation):**

```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    email: EmailStr
    password: str
    full_name: str

class UserUpdate(BaseModel):
    full_name: str | None = None
    email: EmailStr | None = None

class UserResponse(BaseModel):
    id: int
    email: EmailStr
    full_name: str
    is_active: bool
    created_at: datetime

    model_config = {"from_attributes": True}
```

The `UserResponse` omits `password`. The `UserCreate` requires `password` but not `id`. The `UserUpdate` makes all fields optional for partial updates. Each schema documents its contract explicitly.

Reference: https://fastapi.tiangolo.com/tutorial/extra-models/
