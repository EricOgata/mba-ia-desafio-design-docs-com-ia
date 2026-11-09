# Architectural Analysis Report

**Project:** order-management-api (`mba-ia-desafio-design-docs-com-ia`)
**Analysis scope:** entire project root
**Excluded from analysis:** `node_modules`, `.git`, `dist`, `coverage`
**Analysis date:** 2026-11-08 22:46:20
**Analyzed revision:** `e7f6311` (branch working tree)
**Source volume:** 39 TypeScript files / 1,699 lines in `src`, 4 files / 406 lines in `tests`, Prisma schema + seed (473 lines), 1 Docker Compose file

---

## 1. Executive Summary

The repository contains a **modular monolithic REST API** for an Order Management System (OMS), implemented in **Node.js 20+ / TypeScript 5.6**, using **Express 4.21**, **Prisma 5.22**, **MySQL 8.0**, **Zod 3.23**, **JWT (HS256)**, **bcrypt**, **Pino 9** and **Vitest 2 + Supertest 7**. The runtime code is a reference application for a documentation challenge: the deliverable of the repository is a package of design documents (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/`, `docs/TRACKER.md`) for a planned **outbound webhook notification feature** described in `TRANSCRICAO.md`. That feature has **no implementation in the codebase**; the application currently has no events, outbox, queue, worker or outbound HTTP integration.

Architecturally, the application is organized as **five feature modules** - `auth`, `users`, `customers`, `products`, `orders` - each following the same internal layering: `routes` -> `controller` -> `service` -> `repository`, with `schemas` for Zod validation. Composition is manual and centralized in `src/app.ts`, which instantiates every repository, service and controller. Cross-cutting concerns live in `src/middlewares` (authentication, role authorization, validation, request logging, centralized error translation) and `src/shared` (error taxonomy, pagination helper, logger). The order domain has an explicit **state machine** (`src/modules/orders/order.status.ts`) and a **transactional command** for status changes (`src/modules/orders/order.service.ts`).

Key architectural findings:

- **Layering is consistent and the shared kernel is stable.** `src/shared/errors/index.ts` is imported by 11 source files, and the error taxonomy (`AppError` + specific subclasses) is the main cross-cutting contract.
- **The composition root is a single high-fan-out node.** `src/app.ts` imports 18 internal files and is the only place where dependency wiring exists; there is no DI container and no repository interfaces.
- **The most coupled domain component is `OrderService`** (afferent 2, efferent 5). It also bypasses the repository layer for `Customer` and `Product` access, reading those tables directly through `Prisma.TransactionClient` inside `create()` and `changeStatus()`.
- **Write-path concurrency controls are absent.** Stock debit is a read-check-then-decrement sequence without row locking or conditional update, and order status transitions read the order without `SELECT ... FOR UPDATE`. The order-number generator serializes all order creation on a single `order_number_sequence` row (`id = 1`).
- **Authorization is coarse.** Only `GET /api/v1/users/:id` requires the `ADMIN` role. Customer, product and order endpoints require authentication but no role, and there is no tenant/customer scoping on reads or writes. Public registration accepts a client-supplied `role` value, including `ADMIN`.
- **No horizontal runtime topology is defined.** There is no `Dockerfile`, no Kubernetes manifests, no CI configuration and no reverse proxy. `docker-compose.yml` provisions MySQL only; the Node process runs directly on the host via `tsx`/`node`.
- **Observability is log-only.** Pino structured logs with request IDs and redaction exist; there are no metrics, traces, readiness checks or alerting hooks. `pino-http` is declared as a dependency but is not imported anywhere in `src`, `tests` or `prisma`.
- **The planned webhook feature will add a second runtime process and a new outbound trust boundary** (customer-hosted HTTPS endpoints, HMAC signing, retries, DLQ), none of which exist today. Only the integration seams (`OrderService.changeStatus`, `AppError` hierarchy, `requireRole`, Pino logger, Prisma transaction) are present.

---

## 2. System Overview

### 2.1 Project structure

```
project-root/
├── src/
│   ├── app.ts                      # Composition root + Express app assembly
│   ├── server.ts                   # Process entry point, listen, shutdown
│   ├── config/
│   │   ├── env.ts                  # Zod-validated environment, fail-fast
│   │   └── database.ts             # PrismaClient factory + process singleton
│   ├── middlewares/
│   │   ├── auth.middleware.ts      # authenticate (JWT) + requireRole
│   │   ├── error.middleware.ts     # AppError/Zod/Prisma -> HTTP error JSON
│   │   ├── request-logger.middleware.ts  # X-Request-Id + access log
│   │   └── validate.middleware.ts  # Zod validation for body/query/params
│   ├── modules/
│   │   ├── auth/                   # controller, service, routes, schemas
│   │   ├── users/                  # controller, service, repository, routes, schemas
│   │   ├── customers/              # controller, service, repository, routes, schemas
│   │   ├── products/               # controller, service, repository, routes, schemas
│   │   └── orders/                 # controller, service, repository, routes, schemas, status
│   ├── routes/
│   │   └── index.ts                # /api/v1 router registry
│   └── shared/
│       ├── errors/                 # AppError base + HTTP error subclasses + barrel
│       ├── http/response.ts        # pagination envelope
│       └── logger/index.ts         # Pino factory + process singleton
├── prisma/
│   ├── schema.prisma               # 7 models, 2 enums, indexes
│   ├── migrations/20260519182739_init/migration.sql
│   └── seed.ts                     # destructive seed with demo users/customers/products/orders
├── tests/
│   ├── setup.ts                    # DB connect + truncate before each test
│   ├── helpers/factories.ts        # test app cache, user/customer/product factories
│   ├── auth.test.ts                # 7 integration tests
│   └── orders.test.ts              # 10 integration tests
├── docs/
│   ├── PRD.md, RFC.md, FDD.md, TRACKER.md   # placeholders (3 lines each)
│   ├── adrs/README.md              # empty ADR directory
│   └── agents/                     # generated analysis artifacts (this report, MANIFEST)
├── docker-compose.yml              # MySQL 8.0 only
├── package.json / package-lock.json
├── tsconfig.json / tsconfig.build.json
├── vitest.config.ts / .eslintrc.json / .prettierrc
├── .env.example
├── README.md                       # challenge statement, not application docs
├── TRANSCRICAO.md                  # source of the planned webhook feature
└── .opencode/                      # AI tooling (subagents/commands), not runtime code
```

Line-level notes:

- `src` has no `domain`, `entities`, `usecases`, `ports` or `adapters` directories. The layering is technical (HTTP -> service -> ORM), not hexagonal.
- `prisma/migrations/20260519182739_init/migration.sql` is the single migration and matches `prisma/schema.prisma`.
- `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` and `docs/TRACKER.md` contain only headings and placeholder comments. `docs/adrs/` contains no ADR files.
- `README.md` is the challenge statement in Portuguese (deliverable rules), not runtime documentation.
- `.opencode/agents` and `.opencode/commands` define AI subagents and commands (analysis tooling). They do not participate in the application runtime and are excluded from coupling calculations.

### 2.2 Architectural patterns identified

Multiple patterns coexist; each is documented separately.

1. **Modular monolith (primary structural pattern).** One deployable Node process, one Express app, one Prisma client, one database. Business capabilities are separated into `src/modules/<domain>` with no process or network boundary between them. Evidence: `src/app.ts` wires all modules into a single `Express` instance; `src/routes/index.ts` mounts all module routers under `/api/v1`.

2. **Layered architecture (3-tier per module).** Each module implements HTTP adapters (`*.routes.ts`), request handlers (`*.controller.ts`), business logic (`*.service.ts`), persistence (`*.repository.ts`) and input schemas (`*.schemas.ts`). Call direction is strictly downward: routes -> controller -> service -> repository -> Prisma. Evidence: `src/modules/customers/*` and `src/modules/products/*` follow this exactly.

3. **Composition root with manual dependency injection.** `buildControllers(prisma)` in `src/app.ts` (lines 26-53) news up all repositories, services and controllers in topological order and returns a `Controllers` object consumed by `buildApiRouter`. There is no DI container, no decorators, no service locator, and dependencies are concrete classes (no interfaces/ports).

4. **Repository pattern (partial).** `UserRepository`, `CustomerRepository`, `ProductRepository` and `OrderRepository` encapsulate Prisma queries. `OrderService`, however, accesses `customer` and `product` aggregates directly via the transaction client, so the pattern is not applied uniformly.

5. **Shared kernel / error taxonomy.** `src/shared/errors/app-error.ts` defines `AppError` (status code, machine-readable error code, optional details); `src/shared/errors/http-errors.ts` defines `BadRequestError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InvalidStatusTransitionError` and `InsufficientStockError`; `index.ts` is a barrel. All controllers and services throw these types; `error.middleware.ts` translates them.

6. **Schema-first boundary validation.** Zod schemas validate body, query and params in `validate.middleware.ts` before controllers run. Environment variables are validated by a separate Zod schema in `src/config/env.ts` with fail-fast behavior (`process.exit(1)`).

7. **State machine (order lifecycle).** `src/modules/orders/order.status.ts` defines a `Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>>` transition table plus `canTransition`, `allowedTransitions`, `isTerminal`, `shouldDebitStock`, `shouldReplenishStock`. Transitions are enforced in `OrderService.changeStatus` and recorded in `OrderStatusHistory`.

8. **Transactional command / unit of work.** `OrderService.create` and `OrderService.changeStatus` wrap multi-table writes in `prisma.$transaction(async (tx) => ...)`, covering `orders`, `order_items`, `order_status_history`, `products.stockQuantity` and `order_number_sequence`.

9. **Synchronous request/response only (no event-driven architecture).** No message broker, outbox table, scheduler, queue or background worker exists. `grep` over `src`, `tests` and `prisma` finds no queue, event bus, cron or worker entry point.

10. **Absent patterns, explicitly.** No CQRS, no event sourcing, no DDD aggregates/value objects (models are anemic Prisma records), no ports-and-adapters/hexagonal boundaries, no service mesh, no API gateway, no cache layer, no distributed transactions, no multi-tenancy.

### 2.3 Architectural context of the repository

The repository is a **documentation exercise fork** of `devfullcycle/mba-ia-desafio-design-docs-com-ia`. The application code is declared immutable by `README.md` (lines 66-68 and 292); the expected deliverable is design documentation for an outbound webhook system. `TRANSCRICAO.md` records the technical decisions for that future feature (outbox in MySQL, 2-second polling worker as a separate process, 5 retries with 1m/5m/30m/2h/12h backoff, DLQ table, HMAC-SHA256 with per-endpoint secret, at-least-once delivery with `X-Event-Id`, reuse of `AppError`, Pino, `requireRole` and the existing module structure). None of these elements are present in the current source tree, so the analysis below treats them as **planned scope with identifiable integration seams**, not as implemented architecture.

---

## 3. Critical Components Analysis

### 3.1 Coupling methodology

**Afferent coupling (Ca)** is the number of distinct internal source files that import the component; it measures how many other parts of the system depend on it (responsibility/instability pressure). **Efferent coupling (Ce)** is the number of distinct internal source files the component imports; it measures how many other parts the component depends on (exposure to change). Both metrics were computed statically from `import`/`import type`/`export ... from` statements across all 39 files in `src`, resolving relative `.js` specifiers to their `.ts` sources. External npm packages (`express`, `@prisma/client`, `zod`, `jsonwebtoken`, `bcrypt`, `pino`, `uuid`) are **excluded** from Ca/Ce totals and reported separately, because they are not part of the internal dependency graph. For multi-file components (feature modules, shared error taxonomy), Ca/Ce count distinct files outside the component boundary, so intra-component imports are not double-counted. Counts are static and include type-only imports, which can inflate the fan-in of contract modules such as `src/shared/errors/index.ts`.

### 3.2 Component inventory

| Component | Type | Location | Afferent Coupling | Efferent Coupling | Architectural Role |
|-----------|------|----------|-------------------|-------------------|-------------------|
| Application Composition Root | Application bootstrap | `src/app.ts` | 1 | 18 | Builds the Express app, instantiates all repositories/services/controllers, mounts `/health` and `/api/v1`, installs 404 and error handlers |
| Server Bootstrap | Process entry point | `src/server.ts` | 0 | 4 | Starts HTTP listener, registers SIGINT/SIGTERM shutdown, disconnects Prisma |
| API Router Registry | Routing | `src/routes/index.ts` | 1 | 10 | Aggregates the five module routers under `/api/v1`, defines the `Controllers` contract |
| Environment Configuration | Configuration | `src/config/env.ts` | 5 | 0 | Validates `NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`; terminates the process on invalid config |
| Prisma Client Provider | Data infrastructure | `src/config/database.ts` | 1 | 1 | Creates and exports the process-wide `PrismaClient` singleton |
| Authentication Middleware | Security middleware | `src/middlewares/auth.middleware.ts` | 5 | 2 | Bearer-token verification (JWT HS256), request user hydration, `requireRole` authorization |
| Validation Middleware | Request pipeline | `src/middlewares/validate.middleware.ts` | 5 | 1 | Parses body/query/params with Zod and converts failures to `ValidationError` |
| Request Logger Middleware | Observability middleware | `src/middlewares/request-logger.middleware.ts` | 1 | 1 | Assigns/propagates `X-Request-Id`, measures duration, emits access logs |
| Centralized Error Middleware | Cross-cutting error translation | `src/middlewares/error.middleware.ts` | 1 | 2 | Maps `AppError`, `ZodError`, Prisma `P2002`/`P2025` to HTTP JSON; logs and masks all other errors as 500 |
| Error Taxonomy | Shared kernel | `src/shared/errors/` (`app-error.ts`, `http-errors.ts`, `index.ts`) | 11 | 0 | Canonical error contract and HTTP status/code mapping used by all layers |
| Pagination Response Helper | Shared utility | `src/shared/http/response.ts` | 3 | 0 | Standard `{ data, pagination }` envelope shared by list endpoints |
| Logger | Shared infrastructure | `src/shared/logger/index.ts` | 3 | 1 | Pino singleton with redaction paths and development transport |
| Auth Module | Feature module | `src/modules/auth/` (controller, service, routes, schemas) | 2 | 6 | Public registration/login and authenticated identity lookup; delegates user persistence to the Users module |
| Users Module | Feature module | `src/modules/users/` (controller, service, repository, routes, schemas) | 4 | 3 | User creation, password hashing, public projection, admin-only user lookup |
| Customers Module | Feature module | `src/modules/customers/` (controller, service, repository, routes, schemas) | 2 | 4 | Customer CRUD, email uniqueness, paginated search |
| Products Module | Feature module | `src/modules/products/` (controller, service, repository, routes, schemas) | 2 | 4 | Product CRUD, SKU uniqueness, stock field ownership, paginated search |
| Orders Module | Feature module | `src/modules/orders/` (controller, service, repository, routes, schemas) | 2 | 4 | Order CRUD, transactional creation, status transitions, stock movement, audit history |
| Order Service | Core domain service | `src/modules/orders/order.service.ts` | 2 | 5 | Most business-critical component: totals, discount validation, item aggregation, transactions, stock debit/replenish, order-number reservation |
| Order Repository | Data access | `src/modules/orders/order.repository.ts` | 2 | 0 | Order list/count, eager-loaded order projection (items, product summary, history, customer) |
| Order State Machine | Domain rules | `src/modules/orders/order.status.ts` | 1 | 0 | Allowed transitions, terminal states, stock side-effect predicates |
| Order Status History | Audit data component | `prisma/schema.prisma` (`OrderStatusHistory`, `order_status_history`) | 2 (ORM write paths) | 0 | Append-only audit of status changes with actor and reason |
| Prisma Data Model and Migrations | Data model | `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql` | 0 | 0 | Seven tables, two enums, foreign keys and indexes; single migration |
| Seed Script | Development tooling | `prisma/seed.ts` | 0 | 0 | Destructive reset and demo data (2 users, 10 customers, 20 products, 27 orders, history) |
| Test Harness and Factories | Test infrastructure | `tests/setup.ts`, `tests/helpers/factories.ts` | 0 | 3 | DB connect/truncate lifecycle and reusable fixtures for integration tests |
| MySQL Container | Infrastructure | `docker-compose.yml` | 0 | 0 | Local MySQL 8.0 service with named volume and healthcheck |

Notes on the table:

- **Module rows and internal rows overlap by design.** Rows for `Auth`, `Users`, `Customers`, `Products` and `Orders` treat each folder as one boundary; rows for `Order Service`, `Order Repository` and `Order State Machine` break out the three most significant internals of the Orders module and are not additional boundary-level components.
- **`Order Status History` is a data component, not a class.** Its Ca is expressed as the two ORM call sites that write it (`OrderService.create`, `OrderService.changeStatus`); no TypeScript module owns it.
- **`Prisma Data Model and Migrations`, `Seed Script`, `MySQL Container`** have no internal import relationships; their architectural weight comes from schema coupling and runtime dependency rather than Ca/Ce.
- **`src/shared/errors/index.ts` has the highest afferent coupling in the system (11).** It is the de facto contract between the HTTP layer, the domain services and the error middleware.
- **`src/app.ts` has the highest efferent coupling (18).** Any new module, middleware or shared contract tends to require a change in the composition root.

### 3.3 Module-level coupling summary

| Module | Ca (modules) | Ce (modules) | Depends on | Depended on by |
|--------|--------------|--------------|------------|----------------|
| `src/app.ts` (composition) | 1 | 8 | auth, customers, middlewares, orders, products, routes, shared, users | server |
| `src/server.ts` | 0 | 3 | app, config, shared | - |
| `src/routes` | 1 | 5 | auth, customers, orders, products, users | app |
| `src/config` | 4 | 0 | - | auth, middlewares, server, shared |
| `src/middlewares` | 6 | 2 | config, shared | app, auth, customers, orders, products, users |
| `src/shared` | 8 | 1 | config | app, auth, customers, middlewares, orders, products, server, users |
| `src/modules/users` | 3 | 2 | middlewares, shared | app, auth, routes |
| `src/modules/auth` | 2 | 4 | config, middlewares, shared, users | app, routes |
| `src/modules/customers` | 2 | 2 | middlewares, shared | app, routes |
| `src/modules/products` | 2 | 2 | middlewares, shared | app, routes |
| `src/modules/orders` | 2 | 2 | middlewares, shared | app, routes |

No cyclic imports were detected at module or file level. `config` and `shared` are the most stable (high Ca, low Ce); `app`, `routes` and the feature modules are unstable consumers. `auth` is the only feature module with a business dependency on another feature module (`users`).

---

## 4. Dependency Mapping

### 4.1 High-level dependency graph

```
                     +---------------------+
                     |  scripts: dev/start |
                     +----------+----------+
                                |
                                v
 +-------------------+   +------+--------+   +----------------------+
 | src/config/env.ts |<--| src/server.ts |-->| src/config/database  |
 +---------+---------+   +------+--------+   +----------+-----------+
           ^                    |                       |
           |                    v                       v
           |            +-------+--------+        +-------------+
           +------------|    src/app.ts  |------->| PrismaClient|
           |            +---+--------+---+        +------+------+
           |                |        |                   |
           |                v        v                   v
           |          middlewares  routes/index      MySQL 8.0
           |                |        |
           |                |        +--> auth router    --> AuthController    --> AuthService    --> UserRepository/UserService
           |                |        +--> users router   --> UserController    --> UserService    --> UserRepository
           |                |        +--> customers     --> CustomerController--> CustomerService--> CustomerRepository
           |                |        +--> products      --> ProductController --> ProductService --> ProductRepository
           |                |        +--> orders        --> OrderController   --> OrderService   --> OrderRepository
           |                |                                                        |
           |                |                                                        +--> Prisma tx (customers, products, orders, items, history, sequence)
           |                v
           |          shared/errors  <---- thrown by all services/controllers/middlewares
           |          shared/logger  <---- used by server, request logger, error middleware
           |          shared/http    <---- used by customer/product/order services
           |
           +---- read by auth middleware, logger, database
```

### 4.2 Layering rules observed

```
HTTP request
  -> requestLogger middleware
  -> express.json (1 MB limit)
  -> /api/v1 router
     -> module router: authenticate -> validate -> controller
        -> service (business rules, transactions)
           -> repository -> PrismaClient -> MySQL
  -> response JSON

Error path
  controller/service throws AppError | ZodError | PrismaClientKnownRequestError
  -> next(err)
  -> errorMiddleware (status/code mapping, logging, 500 masking)
```

Rules that hold across all five modules:

- Controllers never touch Prisma; they only call services and shape HTTP responses.
- Services never touch `req`/`res`; they receive plain inputs and return domain records or shared error types.
- Repositories never contain business decisions; they translate filters into Prisma queries.
- Routes compose middleware in a fixed order: `authenticate` (where applicable) -> `validate` -> controller.
- Errors always propagate via `next(err)` and are serialized by the single error middleware.

Rules that are violated or partially applied:

- `OrderService` performs direct `tx.customer.findUnique`, `tx.product.findMany`, `tx.product.update` and `tx.orderNumberSequence.upsert` calls instead of delegating to `CustomerRepository`, `ProductRepository` or dedicated helpers. This creates implicit coupling between the Orders module and the `customers`/`products` tables and bypasses the repository boundary.
- `AuthController` depends on `UserService` (not only `AuthService`), and `AuthService` depends on `UserRepository` plus `UserService`. `auth` therefore has both a service-level and a repository-level dependency on `users`.
- `OrderController` depends on `OrderService` and on the `ListOrdersQuery` schema type; the controller casts `req.query` with `as unknown as ListOrdersQuery` after middleware validation.
- `users` route defines an inline `idParamSchema` in `user.routes.ts` instead of using `userIdParamSchema` from `user.schemas.ts`, creating duplicated validation logic.

### 4.3 Data and control flow for the critical path (order lifecycle)

```
POST /api/v1/orders
  authenticate (JWT -> req.user)
  validate(createOrderSchema)
  OrderController.create
    OrderService.create(input, userId)
      aggregate duplicate productIds
      BEGIN TRANSACTION
        SELECT customer by id            (404 if missing)
        SELECT products by ids           (404 if any missing)
        reject inactive products         (422 INACTIVE_PRODUCT)
        compute unitPrice, line totals, subtotal, total
        reject discount > subtotal       (400 VALIDATION_ERROR)
        UPSERT order_number_sequence id=1 (serializes writers)
        INSERT order + items + history(PENDING)
      COMMIT
  -> 201 order with items, history, customer summary

PATCH /api/v1/orders/:id/status
  authenticate -> validate(params, body) -> OrderController.changeStatus
    OrderService.changeStatus(id, {toStatus, reason}, userId)
      BEGIN TRANSACTION
        SELECT order + items            (no row lock)
        reject same-status transition   (409)
        canTransition(from, to)         (409 INVALID_STATUS_TRANSITION)
        if PENDING -> PAID:             read products, check stock, decrement
        if PAID|PROCESSING -> CANCELLED: increment stock
        UPDATE order.status
        INSERT order_status_history
        re-SELECT order with relations
      COMMIT
  -> 200 updated order

Planned webhook seam (not implemented): the outbox insert would occur inside the same
transaction, before COMMIT, so that status change and event publication are atomic.
```

### 4.4 External dependency edges

| Component | External packages |
|-----------|-------------------|
| `src/app.ts` | `express`, `@prisma/client` (type) |
| `src/server.ts` | none |
| `src/config/env.ts` | `zod` |
| `src/config/database.ts` | `@prisma/client` |
| `src/middlewares/auth.middleware.ts` | `express`, `jsonwebtoken` |
| `src/middlewares/error.middleware.ts` | `express`, `@prisma/client`, `zod` |
| `src/middlewares/request-logger.middleware.ts` | `express`, `uuid` |
| `src/middlewares/validate.middleware.ts` | `express`, `zod` |
| `src/modules/auth/auth.service.ts` | `bcrypt`, `jsonwebtoken` |
| `src/modules/users/user.service.ts` | `bcrypt`, `@prisma/client` |
| All repositories | `@prisma/client` |
| `src/modules/orders/order.schemas.ts`, `order.status.ts` | `@prisma/client` (enums/types) |
| `src/shared/logger/index.ts` | `pino` |
| `src/shared/errors/*` | none |

Every HTTP-facing file depends on `express` types; the ORM leaks into five services/repositories and both order domain files. `pino-http` and `@types/uuid` are declared in `package.json` but have no imports in the application source (the logger is a custom middleware).

---

## 5. Integration Points

| Integration | Type | Location | Purpose | Risk Level |
|-------------|------|----------|---------|------------|
| MySQL 8.0 | Relational database | `docker-compose.yml`, `prisma/schema.prisma`, `src/config/database.ts` | Primary and only data store for users, customers, products, orders, items, history, sequence | High |
| Prisma ORM 5.22 | Data access / transaction manager | `src/config/database.ts`, all `*.repository.ts`, `OrderService` | Query building, relation loading, interactive and batch transactions, schema migrations | High |
| Prisma shadow database | Migration infrastructure | `prisma/schema.prisma` (`SHADOW_DATABASE_URL`), `.env.example` | Schema diffing for `prisma migrate dev` | Medium |
| JWT (`jsonwebtoken` 9.0.2) | Authentication token | `src/modules/auth/auth.service.ts`, `src/middlewares/auth.middleware.ts` | HS256 access tokens carrying `sub`, `email`, `role`; 8-hour default lifetime | High |
| bcrypt 5.1.1 | Password hashing | `src/modules/users/user.service.ts`, `src/modules/auth/auth.service.ts`, `prisma/seed.ts` | Hash on user creation (cost 10), constant-time comparison at login | Medium |
| Zod 3.23.8 | Input/config validation | `src/config/env.ts`, `src/middlewares/validate.middleware.ts`, `*.schemas.ts` | Boundary validation, type inference, environment fail-fast | Low |
| Pino 9.5.0 / pino-pretty 11.3.0 | Structured logging | `src/shared/logger/index.ts` | JSON logs in non-development, pretty transport in development, redaction | Low |
| uuid 11.0.3 | Identifier generation | `src/middlewares/request-logger.middleware.ts` | `X-Request-Id` generation when the header is absent | Low |
| Express 4.21.1 | HTTP framework | `src/app.ts`, all routes/controllers | Routing, JSON body parsing (1 MB), middleware pipeline, error propagation | Medium |
| Outbound webhook delivery (HTTPS) | External HTTP integration | Not implemented; specified in `TRANSCRICAO.md` and `README.md` | Notify B2B customers of order status changes with HMAC-signed payloads | High (planned) |
| Customer webhook endpoints / secrets | Third-party endpoints | Not implemented | Per-customer receivers, per-endpoint secrets, rotation with 24h grace period | High (planned) |
| Dead-letter queue | Internal persistence integration | Not implemented (planned `webhook_dead_letter` table) | Store permanently failed deliveries and support admin replay | Medium (planned) |
| Message broker / Redis | Not present | - | Explicitly rejected in `TRANSCRICAO.md` (`[09:07] Diego`) in favor of MySQL outbox | Not applicable |
| SMTP / email | Not present | - | Explicitly out of scope in `TRANSCRICAO.md` (`[09:37] Larissa`) | Not applicable |
| CI/CD, container registry, reverse proxy, APM | Not present | - | No files detected | Not applicable |

Integration observations:

- The application has exactly **one external stateful dependency**: MySQL. All other integrations are in-process libraries.
- Credentials are supplied entirely through environment variables loaded by `node --env-file=.env` / `tsx --env-file=.env`; there is no secret manager, vault or encrypted configuration.
- `docker-compose.yml` publishes MySQL on host port `3306` and uses `${VAR:-default}` fallbacks, so the stack can start with default credentials.
- `SHADOW_DATABASE_URL` is required by `prisma migrate dev` but is **not** validated by `src/config/env.ts` and no `oms_shadow` database is created by `docker-compose.yml`; the value is read directly by Prisma.
- There is no outbound network call anywhere in `src`; the future webhook worker would introduce the first egress path and the first untrusted inbound response surface (customer-controlled status codes, redirects, slow responses, oversized bodies).

---

## 6. Architectural Risks and Single Points of Failure

### 6.1 Risk register

| Risk Level | Component | Issue | Impact | Details |
|------------|-----------|-------|--------|---------|
| Critical | Auth Module / Users Module | Public registration accepts a client-supplied `role`, including `ADMIN` | Privilege escalation, full administrative access | `registerSchema` includes `role: z.enum(['ADMIN','OPERATOR']).default('OPERATOR')` (`src/modules/auth/auth.schemas.ts:7`); `AuthService.register` forwards the payload to `UserService.createUser` (`src/modules/auth/auth.service.ts:27-29`), which persists `input.role` unchanged (`src/modules/users/user.service.ts:21-33`); `POST /api/v1/auth/register` has no authentication or role guard (`src/modules/auth/auth.routes.ts:10`) |
| Critical | Order Service (`changeStatus`) | Status transition reads the order without row locking or optimistic version check | Concurrent duplicate transitions, double stock debit/replenish, duplicate audit rows | Transaction begins with `tx.order.findUnique` (`src/modules/orders/order.service.ts:131-136`); two concurrent requests can both observe `PENDING`, both pass `canTransition`, both debit stock and both insert history |
| High | Order Service (`debitStock`) | Stock is validated by read-then-write without `SELECT ... FOR UPDATE` or a conditional decrement | Overselling / negative `stockQuantity` under concurrency | `tx.product.findMany` then per-item `tx.product.update({ stockQuantity: { decrement } })` (`src/modules/orders/order.service.ts:204-231`); no `where: { stockQuantity: { gte: quantity } }` guard |
| High | Order Number Sequence | A single-row `upsert` on `order_number_sequence.id = 1` runs inside every order-creation transaction | Write serialization and lock contention on the hottest path; throughput ceiling per database | `reserveOrderNumber` performs `upsert` with `nextValue: { increment: 1 }` (`src/modules/orders/order.service.ts:245-254`); InnoDB holds an exclusive row lock until commit |
| High | Authorization layer | Customer, product and order routers require only `authenticate`, not `requireRole`; no customer/tenant scoping | Any authenticated actor can read or mutate all customers, products, orders; B2B data isolation is absent | `src/modules/customers/customer.routes.ts:14`, `src/modules/products/product.routes.ts:14`, `src/modules/orders/order.routes.ts:14`; list queries filter only by request-supplied fields; `order.controller.ts:9-17` forwards query without ownership constraints |
| High | Runtime topology | Single Node process, no container image, no replica set, no load balancer, no process manager config | Application is a single point of failure; deploy/restart drops availability | `src/server.ts` calls `app.listen` once; no `Dockerfile`, `kubernetes/`, `helm/`, `pm2`, `systemd` or CI files exist |
| High | MySQL | One database instance defined in Compose; no replication, failover, backup or PITR configuration | Database outage stops all functionality; data loss risk on volume failure | `docker-compose.yml` defines a single `mysql:8.0` service with one local named volume; no backup jobs or replicas |
| High | Configuration / secrets | Weak defaults in `.env.example` and Compose fallbacks; MySQL port exposed to host | Credential reuse and database exposure in non-hardened environments | `.env.example:5-14` contains `oms_password`, `root_password`, and a sample JWT secret; `docker-compose.yml:7-10,12` maps `3306:3306` and defaults root password to `root_password`; `JWT_SECRET` minimum length is only 16 characters (`src/config/env.ts:8`) |
| Medium | Event/publication layer | No outbox, event, queue or worker exists; status changes are invisible outside the request/response cycle | The planned webhook feature has no implementation substrate; external notification depends on future work | Absence confirmed by source scan; planned design is documented only in `TRANSCRICAO.md` (outbox, polling worker, DLQ) and `README.md` |
| Medium | Error Middleware | Prisma errors other than `P2002`/`P2025` are not mapped | FK violations and other DB errors surface as generic 500 instead of 4xx | `src/middlewares/error.middleware.ts:37-54` handles only unique-constraint and record-not-found; deleting a customer/product referenced by orders triggers `P2003` (FK RESTRICT) and falls through to the 500 branch |
| Medium | Server Bootstrap | `process.exit(0)` runs immediately after `prisma.$disconnect()`, without awaiting `server.close` | In-flight requests are terminated during shutdown; no connection draining | `src/server.ts:13-18`: `server.close(() => logger.info(...))` is non-blocking and its callback is not awaited |
| Medium | Health endpoint | `/health` returns a static payload and does not probe the database | Orchestrators and monitors cannot distinguish process liveness from dependency readiness | `src/app.ts:62-64` returns `{ status: 'ok' }` unconditionally |
| Medium | Order creation API | No idempotency key or request de-duplication | Client retries create duplicate orders and duplicate order numbers | `POST /api/v1/orders` validates body only (`src/modules/orders/order.routes.ts:18`); `OrderController.create` has no idempotency handling (`src/modules/orders/order.controller.ts:28-36`) |
| Medium | Test harness | `beforeEach` truncates every table using the configured `DATABASE_URL` with no environment guard | Running tests against a non-test database destroys data | `tests/setup.ts:8-16` deletes history, items, orders, sequence, products, customers and users; no `NODE_ENV`/URL assertion |
| Medium | Orders Module boundary | `OrderService` writes/reads `customers`, `products` and `order_number_sequence` directly through Prisma | Bypasses repository contracts; schema changes propagate into domain service | `src/modules/orders/order.service.ts:58-123, 204-254` |
| Medium | Business rule validation | Orders can be created without stock availability; stock is checked only at `PENDING -> PAID` | Accepted orders may be unfulfillable; oversell exposure is deferred to payment time | `OrderService.create` checks product existence and `active`, not `stockQuantity` (`src/modules/orders/order.service.ts:62-73`); `tests/orders.test.ts:109-119` confirms creation succeeds with quantity above stock |
| Low | Dependencies | `pino-http` and `@types/uuid` are declared but unused | Unnecessary supply-chain surface and maintenance noise | Only `package.json` references `pino-http`; `uuid` is used, its types package is redundant for uuid v11 |
| Low | API contract | No OpenAPI/Swagger specification or generated API docs | Integration consumers depend on source reading; contract drift is unverifiable | No `swagger`, `openapi` or API documentation files detected |
| Low | Delivery pipeline | No CI workflow, no build/test gate, no container build | Regressions and non-building revisions can reach the default branch | No `.github/`, `.gitlab-ci.yml`, `Jenkinsfile` or equivalent files |

### 6.2 Single points of failure

1. **`order_number_sequence` row 1** - every order creation depends on the same physical row for numbering.
2. **The single MySQL instance** - all state, including audit history, lives in one non-replicated container volume.
3. **The single Node process** - HTTP serving, business logic and ORM share one event loop and one Prisma connection pool; a crash or deploy stops all API traffic.
4. **`src/shared/errors/index.ts`** - 11 files import it; any breaking change to the error contract propagates system-wide (high Ca, zero Ce).
5. **`src/app.ts` composition root** - the only wiring point; it must be edited for every new module or middleware, and it statically imports all 18 internal dependencies.
6. **`JWT_SECRET`** - one symmetric secret validates every token; compromise or rotation invalidates or forges all sessions.
7. **`src/config/env.ts` fail-fast** - any missing/invalid variable terminates the process at startup, including in test and tooling contexts.

### 6.3 Architectural debt inventory (observed, not prescriptive)

- Repository dependencies are concrete classes, not interfaces; there are no ports for persistence or for external calls, so substitution/testing requires the real Prisma client.
- `OrderService` is a large orchestration unit (255 lines) mixing pricing, validation, persistence, stock movement, numbering and audit concerns.
- Validation knowledge is duplicated: `auth.schemas.ts` (`registerSchema`) and `user.schemas.ts` (`createUserSchema`) define the same shape; `user.routes.ts` redeclares the UUID param schema inline instead of importing `userIdParamSchema`.
- Controllers rely on non-null assertions (`req.params.id!`) and double casts (`req.query as unknown as ListOrdersQuery`), so validated data is not statically guaranteed at the handler boundary.
- Order status rules exist only in application code; there is no database-level constraint or version column to protect against out-of-band writes.
- Date handling uses MySQL `DATETIME(3)` with application-side `Date` objects; no explicit timezone strategy is encoded in schema or configuration.
- `prisma/migrations` has a single monolithic migration dated `20260519182739`; no migration verification runs in CI.
- The test suite covers only `auth` and `orders`; `customers`, `products`, `users` and the health endpoint have no tests. Tests are integration-only against a real MySQL instance and run serially (`fileParallelism: false`, single fork).
- Documentation artifacts (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, `docs/adrs/`) are placeholders; the only substantive documentation of future architecture is the transcript.

---

## 7. Technology Stack Assessment

| Layer | Technology | Version | Usage | Architectural Assessment |
|-------|------------|---------|-------|--------------------------|
| Runtime | Node.js | >= 20 (engines) | ESM (`"type": "module"`), `--env-file` support | Modern baseline; single-process model; no cluster/worker threads |
| Language | TypeScript | 5.6.3 | Strict mode, `noUncheckedIndexedAccess`, ES2022 target, bundler resolution | Strong compile-time discipline; `.js` specifiers in TS imports for ESM output |
| HTTP framework | Express | 4.21.1 | Router, middleware chain, JSON parser | Mature but minimal; no built-in security headers, rate limiting, validation or async error handling (hence custom middleware) |
| ORM / data access | Prisma Client | 5.22.0 | Repositories, interactive transactions, nested writes, batch transactions | Strong schema typing and migration tooling; transaction client leaks into `OrderService`; version pinned exactly |
| Database | MySQL | 8.0 (Compose) | InnoDB, utf8mb4, ENUM columns, JSON column for address | Single-node deployment; row-lock-sensitive write paths; JSON `address` lacks DB-level shape constraints |
| Validation | Zod | 3.23.8 | HTTP schemas, environment schema, type inference | Consistent boundary validation; duplicated schema definitions across modules |
| Authentication | jsonwebtoken | 9.0.2 | HS256 signing/verification | Stateless bearer tokens; no refresh, revocation, key rotation or `algorithms` allowlist at verification |
| Password hashing | bcrypt | 5.1.1 | Cost factor 10 | Standard adaptive hashing; sync-free promise API; 72-byte input cap reflected in schema (`max(72)`) |
| Logging | Pino | 9.5.0 | Structured JSON logs, redaction, dev transport | Good baseline; no correlation beyond `X-Request-Id`, no metrics/tracing exporter |
| Logging (dev) | pino-pretty | 11.3.0 | Development transport | Development-only |
| ID generation | uuid | 11.0.3 | Request IDs | Database IDs are Prisma `uuid()` defaults |
| HTTP logging | pino-http | 10.3.0 | Declared only | Not imported; redundant with the custom request logger |
| Testing | Vitest | 2.1.4 | Integration tests, forks, single fork | No unit tests, no coverage thresholds, serial DB-bound execution |
| HTTP assertions | Supertest | 7.0.0 | In-process app requests | Exercises the full middleware/controller/service/DB stack |
| Build | tsc (`tsconfig.build.json`) | 5.6.3 | `src` -> `dist`, source maps off | No bundler; ESM output executed by `node dist/server.js` |
| Lint/format | ESLint 8 (legacy config) + Prettier | 8.57.1 / 3.3.3 | `@typescript-eslint` recommended, consistent type imports, no-explicit-any warn | Enforced locally; not enforced by CI |
| Dev runner | tsx | 4.19.2 | `tsx watch` for dev, seed execution | No nodemon/ts-node; ESM-native |
| Containerization | Docker Compose | - | MySQL only | Application is not containerized |

Version pinning is exact for all direct dependencies (no `^`/`~`), and `package-lock.json` is committed, which makes builds reproducible but requires deliberate upgrades. The lockfile and dependency vulnerability posture are outside this report's scope and are covered by the dependency audit.

Notable stack-level observations:

- `express@4` is used with async controllers that manually `try/catch` and call `next(err)`; Express 5's automatic promise rejection handling is not available.
- `jsonwebtoken` verification does not pass an `algorithms` allowlist; the library constrains HMAC secrets, but algorithm pinning is not explicit.
- `PrismaClient` is instantiated at module import time (`src/config/database.ts:10`), so importing any module that transitively imports it triggers client creation; this affects tests and any future worker entry point.
- The application relies on `--env-file` at the process level rather than `dotenv`, so any tool that does not set the flag (for example a raw `node` invocation or a test runner) must receive variables from the environment.
- There is no OpenAPI generator, no API client generation and no contract test between the API and the future webhook worker.

---

## 8. Security Architecture and Risks

### 8.1 Trust boundaries and controls

```
Untrusted                        Boundary controls                      Trusted
--------------------------------+---------------------------------------+---------------------------
Internet clients                | express.json 1 MB limit                | Service layer
(unknown identities)            | requestLogger (X-Request-Id)           | (business rules)
                                | authenticate: Bearer JWT HS256         |
                                | requireRole: ADMIN|OPERATOR (partial)  | Prisma transaction
                                | validate: Zod body/query/params        | MySQL
--------------------------------+---------------------------------------+---------------------------
Host environment                | env.ts: required DATABASE_URL,         | Process memory
(.env file, Compose env)        | JWT_SECRET >= 16 chars, fail-fast      | (Prisma singleton,
                                |                                        |  JWT secret)
--------------------------------+---------------------------------------+---------------------------
MySQL container                 | Compose default credentials,           | Database files
(host port 3306 published)      | named volume                           | (plaintext PII)
--------------------------------+---------------------------------------+---------------------------
Planned: customer endpoints     | None implemented (planned HMAC-SHA256, | Future worker
(customer-controlled HTTP)       | X-Signature, X-Event-Id, TLS-only URL) | (not present)
```

### 8.2 Authentication and session model

- **Mechanism:** stateless JWT bearer tokens, HMAC-SHA256, signed with a single `JWT_SECRET`; claims are `sub` (user id), `email`, `role`, `iat`, `exp` (`src/modules/auth/auth.service.ts:47-50`).
- **Lifetime:** `JWT_EXPIRES_IN` defaults to `8h` (`src/config/env.ts:9`). There is no refresh token, no logout endpoint, no token denylist and no server-side session record.
- **Verification:** `jwt.verify` only; the payload is cast to `JwtPayload` without a Zod/structural check (`src/middlewares/auth.middleware.ts:40-46`). No `algorithms` allowlist is supplied.
- **Authorization claims:** `req.user.role` comes exclusively from the token. Role changes or user deletion in the database are not re-validated per request, so a still-valid token retains its original privileges for up to 8 hours.
- **Password storage:** bcrypt cost 10; password input capped at 72 characters in schemas; `passwordHash` is never included in `PublicUser` responses. Tests assert that `passwordHash` and `password` are absent from registration responses (`tests/auth.test.ts:14-16`).
- **Registration:** `POST /api/v1/auth/register` is public, unthrottled and accepts `role` from the request body, including `ADMIN`. This is the most direct privilege-escalation path in the system.

### 8.3 Endpoint authorization matrix

| Endpoint | Authentication | Role requirement | Tenant/customer scoping |
|----------|----------------|------------------|-------------------------|
| `GET /health` | None | None | Not applicable |
| `POST /api/v1/auth/register` | None | None (role chosen by caller) | Not applicable |
| `POST /api/v1/auth/login` | None | None | Not applicable |
| `GET /api/v1/auth/me` | JWT | Any | Self only (uses `req.user.id`) |
| `GET /api/v1/users/:id` | JWT | `ADMIN` | None; any user id can be read by an admin |
| `GET/POST/PATCH/DELETE /api/v1/customers` | JWT | Any | None; all customers visible and mutable |
| `GET/POST/PATCH/DELETE /api/v1/products` | JWT | Any | None; stock and price mutable by operators |
| `GET/POST /api/v1/orders` | JWT | Any | None; list has no owner constraint |
| `GET/DELETE /api/v1/orders/:id` | JWT | Any | None |
| `PATCH /api/v1/orders/:id/status` | JWT | Any | None; any operator can pay, ship, deliver or cancel any order |
| Planned `POST /admin/webhooks/dead-letter/:id/replay` | Not implemented | Planned `ADMIN` via existing `requireRole` | Planned |

### 8.4 Security risks and exposure areas

| Severity | Area | Finding | Evidence |
|----------|------|---------|----------|
| Critical | Registration / authorization | Client-controlled role enables self-provisioning of `ADMIN` | `src/modules/auth/auth.schemas.ts:7`, `src/modules/auth/auth.service.ts:27-29`, `src/modules/users/user.service.ts:21-33`, `src/modules/auth/auth.routes.ts:10` |
| High | Authorization granularity | No role checks and no tenant scoping on customers, products and orders; all authenticated actors have full CRUD and lifecycle control | `src/modules/customers/customer.routes.ts:14-24`, `src/modules/products/product.routes.ts:14-24`, `src/modules/orders/order.routes.ts:14-24` |
| High | Transport security | Application does not enforce HTTPS, HSTS or secure headers; TLS termination is undefined (no reverse proxy or ingress files) | No `helmet`, `hsts`, TLS config or proxy configuration in the repository |
| High | Brute force / abuse | No rate limiting, lockout, CAPTCHA or backoff on `POST /auth/login` or `POST /auth/register` | No rate-limiting dependency or middleware; `src/modules/auth/auth.routes.ts:10-12` |
| High | Secret management | Default and example secrets committed; MySQL root password defaulted; database port published | `.env.example:5-14`, `docker-compose.yml:7-12` |
| Medium | Token lifecycle | 8-hour stateless tokens, no revocation/refresh; role changes not re-validated; no algorithm pinning | `src/config/env.ts:8-9`, `src/middlewares/auth.middleware.ts:40-46`, `src/modules/auth/auth.service.ts:47-50` |
| Medium | PII handling | Customer name, email, phone, document and full address stored plaintext, returned to any authenticated actor, with no masking or field-level access control | `prisma/schema.prisma:40-53`, `src/modules/customers/customer.service.ts:23-41` |
| Medium | Logging and privacy | Request logs include full `originalUrl`, which may contain search terms, emails or documents; redaction covers credentials only | `src/middlewares/request-logger.middleware.ts:12-24`, `src/shared/logger/index.ts:4-11` |
| Medium | Seed credentials | Demo users `admin@oms.local`/`admin123` and `operador@oms.local`/`operator123` are hardcoded | `prisma/seed.ts:6-7, 191-210` |
| Medium | Test data safety | Test setup truncates every table of the configured database without verifying it is a test database | `tests/setup.ts:4-16` |
| Medium | Error handling | Unmapped Prisma errors can surface as 500 with server-side logging of the full error object; `P2003` constraint failures are not translated | `src/middlewares/error.middleware.ts:37-64` |
| Low | Input hardening | No maximum URL/route length, no explicit content-type enforcement beyond `express.json`, no CORS policy, no CSRF (token-based API reduces but does not define the policy) | `src/app.ts:58-67` |
| Low | Dependency exposure | `pino-http` declared but unused widens the supply-chain surface | `package.json:31` |

Positive security controls observed:

- Passwords are hashed with bcrypt and never serialized (`PublicUser` projection).
- The error middleware returns generic messages for unexpected failures, avoiding stack-trace disclosure.
- Pino redaction covers `authorization`, `cookie`, `password`, `passwordHash`, `token` and `accessToken` paths.
- `x-powered-by` is disabled.
- All identifiers are validated as UUIDs at the boundary; Prisma parameterizes queries, so string-concatenation SQL injection is not present.
- Role-gated access exists and is correctly applied to the only admin route (`GET /users/:id`).
- Request bodies are capped at 1 MB.
- Secrets must satisfy minimum validation (`JWT_SECRET` >= 16 characters, `DATABASE_URL` non-empty) or the process refuses to start.

Planned webhook security surface (not implemented, therefore not assessable in code): per-endpoint HMAC-SHA256 secrets, secret rotation with a 24-hour grace window, mandatory `https` URLs, 64 KB payload ceiling, `X-Signature`/`X-Timestamp`/`X-Event-Id`/`X-Webhook-Id` headers, and `ADMIN`-only DLQ replay. These are documented in `TRANSCRICAO.md` at `[09:20]-[09:26]`, `[09:35]-[09:36]` and `[09:42]-[09:44]`; no corresponding schema, endpoint or worker exists.

---

## 9. Infrastructure Analysis

### 9.1 Present infrastructure assets

| Asset | File | Content | Notes |
|-------|------|---------|-------|
| Local database service | `docker-compose.yml` | `mysql:8.0`, container `oms-mysql`, restart `unless-stopped`, host port `3306`, named volume `oms_mysql_data`, `mysqladmin ping` healthcheck every 5 s with 20 retries | Only service defined; application is not containerized |
| Environment template | `.env.example` | `NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `SHADOW_DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, MySQL credentials | Contains placeholder/weak values; the real `.env` is gitignored |
| Runtime scripts | `package.json` | `dev` (`tsx watch --env-file=.env src/server.ts`), `build` (`tsc -p tsconfig.build.json`), `start` (`node --env-file=.env dist/server.js`), `db:migrate`, `db:reset`, `db:seed`, `test`, `lint`, `format` | No `worker` script, no `migrate deploy` script, no container build script |
| Build configuration | `tsconfig.build.json` | Compiles `src` to `dist` with `rootDir: ./src` | ESM output, no source maps in build |
| Test configuration | `vitest.config.ts` | Node environment, 30 s timeouts, `tests/setup.ts`, serial forks | Requires an externally provisioned MySQL |
| Database migration | `prisma/migrations/20260519182739_init/migration.sql` | Creates 7 tables, enums, unique indexes, secondary indexes and 6 foreign keys | Applied manually via npm scripts |
| Seed routine | `prisma/seed.ts` | Deletes all rows, creates 2 users, 10 customers, 20 products, 27 orders with full history | Destructive; no environment guard |

### 9.2 Deployment topology as defined by the repository

```
+---------------------------------------------------------------+
| Developer host / single deploy target                         |
|                                                               |
|  [Node process]                                               |
|   node --env-file=.env dist/server.js  (or tsx watch)          |
|      |                                                        |
|      | DATABASE_URL=mysql://oms_user:oms_password@localhost:3306/oms
|      v                                                        |
|  +---------------------------+      published 3306           |
|  | Docker: oms-mysql (8.0)   |<------------------------------+
|  | volume: oms_mysql_data    |                                |
|  | healthcheck: mysqladmin   |                                |
|  +---------------------------+                                |
|                                                               |
|  No container image for the API, no reverse proxy, no replica |
|  set, no orchestrator, no CI/CD, no backup job, no metrics     |
+---------------------------------------------------------------+
```

### 9.3 Deployment and operational observations

- **The API has no deployment artifact.** There is no `Dockerfile`, `.dockerignore`, Kubernetes manifest, Helm chart, ECS task definition, Procfile or systemd unit. Deployment is implicit (`npm run build` + `npm start` on a host with Node 20+).
- **Startup ordering is manual.** The database must be healthy before the process starts; there is no wait-for-database logic, no retry on connection failure, and no migration step in the start script. `bootstrap()` fails fast and exits on any thrown error (`src/server.ts:24-27`).
- **Schema management is developer-driven.** `prisma migrate dev` requires `SHADOW_DATABASE_URL`; `prisma migrate deploy` is not exposed as an npm script.
- **Health checking is asymmetric.** MySQL has a Compose healthcheck; the API exposes a static `/health` that ignores dependency state.
- **Shutdown is not graceful for HTTP traffic.** `server.close` is initiated but not awaited before `process.exit(0)`; Prisma is disconnected first.
- **Observability is limited to stdout logs.** No log aggregation, metrics endpoint, tracing, alerting or dashboard configuration exists. `LOG_LEVEL` is the only runtime knob.
- **Configuration is environment-variable only.** Eleven variables are documented in `.env.example`; `src/config/env.ts` validates six of them, while `SHADOW_DATABASE_URL` and the `MYSQL_*` variables are consumed by Prisma CLI and Compose respectively.
- **Testing requires infrastructure.** `npm test` expects a reachable MySQL with migrations applied; the suite truncates shared tables before each test and runs with a single fork, so parallel CI jobs on the same database would interfere.
- **No release hygiene artifacts.** No changelog automation, versioning policy, CI matrix, artifact retention or environment promotion configuration is present.
- **The planned webhook worker has no infrastructure yet.** There is no `src/worker.ts`, no `npm run worker` script, no second Compose service, no scheduler, and no dead-letter table; `TRANSCRICAO.md` `[09:11]` specifies a separate process on the same database and Prisma stack.

---

## 10. Consolidated Architecture Snapshot

| Dimension | Current State |
|-----------|---------------|
| Architectural style | Modular monolith, layered per feature, synchronous HTTP only |
| Deployable units | 1 (Node API process) + 1 infrastructure service (MySQL 8.0) |
| Business modules | 5 (`auth`, `users`, `customers`, `products`, `orders`) |
| Cross-cutting modules | 4 middlewares, 3 shared packages (`errors`, `http`, `logger`) |
| Source files / lines | 39 / 1,699 (`src`); 4 / 406 (`tests`) |
| Persistence models | 6 (`User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`) + 1 sequence table |
| API versioning | Path-based `/api/v1`, single version, 19 route handlers across 5 module routers (20 including `/health`) |
| Highest afferent coupling | `src/shared/errors/index.ts` (11) |
| Highest efferent coupling | `src/app.ts` (18) |
| Highest-risk component | `OrderService` (transactional stock/status logic without concurrency controls) and the public registration endpoint (role escalation) |
| Event/async infrastructure | None |
| Containerization of the app | None |
| CI/CD | None |
| Observability | Pino structured logs + request IDs only |
| Documentation of runtime architecture | Absent; challenge documentation only |

---

*This report is analysis-only. No project files were modified. The only file created is this report under `docs/agents/architectural-analyzer/`.*
