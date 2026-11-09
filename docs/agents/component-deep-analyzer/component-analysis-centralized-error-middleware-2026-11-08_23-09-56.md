# Component Deep Analysis Report: Centralized Error Middleware

**Component:** Centralized Error Middleware (`errorMiddleware`)
**Primary file:** `src/middlewares/error.middleware.ts` (65 lines)
**Analysis scope:** entire project root (`/home/eric-ogata/codes/full_cycle/mba_engenharia_ia/desafios/desafio4/mba-ia-desafio-design-docs-com-ia`)
**Excluded from analysis:** `node_modules`, `.git`, `dist`, `coverage`
**Analyzed revision:** `e7f6311` (working tree; modified `.gitignore` and untracked analysis artifacts do not affect runtime code)
**Analysis date:** 2026-11-08 23:09:56
**Report type:** Analysis only. No project file was modified. The only file created is this report.

---

## 1. Executive Summary

The Centralized Error Middleware is the single terminal error handler of the Express application. It is a four-argument `ErrorRequestHandler` mounted last in the middleware chain (`src/app.ts:73`), after `express.json`, the request logger, the `/health` route, the versioned API router and the 404 fallback. Every error surfaced anywhere in the HTTP pipeline is expected to converge on this function, which converts it into a JSON error envelope or, when the error is not recognized, logs it and returns a generic 500 response.

The component has a narrow implementation but a system-wide contract. It owns five classification branches: `AppError` (and every subclass in the shared taxonomy), `ZodError`, Prisma `P2002` (unique constraint), Prisma `P2025` (record not found), and a fail-safe fallback for everything else. The output envelope is `{ error: { code, message, details? } }`, and the response status is taken directly from the recognized error type. The middleware is stateless: it reads only the incoming error, the request correlation id and the request method/URL, and it writes exactly one JSON response.

Key findings:

- **Strong single-responsibility design, limited test exposure.** There is no dedicated unit test file for the middleware; its behavior is exercised only indirectly by 15 integration tests, 8 of which assert on error responses. The fallback branch (500 + structured log), the `ZodError` branch, and both Prisma branches have no test evidence.
- **The domain-to-HTTP translation contract is stable and machine-readable.** Error `code` strings are distinct from HTTP status and are consumed by tests and clients (`VALIDATION_ERROR`, `NOT_FOUND`, `INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK`, `EMAIL_ALREADY_USED`, `INVALID_ORDER_STATE_FOR_DELETE`).
- **Unrecognized errors are masked.** Unexpected exceptions never disclose stack traces or internal messages; the response is always `500 INTERNAL_SERVER_ERROR` with the fixed message `Internal server error`.
- **The classification surface is narrower than the error surface.** Body-parser failures (`express.json` malformed JSON, payload over 1 MB), Prisma codes other than `P2002`/`P2025` (notably `P2003` foreign-key restriction), and any library error carrying its own `status` are all flattened into 500 responses. This is the component's most consequential behavioral gap.
- **The handler has no `headersSent` guard and never delegates.** `_next` is unused (`src/middlewares/error.middleware.ts:14`), so an error raised after a response has started would cause a second `res.status().json()` attempt rather than falling back to the Express default error handler.
- **Coupling is low and asymmetric.** Afferent coupling is 1 internal file (`src/app.ts`); efferent coupling is 2 internal files (`shared/errors` barrel and `shared/logger`) plus three runtime library contracts (`express` types, `@prisma/client`, `zod`). Criticality is high because the component is the only place where internal failures become HTTP responses.

---

## 2. Data Flow Analysis

### 2.1 Normal request path (for orientation)

```
1. Request enters the Express app (src/app.ts:55-60)
2. express.json parses the body with a 1 MB limit (src/app.ts:59)
3. requestLogger assigns req.id (from X-Request-Id or uuid v4) and sets the
   response header X-Request-Id (src/middlewares/request-logger.middleware.ts:5-8)
4. /health short-circuits or /api/v1 router dispatches (src/app.ts:62-67)
5. Module router applies authenticate and validate (e.g. src/modules/orders/order.routes.ts:14-24)
6. Controller executes the service inside try/catch (e.g. src/modules/orders/order.controller.ts:9-17)
7. Service returns data; controller writes the success response
```

### 2.2 Error path into the component

```
1. A throw originates in one of four places:
   a. middleware guard        -> UnauthorizedError / ForbiddenError (src/middlewares/auth.middleware.ts:30,36,45,52,56)
   b. validation middleware   -> ValidationError wrapping Zod issues (src/middlewares/validate.middleware.ts:25-33)
   c. service / controller    -> AppError subclasses (src/modules/**/*.service.ts, *.controller.ts)
   d. persistence layer       -> PrismaClientKnownRequestError (P2002, P2025, P2003, ...)
      or body parser            -> SyntaxError / entity.too.large (raised before routing, src/app.ts:59)
2. The throw is converted to next(err) by the nearest catch block
   (all controllers use try/catch + next(err); src/modules/orders/order.controller.ts:14-16)
3. Express skips remaining non-error middleware and invokes the only 4-argument
   handler registered last (src/app.ts:73)
4. errorMiddleware inspects the error in fixed order:
   step 4.1  err instanceof AppError                 -> statusCode, errorCode, message, optional details (15-24)
   step 4.2  err instanceof ZodError                 -> 400 VALIDATION_ERROR + normalized issues (26-35)
   step 4.3  err instanceof PrismaClientKnownRequestError
             4.3.1 code P2002                        -> 409 CONFLICT + target columns (38-47)
             4.3.2 code P2025                        -> 404 NOT_FOUND (48-53)
             4.3.3 any other code                    -> fall through to 4.4
   step 4.4  fallback                                -> logger.error + 500 INTERNAL_SERVER_ERROR (56-64)
5. res.status(...).json({ error: { ... } }) serializes the response
6. The request logger's 'finish' listener emits the access log with the final
   status code and duration (src/middlewares/request-logger.middleware.ts:12-25)
```

### 2.3 Data transformation diagram

```
                       +---------------------------+
   Error object  ----> |  classification (instance) |
                       +------------+--------------+
                                    |
        +----------------+----------+---------+-----------------+
        |                |                    |                 |
        v                v                    v                 v
    AppError         ZodError          Prisma P2002/P2025   anything else
        |                |                    |                 |
        v                v                    v                 v
 statusCode,        fixed 400,         409 / 404,          logger.error(
 errorCode,         VALIDATION_ERROR,  fixed codes,       err, requestId,
 message,           path/message       target message      method, path)
 details?           details[]          (P2002 only)             |
        |                |                    |                 v
        +----------------+--------------------+----------> 500 INTERNAL_SERVER_ERROR
                                    |
                                    v
                 res.status(n).json({ error: { code, message, details? } })
```

### 2.4 Error precedence semantics

The order of the `instanceof` checks is behaviorally significant. `AppError` is tested first, so a `ValidationError` produced by the validation middleware (which is an `AppError`) is handled by branch 4.1 and never reaches the `ZodError` branch. The `ZodError` branch is therefore a fallback for any Zod failure that escapes a handler without being wrapped. Similarly, `PrismaClientKnownRequestError` extends a Prisma base error, not `AppError`, so it is only evaluated after the first two branches. Within the Prisma branch, `P2002` is tested before `P2025`, but the codes are mutually exclusive, so the order between them does not change behavior. Only the two recognized Prisma codes return; all others intentionally fall through to the generic branch.

### 2.5 Request-scoped data used by the component

| Data | Source | Usage | Fallback |
|------|--------|-------|----------|
| `err` | Express error pipeline | Instance checks, message/details extraction, logging | none; always present as the first argument |
| `req.id` | `requestLogger` (`request-logger.middleware.ts:7`) | Correlates the error log with the access log and `X-Request-Id` header | string `'unknown'` (`error.middleware.ts:56`) |
| `req.method` | Express request | Error log field `method` | none |
| `req.originalUrl` | Express request | Error log field `path` (includes query string) | none |
| `res` | Express response | Status and JSON body | none |
| `_next` | Express pipeline | Not used | not used |

---

## 3. Business Rules & Logic

## Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location |
|---------|-----------|------------------|----------|
| BR-01 | Error classification | Any `AppError` instance (including all subclasses) is serialized using its own `statusCode`, `errorCode`, `message` and optional `details`, and this branch takes precedence over every other classification | `src/middlewares/error.middleware.ts:15-24`; `src/shared/errors/app-error.ts:3-16` |
| BR-02 | Response contract | Every error response has the envelope `{ error: { code, message, details? } }`; `details` is present only when the error carries a value other than `undefined` | `src/middlewares/error.middleware.ts:16-22, 27-33, 40-45, 49-52, 62-64` |
| BR-03 | Validation normalization | A raw `ZodError` is converted to HTTP 400 with code `VALIDATION_ERROR`, fixed message `Validation failed`, and a `details` array of `{ path, message }` where dotted paths are built with `.` and empty paths become `(root)` | `src/middlewares/error.middleware.ts:7-12, 26-35` |
| BR-04 | Persistence conflict mapping | Prisma `P2002` (unique constraint) becomes HTTP 409 with code `CONFLICT`; the message lists the constrained columns derived from `err.meta.target` joined by `, `, or falls back to a generic sentence | `src/middlewares/error.middleware.ts:38-47` |
| BR-05 | Persistence absence mapping | Prisma `P2025` (record not found) becomes HTTP 404 with code `NOT_FOUND` and the fixed message `Resource not found` | `src/middlewares/error.middleware.ts:48-53` |
| BR-06 | Fail-safe masking | Every error that is not an `AppError`, `ZodError`, or recognized Prisma code is logged server-side and returned as HTTP 500 with code `INTERNAL_SERVER_ERROR` and the fixed message `Internal server error`, with no internal detail disclosure | `src/middlewares/error.middleware.ts:56-64` |
| BR-07 | Handler terminality | The middleware is the last registered handler, uses the mandatory 4-argument signature, and never calls `next`; each recognized branch ends the response cycle | `src/middlewares/error.middleware.ts:14-65`; `src/app.ts:69-73` |
| BR-08 | Status/code provenance | The HTTP status and machine-readable code are accepted from the `AppError` instance without validation, whitelisting, coercion or normalization | `src/middlewares/error.middleware.ts:15-24`; `src/shared/errors/app-error.ts:8-13` |
| BR-09 | Classification closure | Only Prisma codes `P2002` and `P2025` are translated; all other Prisma codes and all non-Prisma library errors are treated as unhandled failures | `src/middlewares/error.middleware.ts:37-54` |
| BR-10 | Log correlation | The fallback log record carries the request-scoped id, method and full original URL, and falls back to the literal `unknown` when no id is present on the request | `src/middlewares/error.middleware.ts:56-60`; `src/middlewares/request-logger.middleware.ts:5-8` |

## Detailed breakdown of the business rules

---

### Business Rule BR-01: Application error taxonomy takes precedence

**Overview**:
Any object that is an instance of `AppError` is treated as an intentional, fully classified application error. The middleware uses the error's own HTTP status, its machine-readable code, its human-readable message and, when present, its structured details. Because the check occurs first, this rule overrides every other branch.

**Detailed description**:
`AppError` is the base class of the shared error taxonomy (`src/shared/errors/app-error.ts:3-16`). It extends the native `Error`, assigns `name = 'AppError'`, and stores three public readonly fields: `statusCode` (number), `errorCode` (string) and `details` (object, array or undefined). The constructor also captures a V8 stack trace while excluding the constructor frame. Nine concrete subclasses exist in `src/shared/errors/http-errors.ts`: `BadRequestError` (400, `BAD_REQUEST`), `ValidationError` (400, `VALIDATION_ERROR`), `UnauthorizedError` (401, `UNAUTHORIZED`), `ForbiddenError` (403, `FORBIDDEN`), `NotFoundError` (404, `NOT_FOUND`), `ConflictError` (409, `CONFLICT` by default), `UnprocessableEntityError` (422, `UNPROCESSABLE_ENTITY` by default), `InvalidStatusTransitionError` (409, `INVALID_STATUS_TRANSITION`, extends `ConflictError`) and `InsufficientStockError` (422, `INSUFFICIENT_STOCK`, extends `UnprocessableEntityError`). Because the middleware uses `instanceof AppError`, every present and future subclass inherits correct handling automatically without any registration step.

The branch is reached from the entire request/response surface. Authentication guards raise `UnauthorizedError` for a missing or malformed bearer header, an empty token and an invalid or expired token, and `ForbiddenError` when a role check fails (`src/middlewares/auth.middleware.ts:30,36,45,52,56`). Service layers raise `NotFoundError` for absent users, customers, products and orders (`src/modules/users/user.service.ts:38`, `src/modules/customers/customer.service.ts:25`, `src/modules/products/product.service.ts:26`, `src/modules/orders/order.service.ts:46,60,64,136,183`), `ConflictError` for pre-checked uniqueness and state violations (`src/modules/users/user.service.ts:24`, `src/modules/customers/customer.service.ts:32,48`, `src/modules/products/product.service.ts:33,50`, `src/modules/orders/order.service.ts:141,185`), `ValidationError` for domain-level input rules (`src/modules/orders/order.service.ts:52,88`), `UnprocessableEntityError` for inactive products (`src/modules/orders/order.service.ts:68-72`), `InsufficientStockError` for stock shortfalls (`src/modules/orders/order.service.ts:223`), and `InvalidStatusTransitionError` for disallowed order transitions (`src/modules/orders/order.service.ts:148`). The composition root also raises `NotFoundError` for unmatched routes (`src/app.ts:69-71`), so unknown paths produce the same taxonomy-driven 404 instead of Express's default HTML response.

Two behaviors depend on this rule. First, messages and codes are passed through verbatim, which means domain vocabulary is visible to clients: `EMAIL_ALREADY_USED`, `SKU_ALREADY_USED`, `INACTIVE_PRODUCT`, `INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK` and `INVALID_ORDER_STATE_FOR_DELETE` all reach the wire. Second, because the constructor fixes defaults per subclass, the status/code pair can only vary where a subclass exposes parameters: `BadRequestError` accepts a custom code, `ConflictError` accepts a custom code and details, and `UnprocessableEntityError` accepts a custom code and details. This is what allows `InsufficientStockError` to report 422 while remaining a member of the 422 family, and `InvalidStatusTransitionError` to report 409 while carrying a domain-specific code and `{ from, to }` details.

**Rule workflow**:
```
throw new AppError(message, statusCode, errorCode, details?)
   -> controller catch -> next(err)
   -> errorMiddleware: err instanceof AppError
      -> true
         -> res.status(err.statusCode)
         -> body.error.code    = err.errorCode
         -> body.error.message = err.message
         -> if err.details !== undefined then body.error.details = err.details
         -> return (no logging)
      -> false -> continue to BR-03
```

**Confidence:** High (direct code evidence; branch exercised by integration tests).

---

### Business Rule BR-02: Uniform error envelope with conditional details

**Overview**:
Every error response produced by the application uses one JSON shape: an `error` object containing a machine-readable `code`, a human-readable `message`, and an optional `details` member. No response contains top-level fields such as `status`, `errorCode` or `message`, and no error response mixes the success envelope.

**Detailed description**:
The envelope is constructed independently in each of the five branches, but all five share the same outer key and the same three inner keys. `AppError` populates `code`, `message` and conditionally `details` (`src/middlewares/error.middleware.ts:16-22`); `ZodError` populates the fixed code and message plus a normalized array (`27-33`); `P2002` populates code and a derived message (`40-45`); `P2025` populates code and a fixed message (`49-52`); the fallback populates code and a fixed message (`62-64`). Success payloads are written directly by controllers and list endpoints use `{ data, pagination }` from `src/shared/http/response.ts`; the two envelopes never overlap.

The conditional spread at line 20, `...(err.details !== undefined ? { details: err.details } : {})`, is deliberately not an equality check against a truthy value. An empty object `{}`, an empty array `[]`, the number `0` and the boolean `false` are all serialized because they are not `undefined`. Only an omitted value or an explicit `undefined` suppresses the key. This makes it observable whether an error was constructed with details or without them, which clients can use to distinguish, for example, a plain 404 from a 404 accompanied by diagnostic context. Type `ErrorDetails` permits `Record<string, unknown>`, `unknown[]` or `undefined`, so both object-shaped details (for example `{ from, to }` or `{ unavailable }`) and array-shaped details (the validation issue list) are supported by the same key.

The envelope is the public contract that integration tests assert against: `res.body.error.code` is checked for `EMAIL_ALREADY_USED` (`tests/auth.test.ts:27`), `VALIDATION_ERROR` (`tests/auth.test.ts:37`), `UNAUTHORIZED` (`tests/auth.test.ts:62`), `NOT_FOUND` (`tests/orders.test.ts:56`), `INVALID_STATUS_TRANSITION` (`tests/orders.test.ts:106`), `INSUFFICIENT_STOCK` (`tests/orders.test.ts:127`) and `INVALID_ORDER_STATE_FOR_DELETE` (`tests/orders.test.ts:217`). The array-shaped details contract is asserted in `tests/auth.test.ts:38` and, at field level, in `tests/orders.test.ts:128` (`res.body.error.details.unavailable[0].sku`). No status text, no `success: false` flag and no nested `meta` object is part of the contract.

**Rule workflow**:
```
response body = {
  error: {
    code:    <machine-readable string>,
    message: <human-readable string>,
    details: <present only when the source error carries a defined value>
  }
}
HTTP status = <source-specific: AppError.statusCode | 400 | 409 | 404 | 500>
```

**Confidence:** High.

---

### Business Rule BR-03: Zod validation errors are normalized to 400 VALIDATION_ERROR

**Overview**:
A raw `ZodError` is converted into an HTTP 400 response with code `VALIDATION_ERROR`, message `Validation failed`, and a `details` array that exposes only the dotted path and message of each issue. Zod's internal metadata is never disclosed.

**Detailed description**:
`formatZodIssues` (`src/middlewares/error.middleware.ts:7-12`) maps `error.issues` to plain objects of shape `{ path, message }`. The path is produced by `issue.path.join('.')`, and the empty path is normalized to the literal `(root)`. Zod issue paths are arrays whose members may be strings or numbers (array indices and map keys), so joining produces dotted access paths such as `items.0.productId` for nested arrays and `(root)` for whole-object failures such as a missing required body. Only `path` and `message` are copied; the Zod issue `code`, `expected`, `received`, `inclusive`, `exact`, `options`, `validation` and any union errors are discarded. The resulting details array has the same shape whether the error arrived through this branch or through the `ValidationError` produced upstream, which keeps the contract identical for clients.

In the current request pipeline this branch is a defense-in-depth path rather than the primary path. `validate.middleware.ts:25-33` catches `ZodError` inside each validated route and re-throws `new ValidationError('Validation failed', details)` using an identically formatted `{ path, message }` list. Because `ValidationError` extends `AppError`, the error that reaches the centralized middleware is classified by BR-01 and the `ZodError` branch is skipped. The branch would become active if a `ZodError` were thrown from a schema used outside `validate`, from a `.parse` call inside a service or controller, from a transform executed after the middleware's parse, or from any future route that parses input directly. The two formatting implementations are textually duplicated (`src/middlewares/error.middleware.ts:7-12` and `src/middlewares/validate.middleware.ts:27-30`), which is why the observable output is the same even though the executing branch differs.

The status code is fixed at 400 and is not derived from the Zod error. The message is the fixed string `Validation failed`, so a client that only reads the top-level message cannot tell which field failed; the details array is the only place where field-level information appears. Because the array can contain multiple issues, clients are expected to iterate. The branch returns immediately and does not log, which reflects the classification of validation failures as expected client behavior rather than operational incidents.

**Rule workflow**:
```
ZodError.issues = [{ path: ['email'], message: 'Invalid email' }, { path: [], message: 'Required' }]
   -> map:
        { path: 'email',      message: 'Invalid email' }
        { path: '(root)',     message: 'Required' }
   -> res.status(400).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Validation failed',
          details: [ ...normalized issues ]
        }
      })
```

**Confidence:** High for the transformation; the branch's practical reachability from routed requests is Low because `validate.middleware.ts` converts first.

---

### Business Rule BR-04: Prisma unique-constraint errors become 409 CONFLICT

**Overview**:
When Prisma raises a known request error with code `P2002`, the middleware answers 409 with code `CONFLICT`. If Prisma exposes the constrained column list in `err.meta.target`, the message names the columns; otherwise a generic unique-constraint message is used.

**Detailed description**:
Prisma's `P2002` is the unique-constraint violation code. For the MySQL datasource used here (`prisma/schema.prisma:5-9`), the schema declares unique constraints on `User.email`, `Customer.email`, `Product.sku` and `Order.orderNumber` (`prisma/schema.prisma:27,43,58,76`). The middleware reads `err.meta?.target` and casts it to `string[]`, joins the members with `, ` and produces either `Unique constraint violation on: <columns>` or, when `target` is missing or falsy, `Unique constraint violation` (`src/middlewares/error.middleware.ts:39-44`). The response code is the fixed string `CONFLICT`; no `details` member is emitted, so the column information exists only inside the human-readable message.

The branch is a last line of defense rather than the normal duplicate-detection mechanism. User, customer and product services perform an explicit read-before-write check and throw `ConflictError` with domain-specific codes (`EMAIL_ALREADY_USED`, `SKU_ALREADY_USED`) before Prisma is asked to insert (`src/modules/users/user.service.ts:22-25`, `src/modules/customers/customer.service.ts:30-33`, `src/modules/products/product.service.ts:31-34`). Those pre-checks produce the domain codes asserted by tests (`tests/auth.test.ts:19-28`). `P2002` therefore becomes observable mainly under concurrency, when two requests pass the pre-check and the second insert loses the race at the database constraint, or when a unique column is written without a pre-check (for example an `orderNumber` collision). In those cases the client receives the generic `CONFLICT` code instead of the domain-specific code, because this branch has no access to which unique index was violated beyond the column names.

The cast at line 39 assumes `meta.target` is an array. Prisma's connector-specific metadata can represent the target differently across versions and connectors, and a non-array value would make `.join` fail inside the error handler. The code is written for the MySQL/Prisma 5.22 combination declared in `package.json:26,48`, where the target is an array of column names; the assumption is documented here rather than resolved by a guard. The branch has no request logging, consistent with treating a constraint conflict as an expected client-facing outcome.

**Rule workflow**:
```
PrismaClientKnownRequestError
   -> err.code === 'P2002'
      -> target = err.meta?.target as string[] | undefined
      -> message = target ? `Unique constraint violation on: ${target.join(', ')}`
                          : 'Unique constraint violation'
      -> res.status(409).json({ error: { code: 'CONFLICT', message } })
      -> return
```

**Confidence:** High for the mapping logic; Medium for the `meta.target` runtime shape on connectors other than MySQL; branch execution in the current test suite is not evidenced.

---

### Business Rule BR-05: Prisma record-not-found errors become 404 NOT_FOUND

**Overview**:
When Prisma raises a known request error with code `P2025`, the middleware answers 404 with code `NOT_FOUND` and the fixed message `Resource not found`.

**Detailed description**:
`P2025` is Prisma's "operation depends on one or more records that were required but not found" error, typically raised by `update` or `delete` when the target row no longer exists. The middleware maps it to 404 with the fixed message `Resource not found` and no details (`src/middlewares/error.middleware.ts:48-53`). Unlike the application-level `NotFoundError`, which accepts a resource name and produces `Product not found`, `Order not found`, `Customer not found` or `User not found` (`src/shared/errors/http-errors.ts:27-31`), this branch has no way to know which entity was missing, so the response is deliberately generic.

The branch is reachable when a read-then-write sequence races with another request. Services such as product deletion call `getById` and then delete (`src/modules/products/product.service.ts:63-65`), so a row removed between the two operations makes Prisma raise `P2025`. The same applies to updates and to order operations that read an order and then update it. In non-concurrent operation the service's explicit `NotFoundError` is thrown first and the response carries the resource-specific message, so this branch is not the normal path for a missing resource.

The fixed-message behavior has a client-visible consequence: two 404s with code `NOT_FOUND` may carry different messages (`Order not found` for the pre-checked path versus `Resource not found` for the persistence race), and no `details` are available to disambiguate. Because the branch returns immediately, nothing is logged. The mapping is nevertheless important because without it a `P2025` would be treated as an unknown failure and surfaced as 500, which would misclassify a client-addressable absence as a server fault.

**Rule workflow**:
```
PrismaClientKnownRequestError
   -> err.code === 'P2025'
      -> res.status(404).json({
           error: { code: 'NOT_FOUND', message: 'Resource not found' }
         })
      -> return
```

**Confidence:** High for the mapping; branch execution in the current test suite is not evidenced.

---

### Business Rule BR-06: Unrecognized errors are masked as 500 and logged

**Overview**:
Every error that is not an `AppError`, a `ZodError`, or a Prisma `P2002`/`P2025` is treated as an unhandled failure. It is logged with structured context and returned to the client as HTTP 500 with code `INTERNAL_SERVER_ERROR` and the fixed message `Internal server error`.

**Detailed description**:
The fallback executes after all four classification attempts fail (`src/middlewares/error.middleware.ts:56-64`). It builds a structured log record with four fields: the raw error under `err`, the request correlation id under `requestId`, the HTTP method under `method`, and the full original URL under `path`. The message is the fixed string `Unhandled error in request`. It then writes `500` with `{ error: { code: 'INTERNAL_SERVER_ERROR', message: 'Internal server error' } }`. Neither the error message, the stack trace, the error name nor any library-specific metadata is copied into the response. This is the component's principal information-disclosure control: internal implementation details, SQL fragments, file paths and stack frames remain server-side.

The fallback is reached by a broad class of failures. Body-parser errors produced by `express.json({ limit: '1mb' })` (`src/app.ts:59`) belong here: malformed JSON raises a `SyntaxError` carrying `status = 400` and `type = 'entity.parse.failed'`, and an oversized body raises an error carrying `status = 413` and `type = 'entity.too.large'`. Neither is an `AppError`, `ZodError` or Prisma error, so both are masked as 500 even though the library attached an accurate 4xx status. Unmapped Prisma codes also belong here, including `P2003` foreign-key restriction, which is what a delete of a customer or product referenced by an order produces (`prisma/schema.prisma:87-88,108-109`). Any programming error (type error, null dereference), any `jwt` failure that escapes a guard, any logging failure and any library error also belong here. The middleware intentionally ignores `err.status` and `err.statusCode` on arbitrary objects; only the `AppError` branch consumes a status field.

The server-side record is emitted through the shared Pino singleton (`src/shared/logger/index.ts:32`), which applies the configured `LOG_LEVEL`, ISO timestamps, the `service`/`env` base fields and the redaction list (`authorization`, `cookie`, `password`, `passwordHash`, `token`, `accessToken`). The response is written inside a function that returns nothing to the caller; the return statements in earlier branches prevent the fallback from running, but there is no `return` after the final `res.status(500).json(...)` because it is the end of the function. The branch therefore guarantees that the response cycle is completed for every input error object, including non-`Error` values such as strings or `null`, because `instanceof` checks on primitives are simply false.

**Rule workflow**:
```
unknown error (SyntaxError, P2003, TypeError, string, null, ...)
   -> requestId = (req as { id?: string }).id ?? 'unknown'
   -> logger.error({ err, requestId, method: req.method, path: req.originalUrl },
                   'Unhandled error in request')
   -> res.status(500).json({
        error: { code: 'INTERNAL_SERVER_ERROR', message: 'Internal server error' }
      })
```

**Confidence:** High.

---

### Business Rule BR-07: The handler is terminal and never delegates

**Overview**:
The middleware must be the final middleware in the Express stack. It uses the four-argument error-handler signature, processes exactly one error and writes exactly one response. It never calls `next`, so it does not delegate to the Express default error handler or to any subsequent error middleware.

**Detailed description**:
Express recognizes an error handler by its arity of four. `errorMiddleware` is declared as `(err, req, res, _next)` (`src/middlewares/error.middleware.ts:14`). The underscore prefix is required by the project's ESLint configuration, which ignores unused arguments matching `^_` (`.eslintrc.json:19-22`). The parameter is present only to satisfy the framework contract; it is never invoked. The composition root registers the handler after the 404 fallback (`src/app.ts:69-73`), which is the only position from which it can observe errors raised by `express.json`, the request logger, `/health`, the API router, module routers, guards, validation and controllers. Moving it earlier would leave later errors unhandled; the current ordering is therefore part of the rule.

Each classification branch terminates with `return`, and the fallback terminates by exhausting the function. This produces a deterministic single-response behavior for ordinary errors: the handler does not partially write a response and then pass the error on. It also means there is no error-handler chain, no place to add a second translator later without editing this function, and no path by which Express's default handler can add its own HTML body or stack trace in development.

The absence of delegation has a boundary condition: Express's default error handler is the only component that checks `res.headersSent` and, when headers are already sent, destroys the connection instead of writing a second response. This handler performs no such check. If an error occurs after a response has begun (for example a failure inside a streaming write, a second `res.json` call, or an error raised by a listener after the response started), `res.status(...).json(...)` would attempt to modify headers after they were sent. That condition is not exercised by the current tests and is documented under technical debt rather than as a designed behavior.

**Rule workflow**:
```
Express pipeline (last registered handler, src/app.ts:73)
   -> errorMiddleware(err, req, res, _next)
      -> exactly one branch writes res
      -> return
      -> no next(err)
      -> no headersSent inspection
```

**Confidence:** High for the registration and arity; High for the absence of `headersSent` handling; the post-response failure scenario is inferred from Express semantics rather than observed.

---

### Business Rule BR-08: Status and code are accepted from the error without validation

**Overview**:
The middleware trusts the `statusCode` and `errorCode` stored on an `AppError` and writes them unchanged. It does not validate that the status is a legal HTTP status, that the code follows a naming convention, or that the pair is coherent.

**Detailed description**:
`AppError` stores constructor arguments verbatim: `statusCode: number`, `errorCode: string` and `details` (`src/shared/errors/app-error.ts:8-13`). Nothing in the constructor validates range, integer-ness or membership in the HTTP status set. Every concrete subclass in `src/shared/errors/http-errors.ts` supplies a well-formed pair, so the built-in taxonomy is coherent by construction, but the base class remains publicly constructible and extensible. The middleware's branch simply forwards `err.statusCode` to `res.status(...)` and `err.errorCode` to the body (`src/middlewares/error.middleware.ts:16-18`). Consequences of an invalid value would surface at the Express layer (for example an out-of-range status) or at the client (an unexpected code string), not inside this component.

The same trust applies to `message`. A developer-supplied `AppError` message is returned to the client even if it contains internal terminology. This is intentional for domain errors, which carry user-actionable text such as `Invalid status transition from PENDING to SHIPPED` or `Order can only be deleted while in PENDING or CANCELLED status`. The masking rule applies only to errors that are not `AppError`, so the taxonomy itself is the boundary between safe-to-disclose and must-mask.

The code namespace is also unvalidated. The generic subclasses contribute `BAD_REQUEST`, `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT` and `UNPROCESSABLE_ENTITY`; parameterized call sites add `EMAIL_ALREADY_USED`, `SKU_ALREADY_USED`, `INACTIVE_PRODUCT`, `INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK` and `INVALID_ORDER_STATE_FOR_DELETE`. Because the middleware does not normalize case or assert uniqueness, collisions between a generic and a domain-specific code would be indistinguishable to a client except by message and `details`. This is a property of the shared taxonomy, observable through the middleware, rather than a transformation performed here.

**Rule workflow**:
```
AppError instance
   -> statusCode used as-is by res.status(statusCode)
   -> errorCode used as-is in body.error.code
   -> message used as-is in body.error.message
   -> no range, type or naming validation inside the middleware
```

**Confidence:** High.

---

### Business Rule BR-09: Classification closure for Prisma and library errors

**Overview**:
Among Prisma known request errors, only `P2002` and `P2025` are translated. Every other Prisma code and every non-Prisma library error is treated as an unhandled failure and answered with 500. This is an implicit rule expressed by the absence of additional branches.

**Detailed description**:
The Prisma branch is entered only for `Prisma.PrismaClientKnownRequestError` (`src/middlewares/error.middleware.ts:37`). Inside it, two codes are compared. `P2002` returns a 409, `P2025` returns a 404, and any other code exits the `if` chain without returning, causing execution to continue to the fallback logging and 500 response. Prisma's other known codes therefore do not have HTTP semantics in this application: `P2000` (value too long for a column), `P2001` (record does not exist for a `where`), `P2003` (foreign-key constraint failed), `P2004` (constraint failed on the database), `P2011` (null constraint violation), `P2014` (required relation violation), `P2028` (transaction API error) and connector-specific codes all become `500 INTERNAL_SERVER_ERROR`. Prisma's separate `PrismaClientValidationError`, `PrismaClientInitializationError`, `PrismaClientUnknownRequestError` and `PrismaClientRustPanicError` classes are not instances of the known-request class and never enter the branch at all.

The schema makes one of these gaps concretely reachable. `Order.customer` and `Order.createdBy` have no `onDelete` behavior, and `OrderItem.product` has no `onDelete` behavior (`prisma/schema.prisma:87-88,108-109`), so MySQL's default restrict behavior applies. Deleting a customer or product that is referenced by an order therefore raises `P2003`. The product and customer services check existence and then delete (`src/modules/products/product.service.ts:63-65`, `src/modules/customers/customer.service.ts:60-62`), with no relationship pre-check, so the failure is passed to the middleware and classified as unknown. The client receives 500 rather than a 409 or 422, and the error is written to the error log with the full Prisma error object.

The same closure behavior applies to body-parser. `express.json` errors carry their own `status`/`statusCode` and `type` but are not in the recognized taxonomy, so malformed JSON is answered with 500 instead of 400 and an oversized body with 500 instead of 413. This is an accurate description of the implemented behavior, not an inferred intention: the fallback is unconditional for anything that is not one of the five recognized shapes. The rule is therefore best stated positively (only two Prisma codes and two non-Prisma classes are translated) and negatively (all remaining failures are masked).

**Rule workflow**:
```
PrismaClientKnownRequestError
   -> code === 'P2002' ? 409 CONFLICT
   -> code === 'P2025' ? 404 NOT_FOUND
   -> otherwise        -> fallback BR-06 -> 500 INTERNAL_SERVER_ERROR

PrismaClientValidationError / InitializationError / UnknownRequestError / RustPanicError
   -> not an instance of PrismaClientKnownRequestError -> fallback BR-06 -> 500
Body-parser SyntaxError / entity.too.large
   -> not an AppError/ZodError/PrismaKnownRequestError -> fallback BR-06 -> 500
```

**Confidence:** High (explicit code structure). The `P2003` scenario is inferred from the Prisma schema and default referential actions, consistent with the architectural report.

---

### Business Rule BR-10: Error logging is correlated to the request id

**Overview**:
The fallback log record always includes a request correlation id. The id comes from the request object populated by the request logger middleware; when it is absent, the literal string `unknown` is used. The log also carries the HTTP method and the full original URL.

**Detailed description**:
`requestLogger` is mounted before routing (`src/app.ts:60`). It takes `x-request-id` from the incoming headers when present, otherwise generates a UUID v4, assigns it to `req.id`, and echoes it in the `X-Request-Id` response header (`src/middlewares/request-logger.middleware.ts:5-8`). The same middleware registers a `finish` listener that logs method, path, status code, duration and optional user id (`12-25`). The error middleware reads `(req as { id?: string }).id` and substitutes `'unknown'` when the value is `undefined` (`src/middlewares/error.middleware.ts:56`). Because the `id` property is declared through module augmentation in `src/middlewares/auth.middleware.ts:12-17` rather than in a shared request type, the cast is required.

The correlation contract allows an operator to join the access log line (`http_request`) with the error log line (`Unhandled error in request`) by `requestId`, and to join both with the `X-Request-Id` value that the client received. The error record includes the raw error object, which Pino serializes with its stack, plus the method and `req.originalUrl`. The access record includes the final status code and duration, so the pair together describes both the failure and its observable HTTP outcome. This is the only observability mechanism attached to this component; there are no metrics, traces or counters.

Two boundary conditions follow from the rule. First, if the request logger is not mounted (for example a unit test that invokes the middleware directly, or a future app composition that omits it), every error log shares the literal id `unknown`, which removes the join key and collapses distinct failures into one correlation value. Second, `originalUrl` includes the query string, so search parameters and any other query data are written to the error log. The logger's redaction list covers credential-shaped keys (`src/shared/logger/index.ts:4-11`) but not PII embedded in a URL. Because only the fallback branch logs, expected 4xx classifications produce no error log line at all; they appear only in the access log with their status.

**Rule workflow**:
```
requestLogger: req.id = x-request-id header ?? uuidv4()   (src/middlewares/request-logger.middleware.ts:6-7)
   -> downstream error
   -> errorMiddleware fallback
      -> requestId = req.id ?? 'unknown'
      -> logger.error({ err, requestId, method, path: req.originalUrl },
                      'Unhandled error in request')
      -> 500 response (client can correlate via X-Request-Id header)
```

**Confidence:** High.

---

## 4. Component Structure

### 4.1 Component boundary

The analyzed component is the single file `src/middlewares/error.middleware.ts`. Its contract, however, is defined by the shared error taxonomy it consumes and by the logger it calls, so those files are analyzed as immediate collaborators. The validation middleware and request logger are analyzed because they produce the inputs the component classifies (`ValidationError`, `req.id`) and because one of them duplicates the Zod formatting logic.

```
src/
├── middlewares/
│   ├── error.middleware.ts          # ANALYZED COMPONENT (65 lines)
│   │   ├── formatZodIssues()        # Zod issue -> { path, message }[] (lines 7-12)
│   │   └── errorMiddleware          # ErrorRequestHandler (lines 14-65)
│   │       ├── AppError branch                    (lines 15-24)
│   │       ├── ZodError branch                    (lines 26-35)
│   │       ├── Prisma P2002 branch                (lines 38-47)
│   │       ├── Prisma P2025 branch                (lines 48-53)
│   │       └── fallback: log + 500                (lines 56-64)
│   ├── validate.middleware.ts       # Upstream producer of ValidationError;
│   │                                #   duplicates Zod path formatting (25-33)
│   ├── request-logger.middleware.ts # Populates req.id and X-Request-Id (5-8)
│   └── auth.middleware.ts           # Produces Unauthorized/Forbidden; declares req.id (12-17)
├── shared/
│   ├── errors/
│   │   ├── app-error.ts             # AppError base: statusCode/errorCode/details (3-16)
│   │   ├── http-errors.ts           # 9 concrete subclasses (3-63)
│   │   └── index.ts                 # Barrel re-export (1-13)
│   └── logger/
│       └── index.ts                 # Pino singleton + redaction (4-32)
├── app.ts                           # Imports and mounts the handler last (4, 69-73)
├── modules/                         # Error producers (services, controllers)
│   ├── auth/
│   ├── users/
│   ├── customers/
│   ├── products/
│   └── orders/
└── routes/
    └── index.ts                     # /api/v1 aggregation (21-31)
```

### 4.2 Internal organization and size

| Element | Lines | Responsibility |
|---------|-------|----------------|
| Imports and helper | 1-12 | Type import of `ErrorRequestHandler`; runtime imports of `Prisma`, `ZodError`, `AppError`, `logger`; `formatZodIssues` |
| Handler signature | 14 | Four-argument Express error handler, `_next` intentionally unused |
| AppError classification | 15-24 | Status, code, message, conditional details; no logging |
| Zod classification | 26-35 | Fixed 400 envelope with normalized issue list; no logging |
| Prisma classification | 37-54 | `P2002` -> 409, `P2025` -> 404; no logging |
| Fail-safe | 56-64 | Correlation-aware `logger.error`, fixed 500 envelope |
| Total | 65 | Single exported symbol: `errorMiddleware` |

The file has no class, no state, no configuration reads, no dependency injection and no initialization. Its only exports are `errorMiddleware`; `formatZodIssues` is module-private. There are no environment-dependent branches: behavior is identical in development, test and production. The logger's transport differs by `NODE_ENV`, but that difference lives in `src/shared/logger/index.ts:22-28`, outside the component.

### 4.3 Position in the middleware pipeline

```
app.disable('x-powered-by')            src/app.ts:58
app.use(express.json({ limit: '1mb' })) src/app.ts:59
app.use(requestLogger)                  src/app.ts:60
app.get('/health')                      src/app.ts:62-64
app.use('/api/v1', buildApiRouter(...)) src/app.ts:67
app.use(404 -> NotFoundError)           src/app.ts:69-71
app.use(errorMiddleware)                src/app.ts:73   <-- analyzed component
```

Every layer above can produce the inputs this component classifies. `express.json` produces body-parser errors; `requestLogger` produces the correlation id and can in principle fail; the router and guards produce `AppError` and `ZodError`; controllers forward everything through `next(err)`. No layer below the handler exists, so the handler is the last executable component in the request lifecycle.

---

## 5. Dependency Analysis

### 5.1 Internal dependency graph (static imports)

```
src/app.ts
   |  (import, line 4; mount, line 73)
   v
src/middlewares/error.middleware.ts
   |------------------------------------+
   |                                    |
   v                                    v
src/shared/errors/index.ts          src/shared/logger/index.ts
   |                                    |
   v                                    v
app-error.ts + http-errors.ts       src/config/env.ts
   (AppError, ErrorDetails)          (LOG_LEVEL, NODE_ENV)
```

Runtime (non-import) dependency edges into the component:

```
src/middlewares/validate.middleware.ts  --next(new ValidationError(...))--> errorMiddleware  (indirect, lines 31-34)
src/middlewares/auth.middleware.ts      --next(new Unauthorized/Forbidden)--> errorMiddleware (indirect, lines 30,36,45,52,56)
src/middlewares/request-logger.middleware.ts --req.id / X-Request-Id-------> errorMiddleware  (request state, lines 6-8)
src/modules/**/*.service.ts             --throw AppError subclasses-------> errorMiddleware  (indirect via controller next)
src/modules/**/*.controller.ts          --try/catch + next(err)-----------> errorMiddleware  (e.g. orders controller 14-16)
src/app.ts (404 fallback)               --next(new NotFoundError(...))----> errorMiddleware  (line 70)
@prisma/client (runtime throw)          --PrismaClientKnownRequestError---> errorMiddleware  (P2002/P2025 only)
zod (runtime throw)                     --ZodError------------------------> errorMiddleware  (branch 26-35)
express.json (runtime throw)            --SyntaxError / entity.too.large--> errorMiddleware  (unclassified -> 500)
```

### 5.2 Dependency edges and nature

| From | To | Type | Binding | Evidence |
|------|----|------|---------|----------|
| `src/app.ts` | `src/middlewares/error.middleware.ts` | import + registration | static / runtime | `src/app.ts:4,73` |
| `src/middlewares/error.middleware.ts` | `src/shared/errors/index.ts` | import | static / runtime | `src/middlewares/error.middleware.ts:4` |
| `src/middlewares/error.middleware.ts` | `src/shared/logger/index.ts` | import | static / runtime | `src/middlewares/error.middleware.ts:5` |
| `src/middlewares/validate.middleware.ts` | `src/middlewares/error.middleware.ts` | indirect via `next(err)` | runtime | `src/middlewares/validate.middleware.ts:31,34` |
| `src/middlewares/request-logger.middleware.ts` | `src/middlewares/error.middleware.ts` | request state (`req.id`) | runtime | `request-logger.middleware.ts:7`; `error.middleware.ts:56` |
| `src/middlewares/auth.middleware.ts` | `src/middlewares/error.middleware.ts` | indirect via `next(err)` | runtime | `auth.middleware.ts:30,36,45,52,56` |
| Services / controllers (5 modules) | `src/middlewares/error.middleware.ts` | indirect via `next(err)` | runtime | e.g. `order.controller.ts:14-16`, `order.service.ts:46-223` |
| `src/shared/logger/index.ts` | `src/config/env.ts` | import | static / runtime | `logger/index.ts:2` |

### 5.3 External dependencies

| Package | Version | Usage in this component | API surface used |
|---------|---------|--------------------------|------------------|
| `express` | 4.21.1 | Error-handler signature and response API | `ErrorRequestHandler`, `res.status`, `res.json` |
| `@prisma/client` | 5.22.0 | Error classification only; no database access | `Prisma.PrismaClientKnownRequestError`, `err.code`, `err.meta.target` |
| `zod` | 3.23.8 | Error classification only; no schema execution | `ZodError`, `error.issues`, `issue.path`, `issue.message` |
| `pino` | 9.5.0 (transitive via `shared/logger`) | Structured error logging | `Logger.error(obj, msg)` |
| `pino-pretty` | 11.3.0 (dev-only, via logger factory) | Development log transport | not called directly |
| Node.js | `>=20` (`package.json:8`; runtime observed v24.13.0) | Runtime for `Error`, `instanceof` | standard |

The component performs no network I/O, no database access and no filesystem access. `@prisma/client` is imported for its error class only, which means the middleware carries an ORM dependency despite being an HTTP-layer component.

### 5.4 Dependency direction and stability

The component sits between the application taxonomy and the HTTP boundary. It depends on `shared/errors` (a stable shared kernel with 11 afferent importers and zero internal dependencies) and on `shared/logger`, which in turn depends on the validated environment configuration. It is depended on, statically, by exactly one file (the composition root). All other dependencies are runtime control-flow edges produced by `next(err)` and by thrown errors, not imports. This yields a stable, low-fan-in node whose change impact is nevertheless system-wide because it defines the response contract.

---

## 6. Afferent and Efferent Coupling

Methodology: afferent coupling (Ca) counts distinct internal source files that import the module; efferent coupling (Ce) counts distinct internal source files the module imports, excluding npm packages. Counts are static and include type-only imports. For collaborators, the package-level relationship to the analyzed component is described in the notes.

| Component | Afferent Coupling | Efferent Coupling | Critical | Notes |
|-----------|-------------------|-------------------|----------|-------|
| `errorMiddleware` (`src/middlewares/error.middleware.ts`) | 1 | 2 | High | Fan-in from `src/app.ts` only; fan-out to `shared/errors/index.ts` and `shared/logger/index.ts`; plus 3 external packages. Critical because it is the only error-to-HTTP translator |
| `formatZodIssues` (module-private helper) | 1 (same file) | 0 | Low | Pure transformation; not exported; duplicate logic exists in `validate.middleware.ts` |
| `AppError` (`src/shared/errors/app-error.ts`) | 1 direct + 9 subclasses in same package | 0 | High | Base of the taxonomy consumed by the middleware; any field change alters the response envelope |
| Error taxonomy barrel (`src/shared/errors/index.ts`) | 11 | 0 | High | Highest afferent coupling in the project; the contract between services, guards, middleware and controllers |
| `logger` singleton (`src/shared/logger/index.ts`) | 3 | 1 | Medium | Used by server bootstrap, request logger and this component; depends on `env.ts` |
| `requestLogger` (`src/middlewares/request-logger.middleware.ts`) | 1 | 1 | Medium | Supplies `req.id`; its presence determines whether fallback logs are correlatable |
| `validate` (`src/middlewares/validate.middleware.ts`) | 5 | 1 | Medium | Upstream producer of `ValidationError`; duplicates Zod issue formatting |
| `auth.middleware.ts` producers | 5 | 2 | Medium | Two guards produce four `AppError` variants consumed here |
| External: `ZodError` contract | n/a | n/a | Medium | Only `issues[].path` and `issues[].message` consumed |
| External: `PrismaClientKnownRequestError` contract | n/a | n/a | Medium | Only `code` and `meta.target` consumed |

Interpretation:

- **Low afferent coupling is expected for a terminal cross-cutting handler.** Nothing imports it except the composition root; all other relationships are runtime error flow. Low Ca does not reduce criticality because the component is on the only path from internal failure to client response.
- **Efferent coupling is concentrated in two internal modules.** One is a pure type/value contract (`shared/errors`) and one is infrastructure (`shared/logger`). Neither is volatile, so the component's exposure to internal change is limited.
- **The external coupling to `@prisma/client` is a layering tension.** An HTTP middleware imports an ORM error class. This couples the response contract to Prisma's error-code taxonomy, and additions to Prisma's code set require edits here to be translated.
- **The duplicate Zod formatting in `validate.middleware.ts` is a parallel, not a coupling.** The two implementations are not linked by import, so they can diverge without a compile error; the only safeguard is the identical shape produced today.

---

## 8. Integration Points

The component exposes no endpoints of its own. It participates in every route by terminating the error path; the endpoint surface it affects is the full `/api/v1` router plus `/health` and the 404 fallback, as listed in `src/app.ts:62-73`.

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Express error pipeline | Framework integration | Receives every error passed to `next(err)` and writes the terminal response | In-process function call | JavaScript `Error` object -> `res.status().json()` | This component is the handler; no further delegation (`_next` unused) |
| `express.json` body parser | Framework integration (upstream, unclassified) | Produces `SyntaxError` (malformed JSON) and payload-too-large errors before routing | In-process | body-parser error objects with `status`/`type` | Not recognized -> logged and masked as 500 (`src/app.ts:59`, `error.middleware.ts:56-64`) |
| Zod (`ZodError`) | Validation library contract | Provides `issues[]` with `path` and `message` for normalization | In-process | `ZodError` -> `{ path, message }[]` | Fixed 400 `VALIDATION_ERROR`; malformed issue objects are not defended against (`error.middleware.ts:7-12,26-35`) |
| Prisma ORM (`PrismaClientKnownRequestError`) | ORM error contract | Supplies `code` and `meta.target` for conflict/absence translation | In-process | Prisma error object -> 409/404 JSON | Only `P2002`/`P2025` mapped; all other codes fall to 500 (`error.middleware.ts:37-54`) |
| Pino logger | Observability infrastructure | Emits one structured error record for unhandled failures | In-process | Pino JSON record (`err`, `requestId`, `method`, `path`) with redaction | No try/catch around `logger.error`; a logger failure would propagate out of the handler (`error.middleware.ts:57-60`) |
| Request logger middleware | Internal middleware | Supplies `req.id` and `X-Request-Id` for log/response correlation | In-process request state | string id | Fallback literal `'unknown'` when absent (`request-logger.middleware.ts:6-8`; `error.middleware.ts:56`) |
| HTTP clients (consumers) | Downstream contract | Receive the error envelope and status | HTTP/1.1 (Express) | JSON `{ error: { code, message, details? } }` | Contract asserted by integration tests; 500 responses carry no correlation id in the body, only in the `X-Request-Id` header |
| MySQL 8.0 (indirect) | Data store | Origin of Prisma errors classified here | Prisma/MySQL protocol | Prisma error metadata | Indirect only; the component never queries the database |

Integration observations:

- The component's integration surface is entirely in-process. No network call is made and no timeout, retry or circuit breaker applies.
- There is no contract test for the Prisma branches; the only Prisma-dependent assertions are 404/409/422 responses produced by application-level `AppError`s.
- The logger integration is synchronous from the component's perspective. Pino's destination may buffer, so a logging failure is not handled.
- The `X-Request-Id` header set upstream is the only correlation channel available to clients, because the error body does not include it.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Chain of Responsibility (terminal handler) | Single four-argument Express error handler registered last | `src/middlewares/error.middleware.ts:14`; `src/app.ts:73` | Centralize error termination in one place |
| Centralized Exception Translation / Anti-Corruption Layer | Converts domain/library errors into the HTTP JSON envelope | `src/middlewares/error.middleware.ts:15-64` | Isolate the wire contract from internal error types |
| Error Taxonomy / Shared Kernel | `AppError` base plus nine concrete subclasses re-exported by a barrel | `src/shared/errors/app-error.ts`, `http-errors.ts`, `index.ts` | Canonical, machine-readable error vocabulary across layers |
| Polymorphic Dispatch via `instanceof` | Ordered type checks select the branch; subclasses inherit handling | `src/middlewares/error.middleware.ts:15,26,37` | Extensible classification without a registry |
| Fail-Safe / Fail-Closed Default | Unknown errors are masked as generic 500 after logging | `src/middlewares/error.middleware.ts:56-64` | Prevent information disclosure |
| Information Hiding | Fixed message for unhandled errors; only `path`/`message` from Zod issues | `src/middlewares/error.middleware.ts:9-10,62-64` | Avoid leaking stacks, SQL, internals |
| Adapter (error -> transport) | Maps `statusCode`/`errorCode`/`details` to `res.status`/JSON | `src/middlewares/error.middleware.ts:16-22` | Decouple error representation from HTTP serialization |
| Null Object (correlation fallback) | `req.id ?? 'unknown'` | `src/middlewares/error.middleware.ts:56` | Guarantee a log field is always present |
| Middleware Pipeline (pipe-and-filter) | Registered in a fixed order in the composition root | `src/app.ts:58-73` | Ordered cross-cutting concerns |
| Composition Root | Handler imported and mounted once | `src/app.ts:4,73` | Single wiring point, no DI container |
| Template Method (response shape) | Every branch emits the same envelope skeleton | `src/middlewares/error.middleware.ts:16-22,27-33,40-45,49-52,62-64` | Contract consistency across error classes |

Architectural decisions observed:

1. **One terminal error handler, no chain.** All errors end in this function; no second handler exists and `_next` is unused. The decision simplifies reasoning but removes the Express-native fallback for post-response failures.
2. **Error codes are first-class, statuses are derived from error types.** Clients can branch on `code` rather than parsing messages, and domain-specific codes coexist with generic ones.
3. **The domain throws HTTP-aware errors.** `AppError` carries `statusCode`, so service and middleware layers encode HTTP semantics. This keeps controllers thin and the translation table small, at the cost of HTTP awareness below the adapter layer.
4. **Fail-closed default with server-side diagnostics.** Unknown failures are never echoed; the diagnostic record is emitted through the same Pino singleton used by request logging.
5. **Trust in the taxonomy.** No validation of status/code values, no normalization, no code registry. Correctness depends on the subclasses and call sites that construct `AppError`.
6. **Library error recognition is minimal.** Prisma and Zod are recognized by class identity and a small subset of their fields, not by a mapping table or a general `status` convention.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Body-parser failure path | `express.json` raises `SyntaxError` (400) and payload-too-large (413) errors that are not in the recognized taxonomy, so they are logged and answered as 500 (`src/app.ts:59`; `src/middlewares/error.middleware.ts:14-64`) | Clients receive 500 for invalid JSON and oversized bodies; contract contradicts HTTP semantics; malformed client input pollutes error logs |
| High | Response lifecycle | No `res.headersSent` check and no delegation to the Express default handler (`_next` unused, `src/middlewares/error.middleware.ts:14-65`) | An error raised after the response started causes a second write attempt (`ERR_HTTP_HEADERS_SENT`), loses Express's connection-destroy fallback and can surface a secondary failure |
| Medium | Prisma coverage | Only `P2002` and `P2025` are mapped; `P2003` foreign-key restriction and other known codes become 500 (`src/middlewares/error.middleware.ts:37-54`; schema relations `prisma/schema.prisma:87-88,108-109`) | Deleting a referenced customer or product returns 500 instead of a client-actionable 409/422; full Prisma error object is logged |
| Medium | Prisma metadata handling | `err.meta?.target` is cast to `string[]` and `.join(', ')` is called unconditionally when present (`src/middlewares/error.middleware.ts:39`) | If a connector reports `target` as a non-array, the error handler itself throws, producing a secondary unhandled failure |
| Medium | Duplicate validation formatting | Zod issue normalization is implemented twice (`src/middlewares/error.middleware.ts:7-12` and `src/middlewares/validate.middleware.ts:27-30`) | The two copies can drift, splitting the `details` contract between the primary and fallback paths |
| Medium | Unreachable classification branch | The `ZodError` branch is effectively unreachable for validated routes because `validate.middleware.ts:31` wraps Zod errors in `ValidationError` first | Dead-ish code path carries maintenance cost and is untested; any future direct `.parse` call would use a different code path than routed validation |
| Medium | Test coverage | No unit test file for the middleware; the fallback, both Prisma branches, the `ZodError` branch and the correlation fallback are unverified (`tests/auth.test.ts`, `tests/orders.test.ts` only) | Regressions in masking, logging or Prisma mapping would not be caught by the suite |
| Medium | Status/code validation | `AppError` accepts any number/string and the middleware forwards them (`src/shared/errors/app-error.ts:8-13`; `error.middleware.ts:16-18`) | A programming error can emit an invalid HTTP status or an inconsistent code pair; no compile-time or runtime guard exists |
| Low | Details serialization | `details` is typed as arbitrary `unknown`/`Record<string, unknown>`/array and serialized without inspection (`src/middlewares/error.middleware.ts:20`) | A non-serializable value (circular reference, BigInt) would make `res.json` throw inside the handler |
| Low | Correlation fallback | Missing `req.id` collapses all error logs to `unknown` (`src/middlewares/error.middleware.ts:56`) | Direct invocation or a composition without `requestLogger` removes the only log join key |
| Low | Log content | `req.originalUrl` is logged in full, including query strings (`src/middlewares/error.middleware.ts:58`) | Search terms or identifiers in query parameters can enter error logs outside the redaction list |
| Low | Error-code governance | Codes are string literals spread across `AppError` subclasses and call sites; no registry or uniqueness check | Client-facing code drift and generic/domain collisions are unverifiable statically |
| Low | ORM leakage into HTTP layer | The middleware imports `Prisma` to recognize database errors (`src/middlewares/error.middleware.ts:2`) | The HTTP contract is coupled to the ORM's error taxonomy and version |
| Low | Logger failure | `logger.error` is not guarded (`src/middlewares/error.middleware.ts:57-60`) | A logging exception would escape the error handler after no response has been written |
| Low | No error metrics | Only a log line is produced for unhandled failures (`src/middlewares/error.middleware.ts:57-60`) | Error-rate observability depends entirely on log aggregation; no counters or traces exist |
| Low | Method-not-allowed semantics | Unmatched methods reach the 404 fallback and produce `NotFoundError` rather than 405 (`src/app.ts:69-71`) | Clients cannot distinguish an unknown path from an unsupported method; behavior is outside this component but visible through it |

Positive controls observed:

- No stack trace, error name, SQL fragment or internal message is returned for unhandled errors (`src/middlewares/error.middleware.ts:62-64`).
- Zod issue metadata beyond `path` and `message` is discarded (`src/middlewares/error.middleware.ts:8-11`).
- Error logging uses the shared Pino instance with credential redaction (`src/shared/logger/index.ts:4-20`).
- Domain messages are only exposed through the intentional `AppError` taxonomy, never through arbitrary error objects.
- The terminal handler is registered exactly once, after the 404 fallback, so no route can bypass HTTP error serialization (`src/app.ts:69-73`).

---

## 11. Test Coverage Analysis

### 11.1 Test inventory and locations

| File | Tests | Relation to the component |
|------|-------|---------------------------|
| `tests/auth.test.ts` | 7 integration tests | 4 exercise the error path: duplicate email 409 `EMAIL_ALREADY_USED` (lines 19-28), invalid payload 400 `VALIDATION_ERROR` with array details (30-39), wrong password 401 `UNAUTHORIZED` (54-63), `/auth/me` without token 401 (79-83) |
| `tests/orders.test.ts` | 8 integration tests | 4 exercise the error path: non-existent product 404 `NOT_FOUND` (42-57), invalid transition 409 `INVALID_STATUS_TRANSITION` (89-107), insufficient stock 422 `INSUFFICIENT_STOCK` with nested details (109-132), delete of non-deletable order 409 `INVALID_ORDER_STATE_FOR_DELETE` (197-218) |
| `tests/helpers/factories.ts` | n/a (helpers) | Builds the real Express app through `buildApp({ prisma })` (lines 10-15), so every integration test uses the actual middleware |
| `tests/setup.ts` | n/a (lifecycle) | Connects Prisma and truncates tables before each test (lines 4-16) |
| `tests/**/*.test.ts` include pattern | n/a | `vitest.config.ts:7` includes only `tests/**/*.test.ts`; there is no unit-test directory and no `error.middleware.test.ts` |
| Coverage tooling | none | `package.json:17` runs `vitest run`; no `test:coverage` script and no coverage provider/threshold in `vitest.config.ts` |

No tests exist for the middleware as a unit. Every observation below is derived statically by mapping the code branches to the assertions in the two integration files; no test execution or coverage instrumentation was performed.

### 11.2 Coverage by component and branch

| Component / Branch | Unit Tests | Integration Tests (indirect) | Estimated Coverage | Test Quality |
|--------------------|------------|------------------------------|--------------------|--------------|
| `errorMiddleware` overall | 0 | 15 tests traverse it; 8 assert error responses | ~40-50% statements, ~35-40% branches (estimate) | Status/code assertions are precise; message text, envelope shape completeness and logging are largely unasserted |
| AppError branch (`:15-24`) | 0 | 6+ assertions across auth/orders | High for common subclasses (~70%) | Good: 401, 404, 409, 422 all observed; 400 `ValidationError` also observed; `BadRequestError`, `ForbiddenError`, `UnprocessableEntityError` (`INACTIVE_PRODUCT`) not observed |
| `details` present (`:20`) | 0 | `tests/orders.test.ts:128` (nested `unavailable[0].sku`), `tests/auth.test.ts:38` (array) | Covered for object and array shapes | Good field-level assertion for stock; validation asserts only `Array.isArray` |
| `details` omitted (`:20`) | 0 | Implicit in 401 `UNAUTHORIZED` and 404 `NOT_FOUND` assertions | Covered implicitly | No explicit `not.toHaveProperty('details')` assertion |
| ZodError branch (`:26-35`, `:7-12`) | 0 | 0 (unreachable because `validate` wraps first) | 0% for `formatZodIssues` | No evidence; duplicate implementation is untested as well |
| Prisma P2002 branch (`:38-47`) | 0 | 0 | 0% | Not exercised; requires a concurrency race or an unguarded unique write |
| Prisma P2025 branch (`:48-53`) | 0 | 0 | 0% | Not exercised; requires a read-then-write race |
| Fallback log + 500 (`:56-64`) | 0 | 0 | 0% | Highest-risk path has no test evidence; no assertion on `INTERNAL_SERVER_ERROR` or on log emission |
| `requestId` fallback `'unknown'` (`:56`) | 0 | 0 | 0% (request logger always sets `req.id` in integration tests) | Not exercised |
| `meta.target` present/absent ternary (`:39-43`) | 0 | 0 | 0% | Not exercised |

Estimated aggregate: the AppError branch and the conditional-details behavior are covered; the Zod, P2002, P2025 and fallback branches are not. Because these are independent `if` blocks, the untested branches represent roughly half of the executable lines and the majority of branch outcomes.

### 11.3 What the tests verify versus what they do not

Verified through the real pipeline:

- The envelope key `error` and the property `code` for 400, 401, 404, 409 and 422 responses.
- Array-shaped `details` for validation (`tests/auth.test.ts:38`).
- Object-shaped `details` with nested domain payload for stock (`tests/orders.test.ts:128`).
- That `AppError` messages are not replaced by generic text: `EMAIL_ALREADY_USED` and `INVALID_STATUS_TRANSITION` survive to the client.
- That domain codes not present in the generic taxonomy (`EMAIL_ALREADY_USED`, `SKU_ALREADY_USED` by implication, `INVALID_ORDER_STATE_FOR_DELETE`) are preserved.
- That 401 responses are produced for missing credentials and wrong passwords (message text not asserted).

Not verified:

- That the 500 response body is exactly `{ error: { code: 'INTERNAL_SERVER_ERROR', message: 'Internal server error' } }`.
- That the fallback emits exactly one structured log record with `requestId`, `method`, `path` and the error object.
- That stack traces and internal messages are absent from 500 responses.
- That malformed JSON and oversized bodies produce the implemented 500 (rather than an expected 400/413).
- That `P2002` produces 409 with the joined column list, and that a missing `meta.target` produces the generic message.
- That `P2025` produces 404 `NOT_FOUND`.
- That `details` is omitted (not `null`) when undefined.
- That the `X-Request-Id` header correlates with the logged `requestId`.
- Any direct invocation of `errorMiddleware` with synthetic errors, which would be the cheapest way to cover the four untested branches.
- The `formatZodIssues` root-path fallback `(root)`.

### 11.4 Test quality assessment

The integration tests are behavior-oriented and assert the externally visible contract rather than implementation details, which is appropriate for a cross-cutting middleware. The strongest assertions are the nested `details` check for insufficient stock and the domain-code checks, because they prove that the taxonomy survives translation. The weakest area is the absence of negative-path coverage for unexpected failures: the masking rule is the component's principal security control and is entirely unasserted. There is also no test that drives the middleware with a non-`AppError` library error, so the body-parser and Prisma gaps are invisible to the suite. Coverage cannot be measured from repository configuration because no coverage provider, reporter or threshold is configured.

---

## Appendix A: Files Analyzed

| File | Lines | Role in this analysis |
|------|-------|-----------------------|
| `src/middlewares/error.middleware.ts` | 65 | Component under analysis |
| `src/shared/errors/app-error.ts` | 16 | Base error contract (`statusCode`, `errorCode`, `details`) |
| `src/shared/errors/http-errors.ts` | 63 | Nine concrete `AppError` subclasses and their status/code defaults |
| `src/shared/errors/index.ts` | 13 | Barrel used by the component |
| `src/shared/logger/index.ts` | 32 | Logger singleton, redaction, transport selection |
| `src/config/env.ts` | 27 | `LOG_LEVEL` and `NODE_ENV` consumed by the logger |
| `src/middlewares/validate.middleware.ts` | 37 | Upstream `ZodError` conversion; duplicate formatting |
| `src/middlewares/request-logger.middleware.ts` | 28 | `req.id` / `X-Request-Id` producer; access log |
| `src/middlewares/auth.middleware.ts` | 61 | `Unauthorized`/`Forbidden` producers; request type augmentation |
| `src/app.ts` | 76 | Import, 404 fallback and terminal registration of the handler |
| `src/server.ts` | 27 | Process lifecycle context (no direct interaction) |
| `src/routes/index.ts` | 31 | Route aggregation feeding the error path |
| `src/modules/orders/order.controller.ts` | 56 | Representative `try/catch + next(err)` pattern |
| `src/modules/orders/order.service.ts` | 255 | Largest producer of `AppError` variants (404/409/422/400) |
| `src/modules/products/product.service.ts` | 67 | `NotFoundError`, `ConflictError` producers |
| `src/modules/customers/customer.service.ts` | 64 | `NotFoundError`, `ConflictError` producers |
| `src/modules/users/user.service.ts` | 52 | `NotFoundError`, `ConflictError` producers |
| `src/modules/auth/auth.service.ts` | 51 | `UnauthorizedError` producer |
| `src/modules/auth/auth.controller.ts` | 39 | `UnauthorizedError` producer; `next(err)` |
| `src/modules/products/product.controller.ts` | 53 | `next(err)` pattern |
| `src/modules/customers/customer.controller.ts` | 53 | `next(err)` pattern |
| `src/modules/users/user.controller.ts` | 15 | `next(err)` pattern |
| `src/modules/orders/order.routes.ts` | 27 | Guard/validation order for order endpoints |
| `src/modules/auth/auth.routes.ts` | 15 | Public and authenticated route composition |
| `src/shared/http/response.ts` | 24 | Success envelope, for contrast with the error envelope |
| `prisma/schema.prisma` | 138 | Unique constraints (`P2002` sources) and restrictive relations (`P2003` source) |
| `tests/auth.test.ts` | 84 | Integration coverage of 400/401/409 paths |
| `tests/orders.test.ts` | 219 | Integration coverage of 404/409/422 paths |
| `tests/helpers/factories.ts` | 83 | Real app construction used by tests |
| `tests/setup.ts` | 20 | Database lifecycle for tests |
| `vitest.config.ts` | 19 | Test inclusion, setup and execution model |
| `package.json` | 54 | Versions and absence of coverage tooling |
| `tsconfig.json` | 30 | Strictness settings affecting the component (`strict`, `noUncheckedIndexedAccess`) |
| `.eslintrc.json` | 39 | `argsIgnorePattern: "^_"` explaining `_next` |
| `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md` | n/a | Context on coupling, risks and the component's role |
| `docs/FDD.md`, `docs/PRD.md`, `docs/RFC.md`, `docs/TRACKER.md` | 3 each | Checked for an error-handling specification; contain placeholders only |

No files under `node_modules`, `.git`, `dist` or `coverage` were read.

## Appendix B: Assumptions, Ambiguities, and Confidence

| Topic | Statement | Confidence |
|-------|-----------|------------|
| Component boundary | The component is the single file `src/middlewares/error.middleware.ts`; collaborators are analyzed only for contract context | High |
| AppError branch reachability | Every controller catches and forwards errors with `next(err)`; no controller swallows errors silently | High (all five controllers inspected) |
| ZodError branch reachability | The branch is not reached for routes that use `validate`, because `validate` wraps the error in `ValidationError` first | High |
| Body-parser classification | Malformed JSON and oversized bodies are not `AppError`/`ZodError`/Prisma errors and therefore fall to the generic 500 | High (by code structure; not executed) |
| Prisma `P2002` target shape | The code assumes `meta.target` is `string[]`, consistent with Prisma 5.22 on MySQL; other connectors may differ | Medium |
| Prisma `P2003` scenario | Deleting a referenced customer/product raises `P2003` under MySQL's default restrict behavior; this is inferred from the schema, not observed | Medium |
| `headersSent` behavior | The handler writes a response unconditionally; the post-response failure scenario is inferred from Express semantics, not from a test | Medium |
| Coverage estimate | Percentages are static estimates from branch mapping; no coverage report or instrumentation exists in the repository | Low/Medium |
| Test counts | Direct count: 7 tests in `tests/auth.test.ts`, 8 in `tests/orders.test.ts`. The architectural report mentions 10 order tests; the file contains 8 `it` blocks at the analyzed revision | High (direct count) |
| Error-code inventory | Codes listed are those found by static search; no runtime registry exists, so completeness depends on the search | Medium |
| Logger redaction of the `err` object | Redaction paths in `src/shared/logger/index.ts:4-11` are configured for credential-shaped keys; whether Pino applies them to the serialized `err` subtree was not executed or verified | Low |
| Absence of an error-handling specification | `docs/FDD.md`, `docs/RFC.md`, `docs/PRD.md` and `docs/TRACKER.md` contain only placeholders, so business rules were extracted from code only | High |

---

*End of report. This analysis did not modify any project file; the only file created is this report.*
