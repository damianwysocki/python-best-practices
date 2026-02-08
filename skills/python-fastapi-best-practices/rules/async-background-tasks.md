---
title: Use BackgroundTasks for Fire-and-Forget
impact: MEDIUM
impactDescription: Slow response times
tags: async, background, tasks, fire-and-forget
---

## Use BackgroundTasks for Fire-and-Forget

Use `BackgroundTasks` for operations that should not block the response, such as sending emails, writing audit logs, or triggering webhooks. The response is sent immediately while the task runs after.

**Incorrect (blocking on non-essential work):**

```python
@router.post("/orders", status_code=201)
async def create_order(payload: OrderCreate, service=Depends(get_order_service)):
    order = await service.create(payload)
    await send_confirmation_email(order.user_email, order.id)  # blocks 2-3s
    await notify_warehouse(order)  # blocks another 1s
    return order  # user waits 3-4s total
```

**Correct (non-essential work in background):**

```python
from fastapi import BackgroundTasks

@router.post("/orders", status_code=201)
async def create_order(
    payload: OrderCreate,
    background_tasks: BackgroundTasks,
    service: OrderService = Depends(get_order_service),
) -> OrderResponse:
    order = await service.create(payload)
    background_tasks.add_task(send_confirmation_email, order.user_email, order.id)
    background_tasks.add_task(notify_warehouse, order)
    return order  # responds immediately
```

`BackgroundTasks` runs after the response is sent. For long-running or critical tasks that must survive process restarts, use a proper task queue like Celery or arq instead.

```python
# For critical tasks, prefer a task queue
from app.worker import celery_app

@router.post("/reports")
async def generate_report(params: ReportParams):
    task = celery_app.send_task("generate_report", args=[params.model_dump()])
    return {"task_id": task.id}
```

Reference: https://fastapi.tiangolo.com/tutorial/background-tasks/
