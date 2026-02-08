---
title: Use django-filter with Typed FilterSet Classes
impact: MEDIUM
impactDescription: Safe declarative filtering
tags: drf, filtering, django-filter, filterset, query-params
---

## Use django-filter with Typed FilterSet Classes

Hand-parsing query parameters in `get_queryset()` is error-prone and leads to
SQL injection risks or unvalidated input. `django-filter` provides declarative
`FilterSet` classes that validate, type-cast, and apply filters safely.

**Incorrect (manual query parameter parsing):**

```python
# views.py
class ProductViewSet(ModelViewSet):
    serializer_class = ProductSerializer

    def get_queryset(self):
        qs = Product.objects.all()
        if "min_price" in self.request.query_params:
            qs = qs.filter(price__gte=self.request.query_params["min_price"])
        if "category" in self.request.query_params:
            qs = qs.filter(category__slug=self.request.query_params["category"])
        if "in_stock" in self.request.query_params:
            qs = qs.filter(stock__gt=0)  # ignores actual param value
        return qs
```

**Correct (django-filter with typed FilterSet):**

```python
# filters.py
import django_filters
from .models import Product

class ProductFilterSet(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(field_name="price", lookup_expr="gte")
    max_price = django_filters.NumberFilter(field_name="price", lookup_expr="lte")
    category = django_filters.CharFilter(field_name="category__slug")
    in_stock = django_filters.BooleanFilter(method="filter_in_stock")

    class Meta:
        model = Product
        fields = ["min_price", "max_price", "category", "in_stock"]

    def filter_in_stock(self, queryset, name, value):
        if value:
            return queryset.filter(stock__gt=0)
        return queryset

# views.py
class ProductViewSet(ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filterset_class = ProductFilterSet
```

`FilterSet` classes are auto-documented by `drf-spectacular` and can be
unit-tested independently from views.

Reference: https://django-filter.readthedocs.io/en/stable/
