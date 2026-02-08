---
title: Use CursorPagination or LimitOffsetPagination
impact: MEDIUM
impactDescription: Stable pagination for clients
tags: drf, pagination, cursor, performance, api-design
---

## Use CursorPagination or LimitOffsetPagination with Consistent Format

The default `PageNumberPagination` breaks when rows are inserted or deleted
between pages, causing duplicates or missed records. `CursorPagination`
provides stable, opaque cursors suitable for infinite-scroll UIs.
`LimitOffsetPagination` works for admin dashboards with known total counts.

**Incorrect (default page number pagination with no configuration):**

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 100,  # no max, no consistent envelope
}

# Response body varies across endpoints — no standard format.
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
`LimitOffsetPagination` only when clients genuinely need total counts
(e.g., admin dashboards) and the table size is bounded.

Reference: https://www.django-rest-framework.org/api-guide/pagination/#cursorpagination
