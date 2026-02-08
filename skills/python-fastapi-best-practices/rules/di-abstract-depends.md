---
title: Depend on Abstractions for Testing
impact: HIGH
impactDescription: Hard-to-test dependencies
tags: dependency-injection, protocol, abc, testing
---

## Depend on Abstractions for Testing

Use `Depends()` with `Protocol` or `ABC` types so implementations can be swapped for testing. This inversion of control decouples endpoints from infrastructure details like databases, caches, or third-party APIs.

**Incorrect (endpoint depends on concrete implementation):**

```python
from app.repositories.user_repo import PostgresUserRepo

def get_repo():
    return PostgresUserRepo(connection_string=PROD_DB)

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    repo=Depends(get_repo),  # always hits Postgres
):
    return await repo.get(user_id)
```

**Correct (endpoint depends on Protocol, swap in tests):**

```python
from typing import Protocol

class UserRepository(Protocol):
    async def get(self, user_id: int) -> User | None: ...
    async def create(self, data: UserCreate) -> User: ...

def get_user_repo(
    db: AsyncSession = Depends(get_db),
) -> UserRepository:
    return PostgresUserRepo(db)

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    repo: UserRepository = Depends(get_user_repo),
) -> UserResponse:
    user = await repo.get(user_id)
    if not user:
        raise HTTPException(status_code=404)
    return user
```

In tests, override with an in-memory implementation:

```python
class FakeUserRepo:
    async def get(self, user_id: int) -> User | None:
        return User(id=user_id, email="test@test.com")

app.dependency_overrides[get_user_repo] = lambda: FakeUserRepo()
```

Reference: https://fastapi.tiangolo.com/advanced/testing-dependencies/
