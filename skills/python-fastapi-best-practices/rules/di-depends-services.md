---
title: Inject Services via Depends
impact: CRITICAL
impactDescription: Untestable coupled code
tags: dependency-injection, depends, services, testing
---

## Inject Services via Depends

Inject services via `Depends()`; never instantiate services directly in endpoint bodies. Direct instantiation couples endpoints to concrete implementations, making testing difficult and preventing service reuse across endpoints.

**Incorrect (service instantiated in endpoint):**

```python
from app.services.user_service import UserService
from app.db import get_session

@router.get("/users/{user_id}")
async def get_user(user_id: int):
    session = get_session()
    service = UserService(session)  # tight coupling
    return await service.get_by_id(user_id)
```

**Correct (service injected via Depends):**

```python
from fastapi import Depends
from app.services.user_service import UserService

def get_user_service(
    session: AsyncSession = Depends(get_async_session),
) -> UserService:
    return UserService(session)

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    return await service.get_by_id(user_id)
```

This pattern allows overriding the dependency in tests without monkeypatching:

```python
app.dependency_overrides[get_user_service] = lambda: mock_service
```

Reference: https://fastapi.tiangolo.com/tutorial/dependencies/
