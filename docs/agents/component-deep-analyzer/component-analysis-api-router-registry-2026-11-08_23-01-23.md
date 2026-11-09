# Component Deep Analysis Report

**Component:** API Router Registry
**Primary file:** `src/routes/index.ts`
**Project:** `mba-ia-desafio-design-docs-com-ia` (order-management-api)
**Analysis date:** 2026-11-08 23:01:23
**Scope:** component boundary plus directly attached route definitions in the five feature modules and the mounting code in `src/app.ts`
**Excluded from analysis:** `node_modules`, `.git`, `dist`, `coverage`

---

## 1. Executive Summary

The API Router Registry is the HTTP surface aggregation component of a modular monolith. Its primary file, `src/routes/index.ts` (31 lines), declares a `Controllers` contract (`src/routes/index.ts:13-19`) and exposes a single factory, `buildApiRouter(controllers)` (`src/routes/index.ts:21-31`), which creates one Express `Router` and mounts five module routers under fixed prefixes:

| Prefix | Module router factory | Source |
|--------|-----------------------|--------|
| `/auth` | `buildAuthRouter` | `src/modules/auth/auth.routes.ts` |
| `/users` | `buildUserRouter` | `src/modules/users/user.routes.ts` |
| `/customers` | `buildCustomerRouter` | `src/modules/customers/customer.routes.ts` |
| `/products` | `buildProductRouter` | `src/modules/products/product.routes.ts` |
| `/orders` | `buildOrderRouter` | `src/modules/orders/order.routes.ts` |

The registry itself contains no business logic, no persistence access, and no request handlers. It is a pure composition layer: it converts a typed bag of controllers into a routing tree. The actual route table (19 HTTP endpoints) lives in the five `*.routes.ts` files, which the registry imports and delegates to. For that reason this analysis treats the five route builders as part of the component boundary, because they define the observable contract that the registry publishes.

The component is mounted exactly once, by the composition root, at `src/app.ts:67` (`app.use('/api/v1', buildApiRouter(controllers))`). The only HTTP route outside the registry is the liveness probe `GET /health` (`src/app.ts:62-64`). After the registry, an unmatched-path middleware produces a 404 `NotFoundError` (`src/app.ts:69-71`), followed by the centralized error middleware (`src/app.ts:73`).

Key findings:

- The component is a thin, stable, low-logic aggregation layer with one afferent dependency (`src/app.ts`) and ten internal efferent dependencies (five controller contracts plus five router factories). It has no cycles.
- Authorization is inconsistent at the routing layer: only `GET /users/:id` applies `requireRole('ADMIN')` (`src/modules/users/user.routes.ts:15`). Customers, products and orders apply authentication only (`router.use(authenticate)` at `customer.routes.ts:14`, `product.routes.ts:14`, `order.routes.ts:14`). Any authenticated actor can perform full CRUD on those resources.
- Authentication is applied as a router-level guard on three modules, which means requests to unknown paths under `/customers`, `/products` and `/orders` return `401` before the app-level 404 handler is reached. This is an observable behavioral rule that is not documented anywhere in the codebase.
- The public `POST /api/v1/auth/register` route accepts a client-supplied `role` field including `ADMIN` (`src/modules/auth/auth.schemas.ts:7`, applied at `src/modules/auth/auth.routes.ts:10`), and `AuthService.register` forwards it to `UserService.createUser`, which persists it unchanged (`src/modules/users/user.service.ts:31`). This is a privilege-escalation surface reachable without authentication.
- Validation is fail-fast and uniform: every non-trivial endpoint applies `validate(...)` with Zod schemas before the controller runs (`src/middlewares/validate.middleware.ts:11-37`), producing `400 VALIDATION_ERROR` with `details[]`.
- Two parsing failure modes bypass the `AppError` taxonomy: malformed JSON and oversized bodies raise body-parser errors that the error middleware does not recognize, so they are logged and answered as `500 INTERNAL_SERVER_ERROR` (`src/middlewares/error.middleware.ts:56-64`).
- Test coverage is integration-only. `tests/auth.test.ts` (7 tests) and `tests/orders.test.ts` (8 tests) exercise the registry indirectly through Supertest. There are zero direct tests for `src/routes/index.ts`, zero tests for the users, customers and products route files, and no coverage instrumentation in the project.

Confidence note: all rules below are derived directly from source. Where a behavior depends on framework semantics (Express 4.21.1 router and middleware propagation), it is marked as such. No undocumented domain rule is inferred.

---

## 2. Data Flow Analysis

```
1. HTTP request arrives at the Express app created by buildApp(deps) (src/app.ts:55-76).
2. Global middleware: x-powered-by disabled, express.json({ limit: '1mb' }) parses JSON bodies (src/app.ts:58-59).
3. requestLogger assigns or propagates X-Request-Id, sets the response header, and starts a duration timer (src/middlewares/request-logger.middleware.ts:5-27).
4. If the path is exactly /health, the handler answers 200 { status: 'ok' } and the router registry is bypassed (src/app.ts:62-64).
5. Otherwise app.use('/api/v1', apiRouter) strips the /api/v1 prefix and enters the registry (src/app.ts:67, src/routes/index.ts:24-28).
6. The registry matches the first path segment against its five fixed prefixes and delegates to the corresponding module router.
7. Module-level guard: customers, products and orders run authenticate on every request entering the router (customer.routes.ts:14, product.routes.ts:14, order.routes.ts:14); auth runs authenticate only on /me (auth.routes.ts:12); users runs it per-route (user.routes.ts:14).
8. Route-level guard: users additionally runs requireRole('ADMIN') (user.routes.ts:15).
9. validate(...) parses body, query and/or params with Zod, replacing or merging the parsed values onto the request; a ZodError is converted to ValidationError (src/middlewares/validate.middleware.ts:11-37).
10. The bound controller handler runs (for example controller.create), reads req.body/req.params/req.query/req.user, and calls the module service.
11. The service calls its repository; the repository issues Prisma queries against MySQL. This is the only data persistence path reachable from the registry.
12. The controller writes the response: 201 for creates, 200 for reads and updates, 204 for deletes, or forwards an error with next(err).
13. On error, Express skips remaining route handlers and reaches errorMiddleware (src/app.ts:73), which maps AppError, ZodError and Prisma P2002/P2025 to JSON envelopes, and masks all other errors as 500 (src/middlewares/error.middleware.ts:14-65).
14. If no route matched and no error occurred, the app-level fallback creates NotFoundError('Route <METHOD> <originalUrl>') and returns 404 NOT_FOUND (src/app.ts:69-71).
15. The response 'finish' event triggers the access log line with requestId, method, path, statusCode, durationMs and userId (src/middlewares/request-logger.middleware.ts:12-25).
```

Control-flow summary: `app -> /api/v1 registry -> module router -> [authenticate] -> [requireRole] -> validate -> controller -> service -> repository -> Prisma -> MySQL -> controller response -> [errorMiddleware on failure] -> requestLogger completion log`.

---

## 3. Business Rules & Logic

### Overview of the business rules

| ID | Rule Type | Rule Description | Location |
|----|-----------|------------------|----------|
| BR-01 | Routing / API contract | All business endpoints are published under the single versioned prefix `/api/v1` | `src/app.ts:67`, `src/routes/index.ts:21-31` |
| BR-02 | Routing | Exactly five module prefixes are registered: `/auth`, `/users`, `/customers`, `/products`, `/orders`; no aliases exist | `src/routes/index.ts:24-28` |
| BR-03 | Contract / composition | The registry requires a complete `Controllers` object with all five controller instances; absence is a compile-time error | `src/routes/index.ts:13-19`, `src/app.ts:26-53` |
| BR-04 | Security / routing | `POST /auth/register` and `POST /auth/login` are public; `GET /auth/me` requires a Bearer token | `src/modules/auth/auth.routes.ts:10-12` |
| BR-05 | Security / routing | Every customers, products and orders endpoint requires a valid JWT, enforced at router level before any route match | `src/modules/customers/customer.routes.ts:14`, `src/modules/products/product.routes.ts:14`, `src/modules/orders/order.routes.ts:14` |
| BR-06 | Security / authorization | `GET /users/:id` is restricted to authenticated users with role `ADMIN` | `src/modules/users/user.routes.ts:12-18` |
| BR-07 | Validation | All mutating and parameterized endpoints validate input with Zod before executing business logic; failures return 400 `VALIDATION_ERROR` | `src/middlewares/validate.middleware.ts:11-37` |
| BR-08 | Validation | Every `:id` path parameter must be a UUID; non-UUID identifiers are rejected with 400 before reaching services | `src/modules/orders/order.schemas.ts:4`, `customer.schemas.ts:3-5`, `product.schemas.ts:3`, `user.routes.ts:7` |
| BR-09 | Validation / pagination | List endpoints share a pagination contract: `page >= 1` (default 1), `pageSize` in `[1,100]` (default 20), plus module-specific filters | `customer.schemas.ts:25-29`, `product.schemas.ts:16-24`, `order.schemas.ts:23-30` |
| BR-10 | API contract | Order creation accepts `customerId`, at least one item (`productId`, positive integer `quantity`), optional non-negative `discountCents` (default 0) and optional `notes` (max 1000) | `src/modules/orders/order.routes.ts:18`, `order.schemas.ts:6-16` |
| BR-11 | API contract | Order status changes use a dedicated `PATCH /orders/:id/status` endpoint with `toStatus` from the `OrderStatus` enum and optional `reason` (max 500) | `src/modules/orders/order.routes.ts:19-23`, `order.schemas.ts:18-21` |
| BR-12 | API contract | Customer and product updates use shallow partial schemas: all top-level fields optional, but a supplied `address` must be complete | `src/modules/customers/customer.routes.ts:19-23`, `customer.schemas.ts:23`, `product.schemas.ts:14` |
| BR-13 | Error handling / routing | Any unmatched method+path that reaches the app-level fallback returns 404 `NOT_FOUND` with the method and original URL in the message | `src/app.ts:69-71` |
| BR-14 | Security / validation | Public registration accepts a client-supplied `role` (`ADMIN` or `OPERATOR`, default `OPERATOR`) and persists it without privilege checks | `src/modules/auth/auth.schemas.ts:7`, `src/modules/auth/auth.routes.ts:10`, `src/modules/users/user.service.ts:21-33` |

---

### Business Rule: BR-01 - Versioned API Namespace

**Overview**:
Every business endpoint is exposed exclusively through the `/api/v1` prefix. The registry is never mounted at the application root.

**Detailed description**:
The composition root mounts the registry with `app.use('/api/v1', buildApiRouter(controllers))` at `src/app.ts:67`. Because `buildApiRouter` itself returns a bare `Router()` with no internal base path (`src/routes/index.ts:22`), the version segment is owned entirely by the mounting call, not by the registry. This separates the versioning policy from the routing table: a future `/api/v2` would require a second mount rather than edits inside `src/routes/index.ts`. In the current codebase there is exactly one mount and therefore exactly one published API version. The only non-versioned route in the entire application is `GET /health` (`src/app.ts:62-64`), which is registered before the registry and is intentionally infrastructure-facing.

The rule has practical consequences for clients and for error semantics. A request to an unversioned path such as `POST /orders` never reaches the orders router; it falls through the entire middleware chain to the app-level 404 fallback and is reported as `Route POST /orders not found`. Similarly, a wrong version such as `/api/v2/orders` is a 404, not a redirect or a version-negotiation response. There is no `Accept-Version` header handling, no URL rewriting and no deprecation mechanism in the codebase.

Because the prefix is a mount path rather than a route, Express matches it as a path segment boundary: `/api/v1/orders` matches, while `/api/v1x/orders` does not. Matching is performed by Express 4 default semantics (case-insensitive paths, non-strict trailing slash), which means `/API/V1/ORDERS` and `/api/v1/orders/` also resolve to the orders list. This is framework behavior, not an explicit registry decision, and no test asserts it.

**Rule workflow**:
1. Client sends a request to a path beginning with `/api/v1`.
2. `express.json` and `requestLogger` run first (`src/app.ts:59-60`).
3. The `/health` route is checked; non-matching paths continue.
4. `app.use('/api/v1', ...)` strips the prefix and hands the remaining path to the registry.
5. A path that does not start with `/api/v1` never enters the registry and is answered by the 404 fallback.

---

### Business Rule: BR-02 - Fixed Module Prefix Registry

**Overview**:
The registry publishes exactly five module prefixes, in a fixed registration order, each bound to one module router factory and one controller instance.

**Detailed description**:
`src/routes/index.ts:24-28` performs five sequential `router.use(prefix, factory(controller))` calls: `/auth` with `controllers.auth`, `/users` with `controllers.users`, `/customers` with `controllers.customers`, `/products` with `controllers.products` and `/orders` with `controllers.orders`. The prefix is the only routing knowledge the registry owns. All methods, sub-paths, guards and validators are encapsulated inside the module routers, which is why the registry file is only 31 lines despite publishing 19 endpoints.

The prefixes are disjoint at the first segment, so registration order does not affect resolution today. This matters because the registry relies on prefix uniqueness as an implicit invariant: there is no runtime collision detection, no duplicate-prefix assertion and no test that enumerates the mounted prefixes. If a future module were mounted at `/` or at a prefix that is a prefix of another segment, the earlier mount could shadow later routes. The current order (`auth`, `users`, `customers`, `products`, `orders`) follows the module inventory rather than any functional grouping.

Each module router is built at application construction time, once per `buildApp` call, and holds references to controller methods captured at that moment. There is no lazy loading, no dynamic route registration and no feature-flag-controlled mounting. The registry therefore defines a static, deterministic route tree for the lifetime of the process. Adding a module requires coordinated edits in three places: the module folder, the `Controllers` type and factory call in `src/routes/index.ts`, and the controller construction in `src/app.ts:26-53`.

**Rule workflow**:
1. `buildApiRouter(controllers)` creates a new `Router()`.
2. It calls `buildAuthRouter(controllers.auth)` and mounts the result at `/auth`.
3. It repeats for `/users`, `/customers`, `/products`, `/orders` in that exact order.
4. The assembled router is returned and mounted once at `/api/v1`.
5. Any request whose first segment after `/api/v1` is not one of the five prefixes falls through to the 404 fallback.

---

### Business Rule: BR-03 - Complete Controllers Contract

**Overview**:
The registry cannot be constructed without all five controllers; the shape is enforced statically by the exported `Controllers` type.

**Detailed description**:
`Controllers` (`src/routes/index.ts:13-19`) is a structural type with five required properties: `auth`, `users`, `customers`, `products`, `orders`, each typed as the concrete controller class (`AuthController`, `UserController`, `CustomerController`, `ProductController`, `OrderController`). `buildApiRouter` accepts exactly this type, and `src/app.ts:26-53` constructs the object by instantiating every repository, service and controller in dependency order. Because all five properties are required and non-optional, TypeScript fails the build if a controller is missing, and the application cannot start with a partially wired registry.

The type is imported into `src/app.ts:19` together with the factory, so the composition root is forced to satisfy the same contract that the registry consumes. There is no runtime validation of the object: if the code were executed as untyped JavaScript with a missing controller, `build*Router(undefined)` would fail at controller method access time (for example `controller.register` at `src/modules/auth/auth.routes.ts:10`) during `buildApp`, before the server listens. In practice the compile step plus the single construction site makes this a build-time rule.

The contract also documents an architectural boundary: the registry depends on controller classes only through `import type` statements (`src/routes/index.ts:2-6`), so the controller imports are erased at runtime. The runtime dependency is on the five router factories (`src/routes/index.ts:7-11`), which in turn import their controllers as types as well. This keeps the registry free of service and repository knowledge, but it also means the contract is class-based rather than interface-based, so alternative implementations cannot be substituted without changing the controller class type.

**Rule workflow**:
1. `buildApp` calls `buildControllers(prisma)` (`src/app.ts:66`).
2. `buildControllers` instantiates repositories, services and controllers and returns an object literal with all five keys (`src/app.ts:46-52`).
3. The object is passed to `buildApiRouter`, which type-checks it against `Controllers`.
4. Each router factory receives its specific controller and binds handler methods.
5. A missing or mistyped controller prevents compilation or throws during app construction.

---

### Business Rule: BR-04 - Public Authentication Entry Points, Protected Identity Endpoint

**Overview**:
The auth module is the only place where unauthenticated business traffic is accepted: registration and login. Identity lookup requires a valid Bearer token.

**Detailed description**:
`src/modules/auth/auth.routes.ts:10` registers `POST /register` with only body validation, and line 11 registers `POST /login` with only body validation. Neither route includes `authenticate`, so both are reachable anonymously. This is required by the domain: a user cannot present a token before obtaining one. Registration validates `email`, `password` (8 to 72 characters), `name` (2 to 150 characters) and an optional `role` (BR-14). Login validates `email` and a password of 1 to 72 characters, deliberately allowing legacy or short passwords to reach the credential comparison rather than failing at the schema. Both routes return data through the controller: 201 with the public user projection for registration, 200 with `{ user, tokens }` for login.

`GET /me` at line 12 is different: it binds `authenticate` inline before `controller.me`. The middleware requires an `Authorization` header whose value starts with `bearer ` (case-insensitive), extracts the token after character 7, verifies it with `env.JWT_SECRET` and hydrates `req.user` with `{ id, email, role }` from the `sub`, `email` and `role` claims (`src/middlewares/auth.middleware.ts:27-47`). Missing headers, empty tokens and invalid or expired tokens all produce `UnauthorizedError` with code `UNAUTHORIZED` and status 401. The controller adds a defensive second check (`if (!req.user) throw new UnauthorizedError()` at `src/modules/auth/auth.controller.ts:32`) even though the middleware guarantees it in the normal pipeline.

There is no logout, refresh-token, password-reset or token-revocation endpoint in the routing table. Tokens are stateless JWTs with an expiry controlled by `JWT_EXPIRES_IN` (default `8h`, `src/config/env.ts:9`), so a stolen token remains valid until expiry and cannot be invalidated by the registry. This is an important boundary of the component: the registry can only decide whether a token is required, not whether it is still authorized beyond signature and expiry.

**Rule workflow**:
1. Anonymous client sends `POST /api/v1/auth/register` or `POST /api/v1/auth/login`.
2. `validate` parses the body schema and rejects malformed payloads with 400.
3. The controller delegates to `AuthService`.
4. On success, registration returns 201 with the public user; login returns 200 with an access token.
5. A request to `GET /api/v1/auth/me` without a valid Bearer token stops at `authenticate` with 401.
6. With a valid token, `req.user` is populated, the controller loads the user by `req.user.id` and returns 200.

---

### Business Rule: BR-05 - Router-Level Authentication for Customers, Products and Orders

**Overview**:
All endpoints in the customers, products and orders modules require a valid JWT, applied with `router.use(authenticate)` before any route declaration.

**Detailed description**:
The three modules each place `router.use(authenticate)` immediately after creating the router (`customer.routes.ts:14`, `product.routes.ts:14`, `order.routes.ts:14`). Because `router.use` without a path matches every request entering that mounted router, the guard covers all five CRUD routes in each module and any future route added after it. It also covers requests that do not match any route: an unauthenticated request to `/api/v1/orders/does-not-exist` is rejected with 401 rather than reaching the app-level 404 handler, because the error propagates out of the router to `errorMiddleware`. This is an observable rule with no documentation and no test.

Authentication is signature-based only. The middleware does not query the database, check whether the user still exists, check whether the account is active (there is no active flag on `User`) or compare token claims against stored state. Any token signed with the shared `JWT_SECRET` and not expired is accepted, regardless of which user it names. The registry does not distinguish between `ADMIN` and `OPERATOR` in these three modules; both roles receive identical access (BR-06 is the only role gate).

Placement order inside each router also defines precedence: `authenticate` runs before `validate`, so an unauthenticated request with an invalid payload receives 401, not 400. This ordering is uniform across all three modules and is a deliberate fail-closed choice, but it means clients cannot discover validation errors without first authenticating. The auth module's `/me` route follows the same precedence with its inline guard.

**Rule workflow**:
1. Request enters `/api/v1/customers`, `/api/v1/products` or `/api/v1/orders`.
2. The module router's `use(authenticate)` runs before any route match.
3. No `Authorization` header or a non-Bearer scheme produces 401 `UNAUTHORIZED`.
4. An invalid or expired JWT produces 401 `UNAUTHORIZED`.
5. A valid JWT hydrates `req.user` and the request proceeds to route matching, then validation, then the controller.
6. Unknown sub-paths under these prefixes with a valid token fall through to the app-level 404.

---

### Business Rule: BR-06 - Admin-Only User Lookup

**Overview**:
`GET /api/v1/users/:id` is the only endpoint in the application protected by role-based authorization, restricted to `ADMIN`.

**Detailed description**:
`src/modules/users/user.routes.ts:12-18` registers a single route with three middleware stages: `authenticate`, `requireRole('ADMIN')` and `validate({ params: idParamSchema })`, followed by `controller.getById`. `requireRole` (`src/middlewares/auth.middleware.ts:49-61`) reads `req.user.role` and compares it against the allowed list; a missing user yields 401 `UNAUTHORIZED`, and a role outside the list yields 403 `FORBIDDEN` with message "Insufficient permissions". Because `authenticate` runs first, an anonymous request is stopped at 401 before the role check executes.

The route exposes a user's public projection by identifier: `id`, `email`, `name`, `role`, `createdAt`, `updatedAt` (`src/modules/users/user.service.ts:7-14`, `42-51`). It does not expose `passwordHash`. A well-formed UUID that does not exist produces 404 `NOT_FOUND` from `UserService.getById`, while a non-UUID string is rejected earlier by `validate` with 400 `VALIDATION_ERROR`. This gives a clean three-way distinction between unauthenticated (401), authenticated-but-not-admin (403), malformed identifier (400) and missing resource (404).

No other route uses `requireRole`. The role vocabulary is defined in two places: `AuthUser['role']` as `'ADMIN' | 'OPERATOR'` in the middleware (`src/middlewares/auth.middleware.ts:9`) and the Prisma `Role` enum. The registry does not enforce any global role policy; it merely attaches the one guard. This asymmetry is the most significant authorization gap in the component (see section 10).

**Rule workflow**:
1. Request `GET /api/v1/users/:id`.
2. `authenticate` verifies the Bearer JWT; failure yields 401.
3. `requireRole('ADMIN')` checks `req.user.role`; `OPERATOR` yields 403 `FORBIDDEN`.
4. `validate` checks that `id` is a UUID; failure yields 400 `VALIDATION_ERROR`.
5. The controller calls `UserService.getById`; a missing user yields 404 `NOT_FOUND`.
6. A found user is returned as a public projection with status 200.

---

### Business Rule: BR-07 - Fail-Fast Zod Validation Before Business Logic

**Overview**:
Every endpoint that accepts input validates it with a Zod schema before the controller executes, returning a normalized 400 payload on failure.

**Detailed description**:
`validate(schemas)` (`src/middlewares/validate.middleware.ts:11-37`) accepts up to three schema slots: `body`, `query` and `params`. It parses each provided source in that order. Parsed bodies replace `req.body` with the schema output, which means Zod defaults and transforms are applied before the controller sees the data (for example `discountCents` defaulting to 0 and `active` transforming from the string `"true"`/`"false"` to a boolean). Parsed query values are merged into `req.query` with `Object.assign` rather than replacing the object, and parsed params replace `req.params`.

On failure, a `ZodError` is converted into `ValidationError('Validation failed', details)` where each detail is `{ path, message }` and `path` falls back to `(root)` for top-level issues. `errorMiddleware` serializes it as HTTP 400 with `error.code = 'VALIDATION_ERROR'` and `error.details` as an array (`src/middlewares/error.middleware.ts:15-24`). Non-Zod errors are forwarded unchanged, so a programming error inside a schema (for example a throwing transform) becomes a 500 rather than a 400. Schemas are not applied to `GET /auth/me` or to `GET /health`, which take no user input; header values (including `Authorization`) are never schema-validated.

Zod's default object behavior strips unknown keys instead of rejecting them. A request body containing extra properties therefore passes validation and the extra properties are silently discarded. Combined with shallow `.partial()` on update schemas (BR-12), this means the API is permissive about unknown input but strict about the shape of known input. No route configures `.strict()` or `.passthrough()`.

**Rule workflow**:
1. A route handler chain reaches a `validate({ ... })` middleware.
2. For each configured source (body, then query, then params), the schema parses the request value.
3. On success, parsed values are written back to the request and `next()` is called.
4. On `ZodError`, issues are mapped to `{ path, message }` and passed to the error middleware as `ValidationError`.
5. The client receives 400 with `VALIDATION_ERROR` and the details array; the controller is never invoked.

---

### Business Rule: BR-08 - UUID-Only Path Identifiers

**Overview**:
Every `:id` route parameter must be a UUID string; all other identifiers are rejected before any service call.

**Detailed description**:
Each module defines a param schema of the form `z.object({ id: z.string().uuid() })`: orders at `order.schemas.ts:4`, customers at `customer.schemas.ts:3-5`, products at `product.schemas.ts:3`, and users with an inline schema at `user.routes.ts:7`. The schema is attached through `validate({ params: ... })` on every parameterized route, including `DELETE`, `PATCH` and the nested `PATCH /orders/:id/status` (which validates params and body in one call at `order.routes.ts:19-23`).

The rule creates a clear error contract: malformed identifiers never reach Prisma, so a random string cannot trigger a database error or a 500. A syntactically valid but unknown UUID reaches the repository and produces 404 `NOT_FOUND` via the service's `getById` check or Prisma `P2025` translation. This distinction (400 for malformed, 404 for absent) is consistent across customers, products, orders and users. Note that Zod's `uuid()` accepts RFC 4122 versioned UUIDs, so some non-standard UUID-like strings are rejected even if the database column could store them.

The users module duplicates the schema instead of importing `userIdParamSchema` from `src/modules/users/user.schemas.ts:3-5`; the exported schema is currently unused. This duplication is a maintainability risk rather than a behavioral difference today, because both definitions are identical. It does, however, mean a future change to the shared schema would silently not apply to the route.

**Rule workflow**:
1. A parameterized request reaches its route.
2. `validate` parses `req.params.id` with the UUID schema.
3. A non-UUID value produces 400 `VALIDATION_ERROR` with `path = "id"`.
4. A UUID value is written to `req.params` and the controller receives it.
5. The service either returns the resource (200) or throws `NotFoundError` (404).

---

### Business Rule: BR-09 - Shared Pagination Contract and Module-Specific List Filters

**Overview**:
All three list endpoints share identical pagination rules and add module-specific filters; no endpoint accepts a sort parameter.

**Detailed description**:
Every list schema defines `page: z.coerce.number().int().min(1).default(1)` and `pageSize: z.coerce.number().int().min(1).max(100).default(20)` (`customer.schemas.ts:26-27`, `product.schemas.ts:17-18`, `order.schemas.ts:24-25`). Because query values arrive as strings, `z.coerce.number()` performs the conversion, and the defaults apply when the parameter is absent. Values below 1, non-integers and `pageSize > 100` are rejected with 400. The upper bound of 100 is the only protection against unbounded page sizes; there is no maximum page number and no cursor-based pagination.

Module-specific filters are additive. Customers accept `search`, a trimmed non-empty string (`customer.schemas.ts:28`). Products accept `search` and `active`, where `active` only accepts the literal strings `"true"` or `"false"` and is transformed to a boolean (`product.schemas.ts:20-23`). Orders accept `status` (the `OrderStatus` enum), `customerId` (UUID) and `from`/`to` (coerced dates) (`order.schemas.ts:26-29`). The parsed query is merged into `req.query`, and each controller casts it to its inferred query type before passing it to the service, which computes `skip = (page - 1) * pageSize` and delegates to the repository.

Two behavioral consequences follow from coercion and merging. First, repeated query parameters (for example `?status=PAID&status=PENDING`) arrive as arrays, which fail the single-value schemas and return 400 rather than being interpreted as a multi-select filter. Second, there is no validation that `from <= to` on orders; an inverted range is accepted by the schema and its meaning is left entirely to the repository query. Both behaviors are consistent with the schema definitions and outside the registry's control, but they form part of the published contract.

**Rule workflow**:
1. Client calls `GET /customers`, `GET /products` or `GET /orders`.
2. `authenticate` runs first (BR-05).
3. `validate({ query })` coerces and validates pagination and filters.
4. Missing values are defaulted (page 1, pageSize 20); invalid values return 400.
5. Parsed values are merged into `req.query`.
6. The controller forwards the query to the service, which returns `{ data, pagination }`.

---

### Business Rule: BR-10 - Order Creation Route Contract

**Overview**:
`POST /api/v1/orders` accepts a customer, at least one line item, an optional discount and optional notes, and records the authenticated user as the actor.

**Detailed description**:
`order.routes.ts:18` binds `authenticate` (via router-level use), `validate({ body: createOrderSchema })` and `controller.create`. The schema (`order.schemas.ts:6-16`) requires `customerId` as a UUID, `items` as an array with `min(1)` and the message "order must contain at least one item", each item having a UUID `productId` and a positive integer `quantity`, plus `discountCents` as a non-negative integer defaulting to 0 and `notes` up to 1000 characters. There is no maximum item count, no duplicate-product consolidation at the schema level and no check that `discountCents` is below the computed subtotal; those rules, where they exist, live in `OrderService`.

The controller requires `req.user` and passes `req.user.id` as the actor to `OrderService.create` (`src/modules/orders/order.controller.ts:28-36`), returning 201 with the created order. Integration tests confirm downstream behavior that the route exposes: server-computed `subtotalCents`, `totalCents`, `PENDING` initial status, one history entry, and an `ORD-\d{6}` order number (`tests/orders.test.ts:12-40`). A non-existent product yields 404 `NOT_FOUND` (`tests/orders.test.ts:42-57`), which reaches the client through the shared error middleware rather than through route-specific handling.

The route is not idempotent. There is no `Idempotency-Key` header handling, no request de-duplication and no unique client reference in the schema. A client retry after a timeout creates a second order and consumes a second order number. This is a documented consequence of the route design rather than a defect in the registry, but it is a property clients must account for.

**Rule workflow**:
1. Authenticated client sends `POST /api/v1/orders` with a JSON body.
2. `authenticate` validates the Bearer token.
3. `validate` enforces the creation schema and applies defaults.
4. The controller checks `req.user` and forwards body plus actor id to `OrderService.create`.
5. The service validates downstream rules (customer existence, products, totals, stock) inside a transaction.
6. Success returns 201 with the created order; domain errors propagate to `errorMiddleware`.

---

### Business Rule: BR-11 - Dedicated Order Status Transition Endpoint

**Overview**:
Orders expose status changes only through `PATCH /api/v1/orders/:id/status` with an enum target status and an optional reason; there is no generic order update endpoint.

**Detailed description**:
`order.routes.ts:19-23` registers the status route with `validate({ params: orderIdParamSchema, body: updateOrderStatusSchema })` and `controller.changeStatus`. The body schema (`order.schemas.ts:18-21`) requires `toStatus` as `z.nativeEnum(OrderStatus)` and allows an optional `reason` of up to 500 characters. The target status is therefore constrained to the Prisma enum at the HTTP boundary; an unknown status string returns 400 before any state-machine evaluation. The identifier is a UUID and is validated in the same middleware call.

Unlike customers and products, orders have no `PATCH /:id` route. The route table intentionally exposes only status mutation, which prevents clients from editing totals, items or customer references after creation. The controller passes both the target status and the actor id to `OrderService.changeStatus`, so audit history can record who performed the transition. The response is 200 with the updated order; invalid transitions are surfaced as 409 `INVALID_STATUS_TRANSITION` and insufficient stock as 422 `INSUFFICIENT_STOCK` by the domain layer and translated by the shared error middleware.

The registry does not encode the state machine itself. Allowed transitions, stock debit and replenishment, and deletability constraints are enforced in `OrderService` and `order.status.ts`. What the route guarantees is narrower but important: the only way to change status is a validated, authenticated, actor-attributed PATCH to a dedicated sub-resource path, and there is no route that bypasses that path. The nesting `/:id/status` also means the generic `GET /:id` route at line 17 cannot shadow it because the methods and path shapes differ.

**Rule workflow**:
1. Authenticated client sends `PATCH /api/v1/orders/:id/status`.
2. `authenticate` validates the token.
3. `validate` checks the UUID param and the `toStatus`/`reason` body.
4. The controller requires `req.user` and calls `changeStatus(id, body, req.user.id)`.
5. The service applies the state machine and stock side effects transactionally.
6. Success returns 200; invalid transition returns 409; insufficient stock returns 422.

---

### Business Rule: BR-12 - Shallow Partial Updates for Customers and Products

**Overview**:
`PATCH /customers/:id` and `PATCH /products/:id` accept partial payloads where every top-level field is optional, but nested objects must still satisfy their full schema.

**Detailed description**:
Both modules derive update schemas with `.partial()`: `updateCustomerSchema = createCustomerSchema.partial()` (`customer.schemas.ts:23`) and `updateProductSchema = createProductSchema.partial()` (`product.schemas.ts:14`). At the top level this makes every field optional, so a client may send only `phone`, only `priceCents`, or any subset. Zod applies defaults only when a key is absent from the input during a full parse; with `.partial()`, absent keys remain absent rather than being replaced by create-time defaults, which is what allows sparse updates to avoid overwriting unrelated columns.

The partiality is shallow. In the customer schema, `address` is a nested object with five required fields (`customer.schemas.ts:7-13`). `.partial()` makes the `address` property optional, but if a client supplies `address` it must supply `street`, `number`, `city`, a two-character `state` and a `zipCode` of 5 to 15 characters. There is no separate address patch schema, so partial address updates are impossible through this endpoint. Products have no nested objects, so partiality is fully effective there.

Two edge behaviors follow. First, an empty object `{}` passes validation for both update routes; the service receives an empty update and the endpoint does not reject a no-op request at the routing layer. Second, unknown keys are stripped (BR-07), so a typo such as `pricecent` is silently ignored and the request may succeed without changing the intended field. Both behaviors are consistent and testable but undocumented, and neither is covered by tests.

**Rule workflow**:
1. Authenticated client sends `PATCH /api/v1/customers/:id` or `/api/v1/products/:id`.
2. `authenticate` runs, then `validate` checks the UUID param and the partial body.
3. Missing top-level fields are accepted; supplied fields must satisfy their individual constraints.
4. A supplied customer `address` must contain all five address fields.
5. The controller calls the service with the sparse update.
6. The service merges only defined fields and returns the updated resource with status 200.

---

### Business Rule: BR-13 - Unmatched Route Fallback

**Overview**:
Any request that passes through the registry without matching a route and without an error returns 404 `NOT_FOUND` with the method and original URL in the message.

**Detailed description**:
After the `/api/v1` mount, `src/app.ts:69-71` registers a catch-all middleware `(req, _res, next) => next(new NotFoundError(\`Route ${req.method} ${req.originalUrl}\`))`. `NotFoundError` appends " not found" to the supplied resource (`src/shared/errors/http-errors.ts:27-31`), so the response body is `{ error: { code: 'NOT_FOUND', message: 'Route GET /api/v1/unknown not found' } }` with status 404. Because the message includes `req.originalUrl`, the full path including the `/api/v1` prefix and query string is echoed back. This is a deliberate diagnostic aid, but it also reflects arbitrary client input into the response body.

The fallback is not reached in three situations. First, if an earlier middleware errors (invalid token under an authenticated module, validation failure, controller error), the error path takes precedence and the fallback is skipped. Second, unauthenticated requests to unknown paths under `/customers`, `/products` or `/orders` are stopped by `router.use(authenticate)` with 401, so the fallback only sees authenticated requests to those prefixes. Third, body-parser failures occur before routing and are handled by the error middleware. There is no method-not-allowed handling: `PUT /api/v1/orders/:id` with a valid token is a 404, not a 405.

A related gap is visible in the error middleware: `express.json` raises `SyntaxError` for malformed JSON and a 413 error for bodies over 1 MB, both of which carry their own status codes, but `errorMiddleware` only branches on `AppError`, `ZodError` and two Prisma codes (`src/middlewares/error.middleware.ts:14-54`). Those body-parser errors therefore fall to the generic branch, are logged, and are returned as 500 `INTERNAL_SERVER_ERROR` (`src/middlewares/error.middleware.ts:56-64`). This contradicts the usual expectation of 400 for malformed JSON and 413 for oversized payloads.

**Rule workflow**:
1. Request enters the app and passes global middleware and the `/api/v1` registry.
2. If any middleware or handler calls `next(err)`, the error path runs instead of the fallback.
3. If no route matched and no error occurred, the fallback constructs `NotFoundError` with method and original URL.
4. `errorMiddleware` serializes it as 404 with code `NOT_FOUND`.
5. Requests under authenticated modules without a token never reach this step (401 instead).

---

### Business Rule: BR-14 - Client-Supplied Role on Public Registration

**Overview**:
The public registration endpoint accepts a `role` value from the request body, including `ADMIN`, and persists it without any authorization check.

**Detailed description**:
`registerSchema` (`src/modules/auth/auth.schemas.ts:3-8`) includes `role: z.enum(['ADMIN', 'OPERATOR']).default('OPERATOR')`. The route applies this schema without any authentication or role guard (`src/modules/auth/auth.routes.ts:10`). The parsed body is passed by `AuthController.register` to `AuthService.register` (`src/modules/auth/auth.service.ts:27-29`), which calls `UserService.createUser`, and that method writes `role: input.role` directly to the database (`src/modules/users/user.service.ts:27-32`). No layer between the HTTP boundary and Prisma restricts the value to `OPERATOR`.

The consequence is that any anonymous caller can create an `ADMIN` account by sending `"role": "ADMIN"` and then authenticate normally. The resulting JWT carries `role: 'ADMIN'` (`src/modules/auth/auth.service.ts:47-50`), which satisfies `requireRole('ADMIN')` on `GET /users/:id`. In the current codebase that is the only role-gated route, so the immediate blast radius is user enumeration; if additional role guards are added later without fixing registration, the escalation would extend to those routes automatically. This rule is a security defect rather than an intended business capability: there is no invitation flow, no admin approval and no domain rule that justifies honoring a client-chosen role at self-registration.

Confidence: high. The behavior is directly traceable from schema to SQL write. The intent is not documented anywhere in the repository; the only related documentation is the architectural report, which classifies it as a critical risk.

**Rule workflow**:
1. Anonymous client sends `POST /api/v1/auth/register` with `role: "ADMIN"`.
2. `validate` accepts the body and defaults the role to `OPERATOR` only if the field is absent.
3. The controller calls `AuthService.register`.
4. `AuthService` delegates to `UserService.createUser`.
5. `createUser` hashes the password and persists the supplied role.
6. The response returns 201 with the public user, including `role: "ADMIN"`; the client can then log in and receive an ADMIN JWT.

---

## 4. Component Structure

```
src/
├── routes/
│   └── index.ts                          # PRIMARY: Controllers contract + buildApiRouter; mounts 5 module routers
├── app.ts                                # Composition root: builds controllers, mounts registry at /api/v1, health, 404, errors
├── server.ts                             # Process entry point: listen, signal handling (outside registry, consumer of app)
├── middlewares/
│   ├── auth.middleware.ts                # authenticate (JWT Bearer), requireRole(...roles)
│   ├── validate.middleware.ts            # Zod body/query/params parsing -> ValidationError
│   ├── error.middleware.ts               # AppError / ZodError / Prisma P2002,P2025 -> JSON; masks others as 500
│   └── request-logger.middleware.ts      # X-Request-Id, duration, access log
├── modules/
│   ├── auth/
│   │   ├── auth.routes.ts                # /register (public), /login (public), /me (authenticate)
│   │   ├── auth.schemas.ts               # registerSchema (includes client-supplied role), loginSchema
│   │   ├── auth.controller.ts            # register/login/me handlers
│   │   └── auth.service.ts               # registration delegation, credential check, JWT signing
│   ├── users/
│   │   ├── user.routes.ts                # GET /:id with authenticate + requireRole('ADMIN'); inline UUID schema
│   │   ├── user.schemas.ts               # userIdParamSchema (unused by the route), createUserSchema
│   │   ├── user.controller.ts            # getById handler
│   │   └── user.service.ts               # createUser persistence incl. role, public projection
│   ├── customers/
│   │   ├── customer.routes.ts            # router.use(authenticate); 5 CRUD routes
│   │   ├── customer.schemas.ts           # create/partial update/list query/uuid param
│   │   ├── customer.controller.ts        # list/getById/create/update/delete
│   │   ├── customer.service.ts           # email uniqueness, pagination
│   │   └── customer.repository.ts
│   ├── products/
│   │   ├── product.routes.ts             # router.use(authenticate); 5 CRUD routes
│   │   ├── product.schemas.ts            # create/partial update/list query (active transform)/uuid param
│   │   ├── product.controller.ts
│   │   ├── product.service.ts            # SKU uniqueness, pagination
│   │   └── product.repository.ts
│   └── orders/
│       ├── order.routes.ts               # router.use(authenticate); list/get/create/status/delete
│       ├── order.schemas.ts              # create/status transition/list query/uuid param
│       ├── order.controller.ts           # actor-aware create and changeStatus
│       ├── order.service.ts              # totals, transactions, stock, history
│       ├── order.repository.ts
│       └── order.status.ts               # state machine and stock side-effect predicates
└── shared/
    ├── errors/                           # AppError taxonomy used by route guards and error middleware
    ├── http/response.ts                  # { data, pagination } envelope for list endpoints
    └── logger/index.ts                   # Pino logger used by middleware and services

tests/
├── setup.ts                              # DB connect, truncate tables before each test, disconnect
├── helpers/factories.ts                  # cached buildApp, user/customer/product fixtures, token helper
├── auth.test.ts                          # 7 integration tests: register/login/me
└── orders.test.ts                        # 8 integration tests: create/transition/list/delete
```

Boundary notes:

- `src/routes/index.ts` owns only the `Controllers` type and the five mount calls. It has no knowledge of services, repositories, Prisma or middleware.
- The five `*.routes.ts` files are part of the published routing surface and are analyzed as part of this component because they define the endpoint table, guards and validation bindings.
- `src/app.ts` is the only consumer of the registry; it owns the `/api/v1` prefix, `/health`, the 404 fallback and the error handler installation.
- `src/server.ts` is outside the component and does not reference the registry directly.

---

## 5. Dependency Analysis

### Internal Dependencies

```
src/app.ts:67
  -> buildApiRouter (src/routes/index.ts:21)
       -> Router (express, external)
       -> Controllers type (src/routes/index.ts:13)
            -> AuthController type        (src/modules/auth/auth.controller.ts:6)
            -> UserController type        (src/modules/users/user.controller.ts:4)
            -> CustomerController type    (src/modules/customers/customer.controller.ts:5)
            -> ProductController type     (src/modules/products/product.controller.ts:5)
            -> OrderController type       (src/modules/orders/order.controller.ts:6)
       -> buildAuthRouter (auth.routes.ts:7)
            -> validate (middlewares/validate.middleware.ts:11)
            -> authenticate (middlewares/auth.middleware.ts:27)
            -> loginSchema, registerSchema (auth.schemas.ts:3,10)
            -> AuthController type
       -> buildUserRouter (user.routes.ts:9)
            -> authenticate, requireRole (auth.middleware.ts:27,49)
            -> validate
            -> UserController type
            -> inline idParamSchema (zod, no shared schema)
       -> buildCustomerRouter (customer.routes.ts:12)
            -> authenticate
            -> validate
            -> customer schemas (create, idParam, listQuery, update)
            -> CustomerController type
       -> buildProductRouter (product.routes.ts:12)
            -> authenticate
            -> validate
            -> product schemas (create, listQuery, idParam, update)
            -> ProductController type
       -> buildOrderRouter (order.routes.ts:12)
            -> authenticate
            -> validate
            -> order schemas (create, listQuery, idParam, updateStatus)
            -> OrderController type
```

Runtime dependency chains:

```
Router registry -> module router -> middleware (authenticate/requireRole/validate) -> controller instance -> service -> repository -> Prisma -> MySQL
Router registry -> module router -> controller instance -> service -> shared error taxonomy (throw) -> errorMiddleware -> HTTP JSON
```

Compile-time-only dependencies (erased at runtime, declared with `import type`):

- `src/routes/index.ts:2-6` imports the five controller classes as types only.
- `src/modules/*/*.routes.ts` import their controller as a type only.
- `src/middlewares/validate.middleware.ts:2` imports Zod types and `ZodError` (value import for the class).

### External Dependencies

| Package | Version | Role in this component |
|---------|---------|------------------------|
| `express` | 4.21.1 | `Router`, `app.use` mounting, middleware chain, path matching, `req.params/query/body` |
| `zod` | 3.23.8 | Route schemas, coercion, defaults, transforms, `ZodError` |
| `jsonwebtoken` | 9.0.2 | `authenticate` token verification (HS256 via shared secret) |
| `@prisma/client` | 5.22.0 | Downstream only: `OrderStatus` enum imported by `order.schemas.ts:2`; Prisma error classes in error middleware |
| `uuid` | 11.0.3 | Indirect: request id generation in `requestLogger` |
| `pino` | 9.5.0 | Indirect: access/error logging through `src/shared/logger` |

No external HTTP APIs, message brokers, caches or third-party identity providers are called by this component. The registry has no database dependency of its own; persistence is reached only through controllers.

---

## 6. Afferent and Efferent Coupling

Method: afferent coupling (Ca) is the number of distinct internal source files that import the unit; efferent coupling (Ce) is the number of distinct internal source files the unit imports. External npm packages are excluded from the counts and listed separately in section 5. Type-only imports are counted, consistent with the project architecture report.

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| `src/routes/index.ts` (registry) | 1 (`src/app.ts`) | 10 (5 controller types + 5 router factories) | High |
| `buildAuthRouter` (`auth.routes.ts`) | 1 (`routes/index.ts`) | 4 (validate, authenticate, auth.schemas, AuthController type) | Medium |
| `buildUserRouter` (`user.routes.ts`) | 1 (`routes/index.ts`) | 3 (authenticate/requireRole, validate, UserController type) | Medium |
| `buildCustomerRouter` (`customer.routes.ts`) | 1 (`routes/index.ts`) | 4 (authenticate, validate, customer.schemas, CustomerController type) | Medium |
| `buildProductRouter` (`product.routes.ts`) | 1 (`routes/index.ts`) | 4 (authenticate, validate, product.schemas, ProductController type) | Medium |
| `buildOrderRouter` (`order.routes.ts`) | 1 (`routes/index.ts`) | 4 (authenticate, validate, order.schemas, OrderController type) | High |
| `authenticate` (middleware) | 5 (all five route files) | 2 (`config/env`, `shared/errors`) | High |
| `requireRole` (middleware) | 1 (`user.routes.ts`) | 2 (`config/env` via same file, `shared/errors`) | Medium |
| `validate` (middleware) | 5 (all five route files) | 1 (`shared/errors`) | High |
| `controllers.*` handlers | 1 registry (via module router) | 1-2 each (service, shared errors) | High (for orders), Medium (others) |

Interpretation:

- The registry is maximally unstable in Martin's sense: Ca = 1 and Ce = 10, so it has no responsibility to be stable and every module change flows through it. This is expected for an aggregation layer and is not a defect, but it means the file is edited whenever a module is added, renamed or remounted.
- The two cross-cutting middlewares (`authenticate`, `validate`) have the highest fan-in inside this component (5 route files each). They are the real contracts of the routing layer; any signature change breaks all five modules at once.
- Module routers are leaves with Ca = 1. Their criticality comes from the business surface they publish, not from coupling pressure. `buildOrderRouter` is rated High because it exposes state-changing, stock-affecting operations.
- There are no cycles at file or module level. The dependency direction is strictly `app -> routes -> module routes -> middlewares/schemas/controller types`, and module routers never import each other.

---

## 7. Endpoints

The registry publishes 19 endpoints under `/api/v1`. `GET /health` exists in `src/app.ts:62-64` and is outside this component.

| Endpoint | Method | Auth | Validation | Description |
|----------|--------|------|------------|-------------|
| `/api/v1/auth/register` | POST | Public | body `registerSchema` | Self-registration; returns 201 with public user; accepts `role` (BR-14) |
| `/api/v1/auth/login` | POST | Public | body `loginSchema` | Credential check; returns 200 with `{ user, tokens }` |
| `/api/v1/auth/me` | GET | Bearer | none | Returns the authenticated user's public profile |
| `/api/v1/users/{id}` | GET | Bearer + `ADMIN` | params UUID | Returns a user by id; 403 for non-admin |
| `/api/v1/customers` | GET | Bearer | query `listCustomersQuerySchema` | Paginated customer list with `search` |
| `/api/v1/customers/{id}` | GET | Bearer | params UUID | Returns a customer by id |
| `/api/v1/customers` | POST | Bearer | body `createCustomerSchema` | Creates a customer with full address; 201 |
| `/api/v1/customers/{id}` | PATCH | Bearer | params UUID + body `updateCustomerSchema` | Shallow partial update; complete `address` if supplied |
| `/api/v1/customers/{id}` | DELETE | Bearer | params UUID | Deletes a customer; 204 |
| `/api/v1/products` | GET | Bearer | query `listProductsQuerySchema` | Paginated product list with `search` and `active` |
| `/api/v1/products/{id}` | GET | Bearer | params UUID | Returns a product by id |
| `/api/v1/products` | POST | Bearer | body `createProductSchema` | Creates a product; 201 |
| `/api/v1/products/{id}` | PATCH | Bearer | params UUID + body `updateProductSchema` | Shallow partial update |
| `/api/v1/products/{id}` | DELETE | Bearer | params UUID | Deletes a product; 204 |
| `/api/v1/orders` | GET | Bearer | query `listOrdersQuerySchema` | Paginated order list; filters `status`, `customerId`, `from`, `to` |
| `/api/v1/orders/{id}` | GET | Bearer | params UUID | Returns an order by id |
| `/api/v1/orders` | POST | Bearer | body `createOrderSchema` | Creates an order for the authenticated actor; 201 |
| `/api/v1/orders/{id}/status` | PATCH | Bearer | params UUID + body `updateOrderStatusSchema` | State-machine transition with optional reason |
| `/api/v1/orders/{id}` | DELETE | Bearer | params UUID | Deletes an order when domain state allows; 204 or 409 |

Notes:

- There is no `PUT`, no bulk endpoint, no file upload, no GraphQL or gRPC surface. Everything is REST over HTTP/JSON.
- Orders have no generic `PATCH /orders/{id}`; only the status sub-resource is mutable.
- `POST /auth/register` and `POST /auth/login` are the only anonymous endpoints.
- Unmatched methods on existing paths return 404, not 405 (BR-13).
- Response envelope: success payloads are returned directly by controllers; errors always use `{ error: { code, message, details? } }` from the error middleware; list endpoints use `{ data, pagination }` from `src/shared/http/response.ts`.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| Express application (`src/app.ts`) | Internal consumer | Mounts the registry at `/api/v1` and provides global middleware | In-process function call | Express `Router` object | n/a (construction-time) |
| Module routers (five factories) | Internal composition | Publish module route tables under fixed prefixes | In-process router mount | Express `Router` | Router errors propagate to `errorMiddleware` |
| `authenticate` middleware | Internal middleware | JWT Bearer verification and `req.user` hydration | In-process | JWT claims `{ sub, email, role, iat, exp }` | 401 `UNAUTHORIZED` via `next(err)` |
| `requireRole` middleware | Internal middleware | Role gate for `GET /users/:id` | In-process | `req.user.role` | 401 when no user, 403 `FORBIDDEN` on role mismatch |
| `validate` middleware | Internal middleware | Zod parsing of body/query/params | In-process | Parsed JS values | 400 `VALIDATION_ERROR` with `details[]` |
| Controller handlers (five classes) | Internal adapter | Translate HTTP request into service calls and responses | In-process | JSON request/response | Controllers catch and forward errors with `next(err)` |
| Module services / Prisma repositories | Internal downstream | Persistence and domain logic behind every endpoint | In-process, Prisma query protocol | Prisma models | Service `AppError` subclasses; Prisma `P2002` -> 409, `P2025` -> 404 |
| MySQL 8.0 (via Prisma) | External infrastructure | Data storage for users, customers, products, orders, history | TCP (Prisma) | Relational rows | Not handled by the registry; connection errors surface as 500 |
| Pino logger (`src/shared/logger`) | Cross-cutting | Access logs and unhandled-error logs | In-process | Structured JSON logs | Logging failure is not handled |
| Error middleware | Cross-cutting terminal handler | Normalize all failures into JSON envelopes | In-process | `{ error: { code, message, details? } }` | Terminal; unhandled errors masked as 500 |

There are no outbound third-party API calls, no webhook emissions, no queue publishing and no cache integration in this component. The only network dependency is the database, reached indirectly through controllers.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Composite / aggregating Router | One root `Router` mounting five sub-routers | `src/routes/index.ts:21-31` | Build a single mountable routing tree from module routers |
| Factory function per module | `buildAuthRouter`, `buildUserRouter`, etc. receive a controller and return a `Router` | `src/modules/*/*.routes.ts` | Isolate route table construction; enable dependency injection of handlers |
| Composition root / manual DI | `buildControllers(prisma)` and `buildApp({ prisma })` instantiate the object graph | `src/app.ts:26-76` | Centralize wiring; no DI container |
| Contract object (typed registry input) | `Controllers` type with five required controller properties | `src/routes/index.ts:13-19` | Compile-time guarantee that all modules are wired |
| Layered architecture (per module) | routes -> controller -> service -> repository -> Prisma | all modules | Separate HTTP adaptation from domain and persistence |
| Chain of responsibility / middleware pipeline | `app.use` and route-level middleware arrays (`authenticate`, `requireRole`, `validate`, handler) | `src/app.ts:58-73`, all route files | Ordered cross-cutting concerns before handlers |
| Guard clause / fail-closed authentication | Router-level `use(authenticate)` before route declarations | customers/products/orders route files:14 | Reject unauthenticated traffic before matching routes |
| Schema-based input validation | `validate({ body, query, params })` with Zod schemas | `src/middlewares/validate.middleware.ts` and every route file | Uniform 400 contract, coercion and defaults |
| Centralized error translation | Single `ErrorRequestHandler` mapping `AppError`, `ZodError`, Prisma codes | `src/middlewares/error.middleware.ts:14-65` | Consistent error envelope and 500 masking |
| Sub-resource routing for commands | `PATCH /orders/:id/status` instead of generic update | `src/modules/orders/order.routes.ts:19-23` | Expose only the permitted mutation of an order |
| Strategy-like factory for guards | `requireRole(...roles)` returns a `RequestHandler` | `src/middlewares/auth.middleware.ts:49-61` | Parameterized authorization middleware |
| Type-only dependency inversion (partial) | `import type` controller contracts in route files | `src/routes/index.ts:2-6`, all route files | Erase controller imports at runtime; decouple compilation |
| Versioned namespace via mount | `app.use('/api/v1', ...)` rather than prefixing every route | `src/app.ts:67` | Central version boundary |

Architectural observations:

- The registry is a pure function of the controller bag; it has no state, no I/O and no configuration reads. This makes it trivially replaceable and easy to test in isolation, although no such test exists.
- Route files mix three concerns: path/method declaration, middleware ordering and schema binding. There is no declarative route manifest or OpenAPI document, so the route table can only be discovered by reading five files.
- The component is firmly inside a modular monolith: no process, network or transaction boundary exists between modules. The only cross-module runtime dependency in the whole system is `auth -> users`, and it is not visible at the routing layer.

---

## 10. Technical Debt & Risks

Security posture summary: the routing layer enforces authentication broadly (three modules at router level, one route inline) and authorization almost nowhere (one admin-only route). The public registration route accepts a privileged role. There is no rate limiting, no account lockout, no security headers middleware, no CORS configuration and no request schema for headers. Error masking for unexpected failures is correct (`src/middlewares/error.middleware.ts:62-64`), and password hashes are never returned by the user projection.

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Critical | `POST /api/v1/auth/register` | Public registration accepts `role: "ADMIN"` and persists it without checks (`auth.schemas.ts:7`, `auth.routes.ts:10`, `user.service.ts:31`) | Any anonymous caller can self-provision an administrator account and obtain an ADMIN JWT |
| High | Customers, products, orders routers | Only `authenticate` is applied; no `requireRole` and no ownership/tenant scoping (`customer.routes.ts:14`, `product.routes.ts:14`, `order.routes.ts:14`) | Any authenticated `OPERATOR` can read, mutate and delete all business data |
| High | Body parsing error path | Malformed JSON and payloads over 1 MB produce body-parser errors that the error middleware does not classify (`app.ts:59`, `error.middleware.ts:14-64`) | Clients receive 500 instead of 400/413; noisy error logs; misleading contract |
| High | Users, customers, products routes | No request-level tests at all; order `GET /:id` untested; no 403 test for the only role guard | Regressions in 12 of 19 endpoints would not be detected by the suite |
| Medium | Router-level authentication | `router.use(authenticate)` causes unknown sub-paths under protected prefixes to return 401 instead of 404 | Client debugging confusion; information about route existence is not leaked, but contract is inconsistent with `/health` and `/auth` |
| Medium | Auth routes | No rate limiting, lockout or backoff on `POST /auth/login` or `POST /auth/register` | Credential stuffing and account-enumeration attempts are unthrottled |
| Medium | `user.routes.ts:7` | Inline `idParamSchema` duplicates `userIdParamSchema` (`user.schemas.ts:3-5`), which is now unused | Divergent validation if one copy changes; dead export |
| Medium | Routing table maintenance | 19 endpoints are spread across five files with no OpenAPI/manifest; registry hardcodes module list | Contract drift, discovery cost, and two-file edits for every new module |
| Medium | `validate` middleware | Mutates `req.query` via `Object.assign` and reassigns `req.params`/`req.body` (`validate.middleware.ts:15-23`) | Coupled to Express 4 writable request properties; Express 5 upgrade risk for `req.query` |
| Low | 404 fallback | Echoes `req.originalUrl` (including query string) into the error message (`app.ts:70`) | Reflected client input in responses; log/response noise, minor XSS surface for naive consumers |
| Low | Error/status semantics | No 405 handling; unknown methods on existing paths become 404 | API consumers cannot distinguish wrong method from wrong path |
| Low | Route ordering | `GET /:id` is declared before any future static GET sub-paths in orders/customers/products | Latent shadowing if a static GET route (for example `/export`) is added later |
| Low | Update schemas | Empty `{}` passes PATCH validation; unknown keys are silently stripped (Zod default) | No-op updates succeed; typos are ignored without feedback |
| Low | Dependency type coupling | `Controllers` exposes concrete controller classes, not interfaces | Alternative implementations or test doubles require class substitution |
| Low | Testing infrastructure | No coverage provider or `coverage` script in `package.json`; tests require a live MySQL instance and share one fork | Coverage cannot be measured or enforced; suite is environment-dependent |

---

## 11. Test Coverage Analysis

Test infrastructure:

- Runner: Vitest 2.1.4 with `include: ['tests/**/*.test.ts']`, `setupFiles: ['tests/setup.ts']`, `fileParallelism: false`, single fork (`vitest.config.ts:3-18`).
- Setup: connects Prisma in `beforeAll`, truncates `OrderStatusHistory`, `OrderItem`, `Order`, `OrderNumberSequence`, `Product`, `Customer`, `User` before each test, disconnects in `afterAll` (`tests/setup.ts:4-20`).
- Helpers: `getTestApp()` caches a single `buildApp({ prisma })` instance, so every Supertest request goes through the real registry, middleware and error handler (`tests/helpers/factories.ts:8-15`). `createTestUser`, `loginAndGetToken`, `createTestCustomer`, `createTestProduct` and `bootstrapAuthenticatedUser` build fixtures directly through Prisma (`tests/helpers/factories.ts:17-83`).
- No mocks or stubs are used for HTTP, Express or Prisma. Tests are full-stack integration tests against a real database.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| API Router Registry (`src/routes/index.ts`) | 0 | 15 indirect (all requests traverse it) | Not measurable; no coverage instrumentation | No direct assertions on prefix mapping, mount order or controller contract |
| Auth routes (`auth.routes.ts`) | 0 | 7 (`tests/auth.test.ts`) | 3/3 endpoints exercised | Good: 201, duplicate 409, invalid body 400, login 200/401, `/me` 200/401; missing malformed-token, expired-token, Bearer-case and role-escalation tests |
| Users routes (`user.routes.ts`) | 0 | 0 | 0/1 endpoints | None: no 401/403/400/404/200 coverage for the only role-gated route |
| Customers routes (`customer.routes.ts`) | 0 | 0 | 0/5 endpoints | None |
| Products routes (`product.routes.ts`) | 0 | 0 | 0/5 endpoints | None |
| Orders routes (`order.routes.ts`) | 0 | 8 (`tests/orders.test.ts`) | 4/5 endpoints (`GET /:id` untested) | Strong domain assertions (totals, stock, history, status codes); missing auth, validation, pagination bounds and not-found tests |
| `validate` middleware (route-bound) | 0 | 1 indirect (invalid register body) | Partial: body only | No tests for query coercion, params UUID rejection or unknown-key stripping |
| `authenticate` middleware (route-bound) | 0 | 2 indirect (`/me` with and without token) | Partial | No invalid signature, expired token, wrong scheme or role-mismatch tests |
| 404 fallback (`app.ts:69-71`) | 0 | 0 | 0 | Untested; no 401-before-404 behavior test |
| `GET /health` (outside component) | 0 | 0 | 0 | Untested |

Static coverage indicators:

- Endpoint coverage: 7 of 19 published endpoints are exercised (36.8%).
- Route-file coverage: 2 of 5 module route files are exercised (`auth.routes.ts`, `order.routes.ts`). The other three files are loaded during `buildApp` but never receive a request in the suite.
- Test case count: 15 (`tests/auth.test.ts` 7, `tests/orders.test.ts` 8). All are integration tests through Supertest; there are zero unit tests for the registry, route factories or validation bindings.
- Branch-level coverage cannot be computed: `package.json` has no coverage script and no `@vitest/coverage-*` dependency, and no coverage output directory is configured or present.

Untested behaviors worth noting explicitly:

- Role-based authorization: no test asserts 403 for `GET /users/:id` with an `OPERATOR` token, and no test asserts 200 with an `ADMIN` token.
- Authentication edge cases: missing `Authorization`, non-Bearer scheme, empty token, tampered signature and expired token are all untested at the route level.
- Validation edge cases: non-UUID params, `pageSize=101`, `page=0`, repeated query parameters, unknown body keys and empty PATCH bodies are untested.
- Route fallback: no test asserts 404 for unknown `/api/v1` paths, and no test documents the 401-before-404 behavior for protected prefixes.
- Content-type handling: malformed JSON and oversized bodies are untested, so the 500-on-parse-error behavior is unverified by the suite.
- Health probe and the absence of 405 semantics are untested.

Test quality assessment: the existing tests are well-targeted for the auth and order domains. They assert status codes, error codes, computed totals, side effects on stock and history, and the exclusion of `passwordHash` from responses. They are deterministic in data isolation but share a single database and a single fork, so they require a live MySQL instance and are not hermetic. The primary gap is breadth at the routing layer: the component that publishes all 19 endpoints has no test that enumerates or asserts the endpoint table, and three of five module route files have no request-level coverage at all.

---

*End of report.*
