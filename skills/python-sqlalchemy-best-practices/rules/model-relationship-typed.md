---
title: Type All Relationships with Mapped Annotations
impact: HIGH
impactDescription: Catches relationship errors statically
tags: model, relationship, typing, mapped, foreign-key
---

## Type All Relationships with Mapped Annotations

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

Reference: https://docs.sqlalchemy.org/en/20/orm/relationships.html
