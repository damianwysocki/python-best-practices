# Python Django Best Practices

**Version 1.0.0** | Django & Django REST Framework | February 2026

> **Note:** This document contains 23 rules organized across 5 categories for building
> maintainable, performant, and testable Django applications. Rules are ordered by
> priority within each category. Every rule includes incorrect and correct code examples.

## Abstract

This guide codifies best practices for Django and Django REST Framework projects. It
enforces a clean service-layer architecture where views handle HTTP, services handle
business logic, and models handle persistence. It covers model design patterns for
DRY schemas and database-level integrity, DRF patterns for secure and well-documented
APIs, query optimization techniques to eliminate N+1 problems and reduce database load,
and testing strategies that balance speed with coverage.

## Table of Contents

1. [Service Layer Integration](#1-service-layer-integration)
   - 1.1 Business Logic in Service Layer, Not in Views or Serializers
   - 1.2 Models Handle Persistence Schema Only, Never Business Logic
   - 1.3 Views/ViewSets Handle HTTP Concerns Only
   - 1.4 Use Forms/Serializers for Validation, Services for Logic
2. [Model Design](#2-model-design)
   - 2.1 Use Abstract Base Models for Shared Fields
   - 2.2 Always Set related_name on ForeignKey and ManyToMany
   - 2.3 Use Custom Managers for Reusable Query Encapsulation
   - 2.4 Use Meta.constraints for Database-Level Data Integrity
   - 2.5 Use django-stubs with Explicit Type Annotations
3. [DRF Patterns](#3-drf-patterns)
   - 3.1 Separate Serializers for Create/Update/List/Detail Actions
   - 3.2 Keep ViewSets Thin -- Override Only What Is Needed
   - 3.3 Use drf-spectacular for OpenAPI 3.0 Documentation
   - 3.4 Use CursorPagination or LimitOffsetPagination
   - 3.5 Fine-Grained Permission Classes Per View
   - 3.6 Use django-filter with Typed FilterSet Classes
4. [Query Optimization](#4-query-optimization)
   - 4.1 Always Use select_related/prefetch_related for Accessed Relations
   - 4.2 Detect and Prevent N+1 Queries
   - 4.3 Use .only()/.defer() to Load Only Needed Columns
   - 4.4 Use bulk_create/bulk_update for Batch Operations
   - 4.5 Use .exists() Instead of .count() > 0
5. [Testing](#5-testing)
   - 5.1 Use factory_boy for Test Data
   - 5.2 Unit Test Services with Mocked Dependencies
   - 5.3 Integration Test API Endpoints with APIClient

---

## 1. Service Layer Integration

### 1.1 Business Logic in Service Layer, Not in Views or Serializers

**Impact: CRITICAL -- Prevents tangled responsibilities**

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

Services should be plain Python functions or classes with no dependency on `request`,
`Response`, or any DRF/Django HTTP machinery.

---

### 1.2 Models Handle Persistence Schema Only, Never Business Logic

**Impact: CRITICAL -- Keeps models focused and testable**

Django models should define fields, relationships, indexes, constraints, and lightweight
computed properties (`@property`). Business workflows, side effects, and cross-model
orchestration belong in the service layer. Fat models become untestable monoliths that
mix persistence with domain rules.

**Incorrect (business logic inside a model):**

```python
# models.py
class Subscription(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    plan = models.ForeignKey(Plan, on_delete=models.PROTECT)
    status = models.CharField(max_length=20, default="active")
    expires_at = models.DateTimeField()

    def renew(self):
        if self.status != "active":
            raise ValueError("Cannot renew inactive subscription")
        charge = StripeGateway().charge(self.user, self.plan.price)
        if charge.success:
            self.expires_at += timedelta(days=30)
            self.save()
            send_renewal_email.delay(self.user.id)
        return charge
```

**Correct (model defines schema, service handles renewal):**

```python
# models.py
class Subscription(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    plan = models.ForeignKey(Plan, on_delete=models.PROTECT)
    status = models.CharField(max_length=20, default="active")
    expires_at = models.DateTimeField()

    @property
    def is_active(self) -> bool:
        return self.status == "active" and self.expires_at > timezone.now()

# services/subscription_service.py
def renew_subscription(*, subscription: Subscription, gateway: PaymentGateway) -> Charge:
    if not subscription.is_active:
        raise SubscriptionInactiveError(subscription.id)
    charge = gateway.charge(subscription.user, subscription.plan.price)
    if charge.success:
        subscription.expires_at += timedelta(days=30)
        subscription.save(update_fields=["expires_at"])
        send_renewal_email.delay(subscription.user_id)
    return charge
```

Model properties that are pure data derivations (e.g., `is_active`) are acceptable.
Anything that triggers side effects must live in a service.

---

### 1.3 Views/ViewSets Handle HTTP Concerns Only

**Impact: CRITICAL -- Clean HTTP boundary layer**

Views and ViewSets should be thin adapters between HTTP and your domain. Their
responsibilities are limited to: parsing request data, calling serializers for
validation, delegating to service functions, and returning HTTP responses. Any
conditional logic, orchestration, or state mutation belongs in the service layer.

**Incorrect (fat ViewSet with inline logic):**

```python
# views.py
class UserViewSet(ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

    def perform_create(self, serializer):
        user = serializer.save()
        profile = Profile.objects.create(user=user, tier="free")
        WelcomeEmail(user.email).send()
        analytics.track("user_signup", {"user_id": user.id})
        if user.referral_code:
            referrer = User.objects.get(referral_code=user.referral_code)
            Credit.objects.create(user=referrer, amount=10)
```

**Correct (thin ViewSet delegating to service):**

```python
# services/user_service.py
def register_user(*, validated_data: dict) -> User:
    user = User.objects.create_user(**validated_data)
    Profile.objects.create(user=user, tier="free")
    WelcomeEmail(user.email).send()
    analytics.track("user_signup", {"user_id": user.id})
    if user.referral_code:
        _grant_referral_credit(user.referral_code)
    return user

# views.py
class UserViewSet(ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

    def perform_create(self, serializer):
        register_user(validated_data=serializer.validated_data)
```

A good heuristic: if a view method exceeds 8-10 lines, extract the logic into a
service. Views should read like a table of contents, not a novel.

---

### 1.4 Use Forms/Serializers for Validation, Services for Logic

**Impact: HIGH -- Clear validation boundaries**

Serializers and forms own input validation (field types, required fields, uniqueness,
format). Services own business rules (authorization, state transitions, cross-entity
consistency). Mixing the two creates serializers that depend on database state or
services that re-validate field types.

**Incorrect (business validation in serializer):**

```python
# serializers.py
class TransferSerializer(serializers.Serializer):
    from_account = serializers.IntegerField()
    to_account = serializers.IntegerField()
    amount = serializers.DecimalField(max_digits=10, decimal_places=2)

    def validate(self, attrs):
        from_acc = Account.objects.get(id=attrs["from_account"])
        if from_acc.balance < attrs["amount"]:
            raise serializers.ValidationError("Insufficient funds")
        if from_acc.is_frozen:
            raise serializers.ValidationError("Account is frozen")
        return attrs
```

**Correct (serializer validates shape, service validates business rules):**

```python
# serializers.py
class TransferSerializer(serializers.Serializer):
    from_account = serializers.IntegerField()
    to_account = serializers.IntegerField()
    amount = serializers.DecimalField(
        max_digits=10, decimal_places=2, min_value=Decimal("0.01"),
    )

# services/transfer_service.py
class InsufficientFundsError(Exception):
    pass

def execute_transfer(*, from_account_id: int, to_account_id: int, amount: Decimal) -> Transfer:
    from_acc = Account.objects.select_for_update().get(id=from_account_id)
    if from_acc.is_frozen:
        raise AccountFrozenError(from_acc.id)
    if from_acc.balance < amount:
        raise InsufficientFundsError(from_acc.id, amount)
    to_acc = Account.objects.select_for_update().get(id=to_account_id)
    return _perform_transfer(from_acc, to_acc, amount)
```

This separation lets you reuse the same service from CLI commands, tasks, or other
services without going through serializer machinery.

---

## 2. Model Design

### 2.1 Use Abstract Base Models for Shared Fields

**Impact: CRITICAL -- Eliminates field duplication**

Fields like `created_at`, `updated_at`, and `uuid` appear on nearly every model.
Duplicating them across models leads to inconsistent naming, missing auto-now flags,
and forgotten indexes. Define an abstract base model and inherit from it everywhere.

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

Set `abstract = True` in the Meta class so Django does not create a database table for
the base model. This pattern also works for soft-delete mixins, slug mixins, and
audit-trail fields.

---

### 2.2 Always Set related_name on ForeignKey and ManyToMany

**Impact: HIGH -- Readable reverse relations**

Django auto-generates reverse relation names using `<model>_set`, which is ambiguous
when a model has multiple foreign keys to the same target. Explicit `related_name`
values make reverse lookups self-documenting and prevent clash errors in abstract models.

**Incorrect (missing related_name):**

```python
class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    author = models.ForeignKey(User, on_delete=models.CASCADE)

# Usage becomes unclear:
post.comment_set.all()  # implicit, non-descriptive
user.comment_set.all()  # same name for a different relationship
```

**Correct (explicit related_name):**

```python
class Comment(models.Model):
    post = models.ForeignKey(
        Post,
        on_delete=models.CASCADE,
        related_name="comments",
    )
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name="authored_comments",
    )

# Usage is self-documenting:
post.comments.all()
user.authored_comments.filter(is_approved=True)
```

For abstract base models, use `"%(class)s_%(app_label)s"` or
`"%(app_label)s_%(class)s_related"` to avoid reverse accessor clashes across apps.

---

### 2.3 Use Custom Managers for Reusable Query Encapsulation

**Impact: HIGH -- Encapsulates query logic**

Repeating `.filter(is_active=True, deleted_at__isnull=True)` across views, services,
and tasks is fragile. A single change to the "active" definition requires updating every
call site. Custom managers and querysets encapsulate these conditions in one place and
chain naturally with the ORM.

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

# Usage -- composable and readable:
Product.objects.available()
Product.objects.available().affordable(budget)
Product.objects.available().stale(cutoff)
```

Use `QuerySet.as_manager()` for chainable methods. Reserve a custom `Manager` subclass
only when you need to override `get_queryset()` (e.g., soft-delete).

---

### 2.4 Use Meta.constraints for Database-Level Data Integrity

**Impact: HIGH -- Guarantees data integrity**

Application-level validation can be bypassed by raw SQL, management commands, or bulk
operations. Database constraints are the last line of defense. Use `Meta.constraints`
with `CheckConstraint` and `UniqueConstraint` to enforce invariants at the database level.

**Incorrect (relying only on application validation):**

```python
# models.py
class Booking(models.Model):
    room = models.ForeignKey(Room, on_delete=models.CASCADE)
    check_in = models.DateField()
    check_out = models.DateField()
    guests = models.PositiveIntegerField()

    def clean(self):
        if self.check_out <= self.check_in:
            raise ValidationError("check_out must be after check_in")
        # This is never enforced at the DB level!
```

**Correct (database-level constraints):**

```python
# models.py
from django.db.models import Q, F, CheckConstraint, UniqueConstraint

class Booking(models.Model):
    room = models.ForeignKey(Room, on_delete=models.CASCADE, related_name="bookings")
    check_in = models.DateField()
    check_out = models.DateField()
    guests = models.PositiveIntegerField()

    class Meta:
        constraints = [
            CheckConstraint(
                check=Q(check_out__gt=F("check_in")),
                name="booking_checkout_after_checkin",
            ),
            CheckConstraint(
                check=Q(guests__gte=1, guests__lte=20),
                name="booking_guests_range",
            ),
            UniqueConstraint(
                fields=["room", "check_in"],
                name="booking_unique_room_date",
            ),
        ]
```

Name constraints with a `<table>_<description>` pattern so migration errors are easy
to trace. Combine with `clean()` for user-friendly error messages.

---

### 2.5 Use django-stubs with Explicit Type Annotations

**Impact: MEDIUM -- Catches field type errors**

Django's ORM uses descriptor magic that confuses type checkers. Without `django-stubs`,
mypy cannot verify that you are comparing a `DecimalField` with a `Decimal` or passing
the right type to `create()`. Install `django-stubs` and annotate fields for full static
analysis coverage.

**Incorrect (no type annotations, mypy is blind):**

```python
# models.py
class Product(models.Model):
    name = models.CharField(max_length=255)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    is_active = models.BooleanField(default=True)

# service.py -- mypy cannot catch this bug:
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

# service.py -- mypy now catches this:
product.price = "not a number"  # error: Incompatible types
```

Run `mypy --strict` in CI to catch field-type mismatches, missing nullable checks, and
incorrect queryset usage before they reach production.

---

## 3. DRF Patterns

### 3.1 Separate Serializers for Create/Update/List/Detail Actions

**Impact: HIGH -- Prevents data leakage**

A single serializer for all actions leads to over-exposed fields (leaking internal IDs
on list), under-validated input (optional fields on create), and bloated response
payloads. Define action-specific serializers and select them via `get_serializer_class()`.

**Incorrect (one serializer for everything):**

```python
# serializers.py
class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "email", "password", "is_staff", "date_joined", "profile"]

# views.py
class UserViewSet(ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer  # password exposed on list!
```

**Correct (action-specific serializers):**

```python
# serializers.py
class UserListSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "email", "date_joined"]

class UserDetailSerializer(serializers.ModelSerializer):
    profile = ProfileSerializer(read_only=True)

    class Meta:
        model = User
        fields = ["id", "email", "date_joined", "profile"]

class UserCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["email", "password"]
        extra_kwargs = {"password": {"write_only": True}}

# views.py
class UserViewSet(ModelViewSet):
    queryset = User.objects.all()

    def get_serializer_class(self):
        if self.action == "list":
            return UserListSerializer
        if self.action == "create":
            return UserCreateSerializer
        return UserDetailSerializer
```

This pattern also improves OpenAPI documentation by showing distinct request and
response schemas per endpoint.

---

### 3.2 Keep ViewSets Thin -- Override Only What Is Needed

**Impact: HIGH -- Reduces framework coupling**

`ModelViewSet` provides five actions out of the box. If your endpoint only supports list
and retrieve, use `GenericViewSet` with explicit mixins. Overriding `create`, `update`,
and `destroy` just to disable them adds dead code and confuses API consumers.

**Incorrect (disabling actions by raising exceptions):**

```python
# views.py
class ArticleViewSet(ModelViewSet):
    queryset = Article.objects.all()
    serializer_class = ArticleSerializer

    def create(self, request, *args, **kwargs):
        raise MethodNotAllowed("POST")

    def update(self, request, *args, **kwargs):
        raise MethodNotAllowed("PUT")

    def destroy(self, request, *args, **kwargs):
        raise MethodNotAllowed("DELETE")
```

**Correct (compose only the mixins you need):**

```python
# views.py
from rest_framework.mixins import ListModelMixin, RetrieveModelMixin
from rest_framework.viewsets import GenericViewSet

class ArticleViewSet(ListModelMixin, RetrieveModelMixin, GenericViewSet):
    queryset = Article.objects.select_related("author").all()
    serializer_class = ArticleSerializer

    def get_serializer_class(self):
        if self.action == "list":
            return ArticleListSerializer
        return ArticleDetailSerializer
```

This approach produces accurate `Allow` headers, correct OpenAPI schemas, and makes the
available operations immediately obvious from the class definition.

---

### 3.3 Use drf-spectacular for OpenAPI 3.0 Documentation

**Impact: HIGH -- Accurate auto-generated docs**

Hand-maintained API documentation drifts from the actual implementation.
`drf-spectacular` generates an OpenAPI 3.0 schema directly from your serializers, views,
and type hints. Annotate views with `@extend_schema` to add descriptions, examples, and
response codes.

**Incorrect (no schema annotations, relying on guesswork):**

```python
# views.py
class OrderViewSet(ModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderSerializer
    # No documentation -- consumers must read source code
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

Add `drf-spectacular` to `INSTALLED_APPS` and configure `DEFAULT_SCHEMA_CLASS` in
settings to `AutoSchema`. Serve Swagger UI and ReDoc at `/api/docs/` for interactive
exploration.

---

### 3.4 Use CursorPagination or LimitOffsetPagination

**Impact: MEDIUM -- Stable pagination for clients**

The default `PageNumberPagination` breaks when rows are inserted or deleted between
pages, causing duplicates or missed records. `CursorPagination` provides stable, opaque
cursors suitable for infinite-scroll UIs. `LimitOffsetPagination` works for admin
dashboards with known total counts.

**Incorrect (default page number pagination with no configuration):**

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 100,  # no max, no consistent envelope
}

# Response body varies across endpoints -- no standard format.
```

**Correct (CursorPagination with consistent envelope):**

```python
# pagination.py
from rest_framework.pagination import CursorPagination

class StandardCursorPagination(CursorPagination):
    page_size = 25
    max_page_size = 100
    ordering = "-created_at"
    cursor_query_param = "cursor"

    def get_paginated_response(self, data):
        return Response({
            "next": self.get_next_link(),
            "previous": self.get_previous_link(),
            "results": data,
        })

# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "core.pagination.StandardCursorPagination",
    "PAGE_SIZE": 25,
}
```

Use `CursorPagination` for public APIs and mobile clients. Use
`LimitOffsetPagination` only when clients genuinely need total counts (e.g., admin
dashboards) and the table size is bounded.

---

### 3.5 Fine-Grained Permission Classes Per View

**Impact: MEDIUM -- Secure per-endpoint access**

A single global `IsAuthenticated` permission provides no granularity. Different
endpoints require different access rules: owners-only, staff-only, read-public
write-auth, etc. Define small, composable permission classes and attach them at the
view level.

**Incorrect (global permission, no per-view control):**

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
}

# views.py
class InvoiceViewSet(ModelViewSet):
    queryset = Invoice.objects.all()
    serializer_class = InvoiceSerializer
    # Any authenticated user can delete any invoice!
```

**Correct (per-view permission classes):**

```python
# permissions.py
from rest_framework.permissions import BasePermission, SAFE_METHODS

class IsOwnerOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in SAFE_METHODS:
            return True
        return obj.owner == request.user

class IsFinanceTeam(BasePermission):
    def has_permission(self, request, view):
        return request.user.groups.filter(name="finance").exists()

# views.py
class InvoiceViewSet(ModelViewSet):
    queryset = Invoice.objects.all()
    serializer_class = InvoiceSerializer

    def get_permissions(self):
        if self.action in ("update", "partial_update", "destroy"):
            return [IsFinanceTeam()]
        return [IsOwnerOrReadOnly()]
```

Override `get_permissions()` for action-level control. This makes authorization
requirements explicit and testable in isolation.

---

### 3.6 Use django-filter with Typed FilterSet Classes

**Impact: MEDIUM -- Safe declarative filtering**

Hand-parsing query parameters in `get_queryset()` is error-prone and leads to SQL
injection risks or unvalidated input. `django-filter` provides declarative `FilterSet`
classes that validate, type-cast, and apply filters safely.

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

`FilterSet` classes are auto-documented by `drf-spectacular` and can be unit-tested
independently from views.

---

## 4. Query Optimization

### 4.1 Always Use select_related/prefetch_related for Accessed Relations

**Impact: CRITICAL -- Eliminates lazy-load queries**

Accessing a ForeignKey or ManyToMany field on a queryset without `select_related` or
`prefetch_related` triggers a separate SQL query for every object. On a list of 100
items with 2 relations each, this produces 201 queries instead of 3.

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

# serializers.py -- same as before, but now only 3 queries total
class OrderSerializer(serializers.ModelSerializer):
    customer_name = serializers.CharField(source="customer.full_name")
    items = OrderItemSerializer(many=True)

    class Meta:
        model = Order
        fields = ["id", "customer_name", "items", "total"]
```

Use `select_related` for ForeignKey / OneToOneField (SQL JOIN). Use
`prefetch_related` for ManyToMany / reverse ForeignKey (separate query).

---

### 4.2 Detect and Prevent N+1 Queries

**Impact: HIGH -- Catches hidden query explosions**

Even disciplined teams miss N+1 regressions when a new serializer field accesses an
unloaded relation. Automated detection tools catch these problems in development and CI
before they degrade production performance.

**Incorrect (no automated detection):**

```python
# settings.py -- no query monitoring configured

# views.py
class TeamViewSet(ModelViewSet):
    queryset = Team.objects.all()
    serializer_class = TeamSerializer

# serializers.py -- accesses members without prefetch
class TeamSerializer(serializers.ModelSerializer):
    member_count = serializers.SerializerMethodField()

    class Meta:
        model = Team
        fields = ["id", "name", "member_count"]

    def get_member_count(self, obj):
        return obj.members.count()  # N+1: one query per team
```

**Correct (nplusone detection + fixed query):**

```python
# settings.py
INSTALLED_APPS = [
    "nplusone.ext.django",
    # ...
]
MIDDLEWARE = [
    "nplusone.ext.django.NPlusOneMiddleware",
    # ...
]
NPLUSONE_RAISE = True  # raise exception in dev/test

# views.py
class TeamViewSet(ModelViewSet):
    queryset = Team.objects.prefetch_related("members")
    serializer_class = TeamSerializer

# serializers.py
class TeamSerializer(serializers.ModelSerializer):
    member_count = serializers.SerializerMethodField()

    class Meta:
        model = Team
        fields = ["id", "name", "member_count"]

    def get_member_count(self, obj):
        return obj.members.count()  # uses prefetched cache
```

Enable `NPLUSONE_RAISE = True` in development and test environments. In CI, fail the
build on any N+1 detection to prevent regressions.

---

### 4.3 Use .only()/.defer() to Load Only Needed Columns

**Impact: HIGH -- Reduces memory and latency**

Loading all columns by default wastes memory and bandwidth, especially for models with
large text fields, JSON blobs, or binary data. Use `.only()` to whitelist needed fields
or `.defer()` to exclude heavy fields.

**Incorrect (loading all columns including a large body field):**

```python
# views.py
class ArticleListView(ListAPIView):
    queryset = Article.objects.all()  # loads 50KB body per row
    serializer_class = ArticleListSerializer

# serializers.py
class ArticleListSerializer(serializers.ModelSerializer):
    class Meta:
        model = Article
        fields = ["id", "title", "author", "published_at"]
        # body field is loaded from DB but never serialized
```

**Correct (deferring the heavy body field):**

```python
# views.py
class ArticleListView(ListAPIView):
    queryset = Article.objects.defer("body", "raw_html")
    serializer_class = ArticleListSerializer

# Or using .only() for an explicit whitelist:
class ArticleListView(ListAPIView):
    queryset = Article.objects.only(
        "id", "title", "author_id", "published_at",
    ).select_related("author")
    serializer_class = ArticleListSerializer
```

Prefer `.only()` when the serializer uses a small subset of fields. Use `.defer()` when
you need most fields but want to exclude one or two heavy columns. Never access a
deferred field in serializer code, as it triggers an extra query per object.

---

### 4.4 Use bulk_create/bulk_update for Batch Operations

**Impact: HIGH -- Eliminates per-row queries**

Creating or updating objects one at a time inside a loop generates N INSERT or UPDATE
statements. `bulk_create` and `bulk_update` batch these into a single query (or a few
chunked queries), reducing round-trips by orders of magnitude.

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

Set `batch_size` to avoid exceeding database parameter limits (typically around 1000
for PostgreSQL). Note that `bulk_create` does not call `save()` or fire signals by
default.

---

### 4.5 Use .exists() Instead of .count() > 0

**Impact: MEDIUM -- Faster existence checks**

`.count()` forces the database to scan and aggregate all matching rows. `.exists()` adds
`LIMIT 1` to the query, returning as soon as a single row is found. For large tables or
expensive filters, this difference is significant.

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

Use `.count()` only when you actually need the number. For boolean checks, always
prefer `.exists()`. Also avoid `len(queryset)` for counting, as it loads all rows into
Python memory just to count them.

---

## 5. Testing

### 5.1 Use factory_boy for Test Data

**Impact: MEDIUM -- Maintainable test fixtures**

Scattering `Model.objects.create()` across tests creates brittle setups that break when
model fields change. `factory_boy` centralizes default values, supports traits and
sub-factories, and makes test intent clear by only specifying fields relevant to the
test case.

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

Use `factory.Trait` for named variations (e.g., `OrderFactory(vip=True)`) and
`factory.LazyAttribute` for computed defaults.

---

### 5.2 Unit Test Services with Mocked Dependencies

**Impact: MEDIUM -- Fast isolated service tests**

Service functions contain business logic and should be tested in isolation from the
database and external APIs. Mock the data layer and side effects to keep tests fast
(milliseconds, not seconds) and focused on behavior.

**Incorrect (service test hitting the database):**

```python
# tests/test_billing_service.py
@pytest.mark.django_db
class TestBillingService:
    def test_charge_user(self):
        user = User.objects.create(email="a@b.com", password="x")
        account = Account.objects.create(user=user, balance=Decimal("100"))
        # Hits DB, slow, tests persistence + logic together
        result = charge_user(user_id=user.id, amount=Decimal("30"))
        account.refresh_from_db()
        assert account.balance == Decimal("70")
        assert result.success is True
```

**Correct (mocked dependencies, pure logic test):**

```python
# tests/test_billing_service.py
from unittest.mock import patch, MagicMock

class TestBillingService:
    @patch("services.billing_service.Account.objects")
    @patch("services.billing_service.send_receipt")
    def test_charge_user_deducts_balance(self, mock_send, mock_qs):
        account = MagicMock(balance=Decimal("100"), is_frozen=False)
        mock_qs.select_for_update.return_value.get.return_value = account

        result = charge_user(user_id=1, amount=Decimal("30"))

        assert account.balance == Decimal("70")
        account.save.assert_called_once_with(update_fields=["balance"])
        mock_send.delay.assert_called_once_with(1, Decimal("30"))
        assert result.success is True

    @patch("services.billing_service.Account.objects")
    def test_charge_user_rejects_frozen_account(self, mock_qs):
        account = MagicMock(is_frozen=True)
        mock_qs.select_for_update.return_value.get.return_value = account

        with pytest.raises(AccountFrozenError):
            charge_user(user_id=1, amount=Decimal("30"))
```

Keep one integration test that hits the real database for each service to verify ORM
interactions, but the bulk of test cases should be fast unit tests.

---

### 5.3 Integration Test API Endpoints with APIClient

**Impact: MEDIUM -- Validates full request cycle**

Unit testing services verifies business logic, but integration tests validate the full
HTTP cycle: routing, authentication, serialization, permissions, and status codes. DRF's
`APIClient` provides a clean interface for these tests without spinning up a real server.

**Incorrect (testing views by calling methods directly):**

```python
# tests/test_views.py
class TestOrderView:
    def test_create_order(self):
        view = OrderCreateView()
        request = HttpRequest()
        request.method = "POST"
        request.POST = {"product_id": 1, "quantity": 2}
        # Bypasses middleware, auth, content negotiation
        response = view.post(request)
        assert response.status_code == 201
```

**Correct (full-stack test with APIClient):**

```python
# tests/test_api.py
import pytest
from rest_framework.test import APIClient
from tests.factories import UserFactory, ProductFactory

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def authenticated_client(api_client):
    user = UserFactory()
    api_client.force_authenticate(user=user)
    return api_client

@pytest.mark.django_db
class TestOrderAPI:
    def test_create_order_returns_201(self, authenticated_client):
        product = ProductFactory(stock=10)
        response = authenticated_client.post(
            "/api/orders/",
            data={"product_id": product.id, "quantity": 2},
            format="json",
        )
        assert response.status_code == 201
        assert response.data["quantity"] == 2

    def test_create_order_unauthenticated_returns_401(self, api_client):
        response = api_client.post(
            "/api/orders/",
            data={"product_id": 1, "quantity": 1},
            format="json",
        )
        assert response.status_code == 401

    def test_list_orders_returns_only_own(self, authenticated_client):
        response = authenticated_client.get("/api/orders/")
        assert response.status_code == 200
        assert isinstance(response.data["results"], list)
```

Use `force_authenticate()` instead of managing tokens in tests. Use `factory_boy`
fixtures for test data and `pytest.mark.django_db` to enable database access only in
integration tests.
