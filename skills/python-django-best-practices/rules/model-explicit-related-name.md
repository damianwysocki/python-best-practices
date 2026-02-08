---
title: Always Set related_name on ForeignKey and ManyToMany
impact: HIGH
impactDescription: Readable reverse relations
tags: models, foreignkey, related-name, relationships
---

## Always Set related_name on ForeignKey and ManyToMany

Django auto-generates reverse relation names using `<model>_set`, which is
ambiguous when a model has multiple foreign keys to the same target. Explicit
`related_name` values make reverse lookups self-documenting and prevent
clash errors in abstract models.

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
`"%(app_label)s_%(class)s_related"` to avoid reverse accessor clashes
across apps.

Reference: https://docs.djangoproject.com/en/5.0/ref/models/fields/#django.db.models.ForeignKey.related_name
