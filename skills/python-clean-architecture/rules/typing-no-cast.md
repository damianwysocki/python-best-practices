---
title: Never Use cast()
impact: CRITICAL
impactDescription: Bypasses type verification
tags: typing, cast, isinstance, narrowing, type-safety
---

## Never Use cast()

Never use `typing.cast()`. It tells the type checker to trust you without any runtime verification, which masks bugs. Instead, use `isinstance` type narrowing, `@overload`, or redesign the interface so the type is known statically.

**Incorrect (using cast to force types):**

```python
from typing import cast

def get_user_name(data: dict[str, object]) -> str:
    name = data["name"]
    return cast(str, name)  # no runtime check, could be int or None

def process_response(response: Response) -> UserDTO:
    payload = response.json()
    return cast(UserDTO, payload)  # dangerous: payload is a dict, not UserDTO
```

**Correct (using isinstance narrowing):**

```python
def get_user_name(data: dict[str, object]) -> str:
    name = data["name"]
    if not isinstance(name, str):
        raise TypeError(f"Expected str for 'name', got {type(name).__name__}")
    return name  # type checker knows this is str

def process_response(response: Response) -> UserDTO:
    payload = response.json()
    return UserDTO.from_dict(payload)  # explicit construction with validation
```

Runtime type narrowing with `isinstance` gives you both type safety and runtime correctness guarantees that `cast()` cannot provide.
