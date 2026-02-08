---
title: Business Logic in Service Layer, Not in Views or Serializers
impact: CRITICAL
impactDescription: Prevents tangled responsibilities
tags: service-layer, architecture, views, separation-of-concerns
---

## Business Logic in Service Layer, Not in Views or Serializers

Views and serializers are HTTP-boundary components. Embedding business logic in them
couples domain rules to the web framework, making code untestable in isolation and
impossible to reuse from management commands, Celery tasks, or other entry points.
Extract all business logic into a dedicated service module.

**Incorrect (business logic in a view):**

```python
# views.py
class OrderCreateView(APIView):
    def post(self, request):
        serializer = OrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        order = Order.objects.create(**serializer.validated_data)
        order.total = sum(
            item.price * item.quantity for item in order.items.all()
        )
        if order.total > 500:
            order.discount = order.total * Decimal("0.1")
        order.total -= order.discount
        order.save()
        send_order_confirmation.delay(order.id)
        return Response(OrderSerializer(order).data, status=201)
```

**Correct (delegating to a service):**

```python
# services/order_service.py
from decimal import Decimal

def create_order(*, validated_data: dict, user) -> Order:
    order = Order.objects.create(**validated_data, created_by=user)
    order.total = sum(
        item.price * item.quantity for item in order.items.all()
    )
    if order.total > 500:
        order.discount = order.total * Decimal("0.1")
    order.total -= order.discount
    order.save()
    send_order_confirmation.delay(order.id)
    return order

# views.py
class OrderCreateView(APIView):
    def post(self, request):
        serializer = OrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        order = create_order(
            validated_data=serializer.validated_data,
            user=request.user,
        )
        return Response(OrderSerializer(order).data, status=201)
```

Services should be plain Python functions or classes with no dependency on
`request`, `Response`, or any DRF/Django HTTP machinery.

Reference: https://docs.djangoproject.com/en/5.0/misc/design-philosophies/
