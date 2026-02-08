---
title: Use Async for All I/O Operations
impact: HIGH
impactDescription: Blocked event loop
tags: async, await, io, performance
---

## Use Async for All I/O Operations

Use `async def` with `await` for all I/O-bound operations including database queries, HTTP calls, and file operations. Using synchronous `def` for I/O blocks the event loop and serializes all concurrent requests through a threadpool, destroying throughput.

**Incorrect (sync def with blocking I/O):**

```python
import requests

@router.get("/weather/{city}")
def get_weather(city: str):
    # Blocks the event loop thread
    response = requests.get(f"https://api.weather.com/{city}")
    return response.json()

@router.get("/users")
def list_users(db: Session = Depends(get_db)):
    # Synchronous ORM call blocks
    return db.query(User).all()
```

**Correct (async def with async I/O):**

```python
import httpx

@router.get("/weather/{city}")
async def get_weather(city: str) -> WeatherResponse:
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.weather.com/{city}")
        response.raise_for_status()
        return WeatherResponse(**response.json())

@router.get("/users")
async def list_users(
    db: AsyncSession = Depends(get_db),
) -> list[UserResponse]:
    result = await db.execute(select(User))
    return result.scalars().all()
```

Use async-compatible libraries: `httpx` instead of `requests`, `asyncpg`/`aiosqlite` instead of `psycopg2`/`sqlite3`, `aiofiles` instead of built-in `open()`.

Reference: https://fastapi.tiangolo.com/async/
