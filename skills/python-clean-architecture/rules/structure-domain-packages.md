---
title: Top-Level Packages as Bounded Contexts
impact: HIGH
impactDescription: Navigable domain structure
tags: structure, packages, bounded-context, organization
---

## Top-Level Packages as Bounded Contexts

Organize top-level Python packages to mirror your domain's bounded contexts. Each package (e.g., `users/`, `orders/`, `payments/`) contains its own domain models, services, repositories, and API routes. Shared infrastructure lives in a separate `infrastructure/` or `shared/` package.

**Incorrect (technical layer structure):**

```python
project/
    models/
        user.py
        order.py
        payment.py
    services/
        user_service.py
        order_service.py
        payment_service.py
    repositories/
        user_repo.py
        order_repo.py
        payment_repo.py
    api/
        user_routes.py
        order_routes.py
```

**Correct (domain package structure):**

```python
project/
    users/
        __init__.py
        domain.py        # User entity, value objects
        service.py       # UserService
        repository.py    # UserRepository protocol + implementation
        api.py           # user-related API routes
        interface.py     # protocols for cross-context use
    orders/
        __init__.py
        domain.py
        service.py
        repository.py
        api.py
    payments/
        __init__.py
        domain.py
        service.py
        repository.py
        api.py
    shared/
        __init__.py
        database.py      # shared DB engine, session factory
        settings.py      # typed application settings
    main.py              # composition root
```

A developer looking at the top-level directory immediately understands the business domains the system handles.
