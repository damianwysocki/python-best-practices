---
title: Set Metadata Naming Convention for Consistent Constraint Names
impact: LOW
impactDescription: Simplifies migration management
tags: engine, naming-convention, metadata, constraints, alembic
---

## Set Metadata Naming Convention for Consistent Constraint Names

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

Reference: https://docs.sqlalchemy.org/en/20/core/constraints.html#configuring-constraint-naming-conventions
