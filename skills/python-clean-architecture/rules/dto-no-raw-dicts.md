---
title: Never Pass Raw Dicts Between Layers
impact: CRITICAL
impactDescription: Eliminates key typos
tags: dto, dict, typed, dataclass, type-safety
---

## Never Pass Raw Dicts Between Layers

Never use plain `dict` objects to pass data between architectural layers (API, service, persistence). Raw dicts have no type safety, no autocompletion, and make typos in key names invisible until runtime.

**Incorrect (raw dicts flowing between layers):**

```python
# api/routes.py
def create_user_endpoint(request: Request) -> dict:
    data = request.json()
    result = user_service.create_user(data)  # dict in
    return {"id": result["id"], "name": result["nme"]}  # typo: "nme"

# service.py
def create_user(data: dict) -> dict:
    user = repo.save({"name": data["name"], "email": data["email"]})
    return {"id": user["id"], "name": user["name"]}  # no type safety
```

**Correct (typed DTOs):**

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class CreateUserRequest:
    name: str
    email: str

@dataclass(frozen=True, slots=True)
class UserResponse:
    user_id: str
    name: str
    email: str

# api/routes.py
def create_user_endpoint(request: Request) -> UserResponse:
    dto = CreateUserRequest(name=request.json()["name"], email=request.json()["email"])
    return user_service.create_user(dto)

# service.py
def create_user(self, request: CreateUserRequest) -> UserResponse:
    user = self._repo.save(request.name, request.email)
    return UserResponse(user_id=user.id, name=user.name, email=user.email)
```

Typed DTOs turn runtime KeyError and typo bugs into compile-time type errors.
