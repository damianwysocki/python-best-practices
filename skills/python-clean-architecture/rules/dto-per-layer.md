---
title: Separate DTOs per Layer
impact: HIGH
impactDescription: Decouples layer changes
tags: dto, layers, api, service, persistence, separation
---

## Separate DTOs per Layer

Define distinct DTO types for each architectural layer: API request/response models, service-layer DTOs, and persistence models. Sharing a single model across layers creates tight coupling where a database schema change forces API changes and vice versa.

**Incorrect (single model shared across layers):**

```python
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    email: str
    password_hash: str  # exposed to API layer
    created_at: datetime  # database concern in API
    internal_score: float  # internal detail leaked everywhere

# Used in API, service, and persistence -- all tightly coupled
```

**Correct (separate DTOs per layer):**

```python
from dataclasses import dataclass

# API layer
@dataclass(frozen=True, slots=True)
class CreateUserRequest:
    name: str
    email: str
    password: str

@dataclass(frozen=True, slots=True)
class UserResponse:
    user_id: str
    name: str
    email: str

# Service layer
@dataclass(frozen=True, slots=True)
class UserEntity:
    user_id: str
    name: str
    email: str
    password_hash: str

# Persistence layer
@dataclass(frozen=True, slots=True)
class UserRow:
    id: int
    user_id: str
    name: str
    email: str
    password_hash: str
    created_at: datetime
    internal_score: float
```

Each layer evolves independently. Adding a database column does not change the API contract, and changing the API response does not affect persistence.
