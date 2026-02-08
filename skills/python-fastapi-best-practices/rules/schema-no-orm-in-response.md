---
title: Never Return ORM Models Directly
impact: HIGH
impactDescription: Leaked internal fields
tags: pydantic, orm, response, serialization
---

## Never Return ORM Models Directly

Never return ORM models directly from endpoints. Always map ORM objects to a Pydantic response schema. Returning ORM models leaks database columns (passwords, internal flags), breaks when lazy-loaded relationships trigger outside a session, and couples your API contract to your database schema.

**Incorrect (returning ORM model directly):**

```python
from app.models import User  # SQLAlchemy model

@router.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)
    return user  # exposes password_hash, internal flags, etc.
```

**Correct (map to response schema):**

```python
from app.schemas.user import UserResponse

@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return UserResponse.model_validate(user)
```

Use `model_config = {"from_attributes": True}` on the response schema so `model_validate()` reads from ORM attribute access rather than dict keys.

```python
class UserResponse(BaseModel):
    model_config = {"from_attributes": True}

    id: int
    email: str
    full_name: str
    created_at: datetime
```

Reference: https://docs.pydantic.dev/latest/concepts/models/#model-methods-and-properties
