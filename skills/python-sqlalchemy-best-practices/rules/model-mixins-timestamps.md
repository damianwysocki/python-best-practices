---
title: Use Mixins for Common Columns
impact: HIGH
impactDescription: Eliminates column duplication
tags: model, mixin, timestamps, soft-delete, uuid
---

## Use Mixins for Common Columns

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

Reference: https://docs.sqlalchemy.org/en/20/orm/declarative_mixins.html
