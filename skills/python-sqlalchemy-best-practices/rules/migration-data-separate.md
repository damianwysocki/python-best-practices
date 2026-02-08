---
title: Separate Schema Migrations from Data Migrations
impact: MEDIUM
impactDescription: Reduces migration failure risk
tags: migration, alembic, data-migration, schema, separation
---

## Separate Schema Migrations from Data Migrations

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

Reference: https://alembic.sqlalchemy.org/en/latest/ops.html
