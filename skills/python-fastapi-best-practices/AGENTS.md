# Python FastAPI Best Practices

**Version 1.0.0** | Python / FastAPI | February 2026

> **Note:** This document contains 28 rules for building production-grade FastAPI applications. Rules are organized by category and priority. Each rule includes impact assessment, explanation, and incorrect/correct code examples. Apply CRITICAL rules first, then HIGH, MEDIUM, and LOW.

## Abstract

This guide codifies 28 best practices for FastAPI application development, covering OpenAPI documentation, dependency injection, Pydantic model design, router architecture, async patterns, security, and performance. Each rule is derived from real-world production experience and addresses a specific failure mode. Rules are prioritized by impact so teams can adopt them incrementally.

## Table of Contents

1. [OpenAPI Documentation](#1-openapi-documentation)
   - 1.1 Every Endpoint Must Have Descriptions
   - 1.2 Use Tags to Organize Endpoints
   - 1.3 Always Specify Response Models
   - 1.4 Document All Error Responses
   - 1.5 Provide Request and Response Examples
   - 1.6 Version APIs with Prefix
2. [Dependency Injection](#2-dependency-injection)
   - 2.1 Inject Services via Depends
   - 2.2 Scoped Database Session via Depends
   - 2.3 Depend on Abstractions for Testing
   - 2.4 Build Composable Dependencies
3. [Pydantic Models](#3-pydantic-models)
   - 3.1 Enable Strict Mode with Field Validators
   - 3.2 Separate Schemas per Operation
   - 3.3 Never Return ORM Models Directly
   - 3.4 Use Computed Fields for Derived Data
   - 3.5 Extract Reusable Field Patterns
4. [Router Architecture](#4-router-architecture)
   - 4.1 Keep Endpoints Slim
   - 4.2 RESTful Resource Naming
   - 4.3 Register Custom Exception Handlers
   - 4.4 Correct Middleware Ordering
5. [Async Patterns](#5-async-patterns)
   - 5.1 Use Async for All I/O Operations
   - 5.2 Use asyncio.gather for Parallel Operations
   - 5.3 Never Call Blocking I/O in Async Endpoints
   - 5.4 Use BackgroundTasks for Fire-and-Forget
6. [Security & Validation](#6-security--validation)
   - 6.1 Implement Auth as Reusable Dependency
   - 6.2 Apply Rate Limiting
   - 6.3 Configure CORS with Explicit Origins
7. [Performance](#7-performance)
   - 7.1 Configure Connection Pool Size
   - 7.2 Use StreamingResponse for Large Payloads

---

## 1. OpenAPI Documentation

### 1.1 Every Endpoint Must Have Descriptions

**Impact: CRITICAL — Missing API documentation**

Every FastAPI endpoint must include `summary`, `description`, and `response_description` parameters. Without these, auto-generated OpenAPI docs become unusable for consumers. The summary appears in the endpoint list, the description provides detail, and response_description documents the successful return value.

**Incorrect (missing documentation parameters):**

```python
@router.get("/users/{user_id}")
async def get_user(user_id: int):
    user = await user_service.get_by_id(user_id)
    return user
```

**Correct (fully documented endpoint):**

```python
@router.get(
    "/users/{user_id}",
    summary="Get user by ID",
    description="Retrieve a single user by their unique identifier. "
    "Returns 404 if the user does not exist.",
    response_description="The requested user object",
)
async def get_user(user_id: int) -> UserResponse:
    user = await user_service.get_by_id(user_id)
    return user
```

For endpoints with complex behavior, use multi-line docstrings as an alternative to the `description` parameter. FastAPI will use the docstring if no explicit description is provided.

```python
@router.post("/users", summary="Create a new user")
async def create_user(payload: UserCreate) -> UserResponse:
    """
    Create a new user with the provided details.

    - **email**: must be unique across the system
    - **password**: minimum 8 characters, hashed before storage
    """
    return await user_service.create(payload)
```

### 1.2 Use Tags to Organize Endpoints

**Impact: CRITICAL — Unorganized API docs**

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

Apply tags at the router level rather than on each individual endpoint. This keeps tagging consistent and reduces duplication.

### 1.3 Always Specify Response Models

**Impact: CRITICAL — Unvalidated API responses**

Always specify `response_model` or a return type annotation with explicit `status_code` on every endpoint. This ensures response validation, filters out unintended fields, and generates accurate OpenAPI schemas.

**Incorrect (no response model, implicit 200):**

```python
@router.post("/users")
async def create_user(payload: UserCreate):
    user = await user_service.create(payload)
    return user  # ORM model leaks internal fields
```

**Correct (explicit response model and status code):**

```python
from fastapi import status

@router.post(
    "/users",
    response_model=UserResponse,
    status_code=status.HTTP_201_CREATED,
    summary="Create a new user",
)
async def create_user(payload: UserCreate) -> UserResponse:
    user = await user_service.create(payload)
    return UserResponse.model_validate(user)
```

Use `response_model_exclude_none=True` when you want to omit null fields from responses to keep payloads clean.

### 1.4 Document All Error Responses

**Impact: HIGH — Hidden failure modes**

Document all error responses using the `responses` parameter on each endpoint. Undocumented error codes leave API consumers guessing about failure modes.

**Incorrect (errors not documented in OpenAPI spec):**

```python
@router.get("/users/{user_id}")
async def get_user(user_id: int) -> UserResponse:
    user = await user_service.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

**Correct (error responses documented with responses parameter):**

```python
@router.get(
    "/users/{user_id}",
    response_model=UserResponse,
    responses={
        404: {
            "description": "User not found",
            "content": {
                "application/json": {
                    "example": {"detail": "User not found"}
                }
            },
        },
        422: {"description": "Invalid user ID format"},
    },
    summary="Get user by ID",
)
async def get_user(user_id: int) -> UserResponse:
    user = await user_service.get_by_id(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

Define a shared error model to reuse across endpoints for consistency:

```python
class ErrorResponse(BaseModel):
    detail: str

common_error_responses = {
    401: {"model": ErrorResponse, "description": "Not authenticated"},
    403: {"model": ErrorResponse, "description": "Forbidden"},
}
```

### 1.5 Provide Request and Response Examples

**Impact: HIGH — Unclear API contracts**

Provide request and response examples via `model_config` with `json_schema_extra` on Pydantic models. Examples appear in the OpenAPI docs and make it trivial for consumers to understand the expected payload shape.

**Incorrect (no examples in schema):**

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    email: str
    full_name: str
    password: str
```

**Correct (examples provided via model_config):**

```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    email: EmailStr
    full_name: str
    password: str

    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "email": "jane@example.com",
                    "full_name": "Jane Doe",
                    "password": "s3cure!Pass",
                }
            ]
        }
    }
```

You can also provide examples at the field level using `Field`:

```python
from pydantic import BaseModel, Field

class OrderCreate(BaseModel):
    product_id: int = Field(..., examples=[42])
    quantity: int = Field(..., ge=1, examples=[3])
    notes: str | None = Field(None, examples=["Gift wrap please"])
```

### 1.6 Version APIs with Prefix

**Impact: HIGH — Breaking client changes**

Version APIs with `/api/v1/` prefix using `APIRouter`. URL-based versioning is explicit, visible in logs, and easy to route at the infrastructure level.

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

v1_users = APIRouter(prefix="/users", tags=["Users v1"])
v1_router.include_router(v1_users)

v2_users = APIRouter(prefix="/users", tags=["Users v2"])
v2_router.include_router(v2_users)

app.include_router(v1_router)
app.include_router(v2_router)
```

Keep version-specific logic in separate modules to avoid tangled code.

---

## 2. Dependency Injection

### 2.1 Inject Services via Depends

**Impact: CRITICAL — Untestable coupled code**

Inject services via `Depends()`; never instantiate services directly in endpoint bodies. Direct instantiation couples endpoints to concrete implementations, making testing difficult and preventing service reuse.

**Incorrect (service instantiated in endpoint):**

```python
from app.services.user_service import UserService
from app.db import get_session

@router.get("/users/{user_id}")
async def get_user(user_id: int):
    session = get_session()
    service = UserService(session)  # tight coupling
    return await service.get_by_id(user_id)
```

**Correct (service injected via Depends):**

```python
from fastapi import Depends
from app.services.user_service import UserService

def get_user_service(
    session: AsyncSession = Depends(get_async_session),
) -> UserService:
    return UserService(session)

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    return await service.get_by_id(user_id)
```

This allows overriding the dependency in tests:

```python
app.dependency_overrides[get_user_service] = lambda: mock_service
```

### 2.2 Scoped Database Session via Depends

**Impact: CRITICAL — Leaked database connections**

Use a `Depends()` generator (with `yield`) for database session lifecycle management. The generator ensures the session is properly closed after each request, even if an exception occurs.

**Incorrect (manual session management in endpoint):**

```python
from app.db import SessionLocal

@router.get("/users")
async def list_users():
    session = SessionLocal()
    try:
        users = session.query(User).all()
        return users
    finally:
        session.close()  # duplicated everywhere
```

**Correct (generator dependency manages session lifecycle):**

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(DATABASE_URL, pool_size=20)
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

@router.get("/users")
async def list_users(
    db: AsyncSession = Depends(get_db),
) -> list[UserResponse]:
    result = await db.execute(select(User))
    return result.scalars().all()
```

The `async with` context manager and the `yield` pattern guarantee cleanup runs regardless of success or failure.

### 2.3 Depend on Abstractions for Testing

**Impact: HIGH — Hard-to-test dependencies**

Use `Depends()` with `Protocol` or `ABC` types so implementations can be swapped for testing. This inversion of control decouples endpoints from infrastructure details.

**Incorrect (endpoint depends on concrete implementation):**

```python
from app.repositories.user_repo import PostgresUserRepo

def get_repo():
    return PostgresUserRepo(connection_string=PROD_DB)

@router.get("/users/{user_id}")
async def get_user(user_id: int, repo=Depends(get_repo)):
    return await repo.get(user_id)
```

**Correct (endpoint depends on Protocol, swap in tests):**

```python
from typing import Protocol

class UserRepository(Protocol):
    async def get(self, user_id: int) -> User | None: ...
    async def create(self, data: UserCreate) -> User: ...

def get_user_repo(
    db: AsyncSession = Depends(get_db),
) -> UserRepository:
    return PostgresUserRepo(db)

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    repo: UserRepository = Depends(get_user_repo),
) -> UserResponse:
    user = await repo.get(user_id)
    if not user:
        raise HTTPException(status_code=404)
    return user
```

In tests, override with an in-memory implementation:

```python
class FakeUserRepo:
    async def get(self, user_id: int) -> User | None:
        return User(id=user_id, email="test@test.com")

app.dependency_overrides[get_user_repo] = lambda: FakeUserRepo()
```

### 2.4 Build Composable Dependencies

**Impact: HIGH — Duplicated dependency logic**

Build complex dependencies from simple, composable ones. Each dependency should do one thing. Chain them together so that higher-level dependencies declare their own sub-dependencies via `Depends()`.

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

---

## 3. Pydantic Models

### 3.1 Enable Strict Mode with Field Validators

**Impact: HIGH — Silent type coercion**

Enable strict mode on Pydantic models and use field validators for business rules. Without strict mode, Pydantic silently coerces types (e.g., `"123"` becomes `123`), masking client bugs.

**Incorrect (loose coercion, no business validation):**

```python
from pydantic import BaseModel

class OrderCreate(BaseModel):
    quantity: int       # silently accepts "5" as 5
    discount: float     # accepts any float, even negative
    email: str          # accepts any string
```

**Correct (strict mode with field validators):**

```python
from pydantic import BaseModel, EmailStr, field_validator

class OrderCreate(BaseModel):
    model_config = {"strict": True}

    quantity: int
    discount: float
    email: EmailStr

    @field_validator("quantity")
    @classmethod
    def quantity_must_be_positive(cls, v: int) -> int:
        if v <= 0:
            raise ValueError("Quantity must be greater than zero")
        return v

    @field_validator("discount")
    @classmethod
    def discount_in_range(cls, v: float) -> float:
        if not (0.0 <= v <= 1.0):
            raise ValueError("Discount must be between 0 and 1")
        return v
```

With strict mode, sending `{"quantity": "5"}` now returns a 422 error instead of silently coercing.

### 3.2 Separate Schemas per Operation

**Impact: HIGH — Over-exposed data fields**

Create separate `CreateSchema`, `UpdateSchema`, and `ResponseSchema` for each resource. A single shared model either exposes internal fields to clients or prevents required fields on creation.

**Incorrect (single model for all operations):**

```python
from pydantic import BaseModel

class User(BaseModel):
    id: int
    email: str
    password: str       # leaked in responses
    is_active: bool
    created_at: datetime  # client shouldn't set this
```

**Correct (separate schemas per operation):**

```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    email: EmailStr
    password: str
    full_name: str

class UserUpdate(BaseModel):
    full_name: str | None = None
    email: EmailStr | None = None

class UserResponse(BaseModel):
    id: int
    email: EmailStr
    full_name: str
    is_active: bool
    created_at: datetime

    model_config = {"from_attributes": True}
```

The `UserResponse` omits `password`. The `UserCreate` requires `password` but not `id`. The `UserUpdate` makes all fields optional for partial updates.

### 3.3 Never Return ORM Models Directly

**Impact: HIGH — Leaked internal fields**

Never return ORM models directly from endpoints. Always map ORM objects to a Pydantic response schema. Returning ORM models leaks database columns (passwords, internal flags), breaks when lazy-loaded relationships trigger outside a session, and couples your API contract to your database schema.

**Incorrect (returning ORM model directly):**

```python
from app.models import User  # SQLAlchemy model

@router.get("/users/{user_id}")
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)
    return user  # exposes password_hash, internal flags, etc.
```

**Correct (map to response schema):**

```python
from app.schemas.user import UserResponse

@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return UserResponse.model_validate(user)
```

Use `model_config = {"from_attributes": True}` on the response schema so `model_validate()` reads from ORM attribute access.

### 3.4 Use Computed Fields for Derived Data

**Impact: MEDIUM — Redundant stored data**

Use `@computed_field` for derived data in response models instead of storing computed values in the database or calculating them in the endpoint. Computed fields keep derivation logic co-located with the schema definition.

**Incorrect (computing derived values in the endpoint):**

```python
@router.get("/orders/{order_id}")
async def get_order(order_id: int, service=Depends(get_order_service)):
    order = await service.get(order_id)
    return {
        **order.dict(),
        "total_price": order.quantity * order.unit_price,
        "display_name": f"Order #{order.id}",
    }
```

**Correct (computed fields on the response model):**

```python
from pydantic import BaseModel, computed_field

class OrderResponse(BaseModel):
    model_config = {"from_attributes": True}

    id: int
    quantity: int
    unit_price: float
    status: str

    @computed_field
    @property
    def total_price(self) -> float:
        return self.quantity * self.unit_price

    @computed_field
    @property
    def display_name(self) -> str:
        return f"Order #{self.id}"
```

The `total_price` and `display_name` fields appear in the JSON output and OpenAPI schema without being stored or computed outside the model.

### 3.5 Extract Reusable Field Patterns

**Impact: MEDIUM — Duplicated field definitions**

Extract common field patterns into base schemas to eliminate duplication. When multiple models share the same fields (timestamps, pagination, audit trails), define them once in a base class.

**Incorrect (duplicated fields across models):**

```python
class UserResponse(BaseModel):
    id: int
    created_at: datetime
    updated_at: datetime
    email: str

class OrderResponse(BaseModel):
    id: int
    created_at: datetime
    updated_at: datetime  # duplicated fields
    total: float
```

**Correct (shared base schemas):**

```python
from pydantic import BaseModel, Field

class TimestampMixin(BaseModel):
    created_at: datetime
    updated_at: datetime

class IdentifiedModel(TimestampMixin):
    id: int

    model_config = {"from_attributes": True}

class UserResponse(IdentifiedModel):
    email: str
    full_name: str

class OrderResponse(IdentifiedModel):
    total: float
    status: str

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int = Field(ge=1)
    page_size: int = Field(ge=1, le=100)
    has_next: bool
```

---

## 4. Router Architecture

### 4.1 Keep Endpoints Slim

**Impact: HIGH — Untestable route logic**

Endpoints should call services only; keep zero business logic in route handlers. The endpoint's job is to accept validated input, call a service, and return a response.

**Incorrect (business logic embedded in endpoint):**

```python
@router.post("/orders")
async def create_order(
    payload: OrderCreate,
    db: AsyncSession = Depends(get_db),
):
    product = await db.get(Product, payload.product_id)
    if not product or product.stock < payload.quantity:
        raise HTTPException(400, "Insufficient stock")
    total = product.price * payload.quantity * (1 - payload.discount)
    order = Order(total=total, **payload.model_dump())
    db.add(order)
    product.stock -= payload.quantity
    await db.commit()
    await send_confirmation_email(order)
    return order
```

**Correct (endpoint delegates to service):**

```python
@router.post(
    "/orders",
    response_model=OrderResponse,
    status_code=status.HTTP_201_CREATED,
)
async def create_order(
    payload: OrderCreate,
    service: OrderService = Depends(get_order_service),
) -> OrderResponse:
    return await service.create_order(payload)
```

The service encapsulates validation, calculation, persistence, and side effects. The endpoint is a thin adapter between HTTP and the domain.

### 4.2 RESTful Resource Naming

**Impact: HIGH — Inconsistent API paths**

Use plural nouns for resource paths: `/users`, `/orders`, `/products`. Avoid verbs in URLs, action-oriented paths, or inconsistent singular/plural naming.

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

### 4.3 Register Custom Exception Handlers

**Impact: HIGH — Inconsistent error formats**

Register custom exception handlers at the app level; never use try/except in endpoints to format error responses. Centralized handlers ensure every error follows the same response format.

**Incorrect (try/except in every endpoint):**

```python
@router.get("/users/{user_id}")
async def get_user(user_id: int, service=Depends(get_user_service)):
    try:
        user = await service.get_by_id(user_id)
        return user
    except UserNotFoundError:
        return JSONResponse(status_code=404, content={"error": "Not found"})
    except PermissionError:
        return JSONResponse(status_code=403, content={"error": "Forbidden"})
```

**Correct (custom exception handlers registered on app):**

```python
# app/exceptions.py
class AppException(Exception):
    def __init__(self, status_code: int, detail: str):
        self.status_code = status_code
        self.detail = detail

class NotFoundError(AppException):
    def __init__(self, resource: str, resource_id: int):
        super().__init__(404, f"{resource} with id {resource_id} not found")

# app/main.py
@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.detail},
    )

# app/routes/users.py — clean, no try/except
@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    return await service.get_by_id(user_id)  # raises NotFoundError
```

Services raise domain exceptions; the handler translates them to HTTP responses.

### 4.4 Correct Middleware Ordering

**Impact: MEDIUM — Bypassed security layers**

Apply middleware in the correct order: CORS, Authentication, Logging, Error handling. FastAPI middleware executes in reverse registration order (last registered runs first on request). Incorrect ordering can bypass security checks.

**Incorrect (auth before CORS, errors not caught):**

```python
app = FastAPI()

app.add_middleware(AuthMiddleware)
app.add_middleware(CORSMiddleware, allow_origins=["*"])
app.add_middleware(LoggingMiddleware)
```

**Correct (CORS first, then auth, logging, errors):**

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Registered in reverse execution order.
# Last registered = first to run on incoming request.

# 4. Error handling (innermost)
app.add_middleware(ErrorHandlingMiddleware)

# 3. Request logging
app.add_middleware(LoggingMiddleware)

# 2. Authentication
app.add_middleware(AuthMiddleware)

# 1. CORS (outermost — handles preflight before anything else)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_methods=["*"],
    allow_headers=["*"],
    allow_credentials=True,
)
```

The CORS middleware must handle OPTIONS preflight requests before auth rejects them.

---

## 5. Async Patterns

### 5.1 Use Async for All I/O Operations

**Impact: HIGH — Blocked event loop**

Use `async def` with `await` for all I/O-bound operations including database queries, HTTP calls, and file operations. Using synchronous `def` for I/O blocks the event loop and serializes all concurrent requests through a threadpool.

**Incorrect (sync def with blocking I/O):**

```python
import requests

@router.get("/weather/{city}")
def get_weather(city: str):
    response = requests.get(f"https://api.weather.com/{city}")
    return response.json()

@router.get("/users")
def list_users(db: Session = Depends(get_db)):
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

### 5.2 Use asyncio.gather for Parallel Operations

**Impact: HIGH — Sequential async waste**

Use `asyncio.gather()` for independent async operations that can run concurrently. Awaiting operations sequentially when they have no data dependency wastes time.

**Incorrect (sequential awaits for independent operations):**

```python
@router.get("/dashboard/{user_id}")
async def get_dashboard(
    user_id: int,
    user_svc: UserService = Depends(get_user_service),
    order_svc: OrderService = Depends(get_order_service),
    notification_svc: NotificationService = Depends(get_notification_service),
):
    user = await user_svc.get(user_id)            # 100ms
    orders = await order_svc.get_recent(user_id)   # 150ms
    alerts = await notification_svc.get(user_id)   # 80ms
    # Total: ~330ms (sequential)
    return {"user": user, "orders": orders, "alerts": alerts}
```

**Correct (parallel with asyncio.gather):**

```python
import asyncio

@router.get("/dashboard/{user_id}")
async def get_dashboard(
    user_id: int,
    user_svc: UserService = Depends(get_user_service),
    order_svc: OrderService = Depends(get_order_service),
    notification_svc: NotificationService = Depends(get_notification_service),
) -> DashboardResponse:
    user, orders, alerts = await asyncio.gather(
        user_svc.get(user_id),
        order_svc.get_recent(user_id),
        notification_svc.get(user_id),
    )
    # Total: ~150ms (parallel, limited by slowest)
    return DashboardResponse(user=user, orders=orders, alerts=alerts)
```

Use `return_exceptions=True` if you want to handle partial failures instead of having one failure cancel everything.

### 5.3 Never Call Blocking I/O in Async Endpoints

**Impact: HIGH — Event loop starvation**

Never call blocking (synchronous) I/O inside an `async def` endpoint. Blocking calls freeze the entire event loop, preventing all other requests from being processed. If you must use a sync library, offload it to a thread via `run_in_executor`.

**Incorrect (blocking call in async endpoint):**

```python
import time
from PIL import Image

@router.post("/resize")
async def resize_image(file: UploadFile):
    contents = await file.read()
    img = Image.open(io.BytesIO(contents))
    img = img.resize((200, 200))
    buffer = io.BytesIO()
    img.save(buffer, format="PNG")
    return Response(content=buffer.getvalue(), media_type="image/png")
```

**Correct (offload blocking work to executor):**

```python
import asyncio
import io
from concurrent.futures import ThreadPoolExecutor
from PIL import Image

executor = ThreadPoolExecutor(max_workers=4)

def _resize_sync(contents: bytes) -> bytes:
    img = Image.open(io.BytesIO(contents))
    img = img.resize((200, 200))
    buffer = io.BytesIO()
    img.save(buffer, format="PNG")
    return buffer.getvalue()

@router.post("/resize")
async def resize_image(file: UploadFile):
    contents = await file.read()
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(executor, _resize_sync, contents)
    return Response(content=result, media_type="image/png")
```

Alternatively, use `def` (not `async def`) for endpoints that are entirely synchronous. FastAPI will run them in a threadpool automatically.

### 5.4 Use BackgroundTasks for Fire-and-Forget

**Impact: MEDIUM — Slow response times**

Use `BackgroundTasks` for operations that should not block the response, such as sending emails, writing audit logs, or triggering webhooks. The response is sent immediately while the task runs after.

**Incorrect (blocking on non-essential work):**

```python
@router.post("/orders", status_code=201)
async def create_order(payload: OrderCreate, service=Depends(get_order_service)):
    order = await service.create(payload)
    await send_confirmation_email(order.user_email, order.id)  # blocks 2-3s
    await notify_warehouse(order)  # blocks another 1s
    return order  # user waits 3-4s total
```

**Correct (non-essential work in background):**

```python
from fastapi import BackgroundTasks

@router.post("/orders", status_code=201)
async def create_order(
    payload: OrderCreate,
    background_tasks: BackgroundTasks,
    service: OrderService = Depends(get_order_service),
) -> OrderResponse:
    order = await service.create(payload)
    background_tasks.add_task(send_confirmation_email, order.user_email, order.id)
    background_tasks.add_task(notify_warehouse, order)
    return order  # responds immediately
```

For long-running or critical tasks that must survive process restarts, use a proper task queue like Celery or arq instead.

---

## 6. Security & Validation

### 6.1 Implement Auth as Reusable Dependency

**Impact: HIGH — Inconsistent auth checks**

Implement authentication as a reusable `Depends()` callable. Never check auth inline in endpoints. A shared auth dependency ensures every protected endpoint uses the same validation logic.

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

@router.get("/orders")  # duplicated auth logic or forgotten entirely
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

### 6.2 Apply Rate Limiting

**Impact: MEDIUM — Denial of service risk**

Apply rate limiting via middleware or dependency to protect endpoints from abuse. Without rate limits, a single client can exhaust server resources with rapid requests.

**Incorrect (no rate limiting):**

```python
@router.post("/auth/login")
async def login(credentials: LoginRequest):
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

For production, use Redis-backed rate limiting to share state across workers.

### 6.3 Configure CORS with Explicit Origins

**Impact: MEDIUM — Cross-origin vulnerability**

Configure CORS with explicit allowed origins; never use wildcard `"*"` in production. A wildcard allows any website to make requests to your API, enabling CSRF-like attacks and data exfiltration.

**Incorrect (wildcard origin in production):**

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],        # any site can call your API
    allow_credentials=True,     # dangerous with wildcard
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Correct (explicit origins per environment):**

```python
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings

ALLOWED_ORIGINS: dict[str, list[str]] = {
    "production": [
        "https://app.example.com",
        "https://admin.example.com",
    ],
    "staging": [
        "https://staging.example.com",
    ],
    "development": [
        "http://localhost:3000",
        "http://localhost:5173",
    ],
}

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS[settings.environment],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=600,
)
```

Note that `allow_credentials=True` is incompatible with `allow_origins=["*"]` per the CORS specification.

---

## 7. Performance

### 7.1 Configure Connection Pool Size

**Impact: MEDIUM — Connection exhaustion**

Configure the async connection pool size to match your worker count and expected concurrency. An undersized pool causes requests to queue waiting for a connection. An oversized pool wastes database resources.

**Incorrect (default pool size, no tuning):**

```python
from sqlalchemy.ext.asyncio import create_async_engine

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

Sizing guideline: Set `pool_size` per worker to `(max_concurrent_requests_per_worker / avg_queries_per_request)`. Verify total connections across all workers are within your database's `max_connections` setting.

### 7.2 Use StreamingResponse for Large Payloads

**Impact: LOW — High memory usage**

Use `StreamingResponse` for large payloads or file downloads instead of loading the entire content into memory. A 500MB file returned as a regular response consumes 500MB of server memory per concurrent request.

**Incorrect (loading entire file into memory):**

```python
@router.get("/reports/{report_id}/download")
async def download_report(report_id: int):
    file_path = f"/data/reports/{report_id}.csv"
    with open(file_path, "rb") as f:
        content = f.read()  # entire file in memory
    return Response(
        content=content,
        media_type="text/csv",
        headers={"Content-Disposition": f"attachment; filename={report_id}.csv"},
    )
```

**Correct (streaming response):**

```python
from fastapi.responses import StreamingResponse
import aiofiles

async def file_streamer(file_path: str):
    async with aiofiles.open(file_path, "rb") as f:
        while chunk := await f.read(64 * 1024):  # 64KB chunks
            yield chunk

@router.get("/reports/{report_id}/download")
async def download_report(report_id: int):
    file_path = f"/data/reports/{report_id}.csv"
    return StreamingResponse(
        file_streamer(file_path),
        media_type="text/csv",
        headers={
            "Content-Disposition": f"attachment; filename={report_id}.csv"
        },
    )
```

Streaming also works well for server-sent events and large JSON arrays where you want to start sending data before the full result set is computed.
