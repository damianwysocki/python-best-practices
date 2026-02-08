---
title: Build Composable Dependencies
impact: HIGH
impactDescription: Duplicated dependency logic
tags: dependency-injection, composable, reuse, depends
---

## Build Composable Dependencies

Build complex dependencies from simple, composable ones. Each dependency should do one thing. Chain them together so that higher-level dependencies declare their own sub-dependencies via `Depends()`. FastAPI resolves the full graph automatically and caches results within a request.

**Incorrect (monolithic dependency doing too much):**

```python
async def get_everything(request: Request):
    token = request.headers.get("Authorization")
    user = await verify_token(token)
    db = SessionLocal()
    permissions = await db.execute(
        select(Permission).where(Permission.user_id == user.id)
    )
    return {"user": user, "db": db, "permissions": permissions}
```

**Correct (composable single-responsibility dependencies):**

```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    user = await auth_service.verify_token(token, db)
    if not user:
        raise HTTPException(status_code=401)
    return user

async def get_current_active_user(
    user: User = Depends(get_current_user),
) -> User:
    if not user.is_active:
        raise HTTPException(status_code=403, detail="Inactive user")
    return user

def require_role(role: str):
    async def check_role(
        user: User = Depends(get_current_active_user),
    ) -> User:
        if role not in user.roles:
            raise HTTPException(status_code=403)
        return user
    return check_role

@router.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin: User = Depends(require_role("admin")),
    service: UserService = Depends(get_user_service),
) -> None:
    await service.delete(user_id)
```

Each dependency is independently testable and reusable across multiple endpoints.

Reference: https://fastapi.tiangolo.com/tutorial/dependencies/sub-dependencies/
