---
title: Use __all__ to Control Public API
impact: HIGH
impactDescription: Prevents internal leakage
tags: structure, exports, all, public-api, encapsulation
---

## Use __all__ to Control Public API

Define `__all__` in every `__init__.py` to explicitly declare the public API of each package. This prevents internal implementation details from leaking to consumers and makes it clear what is intended for external use.

**Incorrect (no __all__, everything is public):**

```python
# users/__init__.py
from users.domain import User, UserStatus, _hash_password
from users.service import UserService
from users.repository import PostgresUserRepository, _build_query

# Consumers can import internal helpers
from users import _hash_password  # implementation detail exposed
from users import _build_query    # internal repo helper exposed
```

**Correct (explicit __all__):**

```python
# users/__init__.py
from users.domain import User, UserStatus
from users.service import UserService
from users.interface import UserServiceInterface

__all__ = [
    "User",
    "UserStatus",
    "UserService",
    "UserServiceInterface",
]

# Now:
from users import User            # works, part of public API
from users import _hash_password  # still importable but linters/IDEs warn
# from users import * only exports what's in __all__
```

Every `__init__.py` should have `__all__` listing only the symbols that form the package's public contract.
