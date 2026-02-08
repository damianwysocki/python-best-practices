---
title: Prevent Circular Imports
impact: HIGH
impactDescription: Avoids import failures
tags: structure, circular-imports, layers, dependencies
---

## Prevent Circular Imports

Prevent circular imports by enforcing a strict dependency direction: API depends on Service, Service depends on Domain and Repository interfaces, Domain depends on nothing. If two modules need each other, extract the shared abstraction into a separate module.

**Incorrect (circular dependency):**

```python
# users/service.py
from users.repository import UserRepository  # service -> repo

class UserService:
    def get_user(self, user_id: str) -> User: ...

# users/repository.py
from users.service import UserService  # repo -> service (circular!)

class UserRepository:
    def __init__(self, service: UserService) -> None:
        self._service = service  # wrong: repos should not depend on services
```

**Correct (unidirectional dependency flow):**

```python
# users/domain.py  (depends on nothing)
@dataclass(frozen=True, slots=True)
class User:
    user_id: str
    name: str
    email: str

# users/interfaces.py  (depends on domain only)
from typing import Protocol
from users.domain import User

class UserRepository(Protocol):
    def find_by_id(self, user_id: str) -> User | None: ...
    def save(self, user: User) -> None: ...

# users/service.py  (depends on domain + interfaces)
from users.domain import User
from users.interfaces import UserRepository

class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

# users/postgres_repo.py  (depends on domain + interfaces)
from users.domain import User
from users.interfaces import UserRepository

class PostgresUserRepository:
    def find_by_id(self, user_id: str) -> User | None: ...
    def save(self, user: User) -> None: ...
```

The dependency graph flows in one direction: `api -> service -> interfaces <- implementations`.
