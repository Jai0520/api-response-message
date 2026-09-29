# Example integration pattern: Python / FastAPI

One worked example of the general pattern in SKILL.md. Adapt the same shape (catalog file + pure mapper function + single central exception handler) to Django, Flask, or whatever the target project actually uses.

## Files

```
app/errors/message_catalog.json     # seeded from references/message-catalog.json, extended per project
app/errors/map_error.py             # pure function, no I/O
app/errors/handlers.py              # registered exception handlers
```

## `app/errors/map_error.py`

```python
import json
import re
from dataclasses import dataclass
from pathlib import Path

_CATALOG = json.loads((Path(__file__).parent / "message_catalog.json").read_text())


@dataclass
class MappedError:
    message: str
    retryable: bool
    correlation_id: str


def map_error_to_user_message(raw_message: str | None, status: int | None, correlation_id: str) -> MappedError:
    haystack = raw_message or ""

    for rule in _CATALOG["patterns"]:
        if re.search(rule["match"], haystack):
            return MappedError(rule["userMessage"], rule["retryable"], correlation_id)

    by_status = _CATALOG["defaults"].get(str(status)) if status is not None else None
    return MappedError(
        by_status or _CATALOG["defaults"]["default"],
        bool(status and status >= 500),
        correlation_id,
    )
```

## `app/errors/handlers.py`

```python
import logging
import uuid

from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import HTTPException

from .map_error import map_error_to_user_message

logger = logging.getLogger("app")


async def http_exception_handler(request: Request, exc: HTTPException):
    correlation_id = request.headers.get("x-correlation-id", str(uuid.uuid4()))
    logger.error("request failed", extra={"correlation_id": correlation_id, "path": request.url.path}, exc_info=exc)

    mapped = map_error_to_user_message(str(exc.detail), exc.status_code, correlation_id)
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": mapped.message, "retryable": mapped.retryable, "correlationId": correlation_id},
    )


async def unhandled_exception_handler(request: Request, exc: Exception):
    correlation_id = request.headers.get("x-correlation-id", str(uuid.uuid4()))
    logger.exception("unhandled error", extra={"correlation_id": correlation_id, "path": request.url.path})

    mapped = map_error_to_user_message(str(exc), 500, correlation_id)
    return JSONResponse(
        status_code=500,
        content={"error": mapped.message, "retryable": mapped.retryable, "correlationId": correlation_id},
    )
```

Register both in the app setup:

```python
app.add_exception_handler(HTTPException, http_exception_handler)
app.add_exception_handler(Exception, unhandled_exception_handler)
```

## Rewiring existing leak points

Search for and replace patterns like:

```python
# before — leaks str(e) / raw dict straight to the client
return JSONResponse(status_code=500, content={"error": str(e)})
raise HTTPException(status_code=500, detail=str(e))
```

Prefer letting exceptions propagate to the registered handlers above rather than building ad hoc JSON error responses inline in each route.
