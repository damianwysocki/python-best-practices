---
title: Use Final for Constants
impact: MEDIUM
impactDescription: Prevents accidental reassignment
tags: typing, final, constants, immutability
---

## Use Final for Constants

Mark module-level and class-level constants with `Final` to signal that they must never be reassigned. The type checker will flag any reassignment as an error, preventing accidental mutation of configuration values and magic numbers.

**Incorrect (plain variables for constants):**

```python
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30.0
API_VERSION = "v2"

# Somewhere deep in the codebase...
MAX_RETRIES = 10  # accidental reassignment, no warning
```

**Correct (Final annotation):**

```python
from typing import Final

MAX_RETRIES: Final = 3
DEFAULT_TIMEOUT: Final[float] = 30.0
API_VERSION: Final[str] = "v2"

# Somewhere deep in the codebase...
MAX_RETRIES = 10  # type error: cannot assign to final name

class HttpClient:
    BASE_URL: Final[str] = "https://api.example.com"
    TIMEOUT: Final[int] = 30

    def connect(self) -> None:
        self.BASE_URL = "http://other.com"  # type error
```

`Final` provides a clear contract that a value is not meant to change, catching mistakes that would otherwise lead to hard-to-debug configuration drift.
