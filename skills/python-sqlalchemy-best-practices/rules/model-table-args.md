---
title: Define Indexes and Constraints in __table_args__
impact: MEDIUM
impactDescription: Enforces data integrity at DB level
tags: model, table-args, index, constraint, naming
---

## Define Indexes and Constraints in __table_args__

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
vary across databases and make migrations harder to manage. Use a consistent
naming convention via `MetaData(naming_convention={...})`.

Reference: https://docs.sqlalchemy.org/en/20/orm/declarative_tables.html#orm-declarative-table-configuration
