---
title: Use asyncio.gather for Parallel Operations
impact: HIGH
impactDescription: Sequential async waste
tags: async, gather, parallel, concurrency
---

## Use asyncio.gather for Parallel Operations

Use `asyncio.gather()` for independent async operations that can run concurrently. Awaiting operations sequentially when they have no data dependency wastes time proportional to the sum of all latencies instead of the maximum.

**Incorrect (sequential awaits for independent operations):**

```python
@router.get("/dashboard/{user_id}")
async def get_dashboard(
    user_id: int,
    user_svc: UserService = Depends(get_user_service),
    order_svc: OrderService = Depends(get_order_service),
    notification_svc: NotificationService = Depends(get_notification_service),
):
    user = await user_svc.get(user_id)            # 100ms
    orders = await order_svc.get_recent(user_id)   # 150ms
    alerts = await notification_svc.get(user_id)   # 80ms
    # Total: ~330ms (sequential)
    return {"user": user, "orders": orders, "alerts": alerts}
```

**Correct (parallel with asyncio.gather):**

```python
import asyncio

@router.get("/dashboard/{user_id}")
async def get_dashboard(
    user_id: int,
    user_svc: UserService = Depends(get_user_service),
    order_svc: OrderService = Depends(get_order_service),
    notification_svc: NotificationService = Depends(get_notification_service),
) -> DashboardResponse:
    user, orders, alerts = await asyncio.gather(
        user_svc.get(user_id),
        order_svc.get_recent(user_id),
        notification_svc.get(user_id),
    )
    # Total: ~150ms (parallel, limited by slowest)
    return DashboardResponse(user=user, orders=orders, alerts=alerts)
```

Use `return_exceptions=True` if you want to handle partial failures instead of having one failure cancel everything.

Reference: https://docs.python.org/3/library/asyncio-task.html#asyncio.gather
