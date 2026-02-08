---
title: Views/ViewSets Handle HTTP Concerns Only
impact: CRITICAL
impactDescription: Clean HTTP boundary layer
tags: service-layer, views, viewsets, http, thin-views
---

## Views/ViewSets Handle HTTP Concerns Only, Delegate to Services

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

A good heuristic: if a view method exceeds 8-10 lines, extract the logic into
a service. Views should read like a table of contents, not a novel.

Reference: https://www.django-rest-framework.org/api-guide/viewsets/
