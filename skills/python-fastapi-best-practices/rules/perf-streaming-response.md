---
title: Use StreamingResponse for Large Payloads
impact: LOW
impactDescription: High memory usage
tags: performance, streaming, response, files
---

## Use StreamingResponse for Large Payloads

Use `StreamingResponse` for large payloads or file downloads instead of loading the entire content into memory. A 500MB file returned as a regular response consumes 500MB of server memory per concurrent request. Streaming sends chunks as they are produced.

**Incorrect (loading entire file into memory):**

```python
@router.get("/reports/{report_id}/download")
async def download_report(report_id: int):
    file_path = f"/data/reports/{report_id}.csv"
    with open(file_path, "rb") as f:
        content = f.read()  # entire file in memory
    return Response(
        content=content,
        media_type="text/csv",
        headers={"Content-Disposition": f"attachment; filename={report_id}.csv"},
    )
```

**Correct (streaming response):**

```python
from fastapi.responses import StreamingResponse
import aiofiles

async def file_streamer(file_path: str):
    async with aiofiles.open(file_path, "rb") as f:
        while chunk := await f.read(64 * 1024):  # 64KB chunks
            yield chunk

@router.get("/reports/{report_id}/download")
async def download_report(report_id: int):
    file_path = f"/data/reports/{report_id}.csv"
    return StreamingResponse(
        file_streamer(file_path),
        media_type="text/csv",
        headers={
            "Content-Disposition": f"attachment; filename={report_id}.csv"
        },
    )
```

Streaming also works well for server-sent events and large JSON arrays:

```python
async def stream_users(db: AsyncSession):
    result = await db.stream(select(User))
    async for row in result:
        yield json.dumps(UserResponse.model_validate(row).model_dump()) + "\n"
```

Reference: https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse
