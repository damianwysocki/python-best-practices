---
title: Configure Eager Loading Explicitly Per Query
impact: HIGH
impactDescription: Eliminates N+1 query problems
tags: query, eager-loading, joinedload, selectinload, n-plus-one
---

## Configure Eager Loading Explicitly Per Query

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

Reference: https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html
