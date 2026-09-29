# Example integration pattern: Node.js / Express (TypeScript)

This is one worked example of the general pattern described in SKILL.md. Adapt the same shape (catalog file + pure mapper function + single central error boundary) to whatever framework the target project actually uses (Koa, Fastify, NestJS, Hapi, ...) rather than copying this verbatim.

## Files

```
src/errors/message-catalog.json     # seeded from references/message-catalog.json, extended per project
src/errors/mapError.ts              # pure function, no I/O
src/middleware/errorHandler.ts      # central Express error middleware (app.use, 4 args, registered last)
```

## `src/errors/mapError.ts`

```ts
import catalog from "./message-catalog.json";

export interface MappedError {
  message: string;
  retryable: boolean;
  correlationId: string;
}

export function mapErrorToUserMessage(
  rawMessage: string | undefined,
  status: number | undefined,
  correlationId: string,
): MappedError {
  const haystack = rawMessage ?? "";

  for (const rule of catalog.patterns) {
    if (new RegExp(rule.match).test(haystack)) {
      return { message: rule.userMessage, retryable: rule.retryable, correlationId };
    }
  }

  const byStatus = status != null ? catalog.defaults[String(status) as keyof typeof catalog.defaults] : undefined;
  return {
    message: byStatus ?? catalog.defaults.default,
    retryable: status != null && status >= 500,
    correlationId,
  };
}
```

## `src/middleware/errorHandler.ts`

```ts
import { Request, Response, NextFunction } from "express";
import { randomUUID } from "crypto";
import { mapErrorToUserMessage } from "../errors/mapError";
import { logger } from "../logger"; // use the project's existing logger

export function errorHandler(err: any, req: Request, res: Response, _next: NextFunction) {
  const correlationId = req.headers["x-correlation-id"]?.toString() ?? randomUUID();
  const status = err.status ?? err.statusCode ?? 500;

  // Full detail server-side only.
  logger.error({ err, correlationId, path: req.path }, "request failed");

  const { message, retryable } = mapErrorToUserMessage(err.message, status, correlationId);

  res.status(status).json({ error: message, retryable, correlationId });
}
```

Register it **last**, after all routes: `app.use(errorHandler)`.

## Rewiring existing leak points

Search for and replace patterns like:

```ts
// before — leaks err.message / err.stack straight to the client
res.status(500).json({ error: err.message });
res.status(500).send(err.stack);
res.json({ error: JSON.stringify(err) });
```

with either: (a) `next(err)` so it flows to the central `errorHandler`, or (b) a direct call to `mapErrorToUserMessage` if the route needs a custom status first. Prefer (a) — one funnel point is easier to keep correct than many call sites each remembering to sanitize.
