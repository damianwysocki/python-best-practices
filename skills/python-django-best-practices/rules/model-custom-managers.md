---
title: Use Custom Managers for Reusable Query Encapsulation
impact: HIGH
impactDescription: Encapsulates query logic
tags: models, managers, querysets, DRY, encapsulation
---

## Use Custom Managers for Reusable Query Encapsulation

Repeating `.filter(is_active=True, deleted_at__isnull=True)` across views,
services, and tasks is fragile. A single change to the "active" definition
requires updating every call site. Custom managers and querysets encapsulate
these conditions in one place and chain naturally with the ORM.

**Incorrect (scattered filter logic):**

```python
# views.py
products = Product.objects.filter(is_active=True, stock__gt=0)

# services/cart_service.py
available = Product.objects.filter(is_active=True, stock__gt=0, price__lte=budget)

# tasks.py
stale = Product.objects.filter(is_active=True, stock__gt=0, updated_at__lt=cutoff)
```

**Correct (custom QuerySet and Manager):**

```python
# models.py
class ProductQuerySet(models.QuerySet):
    def available(self):
        return self.filter(is_active=True, stock__gt=0)

    def affordable(self, budget: Decimal):
        return self.filter(price__lte=budget)

    def stale(self, cutoff: datetime):
        return self.filter(updated_at__lt=cutoff)

class Product(TimestampedModel):
    name = models.CharField(max_length=255)
    is_active = models.BooleanField(default=True)
    stock = models.PositiveIntegerField(default=0)
    price = models.DecimalField(max_digits=10, decimal_places=2)

    objects = ProductQuerySet.as_manager()

# Usage — composable and readable:
Product.objects.available()
Product.objects.available().affordable(budget)
Product.objects.available().stale(cutoff)
```

Use `QuerySet.as_manager()` for chainable methods. Reserve a custom `Manager`
subclass only when you need to override `get_queryset()` (e.g., soft-delete).

Reference: https://docs.djangoproject.com/en/5.0/topics/db/managers/#custom-managers
