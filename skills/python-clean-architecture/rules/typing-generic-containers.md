---
title: Use Generic[T] for Reusable Typed Containers
impact: HIGH
impactDescription: Preserves type information
tags: typing, generics, typevar, container, reuse
---

## Use Generic[T] for Reusable Typed Containers

When building reusable classes that operate on different types, use `Generic[T]` with `TypeVar` to preserve type information through the container. This avoids falling back to `Any` or `object` which lose type specificity.

**Incorrect (losing type information):**

```python
class ResultWrapper:
    def __init__(self, value: object, error: str | None = None) -> None:
        self.value = value
        self.error = error

    def unwrap(self) -> object:
        if self.error:
            raise ValueError(self.error)
        return self.value  # caller must cast or guess the type

result = ResultWrapper(value=User(name="Alice"))
user = result.unwrap()  # type is 'object', not 'User'
```

**Correct (using Generic[T]):**

```python
from typing import Generic, TypeVar

T = TypeVar("T")

class Result(Generic[T]):
    def __init__(self, value: T, error: str | None = None) -> None:
        self._value = value
        self._error = error

    def unwrap(self) -> T:
        if self._error:
            raise ValueError(self._error)
        return self._value

result = Result(value=User(name="Alice"))
user = result.unwrap()  # type checker knows this is 'User'
```

Generic containers preserve full type information, enabling autocompletion, refactoring support, and compile-time error detection.
