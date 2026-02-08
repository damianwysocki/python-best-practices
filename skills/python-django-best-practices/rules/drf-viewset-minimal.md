---
title: Keep ViewSets Thin — Override Only What Is Needed
impact: HIGH
impactDescription: Reduces framework coupling
tags: drf, viewsets, minimal, override, clean-code
---

## Keep ViewSets Thin; Override Only What Is Needed

`ModelViewSet` provides five actions out of the box. If your endpoint only
supports list and retrieve, use `GenericViewSet` with explicit mixins.
Overriding `create`, `update`, and `destroy` just to disable them adds dead
code and confuses API consumers.

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

This approach produces accurate `Allow` headers, correct OpenAPI schemas, and
makes the available operations immediately obvious from the class definition.

Reference: https://www.django-rest-framework.org/api-guide/viewsets/#custom-viewset-base-classes
