---
title: Use Mapped[type] and mapped_column() for All Columns
impact: CRITICAL
impactDescription: Ensures static type safety
tags: model, mapped-column, typing, columns
---

## Use Mapped[type] and mapped_column() for All Columns

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

This removes ambiguity and makes the schema self-documenting at the Python level.

Reference: https://docs.sqlalchemy.org/en/20/orm/mapped_attributes.html
