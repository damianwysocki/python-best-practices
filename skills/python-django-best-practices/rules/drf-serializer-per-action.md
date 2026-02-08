---
title: Separate Serializers for Create/Update/List/Detail Actions
impact: HIGH
impactDescription: Prevents data leakage
tags: drf, serializers, viewsets, actions, security
---

## Separate Serializers for Create/Update/List/Detail Actions

A single serializer for all actions leads to over-exposed fields (leaking
internal IDs on list), under-validated input (optional fields on create),
and bloated response payloads. Define action-specific serializers and select
them via `get_serializer_class()`.

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

This pattern also improves OpenAPI documentation by showing distinct request
and response schemas per endpoint.

Reference: https://www.django-rest-framework.org/api-guide/generic-views/#get_serializer_class
