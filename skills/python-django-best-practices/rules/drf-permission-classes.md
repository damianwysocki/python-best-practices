---
title: Fine-Grained Permission Classes Per View
impact: MEDIUM
impactDescription: Secure per-endpoint access
tags: drf, permissions, security, authorization, access-control
---

## Fine-Grained Permission Classes Per View, Not Global

A single global `IsAuthenticated` permission provides no granularity. Different
endpoints require different access rules: owners-only, staff-only, read-public
write-auth, etc. Define small, composable permission classes and attach them at
the view level.

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

Override `get_permissions()` for action-level control. This makes
authorization requirements explicit and testable in isolation.

Reference: https://www.django-rest-framework.org/api-guide/permissions/#custom-permissions
