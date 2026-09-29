# Example integration pattern: Python / Django (Django REST Framework)

One worked example of the general pattern in SKILL.md — plus the field-validation template tier from `error-classification.md`, since DRF's `ValidationError` already carries structured `{field: [reasons]}` data, exactly the shape that tier is built for. If the project doesn't use DRF, see "Plain Django (no DRF)" below and adapt the rest of this shape (catalog files + pure mapper functions + one central boundary) accordingly.

## Files

```
app/errors/message_catalog.json     # seeded from references/message-catalog.json
app/errors/field_labels.json        # seeded from references/field-labels.json
app/errors/format_hints.json        # seeded from references/format-hints.json, corrected to the project's actual formats
app/errors/map_error.py             # pure functions, no I/O
app/errors/handlers.py              # DRF EXCEPTION_HANDLER
```

## `app/errors/map_error.py`

```python
import json
import re
from dataclasses import dataclass
from pathlib import Path

_DIR = Path(__file__).parent
_CATALOG = json.loads((_DIR / "message_catalog.json").read_text())
_FIELD_LABELS = json.loads((_DIR / "field_labels.json").read_text())["labels"]
_FORMAT_HINTS = json.loads((_DIR / "format_hints.json").read_text())["hints"]

# Project-specific: which fields are of which known type, for format hints.
# Kept explicit rather than guessed from the field name, so it stays correct.
_FIELD_TYPES = {
    "dob": "date",
    "poc_email": "email",
    "poc_phone": "phone",
}


@dataclass
class MappedError:
    message: str
    retryable: bool
    correlation_id: str


def _humanize_field(field: str) -> str:
    if field in _FIELD_LABELS:
        return _FIELD_LABELS[field]
    words = re.split(r"[_\-]|(?<=[a-z])(?=[A-Z])", field)
    return " ".join(w.capitalize() for w in words if w)


def map_validation_error_to_user_message(field: str, correlation_id: str) -> MappedError:
    """Tier 1 — structured field-validation template. See error-classification.md."""
    label = _humanize_field(field)
    hint = _FORMAT_HINTS.get(_FIELD_TYPES.get(field))
    message = f"{label} has an invalid value." + (f" {hint}" if hint else " Please check it and try again.")
    return MappedError(message, retryable=False, correlation_id=correlation_id)


def map_error_to_user_message(raw_message: str | None, status: int | None, correlation_id: str) -> MappedError:
    """Tiers 2–4 — pattern match, status-class fallback, generic fallback."""
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

from rest_framework.views import exception_handler as drf_default_exception_handler
from rest_framework.response import Response
from rest_framework.exceptions import ValidationError

from .map_error import map_error_to_user_message, map_validation_error_to_user_message

logger = logging.getLogger("app")


def custom_exception_handler(exc, context):
    request = context.get("request")
    correlation_id = (request.headers.get("x-correlation-id") if request else None) or str(uuid.uuid4())

    response = drf_default_exception_handler(exc, context)

    logger.error(
        "request failed",
        exc_info=exc,
        extra={"correlation_id": correlation_id, "path": getattr(request, "path", None)},
    )

    if response is None:
        # DRF didn't produce a response — an unhandled exception, typically a 500.
        mapped = map_error_to_user_message(str(exc), 500, correlation_id)
        return Response(
            {"error": mapped.message, "retryable": mapped.retryable, "correlationId": correlation_id},
            status=500,
        )

    if isinstance(exc, ValidationError) and isinstance(response.data, dict):
        # DRF gives {"field": ["reason", ...], ...} — map every field individually (see
        # "Multiple simultaneous field errors" in error-classification.md).
        response.data = {
            "errors": [
                {"field": field, "message": map_validation_error_to_user_message(field, correlation_id).message}
                for field in response.data.keys()
            ],
            "correlationId": correlation_id,
        }
        return response

    mapped = map_error_to_user_message(str(exc), response.status_code, correlation_id)
    response.data = {"error": mapped.message, "retryable": mapped.retryable, "correlationId": correlation_id}
    return response
```

Register it in settings:

```python
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "app.errors.handlers.custom_exception_handler",
}
```

## Plain Django (no DRF)

Without DRF there's no structured per-field `ValidationError` payload to key the template off, so use a `process_exception` middleware with just the pattern/status-fallback mapper:

```python
# app/middleware.py
import uuid
from django.http import JsonResponse
from app.errors.map_error import map_error_to_user_message


class ErrorMappingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        return self.get_response(request)

    def process_exception(self, request, exc):
        correlation_id = request.headers.get("x-correlation-id", str(uuid.uuid4()))
        mapped = map_error_to_user_message(str(exc), 500, correlation_id)
        return JsonResponse(
            {"error": mapped.message, "retryable": mapped.retryable, "correlationId": correlation_id},
            status=500,
        )
```

Register it near the end of `MIDDLEWARE`, so it wraps as much of the view stack as possible. If individual views build their own Django forms and want field-level messages, call `map_validation_error_to_user_message` directly on each invalid form field instead.

## Rewiring existing leak points

Search for and replace patterns like:

```python
# before — leaks str(e), or raw unmapped DRF field errors, straight to the client
return JsonResponse({"error": str(e)}, status=500)
return Response({"error": serializer.errors}, status=400)
```

Prefer letting exceptions propagate to `custom_exception_handler` (DRF) or the middleware (plain Django) rather than building ad hoc JSON error responses inline in each view. For a `serializer.errors` dict specifically, route each field through `map_validation_error_to_user_message` instead of returning DRF's raw field-error text — it's already fairly plain-English, but won't match this project's chosen field labels/format hints/tone unless it goes through the same mapper.
