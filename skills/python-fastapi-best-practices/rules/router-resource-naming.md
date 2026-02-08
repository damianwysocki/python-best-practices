---
title: RESTful Resource Naming
impact: HIGH
impactDescription: Inconsistent API paths
tags: router, rest, naming, resources
---

## RESTful Resource Naming

Use plural nouns for resource paths: `/users`, `/orders`, `/products`. Avoid verbs in URLs, action-oriented paths, or inconsistent singular/plural naming. Consistent RESTful naming makes APIs predictable and self-documenting.

**Incorrect (verbs and inconsistent naming):**

```python
router = APIRouter()

@router.post("/createUser")
async def create_user(payload: UserCreate): ...

@router.get("/getUser/{id}")
async def get_user(id: int): ...

@router.get("/order/list")
async def list_orders(): ...

@router.post("/delete-product/{id}")
async def delete_product(id: int): ...
```

**Correct (RESTful plural nouns with HTTP methods):**

```python
router = APIRouter(prefix="/users")

@router.post("/", status_code=201)
async def create_user(payload: UserCreate): ...

@router.get("/{user_id}")
async def get_user(user_id: int): ...

@router.get("/")
async def list_users(skip: int = 0, limit: int = 20): ...

@router.patch("/{user_id}")
async def update_user(user_id: int, payload: UserUpdate): ...

@router.delete("/{user_id}", status_code=204)
async def delete_user(user_id: int): ...
```

For actions that don't map cleanly to CRUD, use a sub-resource:

```python
@router.post("/{user_id}/activate")
async def activate_user(user_id: int): ...
```

Reference: https://restfulapi.net/resource-naming/
