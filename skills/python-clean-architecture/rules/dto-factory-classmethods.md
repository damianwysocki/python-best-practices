---
title: Use Classmethod Factories for DTO Construction
impact: HIGH
impactDescription: Centralizes conversion logic
tags: dto, factory, classmethod, construction, conversion
---

## Use Classmethod Factories for DTO Construction

Use `@classmethod` factory methods to construct DTOs from different sources (database rows, API payloads, other DTOs). This keeps conversion logic co-located with the type definition and provides a clear, discoverable API for constructing objects.

**Incorrect (conversion logic scattered across codebase):**

```python
# In the API handler
user_dto = UserDTO(
    user_id=row["id"],
    name=row["name"],
    email=row["email"],
)

# In the service layer -- same conversion duplicated
user_dto = UserDTO(
    user_id=db_user.id,
    name=db_user.name,
    email=db_user.email,
)
```

**Correct (classmethod factories on the DTO):**

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class UserDTO:
    user_id: str
    name: str
    email: str

    @classmethod
    def from_db_row(cls, row: UserRow) -> "UserDTO":
        return cls(
            user_id=row.user_id,
            name=row.name,
            email=row.email,
        )

    @classmethod
    def from_api_payload(cls, payload: CreateUserRequest) -> "UserDTO":
        return cls(
            user_id=generate_id(),
            name=payload.name,
            email=payload.email,
        )

# Usage is clean and discoverable
user = UserDTO.from_db_row(row)
user = UserDTO.from_api_payload(request)
```

Factory classmethods centralize conversion logic, eliminate duplication, and make it easy to find all the ways a DTO can be constructed.
