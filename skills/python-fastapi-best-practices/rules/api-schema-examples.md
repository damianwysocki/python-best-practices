---
title: Provide Request and Response Examples
impact: HIGH
impactDescription: Unclear API contracts
tags: openapi, examples, schema, pydantic
---

## Provide Request and Response Examples

Provide request and response examples via `model_config` with `json_schema_extra` on Pydantic models. Examples appear in the OpenAPI docs and make it trivial for consumers to understand the expected payload shape without reading code.

**Incorrect (no examples in schema):**

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    email: str
    full_name: str
    password: str
```

**Correct (examples provided via model_config):**

```python
from pydantic import BaseModel, EmailStr

class UserCreate(BaseModel):
    email: EmailStr
    full_name: str
    password: str

    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "email": "jane@example.com",
                    "full_name": "Jane Doe",
                    "password": "s3cure!Pass",
                }
            ]
        }
    }
```

You can also provide examples at the field level using `Field`:

```python
from pydantic import BaseModel, Field

class OrderCreate(BaseModel):
    product_id: int = Field(..., examples=[42])
    quantity: int = Field(..., ge=1, examples=[3])
    notes: str | None = Field(None, examples=["Gift wrap please"])
```

Reference: https://docs.pydantic.dev/latest/concepts/json_schema/#schema-customization
