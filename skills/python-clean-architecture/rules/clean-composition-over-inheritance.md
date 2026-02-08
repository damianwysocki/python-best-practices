---
title: Prefer Composition Over Inheritance
impact: MEDIUM
impactDescription: Reduces fragile hierarchies
tags: clean-code, composition, inheritance, delegation, design
---

## Prefer Composition Over Inheritance

Favor composition and delegation over deep inheritance hierarchies. Inheritance creates tight coupling between parent and child classes, making changes to the base class risky. Composition allows mixing behaviors flexibly and testing components in isolation.

**Incorrect (deep inheritance hierarchy):**

```python
class BaseRepository:
    def connect(self) -> None: ...
    def disconnect(self) -> None: ...

class CachedRepository(BaseRepository):
    def get_from_cache(self, key: str) -> object: ...

class LoggedCachedRepository(CachedRepository):
    def log(self, message: str) -> None: ...

class UserRepository(LoggedCachedRepository):
    def find_user(self, user_id: str) -> User:
        self.log(f"Finding user {user_id}")
        cached = self.get_from_cache(user_id)
        if cached:
            return cached
        self.connect()
        # ... fragile chain of inherited behavior
```

**Correct (composition with delegation):**

```python
from typing import Protocol

class Cache(Protocol):
    def get(self, key: str) -> object | None: ...
    def set(self, key: str, value: object) -> None: ...

class Logger(Protocol):
    def info(self, message: str) -> None: ...

class UserRepository:
    def __init__(
        self,
        db: DatabaseConnection,
        cache: Cache,
        logger: Logger,
    ) -> None:
        self._db = db
        self._cache = cache
        self._logger = logger

    def find_user(self, user_id: str) -> User:
        self._logger.info(f"Finding user {user_id}")
        cached = self._cache.get(user_id)
        if cached is not None:
            return cached
        user = self._db.query(User, user_id)
        self._cache.set(user_id, user)
        return user
```

Composition keeps each concern (caching, logging, persistence) independently testable and replaceable.
