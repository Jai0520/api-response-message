# Error classification rules

This is the rulebook the mapper logic must implement, in any language. Read it in full before generating or editing any code.

## Golden rule

The user sees **why** (briefly, safely) and **what to do next**. The user never sees **how it broke internally**.

Raw errors go to the server-side log (with full detail + a correlation ID). Only the mapped, friendly message + correlation ID goes to the client/frontend.

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

1. **Pattern match** — check the error's message/code against the `patterns` list (see `message-catalog.json`). Patterns are regex-ish keyword matches against technical content (e.g. "s3", "token", "timeout", "sql").
2. **Status-class fallback** — if no pattern matches, fall back to the message for the specific HTTP status code (or nearest equivalent for non-HTTP transports) from `defaults`.
3. **Generic fallback** — if there's no status code either, use `defaults.default`.

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
| `Error: connect ECONNREFUSED 10.0.4.12:5432` | 500 | pattern match (`connection refused`/`ECONNREFUSED`) | "We're having trouble connecting to one of our services. Please try again shortly." |
| `{"code":"RATE_LIMITED","message":"Too many requests, retry after 30s"}` | 429 | pattern match (`rate limit`/`too many requests`) | "You're doing that a bit too fast. Please wait a moment and try again." |
| `ValidationError: "email" must be a valid email address` | 422 | pattern match (`validation`/`invalid field`) | "Some of the information provided isn't valid. Please review and try again." |
| `TypeError: Cannot read properties of undefined (reading 'id') at OrderService.js:142:18` | 500 | no pattern matches → status-class fallback | "Something went wrong on our end. Please try again in a little while." |
| `Error: unexpected condition` (thrown from a background job, no status code available) | — | no pattern, no status → generic fallback | "Something unexpected happened. Please try again, and contact support if the issue continues." |

The fourth row is the important one: a bare `TypeError` with a file path and line number in it doesn't match any known pattern, and that's fine — it must never be passed through as-is just because nothing recognized it. Falling through to the plain status-class default is exactly the safe behavior; the mapper's job is to guarantee *something* sanitized comes out the other end, never to guess at a message when it isn't confident, and never to leak the raw text as a "better than nothing" fallback.
