# Success / confirmation message style

Rules for the messages shown after an action actually succeeds — created, updated, deleted, uploaded, submitted, paid, etc. These are a different problem from error messages: there's usually nothing sensitive to hide, the issue is inconsistency (some endpoints return a message, some don't; wording, tense and tone vary; internal field names leak through even on the happy path).

## Golden rule

Confirm what happened, briefly, in the same calm voice as the error catalog (see `error-classification.md`'s tone rules) — first-person-plural where it reads naturally, no jargon, no restating internal identifiers or storage details even though nothing failed.

## Structure

`<what the user did> + <what happened>`, one short sentence, past tense:

- "Document uploaded successfully."
- "Your changes have been saved."
- "Your password has been updated."
- "Your order has been placed."
- "Payment received — your order is confirmed."

Avoid:
- Restating technical internals on success just because they're harmless (`"Uploaded to s3://my-bucket/uploads/f3a9.pdf"` → say `"Document uploaded successfully."` instead).
- Mixing tone with the error catalog — if errors are calm and plain, success messages shouldn't suddenly turn chatty/exclamatory ("Yay! All done!! 🎉") unless the project's existing voice already does that everywhere.
- A different sentence shape per endpoint for the same kind of action (don't let `POST /documents` say "Document uploaded successfully." while `POST /images` says "Image has been successfully uploaded by you.").

## Catalog-driven, like errors

Success messages should live in one catalog keyed by **action**, not be classified/pattern-matched like errors (there's nothing to detect — the caller already knows which action just happened). See `references/success-message-catalog.json` for the shape: `{ "action.key": "message" }`.

Look up by action key (`document.upload`, `profile.update`, ...) rather than scattering literal strings across route handlers, so wording can be changed centrally and stays consistent.

## When a success message is warranted

- User-triggered mutating actions (create/update/delete/upload/submit/pay/verify) generally deserve one.
- Plain reads (`GET` list/detail endpoints) usually don't need one — don't invent messages where the UI wouldn't show them.
- If the project already returns data-only success responses by convention (e.g. just the created resource, no message field) and the user hasn't asked to add message fields everywhere, don't add them uninvited — ask first, since that's a response-shape/API-contract change, not just a wording fix.

## Refactoring existing success messages

When auditing an existing codebase (not just wiring new endpoints):
1. Collect the existing success/confirmation strings per action.
2. Normalize wording/tense/punctuation against the rules above, and de-duplicate near-identical messages for the same action into one catalog entry.
3. Replace the inline literals with catalog lookups.
4. Don't change the response's data payload or HTTP status — only the message text — unless asked to change the contract itself.
