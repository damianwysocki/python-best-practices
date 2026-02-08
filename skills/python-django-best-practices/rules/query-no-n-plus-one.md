---
title: Detect and Prevent N+1 Queries
impact: HIGH
impactDescription: Catches hidden query explosions
tags: queries, n-plus-one, nplusone, performance, debugging
---

## Detect and Prevent N+1 Queries with django-query-count or nplusone

Even disciplined teams miss N+1 regressions when a new serializer field
accesses an unloaded relation. Automated detection tools catch these problems
in development and CI before they degrade production performance.

**Incorrect (no automated detection):**

```python
# settings.py — no query monitoring configured

# views.py
class TeamViewSet(ModelViewSet):
    queryset = Team.objects.all()
    serializer_class = TeamSerializer

# serializers.py — accesses members without prefetch
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

Enable `NPLUSONE_RAISE = True` in development and test environments.
In CI, fail the build on any N+1 detection to prevent regressions.

Reference: https://github.com/jmcarp/nplusone
