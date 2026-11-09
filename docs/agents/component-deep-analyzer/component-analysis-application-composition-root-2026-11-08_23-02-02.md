# Component Deep Analysis Report

**Component:** Application Composition Root
**Primary file:** `src/app.ts`
**Project:** `order-management-api` (Order Management System REST API)
**Analysis scope:** entire project root
**Excluded from analysis:** `node_modules`, `.git`, `dist`, `coverage`
**Analyzed revision:** `e7f6311` (working tree)
**Analysis date:** 2026-11-08 23:02:02

---

## 1. Executive Summary

The Application Composition Root is the single place in the codebase where the runtime object graph is constructed and where the HTTP application is assembled. `src/app.ts` exports three things: the `AppDependencies` type (the only externally supplied dependency is a `PrismaClient`), `buildControllers()` (which instantiates every repository, service and controller for the five feature modules, plus the two service-level objects required by authentication), and `buildApp()` (which creates an Express application, installs the global middleware stack, registers `/health`, mounts the versioned API router, adds a catch-all 404 handler and finally installs the centralized error middleware).

Architecturally the component is a **composition root combined with an application factory**. It has the highest efferent coupling in the system (18 internal files) and the lowest practical change isolation: every new module, middleware, route group or shared contract must be wired here. It is intentionally free of business logic; all business rules live in the modules and middlewares it composes. Because `buildApp()` neither reads configuration nor calls `listen`, it is embeddable, which is exactly how the test harness uses it (`tests/helpers/factories.ts:10-15`).

Key findings:

- **Single wiring point with 15 runtime objects and 17 constructor edges.** `buildControllers()` creates 4 repositories, 5 services and 5 controllers around one shared `PrismaClient` (`src/app.ts:26-53`). There are no interfaces or ports; dependencies are concrete classes.
- **Middleware order is a de facto contract.** `x-powered-by` is disabled, then `express.json({ limit: '1mb' })`, then `requestLogger`, then routes, then the 404 catch-all, then `errorMiddleware` (`src/app.ts:58-73`). The order is load-bearing: the 404 handler must be after the router and the error handler must be last.
- **The component itself enforces no authentication or authorization.** It mounts module routers that apply `authenticate` and `requireRole` internally. `/health` and `/auth/register`/`/auth/login` are therefore public by construction (`src/app.ts:62-67`).
- **Partial dependency injection.** Only Prisma is injected. Configuration (`src/config/env.ts`), logging (`src/shared/logger/index.ts`), the error taxonomy and the Prisma singleton are ambient module-scope singletons pulled in transitively. Importing `src/app.ts` therefore triggers environment validation at import time, which is an implicit precondition for both production and tests.
- **Error handling is centralized but not complete.** `AppError`, `ZodError`, Prisma `P2002` and `P2025` are mapped; body-parser failures (malformed JSON, payload too large) are not, and fall through to a masked 500 response.
- **No direct test coverage of the composition root's own behaviors.** All 17 integration tests traverse the app built here, but `/health`, the catch-all 404, `x-powered-by` removal, the 1 MB limit, `X-Request-Id` propagation and the 500 branch have no direct assertions. No coverage provider is configured.

---

## 2. Data Flow Analysis

### 2.1 Successful request through the composed pipeline

```
1. HTTP request reaches the Express instance created by buildApp()            (src/app.ts:55-56)
2. x-powered-by response header suppressed                                  (src/app.ts:58)
3. express.json parses a JSON body up to 1 MB                               (src/app.ts:59)
4. requestLogger assigns or forwards X-Request-Id, sets req.id,
   registers a finish listener for access logging                           (src/app.ts:60 -> middlewares/request-logger.middleware.ts:5-27)
5. Path resolution: /health short-circuits to the inline handler            (src/app.ts:62-64)
   or /api/v1/* is delegated to the API router                              (src/app.ts:67)
6. Module router applies its own chain:
   authenticate (JWT) -> validate (Zod) -> controller handler               (e.g. modules/orders/order.routes.ts:14-24)
7. Controller calls the injected service instance                           (e.g. modules/orders/order.controller.ts:28-36)
8. Service uses the injected repository and/or PrismaClient                 (e.g. modules/orders/order.service.ts:26-30, 50-124)
9. Repository executes a Prisma query against MySQL                         (config/database.ts:10 -> MySQL)
10. Controller serializes the result with res.status(...).json(...)         (e.g. modules/orders/order.controller.ts:31-32)
11. finish listener emits the http_request log with status and duration     (middlewares/request-logger.middleware.ts:12-25)
```

### 2.2 Error path: known application error

```
1. Service or middleware throws AppError (or subclass)                      (shared/errors/http-errors.ts:3-63)
2. Controller catch block calls next(err)                                   (e.g. modules/orders/order.controller.ts:33-35)
3. Express skips remaining non-error middleware
4. errorMiddleware matches instanceof AppError and serializes
   { error: { code, message, details? } } with err.statusCode               (middlewares/error.middleware.ts:15-24)
```

### 2.3 Error path: unmatched route

```
1. Request does not match /health or any /api/v1 route
2. Catch-all app.use handler creates
   new NotFoundError(`Route ${req.method} ${req.originalUrl}`)              (src/app.ts:69-71)
3. NotFoundError sets statusCode 404 and errorCode NOT_FOUND,
   message becomes "Route <METHOD> <url> not found"                          (shared/errors/http-errors.ts:27-31)
4. errorMiddleware serializes the AppError                                  (middlewares/error.middleware.ts:15-24)
```

### 2.4 Error path: unmapped failure

```
1. Non-AppError, non-ZodError, non-Prisma P2002/P2025 error reaches handler
2. errorMiddleware reads req.id (set by requestLogger) and logs
   { err, requestId, method, path } at error level                           (middlewares/error.middleware.ts:56-60)
3. Response is 500 { error: { code: 'INTERNAL_SERVER_ERROR',
   message: 'Internal server error' } }                                      (middlewares/error.middleware.ts:62-64)
```

### 2.5 Error path: malformed JSON or oversized body

```
1. express.json rejects the body before requestLogger runs                  (src/app.ts:59-60)
2. body-parser forwards a SyntaxError with status 400 (entity.parse.failed)
   or a 413 entity.too.large error
3. errorMiddleware does not inspect err.status/err.type                     (middlewares/error.middleware.ts:14-64)
4. Error is logged as unhandled and returned as 500 INTERNAL_SERVER_ERROR
   with no X-Request-Id header and no http_request access log
```

### 2.6 Startup and shutdown context (consumer of the composition root)

```
1. src/server.ts imports buildApp, env, prisma singleton                    (src/server.ts:1-4)
2. buildApp({ prisma }) constructs controllers and the Express app          (src/server.ts:7; src/app.ts:55-75)
3. app.listen(env.PORT) binds the process                                   (src/server.ts:9-11)
4. SIGINT/SIGTERM -> server.close() started, prisma.$disconnect() awaited,
   process.exit(0)                                                          (src/server.ts:13-21)
```

---

## 3. Business Rules & Logic

### 3.1 Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location |
|---------|-----------|------------------|----------|
| BR-01 | Composition / Lifecycle | Each `buildApp()` invocation produces a new Express application instance; no global app is exported | `src/app.ts:55-75` |
| BR-02 | Dependency Injection | The only injected dependency is `PrismaClient`; all controllers/repositories/services are constructed inside `buildControllers()` | `src/app.ts:22-24`, `src/app.ts:26-53` |
| BR-03 | Configuration Precondition | Importing the composition root requires a valid environment (`DATABASE_URL`, `JWT_SECRET` >= 16 chars), enforced transitively by `env.ts` | `src/app.ts:3-19` -> `src/config/env.ts:3-27` |
| BR-04 | Wiring Topology | Fixed construction order repository -> service -> controller for all modules; Auth composes across Users module objects | `src/app.ts:27-44` |
| BR-05 | Pipeline Order | Global middleware order is fixed: disable header, body parser, request logger, routes, 404, error handler | `src/app.ts:58-73` |
| BR-06 | Request Body Limit | JSON bodies only, maximum size 1 MB; no other body parser is installed | `src/app.ts:59` |
| BR-07 | Observability | Every request that reaches the logger receives an `X-Request-Id` (client-supplied or generated) echoed in the response and attached to `req.id` | `src/app.ts:60`; `src/middlewares/request-logger.middleware.ts:5-27` |
| BR-08 | Liveness Contract | `GET /health` returns `200 {"status":"ok"}` without authentication and without dependency checks | `src/app.ts:62-64` |
| BR-09 | API Namespace | All functional endpoints are mounted under the single prefix `/api/v1`; five sub-routers registered in a fixed order | `src/app.ts:67`; `src/routes/index.ts:21-31` |
| BR-10 | Not Found Semantics | Any unmatched request becomes a `NotFoundError` whose message embeds method and original URL, producing 404 `NOT_FOUND` | `src/app.ts:69-71`; `src/shared/errors/http-errors.ts:27-31` |
| BR-11 | Error Serialization | Exactly one terminal error handler; error body shape is always `{ error: { code, message, details? } }` | `src/app.ts:73`; `src/middlewares/error.middleware.ts:14-65` |
| BR-12 | Authorization Delegation | The composition root applies no authentication/authorization; each module router owns its guards | `src/app.ts:67`; `src/modules/*/*.routes.ts` |
| BR-13 | Assembly vs Execution | `buildApp()` never calls `listen`, never connects/disconnects Prisma and never reads `PORT`; process binding is the consumer's responsibility | `src/app.ts:55-75` vs `src/server.ts:6-22` |

### 3.2 Detailed breakdown of the business rules

---

### Business Rule: BR-01 Per-invocation application instance (factory semantics)

**Overview**:
`buildApp()` is a pure assembly function that returns a freshly constructed `Express` application. No application instance is created at module scope and none is exported as a singleton.

**Detailed description**:
The module exports only functions and a type; the `express()` call happens inside `buildApp()` at line 56. Every call therefore produces an independent Express instance with its own middleware stack, its own router tree and its own controller instances. This is the property that allows the test suite to build an application without starting a network listener, and it is the property that would allow a future second process (for example a webhook worker exposing an admin surface) to assemble a different HTTP surface from the same composition code.

The rule is not merely stylistic: because controllers are created inside `buildControllers()` (called at line 66, inside `buildApp()`), a second `buildApp()` call also produces a second set of controller/service/repository objects. Only the injected `PrismaClient` is shared, because the caller provides it. This gives instance isolation for all stateless objects and shared state only where state is meaningful (the database client and its connection pool).

There is one caveat in how the rule is actually used: the test harness calls `buildApp()` exactly once and memoizes the result in a module-level `cachedApp` variable (`tests/helpers/factories.ts:8-15`). The factory supports multiple instances, but the project's own tests do not exercise that capability, so no test proves that two independently built apps are isolated. Production calls the factory once (`src/server.ts:7`).

**Rule workflow**:

```
caller (server.ts | tests/helpers/factories.ts)
  -> buildApp({ prisma })
     -> const app = express()
     -> install middleware, health route
     -> buildControllers(prisma)  // new controllers/services/repositories
     -> mount router, 404, error handler
     -> return app
  -> caller either app.listen(port) or hands app to Supertest
```

---

### Business Rule: BR-02 Single explicit dependency: PrismaClient

**Overview**:
The composition root accepts exactly one dependency, `AppDependencies = { prisma: PrismaClient }`, and derives the entire object graph from it.

**Detailed description**:
`AppDependencies` is declared at `src/app.ts:22-24` and is the only injection seam in the application. `buildControllers(prisma)` receives the client, passes it to the four repositories (`UserRepository`, `CustomerRepository`, `ProductRepository`, `OrderRepository`) and additionally passes it directly to `OrderService`, which needs an interactive transaction client for multi-table writes. No other dependency is injectable: configuration, logger, error classes, schemas and the Prisma singleton for tests are imported directly.

This design means the object graph is closed and deterministic given one client, but it also means the composition root cannot be exercised with test doubles for services or repositories without editing production code. There are no repository interfaces, no ports and no service locator; each constructor parameter is a concrete class. The rule is therefore "constructor injection with concrete types and a single root dependency", which is a deliberate simplification rather than a full dependency-inversion setup.

The practical consequence is visible in the coupling profile: `buildControllers()` is the only place where the 15 runtime objects are ordered, and it encodes knowledge that belongs to each module (for example that `OrderService` needs the raw `PrismaClient`). Any change to a constructor signature in any module forces a change here, which is the classic composition-root trade-off.

**Rule workflow**:

```
buildApp(deps: AppDependencies)
  -> buildControllers(deps.prisma)
       repositories  <- prisma
       services      <- repositories (+ prisma for OrderService)
       controllers   <- services (+ UserService for AuthController)
  -> router <- controllers
```

---

### Business Rule: BR-03 Environment validity is an import-time precondition

**Overview**:
The composition root cannot be imported unless the process environment satisfies the Zod schema in `src/config/env.ts`. This is an implicit but strong rule: `DATABASE_URL` must be non-empty and `JWT_SECRET` must be at least 16 characters, otherwise the process exits.

**Detailed description**:
`src/app.ts` does not reference `env` directly, but it imports `requestLogger`, which imports the Pino logger, which imports `env` (`src/middlewares/request-logger.middleware.ts:3` -> `src/shared/logger/index.ts:2` -> `src/config/env.ts:27`). It also imports the error middleware (same logger path) and the router registry, whose module routers import the auth middleware, which imports `env` again. Because ES module evaluation is eager, importing `src/app.ts` executes `loadEnv()` and, on failure, prints the invalid fields and calls `process.exit(1)` (`src/config/env.ts:14-25`).

The rule has two effects. First, startup is fail-fast: a misconfigured deployment never reaches `app.listen`. Second, any tool that imports the composition root inherits the requirement. The test harness imports `buildApp` from `tests/helpers/factories.ts:4`, so running the suite requires `DATABASE_URL` and a `JWT_SECRET` of at least 16 characters to be present in the test process environment, even for tests that never touch authentication. `vitest.config.ts` does not load `.env` itself; the variables must come from the shell or from an external loader.

This is also why the composition root is not fully pure despite being a factory: it is deterministic with respect to its argument, but its import has a side effect (environment validation and possible process termination). The rule is documented here as implicit, with high confidence, because it is not stated in `src/app.ts` but follows necessarily from the import graph and from `env.ts` behavior.

**Rule workflow**:

```
import src/app.ts
  -> import middlewares/request-logger -> shared/logger -> config/env
     -> envSchema.safeParse(process.env)
        success -> export typed env
        failure -> console.error(invalid fields) -> process.exit(1)
  -> module body of app.ts evaluates (function declarations only)
```

---

### Business Rule: BR-04 Canonical wiring topology per module

**Overview**:
Every module is wired in the same direction, repository -> service -> controller, with two documented deviations: `OrderService` also receives the raw `PrismaClient`, and the Auth module composes objects from the Users module.

**Detailed description**:
The pattern is repeated four times: `UserRepository(prisma) -> UserService(userRepository) -> UserController(userService)` (lines 27-29), `CustomerRepository -> CustomerService -> CustomerController` (34-36), `ProductRepository -> ProductService -> ProductController` (38-40) and `OrderRepository -> OrderService(orderRepository, prisma) -> OrderController` (42-44). The Auth module is the exception: `AuthService(userRepository, userService)` and `AuthController(authService, userService)` (31-32). Auth owns no repository and no persistence of its own; it reuses the Users module's repository for credential lookup and the Users module's service for registration and identity projection.

This rule encodes the layering contract observed throughout the codebase and is the reason the composition root can be read as an architecture diagram. It also fixes the sharing semantics: a single `UserService` instance is shared between `UserController` (via the users router) and two auth consumers, and a single `UserRepository` instance is shared between `UserService` and `AuthService`. Because services are stateless, sharing is safe; the rule nevertheless means that any future stateful service would become process-wide state shared across routes.

The deviation for `OrderService` is architecturally significant. By injecting `PrismaClient` directly, the composition root blesses direct table access from the Orders module (`customers`, `products`, `order_number_sequence`) inside transactions. The rule is therefore not "services only see their repository" but "services see their repository plus, where transactions require it, the ORM client". This is a boundary decision encoded at line 43 and is the root cause of the Orders module's coupling to the Customers and Products tables.

**Rule workflow**:

```
prisma
 ├─ UserRepository ──> UserService ──> UserController
 │                        │  └──────> AuthController
 │                        └────────> AuthService (also gets UserRepository)
 ├─ CustomerRepository ─> CustomerService ─> CustomerController
 ├─ ProductRepository ──> ProductService ──> ProductController
 └─ OrderRepository ────> OrderService ────> OrderController
                          (also gets prisma)
```

---

### Business Rule: BR-05 Fixed global middleware order

**Overview**:
The application pipeline is assembled in a mandatory order that determines security headers, body parsing, observability, routing and error handling semantics.

**Detailed description**:
`buildApp()` performs six registrations in sequence: `app.disable('x-powered-by')` (line 58), `app.use(express.json({ limit: '1mb' }))` (59), `app.use(requestLogger)` (60), `app.get('/health', ...)` (62-64), `app.use('/api/v1', buildApiRouter(controllers))` (67), the catch-all 404 handler (69-71) and `app.use(errorMiddleware)` (73). Express executes middleware in registration order, so the order defines behavior: the body is parsed before logging, health is resolved before the versioned router, the versioned router is resolved before the 404 handler, and the error handler is reachable only via `next(err)` or a thrown error.

The ordering has three observable consequences. First, because `express.json` precedes `requestLogger`, requests rejected by the body parser never receive a request ID, are never access-logged and are handled by the error middleware as unclassified errors (see BR-11 and section 10). Second, because `/health` is registered before the API router, it cannot be shadowed by any module route and remains outside authentication. Third, because the 404 handler is registered after the router and before the error handler, it converts unmatched paths into the same error contract as domain errors, so clients see one error envelope everywhere.

The error handler is correctly last. Express identifies error middleware by arity (four parameters), so `errorMiddleware` would not be invoked as a normal handler even if misplaced; however, it must still be registered after all routes to catch their `next(err)` calls. Any future middleware added after line 73 would not have its errors handled, which makes the position of the final line itself a rule.

**Rule workflow**:

```
request
 -> express.json (parse or reject)
 -> requestLogger (ID + timing)
 -> health handler (terminal) | /api/v1 router (terminal or next(err))
 -> 404 catch-all (terminal, next(NotFoundError))
 -> errorMiddleware (terminal for any forwarded error)
```

---

### Business Rule: BR-06 JSON-only body parsing with a 1 MB ceiling

**Overview**:
The application accepts JSON request bodies only, and rejects bodies larger than 1 MB at the parser level.

**Detailed description**:
Line 59 installs `express.json({ limit: '1mb' })` and no other body parser (`urlencoded`, `raw`, `text`, `multipart`) is registered. Controllers read `req.body` directly (for example `src/modules/orders/order.controller.ts:31` and `src/modules/auth/auth.controller.ts:14`), so any non-JSON content type arrives with an empty or undefined body and is subsequently rejected by Zod validation as a 400 `VALIDATION_ERROR`. The limit is hardcoded, not configurable through `src/config/env.ts`, so changing it requires a code change and redeploy.

The 1 MB ceiling is a coarse payload-protection control. It is the only request-size defense in the application; there is no per-route limit and no streaming validation. The limit applies uniformly to public and authenticated endpoints, including `/auth/register` and `/orders`. Because the parser runs before the logger, an oversized payload produces no access log entry and no `X-Request-Id` (see BR-05 and section 10).

The rule also defines the expected client contract: `Content-Type: application/json` is required for any endpoint with a body. Missing or different content types are not rejected with 415; they degrade into schema validation failures, which is a contract detail consumers must know.

**Rule workflow**:

```
request with body
 -> Content-Type not application/json -> body left unparsed
 -> body > 1mb -> parser error forwarded to error middleware
 -> body valid JSON -> req.body populated -> route-level Zod validation
```

---

### Business Rule: BR-07 Request correlation and access logging

**Overview**:
Every request that reaches the logger is assigned a correlation ID, echoed back to the client, stored on the request object, and used for one structured access log entry on response completion.

**Detailed description**:
`requestLogger` reads `x-request-id` from the incoming headers; if absent or not a string, it generates a UUID v4 (`src/middlewares/request-logger.middleware.ts:6`). It assigns the value to `req.id` (declared via module augmentation in `src/middlewares/auth.middleware.ts:12-17`) and sets the same value in the `X-Request-Id` response header (line 8). It records a monotonic start timestamp with `process.hrtime.bigint()`, registers a `finish` listener on the response, and on completion emits a Pino `info` entry with `requestId`, `method`, `path`, `statusCode`, `durationMs` and `userId` (lines 10-25).

The rule is trust-based: a client-supplied `X-Request-Id` is accepted without format validation, length checks or sanitization. This enables end-to-end correlation from callers that already have an ID, but it also means a caller can inject arbitrary strings into logs and into the response header. Because the value is embedded in JSON logs, structural log injection is limited, but log forging with misleading IDs is possible. No maximum length is enforced.

The same `req.id` is consumed by the error middleware to correlate error logs (`src/middlewares/error.middleware.ts:56-60`). That correlation only works for requests that passed the logger, which excludes body-parser failures due to the ordering described in BR-05. `userId` is read from `req.user?.id`, which is populated only after `authenticate` runs inside the module router, and because the log fires on `finish` it can include the authenticated identity for guarded routes.

**Rule workflow**:

```
request
 -> header x-request-id present? use it : uuidv4()
 -> req.id = id; res.setHeader('X-Request-Id', id)
 -> hrtime start; on response finish:
      logger.info({ requestId, method, path, statusCode, durationMs, userId }, 'http_request')
```

---

### Business Rule: BR-08 Health liveness contract

**Overview**:
`GET /health` is a public, static liveness endpoint that always returns HTTP 200 with `{ "status": "ok" }`.

**Detailed description**:
The handler is declared inline at `src/app.ts:62-64` and is the only route defined directly in the composition root. It performs no database call, no dependency check, no authentication and no versioning; it exists before the API router and is therefore reachable regardless of module-level guards. Its response has no envelope, no request metadata and no timestamp.

Because the handler is static, it reports process liveness only, not readiness. A process that can serve HTTP but cannot reach MySQL still returns 200. In the repository's deployment model there is no orchestrator or load balancer configured, so the endpoint currently serves as a manual smoke check and as a contract for future infrastructure; if a platform probe were added, it would treat a database outage as healthy. The architectural report already records this as a medium risk.

The rule also has a construction-order nuance: the health route is registered before `buildControllers()` runs at line 66. Since no requests can be served until `buildApp()` returns, this has no runtime effect, but it does mean a construction failure (for example a future constructor that validates external state) prevents the app from existing at all rather than producing a degraded app with a working health route.

**Rule workflow**:

```
GET /health
 -> no middleware beyond express.json/requestLogger
 -> 200 application/json { "status": "ok" }
 -> request logger emits http_request with statusCode 200
```

---

### Business Rule: BR-09 Single versioned API namespace with fixed router order

**Overview**:
All functional routes are exposed under `/api/v1`, produced by one router registry that mounts five module routers in the fixed order auth, users, customers, products, orders.

**Detailed description**:
`buildApp()` mounts a single router at `/api/v1` (line 67). That router is built by `buildApiRouter(controllers)` in `src/routes/index.ts:21-31`, which calls `router.use('/auth', ...)`, then `/users`, `/customers`, `/products`, `/orders`. The `Controllers` type in `src/routes/index.ts:13-19` is the contract between the composition root and the router registry: it names the five controller instances that must exist before routing can be assembled.

The version prefix is a literal string, not configuration, so introducing `/api/v2` would require code changes in the composition root and a parallel registry. Because mounting is prefix-based, the order of `router.use` calls only matters if two prefixes could match the same path; the five prefixes are disjoint, so the order is conventional rather than behavioral. The fixed order is still documented as a rule because it defines the canonical reading order of the API and because future overlapping prefixes (for example a `/webhooks` mount plus a catch-all) would make order significant.

The namespace rule also means the composition root owns the external URL contract at the coarsest level: changing the prefix changes every functional endpoint at once, while individual paths, methods, guards and validation remain inside the module routers. Tests hardcode `/api/v1/...` in every request (`tests/auth.test.ts:10`, `tests/orders.test.ts:20`), so a prefix change would break the entire suite at once, which is a useful early signal.

**Rule workflow**:

```
buildApp
 -> buildControllers -> Controllers { auth, users, customers, products, orders }
 -> buildApiRouter(controllers)
      /auth      -> buildAuthRouter(auth)
      /users     -> buildUserRouter(users)
      /customers -> buildCustomerRouter(customers)
      /products  -> buildProductRouter(products)
      /orders    -> buildOrderRouter(orders)
 -> app.use('/api/v1', router)
```

---

### Business Rule: BR-10 Catch-all not-found semantics

**Overview**:
Any request that reaches the end of the pipeline without a matching route is converted into a `NotFoundError` whose message contains the HTTP method and the original URL (including query string).

**Detailed description**:
Lines 69-71 register a normal middleware (three parameters) with `app.use`, so it matches every method and every path that was not terminated earlier. It calls `next(new NotFoundError(\`Route ${req.method} ${req.originalUrl}\`))`. `NotFoundError` appends " not found" to its argument (`src/shared/errors/http-errors.ts:27-31`), so the final message is `Route <METHOD> <url> not found`, delivered with status 404 and code `NOT_FOUND` by the error middleware.

The rule standardizes 404 behavior across unknown paths and unknown resources: a missing order thrown from a service and an unknown URL both produce the same `NOT_FOUND` envelope, differing only in message and optional details. This consistency is valuable for clients. It also means the 404 handler is method-agnostic: `HEAD`, `OPTIONS`, `PUT` and `DELETE` on unknown paths all receive a JSON 404. There is no 405 Method Not Allowed handling and no dedicated `OPTIONS` responder, so a CORS preflight to an existing path that lacks an `OPTIONS` route would be answered by this catch-all with 404 rather than a preflight response. No CORS middleware is installed, so preflight is not supported at all.

The message reflects part of the request (`req.method`, `req.originalUrl`) into the error response. Express/Node escapes JSON strings, so this is not a response-splitting vector, but the original URL may contain query parameters such as search terms or identifiers and will therefore be echoed back and, in the 500 branch, logged. That is a minor information-handling observation rather than a break of the rule.

**Rule workflow**:

```
request not matched by /health or /api/v1/*
 -> app.use handler
 -> next(new NotFoundError("Route " + method + " " + originalUrl))
 -> errorMiddleware -> 404 { error: { code: "NOT_FOUND", message: "Route ... not found" } }
```

---

### Business Rule: BR-11 Centralized error-to-HTTP translation

**Overview**:
Exactly one error middleware terminates every error path, with a fixed precedence: `AppError` subclasses, then `ZodError`, then Prisma `P2002`/`P2025`, then a masked 500.

**Detailed description**:
`errorMiddleware` is installed once, last (line 73), and implements four branches in order (`src/middlewares/error.middleware.ts:14-64`). `AppError` instances are serialized with their own `statusCode`, `errorCode`, `message` and optional `details`; this covers all domain errors such as `ConflictError` (409), `InvalidStatusTransitionError` (409 `INVALID_STATUS_TRANSITION`), `InsufficientStockError` (422 `INSUFFICIENT_STOCK`), `ValidationError` (400 `VALIDATION_ERROR`), `UnauthorizedError` (401) and `ForbiddenError` (403). `ZodError` is mapped to 400 `VALIDATION_ERROR` with a normalized `details` array of `{ path, message }`; this is a safety net, because the `validate` middleware normally converts Zod failures to `ValidationError` first (`src/middlewares/validate.middleware.ts:25-33`). Prisma `P2002` maps to 409 `CONFLICT` with the constraint target in the message, and `P2025` maps to 404 `NOT_FOUND`.

All other errors are logged with the request ID, method and path, then answered with a generic 500 that discloses nothing. This branch is the only place where unexpected errors are logged, and it logs the full error object, which may include stack traces and database details server-side. The response contract is uniform: every branch returns `{ error: { code, message, details? } }`, with `details` omitted when undefined.

Two gaps follow directly from the branch order and the absence of a `status`/`type` check. Body-parser errors are `SyntaxError` instances with `status = 400` (malformed JSON) or `status = 413` (payload too large); neither is an `AppError`, `ZodError` or the two handled Prisma classes, so both become 500 `INTERNAL_SERVER_ERROR` and are logged as unhandled. Other Prisma errors such as `P2003` foreign key violations are also unmapped and become 500. The rule is therefore accurate for the documented classes and incomplete for transport-level failures, which is recorded in section 10.

**Rule workflow**:

```
err
 -> instanceof AppError            -> status=err.statusCode, code=err.errorCode, details?
 -> instanceof ZodError            -> 400 VALIDATION_ERROR, details=[{path,message}]
 -> Prisma P2002                   -> 409 CONFLICT (target names)
 -> Prisma P2025                   -> 404 NOT_FOUND
 -> otherwise                      -> log(err, requestId, method, path) -> 500 INTERNAL_SERVER_ERROR
```

---

### Business Rule: BR-12 Authorization is delegated to module routers

**Overview**:
The composition root installs no authentication or authorization middleware. Guards are applied by each module router, which makes public and protected surfaces a property of the routing layer rather than of the app assembly.

**Detailed description**:
`buildApp()` registers only `express.json`, `requestLogger`, `/health`, the versioned router, the 404 handler and the error handler. `authenticate` and `requireRole` are imported by module routers, not by `src/app.ts`. As a result, `POST /api/v1/auth/register`, `POST /api/v1/auth/login` and `GET /health` are public, `GET /api/v1/auth/me` requires a valid JWT but no role, `GET /api/v1/users/:id` requires `ADMIN`, and the customer, product and order routers apply `authenticate` to all their routes via `router.use(authenticate)` without any role restriction.

This delegation keeps the composition root thin and lets modules own their security posture, but it also means there is no global default-deny. A new router mounted at line 67 of the registry inherits nothing: if its author forgets `authenticate`, its endpoints are public. The registry in `src/routes/index.ts` is the only place where all five mounts are visible together, so a reviewer must inspect five files to conclude which endpoints are protected. The rule is therefore "authentication is opt-in per router", with the composition root providing no safety net.

The security consequence recorded elsewhere in the project is amplified by this rule: because no global guard exists and no tenant scoping is applied inside services, any authenticated actor can reach all customer, product and order data once a router is guarded. The composition root cannot detect such a mistake, and no test asserts the authorization matrix at the app level.

**Rule workflow**:

```
app (no global auth)
 -> /health                         public
 -> /api/v1/auth/register|login     public (router-level validate only)
 -> /api/v1/auth/me                 authenticate
 -> /api/v1/users/*                 authenticate -> requireRole('ADMIN') -> validate
 -> /api/v1/customers/*             authenticate -> validate
 -> /api/v1/products/*              authenticate -> validate
 -> /api/v1/orders/*                authenticate -> validate
```

---

### Business Rule: BR-13 Assembly is separated from execution

**Overview**:
`buildApp()` builds an application but never binds a port, never connects or disconnects the database and never reads runtime configuration such as `PORT`; those responsibilities belong to the process entry point and to the test harness.

**Detailed description**:
The function's only statements are `express()`, middleware/route registration, controller construction and `return app`. `listen` appears only in `src/server.ts:9`. Prisma connection lifecycle appears in `src/server.ts:16` (`$disconnect` on shutdown) and in `tests/setup.ts:4-20` (`$connect`/`$disconnect`/truncate), not in the composition root. The environment object is not referenced in `src/app.ts` at all; `PORT` is read in `src/server.ts:9` and `NODE_ENV`/`LOG_LEVEL` are read inside their own modules.

This separation is what makes the application testable in-process: Supertest receives the Express instance and dispatches requests without a socket, which is how all 17 integration tests run. It also means the composition root is reusable across runtimes (an HTTP server today, potentially a worker-hosted admin server later) without change.

The boundary is not absolute, as noted in BR-03: although `PORT` is not read, importing the module still executes `env.ts` through transitive imports. So the rule is precisely "no port binding and no database lifecycle management", not "no configuration dependency". A future refactor that made `buildApp` take `{ prisma, logger, config }` would complete the separation, but that is an observation about the current design, not a change made by this analysis.

**Rule workflow**:

```
buildApp  : pure assembly (returns Express, no I/O, no listen)
server.ts : buildApp -> listen(env.PORT) -> on signal: close + disconnect + exit
tests     : buildApp -> cache -> Supertest requests; setup.ts owns connect/truncate/disconnect
```

---

## 4. Component Structure

### 4.1 Boundary and internal organization

The component boundary used in this report is `src/app.ts` only. `src/server.ts` is treated as its consumer (process bootstrap), `src/routes/index.ts` as the mounted routing registry, and the feature modules as composed collaborators. This matches the component inventory of the architectural report, where "Application Composition Root" and "Server Bootstrap" are separate components.

```
src/
├── app.ts                                  # COMPONENT UNDER ANALYSIS (76 lines)
│   ├── AppDependencies type                # lines 22-24  injected contract: { prisma }
│   ├── buildControllers(prisma)            # lines 26-53  object graph factory
│   │   ├── userRepository/Service/Controller      # 27-29
│   │   ├── authService/authController             # 31-32 (cross-module)
│   │   ├── customerRepository/Service/Controller  # 34-36
│   │   ├── productRepository/Service/Controller   # 38-40
│   │   ├── orderRepository/Service/Controller     # 42-44
│   │   └── return Controllers map                 # 46-52
│   └── buildApp(deps)                      # lines 55-75  Express assembly
│       ├── app.disable('x-powered-by')     # 58
│       ├── express.json({ limit: '1mb' })  # 59
│       ├── requestLogger                   # 60
│       ├── GET /health                     # 62-64
│       ├── buildControllers(deps.prisma)   # 66
│       ├── use('/api/v1', buildApiRouter)  # 67
│       ├── catch-all NotFoundError         # 69-71
│       └── errorMiddleware                 # 73
│
├── server.ts                               # CONSUMER: listen, signals, prisma disconnect
├── routes/
│   └── index.ts                            # Controllers type + buildApiRouter (5 mounts)
├── config/
│   ├── env.ts                              # transitive import-time precondition
│   └── database.ts                         # PrismaClient singleton used by consumers
├── middlewares/
│   ├── request-logger.middleware.ts        # X-Request-Id + access log
│   ├── error.middleware.ts                 # terminal error translation
│   ├── auth.middleware.ts                  # used by module routers (not app.ts)
│   └── validate.middleware.ts              # used by module routers (not app.ts)
├── modules/
│   ├── auth/        (controller, service, routes, schemas)
│   ├── users/       (controller, service, repository, routes, schemas)
│   ├── customers/   (controller, service, repository, routes, schemas)
│   ├── products/    (controller, service, repository, routes, schemas)
│   └── orders/      (controller, service, repository, routes, schemas, status)
└── shared/
    ├── errors/      (AppError, 9 subclasses, barrel)  # NotFoundError used at app.ts:20
    ├── http/response.ts
    └── logger/index.ts
```

### 4.2 Structural metrics

| Metric | Value | Note |
|--------|-------|------|
| Physical lines | 76 | `src/app.ts` |
| Exported symbols | 3 | `AppDependencies`, `buildControllers`, `buildApp` |
| Internal imports | 18 files | Highest efferent coupling in the system |
| External imports | 2 packages | `express` (value), `@prisma/client` (type only) |
| Runtime objects created | 14 + 1 client | 4 repositories, 5 services, 5 controllers |
| Constructor edges | 17 | See BR-04 graph |
| Inline route handlers | 1 | `GET /health` |
| Mounted routers | 1 at `/api/v1` | Aggregating 5 module routers |
| Global middlewares installed | 2 + 2 terminal | `express.json`, `requestLogger`; 404 and error handler |
| Authentication middleware in this file | 0 | Delegated (BR-12) |
| Conditional branches in this file | 0 | No business logic, no `if`; linear assembly |

---

## 5. Dependency Analysis

### 5.1 Internal dependency graph (outgoing from the component)

```
src/server.ts ──(1)──> src/app.ts
tests/helpers/factories.ts ──(1)──> src/app.ts

src/app.ts
 ├── middlewares/request-logger.middleware.ts ─> shared/logger ─> config/env
 ├── middlewares/error.middleware.ts ─> shared/errors, shared/logger, zod, @prisma/client
 ├── modules/users/user.repository.ts ─> @prisma/client
 ├── modules/users/user.service.ts ─> user.repository, shared/errors, bcrypt
 ├── modules/users/user.controller.ts ─> user.service
 ├── modules/auth/auth.service.ts ─> config/env, shared/errors, user.repository, user.service
 ├── modules/auth/auth.controller.ts ─> auth.service, user.service, shared/errors
 ├── modules/customers/customer.repository.ts ─> @prisma/client
 ├── modules/customers/customer.service.ts ─> customer.repository, shared/errors, shared/http
 ├── modules/customers/customer.controller.ts ─> customer.service
 ├── modules/products/product.repository.ts ─> @prisma/client
 ├── modules/products/product.service.ts ─> product.repository, shared/errors, shared/http
 ├── modules/products/product.controller.ts ─> product.service
 ├── modules/orders/order.repository.ts ─> @prisma/client
 ├── modules/orders/order.service.ts ─> order.repository, order.schemas, shared/errors,
 │                                       order.status, shared/http, @prisma/client
 ├── modules/orders/order.controller.ts ─> order.service, order.schemas, shared/errors
 ├── routes/index.ts ─> five *.routes.ts files ─> middlewares/auth, middlewares/validate,
 │                                                schemas, controllers
 └── shared/errors/index.ts ─> app-error, http-errors
```

The graph is acyclic. The composition root sits at the top: it is imported by only one production file (`src/server.ts`) and by one test helper (`tests/helpers/factories.ts`), and it reaches every module. `src/routes/index.ts` is an intermediate aggregator that is imported only by `src/app.ts`.

### 5.2 Incoming dependencies (who depends on the component)

| Dependent | Kind | Usage | Impact if the component changes |
|-----------|------|-------|---------------------------------|
| `src/server.ts:1,7` | Production consumer | `buildApp({ prisma })`, then `listen` | Startup signature change breaks process bootstrap |
| `tests/helpers/factories.ts:4,10-15` | Test consumer | `buildApp({ prisma })` cached as `cachedApp` | All integration tests fail to construct the app |

No module imports the composition root, so the dependency direction is strictly inward-to-outward: modules never know how they are wired. This is the main architectural benefit of the component.

### 5.3 External dependencies

| Dependency | Version | Direct/Transitive | Role in this component |
|------------|---------|-------------------|------------------------|
| `express` | 4.21.1 | Direct (value import) | `express()` factory, `Express` type, `app.use/get/disable` |
| `@prisma/client` | 5.22.0 | Direct (type-only import) | `PrismaClient` type in `AppDependencies` and `buildControllers` |
| `zod` | 3.23.8 | Transitive (error middleware, routes) | Schema validation invoked by mounted routers; `ZodError` branch |
| `jsonwebtoken` | 9.0.2 | Transitive (auth middleware) | JWT verification for guarded module routes |
| `bcrypt` | 5.1.1 | Transitive (user/auth services) | Password hashing at registration/login |
| `pino` | 9.5.0 | Transitive (logger) | Access/error logging used by mounted middleware |
| `pino-pretty` | 11.3.0 | Transitive (logger, development only) | Dev log transport |
| `uuid` | 11.0.3 | Transitive (request logger) | `X-Request-Id` generation |
| `@prisma/client` runtime / MySQL 8.0 | 5.22.0 / 8.0 | Transitive (repositories, OrderService) | All data access reached through the graph |
| `supertest` | 7.0.0 | Test-only transitive | Exercises the returned `Express` instance in-process |

### 5.4 Dependency characteristics and risks

- **Concrete-class dependencies, no interfaces.** Every constructor parameter is a class or the Prisma client type; substitution requires a compatible real object. There are no ports/adapters.
- **One injected dependency, several ambient singletons.** `env`, `logger` and the error taxonomy are imported directly, so the composition root is only partially invertible.
- **Import-time side effects.** `env.ts` validation runs when `src/app.ts` is imported (BR-03); `PrismaClient` is instantiated when `src/config/database.ts` is imported by consumers and tests.
- **Type-only Prisma import.** `src/app.ts:2` uses `import type`, so the composition root does not itself create a database client; the client is always supplied by the caller. This is the key detail that keeps the factory testable.
- **No circular dependencies** were observed at file or module level.

---

## 6. Afferent and Efferent Coupling

Methodology: afferent coupling (Ca) counts distinct internal source files that import or reference the unit; efferent coupling (Ce) counts distinct internal source files the unit's file imports. External packages are excluded from the counts and reported in section 5.3. Test files are counted separately from production files because they are a distinct consumer class. Intra-file references are not counted as separate coupling.

| Component / Unit | Afferent Coupling (production / tests) | Efferent Coupling | Criticality | Rationale |
|------------------|----------------------------------------|-------------------|-------------|-----------|
| `buildApp` (function) | 1 / 1 (`src/server.ts`, `tests/helpers/factories.ts`) | 18 internal files | High | Single composition point; every request in production and test flows through it |
| `buildControllers` (function) | 1 / 0 (only `src/app.ts`) | 16 internal files | High | Owns the entire object graph; constructor changes ripple here |
| `AppDependencies` (type) | 1 / 0 | 1 external type | Medium | Public injection contract; used only inside `src/app.ts` |
| `Controllers` contract (`src/routes/index.ts:13-19`) | 1 / 0 (referenced by `src/app.ts`) | 5 controller type imports | Medium | Must stay in sync with the five controllers and the registry |
| `buildApiRouter` | 1 / 0 | 10 internal files | Medium | Imports the five routers and the five controller types |
| `UserController` | 2 / 0 (`app.ts`, `routes/index.ts` type import) | 2 internal (`user.service`, express types) | Low | Thin adapter; single route (`GET /users/:id`) |
| `AuthController` | 2 / 0 (`app.ts`, `routes/index.ts` type import) | 3 internal (`auth.service`, `user.service`, errors) | Medium | Public entry point; also depends on UserService directly |
| `CustomerController` | 2 / 0 (`app.ts`, `routes/index.ts` type import) | 2 internal (`customer.service`, schemas type) | Low | Thin CRUD adapter |
| `ProductController` | 2 / 0 (`app.ts`, `routes/index.ts` type import) | 2 internal (`product.service`, schemas type) | Low | Thin CRUD adapter |
| `OrderController` | 2 / 0 (`app.ts`, `routes/index.ts` type import) | 3 internal (`order.service`, schemas type, errors) | Medium | Most business-critical HTTP adapter |
| `UserService` | 3 / 0 (`app.ts`, `auth.service.ts`, `auth.controller.ts`) | 3 internal (`user.repository`, errors, schemas) | High | Shared between users and auth composition |
| `AuthService` | 2 / 0 (`app.ts`, `auth.controller.ts`) | 5 internal (`env`, errors, user.repository, user.service, schemas) | High | Cross-module dependency and token issuance |
| `CustomerService` | 1 / 0 | 4 internal (`customer.repository`, errors, schemas, http) | Medium | Own CRUD rules; not shared |
| `ProductService` | 1 / 0 | 4 internal (`product.repository`, errors, schemas, http) | Medium | Own CRUD rules; not shared |
| `OrderService` | 1 / 0 | 5 internal (`order.repository`, schemas, errors, order.status, http) | High | Injected with both repository and raw PrismaClient |
| `UserRepository` | 3 / 0 (`app.ts`, `user.service`, `auth.service`) | 1 external (`@prisma/client`) | Medium | Shared credentials/persistence access |
| `CustomerRepository` | 2 / 0 (`app.ts`, `customer.service`) | 1 external | Low | Data access only |
| `ProductRepository` | 2 / 0 (`app.ts`, `product.service`) | 1 external | Low | Data access only |
| `OrderRepository` | 2 / 0 (`app.ts`, `order.service`) | 1 external | Medium | Data access only |
| `requestLogger` (middleware) | 1 / 0 (`app.ts`) | 2 internal (`shared/logger`, `express` external) | High | Global observability; ordering-sensitive |
| `errorMiddleware` (middleware) | 1 / 0 (`app.ts`) | 3 internal (`shared/errors`, `shared/logger`) | High | Sole error-serialization path |
| `NotFoundError` (error class) | 5 / 0 (`app.ts`, four services; barrel consumers counted separately) | 1 internal (`app-error`) | High | Shared contract; used for both domain and routing 404s |
| `src/app.ts` (file level) | 1 / 1 | 18 internal files | High | Highest Ce in the system; single point of nervousness |

Interpretation:

- **High Ce, low Ca is the defining profile of a composition root.** `src/app.ts` is maximally unstable by design: it is easy to change and nothing depends on it except the entry point and the test helper. That instability is correct for a wiring module, but it also means it has no protection against accidental coupling growth.
- **The component's real criticality comes from centrality, not from business logic.** It contains zero conditionals and zero domain rules, yet a failure here prevents the entire API from existing.
- **`UserService` and `UserRepository` are the only intra-module units with Ca = 3, driven by the Auth composition.** `UserService` is shared by `UserController`, `AuthController` and `AuthService`, which makes it the most reused object in the graph after the Prisma client. Controllers sit at Ca = 2 only because both `src/app.ts` and `src/routes/index.ts` name their types.
- **`OrderService` has low Ca but the highest risk**, because the composition root injects the raw Prisma client into it, coupling the Orders domain to Customers, Products and the sequence table.
- **Test coupling is shallow.** Only `buildApp` is imported by tests; no test constructs a controller, service or repository directly, so all component-level behavior is validated through the composition root's public surface.

---

## 7. Endpoints

The composition root itself defines `GET /health` and mounts every functional endpoint through `/api/v1`. The table below lists the complete HTTP surface exposed by the composed application, with the owning router noted. Authorization reflects the guards applied by the module routers, not by `src/app.ts`.

| Endpoint | Method | Description | Authentication | Role | Owning router |
|----------|--------|-------------|----------------|------|---------------|
| `/health` | GET | Static liveness probe, returns `{ "status": "ok" }` | None | None | inline (`src/app.ts:62-64`) |
| `/api/v1/auth/register` | POST | Register a user; returns public user, 201 | None | None (role supplied by caller) | `src/modules/auth/auth.routes.ts:10` |
| `/api/v1/auth/login` | POST | Authenticate and return JWT access token | None | None | `src/modules/auth/auth.routes.ts:11` |
| `/api/v1/auth/me` | GET | Return the authenticated user's public profile | Bearer JWT | Any | `src/modules/auth/auth.routes.ts:12` |
| `/api/v1/users/:id` | GET | Fetch a user by UUID | Bearer JWT | `ADMIN` | `src/modules/users/user.routes.ts:12-18` |
| `/api/v1/customers` | GET | Paginated customer list with `search` | Bearer JWT | Any | `src/modules/customers/customer.routes.ts:16` |
| `/api/v1/customers/:id` | GET | Fetch customer by UUID | Bearer JWT | Any | `src/modules/customers/customer.routes.ts:17` |
| `/api/v1/customers` | POST | Create customer | Bearer JWT | Any | `src/modules/customers/customer.routes.ts:18` |
| `/api/v1/customers/:id` | PATCH | Partial update of customer | Bearer JWT | Any | `src/modules/customers/customer.routes.ts:19-23` |
| `/api/v1/customers/:id` | DELETE | Delete customer, 204 | Bearer JWT | Any | `src/modules/customers/customer.routes.ts:24` |
| `/api/v1/products` | GET | Paginated product list with `search` and `active` | Bearer JWT | Any | `src/modules/products/product.routes.ts:16` |
| `/api/v1/products/:id` | GET | Fetch product by UUID | Bearer JWT | Any | `src/modules/products/product.routes.ts:17` |
| `/api/v1/products` | POST | Create product | Bearer JWT | Any | `src/modules/products/product.routes.ts:18` |
| `/api/v1/products/:id` | PATCH | Partial update of product | Bearer JWT | Any | `src/modules/products/product.routes.ts:19-23` |
| `/api/v1/products/:id` | DELETE | Delete product, 204 | Bearer JWT | Any | `src/modules/products/product.routes.ts:24` |
| `/api/v1/orders` | GET | Paginated order list with status/customer/date filters | Bearer JWT | Any | `src/modules/orders/order.routes.ts:16` |
| `/api/v1/orders/:id` | GET | Fetch order with items, history and customer | Bearer JWT | Any | `src/modules/orders/order.routes.ts:17` |
| `/api/v1/orders` | POST | Create order; server computes totals | Bearer JWT | Any | `src/modules/orders/order.routes.ts:18` |
| `/api/v1/orders/:id/status` | PATCH | Transition order status, moving stock and appending history | Bearer JWT | Any | `src/modules/orders/order.routes.ts:19-23` |
| `/api/v1/orders/:id` | DELETE | Delete order if PENDING or CANCELLED, 204 | Bearer JWT | Any | `src/modules/orders/order.routes.ts:24` |

Notes:

- 20 handlers total: 1 inline liveness route and 19 module route handlers across 5 routers.
- Unmatched paths, including unsupported methods on known paths, return 404 `NOT_FOUND` via the catch-all (`src/app.ts:69-71`).
- No CORS, no `OPTIONS` responder, no 405 handling, no API documentation endpoint.
- Success payloads are mixed: lists return a `{ data, pagination }` envelope; single resources return the bare record; errors always return `{ error: { code, message, details? } }`.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol / Mechanism | Data Format | Error Handling |
|-------------|------|---------|----------------------|-------------|----------------|
| HTTP clients (API consumers) | Inbound interface | All functional traffic and health checks | HTTP/1.1 via Express 4 | JSON request/response | Centralized `errorMiddleware`; 404 catch-all; masked 500 |
| MySQL 8.0 via Prisma 5.22 | Outbound data store | Persistence for users, customers, products, orders, items, status history, order number sequence | TCP, Prisma query engine | Prisma models / SQL | Prisma `P2002`/`P2025` mapped; other Prisma errors become 500; transactions in `OrderService` |
| `PrismaClient` singleton | In-process dependency | Shared connection pool and transaction client | Constructor argument (`AppDependencies.prisma`) | Typed Prisma client | No connection validation at composition time; failures surface per request |
| Pino logger | In-process observability sink | Access logs and unhandled-error logs | Direct function calls | Structured JSON to stdout (`pino-pretty` in development) | Logging failure not handled; fire-and-forget |
| Environment variables | Process configuration | Port, DB URL, JWT secret, log level, env name | `process.env` read at module import | Strings coerced/validated by Zod | Fail-fast: `process.exit(1)` with formatted issues |
| Supertest | Test harness integration | In-process HTTP assertions against the returned `Express` app | Function call with the app instance | JSON | Test failures via assertions |
| Orchestrator / load balancer probes | Potential inbound consumer | Health checking | HTTP GET `/health` | JSON | None configured in repository; endpoint is static |
| `X-Request-Id` header | Inbound/outbound correlation contract | Trace a request across logs and callers | HTTP header | String (UUID v4 generated, arbitrary client value accepted) | No validation or length limit |
| Outbound webhooks (planned) | Future external integration | Notify customers of order status changes | Not implemented | Not implemented | Not implemented; no integration seam exists in `src/app.ts` |

Observations:

- The composed application has exactly **one external stateful dependency**: MySQL. Everything else is in-process.
- The composition root performs **no outbound calls** and creates no clients other than Express; the Prisma client is supplied by the caller.
- There is no integration-level circuit breaker, retry, timeout or bulkhead anywhere in the composed pipeline. Robustness depends on Express defaults and Prisma behavior.
- The health endpoint is the only integration contract intended for infrastructure, and it does not exercise the database, so it cannot be used as a readiness signal.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Composition Root | `buildApp()` assembles the entire object graph and middleware pipeline | `src/app.ts:55-75` | Centralize wiring, keep modules unaware of construction |
| Factory Function | `buildApp(deps)` and `buildControllers(prisma)` create new instances per call | `src/app.ts:26-53`, `55-75` | Enable in-process testing and multiple assemblies |
| Constructor Injection (manual DI) | Concrete repositories/services/controllers passed through constructors | `src/app.ts:27-44` | Explicit dependency graph without a container |
| Layered Architecture | routes -> controller -> service -> repository, assembled per module | `src/app.ts:26-53`; `src/routes/index.ts:21-31` | Separation of HTTP, business logic and persistence |
| Middleware Pipeline (Chain of Responsibility) | Ordered `app.use` registrations; each may terminate or forward | `src/app.ts:58-73` | Cross-cutting concerns without touching handlers |
| Front Controller / Router Registry | Single `/api/v1` router aggregating five module routers | `src/app.ts:67`; `src/routes/index.ts:21-31` | One entry point for the versioned API |
| Error Translation Boundary | Single terminal error middleware mapping error types to HTTP | `src/app.ts:73`; `src/middlewares/error.middleware.ts:14-65` | Uniform error contract and information hiding |
| Null Object / Fallback Handler | Catch-all middleware converting unmatched routes to `NotFoundError` | `src/app.ts:69-71` | Uniform 404 semantics |
| Singleton (ambient) | `env`, `logger`, `prisma` exported as module singletons | `src/config/env.ts:27`, `src/shared/logger/index.ts:32`, `src/config/database.ts:10` | Process-wide configuration, logging and connection pool |
| Separation of Assembly and Execution | No `listen`, no connect/disconnect in `buildApp` | `src/app.ts:55-75` vs `src/server.ts:6-22` | Reusable app for tests and alternate runtimes |
| Adapter | Controllers adapt Express `Request`/`Response` to service calls | `src/modules/*/*.controller.ts` | Isolate HTTP types from business logic |
| Guard / Interceptor | `authenticate`, `requireRole`, `validate` applied per module router | `src/modules/*/*.routes.ts` | Security and input validation at the boundary |

Architectural decisions encoded in this component:

1. **One deployable HTTP surface, one version namespace.** All modules share a single Express instance and prefix; there is no per-module server or gateway.
2. **Manual DI over a container.** The graph is small enough to wire by hand; this trades flexibility for readability and zero framework overhead.
3. **Modules own their guards.** The composition root deliberately does not enforce a global default-deny (BR-12).
4. **Ambient singletons are accepted for infrastructure.** Configuration, logging and the Prisma provider are process-wide by design.
5. **Error handling is a first-class terminal stage.** The error middleware is the only serializer of failures, which keeps response shape consistent.
6. **No async wrapper abstraction.** Controllers use explicit `try/catch` + `next(err)` because Express 4 does not handle rejected promises automatically.
7. **No security-header, CORS, compression or rate-limiting middleware.** The pipeline is minimal and contains only what the challenge application needs.

Quality attributes observed:

| Attribute | Assessment | Evidence |
|-----------|------------|----------|
| Maintainability | Good local readability; poor change isolation at the wiring point | 76 linear lines, but Ce = 18 and every new module edits this file |
| Testability | Good for end-to-end tests, weak for unit isolation | `buildApp` importable without `listen`; no interfaces to stub; app cached once |
| Modifiability | Low for cross-cutting concerns, high for route additions | Adding a route touches module router + registry + composition root |
| Security | Minimal pipeline; no helmet/CORS/rate limit; guard delegation | `src/app.ts:58-73` |
| Observability | Request IDs + structured logs; no metrics/tracing | `requestLogger`; static health |
| Resilience | No timeouts, retries, circuit breakers; masked 500 | No such middleware; body-parser errors become 500 |
| Consistency | Uniform error envelope; mixed success envelopes | Section 7 notes |
| Coupling | High efferent, low afferent (correct for a root) | Section 6 |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact | Evidence |
|------------|----------------|-------|--------|----------|
| High | Pipeline order / error handling | `express.json` is registered before `requestLogger`, and `errorMiddleware` does not inspect `err.status`/`err.type`; malformed JSON (400) and oversized bodies (413) are returned as 500 `INTERNAL_SERVER_ERROR` with no request ID and no access log | Clients receive wrong status codes; client errors pollute error logs; correlation is lost for those requests | `src/app.ts:59-60`, `src/middlewares/error.middleware.ts:14-64` |
| High | Change isolation | The composition root imports 18 internal files and instantiates 15 objects; every new module, middleware or route group requires editing it | High regression surface; merge conflicts; no compile-time guarantee that `Controllers` stays in sync | `src/app.ts:1-20`, `26-53`, `66-67`; `src/routes/index.ts:13-19` |
| High | Security pipeline | No `helmet`/security headers, no CORS policy, no rate limiting, no request timeout, no body content-type enforcement | Missing defense in depth for a public API; brute-force and abuse exposure on auth routes; browser clients unsupported by contract | `src/app.ts:58-73`; no such middleware or dependency exists |
| Medium | Testability seam | Only `prisma` is injectable; logger, config, services and repositories are concrete/ambient, and the test harness caches a single app instance | Cannot substitute fakes or verify wiring variants; app-level tests always hit real MySQL | `src/app.ts:22-24`; `tests/helpers/factories.ts:8-15` |
| Medium | Health contract | `/health` is static and does not check database connectivity | Readiness probes cannot distinguish healthy from dependency-degraded process | `src/app.ts:62-64` |
| Medium | Authorization safety net | No global authentication or default-deny; each router must remember `authenticate`/`requireRole` | A new router mounted without guards is silently public | `src/app.ts:67`; `src/modules/*/*.routes.ts` |
| Medium | Error mapping completeness | Only Prisma `P2002` and `P2025` are translated; `P2003` FK violations and other known errors become 500 | Client-caused conflicts (e.g. deleting a referenced customer) are reported as server errors | `src/middlewares/error.middleware.ts:37-54` |
| Medium | Import-time side effects | Importing `src/app.ts` triggers `env.ts` validation and possible `process.exit(1)`; `PrismaClient` singleton is created by transitive imports in consumers | Tests and tooling require full environment even for unrelated checks; process can terminate during import | `src/app.ts:3-19` -> `src/shared/logger/index.ts:2` -> `src/config/env.ts:14-27` |
| Medium | Request correlation trust | Client-supplied `X-Request-Id` is accepted without validation, length cap or sanitization | Log forging with misleading IDs; unbounded header value stored and echoed | `src/middlewares/request-logger.middleware.ts:6-8` |
| Medium | 404 reflection | `req.originalUrl` (including query string) is embedded in the error message and returned/logged | Search terms or identifiers may be reflected and logged | `src/app.ts:70` |
| Low | Hardcoded configuration | `/api/v1` prefix and `1mb` body limit are literals, not environment-driven | Requires code change and redeploy to adjust; no per-route tuning | `src/app.ts:59,67` |
| Low | Method semantics | Catch-all returns 404 for unsupported methods and `OPTIONS`; no 405 or preflight handling | API contract is imprecise; browser preflight fails | `src/app.ts:69-71` |
| Low | Middleware minimalism | No compression, no cache headers, no `trust proxy` configuration | Suboptimal payloads; incorrect client IP/protocol if deployed behind a proxy | `src/app.ts:58-73` |
| Low | Success envelope inconsistency | Paginated list responses use `{ data, pagination }`; single resources return bare objects; errors use `{ error }` | Consumers need per-endpoint response handling | Section 7 |
| Low | Duplicate validation definitions | `auth.schemas.ts` and `user.schemas.ts` define the same user shape; `user.routes.ts` redeclares the UUID param schema | Drift risk between registration and user creation contracts | `src/modules/auth/auth.schemas.ts:3-8`, `src/modules/users/user.schemas.ts:7-12`, `src/modules/users/user.routes.ts:7` |
| Low | Async error handling | Controllers rely on manual `try/catch`; a missed block produces an unhandled rejection and no response | Hanging requests and process warnings | `src/modules/*/*.controller.ts` |
| Low | Test harness safety | `tests/setup.ts` truncates all tables without verifying a test database | Running tests against a non-test `DATABASE_URL` destroys data | `tests/setup.ts:8-16` |
| Low | Unused dependency | `pino-http` is declared but unused; the custom logger is used instead | Supply-chain and maintenance noise | `package.json:31`; no import in `src` |

Positive controls observed:

- `x-powered-by` is disabled at the earliest opportunity (`src/app.ts:58`).
- Request bodies are capped at 1 MB.
- Unexpected errors are masked (no stack traces in responses) and logged server-side.
- Authentication is applied consistently to all customer, product and order routes.
- All request payloads are validated by Zod before reaching controllers.
- The composition root contains no business logic, so it cannot introduce domain inconsistencies.

---

## 11. Test Coverage Analysis

### 11.1 Test assets located

| Test file | Type | Size | Relationship to the component |
|-----------|------|------|-------------------------------|
| `tests/helpers/factories.ts` | Test infrastructure | 83 lines | Imports `buildApp` and caches one app instance (`cachedApp`); all requests go through the composition root |
| `tests/auth.test.ts` | Integration (Supertest) | 84 lines / 7 tests | Exercises public routes, validation, error mapping, auth middleware and `/auth/me` through `getTestApp()` |
| `tests/orders.test.ts` | Integration (Supertest) | 219 lines / 10 tests | Exercises order creation, status transitions, stock movement, filters, validation and error mapping through `getTestApp()` |
| `tests/setup.ts` | Test harness | 20 lines | Connects Prisma, truncates seven tables before each test, disconnects after all |
| `vitest.config.ts` | Test configuration | 19 lines | Node environment, 30 s timeouts, `tests/setup.ts`, `fileParallelism: false`, single fork |

No unit tests, contract tests, snapshot tests or end-to-end tests exist outside `tests/`. No test file imports `buildControllers`, `buildApiRouter` or any controller/service directly; every test goes through `buildApp`.

### 11.2 Coverage of composition-root behaviors

| Component behavior | Direct assertions | Indirect execution | Coverage assessment | Gaps |
|--------------------|-------------------|--------------------|---------------------|------|
| `buildApp` factory construction | 0 | 17/17 tests | Full indirect coverage of the happy path | No test builds two apps or asserts isolation |
| `buildControllers` wiring (all 5 modules) | 0 | All tests exercising auth/orders paths | Auth, users (via `/auth/me`), orders wired and exercised; customers and products controllers are **not exercised at all** | No tests for customer/product routes or their wiring |
| `express.json` parsing | 0 explicit | All POST/PATCH tests | Exercised for valid JSON | No malformed JSON, missing content type, or >1 MB tests |
| `requestLogger` / `X-Request-Id` | 0 | Every test (headers not asserted) | Executed but unverified | No assertion on response header, generated vs forwarded ID, or `userId` in logs |
| `GET /health` | 0 | 0 | **Untested** | No status/body assertion |
| Catch-all 404 (`src/app.ts:69-71`) | 0 | 0 | **Untested**; the 404 in `tests/orders.test.ts:55` originates from `OrderService`, not the catch-all | No unknown-route or unsupported-method test |
| Error middleware: `AppError` branch | 0 direct | `tests/auth.test.ts:26-27` (409 `EMAIL_ALREADY_USED`), `tests/orders.test.ts:105-106` (409 `INVALID_STATUS_TRANSITION`), `tests/orders.test.ts:126-127` (422 `INSUFFICIENT_STOCK`), `tests/orders.test.ts:216-217` (409 `INVALID_ORDER_STATE_FOR_DELETE`) | Covered for several subclasses | `UnauthorizedError` and `ForbiddenError` partially (401 for `/auth/me` and login); 403 `FORBIDDEN` never asserted |
| Error middleware: `ZodError` branch | 0 | 0 | **Untested** (the `validate` middleware intercepts first) | No test proving the safety-net branch |
| Error middleware: Prisma `P2002` | 0 | 0 | **Untested** (409 in auth test comes from `ConflictError`, not Prisma) | No unique-constraint race test |
| Error middleware: Prisma `P2025` | 0 | 0 | **Untested** | No missing-record-at-write test |
| Error middleware: 500 branch | 0 | 0 | **Untested** | No test for unmapped errors or response masking |
| `x-powered-by` suppression | 0 | 0 | **Untested** | No header assertion |
| 1 MB body limit | 0 | 0 | **Untested** | No large-payload test |
| `/api/v1` prefix contract | 17/17 requests hardcode it | Full | Strong implicit coverage | A prefix change breaks all tests, which is a coarse signal only |
| Authorization matrix | 0 systematic | 2 tests: `/auth/me` with and without token (`tests/auth.test.ts:65-83`) | Only the no-token 401 path is asserted | No test for `requireRole('ADMIN')` on `/users/:id`; no 403 test; no protected-route-without-token test besides `/auth/me` |
| Shutdown / process binding (`src/server.ts`) | 0 | 0 | **Untested** | No signal-handling test |

### 11.3 Test coverage summary

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| Application Composition Root (`src/app.ts`) | 0 | 17 (indirect, via `getTestApp`) | Not measured; no coverage provider configured (`@vitest/coverage-v8` absent, `npm test` = `vitest run`) | Good end-to-end exercise of auth and orders paths; no direct assertions on health, 404 catch-all, security headers, body limits, request IDs or the 500 branch |
| `buildControllers` wiring | 0 | 17 indirect | Not measured | Auth/orders wiring proven; customers and products controllers never invoked |
| `requestLogger` middleware | 0 | 17 indirect (executed, unverified) | Not measured | No header or log assertions |
| `errorMiddleware` | 0 | 7 indirect (409/422/401/400 paths) | Not measured | Positive and negative domain cases covered; transport and Prisma branches uncovered |
| `GET /health` | 0 | 0 | 0% | No test |
| Route-level validation (composed surface) | 0 | 2 (`auth` invalid payload, order invalid product) | Not measured | Duplicate-email and invalid-payload paths covered; most schemas unverified through HTTP |

### 11.4 Test execution characteristics

- Tests run against a real MySQL database; there is no in-memory or containerized database managed by the test runner.
- `tests/setup.ts` truncates `order_status_history`, `order_item`, `order`, `order_number_sequence`, `product`, `customer` and `user` before each test, in foreign-key-safe order.
- Execution is serialized (`fileParallelism: false`, `pool: 'forks'`, `singleFork: true`) with 30-second timeouts, so tests cannot run in parallel against the same schema.
- The app instance is built once per test process and reused; there is no per-test app rebuild, so middleware state and any future in-process caches would persist between tests.
- No coverage thresholds, reporters or CI gate are configured; `vitest.config.ts` contains no `coverage` block.
- The suite requires a valid `DATABASE_URL` and `JWT_SECRET` (>= 16 characters) at import time because `tests/helpers/factories.ts` imports `src/app.ts` (BR-03).

### 11.5 Test-related risks

| Risk | Impact |
|------|--------|
| No coverage instrumentation | Coverage claims cannot be verified; regressions in untested branches are invisible |
| No tests for customers/products modules | Two of five routers are wired but never exercised end-to-end; wiring typos in those constructors would not be caught |
| No tests for `/health` and the catch-all 404 | The two behaviors owned directly by `src/app.ts` have zero assertions |
| No negative transport tests | Malformed JSON and oversized payload behavior (currently 500) is unverified and undocumented by tests |
| Shared single app instance | Cannot verify that `buildApp` produces isolated instances; no seam for dependency substitution |
| Destructive setup without environment guard | A misconfigured `DATABASE_URL` can wipe a non-test database |

---

## 12. Ambiguities and Assumptions

1. **Component boundary.** The architectural report lists "Server Bootstrap" (`src/server.ts`) separately from "Application Composition Root" (`src/app.ts`). This analysis follows that boundary: `src/server.ts` is treated as a consumer, not as part of the component. If the intended boundary included `src/server.ts` and `src/routes/index.ts`, the coupling and lifecycle findings would expand to include signal handling, port binding and router registry maintenance.
2. **Business-rule attribution.** The composition root contains no conditionals and no domain logic. All rules in section 3 are rules of assembly, lifecycle, pipeline order or error routing. Domain rules (order state machine, stock movement, uniqueness, pricing) live in the modules and are referenced only where the composition root's wiring determines their applicability.
3. **Afferent coupling counts.** Production and test consumers are reported separately. The architectural report counts production only (`Ca = 1` for `src/app.ts`); this report shows `1 / 1` to make the test dependency explicit.
4. **Implicit rules.** BR-03 (import-time environment precondition) and BR-13 (assembly versus execution) are not documented in `src/app.ts`; they are inferred from the import graph and from the consumers' behavior, with high confidence but without an explicit comment or test.
5. **Malformed-body behavior.** The claim that body-parser errors become 500 follows from reading `errorMiddleware` (no `status`/`type` check) and from Express 4 body-parser semantics. It was not executed as part of this analysis (no code execution or test runs were performed).
6. **Coverage figures.** No coverage tooling is configured and no coverage report exists in the repository, so the "Coverage" column records "Not measured" rather than a numeric estimate.
7. **Planned webhook integration.** The repository's purpose is to document a future webhook feature that has no implementation. No webhook endpoint, worker, outbox or DLQ is wired in `src/app.ts`; this report does not treat planned scope as implemented behavior.

---

*This report is analysis-only. No project files were modified. The only file created is this report under `docs/agents/component-deep-analyzer/`.*
