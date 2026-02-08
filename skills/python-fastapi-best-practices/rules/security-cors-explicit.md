---
title: Configure CORS with Explicit Origins
impact: MEDIUM
impactDescription: Cross-origin vulnerability
tags: security, cors, origins, middleware
---

## Configure CORS with Explicit Origins

Configure CORS with explicit allowed origins; never use wildcard `"*"` in production. A wildcard allows any website to make requests to your API, enabling CSRF-like attacks and data exfiltration. List only the origins your frontend actually runs on.

**Incorrect (wildcard origin in production):**

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],        # any site can call your API
    allow_credentials=True,     # dangerous with wildcard
    allow_methods=["*"],
    allow_headers=["*"],
)
```

**Correct (explicit origins per environment):**

```python
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings

ALLOWED_ORIGINS: dict[str, list[str]] = {
    "production": [
        "https://app.example.com",
        "https://admin.example.com",
    ],
    "staging": [
        "https://staging.example.com",
    ],
    "development": [
        "http://localhost:3000",
        "http://localhost:5173",
    ],
}

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS[settings.environment],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    max_age=600,  # cache preflight for 10 minutes
)
```

Note that `allow_credentials=True` is incompatible with `allow_origins=["*"]` per the CORS specification. Browsers will reject such a configuration.

Reference: https://fastapi.tiangolo.com/tutorial/cors/
