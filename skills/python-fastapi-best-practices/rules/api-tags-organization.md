---
title: Use Tags to Organize Endpoints
impact: CRITICAL
impactDescription: Unorganized API docs
tags: openapi, tags, organization, swagger
---

## Use Tags to Organize Endpoints

Use tags to group endpoints by domain in the OpenAPI docs. Define tag metadata at the app level with descriptions and ordering so that consumers can navigate the API logically.

**Incorrect (no tags or inconsistent tagging):**

```python
app = FastAPI()

@app.get("/users")
async def list_users():
    ...

@app.get("/orders")
async def list_orders():
    ...
```

**Correct (tags defined with metadata and applied to routers):**

```python
from fastapi import FastAPI

tags_metadata = [
    {"name": "Users", "description": "User registration and profile management"},
    {"name": "Orders", "description": "Order creation, tracking, and fulfillment"},
    {"name": "Health", "description": "Service health and readiness checks"},
]

app = FastAPI(openapi_tags=tags_metadata)

users_router = APIRouter(prefix="/users", tags=["Users"])
orders_router = APIRouter(prefix="/orders", tags=["Orders"])

app.include_router(users_router)
app.include_router(orders_router)
```

Apply tags at the router level rather than on each individual endpoint. This keeps tagging consistent and reduces duplication. Only override at the endpoint level when an endpoint legitimately belongs to a different group.

Reference: https://fastapi.tiangolo.com/tutorial/metadata/#metadata-for-tags
