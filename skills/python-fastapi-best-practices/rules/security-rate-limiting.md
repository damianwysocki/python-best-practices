---
title: Apply Rate Limiting
impact: MEDIUM
impactDescription: Denial of service risk
tags: security, rate-limiting, middleware, throttle
---

## Apply Rate Limiting

Apply rate limiting via middleware or dependency to protect endpoints from abuse. Without rate limits, a single client can exhaust server resources with rapid requests. Implement at the dependency level for per-endpoint control or at the middleware level for global limits.

**Incorrect (no rate limiting):**

```python
@router.post("/auth/login")
async def login(credentials: LoginRequest):
    # No rate limit — brute force attacks possible
    return await auth_service.authenticate(credentials)
```

**Correct (rate limiting via dependency):**

```python
from datetime import datetime
from collections import defaultdict
from fastapi import Depends, HTTPException, Request

class RateLimiter:
    def __init__(self, max_requests: int, window_seconds: int):
        self.max_requests = max_requests
        self.window = window_seconds
        self.requests: dict[str, list[float]] = defaultdict(list)

    async def __call__(self, request: Request) -> None:
        client_ip = request.client.host
        now = datetime.now().timestamp()
        window_start = now - self.window
        self.requests[client_ip] = [
            ts for ts in self.requests[client_ip] if ts > window_start
        ]
        if len(self.requests[client_ip]) >= self.max_requests:
            raise HTTPException(
                status_code=429,
                detail="Too many requests",
                headers={"Retry-After": str(self.window)},
            )
        self.requests[client_ip].append(now)

rate_limit_login = RateLimiter(max_requests=5, window_seconds=60)

@router.post("/auth/login", dependencies=[Depends(rate_limit_login)])
async def login(credentials: LoginRequest) -> TokenResponse:
    return await auth_service.authenticate(credentials)
```

For production, use Redis-backed rate limiting (e.g., `slowapi` or a custom Redis limiter) to share state across workers.

Reference: https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-in-path-operation-decorators/
