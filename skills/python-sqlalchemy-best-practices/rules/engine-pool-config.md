---
title: Configure Connection Pool for Production
impact: MEDIUM
impactDescription: Prevents connection exhaustion
tags: engine, pool, connection, production, performance
---

## Configure Connection Pool for Production

The default pool settings (`pool_size=5`, `max_overflow=10`) are designed for
development. In production, misconfigured pools cause connection exhaustion
under load, stale connections after database restarts, and slow recovery. Always
set `pool_size`, `max_overflow`, `pool_recycle`, and `pool_pre_ping`.

**Incorrect (default pool settings in production):**

```python
from sqlalchemy import create_engine

# Defaults: pool_size=5, max_overflow=10, no recycle, no pre_ping
engine = create_engine("postgresql://localhost/mydb")
```

**Correct (production-tuned pool configuration):**

```python
from sqlalchemy import create_engine

engine = create_engine(
    "postgresql://localhost/mydb",
    pool_size=20,           # Steady-state connections
    max_overflow=10,        # Burst capacity above pool_size
    pool_recycle=3600,      # Recycle connections after 1 hour
    pool_pre_ping=True,     # Test connection before checkout
    pool_timeout=30,        # Seconds to wait for a connection
    echo=False,             # Never True in production
)
```

Key parameters explained:
- `pool_size`: Number of persistent connections. Match to your expected
  concurrency, not to the DB max connections.
- `max_overflow`: Extra connections allowed under burst load. These are
  discarded when returned to the pool.
- `pool_recycle`: Prevents the database or firewall from dropping idle
  connections. Set below your DB's `wait_timeout`.
- `pool_pre_ping`: Issues a lightweight `SELECT 1` before each checkout
  to detect disconnected connections.

For async engines, the same parameters apply to `create_async_engine`.

Reference: https://docs.sqlalchemy.org/en/20/core/pooling.html
