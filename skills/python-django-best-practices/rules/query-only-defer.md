---
title: Use .only()/.defer() to Load Only Needed Columns
impact: HIGH
impactDescription: Reduces memory and latency
tags: queries, only, defer, optimization, columns
---

## Use .only()/.defer() to Load Only Needed Columns

Loading all columns by default wastes memory and bandwidth, especially for
models with large text fields, JSON blobs, or binary data. Use `.only()` to
whitelist needed fields or `.defer()` to exclude heavy fields. This reduces
both database transfer time and Python object size.

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

Prefer `.only()` when the serializer uses a small subset of fields.
Use `.defer()` when you need most fields but want to exclude one or two
heavy columns. Never access a deferred field in serializer code, as it
triggers an extra query per object.

Reference: https://docs.djangoproject.com/en/5.0/ref/models/querysets/#only
