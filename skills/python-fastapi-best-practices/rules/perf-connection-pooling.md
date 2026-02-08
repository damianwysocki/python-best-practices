---
title: Configure Connection Pool Size
impact: MEDIUM
impactDescription: Connection exhaustion
tags: performance, database, pool, connections
---

## Configure Connection Pool Size

Configure the async connection pool size to match your worker count and expected concurrency. An undersized pool causes requests to queue waiting for a connection. An oversized pool wastes database resources and can hit server connection limits.

**Incorrect (default pool size, no tuning):**

```python
from sqlalchemy.ext.asyncio import create_async_engine

# Default pool_size=5 with 4 workers = 20 connections.
# Under load with 100 concurrent requests, 80 queue.
engine = create_async_engine("postgresql+asyncpg://localhost/mydb")
```

**Correct (pool sized to match deployment):**

```python
from sqlalchemy.ext.asyncio import create_async_engine
from app.config import settings

engine = create_async_engine(
    settings.database_url,
    pool_size=settings.db_pool_size,       # e.g., 20 per worker
    max_overflow=settings.db_max_overflow,  # e.g., 10 burst
    pool_timeout=30,                        # seconds to wait for conn
    pool_recycle=1800,                      # recycle conns every 30min
    pool_pre_ping=True,                     # verify conn is alive
)
```

**Sizing guideline:** Set `pool_size` per worker to `(max_concurrent_requests_per_worker / avg_queries_per_request)`. For example, 4 uvicorn workers with `pool_size=20` and `max_overflow=10` gives 120 max connections. Verify this is within your database's `max_connections` setting.

```python
# config.py — derive pool size from environment
class Settings(BaseModel):
    db_pool_size: int = 20
    db_max_overflow: int = 10
    workers: int = 4

    @computed_field
    @property
    def max_total_connections(self) -> int:
        return (self.db_pool_size + self.db_max_overflow) * self.workers
```

Reference: https://docs.sqlalchemy.org/en/20/core/pooling.html
