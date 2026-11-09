# Component Deep Analysis Report: Error Taxonomy

| Attribute | Value |
|-----------|-------|
| Component | Error Taxonomy |
| Primary location | `src/shared/errors/` |
| Files analyzed | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts` |
| Component size | 92 lines of TypeScript across 3 files |
| Analysis scope | Entire project root, excluding `node_modules`, `.git`, `dist`, `coverage` |
| Analysis date | 2026-11-08 |
| Report type | Read-only component-level deep analysis |

---

## 1. Executive Summary

The Error Taxonomy is the shared-kernel exception model of the Order Management API. It defines one abstract base class, `AppError`, and nine concrete subclasses that pair a semantic failure category with an HTTP status code, a stable machine-readable error code, a human-readable message, and an optional structured `details` payload. The component has no endpoints, no persistence, no external service calls, and no third-party runtime dependencies. Its entire responsibility is to give every layer of the application a common vocabulary for expected failures so that a single Express error handler can translate those failures into a uniform JSON response.

Architecturally, the component is small but disproportionately important. It has the highest afferent coupling in the codebase: 11 source files outside the component import the barrel `src/shared/errors/index.ts` (`src/app.ts:20`, `src/middlewares/error.middleware.ts:4`, `src/middlewares/auth.middleware.ts:4`, `src/middlewares/validate.middleware.ts:3`, `src/modules/auth/auth.controller.ts:4`, `src/modules/auth/auth.service.ts:5`, `src/modules/orders/order.controller.ts:4`, `src/modules/orders/order.service.ts:9-16`, `src/modules/customers/customer.service.ts:3`, `src/modules/products/product.service.ts:3`, `src/modules/users/user.service.ts:4`). Its efferent coupling is zero: it imports nothing from the project and nothing from npm. That combination makes it the most stable contract in the system and the primary seam between domain logic and HTTP representation.

The extracted business rules define a clear failure classification policy: 400 for structural or schema-invalid input, 401 for absent or invalid authentication, 403 for authenticated but unauthorized access, 404 for missing resources, 409 for state and uniqueness conflicts, and 422 for syntactically valid requests that violate a domain rule. Two specialized subclasses refine the conflict and unprocessable categories for order status transitions and inventory shortages. Error metadata fields are declared `readonly`, and stack traces are captured at construction time.

The main findings are positive but not risk-free. The hierarchy is cohesive, dependency-free, and consistently used by services and middleware. However, all instances report `name = 'AppError'` rather than their concrete subclass name, which reduces observability. `BadRequestError` is exported but never used. Error codes have no central registry, no uniqueness guarantee, and at least one code (`INVALID_STATUS_TRANSITION`) is produced by two different classes. The taxonomy also has no representation for transport-level failures such as malformed JSON or oversized payloads, so those client errors become generic 500 responses. There are no direct unit tests for any taxonomy class; coverage is entirely indirect through HTTP integration tests in `tests/auth.test.ts` and `tests/orders.test.ts`, leaving `BadRequestError`, `ForbiddenError`, `INACTIVE_PRODUCT`, the route-fallback 404, default-message branches, and the `details` omission branch unverified.

---

## 2. Data Flow Analysis

### 2.1 Domain-originated failure path

```
1. A service or middleware detects an expected failure
   (for example OrderService.create finds an inactive product)
2. The failure is expressed as a taxonomy subclass constructor call
   (new UnprocessableEntityError('One or more products are inactive', 'INACTIVE_PRODUCT', ...))
3. The subclass invokes super(...) and AppError assigns
   name = 'AppError', statusCode, errorCode, details
4. AppError captures the stack trace with Error.captureStackTrace
5. The error is thrown up the synchronous or awaited call stack
6. The controller catch block calls next(err)
   (all controllers follow this pattern, e.g. src/modules/orders/order.controller.ts:33-35)
7. Express skips normal middleware and enters errorMiddleware
   (registered last in src/app.ts:73)
8. errorMiddleware evaluates branches in order:
   AppError -> ZodError -> Prisma P2002 -> Prisma P2025 -> unknown
9. For AppError, the handler reads err.statusCode, err.errorCode,
   err.message and optional err.details
10. The handler writes { error: { code, message, details? } }
    with details included only when it is not undefined
    (src/middlewares/error.middleware.ts:15-24)
11. The request logger finish hook emits an access log containing
    method, path, statusCode and duration
    (src/middlewares/request-logger.middleware.ts:12-25)
```

### 2.2 Middleware-originated failure path

```
1. Request enters a route
2. authenticate rejects missing/invalid Authorization -> UnauthorizedError (401 UNAUTHORIZED)
3. requireRole rejects insufficient role           -> ForbiddenError (403 FORBIDDEN)
   OR requireRole rejects missing req.user         -> UnauthorizedError (401 UNAUTHORIZED)
4. validate converts ZodError                      -> ValidationError (400 VALIDATION_ERROR)
   with details = [{ path, message }, ...]
5. next(err) transfers control to errorMiddleware
6. The same AppError serialization branch runs
```

### 2.3 Route fallback failure path

```
1. No API route matches the request
2. The terminal middleware in src/app.ts:69-71 constructs
   new NotFoundError(`Route ${req.method} ${req.originalUrl}`)
3. NotFoundError composes the message `${resource} not found`
4. errorMiddleware serializes status 404 and code NOT_FOUND
5. Response body message becomes
   "Route GET /some/path not found"
```

### 2.4 Unhandled/unmapped failure path

```
1. An error is not an AppError, not a ZodError, and not Prisma P2002/P2025
   (examples: body-parser SyntaxError for malformed JSON,
    payload-too-large error, Prisma P2003, P2034, P2024)
2. errorMiddleware extracts requestId from req.id or "unknown"
3. logger.error logs the raw error, requestId, method and path
4. The client receives 500 { error: { code: 'INTERNAL_SERVER_ERROR',
   message: 'Internal server error' } }
5. No stack, cause or internal detail is exposed to the client
```

### 2.5 Branch precedence in the error handler

| Order | Condition | Outcome |
|-------|-----------|---------|
| 1 | `err instanceof AppError` | Use `err.statusCode`, `err.errorCode`, `err.message`, optional `err.details` |
| 2 | `err instanceof ZodError` | 400 `VALIDATION_ERROR`, details normalized to `{path,message}[]` |
| 3 | `Prisma.PrismaClientKnownRequestError` with code `P2002` | 409 `CONFLICT`, message includes joined constraint target when present |
| 4 | `Prisma.PrismaClientKnownRequestError` with code `P2025` | 404 `NOT_FOUND`, fixed message `Resource not found` |
| 5 | Anything else | Logged as unhandled and returned as 500 `INTERNAL_SERVER_ERROR` |

Because branches 2 through 4 bypass the taxonomy and write literal code strings, the taxonomy is not the only producer of error codes in the system. `VALIDATION_ERROR`, `CONFLICT`, `NOT_FOUND`, and `INTERNAL_SERVER_ERROR` can all appear in responses without any taxonomy instance being constructed.

---

## 3. Business Rules & Logic

### 3.1 Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location | Confidence |
|---------|-----------|------------------|----------|------------|
| BR-01 | Contract / structural | Expected domain failures must be represented by `AppError` subclasses, never by ad hoc `Error` objects | `src/shared/errors/app-error.ts:3-16`; usage across services and middleware | Explicit |
| BR-02 | Contract / HTTP mapping | Every `AppError` carries a mandatory, immutable HTTP status code fixed by its subclass | `src/shared/errors/app-error.ts:4,8,11`; each subclass `super(...)` call | Explicit |
| BR-03 | Contract / HTTP mapping | Every `AppError` carries a mandatory, immutable machine-readable error code | `src/shared/errors/app-error.ts:5,8,12`; defaults and overrides in `http-errors.ts` | Explicit |
| BR-04 | Contract / data shape | A `details` payload is optional and may be either an object or an array | `src/shared/errors/app-error.ts:1,6,13` | Explicit |
| BR-05 | Presentation | Every error class supplies or requires a human-readable message; defaults exist for most classes | `src/shared/errors/http-errors.ts:4,10,16,22,28-29,34,40` | Explicit |
| BR-06 | Classification | Malformed or schema-invalid client input maps to the 400 family (`BadRequestError`, `ValidationError`) | `src/shared/errors/http-errors.ts:3-13` | Explicit |
| BR-07 | Classification / security | Missing or invalid authentication maps to 401 `UNAUTHORIZED`; authenticated but insufficient role maps to 403 `FORBIDDEN` | `src/shared/errors/http-errors.ts:15-25`; `src/middlewares/auth.middleware.ts:27-60` | Explicit |
| BR-08 | Classification / presentation | Missing resources map to 404 `NOT_FOUND` with a message interpolated from a resource label | `src/shared/errors/http-errors.ts:27-31` | Explicit |
| BR-09 | Classification | Uniqueness and state conflicts map to 409 `CONFLICT`, with an optional domain-specific code override | `src/shared/errors/http-errors.ts:33-37`; services | Explicit |
| BR-10 | Classification | Syntactically valid requests that violate a domain rule map to 422 `UNPROCESSABLE_ENTITY` | `src/shared/errors/http-errors.ts:39-43`; `src/modules/orders/order.service.ts:68-72` | Explicit |
| BR-11 | Domain specialization | An illegal order status transition maps to 409 `INVALID_STATUS_TRANSITION` with `{ from, to }` details | `src/shared/errors/http-errors.ts:45-53`; `src/modules/orders/order.service.ts:147-149` | Explicit |
| BR-12 | Domain specialization | Insufficient product stock maps to 422 `INSUFFICIENT_STOCK` with an `unavailable[]` details array | `src/shared/errors/http-errors.ts:55-62`; `src/modules/orders/order.service.ts:204-224` | Explicit |
| BR-13 | Inheritance | Specialized error subclasses inherit the HTTP status and default code behavior of their parent class | `src/shared/errors/http-errors.ts:45,55` | Explicit |
| BR-14 | Object invariants | `statusCode`, `errorCode` and `details` are read-only after construction; stack trace is captured at construction | `src/shared/errors/app-error.ts:4-14` | Explicit |
| BR-15 | Architecture | Domain code throws taxonomy errors and never formats HTTP error responses; translation belongs to one centralized middleware | Controllers and services vs `src/middlewares/error.middleware.ts:14-24` | Explicit |
| BR-16 | Serialization | Error responses use the envelope `{ error: { code, message, details? } }`; `details` is omitted when `undefined` | `src/middlewares/error.middleware.ts:15-24` | Explicit |
| BR-17 | Routing fallback | Any unmatched route is converted into a taxonomy `NotFoundError` with method and URL embedded in the message | `src/app.ts:69-71` | Explicit |
| BR-18 | Public contract | Error code strings are stable identifiers; only some classes permit overriding the default code | `src/shared/errors/http-errors.ts:3-43`; all error-code literals | Implicit / observed |

### 3.2 Detailed breakdown of the business rules

---

### Business Rule: BR-01 - AppError is the single base contract for expected domain failures

**Overview**:
All operational or expected failures in the application are represented by instances of `AppError` or one of its nine concrete subclasses. Ad hoc `Error` objects are reserved for programming defects, unmapped infrastructure errors, and test harness failures, which fall through to a generic 500 response.

**Detailed description**:
`AppError` is declared at `src/shared/errors/app-error.ts:3-16` as a subclass of the built-in `Error`. It adds three fields on top of the standard error contract: `statusCode: number`, `errorCode: string`, and `details: ErrorDetails`. The nine concrete subclasses are declared in `src/shared/errors/http-errors.ts`: `BadRequestError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InvalidStatusTransitionError`, and `InsufficientStockError`. Every one of them ultimately calls the `AppError` constructor at line 8, so the base class is the single initialization point for all taxonomy state.

The rule is observable in the source tree rather than merely documented. A search for `throw new Error` across `src/` returns no matches; the only plain `Error` construction in the project is in the test helper at `tests/helpers/factories.ts:37`, which throws when login fails during fixture setup and is not part of application behavior. Service classes throw taxonomy instances directly: `OrderService` imports six subclasses at `src/modules/orders/order.service.ts:9-16`, while `UserService`, `CustomerService`, and `ProductService` import `ConflictError` and `NotFoundError`. Controllers and middlewares also construct instances for authorization and fallback cases.

The practical effect is that `errorMiddleware` can classify every expected failure with one check, `err instanceof AppError`, at `src/middlewares/error.middleware.ts:15`. The rule also defines a boundary: it does not claim that all errors in the process are taxonomy errors. Body-parser failures, unhandled Prisma codes, and unexpected runtime exceptions intentionally bypass the hierarchy and are logged then masked as 500. Compliance by future code is conventional rather than enforced; there is no lint rule, type-level constraint, or runtime guard that prevents a developer from throwing a raw `Error` from a service.

**Rule workflow**:
```
1. Detect an expected failure condition in a service, controller or middleware
2. Select the taxonomy subclass whose category matches the condition
3. Construct the subclass with message, optional code override and optional details
4. Throw the instance
5. Controller or middleware forwards it with next(err)
6. errorMiddleware recognizes it via instanceof AppError
7. The instance is serialized with its own status code and error code
```

---

### Business Rule: BR-02 - Every taxonomy error carries a mandatory immutable HTTP status code

**Overview**:
Each `AppError` instance must carry a numeric HTTP status code. The subclass constructor fixes that status, making the mapping between semantic category and HTTP status explicit and impossible to change after construction.

**Detailed description**:
The base constructor signature is `constructor(message: string, statusCode: number, errorCode: string, details?: ErrorDetails)` at `src/shared/errors/app-error.ts:8`. The value is stored in a `public readonly statusCode: number` field at line 4 and assigned at line 11. Concrete subclasses supply the value through their `super(...)` calls: 400 for `BadRequestError` and `ValidationError`, 401 for `UnauthorizedError`, 403 for `ForbiddenError`, 404 for `NotFoundError`, 409 for `ConflictError`, and 422 for `UnprocessableEntityError` (`src/shared/errors/http-errors.ts:3-43`). There is deliberately no taxonomy class for 500; server faults are not considered expected domain outcomes.

The status code is independent of the error code. A class can therefore combine a standard HTTP status with a domain-specific machine code, as `ConflictError` does when called with `EMAIL_ALREADY_USED` or `INVALID_ORDER_STATE_FOR_DELETE`. The middleware never derives the HTTP status from the code string; it reads `err.statusCode` directly at `src/middlewares/error.middleware.ts:16`. This keeps the taxonomy deterministic at runtime and removes mapping logic from the HTTP layer for expected failures.

There is no runtime validation that `statusCode` falls within 400-599 or that it is an integer. The base class is technically public API and can be constructed directly with any number, although no source file does so. The rule is therefore a design convention backed by constructor signatures and subclass defaults, not a guarded invariant. The choice of 422 for domain-state violations is the most semantically significant mapping because it separates "the request is well formed but the domain refuses it" from "the request is malformed" (400).

**Rule workflow**:
```
1. Caller selects an error class whose semantic category matches the failure
2. The subclass calls super(message, <fixed status>, code, details)
3. AppError stores the status in a readonly field
4. errorMiddleware calls res.status(err.statusCode)
5. The HTTP status line reflects the subclass, not the error code
```

---

### Business Rule: BR-03 - Every taxonomy error carries a mandatory machine-readable error code

**Overview**:
Each instance must expose a stable string code that identifies the failure category for programmatic clients. The code is independent of the HTTP status and of the human-readable message.

**Detailed description**:
`errorCode` is declared `public readonly errorCode: string` at `src/shared/errors/app-error.ts:5` and assigned from the constructor at line 12. Every concrete class supplies a default code: `BAD_REQUEST` (`http-errors.ts:4`), `VALIDATION_ERROR` (line 11), `UNAUTHORIZED` (line 17), `FORBIDDEN` (line 23), `NOT_FOUND` (line 29), `CONFLICT` (line 34), `UNPROCESSABLE_ENTITY` (line 40), `INVALID_STATUS_TRANSITION` (line 49), and `INSUFFICIENT_STOCK` (line 59). Three classes allow the caller to override the default: `BadRequestError`, `ConflictError`, and `UnprocessableEntityError`.

Domain services exploit the override to publish more precise codes while retaining the parent HTTP status. `UserService` and `CustomerService` use `EMAIL_ALREADY_USED` (`src/modules/users/user.service.ts:24`, `src/modules/customers/customer.service.ts:32,48`), `ProductService` uses `SKU_ALREADY_USED` (`src/modules/products/product.service.ts:33,50`), and `OrderService` uses `INACTIVE_PRODUCT` (`src/modules/orders/order.service.ts:68-72`), `INVALID_STATUS_TRANSITION` via `ConflictError` (`order.service.ts:141-146`), and `INVALID_ORDER_STATE_FOR_DELETE` (`order.service.ts:185-189`). These strings are part of the external API contract: integration tests assert them directly at `tests/auth.test.ts:27,37,62` and `tests/orders.test.ts:56,106,127,217`.

The code space is not centrally declared. There is no enum, union type, constant map, or registry that enumerates valid codes, and no check prevents two classes from producing the same code. In practice `INVALID_STATUS_TRANSITION` is produced both by `InvalidStatusTransitionError` and by a `ConflictError` constructed with that override. Codes follow an uppercase snake-case convention, but the convention is not enforced by types or linting, so a typo becomes a de facto public contract as soon as it ships.

**Rule workflow**:
```
1. Constructor parameter code is provided, or the class default is used
2. AppError stores the code in a readonly field
3. errorMiddleware copies err.errorCode into response.error.code
4. Clients branch on the code rather than parsing the message
5. Changing or mistyping a code is a breaking API change
```

---

### Business Rule: BR-04 - The details payload is optional and polymorphic

**Overview**:
Errors may expose additional structured context through an optional `details` field. The field can hold either a key-value object or an array, and it is passed through to the response without transformation.

**Detailed description**:
`ErrorDetails` is defined as `Record<string, unknown> | unknown[] | undefined` at `src/shared/errors/app-error.ts:1`. The base constructor accepts `details?: ErrorDetails` at line 8, stores it in `public readonly details: ErrorDetails` at line 6, and assigns it at line 13. Four classes expose a `details` parameter to callers: `BadRequestError` (`http-errors.ts:4`), `ValidationError` (line 10), `ConflictError` (line 34), and `UnprocessableEntityError` (line 40). `UnauthorizedError`, `ForbiddenError`, and `NotFoundError` do not accept details at all, which constrains what those failure categories can communicate.

The two specialized subclasses supply fixed shapes. `InvalidStatusTransitionError` always provides `{ from, to }` (`http-errors.ts:47-51`), and `InsufficientStockError` always provides `{ unavailable }` where each array element has `{ sku, requested, available }` (`http-errors.ts:56-61`). Validation failures use an array of `{ path, message }` objects produced in `src/middlewares/validate.middleware.ts:27-31` and, as a safety net, in `src/middlewares/error.middleware.ts:7-12`. The order service uses array details for the discount rule at `src/modules/orders/order.service.ts:88-90`.

There is no schema, type refinement, deep clone, or runtime validation applied to `details`. It is stored by reference and emitted by reference. Consumers therefore depend on shapes that are documented only by usage: the tests assert `Array.isArray(res.body.error.details)` at `tests/auth.test.ts:38` and `res.body.error.details.unavailable[0].sku` at `tests/orders.test.ts:128`. Because the value is `unknown`, a caller could place a circular object or a non-serializable value into `details`, and the failure would surface during response serialization rather than at construction time.

**Rule workflow**:
```
1. Caller optionally passes an object or array as the fourth constructor argument
2. AppError stores the reference without copying or validation
3. errorMiddleware checks err.details !== undefined
4. If defined, the value is included verbatim under response.error.details
5. If undefined, the key is omitted entirely
```

---

### Business Rule: BR-05 - Every error class supplies or requires a human-readable message

**Overview**:
Every taxonomy instance must carry a non-empty human-readable message. Most classes define a default, while `ConflictError` and `UnprocessableEntityError` require the caller to provide one, and `NotFoundError` composes one from a resource label.

**Detailed description**:
Default messages are declared directly in constructor parameters: `'Bad request'` (`http-errors.ts:4`), `'Validation failed'` (line 10), `'Unauthorized'` (line 16), `'Forbidden'` (line 22), and `'Validation failed'` internally for the Zod branch of the middleware. `NotFoundError` takes a resource label defaulting to `'Resource'` and composes `` `${resource} not found` `` at line 29. `ConflictError` and `UnprocessableEntityError` declare `message: string` without a default at lines 34 and 40, which forces the domain to describe the conflict or semantic violation. Both specialized subclasses override the message with fixed domain text: `Invalid status transition from ${from} to ${to}` and `One or more products do not have enough stock`.

Messages are intended for humans and are not treated as stable identifiers. The tests assert codes rather than messages in most cases, and no test asserts the default message strings. `NotFoundError` is the exception where the message is structurally generated, and its output depends on the caller-supplied label: `OrderService` passes `'Order'` (`src/modules/orders/order.service.ts:46,136,183`), `UserService` passes `'User'` (`src/modules/users/user.service.ts:38`), and the route fallback passes a full request description (`src/app.ts:70`). The resulting messages are useful for logs and debugging but are not localized and have no punctuation policy.

Because `AppError` extends `Error`, the message is also what appears in stack traces and in Pino error serialization when an error is logged. The taxonomy does not redact or transform messages, so any caller that interpolates user-controlled input into a message is responsible for the content. The route fallback does interpolate `req.originalUrl` into the message, but the response is JSON-encoded and no HTML rendering occurs in this API.

**Rule workflow**:
```
1. Construct an error class
2. If the class defines a default and no message is passed, the default is used
3. If the class requires a message, the caller must supply one at compile time
4. NotFoundError ignores caller messages and composes one from the resource label
5. errorMiddleware copies the final message into response.error.message
```

---

### Business Rule: BR-06 - The 400 family covers malformed requests and schema validation failures

**Overview**:
Client input failures are represented by two classes: `BadRequestError` for a generic bad request and `ValidationError` for schema validation failures. Both map to HTTP 400, but only `ValidationError` fixes the code to `VALIDATION_ERROR` and standardizes the details array.

**Detailed description**:
`BadRequestError` (`http-errors.ts:3-7`) uses status 400, default message `'Bad request'`, default code `BAD_REQUEST`, and allows both the code and details to be overridden. `ValidationError` (`http-errors.ts:9-13`) uses status 400, default message `'Validation failed'`, a fixed code `VALIDATION_ERROR`, and accepts details. The distinction is semantic: `BadRequestError` is a general-purpose 400, while `ValidationError` commits to a specific machine-readable meaning and to a details structure that clients can parse.

`ValidationError` is produced by the validation middleware at `src/middlewares/validate.middleware.ts:31` after catching a `ZodError`, and by `OrderService.create` for the discount rule at `src/modules/orders/order.service.ts:87-91`. The latter passes an array with one element `{ path: 'discountCents', message: 'Discount exceeds subtotal' }`, matching the shape produced for Zod issues. The central error middleware also has a direct `ZodError` branch at `src/middlewares/error.middleware.ts:26-35` that writes the same code and shape without constructing a `ValidationError`; this is a defensive path if a Zod error escapes the validation middleware.

`BadRequestError` is exported from the barrel but is never constructed anywhere in `src/` or `tests/`. It represents an available but unused extension point. The overlap between the two classes means a future developer has two plausible choices for a 400, and the taxonomy does not encode which one is correct beyond the naming and the fixed code on `ValidationError`. The integration tests verify the `VALIDATION_ERROR` code and the array details at `tests/auth.test.ts:30-39`, but no test exercises `BadRequestError`.

**Rule workflow**:
```
1. Request body, query or params fail schema parsing
2. validate middleware catches ZodError
3. Issues are normalized to [{ path, message }, ...]
4. new ValidationError('Validation failed', details) is forwarded
5. errorMiddleware returns 400 { error: { code: VALIDATION_ERROR, message, details } }
6. A generic 400 would instead use BadRequestError with an optional custom code
```

---

### Business Rule: BR-07 - Authentication failures map to 401 and authorization failures map to 403

**Overview**:
The taxonomy enforces a strict separation between unauthenticated requests and authenticated requests that lack permission. `UnauthorizedError` always yields 401 `UNAUTHORIZED`; `ForbiddenError` always yields 403 `FORBIDDEN`.

**Detailed description**:
`UnauthorizedError` (`http-errors.ts:15-19`) takes only an optional message and fixes status 401 and code `UNAUTHORIZED`. `ForbiddenError` (`http-errors.ts:21-25`) takes only an optional message and fixes status 403 and code `FORBIDDEN`. Neither class accepts details, so neither can leak token contents, role lists, or user identifiers through the structured payload.

The authentication middleware applies the rule in four places. A missing or non-Bearer `Authorization` header produces `UnauthorizedError('Missing or invalid Authorization header')` (`src/middlewares/auth.middleware.ts:29-32`). An empty bearer token produces `UnauthorizedError('Missing bearer token')` (lines 34-38). A JWT verification failure produces `UnauthorizedError('Invalid or expired token')` (lines 40-46). `requireRole` produces `UnauthorizedError()` when `req.user` is absent (lines 50-54) and `ForbiddenError('Insufficient permissions')` when the user's role is not in the allowed list (lines 55-58). `AuthService.login` also uses `UnauthorizedError('Invalid credentials')` for both unknown email and wrong password (`src/modules/auth/auth.service.ts:34,38`).

The consistent use of the same 401 code and message for unknown-email and wrong-password cases avoids account enumeration. The 401 versus 403 split lets clients distinguish "re-authenticate" from "you are known but not allowed." The `requireRole` path is wired only for the admin-only user listing route (`src/modules/users/user.routes.ts:12-16`), and no current integration test exercises a 403 response, so the `ForbiddenError` branch is defined but unverified by tests.

**Rule workflow**:
```
1. authenticate reads the Authorization header
2. Missing header / empty token / invalid JWT -> UnauthorizedError -> 401 UNAUTHORIZED
3. Valid JWT -> req.user is populated and next() is called
4. requireRole checks req.user and role membership
5. Missing req.user -> UnauthorizedError -> 401 UNAUTHORIZED
6. Role not allowed -> ForbiddenError -> 403 FORBIDDEN
```

---

### Business Rule: BR-08 - Not-found errors interpolate a resource label and expose no custom message

**Overview**:
Missing resources use `NotFoundError`, which always yields 404 `NOT_FOUND` and always composes its message from a caller-supplied resource label. The class deliberately exposes no message or details parameter.

**Detailed description**:
`NotFoundError` is declared at `src/shared/errors/http-errors.ts:27-31` with a single constructor parameter `resource = 'Resource'`. It calls `super(`${resource} not found`, 404, 'NOT_FOUND')`. The default produces `Resource not found`, and domain callers pass singular nouns such as `Order`, `Customer`, `Product`, or `User`. This gives a consistent message shape across modules and prevents each service from inventing a different 404 wording.

The constructor signature is the rule: callers cannot pass a custom message and cannot attach details. That restriction keeps 404 responses small and avoids leaking identifiers or database state, but it also means the response cannot say which identifier was missing or which lookup failed. The route fallback at `src/app.ts:69-71` passes a synthesized label, `Route ${req.method} ${req.originalUrl}`, so an unmatched request produces a message like `Route GET /api/v1/unknown not found`. This is grammatically awkward but still classified correctly as 404 `NOT_FOUND`.

The order service uses this class for four distinct lookups: order retrieval (`order.service.ts:46`), missing customer during creation (line 60), missing product set during creation (line 64), and order lookup during status change (line 136) and deletion (line 183). The tests assert 404 `NOT_FOUND` only for the missing-product scenario at `tests/orders.test.ts:42-57`. The route fallback, missing customer, missing order, and missing user paths are not directly asserted for their 404 body shape.

**Rule workflow**:
```
1. A lookup returns null or an expected set of products is incomplete
2. Caller throws new NotFoundError('<ResourceLabel>')
3. The constructor composes '<ResourceLabel> not found'
4. errorMiddleware returns 404 { error: { code: NOT_FOUND, message } }
5. No details key is present because the class does not accept details
```

---

### Business Rule: BR-09 - Conflicts use 409 with an optional domain-specific code override

**Overview**:
State conflicts, uniqueness violations, and illegal operations on the current state map to HTTP 409. `ConflictError` requires a descriptive message and permits the caller to replace the default `CONFLICT` code with a domain-specific code.

**Detailed description**:
`ConflictError` is declared at `src/shared/errors/http-errors.ts:33-37` with `constructor(message: string, code = 'CONFLICT', details?: ErrorDetails)`. Because `message` has no default, every conflict must be described. Because `code` has a default but can be overridden, the class supports both generic conflicts and precise domain codes without creating a subclass for every case. `details` is optional and is emitted verbatim.

Four domain codes are produced through this class in the current codebase. `EMAIL_ALREADY_USED` is used by both user registration (`src/modules/users/user.service.ts:24`) and customer creation/update (`src/modules/customers/customer.service.ts:32,48`). `SKU_ALREADY_USED` is used by product creation and update (`src/modules/products/product.service.ts:33,50`). `INVALID_ORDER_STATE_FOR_DELETE` is used when an order is not `PENDING` or `CANCELLED` (`src/modules/orders/order.service.ts:185-189`) with details `{ status }`. `INVALID_STATUS_TRANSITION` is used for a same-status transition (`order.service.ts:141-146`) with details `{ from, to }`, duplicating the specialized class described in BR-11.

The 409 category is also produced outside the taxonomy by the Prisma mapping: a `P2002` unique constraint error becomes a 409 with code `CONFLICT` and a message derived from the constraint target (`src/middlewares/error.middleware.ts:37-47`). That branch does not construct a `ConflictError`, so two responses with status 409 and code `CONFLICT` can have different provenance and slightly different message conventions. The taxonomy rule itself does not distinguish optimistic-locking conflicts, duplicate keys, or invalid state; it groups them under one status and relies on the machine code for precision.

**Rule workflow**:
```
1. Detect a uniqueness violation, illegal state or duplicate operation
2. Throw new ConflictError('<message>', '<DOMAIN_CODE>', details?)
3. If no code is supplied, the default CONFLICT is used
4. errorMiddleware returns 409 with the message, code and optional details
5. Prisma P2002 can independently produce a 409 CONFLICT without using this class
```

---

### Business Rule: BR-10 - Domain-rule violations use 422 rather than 400

**Overview**:
A request that is syntactically valid but violates a business rule maps to HTTP 422 `UNPROCESSABLE_ENTITY`. The taxonomy provides a general `UnprocessableEntityError` and a specialized inventory subclass.

**Detailed description**:
`UnprocessableEntityError` is declared at `src/shared/errors/http-errors.ts:39-43` with `constructor(message: string, code = 'UNPROCESSABLE_ENTITY', details?: ErrorDetails)`. Like `ConflictError`, it requires a message and permits a domain code override. The only direct use in the current codebase is the inactive-product check in `OrderService.create` at `src/modules/orders/order.service.ts:66-73`, which throws with message `'One or more products are inactive'`, code `INACTIVE_PRODUCT`, and details `{ skus: [...] }`.

The 422 choice is meaningful: the request passed Zod validation, the referenced entities exist, but the domain refuses to proceed because a product is inactive. This is different from a missing field (400 `VALIDATION_ERROR`) and from a uniqueness or state conflict (409). The taxonomy does not enforce this distinction algorithmically; it relies on the developer selecting the class that matches the semantics. The same status is reused for insufficient stock through inheritance, so 422 covers both "inactive product" and "not enough inventory" while the machine codes separate them.

There is no direct test for the `INACTIVE_PRODUCT` path. The `UnprocessableEntityError` constructor is executed indirectly whenever `InsufficientStockError` calls `super(...)`, so the lines are reached, but the default code branch and the inactive-product behavior are not asserted. The `details` shape for inactive products (`{ skus }`) also differs from the inventory details shape (`{ unavailable }`), which is permitted because `ErrorDetails` is untyped.

**Rule workflow**:
```
1. Request passes schema validation and referenced entities exist
2. A domain invariant fails (inactive product, unavailable stock)
3. Throw UnprocessableEntityError or a subclass with a domain code and details
4. errorMiddleware returns 422 with code and optional details
5. The client distinguishes the exact cause by the code, not by the status
```

---

### Business Rule: BR-11 - Illegal order status transitions use a specialized 409 error

**Overview**:
An order status change that violates the state machine is represented by `InvalidStatusTransitionError`, a subclass of `ConflictError` that fixes the code to `INVALID_STATUS_TRANSITION` and supplies `{ from, to }` details.

**Detailed description**:
`InvalidStatusTransitionError` is declared at `src/shared/errors/http-errors.ts:45-53` and extends `ConflictError`. Its constructor takes only `from` and `to`, composes the message `Invalid status transition from ${from} to ${to}`, fixes the code to `INVALID_STATUS_TRANSITION`, and passes details `{ from, to }`. Because it extends `ConflictError`, it inherits HTTP 409 and is recognized by `instanceof ConflictError` as well as `instanceof AppError`.

`OrderService.changeStatus` uses it at `src/modules/orders/order.service.ts:147-149` after `canTransition(from, to)` returns false. The state machine itself lives in `src/modules/orders/order.status.ts:3-14` and is the authority on which transitions are legal. A separate branch at `order.service.ts:140-146` handles the case `from === to` by throwing a plain `ConflictError` with the same code `INVALID_STATUS_TRANSITION` and the same `{ from, to }` details. As a result, an identical code can be produced by two different class paths, and the specialized class is not the sole owner of its code.

The `from` and `to` values come from Prisma's `OrderStatus` enum, so they are constrained to a known set of strings. The tests assert the 409 status and the `INVALID_STATUS_TRANSITION` code for `PENDING -> SHIPPED` at `tests/orders.test.ts:89-107`, but they do not assert the `from` and `to` details. The specialization improves readability and gives the domain a named type, but it does not add behavior beyond fixing message, code, and details.

**Rule workflow**:
```
1. changeStatus loads the order and reads its current status
2. If from === to, throw ConflictError with code INVALID_STATUS_TRANSITION
3. Else consult canTransition(from, to) from the state machine
4. If the transition is illegal, throw InvalidStatusTransitionError(from, to)
5. errorMiddleware returns 409 with code INVALID_STATUS_TRANSITION and details { from, to }
```

---

### Business Rule: BR-12 - Insufficient stock uses a specialized 422 error with per-item availability

**Overview**:
When a stock debit cannot be satisfied, the domain throws `InsufficientStockError`, a subclass of `UnprocessableEntityError` that fixes the code to `INSUFFICIENT_STOCK` and reports every unavailable item with requested and available quantities.

**Detailed description**:
`InsufficientStockError` is declared at `src/shared/errors/http-errors.ts:55-62` and extends `UnprocessableEntityError`. Its constructor accepts `unavailable: { sku: string; requested: number; available: number }[]`, composes the fixed message `'One or more products do not have enough stock'`, fixes the code to `INSUFFICIENT_STOCK`, and passes details `{ unavailable }`. Because it extends `UnprocessableEntityError`, it inherits HTTP 422 and is caught by any handler that checks for the parent class.

The data is assembled in `OrderService.debitStock` at `src/modules/orders/order.service.ts:204-231`. The method fetches all referenced products, iterates over the order items, and pushes an entry whenever the product is missing or `stockQuantity < quantity`. A missing product is reported with `sku` equal to the product id and `available` equal to 0. The error is thrown before any product update occurs, and the entire operation runs inside `prisma.$transaction`, so no partial debit is committed.

The details array is the only place in the taxonomy where a nested per-item structure is a de facto contract. The test at `tests/orders.test.ts:109-132` asserts status 422, code `INSUFFICIENT_STOCK`, and `details.unavailable[0].sku`, and also verifies that stock remains unchanged after the failure. Multiple unavailable items, the missing-product fallback, and the `requested`/`available` values are not asserted. The fixed message is generic because the per-item detail carries the actionable information.

**Rule workflow**:
```
1. A transition requires a stock debit (PENDING -> PAID)
2. debitStock loads the referenced products inside the transaction
3. Each item is checked against available stock
4. If any item is unavailable, an entry is added to the unavailable array
5. If the array is non-empty, InsufficientStockError is thrown before any update
6. errorMiddleware returns 422 with code INSUFFICIENT_STOCK and details { unavailable }
7. The transaction rolls back, so stock stays unchanged
```

---

### Business Rule: BR-13 - Specialized errors inherit status and default behavior from their parent

**Overview**:
Domain-specific subclasses do not introduce new HTTP statuses. They inherit the status and base behavior of their parent and add only a fixed message, a fixed or refined code, and a fixed details shape.

**Detailed description**:
`InvalidStatusTransitionError` extends `ConflictError` (`http-errors.ts:45`), and `InsufficientStockError` extends `UnprocessableEntityError` (line 55). Both call the parent constructor through `super(...)`, so the `statusCode` assignment in `AppError` receives 409 and 422 respectively. They also inherit the parent's `instanceof` identity: an `InvalidStatusTransitionError` is a `ConflictError`, an `UnprocessableEntityError`, and an `AppError`. The error middleware only checks `instanceof AppError`, so the inheritance chain does not affect serialization.

The rule has two consequences. First, any code that catches `ConflictError` will also catch invalid transitions, which may be desired for a generic conflict handler but can surprise code that expects only uniqueness conflicts. Second, the specialized classes cannot change the HTTP status without changing their parent or bypassing it, which keeps the category mapping stable. The taxonomy therefore uses inheritance for domain refinement, not for status remapping.

Inheritance is shallow and intentional: two levels below `AppError` at most. There are no mixins, no interfaces, and no abstract methods. The parent classes are concrete and directly instantiable, so the hierarchy is not a strict abstract taxonomy; it is a set of convenience constructors over a shared base. This is consistent with the small size of the component but means there is no compile-time guarantee that a specialized class is semantically a subtype of its parent.

**Rule workflow**:
```
1. A domain-specific failure is recognized
2. The specialized subclass calls its parent constructor
3. The parent fixes the HTTP status and default code
4. The subclass fixes message, code and details
5. The instance is handled by the AppError branch and by any parent-class catch
```

---

### Business Rule: BR-14 - Error metadata is immutable and the stack trace is captured at construction

**Overview**:
All taxonomy metadata is read-only after construction. The base class sets the error name to `AppError` and captures a stack trace trimmed to the constructor call site.

**Detailed description**:
`statusCode`, `errorCode`, and `details` are declared `public readonly` at `src/shared/errors/app-error.ts:4-6`. TypeScript prevents reassignment in typed code, and the fields are assigned only once in the constructor at lines 11-13. The message itself comes from the built-in `Error` and is conventionally mutable, but no source file mutates it. The immutability rule ensures that middleware and logging cannot accidentally change the classification between throw and response.

Line 10 sets `this.name = 'AppError'` unconditionally in the base constructor. Because no subclass overrides `name`, every instance of every subclass reports `name === 'AppError'`. This is a deliberate or accidental loss of subtype identity: logs, error trackers, and serializers that use `err.name` will see only the base name, even though `err.constructor.name` and `instanceof` still identify the subclass. The integration tests do not assert `name`.

Line 14 calls `Error.captureStackTrace?.(this, this.constructor)` with optional chaining. On V8 runtimes this captures a stack trace and omits the constructor frame itself, so the stack starts at the caller that constructed the error. On runtimes without `captureStackTrace`, the call is a no-op and the standard `Error` stack behavior applies. The optional chaining prevents a crash but also means the behavior is runtime-dependent without a test that verifies it.

**Rule workflow**:
```
1. Subclass constructor calls super(...)
2. AppError sets name = 'AppError'
3. AppError assigns readonly statusCode, errorCode and details
4. AppError captures the stack excluding the constructor frame
5. No later layer mutates the instance fields
```

---

### Business Rule: BR-15 - Domain code throws taxonomy errors and never formats HTTP responses

**Overview**:
The taxonomy separates failure signaling from failure presentation. Services and middlewares construct and throw taxonomy errors; only the centralized error middleware converts them into HTTP JSON.

**Detailed description**:
Every controller in the project wraps its handler body in `try/catch` and calls `next(err)` rather than writing an error response. This is visible in `src/modules/orders/order.controller.ts:9-55`, `src/modules/auth/auth.controller.ts:12-38`, and the customer, product, and user controllers. Middlewares likewise call `next(new SomeError(...))` rather than writing a response: `src/middlewares/validate.middleware.ts:25-34`, `src/middlewares/auth.middleware.ts:27-60`. Services simply throw and have no dependency on Express.

A search for `res.status(...)` with a 4xx or 5xx code in `src/` finds matches only in `src/middlewares/error.middleware.ts` (lines 16, 27, 40, 49, 62). This confirms that the response envelope is owned by one place. The taxonomy itself knows nothing about Express, `res`, JSON, or HTTP headers; it only carries the data that the middleware needs. That decoupling is what allows `OrderService` and the other services to be unit-testable without an HTTP server, even though the current test suite does not exploit that.

The rule has an architectural cost: because the taxonomy does not own serialization, any change to the response envelope must be made in the middleware, and the taxonomy's fields are a de facto contract with that middleware. It also means that adding a new error class requires no middleware change, while adding a new response field requires a middleware change that applies to all classes.

**Rule workflow**:
```
1. Service detects a failure and throws a taxonomy instance
2. Controller catch calls next(err)
3. Express skips non-error middleware
4. errorMiddleware serializes the error
5. No controller or service writes an error status or error body directly
```

---

### Business Rule: BR-16 - Error responses use a fixed envelope and omit undefined details

**Overview**:
All taxonomy-originated responses have the shape `{ error: { code, message, details? } }`. `code` and `message` are always present; `details` is present only when the instance holds a value other than `undefined`.

**Detailed description**:
The AppError branch at `src/middlewares/error.middleware.ts:15-24` calls `res.status(err.statusCode).json({ error: { code: err.errorCode, message: err.message, ...(err.details !== undefined ? { details: err.details } : {}) } })`. The conditional spread is the rule: an omitted details value produces no `details` key, while a `null` value would be included because the check is specifically against `undefined`. The same envelope is used by the Zod, Prisma, and unknown branches, which helps clients parse errors uniformly.

The envelope does not include a timestamp, request id, path, method, stack trace, error name, or error class. Correlation is available only through the `X-Request-Id` response header set by `src/middlewares/request-logger.middleware.ts:8` and through server logs. This keeps error bodies small and avoids leaking internals, but it means a client cannot correlate an error body to a log entry without reading the header.

The envelope is not versioned and is not described by a schema in the codebase. Tests assert individual fields rather than the full object shape, for example `res.body.error.code` and `res.body.error.details`. Because the error body is not wrapped in a success-style `data` envelope, clients can distinguish success and failure by the presence of the top-level `error` key, but no explicit discriminator field exists.

**Rule workflow**:
```
1. errorMiddleware selects a branch
2. It always writes code and message
3. It conditionally adds details only when not undefined
4. It writes the object under the top-level error key
5. The HTTP status line carries the status code
```

---

### Business Rule: BR-17 - Unmatched routes are converted into taxonomy NotFoundError instances

**Overview**:
Any request that reaches the end of the middleware stack without matching a route is converted into a `NotFoundError` whose message includes the HTTP method and original URL.

**Detailed description**:
After the API router is mounted, `src/app.ts:69-71` installs a terminal middleware that calls `next(new NotFoundError(`Route ${req.method} ${req.originalUrl}`))`. This runs for every unmatched path and method, including paths outside `/api/v1`. The constructed error is a normal taxonomy instance, so the AppError branch of the error middleware handles it and returns 404 with code `NOT_FOUND`.

The generated message is `Route <METHOD> <URL> not found`, for example `Route GET /api/v1/unknown not found`. The URL is caller-controlled, but the value is placed into a JSON string and no HTML rendering occurs. The class does not accept details, so the response contains no structured path or method fields; clients must parse the message if they need the path, which they already know from their request. The route fallback is the only place where a taxonomy error is constructed from an HTTP request object outside the controllers and middlewares.

The fallback is installed before `errorMiddleware` (line 73) and after the API router (line 67). It therefore does not intercept `/health` (registered at lines 62-64) or any matched API route. The behavior is covered indirectly by the architectural analysis but not by any test in `tests/`; no test requests an unknown route and asserts the 404 body.

**Rule workflow**:
```
1. Request does not match /health or any /api/v1 route
2. Terminal middleware runs
3. It constructs NotFoundError('Route <METHOD> <URL>')
4. next(err) enters errorMiddleware
5. Client receives 404 { error: { code: NOT_FOUND, message: 'Route ... not found' } }
```

---

### Business Rule: BR-18 - Error codes are stable identifiers with a constrained override policy

**Overview**:
Error code strings are a public contract that clients and tests use for programmatic handling. Some classes fix their code, while three classes allow callers to replace the default with a domain-specific code.

**Detailed description**:
The override policy is encoded in constructor signatures. `BadRequestError` accepts `code = 'BAD_REQUEST'` (`http-errors.ts:4`), `ConflictError` accepts `code = 'CONFLICT'` (line 34), and `UnprocessableEntityError` accepts `code = 'UNPROCESSABLE_ENTITY'` (line 40). All other classes hardcode their code in the `super(...)` call and expose no code parameter. This means domain-specific codes can be added without new subclasses for 400, 409, and 422, but not for 401, 403, 404, or the two specialized types.

Codes are used as stable identifiers by tests (`tests/auth.test.ts:27,37,62`; `tests/orders.test.ts:56,106,127,217`) and are the primary field a client should branch on. They are not declared in a central enum, union type, or constant object. The same string can be produced by different classes and different layers: `INVALID_STATUS_TRANSITION` is produced by `InvalidStatusTransitionError` and by `ConflictError`, `VALIDATION_ERROR` is produced by `ValidationError` and by the raw Zod branch, `CONFLICT` is produced by `ConflictError` and by the Prisma P2002 branch, and `NOT_FOUND` is produced by `NotFoundError` and by the Prisma P2025 branch.

The rule is implicit rather than enforced. There is no runtime check for empty strings, casing, or duplicates, and no generated documentation or OpenAPI schema enumerates the codes. Stability is maintained by convention and by tests, not by a type-level contract. A future code change that renames a literal would compile and pass type checking, and only a test or a client contract would catch the break.

**Rule workflow**:
```
1. Select a taxonomy class whose status matches the failure category
2. Use the class default code, or supply a domain code if the class allows an override
3. The chosen code becomes part of the HTTP response
4. Clients and tests branch on the code
5. Renaming or duplicating a code is a contract change with no compile-time guard
```

---

## 4. Component Structure

```
src/shared/errors/
├── app-error.ts        # 16 lines - ErrorDetails type and AppError base class
├── http-errors.ts      # 63 lines - nine concrete HTTP/domain error classes
└── index.ts            # 13 lines - barrel that re-exports the public surface
```

### 4.1 File responsibilities

| File | Responsibility | Public surface |
|------|----------------|----------------|
| `src/shared/errors/app-error.ts` | Defines the `ErrorDetails` union and the `AppError` base class with status code, error code, optional details, name assignment, and stack capture | `AppError`, `ErrorDetails` |
| `src/shared/errors/http-errors.ts` | Defines the concrete hierarchy: 400, 401, 403, 404, 409, 422 classes plus two domain specializations | Nine classes |
| `src/shared/errors/index.ts` | Aggregates and re-exports the public API so consumers import from one path | `AppError`, `ErrorDetails`, nine classes |

### 4.2 Class hierarchy

```
Error (global)
└── AppError                         app-error.ts:3
    ├── BadRequestError              400  BAD_REQUEST
    ├── ValidationError              400  VALIDATION_ERROR
    ├── UnauthorizedError            401  UNAUTHORIZED
    ├── ForbiddenError               403  FORBIDDEN
    ├── NotFoundError                404  NOT_FOUND
    ├── ConflictError                409  CONFLICT
    │   └── InvalidStatusTransitionError  409 INVALID_STATUS_TRANSITION
    └── UnprocessableEntityError     422  UNPROCESSABLE_ENTITY
        └── InsufficientStockError   422  INSUFFICIENT_STOCK
```

### 4.3 Constructor contracts

| Class | Parameters | Status | Default code | Details |
|-------|------------|--------|--------------|---------|
| `AppError` | `message, statusCode, errorCode, details?` | caller-supplied | caller-supplied | optional |
| `BadRequestError` | `message = 'Bad request', code = 'BAD_REQUEST', details?` | 400 | `BAD_REQUEST` | optional |
| `ValidationError` | `message = 'Validation failed', details?` | 400 | `VALIDATION_ERROR` fixed | optional |
| `UnauthorizedError` | `message = 'Unauthorized'` | 401 | `UNAUTHORIZED` fixed | not accepted |
| `ForbiddenError` | `message = 'Forbidden'` | 403 | `FORBIDDEN` fixed | not accepted |
| `NotFoundError` | `resource = 'Resource'` | 404 | `NOT_FOUND` fixed | not accepted |
| `ConflictError` | `message, code = 'CONFLICT', details?` | 409 | `CONFLICT` | optional |
| `UnprocessableEntityError` | `message, code = 'UNPROCESSABLE_ENTITY', details?` | 422 | `UNPROCESSABLE_ENTITY` | optional |
| `InvalidStatusTransitionError` | `from, to` | inherited 409 | `INVALID_STATUS_TRANSITION` fixed | fixed `{from,to}` |
| `InsufficientStockError` | `unavailable[]` | inherited 422 | `INSUFFICIENT_STOCK` fixed | fixed `{unavailable}` |

### 4.4 Absent structure

There are no test files inside `src/shared/errors/`, no index of error codes, no constants module, no error factory, no error-code enum, no serialization helper, and no documentation file inside the component directory. The component boundary is exactly the three files above; everything else that references it is a consumer.

---

## 5. Dependency Analysis

### 5.1 Internal dependency graph

```
Consumers (11 distinct source files)
  src/app.ts
  src/middlewares/error.middleware.ts
  src/middlewares/auth.middleware.ts
  src/middlewares/validate.middleware.ts
  src/modules/auth/auth.controller.ts
  src/modules/auth/auth.service.ts
  src/modules/orders/order.controller.ts
  src/modules/orders/order.service.ts
  src/modules/customers/customer.service.ts
  src/modules/products/product.service.ts
  src/modules/users/user.service.ts
            |
            v
  src/shared/errors/index.ts  (barrel)
       |               |
       v               v
  app-error.ts  <--  http-errors.ts
       |
       v
  ECMAScript global Error (no project or npm dependency)
```

### 5.2 Consumer-to-symbol map

| Consumer file | Symbols imported | Purpose |
|---------------|------------------|---------|
| `src/app.ts:20` | `NotFoundError` | Route fallback for unmatched requests |
| `src/middlewares/error.middleware.ts:4` | `AppError` | Type guard for the serialization branch |
| `src/middlewares/auth.middleware.ts:4` | `UnauthorizedError`, `ForbiddenError` | 401 and 403 responses for auth and role checks |
| `src/middlewares/validate.middleware.ts:3` | `ValidationError` | Convert Zod failures into the taxonomy |
| `src/modules/auth/auth.controller.ts:4` | `UnauthorizedError` | Guard when `req.user` is absent on `/auth/me` |
| `src/modules/auth/auth.service.ts:5` | `UnauthorizedError` | Invalid credentials |
| `src/modules/orders/order.controller.ts:4` | `UnauthorizedError` | Guard when `req.user` is absent on create and status change |
| `src/modules/orders/order.service.ts:9-16` | `ConflictError`, `InsufficientStockError`, `InvalidStatusTransitionError`, `NotFoundError`, `UnprocessableEntityError`, `ValidationError` | All order-domain failure categories |
| `src/modules/customers/customer.service.ts:3` | `ConflictError`, `NotFoundError` | Duplicate email and missing customer |
| `src/modules/products/product.service.ts:3` | `ConflictError`, `NotFoundError` | Duplicate SKU and missing product |
| `src/modules/users/user.service.ts:4` | `ConflictError`, `NotFoundError` | Duplicate email and missing user |

### 5.3 Intra-component imports

```
src/shared/errors/http-errors.ts:1  ->  ./app-error.js        (AppError, ErrorDetails)
src/shared/errors/index.ts:1-2      ->  ./app-error.js        (AppError, ErrorDetails)
src/shared/errors/index.ts:3-13     ->  ./http-errors.js      (nine classes)
```

### 5.4 External dependencies

| Dependency | Type | Version | Usage in component |
|------------|------|---------|--------------------|
| None | - | - | The component imports no npm packages |
| ECMAScript `Error` | Language built-in | ES2022 target | Base class and `Error.captureStackTrace` |

The component has zero runtime third-party dependencies and zero transitive project dependencies. It does not import Express, Prisma, Zod, Pino, or any configuration module. That is why it can be used by every layer without creating cycles.

### 5.5 Reverse dependency notes

The barrel `src/shared/errors/index.ts` is the highest fan-in module in the system (11 external files). Two architectural consequences follow. First, any breaking change to an exported class or type propagates to all layers. Second, because consumers import from the barrel rather than deep paths, the component can add or reorganize files internally without touching consumers, as long as the barrel's exports remain stable.

The error middleware is a special consumer: it imports only `AppError`, but its behavior depends on the fields defined by the base class. It does not import any concrete subclass, so adding a subclass requires no middleware change. This is a deliberate inversion: the taxonomy knows nothing about the middleware, while the middleware knows the base contract.

---

## 6. Afferent and Efferent Coupling

### 6.1 Measurement definition

Afferent coupling (Ca) is the number of distinct files that import the symbol or file, including the barrel re-export where applicable. Efferent coupling (Ce) is the number of distinct internal modules the symbol or file imports. External npm packages are excluded. Counts are static and include type-only imports. For the whole component, intra-component imports are excluded from Ca/Ce, which yields Ca = 11 and Ce = 0, matching the architectural report.

### 6.2 File-level coupling

| File | Afferent Coupling | Efferent Coupling | Critical |
|------|-------------------|-------------------|----------|
| `src/shared/errors/index.ts` | 11 | 2 | High |
| `src/shared/errors/http-errors.ts` | 1 | 1 | High |
| `src/shared/errors/app-error.ts` | 2 | 0 | High |

### 6.3 Symbol-level coupling

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| `AppError` | 3 (`http-errors.ts`, `index.ts`, `error.middleware.ts`) | 0 | High |
| `BadRequestError` | 1 (`index.ts` only) | 1 (`AppError`) | Low |
| `ValidationError` | 3 (`index.ts`, `validate.middleware.ts`, `order.service.ts`) | 1 (`AppError`) | Medium |
| `UnauthorizedError` | 5 (`index.ts`, `auth.controller.ts`, `order.controller.ts`, `auth.service.ts`, `auth.middleware.ts`) | 1 (`AppError`) | Medium |
| `ForbiddenError` | 2 (`index.ts`, `auth.middleware.ts`) | 1 (`AppError`) | Medium |
| `NotFoundError` | 6 (`index.ts`, `app.ts`, `order.service.ts`, `user.service.ts`, `customer.service.ts`, `product.service.ts`) | 1 (`AppError`) | High |
| `ConflictError` | 6 (`index.ts`, `order.service.ts`, `user.service.ts`, `customer.service.ts`, `product.service.ts`, `http-errors.ts` as parent) | 1 (`AppError`) | High |
| `UnprocessableEntityError` | 3 (`index.ts`, `order.service.ts`, `http-errors.ts` as parent) | 1 (`AppError`) | Medium |
| `InvalidStatusTransitionError` | 2 (`index.ts`, `order.service.ts`) | 1 (`ConflictError`) | High |
| `InsufficientStockError` | 2 (`index.ts`, `order.service.ts`) | 1 (`UnprocessableEntityError`) | High |

The "Critical" column is an architectural criticality assessment based on blast radius and domain impact, not a computed metric. The two specialized classes are marked High because they carry order-domain invariants that directly affect money and inventory. `BadRequestError` is Low because it is exported but unused. `NotFoundError` and `ConflictError` are High because they are the most widely reused categories across all feature modules.

The component is maximally stable in Martin's terms: high Ca, zero Ce. It is a responsibility sink, not an unstable consumer. That is appropriate for a shared kernel, but it also means that any defect in the component has system-wide reach.

---

## 7. Integration Points

The component has no database, network, or third-party service integration. Its integrations are in-process contracts and the HTTP boundary owned by the error middleware.

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Domain services -> taxonomy | In-process library contract | Represent expected failures with status, code, message and details | TypeScript constructor calls and `throw` | `AppError` instance | Caller chooses the subclass; no retry or fallback |
| Controllers -> error middleware | Express error pipeline | Forward thrown errors without formatting | `next(err)` | Error object | `try/catch` in every controller; no swallowing |
| Validation middleware -> taxonomy | In-process adapter | Convert `ZodError` into `ValidationError` | Function call inside `catch` | `{path,message}[]` details | If the caught error is not `ZodError`, it is forwarded unchanged |
| Auth middleware -> taxonomy | In-process construction | Signal 401/403 security decisions | Function call via `next` | `UnauthorizedError` / `ForbiddenError` | Missing header, empty token, invalid JWT, missing user, wrong role |
| Route fallback -> taxonomy | In-process construction | Convert unmatched routes into 404 | Function call via `next` | `NotFoundError` | Terminal middleware after the API router |
| Taxonomy -> error middleware | In-process consumption | Serialize taxonomy fields into HTTP JSON | `instanceof` and property access | `{ error: { code, message, details? } }` | Branch order: AppError first, then Zod, Prisma, unknown |
| Error middleware -> HTTP client | HTTP response | Deliver the error contract | HTTPS/HTTP JSON | JSON body plus HTTP status | Fixed envelope; unknown errors masked as 500 |
| Error middleware -> Pino logger | Observability | Record unhandled errors with request correlation | Function call | Pino JSON log | Only the unknown branch logs; AppError 4xx responses are not logged with the error object |
| Prisma errors -> HTTP response | Parallel error path (outside taxonomy) | Map `P2002` to 409 and `P2025` to 404 | In-process branch | Literal code strings | Does not construct taxonomy instances; other Prisma codes fall through to 500 |

The notable integration gaps are that the taxonomy is not the only source of error codes (the Zod and Prisma branches write literals), and that body-parser errors do not participate in the taxonomy at all. Both are recorded in section 9.

---

## 8. Design Patterns & Architecture

### 8.1 Identified patterns

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layer supertype / base exception | `AppError extends Error` with shared metadata | `src/shared/errors/app-error.ts:3-16` | One contract for all expected failures |
| Specialized exception hierarchy | Nine concrete subclasses, two of them second-level | `src/shared/errors/http-errors.ts:3-62` | Semantic failure categories with fixed HTTP mapping |
| Rich domain exception | Error code plus structured `details` | `src/shared/errors/app-error.ts:4-6` | Machine handling and contextual diagnostics |
| Information holder / DTO | `ErrorDetails` union and per-class details shapes | `src/shared/errors/app-error.ts:1`; `http-errors.ts:47-61` | Carry context without behavior |
| Immutable value object fields | `readonly statusCode`, `errorCode`, `details` | `src/shared/errors/app-error.ts:4-6` | Prevent classification drift after construction |
| Default parameter construction | Defaults for message, code, resource and optional details | `src/shared/errors/http-errors.ts:4,10,16,22,28,34,40` | Reduce ceremony for common cases |
| Barrel / facade module | `index.ts` re-exports the public API | `src/shared/errors/index.ts:1-13` | Stable import path and encapsulation of file layout |
| Hierarchy-based type guard | `err instanceof AppError` | `src/middlewares/error.middleware.ts:15` | Single classification point in the HTTP layer |
| Exception translation / adapter | Middleware maps domain errors to HTTP JSON | `src/middlewares/error.middleware.ts:14-64` | Keep HTTP concerns out of the domain |
| Single error handler | One Express `ErrorRequestHandler` registered last | `src/app.ts:73` | Uniform error envelope and branch ordering |
| Open/closed extension | New failure types added by subclassing, not by editing the middleware | `src/shared/errors/http-errors.ts`; `src/middlewares/error.middleware.ts:15` | Extensibility without modifying the consumer |
| Code-as-contract | Uppercase snake-case string identifiers | All constructors and domain throws | Stable client-facing API surface |

### 8.2 Architectural decisions evidenced by the code

- **Class per HTTP category, not class per code.** The hierarchy fixes one status per class and uses code overrides for finer distinctions. This keeps the middleware branch simple but pushes code uniqueness into convention.
- **Details are untyped and pass-through.** The component deliberately avoids a schema dependency, which keeps it dependency-free but makes payload shapes informal.
- **Messages are defaults, codes are contracts.** Defaults make construction ergonomic, while tests and clients branch on codes, not messages.
- **No HTTP framework dependency in the domain.** Services throw taxonomy errors and do not import Express. The translation boundary is the middleware.
- **Inheritance is used for domain specialization.** `InvalidStatusTransitionError` and `InsufficientStockError` are named domain concepts that reuse the parent category rather than inventing a new HTTP status.
- **The base class is concrete and directly constructible.** There is no `abstract` keyword and no protected constructor. The hierarchy is a convenience taxonomy, not a sealed algebraic type.
- **No logging inside the taxonomy.** The component never logs; logging is a consumer concern handled by the middleware and request logger.
- **No serialization inside the taxonomy.** The component never calls `toJSON`; serialization is owned by the middleware.
- **No runtime validation of metadata.** Status codes and error codes are trusted from constructors. TypeScript's `readonly` is the only guard.

---

## 9. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Error identity / observability | `this.name` is hardcoded to `'AppError'` for every instance (`src/shared/errors/app-error.ts:10`); no subclass overrides it | Logs, APM tools and error trackers that group by `err.name` cannot distinguish `NotFoundError` from `InsufficientStockError`; debugging and alerting lose subtype signal |
| High | Taxonomy completeness | No taxonomy class for transport-level failures; malformed JSON and oversized bodies from `express.json()` are `SyntaxError`/413 errors that do not match any branch (`src/middlewares/error.middleware.ts:14-64`) | Client-side syntax errors are returned as 500 `INTERNAL_SERVER_ERROR`, polluting error logs and giving clients misleading status codes |
| High | Taxonomy completeness | Only Prisma `P2002` and `P2025` are mapped; `P2003` foreign key, `P2034` deadlock, `P2024` pool timeout and connection errors fall through to 500 | Infrastructure and constraint failures are indistinguishable from programming bugs; no retry or classification signal for clients |
| Medium | Error-code governance | No central registry, enum, union type or uniqueness check for codes; `INVALID_STATUS_TRANSITION` is produced by both `InvalidStatusTransitionError` (`http-errors.ts:49`) and `ConflictError` (`order.service.ts:141-146`) | Code drift and accidental duplication become public contract changes; clients cannot enumerate valid codes from a type |
| Medium | Error-details contract | `ErrorDetails` is a record or an array (`Record<string, unknown>` or `unknown[]`) and is passed through without validation or cloning (`app-error.ts:1,13`); no runtime check for JSON serializability | Inconsistent payload shapes across endpoints; a circular or non-serializable value could fail during response serialization |
| Medium | Test coverage | No direct unit tests for any taxonomy class; no test file under `src/shared/errors/`; all coverage is indirect via `tests/auth.test.ts` and `tests/orders.test.ts` | `BadRequestError`, `ForbiddenError`, `INACTIVE_PRODUCT`, route fallback, default-message branches and the details-omission branch are unverified |
| Medium | Security / authorization | `ForbiddenError` is defined and wired through `requireRole` (`src/middlewares/auth.middleware.ts:55-58`, `src/modules/users/user.routes.ts:12-16`) but no test asserts a 403 | A regression in role enforcement or in the 403 serialization would not be caught |
| Medium | Not-found context | `NotFoundError` accepts only a resource label, no message or details (`http-errors.ts:27-31`); the route fallback embeds the raw URL into the message (`src/app.ts:70`) | 404 responses cannot identify the missing identifier; the fallback message is awkward and reflects caller-controlled input into logs |
| Medium | Observability of client errors | The `AppError` branch of the middleware does not log (`error.middleware.ts:15-24`); only the unknown branch logs | No audit trail for 4xx business failures beyond the request access log, which records status and path but not error code |
| Low | Dead public surface | `BadRequestError` is exported (`index.ts:4`) but never constructed in `src/` or `tests/` | Misleads consumers into thinking it is a supported, exercised path; two overlapping 400 classes create choice ambiguity |
| Low | Metadata validation | `statusCode` accepts any number; `errorCode` accepts any string including an empty or malformed value (`app-error.ts:8-13`) | A direct `new AppError` or a future subclass could emit an invalid status or code without a type or runtime error |
| Low | Error chaining | `AppError` has no `cause` support and does not wrap the original error (`app-error.ts:8-14`) | Root-cause information is lost when an infrastructure error is translated into a domain error; `details` is not a substitute for a cause chain |
| Low | Name and stack runtime behavior | `Error.captureStackTrace?.` is optional and silently no-ops on non-V8 runtimes (`app-error.ts:14`); no test verifies stack content | Runtime-dependent diagnostics; stack behavior is untested |
| Low | Details redaction | The taxonomy places no constraint on `details`, and response serialization does not run redaction (Pino redaction applies only to logs) | A future details payload could expose internal identifiers or sensitive context to clients |
| Low | Route fallback test gap | The terminal 404 middleware (`src/app.ts:69-71`) is not exercised by any test | A change in middleware order or in the fallback could break 404 semantics silently |

---

## 10. Test Coverage Analysis

### 10.1 Test inventory

| Test file | Type | Relevance to Error Taxonomy |
|-----------|------|-----------------------------|
| `tests/auth.test.ts` | HTTP integration via Supertest | Exercises `ValidationError`, `UnauthorizedError`, `ConflictError` (`EMAIL_ALREADY_USED`) and the serialization envelope |
| `tests/orders.test.ts` | HTTP integration via Supertest | Exercises `NotFoundError`, `InvalidStatusTransitionError`, `InsufficientStockError`, `ConflictError` (`INVALID_ORDER_STATE_FOR_DELETE`) |
| `tests/helpers/factories.ts` | Test infrastructure | Builds the app and fixtures; throws a plain `Error` on login failure at line 37, which is outside the taxonomy |
| `tests/setup.ts` | Test infrastructure | Database lifecycle only; no error assertions |
| `vitest.config.ts` | Configuration | Includes `tests/**/*.test.ts`, no coverage provider, no thresholds |
| `package.json` | Configuration | Has `test` and `test:watch` scripts, no coverage script and no coverage package |

There are no unit tests, no snapshot tests, and no property-based tests for the taxonomy. There is no `*.test.ts` file inside `src/`. Coverage is measured only indirectly through HTTP responses, because the tests assert `res.body.error.*` after driving real service behavior.

### 10.2 Coverage by taxonomy symbol

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| `AppError` | 0 direct | Indirectly executed by every error path in both test files | High indirect line execution; not measured | Constructor fields and stack capture are not asserted directly; `name` and readonly behavior untested |
| `BadRequestError` | 0 | 0 | None | Exported but never constructed or asserted |
| `ValidationError` | 0 | 1 scenario (`tests/auth.test.ts:30-39`) | Executed via `validate.middleware.ts:31` | Status 400, code `VALIDATION_ERROR` and `Array.isArray(details)` asserted; message, default message and object-shaped details untested |
| `UnauthorizedError` | 0 | 2 scenarios (`tests/auth.test.ts:54-63`, `tests/auth.test.ts:79-83`) | Executed for login and `/auth/me` | 401 and `UNAUTHORIZED` asserted once; missing-header, empty-token, invalid-token and default-message branches untested |
| `ForbiddenError` | 0 | 0 | None | 403 path wired through `requireRole` but never exercised |
| `NotFoundError` | 0 | 1 scenario (`tests/orders.test.ts:42-57`) | Executed for a missing product | 404 and `NOT_FOUND` asserted; missing order, customer, user and the route fallback untested; resource interpolation not asserted |
| `ConflictError` | 0 | 2 scenarios (`tests/auth.test.ts:19-28`, `tests/orders.test.ts:197-218`) | Executed with code overrides | `EMAIL_ALREADY_USED` and `INVALID_ORDER_STATE_FOR_DELETE` asserted; `SKU_ALREADY_USED`, default `CONFLICT` code, details payload and Prisma `P2002` mapping untested |
| `UnprocessableEntityError` | 0 | 0 direct; parent constructor executed via `InsufficientStockError` | Constructor lines reached indirectly | `INACTIVE_PRODUCT` behavior at `order.service.ts:66-73` untested; default code branch untested |
| `InvalidStatusTransitionError` | 0 | 1 scenario (`tests/orders.test.ts:89-107`) | Executed | 409 and `INVALID_STATUS_TRANSITION` asserted; `{from,to}` details untested; same-status `ConflictError` variant untested |
| `InsufficientStockError` | 0 direct | 1 scenario (`tests/orders.test.ts:109-132`) with one unavailable item | Executed | 422, code and `details.unavailable[0].sku` asserted; transaction rollback effect on stock asserted; multiple unavailable items, missing-product fallback, `requested`/`available` values and fixed message untested |
| `index.ts` barrel | 0 | Used by every consumer | Not applicable | Export completeness and re-export stability untested |

### 10.3 Coverage gaps and test quality observations

The tests validate the most user-visible contract points: status code plus machine code for validation, authentication, not-found, conflict, invalid transition, and insufficient stock. They also verify two details shapes: the validation details array and the `unavailable[0].sku` inventory field. This is enough to catch gross regressions in the six most common categories. It is not enough to protect the full taxonomy.

Seven behaviors are entirely unverified: `BadRequestError` construction, `ForbiddenError` and the 403 path, `INACTIVE_PRODUCT` and the 422 inactive-product branch, the route-fallback 404, default messages for every class, the `details` omission branch when `details` is `undefined`, and the `name`/stack-capture behavior. Several negative branches are also untested, including `NotFoundError` for missing order, customer and user, `ConflictError` with the default `CONFLICT` code, `SKU_ALREADY_USED`, and the Zod safety-net branch in `error.middleware.ts:26-35`. The Prisma `P2002` and `P2025` branches are similarly unexercised, so the parallel error paths outside the taxonomy are unverified.

Coverage cannot be stated as a precise percentage because no coverage run was requested and no coverage provider is configured in `vitest.config.ts`. Statically, all constructor bodies except `BadRequestError` and `ForbiddenError` are reachable through the current integration tests, so line coverage of `http-errors.ts` is likely high, while branch coverage is materially lower because defaults and alternate details shapes are not exercised. The absence of direct unit tests also means the tests are slow and database-dependent: they require a live MySQL instance through `tests/setup.ts`, which makes them less suitable as a fast feedback loop for a dependency-free component that could be tested in isolation.

---

## Analysis Notes and Ambiguities

- The taxonomy's completeness boundary is not documented in the component. The statement that only expected domain failures use `AppError` is inferred from usage, not from an explicit comment or type constraint.
- The intent behind `BadRequestError` is ambiguous. It is fully implemented and exported but never used, and its semantics overlap with `ValidationError`.
- The intent behind hardcoding `name = 'AppError'` is ambiguous. It may be deliberate normalization for serialization or an oversight; no comment or test explains it.
- The stability of error codes is implicit. Tests assert several codes, but there is no registry or generated contract that declares them public.
- The `details` schema for `INACTIVE_PRODUCT`, `EMAIL_ALREADY_USED`, `SKU_ALREADY_USED`, and `INVALID_ORDER_STATE_FOR_DELETE` is defined only by the call sites, not by a type or validator.
- The component has no relation to the future webhook design described in `TRANSCRICAO.md`; it is the currently implemented error taxonomy only. Any planned reuse of `AppError` for webhook delivery is outside the analyzed source tree.
- All counts and line references are static, based on the repository state at analysis time. No code was executed, no tests were run, and no files were modified.
