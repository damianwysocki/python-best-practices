---
title: Use Abstract Base Models for Shared Fields
impact: CRITICAL
impactDescription: Eliminates field duplication
tags: models, abstract, base-model, DRY, timestamps
---

## Use Abstract Base Models for Shared Fields

Fields like `created_at`, `updated_at`, and `uuid` appear on nearly every model.
Duplicating them across models leads to inconsistent naming, missing auto-now
flags, and forgotten indexes. Define an abstract base model and inherit from it
everywhere.

**Incorrect (duplicated timestamp fields):**

```python
# models.py
class Order(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4)
    created = models.DateTimeField(auto_now_add=True)
    modified = models.DateTimeField(auto_now=True)
    # ...

class Invoice(models.Model):
    uuid = models.UUIDField(primary_key=True, default=uuid.uuid4)
    created_at = models.DateTimeField(auto_now_add=True)
    # missing updated_at entirely
    # ...
```

**Correct (abstract base model):**

```python
# core/models.py
import uuid as _uuid
from django.db import models

class TimestampedModel(models.Model):
    id = models.UUIDField(primary_key=True, default=_uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
        ordering = ["-created_at"]

# orders/models.py
class Order(TimestampedModel):
    customer = models.ForeignKey("users.User", on_delete=models.PROTECT)
    total = models.DecimalField(max_digits=10, decimal_places=2)

# invoices/models.py
class Invoice(TimestampedModel):
    order = models.OneToOneField(Order, on_delete=models.CASCADE)
    paid_at = models.DateTimeField(null=True, blank=True)
```

Set `abstract = True` in the Meta class so Django does not create a database
table for the base model. This pattern also works for soft-delete mixins,
slug mixins, and audit-trail fields.

Reference: https://docs.djangoproject.com/en/5.0/topics/db/models/#abstract-base-classes
