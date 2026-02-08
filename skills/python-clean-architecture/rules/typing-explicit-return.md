---
title: Always Annotate Return Types
impact: CRITICAL
impactDescription: Catches silent bugs
tags: typing, return-type, annotation, function
---

## Always Annotate Return Types

Every function and method must have an explicit return type annotation. Without it, the type checker infers the return type, which can silently change when the implementation changes, propagating incorrect types throughout the codebase.

**Incorrect (missing return annotations):**

```python
class UserService:
    def __init__(self, repo: UserRepository):
        self._repo = repo

    def get_user(self, user_id: str):  # what does this return?
        return self._repo.find_by_id(user_id)

    def get_user_name(self, user_id: str):
        user = self.get_user(user_id)
        if user is None:
            return None  # caller might not handle None
        return user.name
```

**Correct (explicit return annotations):**

```python
class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo

    def get_user(self, user_id: str) -> User:
        return self._repo.find_by_id(user_id)

    def get_user_name(self, user_id: str) -> str | None:
        user = self.get_user(user_id)
        if user is None:
            return None
        return user.name
```

Explicit return types serve as documentation, catch refactoring errors early, and prevent type inference from silently widening or narrowing.
