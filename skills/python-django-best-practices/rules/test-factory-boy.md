---
title: Use factory_boy for Test Data
impact: MEDIUM
impactDescription: Maintainable test fixtures
tags: testing, factory-boy, fixtures, test-data, DRY
---

## Use factory_boy for Test Data; Never Raw Model.objects.create() in Tests

Scattering `Model.objects.create()` across tests creates brittle setups that
break when model fields change. `factory_boy` centralizes default values,
supports traits and sub-factories, and makes test intent clear by only
specifying fields relevant to the test case.

**Incorrect (raw create calls with redundant data):**

```python
# tests/test_orders.py
class TestOrderCreation:
    def test_discount_applied(self):
        user = User.objects.create(
            email="test@example.com",
            password="secret123",
            first_name="Test",
            last_name="User",
            is_active=True,
        )
        product = Product.objects.create(
            name="Widget", price=Decimal("100.00"),
            sku="WDG-001", stock=50, is_active=True,
        )
        order = Order.objects.create(user=user, product=product, quantity=5)
        # if User model adds a required field, ALL tests break
```

**Correct (factory_boy with minimal overrides):**

```python
# tests/factories.py
import factory
from factory.django import DjangoModelFactory

class UserFactory(DjangoModelFactory):
    class Meta:
        model = User

    email = factory.Sequence(lambda n: f"user{n}@example.com")
    first_name = factory.Faker("first_name")
    last_name = factory.Faker("last_name")
    is_active = True

class ProductFactory(DjangoModelFactory):
    class Meta:
        model = Product

    name = factory.Faker("word")
    price = Decimal("100.00")
    sku = factory.Sequence(lambda n: f"SKU-{n:04d}")
    stock = 50

class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    user = factory.SubFactory(UserFactory)
    product = factory.SubFactory(ProductFactory)
    quantity = 1

# tests/test_orders.py
class TestOrderCreation:
    def test_discount_applied(self):
        order = OrderFactory(quantity=5)
        # Only the relevant field is specified
```

Use `factory.Trait` for named variations (e.g., `OrderFactory(vip=True)`)
and `factory.LazyAttribute` for computed defaults.

Reference: https://factoryboy.readthedocs.io/en/stable/
