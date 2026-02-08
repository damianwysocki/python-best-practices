---
title: Register Custom Exception Handlers
impact: HIGH
impactDescription: Inconsistent error formats
tags: router, exceptions, handlers, errors
---

## Register Custom Exception Handlers

Register custom exception handlers at the app level; never use try/except in endpoints to format error responses. Centralized handlers ensure every error follows the same response format, reduce boilerplate, and let endpoints focus on the happy path.

**Incorrect (try/except in every endpoint):**

```python
@router.get("/users/{user_id}")
async def get_user(user_id: int, service=Depends(get_user_service)):
    try:
        user = await service.get_by_id(user_id)
        return user
    except UserNotFoundError:
        return JSONResponse(status_code=404, content={"error": "Not found"})
    except PermissionError:
        return JSONResponse(status_code=403, content={"error": "Forbidden"})
```

**Correct (custom exception handlers registered on app):**

```python
# app/exceptions.py
class AppException(Exception):
    def __init__(self, status_code: int, detail: str):
        self.status_code = status_code
        self.detail = detail

class NotFoundError(AppException):
    def __init__(self, resource: str, resource_id: int):
        super().__init__(404, f"{resource} with id {resource_id} not found")

# app/main.py
@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.detail},
    )

# app/routes/users.py — clean, no try/except
@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    return await service.get_by_id(user_id)  # raises NotFoundError
```

Services raise domain exceptions; the handler translates them to HTTP responses.

Reference: https://fastapi.tiangolo.com/tutorial/handling-errors/#install-custom-exception-handlers
