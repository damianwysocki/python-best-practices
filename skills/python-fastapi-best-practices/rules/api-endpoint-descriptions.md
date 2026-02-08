---
title: Every Endpoint Must Have Descriptions
impact: CRITICAL
impactDescription: Missing API documentation
tags: openapi, documentation, endpoints, swagger
---

## Every Endpoint Must Have Descriptions

Every FastAPI endpoint must include `summary`, `description`, and `response_description` parameters. Without these, auto-generated OpenAPI docs become unusable for consumers. The summary appears in the endpoint list, the description provides detail, and response_description documents the successful return value.

**Incorrect (missing documentation parameters):**

```python
@router.get("/users/{user_id}")
async def get_user(user_id: int):
    user = await user_service.get_by_id(user_id)
    return user
```

**Correct (fully documented endpoint):**

```python
@router.get(
    "/users/{user_id}",
    summary="Get user by ID",
    description="Retrieve a single user by their unique identifier. "
    "Returns 404 if the user does not exist.",
    response_description="The requested user object",
)
async def get_user(user_id: int) -> UserResponse:
    user = await user_service.get_by_id(user_id)
    return user
```

For endpoints with complex behavior, use multi-line docstrings as an alternative to the `description` parameter. FastAPI will use the docstring if no explicit description is provided.

```python
@router.post("/users", summary="Create a new user")
async def create_user(payload: UserCreate) -> UserResponse:
    """
    Create a new user with the provided details.

    - **email**: must be unique across the system
    - **password**: minimum 8 characters, hashed before storage
    """
    return await user_service.create(payload)
```

Reference: https://fastapi.tiangolo.com/tutorial/path-operation-configuration/
