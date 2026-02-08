---
title: Prefer Protocol for Structural Subtyping
impact: HIGH
impactDescription: Reduces coupling overhead
tags: typing, protocol, abc, structural-subtyping
---

## Prefer Protocol for Structural Subtyping

Use `Protocol` for structural subtyping (duck typing with type safety) instead of ABC when implementations do not need to explicitly inherit. Protocol lets any class that has the right methods satisfy the type, without requiring an inheritance relationship.

**Incorrect (forcing ABC inheritance):**

```python
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    def find_by_id(self, entity_id: str) -> dict: ...

    @abstractmethod
    def save(self, entity: dict) -> None: ...

# Every implementation must explicitly inherit
class PostgresRepository(Repository):
    def find_by_id(self, entity_id: str) -> dict: ...
    def save(self, entity: dict) -> None: ...

# Third-party adapter cannot inherit without wrapping
class ThirdPartyAdapter(Repository):  # forced coupling
    ...
```

**Correct (using Protocol):**

```python
from typing import Protocol

class Repository(Protocol):
    def find_by_id(self, entity_id: str) -> dict: ...
    def save(self, entity: dict) -> None: ...

# No inheritance needed -- structural conformance
class PostgresRepository:
    def find_by_id(self, entity_id: str) -> dict: ...
    def save(self, entity: dict) -> None: ...

# Third-party adapters automatically conform if methods match
class ThirdPartyAdapter:
    def find_by_id(self, entity_id: str) -> dict: ...
    def save(self, entity: dict) -> None: ...
```

Reserve ABC for cases where you need shared implementation (template method pattern) or runtime `isinstance` checks with `runtime_checkable`.
