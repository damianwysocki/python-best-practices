---
title: Keep Endpoints Slim
impact: HIGH
impactDescription: Untestable route logic
tags: router, architecture, services, slim
---

## Keep Endpoints Slim

Endpoints should call services only; keep zero business logic in route handlers. The endpoint's job is to accept validated input, call a service, and return a response. Any logic beyond that belongs in a service layer. This keeps routes testable, services reusable, and concerns separated.

**Incorrect (business logic embedded in endpoint):**

```python
@router.post("/orders")
async def create_order(
    payload: OrderCreate,
    db: AsyncSession = Depends(get_db),
):
    product = await db.get(Product, payload.product_id)
    if not product or product.stock < payload.quantity:
        raise HTTPException(400, "Insufficient stock")
    total = product.price * payload.quantity * (1 - payload.discount)
    order = Order(total=total, **payload.model_dump())
    db.add(order)
    product.stock -= payload.quantity
    await db.commit()
    await send_confirmation_email(order)
    return order
```

**Correct (endpoint delegates to service):**

```python
@router.post(
    "/orders",
    response_model=OrderResponse,
    status_code=status.HTTP_201_CREATED,
)
async def create_order(
    payload: OrderCreate,
    service: OrderService = Depends(get_order_service),
) -> OrderResponse:
    return await service.create_order(payload)
```

The service encapsulates validation, calculation, persistence, and side effects. The endpoint is a thin adapter between HTTP and the domain.

Reference: https://fastapi.tiangolo.com/tutorial/bigger-applications/
