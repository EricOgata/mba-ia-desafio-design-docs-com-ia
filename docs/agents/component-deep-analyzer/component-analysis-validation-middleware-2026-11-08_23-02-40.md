# Component Deep Analysis Report

**Component:** Validation Middleware
**Primary file:** `src/middlewares/validate.middleware.ts`
**Project:** `mba-ia-desafio-design-docs-com-ia` (Order Management System REST API)
**Analysis date:** 2026-11-08 23:02:40
**Scope:** Entire project root, excluding `node_modules`, `.git`, `dist`, `coverage`
**Analysis mode:** Static code analysis only. No files were modified, no code was executed, no tests were run.

---

## 1. Executive Summary

The Validation Middleware is a 37-line, schema-driven request validation component that sits between Express routing/authentication and the module controllers. It is implemented as a higher-order middleware factory, `validate(schemas)`, which accepts an optional map of Zod schemas for the three HTTP request sources (`body`, `query`, `params`), parses each configured source, mutates the Express request with the parsed result, and converts Zod failures into the application's canonical `ValidationError` contract (`400` / `VALIDATION_ERROR` / `details[]`).

Its role in the system is architectural rather than domain-specific. The middleware contains no business rules of its own about customers, products, orders, or authentication; instead it is the enforcement mechanism for the declarative Zod schemas defined in each module. In that sense it is the "policy enforcement point" of the application's schema-first boundary: every validated route delegates its input contract to Zod, and this component guarantees that controllers and services only receive normalized, type-correct, unknown-key-stripped input. It is used by all five feature modules (`auth`, `users`, `customers`, `products`, `orders`) across 18 of the 19 registered HTTP endpoints.

Key findings:

- **High fan-in, small surface.** Five route modules import the component; it is applied 18 times at router construction time. It is one of the most widely depended-upon units in the codebase, yet its entire runtime surface is a single exported function.
- **Normalization, not just rejection.** The middleware is a "parse, don't validate" implementation: successful parses replace `req.body` and `req.params` and merge into `req.query`, applying Zod defaults, coercions, transforms, and unknown-key stripping. This materially changes the data seen by controllers.
- **Asymmetric mutation semantics.** `body` and `params` are replaced wholesale (unknown keys removed), while `query` is merged into the pre-existing query object (unknown query parameters survive). This inconsistency is the single most important behavioral subtlety of the component.
- **Error translation is the only domain coupling.** The component imports exactly one internal symbol, `ValidationError`, and translates `ZodError.issues` into `{ path, message }` entries with `'(root)'` as the fallback path for root-level issues.
- **No dedicated unit tests exist.** The component is exercised only indirectly through HTTP integration tests. The single explicit negative test (`tests/auth.test.ts:30-39`) covers the `body` source only; `query`, `params`, defaults, coercion, stripping, and non-Zod error passthrough are untested directly.
- **Runtime contract, no compile-time contract.** The middleware returns a generic Express `RequestHandler`; the parsed shapes are not propagated to controllers through Express typings. Controllers still assert `req.query as unknown as ListOrdersQuery` (e.g., `src/modules/orders/order.controller.ts:11`), so schema/controller type linkage is manual and unchecked.

Component boundary used in this analysis: the middleware file itself, plus the Zod schema contracts it enforces (because the component's observable business behavior is defined by those schemas) and its immediate collaborators (`src/shared/errors`, `src/middlewares/error.middleware.ts`, and the route composition in the five module routers).

---

## 2. Data Flow Analysis

### 2.1 Successful request (validated route)

```
1. HTTP request enters Express (src/app.ts:59-60)
   - express.json({ limit: '1mb' }) parses JSON bodies into req.body
   - requestLogger assigns req.id and X-Request-Id (src/middlewares/request-logger.middleware.ts:5-27)
2. Router mounts module router under /api/v1 (src/routes/index.ts:21-31)
3. [Protected routes only] authenticate verifies the Bearer JWT and hydrates req.user
   (src/middlewares/auth.middleware.ts:27-47); requireRole may run next (users route)
4. validate(schemas) executes (src/middlewares/validate.middleware.ts:11-37)
   4a. If schemas.body exists:  req.body  = schema.parse(req.body)        (line 15)
   4b. If schemas.query exists: parsedQuery = schema.parse(req.query);
                                Object.assign(req.query, parsedQuery)       (lines 18-19)
   4c. If schemas.params exists: req.params = schema.parse(req.params)     (line 22)
   4d. next() -> controller                                               (line 24)
5. Controller reads the already-normalized req.body / req.query / req.params
   (e.g., src/modules/orders/order.controller.ts:11,31,41)
6. Controller calls the module service; service applies domain rules
   (e.g., discount <= subtotal in src/modules/orders/order.service.ts:87-91)
7. Repository/Prisma performs persistence
8. Response is serialized by the controller (res.status(...).json(...))
```

### 2.2 Failure path (Zod validation error)

```
1. schema.parse(...) throws ZodError (first failing source only)
2. catch block detects err instanceof ZodError (src/middlewares/validate.middleware.ts:26)
3. err.issues mapped to details[]:
   { path: issue.path.join('.') || '(root)', message: issue.message }     (lines 27-30)
4. next(new ValidationError('Validation failed', details))                (line 31)
   - ValidationError sets statusCode 400 and errorCode 'VALIDATION_ERROR'
     (src/shared/errors/http-errors.ts:9-13)
5. Express skips remaining route handlers and enters errorMiddleware
   (src/middlewares/error.middleware.ts:14-24)
6. Response body:
   { "error": { "code": "VALIDATION_ERROR",
                "message": "Validation failed",
                "details": [ { "path": "...", "message": "..." } ] } }
7. No controller, service, or repository code executes for the rejected request
```

### 2.3 Failure path (non-Zod error)

```
1. A schema transform or any other statement throws a non-ZodError
2. catch block falls through to next(err) (line 34)
3. errorMiddleware maps it: AppError -> its status; ZodError -> 400;
   Prisma P2002 -> 409; P2025 -> 404; anything else -> logged + 500 masked
   (src/middlewares/error.middleware.ts:15-64)
```

### 2.4 Mutation timing on partial failure

The three `parse` calls are sequential. If `body` parses successfully but `query` or `params` then fails, `req.body` has already been replaced before the request is rejected. Because the request terminates in `errorMiddleware`, the mutated body is not consumed downstream, but the handler is not transactional with respect to request state. This is an important implementation detail for any future middleware inserted after validation and before the controller.

### 2.5 Data transformation summary

| Source | Operation | Defaults applied | Unknown keys | Coercion/transform | Persisted to |
|--------|-----------|------------------|--------------|--------------------|--------------|
| `body` | `parse` + assignment | Yes | Removed (Zod strip) | Yes (e.g., none currently except defaults) | New object in `req.body` |
| `query` | `parse` + `Object.assign` | Yes | Retained from original `req.query` | Yes (`z.coerce`, `.transform`) | Merged into existing `req.query` |
| `params` | `parse` + assignment | Yes | Removed (Zod strip) | Yes | New object in `req.params` |

---

## 3. Business Rules & Logic

The middleware itself implements mechanical/structural rules (rules BR-01 to BR-08 and BR-15 to BR-18). Because it is schema-agnostic, the concrete input contracts it enforces are declared in module schemas; those are documented as delegated rules (BR-09 to BR-14) because they are part of the component's observable behavior. Confidence is high unless explicitly marked otherwise.

### Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location |
|---------|-----------|------------------|----------|
| BR-01 | Structural | Only sources with a supplied schema are validated; absent sources are left untouched | `src/middlewares/validate.middleware.ts:14-23` |
| BR-02 | Ordering / fail-fast | Sources are validated in the fixed order body -> query -> params; the first Zod failure aborts the request | `src/middlewares/validate.middleware.ts:14-24` |
| BR-03 | Normalization | A valid body replaces `req.body` with the parsed output (unknown keys stripped, defaults/transforms applied) | `src/middlewares/validate.middleware.ts:15` |
| BR-04 | Normalization | A valid query is merged into the existing `req.query`; parsed values override, unknown query keys survive | `src/middlewares/validate.middleware.ts:18-19` |
| BR-05 | Normalization | Valid path params replace `req.params` with the parsed output | `src/middlewares/validate.middleware.ts:22` |
| BR-06 | Error contract | Zod failures become `ValidationError` (HTTP 400, code `VALIDATION_ERROR`) with per-issue `{path, message}` details; root path is `(root)` | `src/middlewares/validate.middleware.ts:26-32` |
| BR-07 | Error passthrough | Non-Zod errors are forwarded unchanged to the centralized error middleware | `src/middlewares/validate.middleware.ts:34` |
| BR-08 | Control transfer | On success, control passes to the next handler via `next()` without writing a response | `src/middlewares/validate.middleware.ts:24` |
| BR-09 | Delegated domain validation | Registration and login payload constraints (email, password, name, role) | `src/modules/auth/auth.schemas.ts:3-13` |
| BR-10 | Delegated domain validation | Customer payload and nested address constraints; update accepts a partial payload | `src/modules/customers/customer.schemas.ts:3-29` |
| BR-11 | Delegated domain validation | Product payload constraints; update accepts a partial payload; `active` query accepts only the strings `true`/`false` | `src/modules/products/product.schemas.ts:3-24` |
| BR-12 | Delegated domain validation | Order creation constraints (customer, items, quantity, discount, notes) | `src/modules/orders/order.schemas.ts:4-16` |
| BR-13 | Delegated domain validation | Order status transition payload constraints (`toStatus`, `reason`) | `src/modules/orders/order.schemas.ts:18-21` |
| BR-14 | Delegated domain validation | Pagination and filter constraints (page, pageSize, search, status, customerId, from/to) | `src/modules/orders/order.schemas.ts:23-30`, `src/modules/customers/customer.schemas.ts:25-29`, `src/modules/products/product.schemas.ts:16-24` |
| BR-15 | Identifier validation | Every `:id` path parameter is validated as a UUID string using Zod's permissive UUID regex | `src/modules/orders/order.schemas.ts:4`, `src/modules/customers/customer.schemas.ts:3-5`, `src/modules/products/product.schemas.ts:3`, `src/modules/users/user.routes.ts:7` |
| BR-16 | Defaulting / PATCH semantics | Create schemas apply defaults; `.partial()` update schemas make fields optional and suppress those defaults when omitted | `src/modules/products/product.schemas.ts:10-14`, `src/modules/customers/customer.schemas.ts:15-23` |
| BR-17 | Pipeline ordering | On protected routes validation runs after authentication (and after role authorization on the users route) | `src/modules/orders/order.routes.ts:14-24`, `src/modules/customers/customer.routes.ts:14-24`, `src/modules/products/product.routes.ts:14-24`, `src/modules/users/user.routes.ts:12-18` |
| BR-18 | Boundary rule | The middleware validates shape only; cross-field and state-dependent rules remain in services | `src/middlewares/validate.middleware.ts:11-37` vs `src/modules/orders/order.service.ts:51-91` |

### Detailed breakdown of the business rules

---

### Business Rule BR-01: Source-scoped, optional validation

**Overview**:
Validation is applied independently to `body`, `query`, and `params`, and each source is optional. A source that has no schema is not inspected at all.

**Detailed description**:
The `Sources` type (`src/middlewares/validate.middleware.ts:5-9`) declares three optional members, each typed `ZodTypeAny`. At request time, the handler checks each member with a truthiness test (`if (schemas.body)`, `if (schemas.query)`, `if (schemas.params)`). This design lets a single route validate only the parts of the request that carry input for that operation. For example, `GET /api/v1/customers` validates only `query`, `GET /api/v1/customers/:id` validates only `params`, and `POST /api/v1/customers` validates only `body`.

The optionality is intentional and is used throughout the route layer: 13 of the 18 `validate(...)` call sites pass exactly one source, and only the two `PATCH` routes in `customers`, `products`, and `orders` (three routes total) pass two sources. No route currently passes all three. Because absent sources are never parsed, the middleware does not normalize or protect request parts the route author did not opt into; validation coverage is therefore a per-route design decision rather than a global guarantee.

The rule also means the middleware cannot protect a handler from unvalidated input in a source it was not configured for. A route registered without a `body` schema will pass `req.body` to its controller exactly as produced by `express.json()`, including unknown keys. This is a deliberate separation of concerns but creates a coverage obligation on route authors.

**Rule workflow**:
1. Receive the `schemas` object captured in the factory closure.
2. Test `schemas.body`; parse and assign only if defined.
3. Test `schemas.query`; parse and merge only if defined.
4. Test `schemas.params`; parse and assign only if defined.
5. Call `next()`.

---

### Business Rule BR-02: Fixed validation order with fail-fast semantics

**Overview**:
When multiple sources are configured, validation executes body first, then query, then params, and stops at the first schema that throws.

**Detailed description**:
The sequence is hard-coded at `src/middlewares/validate.middleware.ts:14-23`. The first `parse` that throws transfers control to the catch block, so later sources are never parsed and their issues are never collected. The consequence is that error responses describe only one source at a time. A request with an invalid body and an invalid path parameter receives only the body issues; the client must correct that source before learning about the next.

The fixed order creates a deterministic precedence: body errors outrank query errors, and query errors outrank params errors. This behavior is observable in the API contract even though it is not documented. Because body is parsed first, a successful body parse followed by a query failure leaves `req.body` already replaced (see Section 2.4). No compensating action is taken.

Only one Zod schema is parsed per source, so `parse` returns the first source's complete issue list as produced by Zod (Zod itself accumulates all issues within a single object schema). There is no cross-source aggregation and no ability to configure a different order.

**Rule workflow**:
1. Parse body (if configured). Failure -> jump to error translation.
2. Parse query (if configured). Failure -> jump to error translation.
3. Parse params (if configured). Failure -> jump to error translation.
4. Only if all configured sources pass -> `next()`.

---

### Business Rule BR-03: Body replacement and unknown-key stripping

**Overview**:
A valid body is replaced by the value returned from `schemas.body.parse(req.body)`. The parsed value is a new object containing only keys declared in the schema (plus defaults), with all transforms applied.

**Detailed description**:
Line 15 assigns the parse result directly back to `req.body`, typed as `z.infer<NonNullable<S['body']>>`. Because every schema in the project is a plain `z.object(...)` without `.passthrough()` or `.strict()`, Zod 3.23.8 strips unknown keys by default. This gives the middleware an implicit mass-assignment control: a client cannot inject fields that the schema does not declare, even if the controller later spreads the body into a persistence call. For example, `POST /api/v1/auth/register` cannot smuggle `passwordHash` or `id` into `req.body`.

Defaults declared in the schema are materialized during this parse. `registerSchema` injects `role: 'OPERATOR'` when omitted (`src/modules/auth/auth.schemas.ts:7`); `createOrderSchema` injects `discountCents: 0` (`src/modules/orders/order.schemas.ts:14`); `createProductSchema` injects `stockQuantity: 0` and `active: true` (`src/modules/products/product.schemas.ts:10-11`). Controllers and services therefore always see fully populated objects for those fields, which is why `OrderService.create` can compare `input.discountCents > subtotalCents` without a null check.

A side effect is that unknown or misspelled body fields are silently discarded rather than rejected. A client sending `discount` instead of `discountCents` receives a `201` with a zero discount, because the typo is stripped before the service runs. Similarly, extra descriptive fields are dropped without warning. The middleware cannot distinguish "intentional extra field" from "typo" because no schema uses `.strict()`.

**Rule workflow**:
1. `schemas.body.parse(req.body)`.
2. Zod validates declared fields, applies defaults/transforms, strips unknown keys.
3. Parsed object is assigned to `req.body`.
4. Controller/service consume the normalized object.

---

### Business Rule BR-04: Query merge semantics (values overridden, unknowns retained)

**Overview**:
A valid query is parsed and then merged into the existing `req.query` object with `Object.assign(req.query, parsedQuery)`. Parsed values overwrite their raw counterparts, but keys not declared in the schema remain on `req.query`.

**Detailed description**:
Unlike body and params, query is never replaced. Lines 18-19 keep the original Express query object (produced by Express 4.21.1's default `qs`-based query parser) and copy the parsed keys onto it. This preserves any query keys the schema does not know about, while coercing and defaulting the declared ones. For example, `GET /api/v1/orders?page=2&pageSize=5&foo=bar` leaves `foo: 'bar'` on `req.query` and writes numeric `page: 2` and `pageSize: 5`.

The merge is what allows the middleware to deliver `page` and `pageSize` as numbers even though HTTP query values are strings. `z.coerce.number()` converts `'2'` to `2`; the assignment then replaces the string on `req.query`. Without this merge, `OrderService.list` would compute `(query.page - 1) * query.pageSize` using string arithmetic (`'2' - 1 = 1`, but `1 * '5' = 5`) and pagination would degrade unpredictably. The merge also applies `.transform` results, notably `active` being converted from the literal string `'true'`/`'false'` to a boolean (`src/modules/products/product.schemas.ts:20-23`).

Because `Object.assign` does not delete keys, the query source is the only source whose unknown input survives validation. This is inconsistent with body and params and means downstream code that iterates `req.query` (none currently does) would observe unvalidated data. It also means duplicate or array-shaped query parameters for undeclared keys are retained in raw form.

**Rule workflow**:
1. `parsedQuery = schemas.query.parse(req.query)`.
2. Zod validates, coerces, applies defaults (`page=1`, `pageSize=20`) and transforms.
3. `Object.assign(req.query, parsedQuery)` overwrites declared keys on the existing object.
4. Undeclared keys remain; controller reads `req.query` as the schema's inferred type.

---

### Business Rule BR-05: Params replacement

**Overview**:
A valid params object replaces `req.params` with the parse result, so only declared path parameters survive and any coercions/transforms apply.

**Detailed description**:
Line 22 assigns `schemas.params.parse(req.params)` back to `req.params`. Express 4 path params are always strings, so this replacement is primarily a shape and format gate: the declared `id` must satisfy `z.string().uuid()`, and the resulting object contains only `id`. Any additional params injected by the router (none currently) would be stripped.

Because the assignment replaces the object, controllers that read `req.params.id!` (for example `src/modules/orders/order.controller.ts:21,41,50`) receive the validated string. There is no coercion in any current param schema, but the mechanism supports it if a future route uses numeric path segments.

The replacement happens only after body and query have been processed, so it is the last normalization step. A params validation failure is reported only if the other configured sources passed.

**Rule workflow**:
1. `schemas.params.parse(req.params)`.
2. Zod validates declared params (currently UUID format) and strips extras.
3. Parsed object is assigned to `req.params`.
4. Controller reads validated param values.

---

### Business Rule BR-06: Zod failure to `ValidationError` contract

**Overview**:
Any `ZodError` is converted into the application's `ValidationError`: HTTP 400, error code `VALIDATION_ERROR`, message `Validation failed`, and an array of `{ path, message }` details.

**Detailed description**:
The catch block at lines 25-33 distinguishes only `ZodError`. It maps `err.issues` to plain objects using `issue.path.join('.')` and falls back to the string `'(root)'` when the path is empty, which happens for object-level issues such as "Required" when the whole body is missing. It then calls `next(new ValidationError('Validation failed', details))` and returns. The `ValidationError` class fixes `statusCode = 400` and `errorCode = 'VALIDATION_ERROR'` (`src/shared/errors/http-errors.ts:9-13`), and `errorMiddleware` serializes it as `{ error: { code, message, details } }` (`src/middlewares/error.middleware.ts:15-24`).

The transformation deliberately hides the raw `ZodError`. Clients do not receive the Zod issue `code`, `expected`, `received`, or stack trace, only the dotted path and the human-readable message. This yields a stable, machine-parseable contract: the error middleware's duplicate Zod branch (`src/middlewares/error.middleware.ts:7-12,26-35`) produces an identical shape, so a Zod error escaping any other code path would still serialize consistently. In the current request pipeline, the middleware's branch is the one that actually runs, because it converts the error before the error middleware sees it.

Zod's default messages are used throughout; the only custom messages in the project are `order must contain at least one item` (`src/modules/orders/order.schemas.ts:13`) and the environment-variable messages. Because Zod enum messages include the received value (for example, an invalid `toStatus`), the details array can echo client-supplied scalar values. The response is JSON, so this is information reflection rather than injection, but it is a data-exposure characteristic worth recording.

**Rule workflow**:
1. Catch an exception from any `parse` call.
2. If `instanceof ZodError`: map every issue to `{ path: issue.path.join('.') || '(root)', message: issue.message }`.
3. Construct `ValidationError('Validation failed', details)` and pass it to `next`.
4. `errorMiddleware` serializes status 400 with code `VALIDATION_ERROR`.
5. Return without invoking the controller.

---

### Business Rule BR-07: Non-Zod error passthrough

**Overview**:
Errors that are not `ZodError` are forwarded unchanged with `next(err)`; the middleware never masks or reclassifies them.

**Detailed description**:
Line 34 is the fallback of the catch block. It exists because `schema.parse` can throw for reasons other than validation: a `.transform` or `.refine` callback can throw an arbitrary error, and Zod itself throws a plain `Error` when an async refinement is encountered during synchronous `parse` (for example, "Async refinement encountered during synchronous parse operation. Use .parseAsync instead."). The middleware does not attempt to classify these cases; they travel to `errorMiddleware`, which maps known `AppError` subclasses and Prisma errors, logs unknown errors with the request id, and returns a generic `500 INTERNAL_SERVER_ERROR`.

This rule is a deliberate boundary: the component owns only validation failures and delegates every other failure to the centralized error policy. It also means that a programming defect inside a schema (for example, a transform that dereferences `undefined`) surfaces as a 500 rather than a 400. That is the correct classification for a server defect, and it keeps the middleware from accidentally reporting infrastructure failures as client errors.

No current schema uses `.refine`, `.transform` with throwing logic, or async refinements, so in practice this branch is only reachable by unexpected runtime errors. It remains part of the component's contract.

**Rule workflow**:
1. Catch a non-`ZodError`.
2. Call `next(err)` without wrapping.
3. `errorMiddleware` classifies: `AppError` -> its status, Prisma known errors -> 409/404, otherwise log + 500.

---

### Business Rule BR-08: Success delegation without response writing

**Overview**:
On success the middleware performs no I/O and produces no response; it only calls `next()`.

**Detailed description**:
Line 24 calls `next()` with no arguments after all configured sources pass. The component never touches `res`, never sets headers, never logs, and never emits events. This keeps it orthogonal to the request logger (which owns `X-Request-Id` and access logging) and to the controllers (which own status codes and response bodies).

The absence of a response also means the middleware cannot short-circuit with a "sanitized" echo of the parsed request; clients receive only what the controller returns. Downstream code is expected to trust the normalized request. This trust is the core architectural contract of the component: once `next()` is called, `req.body`, `req.query`, and `req.params` conform to the schemas supplied at route-registration time.

Because `next()` is invoked inside the `try`, a synchronous exception thrown by a downstream Express layer could in principle be observed by this catch block. In Express 4.21.1 each router layer wraps its handler in its own try/catch, so downstream synchronous throws are normally captured by Express before they propagate back. The practical risk is low, but the structural coupling of control transfer and exception handling is worth noting.

**Rule workflow**:
1. All configured parses succeed.
2. `next()` is invoked.
3. Express continues to the controller; the middleware takes no further action.

---

### Business Rule BR-09: Authentication payload constraints (delegated)

**Overview**:
`POST /api/v1/auth/register` and `POST /api/v1/auth/login` validate their bodies with `registerSchema` and `loginSchema`.

**Detailed description**:
Registration (`src/modules/auth/auth.schemas.ts:3-8`) requires `email` to be a valid email string of at most 255 characters, `password` to be a string of 8 to 72 characters, `name` to be 2 to 150 characters, and `role` to be one of `ADMIN` or `OPERATOR` with a default of `OPERATOR`. The 72-character password ceiling aligns with bcrypt's input truncation limit, preventing the service from hashing only a prefix of a longer secret. Because the schema is a plain object, registration requests cannot inject `passwordHash`, `id`, `createdAt`, or other persistence fields.

Login (`src/modules/auth/auth.schemas.ts:10-13`) requires a valid email of at most 255 characters and a password of 1 to 72 characters. The minimum of 1 (rather than 8) is intentional: login must reject wrong credentials with `401 UNAUTHORIZED`, not reject short passwords with `400 VALIDATION_ERROR`, preserving the distinction between "malformed request" and "authentication failed". A 3-character password submitted to login passes validation and is compared by bcrypt.

The constraints are shape-only. Email uniqueness is enforced later by `AuthService`/`UserService` as a `409` conflict, and password strength beyond length is not enforced anywhere. The `role` field is accepted directly from the public registration endpoint, meaning a caller can self-register as `ADMIN`; this is a domain/security decision implemented outside this middleware and is recorded here only because it is part of the validated contract.

| Field | Schema | Constraint |
|-------|--------|------------|
| `email` | register, login | valid email format, max 255 |
| `password` | register | string, min 8, max 72 |
| `password` | login | string, min 1, max 72 |
| `name` | register | string, min 2, max 150 |
| `role` | register | enum `ADMIN`/`OPERATOR`, default `OPERATOR` |

**Rule workflow**:
1. `POST /auth/register` -> `validate({ body: registerSchema })` -> `AuthController.register`.
2. `POST /auth/login` -> `validate({ body: loginSchema })` -> `AuthController.login`.
3. Invalid payload -> 400 `VALIDATION_ERROR`; valid payload -> controller/service.

---

### Business Rule BR-10: Customer payload and address constraints (delegated)

**Overview**:
Customer create/update bodies and the customer list query are validated by `createCustomerSchema`, `updateCustomerSchema`, and `listCustomersQuerySchema`.

**Detailed description**:
Creation (`src/modules/customers/customer.schemas.ts:15-21`) requires `name` (2-150), `email` (valid, max 255), `phone` (8-32), `document` (11-20), and an `address` object. The nested address schema (`lines 7-13`) requires `street` (1-200), `number` (1-20, as a string so values like `"100A"` are allowed), `city` (1-100), `state` (exactly 2 characters), and `zipCode` (5-15). The two-character state rule is a hard constraint that will reject full state names such as `"São Paulo"` or `"SP "` (the untrimmed trailing space makes length 3).

Update (`line 23`) is `createCustomerSchema.partial()`. Every top-level field becomes optional, so `PATCH` can send any subset. The partialization is shallow: if `address` is supplied it must still be a complete, valid address object, because the nested schema is not partialized. This is an important asymmetry: clients cannot PATCH a single address field; they must resend the full address.

The list query (`lines 25-29`) coerces `page` to an integer >= 1 with default 1, `pageSize` to an integer between 1 and 100 with default 20, and accepts an optional `search` that is trimmed and must be at least 1 character after trimming. An empty `?search=` therefore fails validation, while an absent `search` passes. `document` and `phone` are length-bounded but not format-validated (no CPF/CNPJ or E.164 check), and uniqueness of email is enforced by the service, not the schema.

| Field | Constraint |
|-------|------------|
| `name` | 2-150 chars |
| `email` | valid email, max 255 |
| `phone` | 8-32 chars |
| `document` | 11-20 chars |
| `address.street` | 1-200 chars |
| `address.number` | 1-20 chars (string) |
| `address.city` | 1-100 chars |
| `address.state` | exactly 2 chars |
| `address.zipCode` | 5-15 chars |
| `search` | trimmed, min 1, optional |

**Rule workflow**:
1. `POST /customers` -> `validate({ body: createCustomerSchema })`.
2. `PATCH /customers/:id` -> `validate({ params: customerIdParamSchema, body: updateCustomerSchema })`.
3. `GET /customers` -> `validate({ query: listCustomersQuerySchema })`.
4. `GET|DELETE /customers/:id` -> `validate({ params: customerIdParamSchema })`.

---

### Business Rule BR-11: Product payload constraints (delegated)

**Overview**:
Product create/update bodies and product list queries are validated by `createProductSchema`, `updateProductSchema`, and `listProductsQuerySchema`.

**Detailed description**:
Creation (`src/modules/products/product.schemas.ts:5-12`) requires `sku` (2-64), `name` (2-200), and `priceCents` (integer >= 0), and accepts optional `description` (max 2000). It defaults `stockQuantity` to 0 (integer >= 0) and `active` to `true`. Because `priceCents` and `stockQuantity` use `z.number()` and not `z.coerce.number()`, JSON bodies must send numeric literals; the strings `"1000"` or `"10"` are rejected with 400. This is the opposite of query handling and is a recurring source of client integration errors.

Update (`line 14`) is `createProductSchema.partial()`. All fields become optional, and Zod 3.23.8's optional wrapper short-circuits `undefined` before the inner default runs, so omitted `stockQuantity`/`active` remain `undefined` rather than being reset to 0/true. `ProductService.update` relies on this: it builds the Prisma patch by checking `!== undefined` for each field (`src/modules/products/product.service.ts:53-60`). If defaults were applied on PATCH, every update would silently reset stock and activation state.

The list query (`lines 16-24`) coerces `page`/`pageSize` like the customer schema, accepts an optional trimmed `search`, and accepts `active` only as the exact strings `'true'` or `'false'`, transforming them to booleans. Values such as `active=1`, `active=TRUE`, or a repeated parameter that becomes an array are rejected. The transform only runs when the key is present; omission leaves `undefined`, which the repository treats as "no filter".

| Field | Constraint |
|-------|------------|
| `sku` | 2-64 chars |
| `name` | 2-200 chars |
| `description` | max 2000, optional |
| `priceCents` | integer >= 0 (number, not coerced) |
| `stockQuantity` | integer >= 0, default 0 (create only) |
| `active` | boolean, default true (create only) |
| `active` (query) | literal `'true'`/`'false'` -> boolean |

**Rule workflow**:
1. `POST /products` -> `validate({ body: createProductSchema })`.
2. `PATCH /products/:id` -> params + `updateProductSchema`.
3. `GET /products` -> `validate({ query: listProductsQuerySchema })`.
4. `GET|DELETE /products/:id` -> `validate({ params: productIdParamSchema })`.

---

### Business Rule BR-12: Order creation constraints (delegated)

**Overview**:
`POST /api/v1/orders` validates `customerId`, a non-empty `items` array, `discountCents`, and `notes` with `createOrderSchema`.

**Detailed description**:
`createOrderSchema` (`src/modules/orders/order.schemas.ts:11-16`) requires `customerId` as a UUID and `items` as an array of `{ productId: UUID, quantity: positive integer }` with at least one element (custom message: `order must contain at least one item`). `discountCents` must be a non-negative integer and defaults to 0; `notes` is optional with a maximum of 1000 characters. All numeric fields are `z.number()` (not coerced), so a JSON body must use numbers for `quantity` and `discountCents`.

The schema validates structure, not referential integrity or business consistency. It does not verify that the customer or products exist (services return 404), that products are active (422 `INACTIVE_PRODUCT`), or that `discountCents <= subtotalCents` (400 `VALIDATION_ERROR` raised inside `OrderService.create` at `src/modules/orders/order.service.ts:87-91`). It also permits duplicate `productId` entries in `items`; the service aggregates quantities per product (`order.service.ts:194-202`). `quantity` has no upper bound, so very large integers pass validation and are constrained only by stock checks and database types.

This division is a deliberate layering: the middleware can reject `quantity: 0`, `quantity: 1.5`, `discountCents: -1`, an empty item list, or a malformed UUID without touching the database; anything requiring reads from persistence stays in the service transaction.

| Field | Constraint |
|-------|------------|
| `customerId` | UUID string |
| `items` | array, min 1 |
| `items[].productId` | UUID string |
| `items[].quantity` | integer > 0 (number, not coerced) |
| `discountCents` | integer >= 0, default 0 |
| `notes` | max 1000 chars, optional |

**Rule workflow**:
1. `POST /orders` -> `authenticate` -> `validate({ body: createOrderSchema })`.
2. Valid shape -> `OrderController.create` -> `OrderService.create` (referential and financial checks).
3. Invalid shape -> 400 `VALIDATION_ERROR`; no transaction begins.

---

### Business Rule BR-13: Order status transition payload constraints (delegated)

**Overview**:
`PATCH /api/v1/orders/:id/status` validates the path id as a UUID and the body with `updateOrderStatusSchema`.

**Detailed description**:
`updateOrderStatusSchema` (`src/modules/orders/order.schemas.ts:18-21`) requires `toStatus` to be one of the Prisma `OrderStatus` enum values (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`, defined in `prisma/schema.prisma:16-23`) and accepts an optional `reason` of at most 500 characters. The schema does not know the current status, so it cannot decide whether a transition is legal; that is handled by `canTransition` and `InvalidStatusTransitionError` inside the service (`order.service.ts:147-149`).

The distinction between the two failure modes is part of the API contract: an unknown `toStatus` string produces `400 VALIDATION_ERROR` from this middleware, while a known but illegal transition (for example `PENDING -> SHIPPED`) produces `409 INVALID_STATUS_TRANSITION` from the service. A same-status transition produces `409` with the same code but a different message (`order.service.ts:140-146`). Clients must therefore treat 400 and 409 differently.

Because both `params` and `body` are configured on this route, a malformed id is reported only after a valid body has been parsed; the body is normalized even though the request ultimately fails on params.

| Field | Constraint |
|-------|------------|
| `:id` | UUID string |
| `toStatus` | enum `OrderStatus` |
| `reason` | max 500 chars, optional |

**Rule workflow**:
1. `PATCH /orders/:id/status` -> `authenticate`.
2. `validate({ params: orderIdParamSchema, body: updateOrderStatusSchema })`.
3. Valid -> `OrderController.changeStatus` -> `OrderService.changeStatus` (state machine).

---

### Business Rule BR-14: Pagination and filter constraints (delegated)

**Overview**:
List endpoints enforce a shared pagination contract: `page >= 1`, `1 <= pageSize <= 100`, with defaults of 1 and 20, plus module-specific filters.

**Detailed description**:
All three list schemas (`orders`, `customers`, `products`) declare `page: z.coerce.number().int().min(1).default(1)` and `pageSize: z.coerce.number().int().min(1).max(100).default(20)`. The coercion accepts numeric strings from the query string and converts them; the integer and bounds checks reject `0`, negatives, decimals, `NaN` (for example `page=abc`), and `pageSize=101`. Because `z.coerce.number()` uses JavaScript `Number()`, an empty value (`?page=`) becomes `0` and fails `min(1)`; booleans and arrays also fail the subsequent integer check in practice.

The 100-item ceiling is the application's pagination guard rail; it is enforced before any repository call, so an oversized request cannot force an unbounded `take`. The defaults are applied by the query merge (BR-04), so services always receive numeric `page`/`pageSize` even when the client omits them. `OrderService.list` and the other services compute `skip = (page - 1) * pageSize` directly from these values (`order.service.ts:38-39`).

Module-specific filters are: orders `status` (optional `OrderStatus` enum), `customerId` (optional UUID), `from`/`to` (optional `z.coerce.date()`); customers `search`; products `search` and `active`. `z.coerce.date()` accepts anything `new Date(...)` accepts, so date-only strings are interpreted as UTC midnight and numeric strings may be interpreted as timestamps or years; only wholly invalid input produces an `invalid_date` issue. The schema does not enforce `from <= to`; that cross-field rule is not implemented anywhere in the repository layer either (`order.repository.ts:25-29` simply applies both bounds).

| Parameter | Constraint |
|-----------|------------|
| `page` | coerced integer >= 1, default 1 |
| `pageSize` | coerced integer 1-100, default 20 |
| `status` (orders) | optional `OrderStatus` enum |
| `customerId` (orders) | optional UUID |
| `from`, `to` (orders) | optional coerced Date |
| `search` (customers/products) | trimmed, min 1, optional |
| `active` (products) | literal `'true'`/`'false'` -> boolean |

**Rule workflow**:
1. `GET /orders|customers|products` -> `authenticate`.
2. `validate({ query: <listSchema> })` -> coerce, bound, default, transform.
3. Valid -> controller -> service -> repository with numeric pagination.

---

### Business Rule BR-15: Path identifier UUID validation

**Overview**:
Every `:id` path parameter is validated as a UUID string before the controller runs.

**Detailed description**:
Four schema definitions guard identifiers: `orderIdParamSchema` (`src/modules/orders/order.schemas.ts:4`), `customerIdParamSchema` (`src/modules/customers/customer.schemas.ts:3-5`), `productIdParamSchema` (`src/modules/products/product.schemas.ts:3`), and an inline `idParamSchema` in `src/modules/users/user.routes.ts:7`. All use `z.string().uuid()`. Zod 3.23.8's UUID check uses a permissive regex (`^[0-9a-fA-F]{8}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{4}\b-[0-9a-fA-F]{12}$`), so it accepts any 8-4-4-4-12 hexadecimal layout regardless of RFC 4122 version or variant nibbles. This is why the integration test's `11111111-1111-1111-1111-111111111111` passes validation and reaches the service, yielding 404 rather than 400 (`tests/orders.test.ts:47-57`).

The rule rejects non-hex identifiers, wrong group lengths, missing hyphens, and non-string values before Prisma is invoked. It does not guarantee that the identifier exists; existence is checked by the service and produces `404 NOT_FOUND`. This separation keeps malformed-input errors (400) distinct from missing-resource errors (404).

A minor inconsistency exists in the users module: `userIdParamSchema` is defined and exported in `src/modules/users/user.schemas.ts:3-5` but never imported; the router defines its own identical `idParamSchema` (`user.routes.ts:7`). The validation behavior is the same, but the duplication means a future change to one definition will not affect the other.

**Rule workflow**:
1. Router matches `/:id`.
2. `validate({ params: <idParamSchema> })`.
3. Non-UUID -> 400 `VALIDATION_ERROR` with path `id`.
4. UUID -> controller/service; non-existent id -> 404 `NOT_FOUND`.

---

### Business Rule BR-16: Defaults on create, suppression of defaults on partial updates

**Overview**:
Defaults declared in create schemas are applied by the middleware; when the same schema is used through `.partial()` for PATCH, omitted fields stay `undefined` and the default is not applied.

**Detailed description**:
Defaults are `role: 'OPERATOR'` (auth register), `discountCents: 0` (order create), `stockQuantity: 0` and `active: true` (product create), and `page: 1` / `pageSize: 20` (all list queries). These are materialized during `parse`, so controllers and services can rely on their presence. The behavior is high-confidence and is exercised implicitly by every request that omits those fields, although no test asserts the defaulted values directly.

`updateProductSchema` and `updateCustomerSchema` are produced with `.partial()` (`product.schemas.ts:14`, `customer.schemas.ts:23`). In Zod 3.23.8, `partial()` wraps each property in `ZodOptional`; the optional parser returns `undefined` immediately for an absent key and never descends into the wrapped `ZodDefault`. Consequently `PATCH /products/:id` with `{ "name": "x" }` produces `{ name: 'x', sku: undefined, priceCents: undefined, stockQuantity: undefined, active: undefined, ... }`, not `stockQuantity: 0`. The services depend on this to avoid clobbering stored values (`product.service.ts:53-60`, `customer.service.ts:51-57`).

The same partialization is shallow for nested objects: `updateCustomerSchema` makes `address` optional but, when present, requires all five address fields. There is no deep-partial variant. This is a behavioral rule that clients must follow and that is not documented in the repository's Markdown docs.

**Rule workflow**:
1. Create route: parse with full schema -> defaults filled.
2. Update route: parse with `.partial()` schema -> omitted keys become `undefined`, defaults suppressed.
3. Service constructs a conditional Prisma patch using `!== undefined` checks.

---

### Business Rule BR-17: Validation runs after authentication on protected routes

**Overview**:
On all protected modules, `authenticate` is registered before `validate`, and on the users route `requireRole('ADMIN')` runs between them.

**Detailed description**:
Orders, customers, and products call `router.use(authenticate)` before defining routes (`order.routes.ts:14`, `customer.routes.ts:14`, `product.routes.ts:14`), so unauthenticated requests are rejected with 401 before any schema parse occurs. The users route composes `authenticate`, `requireRole('ADMIN')`, then `validate({ params: idParamSchema })` (`user.routes.ts:12-18`), so a non-admin receives 403 without the id being validated. Auth routes are public except `/auth/me`, which uses only `authenticate` and no validation.

This ordering is a security-positive property: malformed payloads from unauthenticated callers do not reach the validation layer, and authorization failures are not masked by validation errors. It also means that error precedence is authentication -> authorization -> validation -> domain. A caller cannot use the validation error responses as an oracle for endpoint schemas without first presenting a valid token.

The middleware itself has no knowledge of authentication; the ordering is a convention enforced only by code review in each router. Nothing prevents a future route from registering `validate` before `authenticate`.

**Rule workflow**:
1. Protected router applies `authenticate` to all routes.
2. Optional `requireRole` for restricted routes.
3. `validate` executes only for authenticated/authorized requests.
4. Unauthenticated -> 401; forbidden -> 403; invalid input -> 400.

---

### Business Rule BR-18: Shape validation boundary (no cross-field or stateful rules)

**Overview**:
The component validates only per-field shape, type, range, enum membership, and format. Cross-field consistency, referential integrity, and state-dependent rules remain in the service layer.

**Detailed description**:
Rules such as "discount cannot exceed subtotal" (`order.service.ts:87-91`), "products must exist and be active" (`order.service.ts:59-73`), "stock must be sufficient" (`order.service.ts:204-231`), "status transition must be legal" (`order.service.ts:147-149`), and "email/SKU must be unique" (`customer.service.ts:29-33`, `product.service.ts:30-34`) all require database reads and are therefore implemented in services and repositories. The middleware cannot express them because it has no access to persistence.

This boundary has two consequences. First, some requests pass validation (400-level gate) and then fail with 404/409/422 from the domain, so clients must handle multiple error classes. Second, any cross-field rule that does not require persistence (for example `from <= to` for order date filters, or "at least one address field on PATCH") is not implemented at all: Zod's `.refine`/`.superRefine` is unused in every schema in the project. The only custom messages are the `items` minimum and the environment schema.

The component's contract therefore is "structural purity": if `parse` succeeds, the request has the declared types and declared fields only; it does not mean the request is semantically valid. This distinction is the most important thing for consumers to understand about the component.

**Rule workflow**:
1. Middleware gate: shape/type/range/format only.
2. Controller: extracts typed values and delegates.
3. Service: referential, cross-field, and state rules inside transactions.
4. Error taxonomy distinguishes 400 (shape) from 404/409/422 (domain).

---

## 4. Component Structure

```
src/
├── middlewares/
│   ├── validate.middleware.ts       # ANALYZED COMPONENT: Zod request validation factory
│   ├── auth.middleware.ts           # Upstream: JWT authentication + role authorization
│   ├── error.middleware.ts          # Downstream: AppError/Zod/Prisma -> HTTP JSON
│   └── request-logger.middleware.ts # Upstream: X-Request-Id + access logging
├── shared/
│   └── errors/
│       ├── app-error.ts             # Base AppError (statusCode, errorCode, details)
│       ├── http-errors.ts           # ValidationError (400 / VALIDATION_ERROR)
│       └── index.ts                 # Barrel export consumed by the middleware
├── modules/
│   ├── auth/
│   │   ├── auth.routes.ts           # 2 validate() call sites (body)
│   │   └── auth.schemas.ts          # registerSchema, loginSchema
│   ├── users/
│   │   ├── user.routes.ts           # 1 validate() call site (params, inline schema)
│   │   └── user.schemas.ts          # userIdParamSchema (unused duplicate)
│   ├── customers/
│   │   ├── customer.routes.ts       # 5 validate() call sites
│   │   └── customer.schemas.ts      # create/update/id/list schemas
│   ├── products/
│   │   ├── product.routes.ts        # 5 validate() call sites
│   │   └── product.schemas.ts       # create/update/id/list schemas
│   └── orders/
│       ├── order.routes.ts          # 5 validate() call sites
│       └── order.schemas.ts         # create/update-status/id/list schemas
├── routes/
│   └── index.ts                     # Mounts all module routers under /api/v1
└── app.ts                           # Pipeline composition, express.json, error middleware
```

### Internal organization of the analyzed file

| Lines | Element | Purpose |
|-------|---------|---------|
| 1 | `import type { RequestHandler } from 'express'` | Type-only dependency on Express middleware signature |
| 2 | `import { ZodError, type ZodTypeAny, type z } from 'zod'` | Runtime `ZodError` class; type-only schema/inference types |
| 3 | `import { ValidationError } from '../shared/errors/index.js'` | Converts Zod failures into the application error taxonomy |
| 5-9 | `type Sources = { body?, query?, params?: ZodTypeAny }` | Internal configuration contract for the factory (not exported) |
| 11 | `export function validate<S extends Sources>(schemas: S): RequestHandler` | Public API; generic preserves schema shape for inference inside the file |
| 12 | `return (req, _res, next) => {...}` | Express-compatible handler closure over `schemas` |
| 14-16 | Body parse/assign | Normalizes `req.body` |
| 17-20 | Query parse/merge | Normalizes `req.query` without replacing it |
| 21-23 | Params parse/assign | Normalizes `req.params` |
| 24 | `next()` | Delegates to the controller |
| 25-33 | `catch` + `ZodError` mapping | Builds `{path, message}` details and throws `ValidationError` via `next` |
| 34 | `next(err)` | Passes unexpected errors to the centralized error middleware |

The file has no runtime state, no singletons, and no side effects at import time. Each router build creates a small closure per call site; the schema objects are shared references. The entire component is synchronous and stateless, which makes it trivially safe for concurrent requests.

---

## 5. Dependency Analysis

### 5.1 Internal dependency graph

```
Module routers (auth, users, customers, products, orders)
        |
        | import { validate }
        v
src/middlewares/validate.middleware.ts
        |
        +--> src/shared/errors/index.ts  (runtime: ValidationError)
        |          |
        |          +--> src/shared/errors/http-errors.ts (class definition)
        |          +--> src/shared/errors/app-error.ts    (base class)
        |
        +--> src/middlewares/error.middleware.ts  (indirect, via next(err))
                   |
                   +--> src/shared/logger/index.ts
                   +--> @prisma/client (error classes)
                   +--> zod (ZodError fallback branch)

Route modules -> module schemas -> validate(schemas) -> req.* mutation -> controllers
```

Concrete internal edges:

| From | To | Direction | Kind | Evidence |
|------|----|-----------|------|----------|
| `src/modules/auth/auth.routes.ts` | `src/middlewares/validate.middleware.ts` | import | runtime | line 2 |
| `src/modules/users/user.routes.ts` | `src/middlewares/validate.middleware.ts` | import | runtime | line 4 |
| `src/modules/customers/customer.routes.ts` | `src/middlewares/validate.middleware.ts` | import | runtime | line 3 |
| `src/modules/products/product.routes.ts` | `src/middlewares/validate.middleware.ts` | import | runtime | line 3 |
| `src/modules/orders/order.routes.ts` | `src/middlewares/validate.middleware.ts` | import | runtime | line 3 |
| `src/middlewares/validate.middleware.ts` | `src/shared/errors/index.ts` | import | runtime | line 3 |
| `src/middlewares/validate.middleware.ts` | `src/middlewares/error.middleware.ts` | indirect (next) | runtime | line 31, 34 |
| `src/modules/*/*.routes.ts` | `src/modules/*/*.schemas.ts` | import | runtime | route files |
| `src/app.ts` | module routers + `error.middleware` | import | runtime | lines 3-19, 73 |

### 5.2 External dependencies

| Dependency | Version | Usage in component | Type |
|------------|---------|--------------------|------|
| `zod` | 3.23.8 | `ZodError` (runtime), `ZodTypeAny`, `z.infer` (types) | External library (runtime + types) |
| `express` | 4.21.1 | `RequestHandler` signature only | External library (type-only import) |
| `@types/express` | 4.17.21 | Express typings | Dev dependency (types) |

The component does **not** depend on Prisma, the logger, configuration, authentication, or any domain module. Its only runtime external package is Zod. The `express` import is erased at compile time because it is `import type`.

### 5.3 Downstream / upstream dependency chains

```
Upstream (callers)
  buildAuthRouter / buildUserRouter / buildCustomerRouter / buildProductRouter / buildOrderRouter
    -> validate(schemas)
Downstream (callees)
  validate -> schemas.<source>.parse
  validate -> new ValidationError -> errorMiddleware -> HTTP response
```

No circular dependencies were identified. The component sits in a stable layer (`middlewares`) that is imported by `modules` and imports only from `shared`, matching the architectural layering reported in `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md:173` (`src/middlewares` depends on `config`, `shared`).

### 5.4 Configuration dependencies

The component is not configurable through environment variables or feature flags. Its only configuration input is the `schemas` argument supplied at router construction time. Therefore it has no per-environment behavior, no toggles, and no secrets. Its effective behavior does vary by route because each call site selects different schemas and sources.

---

## 6. Afferent and Efferent Coupling

Metrics below are measured at the TypeScript module/symbol level. "Critical" reflects the breadth of impact if the unit changes or fails, not a judgment on code quality.

| Component / Symbol | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Instability (Ce / (Ca+Ce)) | Critical |
|--------------------|------------------------|------------------------|----------------------------|----------|
| `validate` (exported factory) | 5 (five route modules) | 3 (`zod`, `express` types, `shared/errors`) | 0.375 | High |
| Request handler closure (inner function) | 0 (not exported) | 5 (`schemas.body`, `schemas.query`, `schemas.params`, `ZodError`, `ValidationError`) | 1.000 | Medium |
| `Sources` type | 1 (`validate` signature) | 1 (`ZodTypeAny` from zod) | 0.500 | Low |
| `ValidationError` (consumed symbol) | 1 from this component; 6 project-wide | 2 (`AppError`, HTTP status/code constants) | 0.25 project-wide | High project-wide |
| `src/middlewares/validate.middleware.ts` (module) | 5 | 3 | 0.375 | High |

### 6.1 Method-level coupling (Ca / Ce)

| Method | Ca | Ce | Notes |
|--------|----|----|-------|
| `validate(schemas)` | 5 routers | 3 modules | Factory; one closure per route call site |
| `handler(req, _res, next)` | 1 (Express invocation) | 5 collaborators | Orchestrates body/query/params plus error mapping |
| `ZodError` branch → `ValidationError` | n/a | 2 (`ZodError.issues`, `ValidationError`) | Only error-translation logic in the file |

### 6.2 Coupling observations

- **High afferent, low efferent:** the component is depended on broadly but depends on little. This is the desired shape for a cross-cutting middleware and yields a low instability score (0.375), meaning changes are more likely to be driven by its many consumers than by its dependencies.
- **Coupling is type-safe at the boundary, not through it:** `validate` accepts a generic `S extends Sources` and returns a plain `RequestHandler`. The generic is used only inside the file for casts; callers receive no type information that links the schema to `req.body`/`req.query`/`req.params`. Controllers compensate with `as unknown as` casts (`order.controller.ts:11`, `customer.controller.ts:10`, `product.controller.ts:10`).
- **Coupling to the error taxonomy is intentional and minimal:** exactly one internal symbol (`ValidationError`) is imported. This is the seam that lets the component participate in the application-wide error contract without knowing about HTTP response formatting.
- **Schema coupling is inverted:** the component does not import the schemas; routers inject them. This keeps the middleware domain-agnostic and allows all 18 call sites to share one implementation.
- **Architecture-report discrepancy:** the architectural report lists this component as Ca 5 / Ce 1 (`architectural-report-2026-11-08_22-46-20.md:137`). The difference is counting method: the report counts workspace module edges only (`shared/errors`), while the table above also counts npm packages (`zod` runtime, `express` type-only). Both are consistent under their respective definitions.

---

## 7. Endpoints

The component itself does not expose endpoints; it is a request-pipeline middleware. The table below documents the **route surface it guards**, because that is the observable contract it participates in. Nineteen endpoints are registered; eighteen are covered by at least one `validate(...)` call site.

| Endpoint | Method | Validation Applied | Schema(s) | Auth |
|----------|--------|--------------------|-----------|------|
| `/api/v1/auth/register` | POST | body | `registerSchema` | Public |
| `/api/v1/auth/login` | POST | body | `loginSchema` | Public |
| `/api/v1/auth/me` | GET | none | - | Bearer JWT |
| `/api/v1/users/{id}` | GET | params | inline `idParamSchema` | JWT + `ADMIN` |
| `/api/v1/customers` | GET | query | `listCustomersQuerySchema` | Bearer JWT |
| `/api/v1/customers/{id}` | GET | params | `customerIdParamSchema` | Bearer JWT |
| `/api/v1/customers` | POST | body | `createCustomerSchema` | Bearer JWT |
| `/api/v1/customers/{id}` | PATCH | params + body | `customerIdParamSchema`, `updateCustomerSchema` | Bearer JWT |
| `/api/v1/customers/{id}` | DELETE | params | `customerIdParamSchema` | Bearer JWT |
| `/api/v1/products` | GET | query | `listProductsQuerySchema` | Bearer JWT |
| `/api/v1/products/{id}` | GET | params | `productIdParamSchema` | Bearer JWT |
| `/api/v1/products` | POST | body | `createProductSchema` | Bearer JWT |
| `/api/v1/products/{id}` | PATCH | params + body | `productIdParamSchema`, `updateProductSchema` | Bearer JWT |
| `/api/v1/products/{id}` | DELETE | params | `productIdParamSchema` | Bearer JWT |
| `/api/v1/orders` | GET | query | `listOrdersQuerySchema` | Bearer JWT |
| `/api/v1/orders/{id}` | GET | params | `orderIdParamSchema` | Bearer JWT |
| `/api/v1/orders` | POST | body | `createOrderSchema` | Bearer JWT |
| `/api/v1/orders/{id}/status` | PATCH | params + body | `orderIdParamSchema`, `updateOrderStatusSchema` | Bearer JWT |
| `/api/v1/orders/{id}` | DELETE | params | `orderIdParamSchema` | Bearer JWT |

Notes:

- `GET /api/v1/auth/me` and `GET /health` (`src/app.ts:62-64`) are not validated; `/health` is outside the module routers entirely.
- `PATCH` routes in customers, products, and orders are the only call sites with two configured sources. Validation order is body then params on customers/products (`validate({ params, body })` object literal order does not matter; the middleware always parses body first), and body then params on orders.
- `requireRole('ADMIN')` executes before validation only on `GET /users/{id}`.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Express request pipeline | Internal framework middleware | Intercept requests between router/auth and controller; mutate `req` | In-process function calls | Express `Request`/`Response` objects | `next(err)` on failure; `next()` on success |
| Zod schemas (per module) | Internal contract injection | Declarative validation, coercion, defaults, transforms, stripping | In-process function calls | TS objects / Zod schema instances | Throws `ZodError`; caught and translated |
| `ValidationError` / `AppError` taxonomy | Internal shared kernel | Canonical 400 error with machine-readable code and details | In-process class instantiation | `{ path, message }[]` details | Serialized by `errorMiddleware` |
| Centralized error middleware | Internal downstream middleware | Convert `ValidationError` into JSON HTTP response; log unknown errors | In-process function calls | JSON `{ error: { code, message, details } }` | Single terminal error handler |
| `express.json()` body parser | Internal upstream middleware | Populate `req.body` before validation; enforce 1 MB limit | In-process | JSON | Malformed/oversized JSON errors bypass this component (see Section 10) |
| `authenticate` / `requireRole` | Internal upstream middleware | Ensure caller identity/role before validation on protected routes | In-process | JWT claims in `req.user` | 401/403 via `AppError` before validation |
| Prisma enum (`OrderStatus`) | External library type | Source of allowed `toStatus` values via `z.nativeEnum` | Compile-time import | TS enum object | Invalid value -> `ZodError` -> 400 |

No network calls, database access, message broker, cache, or external API is invoked by this component. It has no timeouts, retries, circuit breakers, or fallbacks because it performs only in-process CPU work.

### 8.1 Integration error precedence

For a protected route, the observable precedence is:

1. `authenticate` failure -> `401 UNAUTHORIZED` (no validation performed)
2. `requireRole` failure -> `403 FORBIDDEN` (no validation performed)
3. Body parse failure -> `400 VALIDATION_ERROR` with body issues
4. Query parse failure -> `400 VALIDATION_ERROR` with query issues
5. Params parse failure -> `400 VALIDATION_ERROR` with params issues
6. Controller/service domain failure -> `404` / `409` / `422` or other `AppError`
7. Unexpected failure -> logged 500 with masked message

### 8.2 Contract with controllers

The middleware's output contract is positional, not typed: after `next()`, `req.body` matches the body schema, `req.query` contains the query schema's coerced/defaulted keys, and `req.params` matches the params schema. Controllers consume these with assertions or non-null assertions (`req.params.id!`). Any new controller that assumes a different shape will not be caught by the compiler, only by runtime tests.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Middleware Factory / Higher-Order Function | `validate(schemas)` returns a `RequestHandler` closure over `schemas` | `src/middlewares/validate.middleware.ts:11-37` | Reuse one validation implementation across all routes while binding route-specific schemas |
| Chain of Responsibility | Express middleware chain: logger -> auth -> role -> validate -> controller -> error handler | `src/app.ts:59-73`, module routers | Sequential request processing with `next()` handoff |
| Strategy | Interchangeable Zod schemas supplied per route/source | All `*.routes.ts` call sites | Select validation policy without changing the middleware |
| Adapter / Translator | `ZodError.issues` -> `ValidationError` details array | `src/middlewares/validate.middleware.ts:26-32` | Decouple the validation library's error model from the API error contract |
| Parse, Don't Validate | Successful parse replaces request data with typed/normalized values | lines 15, 18-22 | Guarantee downstream code receives canonical types and defaults |
| Fail-Fast Gate | First schema failure aborts before controller/service | lines 14-24 | Prevent invalid input from reaching domain logic and persistence |
| Dependency Inversion (partial) | Component depends on the `ValidationError` abstraction, not on HTTP serialization | line 3; `src/middlewares/error.middleware.ts` | Keep validation independent of response formatting |
| Default Parameter Object | `Sources` groups optional schema slots | lines 5-9 | Uniform API for 1-3 validation targets |
| Declarative Schema-First Boundary | Zod schemas define contracts beside modules | `src/modules/*/*.schemas.ts` | Single source of truth for request shapes; types inferred from schemas |
| Composition Root | Middleware is wired into routers at construction, controllers injected | `src/app.ts:26-53`, `src/routes/index.ts:21-31` | Centralized, manual dependency wiring |

### 9.1 Architectural decisions evidenced by the code

- **Validation is a cross-cutting middleware, not a service concern.** No service accepts unvalidated HTTP input; controllers pass `req.body` directly (`auth.controller.ts:14`, `order.controller.ts:31`).
- **Runtime validation is mandatory; compile-time typing is advisory.** The generic `S extends Sources` exists but its inferred type is not exported into Express's request typings.
- **The error taxonomy is the integration contract.** The middleware produces an `AppError` subclass rather than an HTTP response, preserving the single-error-handler architecture.
- **Synchronous validation only.** The choice of `parse` (not `parseAsync`) means schemas with async refinements are structurally unsupported.
- **Schema ownership is distributed.** The middleware has no schema registry and no knowledge of which modules exist; route modules own their schemas.

### 9.2 Quality attributes

| Attribute | Assessment | Evidence |
|-----------|------------|----------|
| Cohesion | High (single responsibility: request shape enforcement) | 37 lines, one exported function, no unrelated logic |
| Coupling | Low efferent, high afferent; instability 0.375 | 3 external module dependencies, 5 consumers |
| Reusability | High | 18 call sites, domain-agnostic |
| Testability | High in principle, low in practice | Pure closure with injectable schemas, but no unit tests |
| Performance | Low overhead | Synchronous parse; schemas built once per route |
| Security | Positive input gating, but not an authorization control | Strips unknown body keys, bounds sizes, validates enums/UUIDs; no auth logic |
| Observability | None | No logging, metrics, or tracing in the component; failures visible only via 400 responses |
| Resilience | No external I/O, so no resilience patterns needed | In-process CPU only |
| Maintainability | Good locally; duplicated error formatting and schema duplication elsewhere | `error.middleware.ts:7-12` duplicates path formatting; users id schema duplicated |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Test coverage | No unit tests target `validate.middleware.ts`; `query` and `params` branches are never exercised negatively | Regressions in query merge or params replacement would ship undetected; the only explicit negative test covers `body` |
| Medium | Error contract consistency | Zod issue-to-detail formatting is duplicated in `src/middlewares/error.middleware.ts:7-12` and `src/middlewares/validate.middleware.ts:27-30` | Two implementations can drift; the error middleware's Zod branch is effectively unreachable from validated routes but remains a parallel contract |
| Medium | Type safety | `validate` returns a generic `RequestHandler`; parsed types do not flow to controllers, which use `as unknown as` (`order.controller.ts:11`, `customer.controller.ts:10`, `product.controller.ts:10`) | Schema/controller mismatches are compile-time invisible; contract is enforced only at runtime |
| Medium | Input handling consistency | `req.query` retains unknown keys (merge) while body/params strip them (replace) | Unvalidated data can coexist with validated data in the same request object; behavior differs by source |
| Medium | Silent data loss | All body schemas use Zod's default strip; there is no `.strict()` | Misspelled or unintended fields (for example `discount` instead of `discountCents`) are silently ignored rather than rejected, producing successful responses with unintended defaults |
| Medium | Upstream body parsing | Malformed JSON or oversized bodies are rejected by `express.json()` with non-`AppError`/non-`ZodError` errors, which `errorMiddleware` maps to 500 | Clients can receive `500 INTERNAL_SERVER_ERROR` for a client-side syntax error; the validation component cannot intercept this because it runs after body parsing |
| Low | Error aggregation | Fail-fast across sources reports only the first failing source | Clients must make multiple round trips to discover all errors when body, query, and params are all invalid |
| Low | Request state consistency | Body is mutated before query/params validation completes | A later middleware or handler observing `req.body` after a validation failure sees normalized data; no transactional rollback of request state |
| Low | Async validation | Synchronous `parse` cannot execute async refinements; Zod throws a plain `Error` in that case | Introducing an async refinement would surface as 500, not 400, unless the middleware is changed to `parseAsync` |
| Low | Identifier schema duplication | `userIdParamSchema` (`user.schemas.ts:3-5`) duplicates the inline `idParamSchema` (`user.routes.ts:7`) | Two definitions of the same rule can diverge; one is dead code today |
| Low | Error detail reflection | Zod enum messages include the received value in `details[].message` | Client-supplied scalar values are echoed back in JSON error responses; low impact for a JSON API but relevant to data-handling policy |
| Low | Date coercion permissiveness | `z.coerce.date()` accepts anything `new Date()` accepts (numeric strings, date-only strings interpreted as UTC) | Ambiguous filter boundaries; no `from <= to` cross-field check exists |
| Low | Numeric coercion edge cases | `z.coerce.number()` converts empty/whitespace query values to `0` before bounds checks | Some malformed inputs are rejected by `min(1)` rather than by a type error; messages are less precise |
| Low | Query nesting | Express `qs` can produce nested/array values for undeclared query keys, which survive the merge | Downstream consumers that iterate `req.query` would see unvalidated structures (none do today) |
| Low | No validation observability | The component emits no metrics/logs for rejection reasons | Validation failure rates and schema hotspots cannot be measured from application telemetry |
| Low | Configuration coupling | Effective validation depends entirely on route authors calling `validate` | Any new route without the middleware bypasses all schema enforcement silently |

No SQL injection, path traversal, or deserialization gadget risk was identified in this component: it performs no string concatenation, file access, or dynamic code evaluation. SQL safety depends on Prisma parameterization downstream, not on this middleware. String contents are not sanitized (no HTML escaping), which is appropriate for a JSON-only API but should not be mistaken for output encoding.

---

## 11. Test Coverage Analysis

### 11.1 Test infrastructure

| Item | Location | Notes |
|------|----------|-------|
| Test runner config | `vitest.config.ts` | `tests/**/*.test.ts`, node environment, 30s timeouts, single fork, no parallelism |
| Global setup | `tests/setup.ts` | Connects Prisma; truncates all tables before each test; disconnects after all |
| Factories | `tests/helpers/factories.ts` | Builds the real app with the real Prisma client; user/customer/product factories; JWT bootstrap |
| Integration style | `tests/auth.test.ts`, `tests/orders.test.ts` | Supertest against `buildApp({ prisma })`; requires a running MySQL database (docker-compose) |

There are exactly two test files and no unit-test harness for isolated middleware/schema tests. Tests require a live database, so they are integration tests by construction. Static analysis was performed without executing the suite (the workspace has no installed `node_modules`, and the suite requires MySQL), so coverage numbers below are estimates from control-flow inspection, not measured instrumentation.

### 11.2 Coverage by component / contract

| Component | Unit Tests | Integration Tests | Estimated Coverage | Test Quality |
|-----------|------------|-------------------|--------------------|--------------|
| `validate.middleware.ts` | 0 | Indirect via all validated routes; 1 explicit negative assertion | Lines ~85-95% (happy paths exercised constantly); branches ~30-40% (query/params/non-Zod branches largely unexercised) | Happy path strong by volume, but no dedicated suite; no assertions on normalization, defaults, stripping, or mutation semantics |
| Error translation contract (`ZodError` -> `ValidationError`) | 0 | 1 test (`tests/auth.test.ts:30-39`) | ~40% (code, status, details array asserted; path contents and `(root)` fallback not asserted) | Good basic assertions; does not verify `details[].path`/`message` values |
| `auth.schemas.ts` | 0 | 1 negative test + several positive | ~35% | Covers invalid email, short password, empty name; no role enum, max lengths, default role, or login-schema negatives |
| `customer.schemas.ts` | 0 | 0 direct | ~0% | No customer endpoint tests exist at all; factories create customers directly through Prisma, bypassing HTTP validation |
| `product.schemas.ts` | 0 | 0 direct | ~0% | No product endpoint tests; factories bypass HTTP validation |
| `order.schemas.ts` | 0 | Several positive; no schema negatives | ~45% | Valid create/status/list payloads exercised; no test for empty items, bad UUID, negative discount, invalid enum, quantity 0/float, notes length |
| `user.routes.ts` params | 0 | 0 | ~0% | No users route test |

### 11.3 Test-by-test mapping to middleware behavior

| Test | File:Line | What it proves about this component |
|------|-----------|------------------------------------|
| Rejects register with invalid payload | `tests/auth.test.ts:30-39` | `body` parse failure -> 400, code `VALIDATION_ERROR`, `details` is an array |
| Registers a new user | `tests/auth.test.ts:6-17` | Valid body passes validation and is forwarded; response excludes hash |
| Rejects duplicated email | `tests/auth.test.ts:19-28` | Valid body reaches service (validation does not block), service emits 409 |
| Logs in successfully / wrong password | `tests/auth.test.ts:41-63` | Login schema accepts valid payload; wrong credentials reach service -> 401 |
| Creates an order | `tests/orders.test.ts:12-40` | `createOrderSchema` accepts UUIDs, positive quantities, optional discount; defaults applied (discount explicitly sent) |
| Rejects non-existent product | `tests/orders.test.ts:42-57` | UUID body values pass validation (permissive UUID regex) and reach the service -> 404 |
| Status transition PENDING -> PAID | `tests/orders.test.ts:59-87` | `updateOrderStatusSchema` + `orderIdParamSchema` accept valid input |
| Invalid transition PENDING -> SHIPPED | `tests/orders.test.ts:89-107` | Enum-valid `SHIPPED` passes validation; domain rejects with 409 |
| Insufficient stock | `tests/orders.test.ts:109-132` | Valid payload reaches transactional stock check -> 422 |
| Cancel and replenish | `tests/orders.test.ts:134-161` | Status body accepted; domain behavior verified |
| List filtered by status | `tests/orders.test.ts:163-195` | `listOrdersQuerySchema` accepts `status=PAID` and defaults page/pageSize; repository receives numeric pagination |
| Refuse delete non-deletable order | `tests/orders.test.ts:197-218` | Params UUID passes; domain 409 |

### 11.4 Uncovered behavior (directly attributable to this component)

- `query` source: no test sends invalid `page`, `pageSize`, `active`, `from`, or `to`, and no test asserts coercion or defaults on the response/pagination object.
- `params` source: no test sends a non-UUID id and asserts `400` with `path: 'id'`.
- Unknown-field stripping: no test sends extra body fields and asserts they are absent downstream.
- Defaults: no test omits `role`, `discountCents`, `stockQuantity`, or `active` and asserts the defaulted value.
- PATCH partial semantics: no test asserts that omitting `active`/`stockQuantity` preserves stored values (the domain path is untested for products/customers entirely).
- Merge semantics: no test proves that unknown query keys survive or that declared keys are overwritten.
- Non-Zod passthrough: no test triggers a non-Zod error from a schema.
- Root-path detail: no test sends a missing/`undefined` body and asserts `path: '(root)'`.
- Ordering: no test sends both an invalid body and invalid params to verify only body details are returned.
- Multiple configured sources: the `PATCH` routes that pass two schemas are entirely untested.

### 11.5 Test quality assessment

The existing tests are behaviorally strong for domain flows but weak as tests of this component. They use the real application and real database, so they validate the integration contract (status codes, error codes, normalized values) rather than the middleware in isolation. Assertions are specific (`toBe(400)`, `toBe('VALIDATION_ERROR')`, `Array.isArray(details)`), which is good, but they do not inspect the contents of `details` or the request mutations. There are no mocks or stubs for Zod because the real schemas are used. Failure diagnostics would be reasonable, since supertest responses are asserted directly.

The most significant gap is that the component's distinguishing features (cross-source sequential validation, query merging, body stripping, partial-default suppression) have no direct assertions. A regression that replaced `Object.assign` with assignment, removed stripping, or changed source order would likely pass the current suite.

---

## Appendix A: Files Analyzed

| File | Relevance |
|------|-----------|
| `src/middlewares/validate.middleware.ts` | Primary component (all 37 lines) |
| `src/middlewares/error.middleware.ts` | Downstream error serialization, duplicate Zod formatting |
| `src/middlewares/auth.middleware.ts` | Upstream middleware ordering and error types |
| `src/middlewares/request-logger.middleware.ts` | Upstream pipeline context (`req.id`) |
| `src/shared/errors/app-error.ts`, `http-errors.ts`, `index.ts` | `ValidationError` definition and error taxonomy |
| `src/app.ts` | Pipeline composition, `express.json`, error handler mounting |
| `src/routes/index.ts` | Router registry and mount points |
| `src/modules/auth/auth.routes.ts`, `auth.schemas.ts`, `auth.controller.ts` | Validation call sites and auth contracts |
| `src/modules/users/user.routes.ts`, `user.schemas.ts`, `user.controller.ts` | Inline params schema and unused duplicate |
| `src/modules/customers/customer.routes.ts`, `customer.schemas.ts`, `customer.controller.ts`, `customer.service.ts` | Customer contracts and downstream usage |
| `src/modules/products/product.routes.ts`, `product.schemas.ts`, `product.controller.ts`, `product.service.ts` | Product contracts and downstream usage |
| `src/modules/orders/order.routes.ts`, `order.schemas.ts`, `order.controller.ts`, `order.service.ts`, `order.repository.ts` | Order contracts, domain rules after validation |
| `prisma/schema.prisma` | `OrderStatus` enum source |
| `src/config/env.ts` | Separate Zod usage (safeParse), not part of request validation |
| `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `vitest.config.ts` | Test coverage evidence |
| `package.json` | Dependency versions (`zod` 3.23.8, `express` 4.21.1) |
| `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md` | Architecture context and coupling cross-check |

## Appendix B: Assumptions, Ambiguities, and Confidence

| Item | Statement | Confidence |
|------|-----------|------------|
| Component boundary | The middleware's business behavior is defined by the schemas it enforces, so schema constraints are documented as delegated rules | High |
| Zod stripping | Zod 3.23.8 `z.object` strips unknown keys by default | High (library behavior) |
| UUID permissiveness | Zod 3.23.8 uses a permissive hex-layout UUID regex without RFC version/variant enforcement, verified against the package source | High |
| Partial defaults | `.partial()` wraps fields in `ZodOptional`, so omitted fields bypass defaults in PATCH schemas | High (Zod 3.23.8 semantics; consistent with service code) |
| Query merge persistence | Express 4.21.1 stores `req.query` as a normal property set by its query middleware (not a recomputing getter), so `Object.assign` mutations persist | High |
| Coverage percentages | Figures are static estimates; the suite was not executed and no instrumentation was run | Low (estimate) |
| Test pass status | The existing tests were not executed; analysis assumes they reflect current behavior | Medium |
| Async refinements | No async refinements exist today; the limitation is inferred from synchronous `parse` | High |
| Malformed JSON -> 500 | Inferred from `errorMiddleware` handling only `AppError`/`ZodError`/Prisma errors and `express.json` throwing body-parser errors; not executed | Medium |
| Duplicate Zod formatting | `errorMiddleware`'s Zod branch is treated as a fallback because this middleware converts `ZodError` first; no other request-path Zod parse exists | High |

No documentation in `docs/FDD.md`, `docs/PRD.md`, or `docs/RFC.md` covers this component (those files are placeholders), so all business rules above were extracted from source code only.
