---
title: Use DeclarativeBase with Type-Annotated Mapped Columns
impact: CRITICAL
impactDescription: Enables modern type-safe models
tags: model, declarative-base, sqlalchemy-2.0, typing
---

## Use DeclarativeBase with Type-Annotated Mapped Columns

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

The `DeclarativeBase` class also supports the `registry` pattern for
advanced use cases like mapping to existing tables or using imperative
mapping alongside declarative models.

Reference: https://docs.sqlalchemy.org/en/20/orm/declarative_styles.html
