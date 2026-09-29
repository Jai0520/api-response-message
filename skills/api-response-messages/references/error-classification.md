# Error classification rules

This is the rulebook the mapper logic must implement, in any language. Read it in full before generating or editing any code.

## Golden rule

The user sees **why** (briefly, safely) and **what to do next**. The user never sees **how it broke internally**.

Raw errors go to the server-side log (with full detail + a correlation ID). Only the mapped, friendly message + correlation ID goes to the client/frontend.

**Exception for validation errors:** naming *which field* is wrong, in the user's own vocabulary, is the "what to do next" for a validation error — it is not "how it broke internally." Don't over-apply the golden rule here and flatten a validation error down to a generic "some information isn't valid"; that's worse UX, not safer. It only becomes an internal-detail leak the moment it echoes something the user didn't actually type into that field — a validator/library name, a regex, an internal enum, a DB column name that differs from the field's public name. See "Templated field-validation messages" below.

## Never let these reach the frontend

Treat all of the following as leaks, whether they appear in a JSON `error` field, a plain-text response, or an exception re-thrown up to an HTTP handler:

- Stack traces (`at ... (file:line:col)`, `Traceback (most recent call last)`, etc.)
- File paths, line numbers, source file names
- Exception/class names (`NullPointerException`, `TypeError`, `psycopg2.OperationalError`, ...)
- SQL fragments, table/column names, ORM error text
- Cloud/vendor/service names (S3, AWS, GCP, Azure, Redis, Kafka, internal hostnames, ports, IPs)
- Tokens, API keys, JWTs, secrets, credentials, session internals
- Raw stringified JSON error payloads (`{"error_invalidation":"internal api call failed. invalid token"}`) — these are exactly what this skill exists to eliminate
- Environment/config key names
- Anything that reveals internal architecture (queue names, microservice names, retry counts, internal error codes not meant for clients)

If a message contains any of the above, it must be classified and replaced — never passed through, even partially.

## Classification order

For a given error (which may have a status code, an internal error code, and/or a message string), resolve the user-facing message in this order — first match wins:

1. **Structured field-validation template** — if the error carries a machine-readable field/key plus a reason (a validation error object, not just a free-text string — e.g. `{"field": "dob", "error": "invalid"}` or a list of such), build the message from the field-validation template instead of the static catalog. See "Templated field-validation messages" below. This is deliberately checked *before* pattern matching: it produces a more specific, more useful message than any fixed catalog entry could, by naming the actual field.
2. **Pattern match** — check the error's message/code against the `patterns` list (see `message-catalog.json`). Patterns are regex-ish keyword matches against technical content (e.g. "s3", "token", "timeout", "sql").
3. **Status-class fallback** — if no pattern matches, fall back to the message for the specific HTTP status code (or nearest equivalent for non-HTTP transports) from `defaults`.
4. **Generic fallback** — if there's no status code either, use `defaults.default`.

## Templated field-validation messages

A raw validation error like `"email is error"`, `"dob is invalid"`, or `{"field": "postal_code", "error": "invalid"}` isn't solved by the static pattern catalog — no fixed regex-to-message mapping can name the right field for every possible key. This is a template, built from two small, separately-maintained pieces:

### 1. Humanize the field key

- Convert `snake_case` / `camelCase` / `kebab-case` into Title Case with spaces as the mechanical default (`postal_code` → "Postal Code", `dateOfBirth` → "Date Of Birth").
- Prefer a small project-specific glossary over the mechanical result whenever mechanical splitting would stay unclear or come out wrong (`dob` → "Date of Birth", not "Dob"; `poc_email` → "Point of Contact Email", not "Poc Email"). Keep this glossary alongside the message catalog — see `references/field-labels.json` for the starting shape — keyed by raw field name, falling back to mechanical splitting for anything not listed.
- Nested/array paths: humanize each segment and express the path in plain language, never as the raw dotted/bracket path (`items[2].unit_price` → e.g. "Unit Price for item 3"). Pick one phrasing convention and apply it consistently across the project.

### 2. Add a format hint when the type is known — don't invent one when it isn't

When the field's expected type/format is actually known (from the API schema, the validator's declared type, or the project telling you), append it plainly: dates → "Expected format: DD-MMM-YYYY", email → "e.g. name@example.com", phone → "e.g. +91 98765 43210". Keep a small per-type hint table — see `references/format-hints.json` — and read the *actual* format the project's schema/validators expect rather than assuming one; date format especially varies per project (`DD-MMM-YYYY` vs `YYYY-MM-DD` vs `MM/DD/YYYY`).

If the type isn't known, don't guess a hint. `"<Label> has an invalid value. Please check it and try again."` is the correct fallback — it's still more useful than the generic 422 default because it names the field.

### 3. Multiple simultaneous field errors

A single request can fail validation on more than one field at once:

- Map every field error individually — don't just take the first and drop the rest.
- If the frontend can render a list, return the mapped errors as a structured array (`[{ field, message }, ...]`) rather than concatenating them into one paragraph; check what the frontend actually consumes before choosing a shape.
- Only cap/truncate into a single flat sentence (e.g. first 3, "...and N more") if the frontend genuinely can't render a list — prefer the full structured list otherwise.

### 4. Don't let this become a new leak vector

Field-level detail is safe *because* it's expressed entirely in the user's own vocabulary about their own input — see the golden-rule exception above. It stops being safe the instant it echoes something internal instead: the validator/library name, a raw regex, an internal enum value, a DB column name that differs from the field's public name, or a server-computed/internal-only field the user never actually submitted (a "validation failure" on a field the user can't see or edit is an internal bug, not user input to fix — route that through the normal error path instead of the template). Audit every mechanically-generated label against this before trusting it on an unfamiliar key.

## Tone rules

- Calm, first-person-plural ("we"), no jargon, no blaming the user unless it's clearly their input that needs fixing (validation, 4xx).
- One short sentence is enough. A second sentence may add a concrete next action ("Please try again", "Please sign in again", "Please check X and try again").
- Never say "error", "exception", "failed" in a way that sounds alarming without a next step. Always pair the problem with what to do about it.
- Do not apologize excessively or use exclamation marks.

## Correlation IDs

- Always log the raw error (message, stack, status, internal code) server-side together with a request/correlation ID (reuse one if the framework already generates one, e.g. a request-id middleware; otherwise generate a short one, e.g. a UUID or short hash, per request/error).
- When practical, surface that ID in the user-facing payload as a separate field (e.g. `correlationId`) — not appended into the message text — so the frontend can optionally show "Reference: ABC123" and support can look up the real error from logs.
- Never derive the correlation ID from, or embed, any sensitive value (token, user email, internal ID that leaks structure).

## Retryable flag

Each catalog entry may carry a `retryable` boolean so the frontend can decide whether to show a "Retry" button / auto-retry with backoff, independent of the message text itself.

## Debug/dev escape hatch (optional, off by default)

If the project already has an explicit debug/development flag (e.g. `NODE_ENV=development`, `DEBUG=true`, a feature flag), the mapper *may* additionally attach the raw technical detail under a clearly separate field (e.g. `debug: {...}`) that the frontend only renders in non-production builds. This must be opt-in and never the default — production responses always go through the full mapping above regardless of flags. If the project has no such flag already, don't invent one; just always map.

## Status-code defaults (portable baseline)

See `message-catalog.json` → `defaults`. Treat non-HTTP transports (gRPC codes, GraphQL error extensions, message-queue failures) by mapping their nearest HTTP-equivalent status class before falling back to `defaults.default`.

## Worked examples

| Raw error | Status | Classification path | Friendly message |
|---|---|---|---|
| `{"field": "dob", "error": "invalid"}` | 422 | field-validation template (known type: date) | "Date of Birth has an invalid value. Expected format: DD-MMM-YYYY." |
| `{"field": "poc_email", "error": "is error"}` | 422 | field-validation template (glossary label + known type: email) | "Point of Contact Email has an invalid value. e.g. name@example.com." |
| `"token is expired"` | 401 | pattern match (`token`/`session expired`) | "Your session isn't valid anymore. Please sign in again." |
| `Error: connect ECONNREFUSED 10.0.4.12:5432` | 500 | pattern match (`connection refused`/`ECONNREFUSED`) | "We're having trouble connecting to one of our services. Please try again shortly." |
| `{"code":"RATE_LIMITED","message":"Too many requests, retry after 30s"}` | 429 | pattern match (`rate limit`/`too many requests`) | "You're doing that a bit too fast. Please wait a moment and try again." |
| `ValidationError: "email" must be a valid email address` (free-text, no structured field) | 422 | pattern match (`validation`/`invalid field`) | "Some of the information provided isn't valid. Please review and try again." |
| `TypeError: Cannot read properties of undefined (reading 'id') at OrderService.js:142:18` | 500 | no pattern matches → status-class fallback | "Something went wrong on our end. Please try again in a little while." |
| `Error: unexpected condition` (thrown from a background job, no status code available) | — | no pattern, no status → generic fallback | "Something unexpected happened. Please try again, and contact support if the issue continues." |

Note the third row from the bottom: `ValidationError: "email" must be a valid email address` is *free text*, not a structured `{field, error}` object, so it doesn't qualify for the field-validation template (tier 1) — it falls to pattern matching (tier 2) instead, and gets the generic validation message. If that endpoint's error actually carries a structured field name, prefer fixing the endpoint to emit `{"field": "email", "error": "invalid_format"}` so it can get the more specific templated message instead.

The fourth row is the important one: a bare `TypeError` with a file path and line number in it doesn't match any known pattern, and that's fine — it must never be passed through as-is just because nothing recognized it. Falling through to the plain status-class default is exactly the safe behavior; the mapper's job is to guarantee *something* sanitized comes out the other end, never to guess at a message when it isn't confident, and never to leak the raw text as a "better than nothing" fallback.

## Jargon is a defect even when nothing "leaked"

A message can pass every rule above — no stack trace, no vendor name, no raw exception text — and still fail the golden rule, because it's written in the system's internal vocabulary instead of the user's. This is a distinct failure mode from a leak and needs its own check: **read the message as the least technical person who will ever see it, not as whether it technically passed the blocklist.**

Common offenders, found repeatedly in real audits: `access token`, `refresh token`, `JWT`, `cookie` (as in "no refresh token cookie present"), `session` used as backend jargon rather than the user's own plain notion of "being signed in", internal saga/workflow/queue vocabulary (`saga`, `step function`, `compensation`, `reservation`), internal record vocabulary exposed instead of translated (`beneficiary`, `DPA account`, `constraint`), and raw UPPER_SNAKE_CASE status enums dropped into a sentence.

Rule of thumb: if fixing the message means swapping a technical noun for a plain-English one and nothing else, it's still in scope for this skill — "Access token has expired. Please refresh." → "Your session has expired. Please sign in again." is exactly the kind of fix this skill exists to make, not just stripping stack traces.

**Audience matters.** A message returned by a genuinely machine-to-machine or partner-technical endpoint (e.g. a bank's backend calling a B2B integration API with documented error codes) can reasonably stay closer to technical vocabulary, because the reader is another engineer working from an API contract, not an end user in a UI. When auditing, judge each endpoint by who actually reads the response — a browser/mobile client rendering the message to a person always gets the plain-language rule; a partner-to-partner API contract may not. State this judgment explicitly rather than silently applying one rule to everything.

## Tracing leaks through stored state, not just the try/except block

A leak doesn't have to happen at the moment of the exception. Audit for this pattern: an exception is caught, `str(exc)` is stored on a model field or dict (often meant for logging or an internal error column), and *that stored value is later serialized into a different, unrelated read endpoint's response* — e.g. a background job or saga step writes `record.error = {"message": str(exc)}`, and a status/detail endpoint elsewhere in the codebase does `"error": record.error` in its serializer. The write site looks innocuous (it's just persisting for the audit trail); the read site is where the actual leak reaches the client. When auditing an existing codebase, grep for where any `error`/`lastError`/`detail` field that stores `str(exc)` is later read back into a response, not just where exceptions are raised.

## Auditing an existing codebase: don't stop at grep hits

Grepping for `str(e)`, `f"...{exc}..."`, vendor names, etc. finds the mechanical leaks but misses jargon (nothing pattern-matches "access token" as wrong — it's a real word, not a stack trace) and misses the stored-then-serialized leak above. A thorough audit reads every `raise SomeException(...)` call site *and* every custom exception class's default message (e.g. a `default_detail` on a Django REST Framework `APIException` subclass, a default `message` on a custom error class) in full, end to end per service/module, rather than sampling grep hits — a hardcoded default on a shared exception class is a single point of leverage (fix it once, every raiser inherits the fix) but is also easy to skip if the audit only chases grep matches inside view functions.
