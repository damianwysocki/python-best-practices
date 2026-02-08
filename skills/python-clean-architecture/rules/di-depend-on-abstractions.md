---
title: Depend on Abstractions, Not Concretions
impact: CRITICAL
impactDescription: Decouples implementation details
tags: dependency-injection, protocol, abc, abstraction, solid
---

## Depend on Abstractions, Not Concretions

All injected dependencies should be typed as Protocol or ABC, never as concrete classes. This follows the Dependency Inversion Principle and allows swapping implementations without modifying consumers.

**Incorrect (depending on concrete implementation):**

```python
from infrastructure.postgres_repo import PostgresUserRepository

class UserService:
    def __init__(self, repo: PostgresUserRepository) -> None:
        self._repo = repo  # tied to Postgres forever

    def get_user(self, user_id: str) -> User:
        return self._repo.find_by_id(user_id)
```

**Correct (depending on abstraction):**

```python
from typing import Protocol

class UserRepository(Protocol):
    def find_by_id(self, user_id: str) -> User: ...
    def save(self, user: User) -> None: ...

class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo  # accepts any conforming implementation

    def get_user(self, user_id: str) -> User:
        return self._repo.find_by_id(user_id)
```

Now `PostgresUserRepository`, `InMemoryUserRepository`, or any future implementation can be plugged in without touching `UserService`.
