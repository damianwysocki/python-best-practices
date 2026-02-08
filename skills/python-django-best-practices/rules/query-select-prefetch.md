---
title: Always Use select_related/prefetch_related for Accessed Relations
impact: CRITICAL
impactDescription: Eliminates lazy-load queries
tags: queries, select-related, prefetch-related, optimization, performance
---

## Always Use select_related/prefetch_related for Accessed Relations

Accessing a ForeignKey or ManyToMany field on a queryset without
`select_related` or `prefetch_related` triggers a separate SQL query for
every object. On a list of 100 items with 2 relations each, this produces
201 queries instead of 3.

**Incorrect (lazy-loading relations in a loop):**

```python
# views.py
class OrderListView(ListAPIView):
    serializer_class = OrderSerializer
    queryset = Order.objects.all()

# serializers.py
class OrderSerializer(serializers.ModelSerializer):
    customer_name = serializers.CharField(source="customer.full_name")
    items = OrderItemSerializer(many=True)

    class Meta:
        model = Order
        fields = ["id", "customer_name", "items", "total"]
    # 1 query for orders + N queries for customer + N queries for items
```

**Correct (eager-loading with select/prefetch):**

```python
# views.py
class OrderListView(ListAPIView):
    serializer_class = OrderSerializer
    queryset = (
        Order.objects
        .select_related("customer")
        .prefetch_related("items")
    )

# serializers.py — same as before, but now only 3 queries total
class OrderSerializer(serializers.ModelSerializer):
    customer_name = serializers.CharField(source="customer.full_name")
    items = OrderItemSerializer(many=True)

    class Meta:
        model = Order
        fields = ["id", "customer_name", "items", "total"]
```

Use `select_related` for ForeignKey / OneToOneField (SQL JOIN).
Use `prefetch_related` for ManyToMany / reverse ForeignKey (separate query).

Reference: https://docs.djangoproject.com/en/5.0/ref/models/querysets/#select-related
