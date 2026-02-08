---
title: Never Call Blocking I/O in Async Endpoints
impact: HIGH
impactDescription: Event loop starvation
tags: async, blocking, executor, sync
---

## Never Call Blocking I/O in Async Endpoints

Never call blocking (synchronous) I/O inside an `async def` endpoint. Blocking calls freeze the entire event loop, preventing all other requests from being processed. If you must use a sync library, offload it to a thread via `run_in_executor`.

**Incorrect (blocking call in async endpoint):**

```python
import time
from PIL import Image

@router.post("/resize")
async def resize_image(file: UploadFile):
    contents = await file.read()
    # CPU-bound blocking call freezes the event loop
    img = Image.open(io.BytesIO(contents))
    img = img.resize((200, 200))
    buffer = io.BytesIO()
    img.save(buffer, format="PNG")
    return Response(content=buffer.getvalue(), media_type="image/png")
```

**Correct (offload blocking work to executor):**

```python
import asyncio
import io
from concurrent.futures import ThreadPoolExecutor
from PIL import Image

executor = ThreadPoolExecutor(max_workers=4)

def _resize_sync(contents: bytes) -> bytes:
    img = Image.open(io.BytesIO(contents))
    img = img.resize((200, 200))
    buffer = io.BytesIO()
    img.save(buffer, format="PNG")
    return buffer.getvalue()

@router.post("/resize")
async def resize_image(file: UploadFile):
    contents = await file.read()
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(executor, _resize_sync, contents)
    return Response(content=result, media_type="image/png")
```

Alternatively, use `def` (not `async def`) for endpoints that are entirely synchronous. FastAPI will run them in a threadpool automatically.

Reference: https://fastapi.tiangolo.com/async/#in-a-hurry
