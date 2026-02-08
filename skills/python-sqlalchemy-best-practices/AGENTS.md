# Python SQLAlchemy Best Practices

**Version 1.0.0** | Python / SQLAlchemy | February 2026

> Note: These rules target SQLAlchemy 2.0+ with Python 3.10+ type annotations.
> All code examples use the modern `select()` API, `DeclarativeBase`, and
> `Mapped` type annotations. Legacy patterns (`session.query()`, `Column()`,
> `declarative_base()`) are shown only in "Incorrect" examples.

## Abstract

This document defines 19 best-practice rules for SQLAlchemy 2.0+ applications,
organized into five categories: Session Management, Model Design, Query Patterns,
Migration Patterns, and Connection & Engine. Each rule includes an impact rating,
an explanation of the problem, and incorrect/correct code examples. Following these
rules produces type-safe, performant, and maintainable database code.

## Table of Contents

1. [Session Management](#session-management)
   - 1.1 [Always Use Context Manager for Session Lifecycle](#11-always-use-context-manager-for-session-lifecycle)
   - 1.2 [Never Use Global or Module-Level Sessions](#12-never-use-global-or-module-level-sessions)
   - 1.3 [One Session Equals One Unit of Work](#13-one-session-equals-one-unit-of-work)
   - 1.4 [Use Async Engine and AsyncSession for Async Applications](#14-use-async-engine-and-asyncsession-for-async-applications)
2. [Model Design](#model-design)
   - 2.1 [Use DeclarativeBase with Type-Annotated Mapped Columns](#21-use-declarativebase-with-type-annotated-mapped-columns)
   - 2.2 [Use Mapped[type] and mapped_column() for All Columns](#22-use-mappedtype-and-mapped_column-for-all-columns)
   - 2.3 [Type All Relationships with Mapped Annotations](#23-type-all-relationships-with-mapped-annotations)
   - 2.4 [Use Mixins for Common Columns](#24-use-mixins-for-common-columns)
   - 2.5 [Define Indexes and Constraints in __table_args__](#25-define-indexes-and-constraints-in-__table_args__)
3. [Query Patterns](#query-patterns)
   - 3.1 [Use select() Statement Style (2.0 API)](#31-use-select-statement-style-20-api)
   - 3.2 [Configure Eager Loading Explicitly Per Query](#32-configure-eager-loading-explicitly-per-query)
   - 3.3 [Encapsulate Queries in Typed Repository Classes](#33-encapsulate-queries-in-typed-repository-classes)
   - 3.4 [Type Query Results Properly](#34-type-query-results-properly)
   - 3.5 [Disable Autoflush and Flush Explicitly](#35-disable-autoflush-and-flush-explicitly)
4. [Migration Patterns](#migration-patterns)
   - 4.1 [Use Alembic with Autogenerate and Review Every Migration](#41-use-alembic-with-autogenerate-and-review-every-migration)
   - 4.2 [Always Implement downgrade() Properly](#42-always-implement-downgrade-properly)
   - 4.3 [Separate Schema Migrations from Data Migrations](#43-separate-schema-migrations-from-data-migrations)
5. [Connection & Engine](#connection--engine)
   - 5.1 [Configure Connection Pool for Production](#51-configure-connection-pool-for-production)
   - 5.2 [Set Metadata Naming Convention for Consistent Constraint Names](#52-set-metadata-naming-convention-for-consistent-constraint-names)

---

## Session Management

### 1.1 Always Use Context Manager for Session Lifecycle

**Impact: CRITICAL -- Prevents session leaks**

SQLAlchemy sessions hold database connections and transactional state. Failing to
properly close sessions leads to connection pool exhaustion, leaked transactions,
and data corruption. Always use `with Session(engine) as session:` or a
`sessionmaker` combined with a context manager to guarantee cleanup on both
success and failure paths.

**Incorrect (manual session without cleanup):**

```python
from sqlalchemy.orm import Session

def create_user(engine, name: str) -> User:
    session = Session(engine)
    user = User(name=name)
    session.add(user)
    session.commit()  # If this raises, session is never closed
    return user
```

**Correct (context manager guarantees cleanup):**

```python
from sqlalchemy.orm import Session

def create_user(engine, name: str) -> User:
    with Session(engine) as session:
        user = User(name=name)
        session.add(user)
        session.commit()
        return user
```

For web frameworks, use a dependency that yields the session:

```python
from collections.abc import Generator
from sqlalchemy.orm import Session, sessionmaker

SessionLocal = sessionmaker(bind=engine)

def get_session() -> Generator[Session, None, None]:
    with SessionLocal() as session:
        yield session
```

The context manager calls `session.close()` automatically when the block exits,
whether by normal return or exception. This returns the underlying connection
back to the pool and rolls back any uncommitted transaction.

---

### 1.2 Never Use Global or Module-Level Sessions

**Impact: CRITICAL -- Prevents thread-safety bugs**

A global or module-level session is shared across threads and requests, leading
to race conditions, stale data, and transaction cross-contamination. Sessions are
not thread-safe. Always create sessions per-request or per-unit-of-work and
inject them via dependency injection.

**Incorrect (global module-level session):**

```python
from sqlalchemy.orm import Session

engine = create_engine("postgresql://localhost/mydb")
db = Session(engine)  # Global session shared everywhere

class UserService:
    def get_user(self, user_id: int) -> User | None:
        return db.get(User, user_id)  # Thread-unsafe

    def create_user(self, name: str) -> User:
        user = User(name=name)
        db.add(user)
        db.commit()  # Commits other threads' pending work too
        return user
```

**Correct (inject session via dependency injection):**

```python
from sqlalchemy.orm import Session

class UserService:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get_user(self, user_id: int) -> User | None:
        return self._session.get(User, user_id)

    def create_user(self, name: str) -> User:
        user = User(name=name)
        self._session.add(user)
        self._session.commit()
        return user

# Usage: create session per request/unit-of-work
with Session(engine) as session:
    service = UserService(session)
    user = service.create_user("Alice")
```

This pattern makes testing trivial because you can inject a test session backed
by an in-memory SQLite database or a rolled-back transaction.

---

### 1.3 One Session Equals One Unit of Work

**Impact: HIGH -- Ensures transactional integrity**

A session should represent a single logical operation. Commit once at the end
of the operation. Multiple commits within one session break atomicity -- if the
second commit fails, the first is already persisted, leaving data in an
inconsistent state.

**Incorrect (multiple commits in one session):**

```python
def transfer_funds(session: Session, from_id: int, to_id: int, amount: Decimal) -> None:
    sender = session.get(Account, from_id)
    sender.balance -= amount
    session.commit()  # Persisted even if next step fails

    receiver = session.get(Account, to_id)
    receiver.balance += amount
    session.commit()  # If this fails, money vanished
```

**Correct (single commit at end of unit of work):**

```python
def transfer_funds(session: Session, from_id: int, to_id: int, amount: Decimal) -> None:
    sender = session.get(Account, from_id)
    receiver = session.get(Account, to_id)

    if sender is None or receiver is None:
        raise ValueError("Account not found")

    sender.balance -= amount
    receiver.balance += amount

    session.commit()  # Atomic: both changes persist or neither does
```

If you need to perform multiple independent operations, use separate sessions
for each. The session's identity map tracks all loaded objects, so a long-lived
session with many commits accumulates stale state.

---

### 1.4 Use Async Engine and AsyncSession for Async Applications

**Impact: HIGH -- Prevents event-loop blocking**

In async applications (FastAPI, Starlette, aiohttp), using synchronous
`create_engine` and `Session` blocks the event loop on every database call.
Use `create_async_engine` with `AsyncSession` to keep the event loop free and
maintain concurrency.

**Incorrect (synchronous engine in async application):**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

engine = create_engine("postgresql://localhost/mydb")

async def get_user(user_id: int) -> User | None:
    with Session(engine) as session:  # Blocks the event loop
        return session.get(User, user_id)
```

**Correct (async engine with AsyncSession):**

```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.ext.asyncio import async_sessionmaker

engine = create_async_engine("postgresql+asyncpg://localhost/mydb")
AsyncSessionLocal = async_sessionmaker(engine, class_=AsyncSession)

async def get_user(user_id: int) -> User | None:
    async with AsyncSessionLocal() as session:
        return await session.get(User, user_id)
```

For FastAPI dependency injection:

```python
from collections.abc import AsyncGenerator

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

Note that `AsyncSession` requires an async-compatible driver such as `asyncpg`
for PostgreSQL or `aiosqlite` for SQLite. The connection URL scheme must
include the async driver (e.g., `postgresql+asyncpg://`).

---

## Model Design

### 2.1 Use DeclarativeBase with Type-Annotated Mapped Columns

**Impact: CRITICAL -- Enables modern type-safe models**

SQLAlchemy 2.0 introduced `DeclarativeBase` to replace the legacy
`declarative_base()` function. The new style integrates with Python type
annotations, enabling static type checkers (mypy, pyright) to validate
column types, relationship types, and query results.

**Incorrect (legacy declarative_base):**

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String(100), nullable=False)
    email = Column(String(255), nullable=False, unique=True)
```

**Correct (DeclarativeBase with type annotations):**

```python
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True)
```

With `Mapped[str]`, the column is automatically `NOT NULL`. Use
`Mapped[str | None]` to make a column nullable. This eliminates an entire
class of bugs where `nullable` was accidentally omitted.

---

### 2.2 Use Mapped[type] and mapped_column() for All Columns

**Impact: CRITICAL -- Ensures static type safety**

Every column on a model must use the `Mapped[T]` annotation paired with
`mapped_column()`. This gives type checkers full visibility into column types,
nullability, and defaults. The legacy `Column()` API does not participate in
static analysis, making it easy to misuse nullable columns or wrong types.

**Incorrect (legacy Column without type annotations):**

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime

class Product(Base):
    __tablename__ = "products"
    id = Column(Integer, primary_key=True)
    name = Column(String(200))          # Nullable by accident
    price = Column(Integer)              # Type not visible to mypy
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime)        # Nullable by accident
```

**Correct (Mapped + mapped_column for every column):**

```python
from datetime import datetime
from sqlalchemy import String, func
from sqlalchemy.orm import Mapped, mapped_column

class Product(Base):
    __tablename__ = "products"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    price: Mapped[int] = mapped_column()
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    deleted_at: Mapped[datetime | None] = mapped_column(default=None)
```

Key behaviors of `Mapped[T]`:
- `Mapped[str]` implies `NOT NULL` automatically
- `Mapped[str | None]` implies `nullable=True` automatically
- `Mapped[int]` with `mapped_column(primary_key=True)` infers `Integer`
- `Mapped[str]` without an explicit type requires `String(length)` for most DBs

---

### 2.3 Type All Relationships with Mapped Annotations

**Impact: HIGH -- Catches relationship errors statically**

Relationships must use `Mapped[list[Child]]` for one-to-many and
`Mapped[Parent | None]` for many-to-one. Without type annotations,
type checkers cannot verify attribute access on related objects, and
IDE autocompletion is lost.

**Incorrect (untyped relationships):**

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class Author(Base):
    __tablename__ = "authors"
    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    books = relationship("Book", back_populates="author")

class Book(Base):
    __tablename__ = "books"
    id = Column(Integer, primary_key=True)
    author_id = Column(Integer, ForeignKey("authors.id"))
    author = relationship("Author", back_populates="books")
```

**Correct (typed relationships with Mapped):**

```python
from sqlalchemy import ForeignKey, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

class Author(Base):
    __tablename__ = "authors"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    books: Mapped[list["Book"]] = relationship(back_populates="author")

class Book(Base):
    __tablename__ = "books"

    id: Mapped[int] = mapped_column(primary_key=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("authors.id"))
    author: Mapped["Author"] = relationship(back_populates="books")
```

Use `Mapped["Author | None"]` when the foreign key is nullable. Use
`Mapped[list["Book"]]` for collections (defaults to an empty list).
The string form `"Book"` avoids circular import issues.

---

### 2.4 Use Mixins for Common Columns

**Impact: HIGH -- Eliminates column duplication**

Columns like `created_at`, `updated_at`, `deleted_at`, and UUID primary keys
appear on nearly every model. Duplicating them violates DRY and introduces
inconsistencies (different column names, missing defaults). Define reusable
mixins that models inherit from.

**Incorrect (duplicated timestamp columns across models):**

```python
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime | None] = mapped_column(onupdate=func.now())

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(primary_key=True)
    total: Mapped[int] = mapped_column()
    created: Mapped[datetime] = mapped_column()  # Inconsistent name, no default
```

**Correct (mixins for shared columns):**

```python
from datetime import datetime
from uuid import UUID, uuid4
from sqlalchemy import func
from sqlalchemy.orm import Mapped, mapped_column

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime | None] = mapped_column(
        server_default=func.now(), onupdate=func.now()
    )

class SoftDeleteMixin:
    deleted_at: Mapped[datetime | None] = mapped_column(default=None)
    is_deleted: Mapped[bool] = mapped_column(default=False)

class UUIDPrimaryKeyMixin:
    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)

class User(UUIDPrimaryKeyMixin, TimestampMixin, Base):
    __tablename__ = "users"
    name: Mapped[str] = mapped_column(String(100))

class Order(UUIDPrimaryKeyMixin, TimestampMixin, SoftDeleteMixin, Base):
    __tablename__ = "orders"
    total: Mapped[int] = mapped_column()
```

Mixins are listed before `Base` in the MRO so their columns are recognized
by the declarative system. This ensures consistent naming and behavior.

---

### 2.5 Define Indexes and Constraints in __table_args__

**Impact: MEDIUM -- Enforces data integrity at DB level**

Composite indexes, unique constraints, and check constraints should be defined
in `__table_args__` rather than scattered across individual columns. This makes
the table's constraints visible in one place and supports multi-column indexes
that cannot be expressed on a single `mapped_column`.

**Incorrect (missing composite indexes, no constraints):**

```python
class OrderItem(Base):
    __tablename__ = "order_items"

    id: Mapped[int] = mapped_column(primary_key=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("orders.id"))
    product_id: Mapped[int] = mapped_column(ForeignKey("products.id"))
    quantity: Mapped[int] = mapped_column()
    # No composite unique, no check constraint on quantity
```

**Correct (constraints and indexes in __table_args__):**

```python
from sqlalchemy import CheckConstraint, Index, UniqueConstraint

class OrderItem(Base):
    __tablename__ = "order_items"
    __table_args__ = (
        UniqueConstraint("order_id", "product_id", name="uq_order_product"),
        CheckConstraint("quantity > 0", name="ck_order_item_quantity_positive"),
        Index("ix_order_items_order_id", "order_id"),
        Index("ix_order_items_product_id", "product_id"),
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    order_id: Mapped[int] = mapped_column(ForeignKey("orders.id"))
    product_id: Mapped[int] = mapped_column(ForeignKey("products.id"))
    quantity: Mapped[int] = mapped_column()
```

Always provide explicit `name` arguments for constraints. Auto-generated names
vary across databases and make migrations harder to manage.

---

## Query Patterns

### 3.1 Use select() Statement Style (2.0 API)

**Impact: CRITICAL -- Future-proof query pattern**

The legacy `session.query(Model)` API is soft-deprecated in SQLAlchemy 2.0.
Use `select()` statements executed via `session.execute()` instead. The new
API is composable, supports subqueries naturally, works identically in sync
and async contexts, and produces better type inference.

**Incorrect (legacy session.query API):**

```python
def get_active_users(session: Session) -> list[User]:
    return (
        session.query(User)
        .filter(User.is_active == True)
        .order_by(User.name)
        .all()
    )

def get_user_by_email(session: Session, email: str) -> User | None:
    return session.query(User).filter_by(email=email).first()
```

**Correct (select() with session.execute):**

```python
from sqlalchemy import select

def get_active_users(session: Session) -> Sequence[User]:
    stmt = select(User).where(User.is_active == True).order_by(User.name)
    return session.execute(stmt).scalars().all()

def get_user_by_email(session: Session, email: str) -> User | None:
    stmt = select(User).where(User.email == email)
    return session.execute(stmt).scalars().first()
```

The `select()` API is also required for `AsyncSession`:

```python
async def get_active_users(session: AsyncSession) -> Sequence[User]:
    stmt = select(User).where(User.is_active == True)
    result = await session.execute(stmt)
    return result.scalars().all()
```

The statement can be built incrementally and passed around, making it easy
to compose complex queries from reusable fragments.

---

### 3.2 Configure Eager Loading Explicitly Per Query

**Impact: HIGH -- Eliminates N+1 query problems**

By default, SQLAlchemy uses lazy loading for relationships, which triggers a
separate query for each related object access (the N+1 problem). Configure
eager loading strategies explicitly on each query based on what data is needed.

**Incorrect (relying on lazy loading, causing N+1):**

```python
def get_authors_with_books(session: Session) -> Sequence[Author]:
    stmt = select(Author)
    authors = session.execute(stmt).scalars().all()
    for author in authors:
        print(author.books)  # Each access fires a separate SELECT
    return authors
```

**Correct (explicit eager loading per query):**

```python
from sqlalchemy.orm import joinedload, selectinload

def get_authors_with_books(session: Session) -> Sequence[Author]:
    stmt = select(Author).options(selectinload(Author.books))
    return session.execute(stmt).scalars().all()

def get_order_with_items_and_product(session: Session, order_id: int) -> Order | None:
    stmt = (
        select(Order)
        .where(Order.id == order_id)
        .options(
            joinedload(Order.customer),
            selectinload(Order.items).joinedload(OrderItem.product),
        )
    )
    return session.execute(stmt).scalars().first()
```

Choose the right strategy:
- `joinedload` -- best for many-to-one or one-to-one (single JOIN)
- `selectinload` -- best for one-to-many collections (separate IN query)
- `subqueryload` -- alternative for large collections with complex filters

Set `lazy="raise"` on the relationship to make accidental lazy loads an error
during development.

---

### 3.3 Encapsulate Queries in Typed Repository Classes

**Impact: HIGH -- Improves testability and reuse**

Scattering raw `select()` calls across service code makes queries hard to find,
test, and optimize. Encapsulate data access in Repository classes with a
`Protocol` interface so services depend on an abstraction, not SQLAlchemy
directly.

**Incorrect (queries scattered in business logic):**

```python
class OrderService:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get_pending_orders(self) -> Sequence[Order]:
        stmt = select(Order).where(Order.status == "pending")
        return self._session.execute(stmt).scalars().all()

    def cancel_expired(self) -> int:
        stmt = select(Order).where(
            Order.status == "pending",
            Order.created_at < datetime.now() - timedelta(days=7),
        )
        orders = self._session.execute(stmt).scalars().all()
        for order in orders:
            order.status = "cancelled"
        return len(orders)
```

**Correct (repository with Protocol interface):**

```python
from typing import Protocol

class OrderRepository(Protocol):
    def get_pending(self) -> Sequence[Order]: ...
    def get_expired(self, cutoff: datetime) -> Sequence[Order]: ...

class SqlAlchemyOrderRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get_pending(self) -> Sequence[Order]:
        stmt = select(Order).where(Order.status == "pending")
        return self._session.execute(stmt).scalars().all()

    def get_expired(self, cutoff: datetime) -> Sequence[Order]:
        stmt = select(Order).where(
            Order.status == "pending",
            Order.created_at < cutoff,
        )
        return self._session.execute(stmt).scalars().all()

class OrderService:
    def __init__(self, repo: OrderRepository) -> None:
        self._repo = repo

    def cancel_expired(self) -> int:
        cutoff = datetime.now() - timedelta(days=7)
        orders = self._repo.get_expired(cutoff)
        for order in orders:
            order.status = "cancelled"
        return len(orders)
```

The `Protocol` interface allows injecting a fake repository in tests without
touching the database at all.

---

### 3.4 Type Query Results Properly

**Impact: HIGH -- Enables static result validation**

Always annotate query return types with the correct container type. Use
`Sequence[Model]` for `.all()` results and `Model | None` for `.first()`
or `.one_or_none()`. This enables type checkers to catch errors when callers
assume a result is non-None or iterate over a scalar.

**Incorrect (untyped or incorrectly typed results):**

```python
def get_users(session: Session):  # No return type
    stmt = select(User)
    return session.execute(stmt).scalars().all()

def get_user(session: Session, user_id: int):  # No return type
    return session.get(User, user_id)

def get_emails(session: Session) -> list:  # Too broad
    stmt = select(User.email)
    return session.execute(stmt).scalars().all()
```

**Correct (properly typed query results):**

```python
from collections.abc import Sequence

def get_users(session: Session) -> Sequence[User]:
    stmt = select(User)
    return session.execute(stmt).scalars().all()

def get_user(session: Session, user_id: int) -> User | None:
    return session.get(User, user_id)

def get_user_strict(session: Session, user_id: int) -> User:
    stmt = select(User).where(User.id == user_id)
    result = session.execute(stmt).scalars().one()  # Raises if not found
    return result

def get_emails(session: Session) -> Sequence[str]:
    stmt = select(User.email)
    return session.execute(stmt).scalars().all()
```

Use `Sequence` from `collections.abc` instead of `list` because `.all()`
returns a `Sequence`, and `Sequence` is covariant, which is more compatible
with generic code. Use `.one()` when exactly one result is expected, and
`.one_or_none()` when zero or one is acceptable.

---

### 3.5 Disable Autoflush and Flush Explicitly

**Impact: MEDIUM -- Prevents surprising query results**

By default, SQLAlchemy autoflushes pending changes before every query. This can
cause unexpected `IntegrityError` exceptions during reads and makes debugging
difficult because the flush happens implicitly. Disable autoflush and call
`session.flush()` explicitly when needed.

**Incorrect (relying on autoflush, surprising errors on read):**

```python
SessionLocal = sessionmaker(bind=engine)  # autoflush=True by default

def add_and_check(session: Session) -> bool:
    user = User(name="Alice", email="invalid")  # Bad data
    session.add(user)

    # This SELECT triggers autoflush, raising IntegrityError unexpectedly
    stmt = select(User).where(User.name == "Bob")
    bob = session.execute(stmt).scalars().first()
    return bob is not None
```

**Correct (autoflush disabled, explicit flush):**

```python
SessionLocal = sessionmaker(bind=engine, autoflush=False)

def add_and_check(session: Session) -> bool:
    user = User(name="Alice", email="alice@example.com")
    session.add(user)

    # No autoflush: this SELECT runs cleanly
    stmt = select(User).where(User.name == "Bob")
    bob = session.execute(stmt).scalars().first()

    # Flush explicitly when you want pending changes sent to DB
    session.flush()
    return bob is not None
```

When autoflush is disabled, use `session.flush()` to send pending changes to
the database before any read that depends on those writes. This gives you
precise control over when SQL is emitted.

Note: `flush()` does not commit. It only writes changes to the database within
the current transaction. Call `session.commit()` to make them permanent.

---

## Migration Patterns

### 4.1 Use Alembic with Autogenerate and Review Every Migration

**Impact: MEDIUM -- Ensures safe schema evolution**

Alembic's `--autogenerate` compares your models against the database and
generates migration scripts. However, autogenerate does not detect everything
(renamed columns, data type changes on some backends, CHECK constraints).
Always review and edit the generated migration before applying it.

**Incorrect (applying autogenerated migration without review):**

```python
# alembic revision --autogenerate -m "add users"
# alembic upgrade head  # Applied blindly

# Generated migration might miss:
# - Column renames (shows as drop + add, losing data)
# - Server defaults
# - Partial indexes
```

**Correct (review and adjust autogenerated migration):**

```python
# Step 1: Generate
# alembic revision --autogenerate -m "add users table"

# Step 2: Review the generated file
def upgrade() -> None:
    op.create_table(
        "users",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("name", sa.String(100), nullable=False),
        sa.Column("email", sa.String(255), nullable=False),
        sa.Column("created_at", sa.DateTime(), server_default=sa.func.now()),
        sa.PrimaryKeyConstraint("id", name=op.f("pk_users")),
        sa.UniqueConstraint("email", name=op.f("uq_users_email")),
    )
    op.create_index(op.f("ix_users_email"), "users", ["email"])

def downgrade() -> None:
    op.drop_index(op.f("ix_users_email"), table_name="users")
    op.drop_table("users")
```

Configure `env.py` to import your `Base.metadata` so autogenerate can
compare models to the database. Set `compare_type=True` to detect column
type changes.

---

### 4.2 Always Implement downgrade() Properly

**Impact: MEDIUM -- Enables safe migration rollback**

Every Alembic migration must have a working `downgrade()` function that fully
reverses the `upgrade()`. Using `pass` or leaving it empty means you cannot
roll back a failed deployment, which turns a recoverable incident into an
emergency.

**Incorrect (empty or stub downgrade):**

```python
def upgrade() -> None:
    op.add_column("users", sa.Column("phone", sa.String(20), nullable=True))
    op.create_index("ix_users_phone", "users", ["phone"])

def downgrade() -> None:
    pass  # Cannot roll back -- deployment is now irreversible
```

**Correct (fully reversible downgrade):**

```python
def upgrade() -> None:
    op.add_column("users", sa.Column("phone", sa.String(20), nullable=True))
    op.create_index("ix_users_phone", "users", ["phone"])

def downgrade() -> None:
    op.drop_index("ix_users_phone", table_name="users")
    op.drop_column("users", "phone")
```

For destructive operations like dropping a column, the downgrade should
recreate it with the same type and constraints:

```python
def upgrade() -> None:
    op.drop_column("users", "legacy_field")

def downgrade() -> None:
    op.add_column(
        "users",
        sa.Column("legacy_field", sa.String(100), nullable=True),
    )
```

Test both `upgrade` and `downgrade` in CI by running the full migration chain
forward and backward: `alembic upgrade head && alembic downgrade base`.

---

### 4.3 Separate Schema Migrations from Data Migrations

**Impact: MEDIUM -- Reduces migration failure risk**

Mixing DDL (schema changes) and DML (data transformations) in the same migration
makes rollbacks unreliable and increases lock contention. Schema changes acquire
locks on tables; data migrations can run for minutes. Combining them holds locks
for the entire duration, blocking all queries.

**Incorrect (schema and data changes in one migration):**

```python
def upgrade() -> None:
    # Schema change
    op.add_column("users", sa.Column("full_name", sa.String(200)))

    # Data migration in the same step -- holds table lock
    connection = op.get_bind()
    connection.execute(
        sa.text("UPDATE users SET full_name = first_name || ' ' || last_name")
    )

    # More schema changes while data is being migrated
    op.drop_column("users", "first_name")
    op.drop_column("users", "last_name")
```

**Correct (three separate migrations):**

```python
# Migration 1: Add new column (schema only)
def upgrade() -> None:
    op.add_column("users", sa.Column("full_name", sa.String(200)))

def downgrade() -> None:
    op.drop_column("users", "full_name")

# Migration 2: Backfill data (data only)
def upgrade() -> None:
    connection = op.get_bind()
    connection.execute(
        sa.text("UPDATE users SET full_name = first_name || ' ' || last_name")
    )

def downgrade() -> None:
    connection = op.get_bind()
    connection.execute(sa.text("UPDATE users SET full_name = NULL"))

# Migration 3: Drop old columns (schema only, after deploy confirms success)
def upgrade() -> None:
    op.drop_column("users", "first_name")
    op.drop_column("users", "last_name")

def downgrade() -> None:
    op.add_column("users", sa.Column("last_name", sa.String(100)))
    op.add_column("users", sa.Column("first_name", sa.String(100)))
```

This three-phase approach (expand, migrate, contract) allows safe rollback at
each step and minimizes lock duration.

---

## Connection & Engine

### 5.1 Configure Connection Pool for Production

**Impact: MEDIUM -- Prevents connection exhaustion**

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

---

### 5.2 Set Metadata Naming Convention for Consistent Constraint Names

**Impact: LOW -- Simplifies migration management**

Without a naming convention, SQLAlchemy and the database generate constraint
names inconsistently (e.g., some databases auto-name them, others leave them
unnamed). This causes Alembic to generate non-deterministic migration files
and makes it impossible to drop constraints by name in downgrade scripts.

**Incorrect (no naming convention, unnamed constraints):**

```python
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass  # No naming convention

# Constraint names vary by database backend
# Alembic migrations contain None for constraint names
# downgrade() cannot drop constraints reliably
```

**Correct (metadata with naming convention):**

```python
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

convention = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}

class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=convention)
```

With this convention, all constraints get predictable, deterministic names:
- Primary key on `users` table: `pk_users`
- Foreign key `users.company_id` -> `companies`: `fk_users_company_id_companies`
- Unique constraint on `users.email`: `uq_users_email`
- Index on `users.email`: `ix_users_email`

This is especially important for SQLite, which does not support `ALTER`
operations to rename constraints, and for Alembic batch mode migrations.
