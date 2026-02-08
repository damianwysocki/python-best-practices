---
title: Use bulk_create/bulk_update for Batch Operations
impact: HIGH
impactDescription: Eliminates per-row queries
tags: queries, bulk, batch, performance, create, update
---

## Use bulk_create/bulk_update for Batch Operations, Never Loops

Creating or updating objects one at a time inside a loop generates N INSERT
or UPDATE statements. `bulk_create` and `bulk_update` batch these into a
single query (or a few chunked queries), reducing round-trips by orders
of magnitude.

**Incorrect (one query per row in a loop):**

```python
# services/import_service.py
def import_products(rows: list[dict]) -> list[Product]:
    products = []
    for row in rows:  # 10,000 rows = 10,000 INSERT statements
        product = Product.objects.create(
            sku=row["sku"],
            name=row["name"],
            price=row["price"],
        )
        products.append(product)
    return products
```

**Correct (bulk_create with chunking):**

```python
# services/import_service.py
def import_products(rows: list[dict]) -> list[Product]:
    products = [
        Product(sku=row["sku"], name=row["name"], price=row["price"])
        for row in rows
    ]
    return Product.objects.bulk_create(products, batch_size=1000)

def update_prices(updates: list[tuple[int, Decimal]]) -> int:
    products = Product.objects.filter(id__in=[u[0] for u in updates])
    product_map = {p.id: p for p in products}
    for product_id, new_price in updates:
        product_map[product_id].price = new_price
    Product.objects.bulk_update(
        product_map.values(), ["price"], batch_size=1000,
    )
    return len(product_map)
```

Set `batch_size` to avoid exceeding database parameter limits (typically
around 1000 for PostgreSQL). Note that `bulk_create` does not call `save()`
or fire signals by default.

Reference: https://docs.djangoproject.com/en/5.0/ref/models/querysets/#bulk-create
