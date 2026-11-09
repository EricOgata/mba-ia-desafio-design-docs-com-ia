# Component Deep Analysis Report

**Component:** Request Logger Middleware
**Primary file:** `src/middlewares/request-logger.middleware.ts`
**Project:** `order-management-api` (`mba-ia-desafio-design-docs-com-ia`)
**Analysis scope:** entire project root
**Excluded from analysis:** `node_modules`, `.git`, `dist`, `coverage`
**Analysis date:** 2026-11-08 23:09:07
**Analyzed revision:** `e7f6311` (working tree)

---

## 1. Executive Summary

The Request Logger Middleware is a 28-line Express middleware that implements the application's request correlation and HTTP access-logging contract. It resolves a request identifier (forwarding a client-supplied `x-request-id` or generating a UUID v4), attaches it to `req.id`, echoes it back in the `X-Request-Id` response header, starts a monotonic high-resolution timer, and registers a one-shot listener on the response `finish` event. When the response completes, it emits one structured Pino `info` record named `http_request` containing `requestId`, `method`, `path`, `statusCode`, `durationMs` and the authenticated `userId`.

Architecturally the component is a globally mounted, cross-cutting observability unit registered once in the composition root (`src/app.ts:60`). It sits between the JSON body parser and the API router, so it wraps the health endpoint, every `/api/v1` module route, the 404 fallback and the terminal error handler. It owns no domain logic, performs no persistence, and never short-circuits the pipeline: its only control-flow action is a synchronous `next()`.

Key findings:

1. **The component defines the system's correlation identifier.** `req.id` is consumed by `src/middlewares/error.middleware.ts:56-60` for error-log correlation and by the access-log record itself; the `X-Request-Id` response header is the client-facing half of the same contract.
2. **The identifier is trusted, not validated.** Any client-supplied value is accepted verbatim (no format, length or character validation) and echoed back in the response and in logs (`src/middlewares/request-logger.middleware.ts:6-8`). This enables end-to-end tracing from callers but also permits correlation spoofing, log forging with misleading IDs, and unbounded header values.
3. **Logging is deferred to `finish`, giving access to the final status code and to `req.user`** populated later by `authenticate` inside module routers. The trade-off is that aborted/closed connections never produce an access log.
4. **Placement in the pipeline determines coverage.** Because `express.json({ limit: '1mb' })` is registered *before* the logger (`src/app.ts:59-60`), requests rejected by the body parser (malformed JSON, payload over 1 MB) receive no request ID, no `X-Request-Id` header and no access log.
5. **Zero direct test coverage.** The middleware executes in all 15 integration tests through `tests/helpers/factories.ts:12` -> `buildApp`, but no test asserts the response header, ID generation/forwarding, log emission, log fields or duration. There is also no unit test and no coverage provider configured in `vitest.config.ts`.
6. **`path` is logged as `req.originalUrl`, including the query string, and is not covered by Pino redaction** (`src/shared/logger/index.ts:4-11`), so search terms, e-mails or document numbers passed as query parameters are persisted in logs.
7. **The component is a custom replacement for `pino-http`.** `pino-http@10.3.0` is declared in `package.json:31` but is never imported; this file is the only HTTP logger.

---

## 2. Data Flow Analysis

```
1. Client (or upstream gateway) sends an HTTP request, optionally with
   header "x-request-id: <value>"
2. Express app: app.disable('x-powered-by')                    (src/app.ts:58)
3. express.json({ limit: '1mb' }) parses the body              (src/app.ts:59)
   -> on parse failure the request never reaches the logger
4. requestLogger middleware executes                           (src/app.ts:60 -> middleware:5)
   4a. requestId = req.headers['x-request-id'] ?? uuidv4()     (middleware:6)
   4b. req.id = requestId                                      (middleware:7)
   4c. res.setHeader('X-Request-Id', requestId)                (middleware:8)
   4d. startedAt = process.hrtime.bigint()                     (middleware:10)
   4e. res.on('finish', <handler>) registered                  (middleware:12)
   4f. next()                                                  (middleware:27)
5. Downstream pipeline: /health or /api/v1 router
   -> [authenticate] may populate req.user = { id, email, role }
   -> [requireRole], validate, controller, service, repository, Prisma/MySQL
   -> controller writes status code and JSON body
6. Response headers are flushed to the socket (transport layer)
7. Express emits 'finish' on the ServerResponse
8. Listener executes:
   8a. durationMs = Number(hrtime.bigint() - startedAt) / 1e6  (middleware:13)
   8b. logger.info({ requestId, method, path, statusCode,
                    durationMs, userId }, 'http_request')      (middleware:14-24)
9. Pino serializes to JSON (or pino-pretty in development),
   applying LOG_LEVEL filtering and redaction paths
10. Client receives the response including the X-Request-Id header
11. On error paths, errorMiddleware separately reads req.id        (error.middleware.ts:56)
    and emits logger.error({ err, requestId, method, path },
    'Unhandled error in request')                                  (error.middleware.ts:57-60)
```

Ordering facts that shape the flow:

- The middleware runs for `GET /health`, every `/api/v1` route, the catch-all 404 handler and any handler that responds normally. It does not run for routes registered before it (none exist) or for failures thrown by `express.json`.
- The access log is emitted *after* `errorMiddleware` has produced its (masked) error response for handled errors: `errorMiddleware` sends the response, and the `finish` event fires afterwards. Unhandled-error logging and access logging therefore appear as two correlated records sharing `requestId`.
- `userId` is read at `finish` time, not at middleware-entry time, so authenticated routes always report the identity even though `req.user` did not exist when the logger started.

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Validation / Identity | A request ID is resolved as the inbound `x-request-id` header value when present, otherwise a UUID v4 is generated; no format, length or content validation is applied | `src/middlewares/request-logger.middleware.ts:6` |
| Contract / Correlation | The resolved request ID is stored on `req.id` for the remainder of the request lifecycle | `src/middlewares/request-logger.middleware.ts:7` |
| Protocol / Contract | The resolved request ID is echoed to the client in the `X-Request-Id` response header before the handler runs | `src/middlewares/request-logger.middleware.ts:8` |
| Timing | Request duration is measured with a monotonic clock from middleware entry to response `finish` and normalized to a number with two decimal places | `src/middlewares/request-logger.middleware.ts:10,13,20` |
| Observability | Exactly one structured access-log record named `http_request` is emitted per completed response, at `info` level | `src/middlewares/request-logger.middleware.ts:12-25` |
| Log Schema | The access record contains `requestId`, `method`, `path` (original URL including query string), `statusCode`, `durationMs` and `userId` | `src/middlewares/request-logger.middleware.ts:14-24` |
| Enrichment | `userId` is populated from `req.user.id` when authentication has run; it is absent for anonymous requests | `src/middlewares/request-logger.middleware.ts:21` |
| Pipeline Control | The middleware must be transparent: no response writing, no short-circuit, synchronous `next()` after registration | `src/middlewares/request-logger.middleware.ts:27` |
| Failure Tolerance | Logging is best-effort and deferred; failed/aborted responses that never emit `finish` produce no access log and logging failures do not affect the response | `src/middlewares/request-logger.middleware.ts:12` |
| Configuration | Access-log emission is gated by the global Pino level derived from `LOG_LEVEL` (default `info`); `LOG_LEVEL=warn` or `error` silently suppresses access logs | `src/shared/logger/index.ts:15`, `src/config/env.ts:6` |
| Security / Redaction | Access logs must never contain credentials: the record includes only curated fields, and the logger additionally redacts authorization/cookie/password/token paths | `src/shared/logger/index.ts:4-11`, `src/middlewares/request-logger.middleware.ts:14-24` |

Confidence note: the rules above are directly evidenced in source except for the *intent* behind "trust the inbound header" (no comment or documentation states the rationale; inferred from implementation) and the "best-effort" tolerance (no `try/catch` exists; the property follows from how Pino and the `finish` listener behave).

---

### Business Rule: Request ID resolution (forward or generate)

**Overview**:
Every HTTP request that reaches the middleware is assigned a correlation identifier. The inbound `x-request-id` header wins if it is defined; otherwise the middleware generates a UUID v4.

**Detailed description**:
Implementation is a single nullish-coalescing expression at `src/middlewares/request-logger.middleware.ts:6`: the value at `req.headers['x-request-id']` is cast to `string | undefined` and used if it is neither `null` nor `undefined`, otherwise `uuidv4()` from `uuid@11.0.3` produces a random version-4 identifier. The rule exists to support two distinct use cases. First, distributed or gateway-fronted deployments: an upstream proxy, load balancer, mobile SDK or calling service can inject its own correlation ID so that a single trace spans systems; the middleware preserves it rather than replacing it. Second, standalone operation: a caller that supplies nothing still gets a stable identifier that ties the response, the access log and the error log together.

The rule is deliberately permissive. There is no UUID-format assertion, no maximum length, no character whitelist, no trimming and no normalization. Whatever the client sends becomes the application's correlation ID for that request. This makes the contract interoperable with any upstream ID scheme (ULID, trace IDs, numeric ticket IDs) but transfers trust to the caller. Because the same value is written into JSON logs and into a response header, a hostile caller can choose misleading identifiers (for example, reusing the ID of another request) to pollute or confuse log analysis, and can send an arbitrarily long value that inflates every log line and the response header for that request.

A subtle consequence of using `??` rather than `||` is that an **empty-string** header value is accepted: `x-request-id: ` (present but empty) yields `requestId === ''`, which is then stored in `req.id`, echoed as an empty `X-Request-Id` header, and logged as an empty string. Only an absent header triggers generation. The cast to `string | undefined` additionally hides the fact that Node's `IncomingHttpHeaders` types allow `string[]`; for duplicate `x-request-id` headers Node normally joins values with `", "` before Express sees them, but the type assertion means no runtime guard exists.

**Rule workflow**:
```
request enters middleware
  -> read req.headers['x-request-id']
  -> value !== null && value !== undefined ?
       YES -> requestId = value                (any string, incl. "", any length)
       NO  -> requestId = uuidv4()             (random v4 UUID)
  -> no format / length / sanitization check
  -> requestId flows to BR-02 (req.id), BR-03 (response header), BR-05/06 (log)
```

---

### Business Rule: Correlation identifier attached to the request (`req.id`)

**Overview**:
The resolved identifier is published on the Express `Request` object as `req.id` so that every downstream layer can correlate its own diagnostics with the same request.

**Detailed description**:
At `src/middlewares/request-logger.middleware.ts:7` the middleware assigns `req.id = requestId`. The property is not declared in this file: the TypeScript declaration merge lives in `src/middlewares/auth.middleware.ts:12-17` (`interface Request { user?: AuthUser; id?: string }`), which means the compile-time contract for a property owned by the request logger is hosted by the authentication middleware. At runtime this is a plain property assignment on Express's request object; there is no getter, no immutability, and no validation of writes by later code.

The rule's purpose is correlation across log sites. `src/middlewares/error.middleware.ts:56-60` reads `(req as { id?: string }).id ?? 'unknown'` and includes it in the `logger.error` record for every unclassified failure, so an operator can join the 500-level error log to the corresponding `http_request` access log. The request logger itself reads the local `requestId` variable (not `req.id`) for its own record, so a downstream mutation of `req.id` would make the error log disagree with the access log; nothing prevents such a mutation, but no code performs one.

Ownership is therefore split three ways: written by the request logger, typed by the auth middleware, consumed by the error middleware. This is functionally correct because the logger is mounted globally and before any consumer, but it means the `req.id` contract cannot be discovered from the owning file alone, and any future reordering that puts the error middleware ahead of the logger degrades the field to `'unknown'`.

**Rule workflow**:
```
requestId resolved (BR-01)
  -> req.id = requestId
  -> downstream code may read req.id at any time
     - errorMiddleware: requestId ?? 'unknown' for error logs
     - custom handlers / future instrumentation
  -> contract: req.id is a string; property is optional in the type system
```

---

### Business Rule: Request ID echoed in the `X-Request-Id` response header

**Overview**:
The same identifier is written to the response as `X-Request-Id` before the route handler executes, so the client can quote it in support requests and chain it into subsequent calls.

**Detailed description**:
`res.setHeader('X-Request-Id', requestId)` at `src/middlewares/request-logger.middleware.ts:8` sets the header immediately after resolving the ID, i.e. before any handler runs and before the response is committed. The header is therefore present on every response produced after the middleware, including 401/403/404/409/422/500 responses generated by `authenticate`, `requireRole`, validation, the 404 fallback and `errorMiddleware`, because those paths still flow through the same `ServerResponse`. It is *not* present on responses produced before the middleware — specifically body-parser rejections raised by `express.json` at `src/app.ts:59`.

The header serves as the client-visible half of the correlation contract. A caller that received `X-Request-Id: abc` can include `x-request-id: abc` on a retry or a follow-up call, and the middleware will reuse it (BR-01), producing contiguous log records. Because the value is client-controlled, the response header is effectively a reflection of arbitrary caller input. Node's `ServerResponse.setHeader` validates header values and throws `ERR_INVALID_CHAR` if the value contains invalid characters (for example CR/LF), which prevents response splitting; however, since the call is synchronous inside the middleware, such a throw is caught by Express's layer handling and routed to `errorMiddleware` as an unclassified error, producing a 500 response with no access log for that request. No explicit length limit is applied, so a very large header value will be echoed in full.

There is also no `Access-Control-Expose-Headers` configuration in the application, so browser-based clients on a different origin cannot read `X-Request-Id` unless the consuming gateway/proxy exposes it. The header contract is therefore reliable for server-to-server callers and same-origin clients, and conditional for cross-origin browsers.

**Rule workflow**:
```
requestId resolved (BR-01)
  -> res.setHeader('X-Request-Id', requestId)     (before handler)
  -> handler / error middleware writes status + body
  -> header is emitted with the response
  -> client may reuse the value as inbound x-request-id on later calls
```

---

### Business Rule: Monotonic duration measurement with two-decimal normalization

**Overview**:
Duration is measured from middleware entry to response completion using `process.hrtime.bigint()`, converted to milliseconds, and rounded to two decimal places as a JSON number.

**Detailed description**:
`const startedAt = process.hrtime.bigint()` at line 10 captures a monotonic nanosecond timestamp (immune to wall-clock adjustments, unlike `Date.now()`). On `finish`, line 13 computes `Number(process.hrtime.bigint() - startedAt) / 1_000_000`, converting the nanosecond `bigint` delta to a `number` of milliseconds, and line 20 applies `Number(durationMs.toFixed(2))`. The double conversion is deliberate: `toFixed(2)` produces a string with exactly two decimals, and `Number(...)` converts it back so the log field is numeric rather than a quoted string, which keeps downstream aggregation (averages, percentiles, range queries) numerically meaningful.

The measurement window starts when the middleware's function body runs, not when the TCP connection is accepted and not when the request line is parsed. Two consequences follow. First, the time spent by `express.json` parsing and validating the body is excluded, so the reported duration understates total request handling for large payloads. Second, the measurement includes everything from the logger to the moment Express emits `finish`: authentication, validation, controller, service, repository and all database round trips. On the write path it therefore approximates end-to-end application latency, not network transit time.

`finish` is emitted when the response has been handed to the operating system, not when the client has received it. A slow client that reads the body slowly can delay `finish` while the socket flushes (backpressure), so `durationMs` can include socket write time for large responses; conversely, a client that disconnects mid-response emits `close` instead of `finish`, and no measurement is ever logged. Precision is bounded below by timer resolution (nanoseconds on Linux, but practically sub-millisecond) and above by `Number` safety; a request lasting longer than roughly 285 years would lose integer precision, which is not a practical concern. `toFixed(2)` also means very fast responses are reported as `0` (for example `0.42` rounds to `0.42`, but `0.004` becomes `0`), so the field cannot distinguish sub-5-microsecond differences.

**Rule workflow**:
```
middleware entry
  -> startedAt = hrtime.bigint()                      (monotonic ns)
  -> ... downstream processing, response written ...
  -> 'finish' event
  -> deltaNs  = hrtime.bigint() - startedAt
  -> durationMs = Number(deltaNs) / 1_000_000
  -> logged as Number(durationMs.toFixed(2))          (number, 2 decimals)
```

---

### Business Rule: One access log record per completed response, on `finish`

**Overview**:
Exactly one structured access-log entry named `http_request` is emitted per response that reaches `finish`, at `info` level, through the shared Pino logger.

**Detailed description**:
The middleware registers `res.on('finish', handler)` at `src/middlewares/request-logger.middleware.ts:12` rather than logging synchronously. `finish` fires once per response in Node's HTTP implementation, which gives the "exactly one record" property without explicit de-duplication guards. Deferring to `finish` is what makes the record accurate: `res.statusCode` is final only after the handler has written the head, and `req.user` is populated only after `authenticate` has run inside the module router. Logging at middleware entry would capture `undefined` for both.

The record is emitted through the process-wide Pino singleton (`src/shared/logger/index.ts:32`) using `logger.info({...}, 'http_request')`. Because Pino filters by level, the record is only written when the configured level is `info` or more verbose (`debug`, `trace`). `LOG_LEVEL` is validated to one of `fatal|error|warn|info|debug|trace` and defaults to `info` (`src/config/env.ts:6`, `src/shared/logger/index.ts:15`); with `LOG_LEVEL=warn` the application silently stops emitting access logs altogether, while error logs continue. There is no independent switch for access logging, no sampling, and no exclusion for high-volume endpoints such as `/health`.

The best-effort character of the rule comes from what is absent: there is no `try/catch` around the log call, no `res.on('error')` handler, and no `res.on('close')` fallback. If Pino's write throws (for example, a broken stdout stream), the exception surfaces from an EventEmitter listener, which Node reports as an uncaught exception in that tick; the HTTP response has already been sent, so the client is unaffected, but the process may be destabilized depending on the global handlers (none are registered for `uncaughtException`). Requests that never finish — client aborts, server timeouts, socket resets during streaming — produce no access log, so the log is a record of *completed* responses rather than of *received* requests.

**Rule workflow**:
```
middleware entry -> res.once-ish registration via res.on('finish', handler)
  -> next() passes control downstream
  -> handler writes status/body -> response head flushed
  -> 'finish' emitted once
      -> build record (BR-06) -> logger.info(record, 'http_request')
      -> Pino applies level filter (LOG_LEVEL) and redaction
  -> no 'finish' (abort/close) => no record
```

---

### Business Rule: Fixed access-log schema and event name

**Overview**:
Every access record carries the same curated field set — `requestId`, `method`, `path`, `statusCode`, `durationMs`, `userId` — plus the Pino base fields and the static message `http_request`.

**Detailed description**:
The object literal at `src/middlewares/request-logger.middleware.ts:15-22` is the component's output contract. `requestId` links to `X-Request-Id` and error logs; `method` and `path` identify the operation; `statusCode` is the final numeric status; `durationMs` is the normalized timing from BR-04; `userId` is the optional identity from BR-07. Pino adds `level` (30 for info), `time` (ISO 8601, configured at `src/shared/logger/index.ts:21`), `service: 'order-management-api'` and `env` (base fields, `src/shared/logger/index.ts:20`). In development the record is rendered by `pino-pretty` with colors and `HH:MM:ss.l` timestamps (`src/shared/logger/index.ts:22-28`); in test and production it is newline-delimited JSON on stdout.

`path` is `req.originalUrl`, which is the full original URL including the mount path and the query string. That is intentional for diagnostic value — an operator can see exactly which endpoint and filters were used — but it also means query-parameter values are persisted in logs. The Pino `redact` configuration only matches specific key paths (`req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`), none of which apply to this flat record, so a request like `/api/v1/orders?customerEmail=...` writes the e-mail into the log. The explicit field list (rather than logging `req`/`res` wholesale) is nonetheless a strong privacy control: headers, cookies, body and response payload are never included.

The event name is the fixed string `http_request`, used as the Pino `msg`. Any log pipeline must filter on this value to isolate access records from `server_started`, `shutdown_initiated`, `Unhandled error in request` and other messages emitted by the same logger. Field naming is camelCase and stable, which makes the schema easy to index, but there is no TypeScript type or schema validating the object literal, and no versioning marker on the event.

**Rule workflow**:
```
on finish:
  record = {
    requestId:   resolved id (string, possibly "")
    method:      req.method            (e.g. "GET")
    path:        req.originalUrl       (includes query string)
    statusCode:  res.statusCode        (final numeric status)
    durationMs:  number, 2 decimals
    userId:      req.user?.id          (omitted when undefined)
  }
  logger.info(record, 'http_request')
  -> Pino adds level, time, service, env; applies LOG_LEVEL + redaction
```

---

### Business Rule: Authenticated identity enrichment (`userId`)

**Overview**:
The access record includes the authenticated user's ID when a `req.user` exists at response time; otherwise the field is omitted.

**Detailed description**:
`userId: req.user?.id` at line 21 reads the optional property that `authenticate` sets from the JWT `sub` claim (`src/middlewares/auth.middleware.ts:42`). Because the access log fires on `finish`, the value reflects the identity resolved during the request, even though the logger ran first. This makes it possible to answer "which user made this request" from access logs alone for authenticated routes such as `GET /api/v1/auth/me`, all customer/product/order routes, and admin-only user reads.

For anonymous requests the optional chain yields `undefined`. Pino's JSON serialization drops `undefined` values, so the field is absent rather than `null`; consumers must treat "missing `userId`" as anonymous. Two accuracy caveats apply. First, routes that are public by design (`POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `GET /health`) always lack `userId` even when the caller presents a token, because those routers do not run `authenticate` — the login request itself therefore cannot be attributed to the user who is logging in, only to the e-mail in its body (which is not logged). Second, the field reflects token claims, not a fresh database lookup, so a deleted user or changed role is represented by stale claims for up to the token lifetime (`JWT_EXPIRES_IN`, default 8h).

The rule creates an implicit dependency on middleware ordering inside each module router: `authenticate` must run before the response completes for `userId` to appear. Since `authenticate` is mounted with `router.use` in customer, product and order routers and directly on protected auth/user routes, this holds today; a route that verified tokens inside its controller without setting `req.user` would log anonymously.

**Rule workflow**:
```
request completes
  -> req.user set by authenticate? (JWT sub -> req.user.id)
       YES -> userId = req.user.id  -> field present in record
       NO  -> userId = undefined    -> field omitted by JSON serialization
  -> no database lookup, no PII beyond the opaque id
```

---

### Business Rule: Transparent pass-through (no short-circuit, no response writing)

**Overview**:
The middleware must not alter the response, delay the pipeline, or terminate the request; its only control action is a synchronous `next()`.

**Detailed description**:
After registering the listener, the middleware calls `next()` at line 27 without arguments and returns. It never calls `res.send`, `res.end`, `res.status` or `next(err)`, and it performs no I/O on the hot path: UUID generation is CPU-only, `hrtime.bigint()` is a syscall-free clock read, `setHeader` is in-memory, and `res.on` is a listener registration. The only blocking work is the lazy UUID generation performed once per request that lacks a header. This design keeps the middleware orthogonal to authentication, validation, controllers and the error handler: it cannot cause a 4xx/5xx by itself and cannot mask an error.

There are three observable exceptions to full transparency, all edge-case rather than routine. (1) If the resolved ID contains characters illegal in an HTTP header value, `res.setHeader` throws synchronously; Express's `Layer.handle_request` wraps middleware invocation in a `try/catch` and forwards the error to `errorMiddleware`, so the request becomes a 500 with no access log. (2) If the Pino logger construction failed at import time, module evaluation would fail and the application would not start at all — the logger is created eagerly when `src/shared/logger/index.ts` is imported, and it in turn requires valid environment configuration (`src/config/env.ts:26`). (3) The `finish` listener keeps a closure over `req` and `res` until the response completes; for a hung response, that closure is retained for the connection lifetime, which is a bounded but real memory cost under many long-lived streaming responses (the current API has none).

Because the middleware is registered globally (`app.use(requestLogger)` with no path), it applies to every method and path, including `OPTIONS`, `HEAD` and unknown paths. There is no allowlist, no sampling and no health-check exclusion.

**Rule workflow**:
```
resolve id -> attach req.id -> set response header -> start clock
  -> register finish listener
  -> next()   (no error, no response write, no early return path)
  -> downstream owns the entire response lifecycle
```

---

### Business Rule: Logging configuration and redaction boundary

**Overview**:
Access-log visibility is governed by the shared logger's `LOG_LEVEL`, and the logger enforces credential redaction for known sensitive key paths; the middleware itself performs no redaction.

**Detailed description**:
`src/shared/logger/index.ts:4-11` defines six redaction paths — `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken` — with censor `[REDACTED]`, applied by Pino to any logged object. The request logger's record does not contain a `req` object, so the first two paths never match; the wildcard paths would match only if a logged object had a key such as `password` or `accessToken`. In practice the record is already minimal and contains no credentials, so redaction is a defense-in-depth layer rather than an active filter for this component. Its practical value is for the `err` object logged by `errorMiddleware` and for future log sites.

Level filtering interacts with the access log in a way that is easy to overlook: `LOG_LEVEL=warn` or `LOG_LEVEL=error` disables `logger.info` entirely, so the system retains error diagnostics but loses all HTTP traffic records, with no warning or fallback. Conversely `LOG_LEVEL=debug` or `trace` adds no new fields from this component; it only allows other libraries' debug output. The logger also switches transport based on `NODE_ENV`: `pino-pretty` in `development`, plain JSON otherwise, so log parsing behavior differs between local runs and deployed environments while field names remain identical. Timestamps are ISO 8601 (`pino.stdTimeFunctions.isoTime`), and every record carries `service: 'order-management-api'` and `env`, allowing multi-service aggregation to separate environments.

**Rule workflow**:
```
logger.info(record, 'http_request')
  -> level check: record.level(30) >= configured LOG_LEVEL?
       NO  -> dropped silently (e.g. LOG_LEVEL=warn)
       YES -> redaction pass over configured paths
           -> NODE_ENV === 'development' ? pino-pretty transport : JSON to stdout
```

---

## 4. Component Structure

```
src/
├── app.ts                                  # Composition root; mounts requestLogger globally at line 60
├── server.ts                               # Process entry; no direct use of the middleware
├── middlewares/
│   ├── request-logger.middleware.ts        # THIS COMPONENT (28 lines)
│   │                                       #  - requestLogger: RequestHandler
│   │                                       #  - id resolution, header echo, timing, access log
│   ├── auth.middleware.ts                  # Hosts the Request.id / Request.user type augmentation (lines 12-17)
│   │                                       #  Produces req.user consumed by BR-07
│   ├── error.middleware.ts                 # Consumes req.id for error-log correlation (lines 56-60)
│   └── validate.middleware.ts              # Unrelated; downstream in the chain
├── shared/
│   └── logger/
│       └── index.ts                        # Pino factory + singleton; redaction, level, transport
└── config/
    └── env.ts                              # LOG_LEVEL and NODE_ENV validated here; imported via logger
```

Component boundary:

| Artifact | Role in this component |
|----------|------------------------|
| `requestLogger` (`RequestHandler`) | The entire public surface: one named export, no options, no factory |
| `requestId` (local const) | Internal state; not exported |
| `startedAt` (local const, `bigint`) | Internal timing state captured per invocation |
| `res.on('finish')` closure | Internal callback holding the record construction |
| `req.id` | Output to the request object (contract shared with error middleware) |
| `X-Request-Id` header | Output to the client |
| `http_request` log record | Output to the observability channel |

No configuration object, no dependency injection, no class, no interface, no test double seam. The module has exactly one executable statement path and no branches other than the `??` fallback.

---

## 5. Dependency Analysis

```
Internal Dependencies (runtime)
src/app.ts:3,60
  -> requestLogger (src/middlewares/request-logger.middleware.ts:5)
       -> logger (src/shared/logger/index.ts:32)
            -> createLogger() -> pino(...)
            -> env (src/config/env.ts:27)
                 -> zod envSchema (NODE_ENV, LOG_LEVEL, ...)

Internal Dependencies (type-level only)
src/middlewares/request-logger.middleware.ts:1 -> RequestHandler (express types)
src/middlewares/auth.middleware.ts:12-17 -> declares Request.id?: string (compile-time contract,
                                            no runtime import in either direction)

External Dependencies
- express 4.21.1         - RequestHandler type only (erased at runtime); runtime objects are
                           req/res provided by Express
- uuid 11.0.3            - v4() UUID generation when x-request-id is absent
- pino 9.5.0             - indirect: structured logging via src/shared/logger
- pino-pretty 11.3.0     - indirect: development transport only (devDependency in package.json)
- node:process           - process.hrtime.bigint() monotonic clock (built-in)

Inverse (afferent) consumers
- src/app.ts             - registers the middleware (only importer)
- src/middlewares/error.middleware.ts - reads req.id at runtime (contract, not import)
- logging/aggregation pipeline - consumes the http_request records (external, out of repo)
- HTTP clients / gateways - consume the X-Request-Id response header (external contract)
```

Dependency notes:

- The runtime dependency on `src/shared/logger` is eager. Importing the component imports Pino and executes `env` validation; a missing `DATABASE_URL` or `JWT_SECRET` terminates the process even though this component never uses them (`src/config/env.ts:14-25`).
- `uuid` is used only here. The architectural report notes `@types/uuid` is redundant for uuid v11 (`docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md:369`).
- `pino-http` is a declared dependency with zero imports; this component is the hand-rolled equivalent.
- There is no dependency on the database, on any module, or on any domain type. The only domain-adjacent read is `req.user?.id`, typed via the auth middleware's declaration merge.

---

## 6. Afferent and Efferent Coupling

Metrics are computed statically on internal source files (external npm packages excluded), consistent with the architectural report; `Ca` = distinct internal files importing this module, `Ce` = distinct internal files imported by this module. "Contract coupling" rows describe non-import coupling through `req.id`, `req.user` and the header.

| Component | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Critical |
|-----------|------------------------|------------------------|----------|
| `requestLogger` (module, file-level) | 1 (`src/app.ts`) | 1 (`src/shared/logger/index.ts`) | High |
| `requestLogger` (excluding type-only Express import) | 1 | 1 | High |
| `Request.id` contract (as a symbol) | 2 runtime consumers (`request-logger.middleware.ts:7` write; `error.middleware.ts:56` read) + 1 type host (`auth.middleware.ts:12-17`) | 0 | Medium |
| `X-Request-Id` header contract | 1 producer; unbounded external consumers | 0 | Medium |
| `http_request` log schema | 1 producer; external aggregation consumers | 0 | Medium |
| `logger` singleton (for context) | 3 internal importers | 1 (`src/config/env.ts`) | High |

Interpretation:

- **Afferent 1 / efferent 1** makes the module structurally simple and stable, but its *operational* criticality is high because it is mounted globally: a defect here affects every endpoint and every log line in the system. Static coupling understates blast radius for cross-cutting middleware.
- The module is a **leaf consumer** of infrastructure (logger) and a **producer of contracts** (header, `req.id`, log schema). It has no inbound domain coupling, which is why it can be modified in isolation at compile time but not at runtime.
- The `req.id` type contract being declared in `src/middlewares/auth.middleware.ts` rather than here is a **cohesion defect at the type level**: the symbol's ownership is split, and a reader of this file cannot see its declared type.
- Efferent coupling would rise to 2 if the component imported `env` directly (for sampling or log toggles); today it inherits that dependency transitively, so configuration changes to `LOG_LEVEL` affect it without appearing in its import list. This is **hidden coupling**.

Section 7 (Endpoints) is intentionally omitted: this component exposes no endpoint of its own. It participates in every HTTP route by wrapping the request/response lifecycle, but it declares no path, method or protocol surface; the routes it observes are owned by `src/routes/index.ts` and the module routers.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Inbound `x-request-id` header | Client/upstream contract | Correlation ID propagation from gateways, proxies, SDKs, or retrying clients | HTTP header | String (arbitrary, unvalidated) | None: any value accepted; absent value triggers UUID generation; illegal header characters surface as 500 via `setHeader` throw |
| Outbound `X-Request-Id` header | Client contract | Return the correlation ID so callers can quote/chain it | HTTP response header | String | Node validates header characters; no length cap; no CORS expose-headers configuration |
| Pino logger (`src/shared/logger`) | In-process library | Emit the `http_request` access record | In-process function call | Structured JSON (NDJSON) in test/production; pino-pretty in development | No try/catch; write failures are unhandled in the `finish` listener |
| `uuid` library | In-process library | Generate request IDs when absent | In-process function call | RFC 4122 v4 UUID string | No failure path (pure computation) |
| Express `ServerResponse` event emitter | Framework integration | Defer logging until the response is complete; read final status | In-process event (`finish`) | Event callback | Only `finish` is observed; `close`/`error` are not, so aborted requests are unlogged |
| `authenticate` middleware (producer of `req.user`) | Internal middleware contract | Supply `userId` for the access record | In-process shared request state | JWT claims projected to `{id, email, role}` | If auth never runs, field is omitted; no failure propagation |
| `errorMiddleware` (consumer of `req.id`) | Internal middleware contract | Correlate error logs with access logs | In-process shared request state | String | Falls back to `'unknown'` when `req.id` is unset (`error.middleware.ts:56`) |
| `express.json` body parser | Framework, ordering-sensitive | Must run before the logger for JSON bodies to be parsed; its failures bypass the logger | In-process | JSON -> `req.body` | Requests rejected before the logger get no ID and no access log |
| Log aggregation / observability backend | External (out of repo) | Consume stdout records; filter on `msg === 'http_request'`; index camelCase fields | stdout capture / sidecar | JSON lines, ISO timestamps | Not configured in the repository; no metrics/tracing exporter |

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Chain of Responsibility / Middleware Pipeline | `requestLogger: RequestHandler` mounted with `app.use` | `src/middlewares/request-logger.middleware.ts:5`; `src/app.ts:60` | Cross-cutting concern inserted into the Express chain without modifying handlers |
| Observer / Event Listener | `res.on('finish', handler)` | `src/middlewares/request-logger.middleware.ts:12` | Emit telemetry after the response lifecycle completes, without delaying it |
| Correlation Identifier (Correlation ID) | Forward-or-generate ID; `req.id`; `X-Request-Id` header; `requestId` log field | `src/middlewares/request-logger.middleware.ts:6-8,16` | Tie access logs, error logs and client reports to one request |
| Structured Logging / Canonical Log Line | Single curated object with fixed field names and static event `http_request` | `src/middlewares/request-logger.middleware.ts:14-24` | Machine-parseable access records suitable for aggregation |
| Singleton (shared logger) | Module-level `export const logger = createLogger()` | `src/shared/logger/index.ts:32` | One Pino instance, one level, one transport for the process |
| Lazy Evaluation / Deferred Computation | UUID generated only when the header is absent; duration computed only at `finish` | `src/middlewares/request-logger.middleware.ts:6,13` | Avoid work on the hot path and avoid logging incomplete state |
| Monotonic Clock Usage | `process.hrtime.bigint()` instead of `Date.now()` | `src/middlewares/request-logger.middleware.ts:10,13` | Wall-clock-independent, high-resolution latency measurement |
| Defense in Depth (redaction) | Pino `redact` paths with `[REDACTED]` censor | `src/shared/logger/index.ts:4-11,16-19` | Prevent credential leakage even if a caller logs richer objects |
| Environment-Stratified Output | `NODE_ENV === 'development'` selects `pino-pretty` | `src/shared/logger/index.ts:22-28` | Human-readable local logs, machine-readable deployed logs |
| Type Augmentation (module augmentation) | `declare module 'express-serve-static-core'` adds `id?` and `user?` | `src/middlewares/auth.middleware.ts:12-17` | Compile-time typing for properties attached by middleware |
| Custom Middleware replacing Library | Hand-written access logger instead of `pino-http` | `src/middlewares/request-logger.middleware.ts`; `package.json:31` | Full control of the record schema; adds maintenance surface |

Architectural decisions observable in the code:

1. **Access logging is tied to the response, not the request.** The decision makes status and identity accurate but loses aborted requests.
2. **Correlation is client-trusted.** No ID validation was implemented, favoring interoperability over tamper resistance.
3. **The record is a curated projection, not a request dump.** This is a deliberate privacy/volume control; the one leak is the query string inside `path`.
4. **The middleware is stateless and global.** No configuration, no sampling, no per-route opt-out; all traffic is logged at `info`.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Pipeline ordering (integration, `src/app.ts:59-60`) | `express.json` is registered before `requestLogger`, so malformed JSON and payloads over 1 MB bypass the logger entirely | No request ID, no `X-Request-Id` header and no access log for those requests; `errorMiddleware` logs them with `requestId: 'unknown'`, breaking correlation exactly when debugging client errors |
| Medium | Request ID trust (`middleware:6-8`) | Client-supplied `x-request-id` is accepted without format, length or content validation and echoed into logs and the response | Correlation spoofing (reusing another request's ID), log forging with misleading identifiers, unbounded log-line and header size, potential confusion in incident analysis |
| Medium | Log content (`middleware:18`) | `path` is `req.originalUrl` including the query string, and Pino redaction paths do not apply to this flat record | PII and business data (e-mails, documents, search terms, filters) persisted in logs; retention/compliance exposure |
| Medium | Observability completeness (`middleware:12`) | Only `finish` is handled; `close`/`error` are not | Client aborts, timeouts and socket resets produce no access log; traffic volume and failure rates are undercounted |
| Medium | Configuration coupling (`src/shared/logger/index.ts:15`) | Access-log emission depends on the global `LOG_LEVEL`; `warn`/`error` disables all access logs with no dedicated switch or warning | Silent loss of HTTP telemetry in an environment tuned for quieter logs; no sampling alternative to control volume |
| Low | Empty identifier edge case (`middleware:6`) | `??` accepts an empty-string `x-request-id`, producing an empty `req.id`, empty response header and empty log field | Several requests can share `requestId: ""`, making correlation ambiguous; only absent headers get UUIDs |
| Low | Type safety (`middleware:6`) | Header is cast `as string | undefined` while Node types allow `string[]`; `req.id` is assigned from a cast value | Duplicate-header behavior relies on Node's join semantics; no runtime guard; type assertion hides the edge case |
| Low | Header reflection (`middleware:8`) | The response header is a verbatim reflection of caller input with no length cap | Very large header values are echoed and logged; invalid characters cause a 500 via `setHeader` throw with no access log for that request |
| Low | Cohesion / ownership (`auth.middleware.ts:12-17` vs `middleware:7`) | `Request.id` is declared in the auth middleware although written here | Contract is hard to discover from the owning file; couples the request-id type to the auth module |
| Low | Failure handling (`middleware:14-24`) | No `try/catch` around `logger.info`; a throwing transport listener can become an uncaught exception | Process stability risk on logging backpressure/failure; client response is already sent, so impact is availability, not correctness |
| Low | Duplication / dependency surface (`package.json:31`) | `pino-http@10.3.0` is declared but unused while this custom logger exists | Redundant supply-chain surface and maintenance ambiguity about which logger is authoritative |
| Low | Test coverage | No unit or integration test asserts any behavior of this component | Regressions in header propagation, ID generation, field schema, timing units or `userId` enrichment go undetected |
| Low | Timing scope (`middleware:10`) | Clock starts after body parsing and excludes network transit | `durationMs` understates total handling for large bodies and cannot be compared with proxy-measured latency |
| Low | Duration precision (`middleware:20`) | `toFixed(2)` rounds sub-10-microsecond durations to `0` | Loss of resolution for very fast endpoints (for example cached `GET /health`) |
| Low | Memory retention (`middleware:12`) | The `finish` listener closure holds `req`/`res` for the response lifetime | Bounded memory cost; relevant only if long-lived streaming responses are added |
| Low | Event schema governance | No TypeScript type or schema for the `http_request` record; no event version field | Consumers can silently break if field names change; no contract test |
| Low | No cross-origin exposure | No `Access-Control-Expose-Headers` for `X-Request-Id` | Browser clients on another origin cannot read the correlation ID |

Security summary for this component:

- **Strengths:** no headers, cookies, body or response payload are logged; credential redaction is configured even though the record is already minimal; client-supplied values are placed in JSON fields (structurally escaped) and in a Node-validated header (no CRLF injection); IDs generated locally are random v4 UUIDs, not guessable sequential values.
- **Weaknesses:** the trust boundary for `x-request-id` is open (no validation or length cap), which enables log-forging and ID-collision attacks; the query string is logged without redaction; anonymous vs. authenticated attribution is absent for public routes, so login attempts cannot be tied to a user id.
- **Not applicable here:** no authorization or business decisions are made by this component; it cannot grant or deny access.

---

## 11. Test Coverage Analysis

Test inventory (all tests in the repository; `vitest.config.ts` includes `tests/**/*.test.ts`):

| File | Type | Tests | Direct assertions on this component |
|------|------|-------|-------------------------------------|
| `tests/auth.test.ts` | Integration (Supertest + real MySQL) | 7 | None |
| `tests/orders.test.ts` | Integration (Supertest + real MySQL) | 8 | None |
| `tests/setup.ts` | Harness (DB connect, truncate, disconnect) | n/a | None |
| `tests/helpers/factories.ts` | Harness (`getTestApp()` caches `buildApp({ prisma })`) | n/a | Indirect: every request traverses the middleware |
| `docs/...` | Generated analysis artifacts | n/a | Not tests |

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| `requestLogger` middleware | 0 | 0 direct; 15 indirect executions (7 auth + 8 orders), each request passes through `src/app.ts:60` | Not measured: no coverage provider configured in `vitest.config.ts`; statically 0% of this file's behaviors asserted | No assertions on `X-Request-Id` presence/value, generated-vs-forwarded ID, `req.id`, `http_request` emission, log field schema, duration units, or `userId` enrichment. The middleware is exercised as a side effect only. |
| `X-Request-Id` header contract | 0 | 0 | 0% | No test sets an inbound `x-request-id`, no test asserts the response header, no test checks header presence on error responses (401/404/409) |
| Access-log schema (`http_request`) | 0 | 0 | 0% | Pino output is never captured; field names, `msg`, level, and `path`/`query` content unverified |
| Duration measurement | 0 | 0 | 0% | Unit conversion, two-decimal rounding and clock selection unverified |
| ID generation (uuid fallback) | 0 | 0 | 0% | No test covers the absent-header branch |

Verification basis: `grep` for `request-id|X-Request|req.id|requestId` across `tests/` returns no matches (matches exist only in `src/` and `docs/`). `tests/auth.test.ts` and `tests/orders.test.ts` assert status codes and body shapes only. `tests/helpers/factories.ts:10-15` builds the app once and caches it, so all 15 tests do traverse the middleware, meaning it is **executed but unverified**.

Coverage gaps ranked by risk:

1. **No assertion that a client-supplied `x-request-id` is preserved** — the core correlation contract could regress silently (for example a switch from `??` to always-generate).
2. **No assertion that a generated ID is a valid UUID v4 and is echoed** — the fallback branch is entirely untested.
3. **No assertion of the `http_request` record** (capture Pino output via a destination stream) — the schema, the two-decimal numeric `durationMs`, the fixed `msg`, and the presence/absence of `userId` are all unverified.
4. **No assertion of `userId` enrichment** on an authenticated route and its absence on a public route.
5. **No test for the empty-string header** or other edge inputs (very long IDs, duplicate headers).
6. **No test that error responses still carry `X-Request-Id`** or that `errorMiddleware` receives the same ID.
7. **No test of ordering interactions** (body-parser failure producing no ID), which is an integration-level behavior spanning `src/app.ts` and `error.middleware.ts`.
8. **No coverage instrumentation at all**, so even the "executed but unverified" state is invisible in CI.

Serial execution (`fileParallelism: false`, `singleFork: true`, `vitest.config.ts:11-17`) and a real MySQL dependency (`tests/setup.ts:4-6`) mean adding logger assertions would preferably be done as fast unit tests against a mocked `req`/`res` pair or an injected Pino destination rather than through the DB-bound integration suite; such a seam does not exist today because the logger is a module singleton.

---

## 12. Report Metadata

| Item | Value |
|------|-------|
| Component analyzed | Request Logger Middleware |
| Primary file | `src/middlewares/request-logger.middleware.ts` |
| File size | 28 lines, 1 exported symbol (`requestLogger`) |
| Internal coupling | Ca = 1, Ce = 1 |
| External libraries used directly | `express` (types), `uuid` (runtime) |
| External libraries used indirectly | `pino`, `pino-pretty`, `zod` (via env), Node `process` |
| Tests touching the component | 15 indirect (0 direct) |
| Report generated | 2026-11-08 23:09:07 |
