---
title: Always Implement downgrade() Properly
impact: MEDIUM
impactDescription: Enables safe migration rollback
tags: migration, alembic, downgrade, reversible, rollback
---

## Always Implement downgrade() Properly

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

Reference: https://alembic.sqlalchemy.org/en/latest/tutorial.html#running-our-second-migration
