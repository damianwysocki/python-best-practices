---
title: Centralize Config in Typed Settings
impact: HIGH
impactDescription: Eliminates scattered env reads
tags: structure, settings, config, pydantic, dataclass, environment
---

## Centralize Config in Typed Settings

Centralize all configuration in a typed dataclass or Pydantic `BaseSettings` class. Never read environment variables directly with `os.getenv()` scattered throughout the codebase. A single settings object provides type safety, validation, and a clear inventory of all configuration.

**Incorrect (scattered env reads):**

```python
# orders/service.py
import os

class OrderService:
    def process(self) -> None:
        max_retries = int(os.getenv("MAX_RETRIES", "3"))  # scattered
        timeout = float(os.getenv("ORDER_TIMEOUT", "30"))  # no validation
        api_key = os.getenv("PAYMENT_API_KEY")  # might be None

# users/service.py
import os

class UserService:
    def __init__(self) -> None:
        self.db_url = os.getenv("DATABASE_URL")  # duplicated pattern
```

**Correct (typed settings class):**

```python
# shared/settings.py
from dataclasses import dataclass, field
import os

@dataclass(frozen=True, slots=True)
class Settings:
    database_url: str
    payment_api_key: str
    max_retries: int = 3
    order_timeout: float = 30.0

    @classmethod
    def from_env(cls) -> "Settings":
        return cls(
            database_url=cls._require_env("DATABASE_URL"),
            payment_api_key=cls._require_env("PAYMENT_API_KEY"),
            max_retries=int(os.getenv("MAX_RETRIES", "3")),
            order_timeout=float(os.getenv("ORDER_TIMEOUT", "30.0")),
        )

    @staticmethod
    def _require_env(key: str) -> str:
        value = os.getenv(key)
        if value is None:
            raise RuntimeError(f"Missing required env var: {key}")
        return value

# main.py (composition root)
settings = Settings.from_env()  # validated once at startup
order_service = OrderService(max_retries=settings.max_retries)
```

All configuration is discoverable in one place, validated at startup, and injected into services as typed values.
