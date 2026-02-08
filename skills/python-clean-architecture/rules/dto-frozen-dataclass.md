---
title: Use Frozen Dataclasses for DTOs
impact: CRITICAL
impactDescription: Guarantees immutability
tags: dto, dataclass, frozen, immutable, slots
---

## Use Frozen Dataclasses for DTOs

All Data Transfer Objects must use `@dataclass(frozen=True, slots=True)`. Frozen dataclasses are immutable after creation, preventing accidental mutation as data flows between layers. The `slots=True` option reduces memory usage and speeds up attribute access.

**Incorrect (mutable class or plain dict):**

```python
class UserDTO:
    def __init__(self, user_id: str, name: str, email: str) -> None:
        self.user_id = user_id
        self.name = name
        self.email = email

dto = UserDTO(user_id="123", name="Alice", email="alice@example.com")
dto.email = "hacked@evil.com"  # mutation allowed, causes bugs downstream
```

**Correct (frozen dataclass):**

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class UserDTO:
    user_id: str
    name: str
    email: str

dto = UserDTO(user_id="123", name="Alice", email="alice@example.com")
dto.email = "hacked@evil.com"  # raises FrozenInstanceError

# To create a modified copy, use dataclasses.replace
from dataclasses import replace
updated = replace(dto, email="alice@newdomain.com")
```

Immutable DTOs eliminate an entire class of bugs where data is accidentally changed in transit between services, layers, or threads.
