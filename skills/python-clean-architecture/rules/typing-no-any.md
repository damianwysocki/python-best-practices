---
title: Never Use Any Type
impact: CRITICAL
impactDescription: Destroys type safety
tags: typing, any, generics, protocol, type-safety
---

## Never Use Any Type

Never use `Any` in type annotations. It silently disables type checking for everything it touches. Instead, use proper generics with `TypeVar`, `Protocol` for structural typing, or `Union`/`object` when the type is genuinely unknown.

**Incorrect (using Any):**

```python
from typing import Any

def process_event(event: Any) -> Any:
    return event.handle()  # no type safety, no autocompletion

def store_items(items: list[Any]) -> None:
    for item in items:
        item.save()  # could fail at runtime with no warning

cache: dict[str, Any] = {}
```

**Correct (using proper types):**

```python
from typing import Protocol, TypeVar

class Handleable(Protocol):
    def handle(self) -> EventResult: ...

def process_event(event: Handleable) -> EventResult:
    return event.handle()  # fully type-checked

class Saveable(Protocol):
    def save(self) -> None: ...

def store_items(items: list[Saveable]) -> None:
    for item in items:
        item.save()  # type checker verifies .save() exists

T = TypeVar("T")
cache: dict[str, T]  # generic, preserves type info
```

If you find yourself reaching for `Any`, it usually means you need a Protocol, a TypeVar, or a redesign of the interface.

Reference: [mypy docs on Any](https://mypy.readthedocs.io/en/stable/dynamic_typing.html)
