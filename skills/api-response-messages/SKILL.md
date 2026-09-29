---
name: api-response-messages
description: Use when the user wants any API response message — error output or success/confirmation messages — turned into short, calm, user-safe wording fit for a frontend, wants success/confirmation messages made consistent instead of ad hoc per-endpoint wording, asks to sanitize error responses, hide technical detail (stack traces, stringified JSON, internal service/vendor names like S3 or AWS, SQL, tokens, file paths) from end users, add a central error-message mapping layer, or audit/check/review whether an existing codebase's error or success response messages are aligned with user-friendly wording (with or without also fixing them). Triggers on "user friendly error message", "api response message", "sanitize error", "hide technical error details", "error message mapping", "friendly error", "don't leak internal errors to frontend", "make errors user friendly", "success message", "confirmation message", "refactor error messages", "audit error handling", "audit api responses", "check message alignment", "check if our responses are aligned".
---

# API response message handling

Two related jobs, both driven by catalogs instead of scattered literal strings:

- **Errors** — turn raw/technical error output into short, calm, user-safe messages, and make sure the codebase actually returns them instead of the raw error.
- **Successes** — make confirmation messages ("Document uploaded successfully.") consistent in tone and wording across the codebase instead of ad hoc/missing per endpoint.

Both apply equally to a brand-new project and to an existing codebase that already has error/success handling.

Read `references/error-classification.md` (error rules, leak blocklist, tone, worked examples) and, if success messages are in scope for this request, `references/success-message-style.md` in full before doing anything else.

## Workflow

### 1. Detect the target project's stack (shared by both jobs)

Look for `package.json` (Node), `requirements.txt`/`pyproject.toml` (Python), `pom.xml`/`build.gradle` (Java/Kotlin), `go.mod` (Go), etc., and which web framework is in use (Express/Koa/Fastify/NestJS, FastAPI/Django/Flask, Spring, ...). If the project is empty or the stack is genuinely ambiguous, ask rather than guessing.

### 2. Determine what's being asked: audit vs. implement

- **Audit/check request** — the user wants to know whether existing response messages already align with these rules ("check if our error messages are aligned", "audit our API responses", "review our response messages", "does this look right?"). → Go to step 3 in **report-only mode**: no file is created or edited. Produce only the table in "Audit output format" below, then stop.
- **Implement/fix request** — the user wants the catalog/mapper/wiring actually built or fixed ("sanitize our errors", "add friendly error messages", "wire up a mapper", "refactor our error handling to use it"). → Go to step 3 in normal mode.
- **Ambiguous** — when it's genuinely unclear which is wanted, default to report-only mode first: it's cheap and reversible. Present the table, then ask whether to proceed with the edits, and to which rows.

This applies to a request scoped to errors, to successes, or to both — an audit request covers whatever's in scope for the request the same way an implement request would.

### Audit output format (report-only mode)

Produce exactly one Markdown table and nothing else — no catalog files, no mapper code, no wiring, no edits to the target codebase:

| file/section | current response message | suggested message |
|---|---|---|
| `src/routes/orders.ts — POST /orders catch block` | `res.json({ error: err.message })` (raw exception text) | "We're having trouble placing your order right now. Please try again shortly." |
| `OrderSerializer.validate_dob` (Django) | `"dob is invalid"` | "Date of Birth has an invalid value. Expected format: DD-MMM-YYYY." |
| `documents.py — upload_document view` | *(no success message returned)* | "Document uploaded successfully." |

Rules for filling it in:

- One row per distinct message location — a specific route/handler/exception class/serializer/default exception message — not one row per file if a file has several unrelated messages.
- `file/section`: a path plus enough context to find it without re-searching (file + function/handler/class name, not just the file).
- `current response message`: quote the literal string verbatim when there is one; when it's built from an expression rather than a literal, describe the expression (e.g. `str(exc)` passed straight through, or *(no message field on success)*).
- `suggested message`: the actual proposed replacement text, worked out the same way step 3 would (pattern match / status fallback / generic fallback for errors, per-action catalog entry for successes, or the field-validation template result for structured validation errors) — not a placeholder like "make this friendlier."
- After the table, briefly flag anything that needs a judgment call before implementing — e.g. an endpoint that looks genuinely partner/machine-facing and might be exempt from the plain-language rule (see "Audience matters" in `error-classification.md`), or a case where fixing the message would require a response-shape change.
- Then stop. Wait for the user to say which rows to act on (all, a subset, or none) before writing or editing any file.

### 3. Error messages (implement mode)

a. **Read** `references/error-classification.md` (rules) and `references/message-catalog.json` (portable starting catalog). If the target error is a structured field-validation error (has a field/key, not just free text), also read `references/field-labels.json` and `references/format-hints.json`. If the stack matches one of the worked examples (`references/example-node-express.md` for Node/Express, `references/example-python-fastapi.md` for FastAPI, `references/example-python-django.md` for Django/DRF), read that too; otherwise adapt the same shape (catalog file + pure mapper function + one central error boundary) to whatever framework is actually there.

b. **Seed a message catalog** in the target project, in whatever format its ecosystem prefers (JSON/YAML/a dict literal) — start from `references/message-catalog.json`, then fold in any project-specific cases the user mentions. If the project has structured field-validation errors, also seed a field-labels glossary (from `references/field-labels.json`) and a format-hints table (from `references/format-hints.json`, corrected to the project's actual expected formats — don't assume the placeholder date format). Keep these as catalog data, not logic, so they're easy to extend later without touching code.

c. **Write a pure mapper function** (`mapErrorToUserMessage` / `map_error_to_user_message` / equivalent) implementing the classification order from the rules doc: field-validation template (when the error is structured) → pattern match → status-class default → generic fallback. Takes the raw error (message/status/code, or field/reason when structured) and a correlation ID, returns `{ message, retryable, correlationId }`. No I/O, no framework types — easy to unit test.

d. **Wire it into exactly one central error boundary** for the framework in use (Express error middleware, FastAPI exception handlers, Spring `@ControllerAdvice`, Django middleware, a Go handler wrapper, ...). That boundary: logs the *raw* error server-side (via whatever logger the project already uses) together with a correlation ID, then returns only the mapped message + correlation ID (+ status/retryable) to the client. Prefer one funnel over scattering the mapper across every route/controller.

e. **On an existing codebase, scan for and refactor existing leaks** instead of leaving them: grep for a caught exception's `.message`/`str(e)`/`.toString()` sent straight into a response, `JSON.stringify(err)` / stringified error objects returned to callers, stack traces or exception class names in response bodies, and ad hoc `res.json({ error: ... })` / `return {"error": ...}` calls scattered per-route instead of going through the central boundary. Route each one through the mapper (or have it propagate to the central handler via `next(err)` / re-raise / equivalent) instead of hand-rolling sanitization per call site.

### 4. Success messages (implement mode, only when in scope for the request)

a. **Read** `references/success-message-style.md` and `references/success-message-catalog.json`.

b. **Seed a success-message catalog** in the target project (same format convention as the error catalog), keyed by action (`document.upload`, `profile.update`, ...) — start from `references/success-message-catalog.json`, extend with the project's actual actions.

c. **Write a small lookup helper** (`getSuccessMessage(actionKey)` or equivalent) that reads the catalog and falls back to `default` for an unknown key.

d. **On an existing codebase, collect and refactor existing success strings**: find the literal confirmation messages currently scattered across route handlers/controllers/serializers, normalize wording/tense/punctuation per the style doc, de-duplicate near-identical messages for the same action into one catalog entry, then replace the inline literals with lookups. Don't change response data payloads or status codes — only the message text — unless the user asked for a contract change.

### 5. Verify and report (implement mode, both jobs)

a. **Verify.** Run the project's existing lint/typecheck/tests if configured. Show a few before/after examples (errors and, if in scope, successes) so the person can see the transformation.

b. **If a rewiring or refactor would touch more than a handful of files**, summarize the plan (files + change) and confirm before applying it broadly — this applies to both the error leak-scan and the success-message refactor. If step 2 already produced an audit table for this request, this is just the user confirming which rows to act on; don't re-derive the plan from scratch.

c. **Report back**: catalog location(s), mapper/helper location(s), where each is wired in, which files were rewired/refactored, and anything found but intentionally left alone (e.g. internal admin/debug endpoints, or a success response with no message by existing convention) with the reason why.

## Non-negotiables

- On an audit/check request, produce the table and stop — never start editing the codebase as part of answering "is this aligned?"
- Never invent a status code or swallow an error silently — every error still gets logged in full server-side.
- Never let a pattern-matched keyword override context when it would mislead the user about what to do (see the classification-order note and worked examples in `error-classification.md`).
- Don't add a debug/raw-detail escape hatch unless the project already has an established debug/dev flag; see the rules doc.
- Don't change a success response's data shape or status code while "fixing" its message, and don't add message fields to endpoints that don't have them today, without asking first.
