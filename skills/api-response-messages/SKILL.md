---
name: api-response-messages
description: Use when the user wants any API response message — error output or success/confirmation messages — turned into short, calm, user-safe wording fit for a frontend, wants success/confirmation messages made consistent instead of ad hoc per-endpoint wording, asks to sanitize error responses, hide technical detail (stack traces, stringified JSON, internal service/vendor names like S3 or AWS, SQL, tokens, file paths) from end users, add a central error-message mapping layer, or audit/refactor an existing codebase's error or success response messages. Triggers on "user friendly error message", "api response message", "sanitize error", "hide technical error details", "error message mapping", "friendly error", "don't leak internal errors to frontend", "make errors user friendly", "success message", "confirmation message", "refactor error messages", "audit error handling", "audit api responses".
---

# API response message handling

Two related jobs, both driven by catalogs instead of scattered literal strings:

- **Errors** — turn raw/technical error output into short, calm, user-safe messages, and make sure the codebase actually returns them instead of the raw error.
- **Successes** — make confirmation messages ("Document uploaded successfully.") consistent in tone and wording across the codebase instead of ad hoc/missing per endpoint.

Both apply equally to a brand-new project and to an existing codebase that already has error/success handling — in the existing-codebase case, this skill audits what's there and refactors it, not just wires up new endpoints.

Read `references/error-classification.md` (error rules, leak blocklist, tone, worked examples) and, if success messages are in scope for this request, `references/success-message-style.md` in full before generating or editing anything.

## Workflow

### 1. Detect the target project's stack (shared by both jobs)

Look for `package.json` (Node), `requirements.txt`/`pyproject.toml` (Python), `pom.xml`/`build.gradle` (Java/Kotlin), `go.mod` (Go), etc., and which web framework is in use (Express/Koa/Fastify/NestJS, FastAPI/Django/Flask, Spring, ...). If the project is empty or the stack is genuinely ambiguous, ask rather than guessing.

### 2. Error messages

a. **Read** `references/error-classification.md` (rules) and `references/message-catalog.json` (portable starting catalog). If the stack matches one of the worked examples (`references/example-node-express.md`, `references/example-python-fastapi.md`), read that too; otherwise adapt the same shape (catalog file + pure mapper function + one central error boundary) to whatever framework is actually there.

b. **Seed a message catalog** in the target project, in whatever format its ecosystem prefers (JSON/YAML/a dict literal) — start from `references/message-catalog.json`, then fold in any project-specific cases the user mentions. Keep this catalog data, not logic, so it's easy to extend later without touching code.

c. **Write a pure mapper function** (`mapErrorToUserMessage` / `map_error_to_user_message` / equivalent) implementing the classification order from the rules doc: pattern match → status-class default → generic fallback. Takes the raw error (message/status/code) and a correlation ID, returns `{ message, retryable, correlationId }`. No I/O, no framework types — easy to unit test.

d. **Wire it into exactly one central error boundary** for the framework in use (Express error middleware, FastAPI exception handlers, Spring `@ControllerAdvice`, Django middleware, a Go handler wrapper, ...). That boundary: logs the *raw* error server-side (via whatever logger the project already uses) together with a correlation ID, then returns only the mapped message + correlation ID (+ status/retryable) to the client. Prefer one funnel over scattering the mapper across every route/controller.

e. **On an existing codebase, scan for and refactor existing leaks** instead of leaving them: grep for a caught exception's `.message`/`str(e)`/`.toString()` sent straight into a response, `JSON.stringify(err)` / stringified error objects returned to callers, stack traces or exception class names in response bodies, and ad hoc `res.json({ error: ... })` / `return {"error": ...}` calls scattered per-route instead of going through the central boundary. Route each one through the mapper (or have it propagate to the central handler via `next(err)` / re-raise / equivalent) instead of hand-rolling sanitization per call site.

### 3. Success messages (only when in scope for the request)

a. **Read** `references/success-message-style.md` and `references/success-message-catalog.json`.

b. **Seed a success-message catalog** in the target project (same format convention as the error catalog), keyed by action (`document.upload`, `profile.update`, ...) — start from `references/success-message-catalog.json`, extend with the project's actual actions.

c. **Write a small lookup helper** (`getSuccessMessage(actionKey)` or equivalent) that reads the catalog and falls back to `default` for an unknown key.

d. **On an existing codebase, collect and refactor existing success strings**: find the literal confirmation messages currently scattered across route handlers/controllers/serializers, normalize wording/tense/punctuation per the style doc, de-duplicate near-identical messages for the same action into one catalog entry, then replace the inline literals with lookups. Don't change response data payloads or status codes — only the message text — unless the user asked for a contract change.

### 4. Verify and report (both jobs)

a. **Verify.** Run the project's existing lint/typecheck/tests if configured. Show a few before/after examples (errors and, if in scope, successes) so the person can see the transformation.

b. **If a rewiring or refactor would touch more than a handful of files**, summarize the plan (files + change) and confirm before applying it broadly — this applies to both the error leak-scan and the success-message refactor.

c. **Report back**: catalog location(s), mapper/helper location(s), where each is wired in, which files were rewired/refactored, and anything found but intentionally left alone (e.g. internal admin/debug endpoints, or a success response with no message by existing convention) with the reason why.

## Non-negotiables

- Never invent a status code or swallow an error silently — every error still gets logged in full server-side.
- Never let a pattern-matched keyword override context when it would mislead the user about what to do (see the classification-order note and worked examples in `error-classification.md`).
- Don't add a debug/raw-detail escape hatch unless the project already has an established debug/dev flag; see the rules doc.
- Don't change a success response's data shape or status code while "fixing" its message, and don't add message fields to endpoints that don't have them today, without asking first.
