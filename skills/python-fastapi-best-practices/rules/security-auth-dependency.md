---
title: Implement Auth as Reusable Dependency
impact: HIGH
impactDescription: Inconsistent auth checks
tags: security, auth, dependency, jwt
---

## Implement Auth as Reusable Dependency

Implement authentication as a reusable `Depends()` callable. Never check auth inline in endpoints. A shared auth dependency ensures every protected endpoint uses the same validation logic, and forgetting to add it is immediately visible in code review.

**Incorrect (auth logic duplicated in endpoints):**

```python
@router.get("/users/me")
async def get_me(request: Request, db=Depends(get_db)):
    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    try:
        payload = jwt.decode(token, SECRET, algorithms=["HS256"])
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401)
    user = await db.get(User, payload["sub"])
    return user

@router.get("/orders")  # duplicated auth logic or, worse, forgotten
async def list_orders(request: Request, db=Depends(get_db)):
    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    ...
```

**Correct (reusable auth dependency):**

```python
from fastapi import Depends
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    credentials_exception = HTTPException(
        status_code=401,
        detail="Invalid credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    user = await db.get(User, user_id)
    if user is None:
        raise credentials_exception
    return user

@router.get("/users/me")
async def get_me(user: User = Depends(get_current_user)) -> UserResponse:
    return UserResponse.model_validate(user)
```

Reference: https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/
