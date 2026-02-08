---
title: Always Specify Response Models
impact: CRITICAL
impactDescription: Unvalidated API responses
tags: openapi, response, models, validation
---

## Always Specify Response Models

Always specify `response_model` or a return type annotation with explicit `status_code` on every endpoint. This ensures response validation, filters out unintended fields, and generates accurate OpenAPI schemas.

**Incorrect (no response model, implicit 200):**

```python
@router.post("/users")
async def create_user(payload: UserCreate):
    user = await user_service.create(payload)
    return user  # ORM model leaks internal fields
```

**Correct (explicit response model and status code):**

```python
from fastapi import status

@router.post(
    "/users",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create a new user",
)
async def create_user(payload: UserCreate) -> UserResponse:
    user = await user_service.create(payload)
    return UserResponse.model_validate(user)
```

Use `response_model_exclude_none=True` when you want to omit null fields from responses to keep payloads clean.

```python
@router.get(
    "/users/{user_id}",
    response_model=UserResponse,
    response_model_exclude_none=True,
)
async def get_user(user_id: int) -> UserResponse:
    ...
```

Reference: https://fastapi.tiangolo.com/tutorial/response-model/
