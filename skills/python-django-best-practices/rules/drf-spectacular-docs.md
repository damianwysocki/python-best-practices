---
title: Use drf-spectacular for OpenAPI 3.0 Documentation
impact: HIGH
impactDescription: Accurate auto-generated docs
tags: drf, openapi, documentation, drf-spectacular, swagger
---

## Use drf-spectacular for OpenAPI 3.0 with Descriptions and Examples

Hand-maintained API documentation drifts from the actual implementation.
`drf-spectacular` generates an OpenAPI 3.0 schema directly from your
serializers, views, and type hints. Annotate views with `@extend_schema`
to add descriptions, examples, and response codes.

**Incorrect (no schema annotations, relying on guesswork):**

```python
# views.py
class OrderViewSet(ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    # No documentation — consumers must read source code
```

**Correct (annotated with drf-spectacular):**

```python
# views.py
from drf_spectacular.utils import extend_schema, OpenApiExample

class OrderViewSet(ModelViewSet):
    queryset = Order.objects.all()

    @extend_schema(
        request=OrderCreateSerializer,
        responses={201: OrderDetailSerializer},
        description="Create a new order for the authenticated user.",
        examples=[
            OpenApiExample(
                "Basic order",
                value={"product_id": 1, "quantity": 2},
                request_only=True,
            ),
        ],
    )
    def create(self, request, *args, **kwargs):
        return super().create(request, *args, **kwargs)

    @extend_schema(
        responses={200: OrderListSerializer(many=True)},
        description="List orders for the authenticated user.",
    )
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

Add `drf-spectacular` to `INSTALLED_APPS` and configure
`DEFAULT_SCHEMA_CLASS` in settings to `AutoSchema`. Serve Swagger UI and
ReDoc at `/api/docs/` for interactive exploration.

Reference: https://drf-spectacular.readthedocs.io/en/latest/
