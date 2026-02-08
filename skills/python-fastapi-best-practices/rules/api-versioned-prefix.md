---
title: Version APIs with Prefix
impact: HIGH
impactDescription: Breaking client changes
tags: openapi, versioning, router, prefix
---

## Version APIs with Prefix

Version APIs with `/api/v1/` prefix using `APIRouter`. URL-based versioning is explicit, visible in logs, and easy to route at the infrastructure level. Always plan for version coexistence so clients can migrate incrementally.

**Incorrect (no versioning or inconsistent paths):**

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/users")
async def list_users():
    ...

@app.get("/new-users")  # ad-hoc path for "v2"
async def list_users_v2():
    ...
```

**Correct (versioned with APIRouter prefix):**

```python
from fastapi import APIRouter, FastAPI

app = FastAPI(title="My API")

v1_router = APIRouter(prefix="/api/v1")
v2_router = APIRouter(prefix="/api/v2")

# v1 endpoints
v1_users = APIRouter(prefix="/users", tags=["Users v1"])
v1_router.include_router(v1_users)

# v2 endpoints with breaking changes
v2_users = APIRouter(prefix="/users", tags=["Users v2"])
v2_router.include_router(v2_users)

app.include_router(v1_router)
app.include_router(v2_router)
```

Keep version-specific logic in separate modules to avoid tangled code:

```
app/
    api/
        v1/
            routes/
                users.py
        v2/
            routes/
                users.py
```

Reference: https://fastapi.tiangolo.com/tutorial/bigger-applications/
