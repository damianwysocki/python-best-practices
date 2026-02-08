---
title: Correct Middleware Ordering
impact: MEDIUM
impactDescription: Bypassed security layers
tags: router, middleware, cors, ordering
---

## Correct Middleware Ordering

Apply middleware in the correct order: CORS, Authentication, Logging, Error handling. FastAPI middleware executes in reverse registration order (last registered runs first on request). Incorrect ordering can bypass security checks or produce confusing logs.

**Incorrect (auth before CORS, errors not caught):**

```python
app = FastAPI()

# Wrong order: Auth runs before CORS preflight is handled
app.add_middleware(AuthMiddleware)
app.add_middleware(CORSMiddleware, allow_origins=["*"])
app.add_middleware(LoggingMiddleware)
```

**Correct (CORS first, then auth, logging, errors):**

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# Registered in reverse execution order.
# Last registered = first to run on incoming request.

# 4. Error handling (innermost — catches service exceptions)
app.add_middleware(ErrorHandlingMiddleware)

# 3. Request logging (logs after auth has identified the user)
app.add_middleware(LoggingMiddleware)

# 2. Authentication (runs after CORS preflight is handled)
app.add_middleware(AuthMiddleware)

# 1. CORS (outermost — handles preflight before anything else)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_methods=["*"],
    allow_headers=["*"],
    allow_credentials=True,
)
```

The CORS middleware must handle OPTIONS preflight requests before auth rejects them. Logging after auth means you can include the authenticated user in log entries.

Reference: https://fastapi.tiangolo.com/tutorial/middleware/
