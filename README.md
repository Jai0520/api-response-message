# api-response-messages

A Claude Code plugin containing one skill — `api-response-messages` — that:

- turns raw, technical API **error** output into short, calm, user-safe messages fit for a frontend, and wires that translation into your codebase;
- makes **success/confirmation** messages ("Document uploaded successfully.") consistent instead of ad hoc or missing per endpoint;
- works on a brand-new project or as an **audit/refactor pass over an existing codebase** — it finds current error- and success-message handling and rewires it, not just new endpoints.

Error example:

| Raw | Friendly |
|---|---|
| `Error: connect ECONNREFUSED 10.0.4.12:5432` (status 500) | "We're having trouble connecting to one of our services. Please try again shortly." |
| `{"code":"RATE_LIMITED","message":"Too many requests, retry after 30s"}` (status 429) | "You're doing that a bit too fast. Please wait a moment and try again." |
| `TypeError: Cannot read properties of undefined (reading 'id') at OrderService.js:142:18` (status 500) | "Something went wrong on our end. Please try again in a little while." |

Success example:

| Before | After |
|---|---|
| (no message field, or inconsistent per endpoint) | "Document uploaded successfully." |

It's language/framework-agnostic: the rules and starting catalogs in `skills/api-response-messages/references/` are portable, and the skill detects whatever stack it's invoked in (Node/Express, Python/FastAPI, or anything else) to generate the actual mapper/helper + wiring for that project.

## What it does when invoked

**Errors:**
1. Detects the target project's stack.
2. Seeds a message catalog (pattern → friendly message, plus per-status-code defaults) in that project.
3. Generates a small pure mapper function implementing the classification rules.
4. Wires it into a single central error-handling boundary for that framework (raw error logged server-side with a correlation ID; only the friendly message + correlation ID goes to the client).
5. On an existing codebase, scans for and refactors existing places that leak raw error text/JSON/stack traces to the client, routing them through the new layer.

**Successes:**
1. Seeds a success-message catalog keyed by action (`document.upload`, `profile.update`, ...).
2. Generates a small lookup helper.
3. On an existing codebase, collects existing confirmation strings, normalizes wording/tense/punctuation, de-duplicates, and replaces inline literals with catalog lookups — without changing response data shape or status codes.

Full rules live in [`skills/api-response-messages/references/error-classification.md`](skills/api-response-messages/references/error-classification.md) (errors) and [`skills/api-response-messages/references/success-message-style.md`](skills/api-response-messages/references/success-message-style.md) (successes).

## Using it

**As a plugin (recommended for reuse across projects):**

```
/plugin marketplace add Jai0520/api-response-message
/plugin install api-response-messages
```

(A full git URL also works: `/plugin marketplace add https://github.com/Jai0520/api-response-message.git`. This repo is set up as a single-plugin marketplace via `.claude-plugin/marketplace.json`, which is what `/plugin marketplace add` actually reads.)

Then in any project: *"use the api-response-messages skill to sanitize our API error responses"* or *"use api-response-messages to audit and refactor our existing error and success messages"*.

**As a project-level skill (no plugin install needed):**

Copy `skills/api-response-messages/` into the target project's `.claude/skills/` directory. Claude Code will pick it up automatically for that project.
