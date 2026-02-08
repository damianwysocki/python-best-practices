---
title: Document All Error Responses
impact: HIGH
impactDescription: Hidden failure modes
tags: openapi, errors, documentation, responses
---

## Document All Error Responses

Document all error responses using the `responses` parameter on each endpoint. Undocumented error codes leave API consumers guessing about failure modes. Include the status code, description, and response model for every error the endpoint can produce.

**Incorrect (errors not documented in OpenAPI spec):**

```python
@router.get("/users/{user_id}")
async def get_user(user_id: int) -> UserResponse:
    user = await user_service.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

**Correct (error responses documented with responses parameter):**

```python
@router.get(
    "/users/{user_id}",
    response_model=UserResponse,
    responses={
        404: {
            "description": "User not found",
            "content": {
                "application/json": {
                    "example": {"detail": "User not found"}
                }
            },
        },
        422: {"description": "Invalid user ID format"},
    },
    summary="Get user by ID",
)
async def get_user(user_id: int) -> UserResponse:
    user = await user_service.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

Define a shared error model to reuse across endpoints for consistency.

```python
class ErrorResponse(BaseModel):
    detail: str

common_error_responses = {
    401: {"model": ErrorResponse, "description": "Not authenticated"},
    403: {"model": ErrorResponse, "description": "Forbidden"},
}
```

Reference: https://fastapi.tiangolo.com/advanced/additional-responses/
