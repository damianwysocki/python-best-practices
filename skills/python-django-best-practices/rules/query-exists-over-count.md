---
title: Use .exists() Instead of .count() > 0
impact: MEDIUM
impactDescription: Faster existence checks
tags: queries, exists, count, optimization, performance
---

## Use .exists() Instead of .count() > 0 for Existence Checks

`.count()` forces the database to scan and aggregate all matching rows.
`.exists()` adds `LIMIT 1` to the query, returning as soon as a single
row is found. For large tables or expensive filters, this difference is
significant.

**Incorrect (using count for existence checks):**

```python
# services/order_service.py
def can_user_checkout(user: User) -> bool:
    if CartItem.objects.filter(user=user).count() > 0:
        return True
    return False

def has_pending_reviews(product: Product) -> bool:
    return Review.objects.filter(
        product=product, status="pending",
    ).count() != 0
```

**Correct (using .exists()):**

```python
# services/order_service.py
def can_user_checkout(user: User) -> bool:
    return CartItem.objects.filter(user=user).exists()

def has_pending_reviews(product: Product) -> bool:
    return Review.objects.filter(
        product=product, status="pending",
    ).exists()
```

Use `.count()` only when you actually need the number. For boolean checks,
always prefer `.exists()`. The same applies to template code: use
`{% if queryset.exists %}` instead of `{% if queryset.count %}`.

Also avoid `len(queryset)` for counting, as it loads all rows into Python
memory just to count them.

Reference: https://docs.djangoproject.com/en/5.0/ref/models/querysets/#exists
