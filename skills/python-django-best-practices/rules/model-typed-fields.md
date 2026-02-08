---
title: Use django-stubs with Explicit Type Annotations
impact: MEDIUM
impactDescription: Catches field type errors
tags: models, typing, django-stubs, mypy, type-safety
---

## Use django-stubs with Explicit Type Annotations on All Fields

Django's ORM uses descriptor magic that confuses type checkers. Without
`django-stubs`, mypy cannot verify that you are comparing a `DecimalField`
with a `Decimal` or passing the right type to `create()`. Install
`django-stubs` and annotate fields for full static analysis coverage.

**Incorrect (no type annotations, mypy is blind):**

```python
# models.py
class Product(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    is_active = models.BooleanField(default=True)

# service.py — mypy cannot catch this bug:
product.price = "not a number"  # no error reported
```

**Correct (django-stubs installed, explicit annotations):**

```python
# pyproject.toml
# [tool.mypy]
# plugins = ["mypy_django_plugin.main"]
#
# [tool.django-stubs]
# django_settings_module = "config.settings"

# models.py
from decimal import Decimal
from django.db import models

class Product(models.Model):
    name: models.CharField[str, str] = models.CharField(max_length=255)
    price: models.DecimalField[Decimal, Decimal] = models.DecimalField(
        max_digits=10, decimal_places=2,
    )
    is_active: models.BooleanField[bool, bool] = models.BooleanField(
        default=True,
    )

# service.py — mypy now catches this:
product.price = "not a number"  # error: Incompatible types
```

Run `mypy --strict` in CI to catch field-type mismatches, missing nullable
checks, and incorrect queryset usage before they reach production.

Reference: https://github.com/typeddjango/django-stubs
