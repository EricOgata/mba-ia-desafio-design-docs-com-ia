# Component Deep Analysis Report

**Component:** Prisma Client Provider
**Primary file:** `src/config/database.ts` (10 lines)
**Project:** `order-management-api` (`mba-ia-desafio-design-docs-com-ia`)
**Analysis scope:** entire project root
**Excluded from analysis:** `node_modules`, `.git`, `dist`, `coverage`
**Analysis date:** 2026-11-08 23:01:25
**Method:** static source analysis only. No project file was modified, no code was executed, no tests were run.

---

## 1. Executive Summary

The Prisma Client Provider is the data-infrastructure boundary of the application. It is a single 10-line module that performs three responsibilities: (1) construct a `PrismaClient` with an environment-dependent logging configuration, (2) expose a factory function `createPrismaClient()`, and (3) export one process-wide `prisma` singleton consumed by the HTTP bootstrap, the test harness and the test factories.

Despite its small size, the component carries a high architectural weight. It is the only instantiation point of the ORM in production, it is the gate through which all seven database tables are reached, and it triggers the configuration-validation chain that can terminate the process at startup. Every persistence operation in the system (users, customers, products, orders, order items, status history and the order-number sequence) passes through the client it creates.

Key findings:

- **Very small surface, very large blast radius.** The module imports one internal file (`src/config/env.ts`) and one external package (`@prisma/client`), but its exported singleton is imported by `src/server.ts` and by all three test infrastructure files. Any failure in this module stops the entire application, because it is evaluated transitively at process start.
- **No business/domain logic lives here.** All domain rules (totals, discounts, stock, status transitions, uniqueness) live in services and repositories; this component only supplies a configured connection handle. Its "business rules" are therefore infrastructure policies: logging policy, singleton policy, datasource policy, lifecycle policy and error-propagation policy.
- **Configuration is fixed at import time and cannot be overridden.** `createPrismaClient()` accepts no parameters and reads no runtime arguments, so tests, the seed script and any future worker cannot obtain a differently configured client from this module. The seed script demonstrates this by creating its own separate `PrismaClient` (`prisma/seed.ts:4`).
- **Connection is lazy and unverified at startup.** No `$connect()` call exists in `src/`; the server starts its HTTP listener without proving database reachability, and the static `/health` endpoint does not probe the database (`src/app.ts:62-64`).
- **The shutdown path does not drain HTTP traffic.** `prisma.$disconnect()` is awaited, but `server.close()` is not awaited before `process.exit(0)` (`src/server.ts:13-18`).
- **No dedicated tests exist for this component.** It is executed indirectly by all 15 integration tests (7 auth, 8 orders), which gives high statement execution but almost no behavioural assertion and no coverage of failure paths, log-policy branching or signal handling. No coverage provider is installed, so no coverage measurement is produced.

Criticality classification: **High**. The component is small, stable and has a narrow contract, but it is a single point of failure for all functionality and a mandatory participant in process startup and shutdown.

---

## 2. Data Flow Analysis

### 2.1 Startup and client creation

```
1. Process start
   dev:  tsx watch --env-file=.env src/server.ts       (package.json:11)
   prod: node --env-file=.env dist/server.js           (package.json:13)
2. Node.js loads .env into process.env (--env-file), no dotenv dependency
3. ESM module graph evaluation for src/server.ts
   3a. src/config/env.ts evaluates
       - Zod schema validates NODE_ENV, PORT, LOG_LEVEL, DATABASE_URL, JWT_SECRET, JWT_EXPIRES_IN
       - invalid -> console.error + process.exit(1)     (env.ts:14-24)
       - valid   -> export const env                     (env.ts:27)
   3b. src/config/database.ts evaluates
       - import { PrismaClient } from '@prisma/client'
       - import { env } from './env.js'
       - createPrismaClient() -> new PrismaClient({ log: ... })   (database.ts:4-8)
       - export const prisma = <instance>                          (database.ts:10)
       - NOTE: Prisma opens no socket at construction; connection is lazy
4. src/app.ts evaluates (type-only PrismaClient import; no runtime ORM instantiation)
5. src/server.ts bootstrap():
       buildApp({ prisma })                                       (server.ts:7)
         -> buildControllers(prisma)                              (app.ts:26-53)
            -> UserRepository(prisma)
            -> CustomerRepository(prisma)
            -> ProductRepository(prisma)
            -> OrderRepository(prisma)
            -> OrderService(orderRepository, prisma)
6. app.listen(env.PORT) starts accepting HTTP traffic             (server.ts:9)
```

### 2.2 Request-path data flow through the client

```
HTTP request
  -> requestLogger -> express.json (1 MB) -> /api/v1 router
  -> authenticate -> validate(Zod) -> controller -> service
  -> repository (or OrderService direct tx client)
  -> prisma.<model>.<operation>(...)                               (shared singleton)
  -> Prisma query engine (in-process binary) -> connection pool
  -> MySQL 8.0 at DATABASE_URL (docker-compose.yml: mysql:8.0, port 3306)
  -> rows/objects returned as generated TypeScript types
  -> service/controller -> JSON response
```

Concrete consumption patterns observed:

- Single-record reads: `prisma.user.findUnique`, `prisma.customer.findUnique`, `prisma.product.findUnique`, `prisma.order.findUnique`.
- Filtered pagination with batch transactions: `$transaction([findMany, count])` in `customer.repository.ts:26-34`, `product.repository.ts:27-35`, `order.repository.ts:35-43`.
- Relational eager loading: `order.findUnique({ include: { items: { include: { product } }, history, customer } })` (`order.repository.ts:47-60`).
- Nested writes: `order.create({ data: { items: { create }, history: { create } } })` (`order.service.ts:95-120`).
- Interactive transactions: `prisma.$transaction(async (tx) => ...)` in `order.service.ts:58` and `order.service.ts:131`.
- Atomic increments/decrements: `stockQuantity: { increment | decrement }` (`order.service.ts:226-241`).
- Upsert on the single sequence row: `orderNumberSequence.upsert` (`order.service.ts:246-251`).
- Bulk deletion used by tests: `deleteMany()` on seven models (`tests/setup.ts:9-15`).

### 2.3 Failure-path data flow

```
Database unreachable
  -> first query throws PrismaClientInitializationError (e.g. P1001)
  -> error propagates through repository/service (no try/catch in providers)
  -> error.middleware.ts: not AppError, not ZodError, not PrismaClientKnownRequestError
  -> logger.error(...) + HTTP 500 INTERNAL_SERVER_ERROR            (error.middleware.ts:56-64)

Known request error
  -> PrismaClientKnownRequestError
     P2002 -> HTTP 409 CONFLICT (unique constraint)                 (error.middleware.ts:38-47)
     P2025 -> HTTP 404 NOT_FOUND                                    (error.middleware.ts:48-53)
     other codes (P2003, P2034, ...) -> generic HTTP 500            (error.middleware.ts:56-64)
```

### 2.4 Shutdown and test data flow

```
SIGINT / SIGTERM
  -> shutdown(signal)                                               (server.ts:13)
  -> logger.info('shutdown_initiated')
  -> server.close(callback)   [not awaited]
  -> await prisma.$disconnect()
  -> process.exit(0)          [can terminate before close callback]

Tests
  -> tests/setup.ts beforeAll: await prisma.$connect()
  -> beforeEach: deleteMany orderStatusHistory -> orderItem -> order -> orderNumberSequence
                 -> product -> customer -> user                     (tests/setup.ts:8-16)
  -> afterAll: await prisma.$disconnect()                           (tests/setup.ts:18-20)
  -> tests/helpers/factories.ts: buildApp({ prisma }), user/customer/product inserts
  -> tests/orders.test.ts: direct prisma.product.findUnique assertions
```

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location |
|---------|-----------|------------------|----------|
| BR-01 | Configuration | In `development`, Prisma logs `warn` and `error`; in `test` and `production`, it logs only `error` | `src/config/database.ts:6` |
| BR-02 | Lifecycle | Exactly one `PrismaClient` instance exists per Node.js process (module-level singleton) | `src/config/database.ts:10` |
| BR-03 | Construction | Client creation must pass through `createPrismaClient()`; the factory is the single construction point in `src/` | `src/config/database.ts:4-8,10` |
| BR-04 | Configuration | Client options are fixed at creation time; callers cannot override datasource, log level or pool settings | `src/config/database.ts:4-8` |
| BR-05 | Startup validation | The client is only reachable after environment validation succeeds; missing/empty `DATABASE_URL` terminates the process | `src/config/database.ts:2`, `src/config/env.ts:7,14-27` |
| BR-06 | Datasource | The database URL is resolved by Prisma from `DATABASE_URL`; migrations additionally require `SHADOW_DATABASE_URL` | `prisma/schema.prisma:5-9`, `.env.example:5-6` |
| BR-07 | Connection | Connections are established lazily on first query; there is no startup connectivity probe | `src/config/database.ts:5-7`, `src/server.ts:6-11` |
| BR-08 | Lifecycle | The application must disconnect the singleton on `SIGINT`/`SIGTERM` before exiting | `src/server.ts:13-21` |
| BR-09 | Resource management | All request handlers share one connection pool; no per-request or per-repository client is created | `src/config/database.ts:10`, `src/app.ts:26-53` |
| BR-10 | Transactions | No client-level transaction isolation/timeout policy is configured; Prisma and MySQL defaults apply to every `$transaction` | `src/config/database.ts:5-7`, `src/modules/orders/order.service.ts:58,131` |
| BR-11 | Schema contract | The client is generated from `prisma/schema.prisma` using the `prisma-client-js` generator and must match the MySQL schema | `prisma/schema.prisma:1-3`, `prisma/migrations/20260519182739_init/migration.sql` |
| BR-12 | Error handling | The provider never catches or rewraps ORM errors; translation to HTTP is the exclusive responsibility of the error middleware | `src/config/database.ts:4-8`, `src/middlewares/error.middleware.ts:37-54` |
| BR-13 | Test isolation | Tests use the same singleton and must truncate all tables before each test; no dedicated test client exists | `tests/setup.ts:4-20` |
| BR-14 | Tooling deviation | The seed script creates an independent client and destructively resets data, bypassing the provider singleton | `prisma/seed.ts:4,178-187,333-335` |

### Detailed breakdown of the business rules

---

### Business Rule: BR-01 Environment-Conditional Log Configuration

**Overview**:
The provider selects the Prisma log verbosity from the validated `NODE_ENV`. Development exposes warnings and errors; every other environment exposes only errors.

**Detailed description**:
The rule is implemented as a single ternary at `src/config/database.ts:6`: `log: env.NODE_ENV === 'development' ? ['warn', 'error'] : ['error']`. Because `env.NODE_ENV` is validated by the Zod enum in `src/config/env.ts:4` and defaults to `development`, only three values can reach this expression: `development`, `test` and `production`. In development the developer sees Prisma warnings (including connection-pool pressure and deprecation notices) alongside errors. In `test` and `production` the warning channel is discarded, so only error-level events are emitted. No `query`, `info` or `debug` channel is ever enabled, which means the rule intentionally excludes SQL query logging in all environments.

This rule matters for two operational use cases. First, local troubleshooting: a developer can observe connection and engine warnings while working with `tsx watch`, without configuring anything. Second, production noise control: with `NODE_ENV=production`, expected warnings cannot clutter the Pino output consumed by log aggregation. The trade-off is that the same suppression applies to the `test` environment, so integration-test diagnostics about pool or engine warnings are invisible, and production loses early warning signals about connection exhaustion. The rule also means there is no way to temporarily raise Prisma verbosity without changing `NODE_ENV` (which changes the entire application's behaviour) or editing this file, because `LOG_LEVEL` from `src/config/env.ts:6` is not consulted by this component.

The rule has a second-order effect on observability ownership: application-level logging is handled by Pino (`src/shared/logger/index.ts`), and database-level logging is handled here. Because the two systems are not correlated beyond the request id created in `src/middlewares/request-logger.middleware.ts`, a Prisma error log cannot be joined to the HTTP request log except by timestamp. Nothing in the code bridges the two.

**Rule workflow**:
```
env.ts: NODE_ENV <- process.env.NODE_ENV (default 'development')
  |
database.ts: NODE_ENV === 'development' ?
  yes -> log = ['warn','error']
  no  -> log = ['error']
  |
PrismaClient created with that log array (no 'query', no 'info', no 'debug')
```

**Evidence:** `src/config/database.ts:6`, `src/config/env.ts:4,27`
**Confidence:** High (explicit code)

---

### Business Rule: BR-02 Single Process-Wide Client Instance (Singleton)

**Overview**:
The module exports exactly one `PrismaClient` instance, created when the module is first evaluated. Every consumer in the process receives the same object.

**Detailed description**:
`src/config/database.ts:10` assigns `createPrismaClient()` to a module-scoped constant and exports it. ESM module caching guarantees that the initializer runs once per process, so repeated imports from `src/server.ts:3`, `tests/setup.ts:2`, `tests/helpers/factories.ts:3` and `tests/orders.test.ts:9` all resolve to the same instance and the same internal connection pool. This is the standard Node.js pattern for database clients and it is what makes constructor injection in `src/app.ts:26-53` safe: the same handle is passed to four repositories and to `OrderService`, so all of them share one pool and one transaction engine.

The singleton has three practical consequences. First, pool sizing is global: the effective MySQL connection count is determined once by Prisma (from the datasource URL and CPU count), not per module or per request, so adding a repository does not add connections. Second, lifecycle is global: a `$disconnect()` call from any consumer affects all consumers, which is why the test harness can safely disconnect in `afterAll`, and why the production shutdown path can disconnect once in `src/server.ts:16`. Third, any state attached to the client (extensions, middleware, `$on` listeners, transaction options) would be global; today none is attached.

The rule also constrains future architecture. A second runtime process (for example the planned webhook worker described in `TRANSCRICAO.md`) cannot import this module and obtain an isolated client; it would either share the module (and thus the process's client) or, as the seed script does, construct `new PrismaClient()` directly. Similarly, a read replica, a tenant-specific datasource or a test database separate from the runtime database cannot be expressed through this module as written, because the singleton is bound to one `DATABASE_URL` at import time.

**Rule workflow**:
```
first import of src/config/database.ts
  -> env validation (via env.ts)
  -> createPrismaClient()
  -> instance stored in module scope
subsequent imports
  -> cached module namespace returned, same instance
all repositories/services/tests -> shared instance
```

**Evidence:** `src/config/database.ts:10`, `src/server.ts:3,7`, `src/app.ts:26-53`, `tests/setup.ts:2`, `tests/helpers/factories.ts:3,12`, `tests/orders.test.ts:9`
**Confidence:** High (explicit code plus ESM semantics)

---

### Business Rule: BR-03 Client Construction Must Pass Through the Factory

**Overview**:
Within application source, `PrismaClient` is instantiated only inside `createPrismaClient()`. Consumers must receive a client, not create one.

**Detailed description**:
The factory at `src/config/database.ts:4-8` is the single `new PrismaClient(...)` expression in `src/`. All repositories and `OrderService` accept `PrismaClient` through their constructors (`user.repository.ts:4`, `customer.repository.ts:10`, `product.repository.ts:11`, `order.repository.ts:19`, `order.service.ts:27-30`), and the composition root supplies the singleton (`src/app.ts:27,34,38,42-43`). This creates a constructor-injection contract: persistence consumers are testable in principle because they depend on an injected handle rather than importing the singleton themselves. The architecture report confirms this as the "composition root with manual dependency injection" pattern.

The factory is exported but is not imported by any other file in the repository (verified by search: only `src/config/database.ts:4` and the call at line 10 reference `createPrismaClient`). It therefore serves two roles: it isolates the construction expression for readability and future parameterization, and it exposes a seam that is currently unused. There is no interface or port for the client; consumers depend on the concrete generated `PrismaClient` type, so substitution would require a compatible Prisma-shaped object or the real client.

There is one documented deviation. `prisma/seed.ts:4` constructs `new PrismaClient()` directly, and the seed script is executed by `tsx --env-file=.env prisma/seed.ts` (`package.json:16`), outside the API process. This is consistent with the rule's scope ("within application source") but means the logging policy of BR-01 is not applied to seeding, and the seed does not share the API's configuration decisions.

**Rule workflow**:
```
app bootstrap -> createPrismaClient() -> prisma singleton
  -> buildControllers(prisma)
     -> new <Repository>(prisma)
     -> new OrderService(repo, prisma)
request path -> injected client method calls
(no consumer executes `new PrismaClient` in src/)
```

**Evidence:** `src/config/database.ts:4-8`, `src/app.ts:26-53`, repository constructors, `src/modules/orders/order.service.ts:27-30`
**Confidence:** High (explicit code); factory has zero external importers (observed)

---

### Business Rule: BR-04 Client Options Are Fixed at Creation (No Runtime Overrides)

**Overview**:
`createPrismaClient()` takes no arguments and reads no request-scoped or caller-scoped configuration. The only variable input is the validated `env` object.

**Detailed description**:
The signature is `export function createPrismaClient(): PrismaClient` (`src/config/database.ts:4`). Consequently the datasource, log configuration, error format, transaction options and pool settings are identical for every consumer in the process. There is no way for a caller to request a different connection string, a different log level, an alternative error format, or a client bound to a transaction. The composition root cannot vary the client per module, and tests cannot request a test-database client through this API.

For the current single-database, single-tenant application this is a simplifying constraint: it eliminates configuration drift between modules and guarantees that the repositories and `OrderService` operate on the same transactional context. It also makes reasoning about shutdown straightforward, since there is exactly one client to disconnect. The constraint becomes limiting when the system needs more than one datasource, more than one pool policy, or a per-worker configuration; in those cases callers must bypass the factory (as the seed does) or the factory must be extended.

A related implicit rule is that the Prisma schema's `env("DATABASE_URL")` is the actual source of the URL, not this module. The module reads `env` only for `NODE_ENV`; `DATABASE_URL` is read by the generated client/schema layer. So the component's configuration surface is narrower than it appears: it controls logging only, while the connection target is controlled by the schema and the environment.

**Rule workflow**:
```
createPrismaClient()  [no parameters]
  -> new PrismaClient({ log: <from env.NODE_ENV> })
  -> datasource URL resolved by Prisma from env("DATABASE_URL") in schema
  -> no caller-level overrides possible
```

**Evidence:** `src/config/database.ts:4-8`, `prisma/schema.prisma:5-9`
**Confidence:** High (explicit signature)

---

### Business Rule: BR-05 Fail-Fast Environment Validation Before Client Creation

**Overview**:
Importing the provider transitively validates the environment. If `DATABASE_URL` is missing or empty, the process prints the validation issues and exits with code 1 before any client is created.

**Detailed description**:
`src/config/database.ts:2` imports `env` from `./env.js`. That module builds a Zod schema in which `DATABASE_URL` is `z.string().min(1, 'DATABASE_URL is required')` (`src/config/env.ts:7`), parses `process.env` (`env.ts:15`), formats all issues, writes them to `console.error`, and calls `process.exit(1)` on failure (`env.ts:16-23`). Because ESM evaluates imported modules before the importing module body, the validation completes before `createPrismaClient()` runs at `database.ts:10`. The same validation also enforces `JWT_SECRET` length (`env.ts:8`) and enum values for `NODE_ENV` and `LOG_LEVEL`, so a malformed JWT secret also prevents the database provider from initializing.

The rule is a startup guard, not a runtime guard. It guarantees that a process with a missing database URL never reaches `app.listen`, which is the desired fail-fast behaviour for a 12-factor-style application. It does not validate that the URL is syntactically a MySQL DSN, that the host is reachable, that the credentials are correct, or that the schema has been migrated. Those failures surface later, on the first query, and produce Prisma initialization errors that the error middleware maps to HTTP 500. The rule also does not validate `SHADOW_DATABASE_URL`, even though the Prisma schema declares it and `prisma migrate dev` requires it (`prisma/schema.prisma:8`); that gap is outside this component but directly adjacent to it.

A notable operational side effect is that `process.exit(1)` inside an imported module is hostile to embedding scenarios. Any tool that imports `src/config/database.ts` (a test runner, a script, a future worker) inherits the exit behaviour, which means configuration must always be present in the environment even for static analysis, type checking or unit tests that would not need a database. This is why `vitest.config.ts` and the test scripts depend on externally provisioned environment variables.

**Rule workflow**:
```
import database.ts
  -> import env.ts
     -> envSchema.safeParse(process.env)
        success -> export env, continue
        failure -> console.error(issues) -> process.exit(1)
  -> createPrismaClient() reads env.NODE_ENV
  -> prisma singleton exported
```

**Evidence:** `src/config/database.ts:2`, `src/config/env.ts:7,14-27`
**Confidence:** High (explicit code)

---

### Business Rule: BR-06 Datasource Resolution and Shadow Database for Migrations

**Overview**:
The client connects to the MySQL database identified by `DATABASE_URL`. Schema migrations additionally require `SHADOW_DATABASE_URL`, which is consumed by the Prisma CLI rather than by the runtime client.

**Detailed description**:
`prisma/schema.prisma:5-9` declares `provider = "mysql"`, `url = env("DATABASE_URL")` and `shadowDatabaseUrl = env("SHADOW_DATABASE_URL")`. The runtime client resolves the URL from the environment variable at generation/instantiation time; the provider module itself never reads `DATABASE_URL`. The local environment template points to `mysql://oms_user:oms_password@localhost:3306/oms` with a shadow database `oms_shadow` (`/.env.example:5-6`), and `docker-compose.yml:2-24` provisions a single `mysql:8.0` instance with matching default credentials and a named volume.

This rule determines the blast radius of an environment mistake: because every repository and transaction shares the singleton, a wrong `DATABASE_URL` in one environment affects all requests with no fallback. There is no read/write split, no replica URL, no connection-parameter policy (TLS mode, `connect_timeout`, `pool_timeout`, `connection_limit`) expressed in the schema or in this module. Any such parameters would have to be appended to the URL string by the operator. The shadow database is a developer/CI concern: it is required for `prisma migrate dev` (wired as `db:migrate` in `package.json:14`) but is not created by `docker-compose.yml`, so a fresh local environment must provision it separately.

The rule also encodes the database engine choice at the deepest layer: MySQL 8.0 with `utf8mb4` and InnoDB, inherited from Compose and the migration SQL (`prisma/migrations/20260519182739_init/migration.sql:13`). The provider does not adapt to PostgreSQL or SQLite; changing engines would require schema, migration and generated-client changes, not a change in this module.

**Rule workflow**:
```
process env DATABASE_URL
  -> Prisma datasource env("DATABASE_URL")
  -> generated client connects to that DSN
  -> MySQL 8.0 (docker-compose) executes parameterized queries
prisma migrate dev
  -> additionally reads SHADOW_DATABASE_URL for schema diffing
```

**Evidence:** `prisma/schema.prisma:5-9`, `.env.example:5-6`, `docker-compose.yml:2-24`, `package.json:14-15`
**Confidence:** High (explicit schema/config); pool and TLS parameters are absent by observation, not by documented decision

---

### Business Rule: BR-07 Lazy Connection Establishment and No Startup Probe

**Overview**:
Creating the client does not prove database availability. The first query (or an explicit `$connect`) opens connections; production code never performs an explicit connect or readiness check.

**Detailed description**:
`createPrismaClient()` only constructs the client object (`src/config/database.ts:5-7`). Prisma's documented behaviour is to open connections lazily when the first query needs them. In `src/server.ts:6-11`, `bootstrap()` builds the app and calls `app.listen` without any `$connect()` or `$queryRaw` probe. The only explicit `$connect()` in the repository is in the test harness (`tests/setup.ts:4-6`), and the only readiness signal exposed over HTTP is the static `/health` handler returning `{ status: 'ok' }` (`src/app.ts:62-64`). The seed script likewise connects implicitly on its first operation.

The consequence is a deliberate (or at least accepted) availability trade-off: the API process can report itself started while the database is down. The first request that touches the database fails with a Prisma initialization error, which the error middleware masks as HTTP 500 because it only special-cases `PrismaClientKnownRequestError` codes P2002 and P2025. Orchestrators and load balancers cannot distinguish "process alive" from "database reachable", so traffic may be routed to an instance that cannot serve. A second consequence is cold-start latency: the first request after start pays the TCP/TLS and authentication cost, and if the pool is exhausted or the database is slow the first request can hit Prisma's connection timeout rather than a fast failure at boot.

From a resilience standpoint the rule removes the option of failing the deployment early. A process that cannot reach MySQL stays up and keeps returning 500s until an operator intervenes, rather than exiting and letting a supervisor restart it. Conversely, the lazy behaviour is what allows `npm run build`, type checking and any tooling that imports the module to work without a live database, as long as the environment variables are syntactically valid.

**Rule workflow**:
```
module import -> client object only (no socket)
app.listen -> HTTP ready, DB unverified
first query -> Prisma opens pool / authenticates
  success -> normal operation
  failure -> PrismaClientInitializationError -> HTTP 500
explicit connect only in tests/setup.ts:5
```

**Evidence:** `src/config/database.ts:5-7`, `src/server.ts:6-11`, `src/app.ts:62-64`, `tests/setup.ts:4-6`
**Confidence:** High for absence of probe; High for lazy-connect behaviour (documented Prisma semantics, consistent with code)

---

### Business Rule: BR-08 Application-Owned Disconnect on Termination Signals

**Overview**:
The API process must close the shared client on `SIGINT` and `SIGTERM` before exiting. The provider itself registers no hooks; the bootstrap owns the lifecycle.

**Detailed description**:
`src/server.ts:13-21` defines `shutdown(signal)`: it logs the signal, initiates `server.close(...)`, awaits `prisma.$disconnect()`, then calls `process.exit(0)`. Handlers are registered for `SIGINT` and `SIGTERM`. The provider contributes the exported singleton that makes this possible but owns none of the logic, which keeps the module free of process-level side effects beyond instantiation. Because the singleton is shared, a single disconnect closes the pool used by every repository and transaction.

The sequencing has a known weakness: `server.close()` is asynchronous and its callback only logs; it is not awaited before `process.exit(0)`. The await applies exclusively to Prisma. As a result, in-flight HTTP requests can be terminated while their database operations are still pending, and connections may be closed underneath a handler that is mid-transaction. Prisma's `$disconnect` waits for the engine to shut down, but any query issued after that point will fail. The intended pattern (stop accepting connections, drain in-flight requests, then disconnect, then exit) is only partially implemented.

There is no disconnect on uncaught exceptions or unhandled rejections. `bootstrap().catch` logs a fatal error and exits with code 1 (`server.ts:24-27`), but that path runs before or around startup; runtime crashes rely on the process dying and the pool being reclaimed by the OS. There is also no `beforeExit`/`exit` hook, so a future long-running worker that imports the provider must replicate the signal handling itself.

**Rule workflow**:
```
SIGINT/SIGTERM
  -> shutdown(signal)
     -> logger.info
     -> server.close(cb)   [async, not awaited]
     -> await prisma.$disconnect()
     -> process.exit(0)
```

**Evidence:** `src/server.ts:13-21`, `src/config/database.ts:10`
**Confidence:** High (explicit code)

---

### Business Rule: BR-09 Single Shared Connection Pool, No Per-Request Clients

**Overview**:
One client means one pool for the entire process. Request handlers, services, repositories and transactions all compete for the same connection budget.

**Detailed description**:
Because the singleton is created once (`src/config/database.ts:10`) and injected into every repository and into `OrderService` (`src/app.ts:26-53`), there is no request-scoped or module-scoped client. Every concurrent HTTP request shares the same pool. Prisma derives the pool size from the datasource URL and the machine's CPU count when no explicit `connection_limit` is present; this repository sets none, so the default applies. The only per-request isolation is logical: batch transactions (`$transaction([...])`) and interactive transactions (`$transaction(async (tx) => ...)`) borrow connections from the shared pool for their duration.

For the current workload this is the correct default: five feature modules, low request concurrency in a documentation-challenge context, and no long-running background jobs. The rule becomes a capacity constraint under load. Interactive transactions in `OrderService.create` and `OrderService.changeStatus` hold a connection for the whole transaction, including multiple sequential statements and a row-locking `upsert` on `order_number_sequence`; if the pool is smaller than the number of concurrent checkouts, requests queue and eventually hit Prisma's pool timeout. The single shared pool also means one misbehaving query can starve every module, regardless of domain.

The rule additionally couples observability and tuning: pool metrics are only visible through Prisma log warnings/errors (which, per BR-01, are suppressed outside development) or through MySQL-side `SHOW PROCESSLIST`. There is no application metric for pool utilization.

**Rule workflow**:
```
one PrismaClient -> one internal connection pool
  request A -> repository query -> borrow connection -> return
  request B -> interactive transaction -> hold connection until commit
  request C -> waits if pool exhausted -> pool timeout error
```

**Evidence:** `src/config/database.ts:10`, `src/app.ts:26-53`, `src/modules/orders/order.service.ts:58-123,131-179`
**Confidence:** High for single pool; Medium for exact default sizing (library default, not configured here)

---

### Business Rule: BR-10 Transaction Semantics Delegated to Prisma and MySQL Defaults

**Overview**:
The provider configures no transaction isolation level, timeout, max wait or retry policy. Every `$transaction` call inherits Prisma's defaults and MySQL's session defaults.

**Detailed description**:
`new PrismaClient({ log: ... })` (`src/config/database.ts:5-7`) supplies only the `log` option. It does not set `transactionOptions`, does not install middleware that retries deadlocks, and does not pin isolation. Consumers use two transaction forms: batch transactions for list+count (`customer.repository.ts:26`, `product.repository.ts:27`, `order.repository.ts:35`) and interactive transactions for order creation and status change (`order.service.ts:58,131`). Under Prisma 5 defaults, interactive transactions have a 2-second `maxWait` to acquire a connection and a 5-second execution timeout; the isolation level is the database default unless overridden, which for MySQL InnoDB is `REPEATABLE READ`.

This rule is intentional minimalism: the provider stays out of domain semantics, and each consumer decides what to run atomically. The practical effect is that atomicity is guaranteed, but concurrency control is not. The order-creation transaction serializes all writers through a single `order_number_sequence` row, while the status-change transaction performs a read-check-write without row locking; those are consumer-level weaknesses, but the provider neither mitigates nor exposes knobs to mitigate them. A long transaction (for example the per-item update loop in `debitStock`, `order.service.ts:225-230`) can exceed the 5-second interactive timeout and abort, leaving the client to surface a generic Prisma error that the HTTP layer maps to 500.

The rule also implies no retry-on-transient-error behaviour at the client level. Deadlocks (`P2034`) or serialization failures are surfaced to the caller as `PrismaClientKnownRequestError`; the error middleware does not special-case them, so they become HTTP 500. Any future resilience policy (retry with backoff, isolation bump, statement timeout) must be implemented by consumers or by extending this factory.

**Rule workflow**:
```
consumer calls prisma.$transaction(...)
  -> no client-level transactionOptions
  -> Prisma default maxWait/timeout applies
  -> MySQL InnoDB default isolation applies
  -> failure codes propagate to caller (no retry)
```

**Evidence:** `src/config/database.ts:5-7`; consumers at `src/modules/orders/order.service.ts:58,131`; `src/modules/customers/customer.repository.ts:26`; `src/modules/products/product.repository.ts:27`; `src/modules/orders/order.repository.ts:35`
**Confidence:** Medium-High (absence is explicit; numeric defaults come from Prisma 5 documentation and MySQL defaults, not from repository configuration)

---

### Business Rule: BR-11 Generated Client Contract Must Match the Schema

**Overview**:
The exported type and runtime API are produced by `prisma generate` from `prisma/schema.prisma`. The application compiles and runs only when the generated client matches the committed schema.

**Detailed description**:
`prisma/schema.prisma:1-3` selects `generator client { provider = "prisma-client-js" }`, and the datasource is MySQL. The generated package provides `PrismaClient`, the model delegates used by every repository, the `Prisma.*` input types, and the `OrderStatus`/`UserRole` enums imported by `order.status.ts`, `order.schemas.ts`, `order.service.ts`, `order.repository.ts`, `user.service.ts` and others. Thirteen source files and one test helper import from `@prisma/client`, so the generated artifact is a compile-time dependency of most of the codebase. The provider module is the runtime instantiation point of that artifact.

The rule creates a build-order dependency: `npm run build` runs only `tsc -p tsconfig.build.json` (`package.json:12`) and does not run `prisma generate`. Generation normally happens during `npm install` via the `@prisma/client` postinstall hook, but a schema change without a subsequent generate yields stale types and a client that does not know new tables. The workspace currently has no `node_modules` directory, so any build or test run requires a fresh install first. There is no `postinstall` or `prebuild` script making the dependency explicit, and no CI workflow enforcing it.

At runtime the contract is also a version lock: the `prisma` CLI and `@prisma/client` are both pinned to `5.22.0` (`package.json:26,48`). The dependency audit classifies this line as legacy (two majors behind 7.10.1). The `prisma-client-js` generator named in the schema is deprecated in Prisma 7 in favour of `prisma-client`, which changes the generated client location and requires a driver adapter or Accelerate URL; upgrading would therefore force changes in this module and in every import of `@prisma/client`.

**Rule workflow**:
```
prisma/schema.prisma (models, enums, datasource)
  -> prisma generate (install-time or manual)
  -> node_modules/.prisma/client + @prisma/client exports
  -> database.ts imports PrismaClient and instantiates it
  -> 13 src files consume generated types/delegates
build: tsc only (no explicit generate step)
```

**Evidence:** `prisma/schema.prisma:1-9`, `package.json:12,26,48`, imports across `src`, generated-client absence in the workspace
**Confidence:** High (explicit schema/scripts); generation timing is environment-dependent and stated as such

---

### Business Rule: BR-12 Errors Are Propagated Unwrapped to the HTTP Boundary

**Overview**:
The provider and its factory contain no error handling. ORM errors reach the centralized error middleware, which translates only `AppError`, `ZodError`, `P2002` and `P2025`; everything else becomes a generic 500.

**Detailed description**:
`createPrismaClient()` returns the client directly and the singleton is a plain export (`src/config/database.ts:4-10`). There is no wrapper, no `try/catch`, no error callback, no `$on('error')` subscription and no Prisma middleware/extensions. Errors raised by model operations therefore propagate through the repository and service layers unchanged. The single translation point is `src/middlewares/error.middleware.ts`: it maps `AppError` subclasses, `ZodError`, and two Prisma codes (`P2002` unique constraint to 409, `P2025` record not found to 404). All remaining Prisma errors (`P1001` unreachable database, `P1002` timeout, `P1008`, `P2003` foreign-key constraint, `P2034` deadlock, `P2024` pool timeout) fall through to the logging branch and return `INTERNAL_SERVER_ERROR`.

This is a clean separation of concerns in principle: infrastructure does not decide HTTP semantics, and the HTTP layer owns the mapping table. In practice the mapping table is incomplete. A foreign-key violation when deleting a customer or product referenced by an order (`RESTRICT` in the migration, `migration.sql:110-119`) is a client-visible business condition but is reported as 500 and logged at error level. A pool timeout under load is an availability signal but is indistinguishable from a programming bug. The provider's contribution to this rule is the decision not to normalize errors, which keeps it dependency-free but pushes all resilience decisions to consumers and middleware.

The rule also affects security: `PrismaClientKnownRequestError` carries `meta` that can include constraint and column names. Only P2002 exposes part of `meta` (the target list) in the response; other codes are masked. No stack trace or connection string is returned to clients, because the 500 branch returns a fixed message. Full error objects are logged server-side with the request id.

**Rule workflow**:
```
query failure
  -> repository/service: no catch
  -> errorMiddleware
     AppError        -> mapped status/code
     ZodError        -> 400 VALIDATION_ERROR
     P2002           -> 409 CONFLICT
     P2025           -> 404 NOT_FOUND
     anything else   -> logger.error + 500 INTERNAL_SERVER_ERROR
```

**Evidence:** `src/config/database.ts:4-10`, `src/middlewares/error.middleware.ts:14-64`, `prisma/migrations/20260519182739_init/migration.sql:109-125`
**Confidence:** High (explicit code)

---

### Business Rule: BR-13 Tests Share the Singleton and Must Truncate All Tables

**Overview**:
The integration suite imports the same `prisma` singleton, explicitly connects before all tests, deletes all rows before each test, and disconnects after all tests.

**Detailed description**:
`tests/setup.ts` is registered as the Vitest setup file (`vitest.config.ts:10`). Its `beforeAll` calls `prisma.$connect()` (lines 4-6), making connectivity a hard prerequisite for the suite. Its `beforeEach` deletes rows in dependency order: `orderStatusHistory`, `orderItem`, `order`, `orderNumberSequence`, `product`, `customer`, `user` (lines 8-16). The order matters because of the foreign keys declared in the migration: children before parents, and `orderNumberSequence` is independent. Its `afterAll` calls `$disconnect()` (lines 18-20). Test factories (`tests/helpers/factories.ts`) insert users, customers and products through the same singleton and build the app with `buildApp({ prisma })`, and `tests/orders.test.ts` performs direct read assertions through the singleton (lines 85, 130, 150, 159).

The rule guarantees a deterministic starting state per test at the cost of destructive coupling to whatever database `DATABASE_URL` points to. There is no guard asserting `NODE_ENV === 'test'`, no database-name assertion, and no separate test DSN. Running the suite against a development or production `DATABASE_URL` would erase all data. The suite is also serialized (`fileParallelism: false`, `singleFork: true`, `vitest.config.ts:11-16`) precisely because all tests share one database and one client; parallelism would cause cross-test interference through the truncation hook.

Because the singleton is created once per Vitest fork, the in-memory process state is shared across all test files, but the module cache is reset per fork, not per file. The explicit `$connect`/`$disconnect` pair is the only lifecycle management; no per-test transaction rollback strategy is used, so each test pays the cost of seven `DELETE` statements.

**Rule workflow**:
```
vitest -> tests/setup.ts
  beforeAll  -> $connect
  beforeEach -> deleteMany(history, item, order, sequence, product, customer, user)
  test       -> factories/app requests/direct queries on same singleton
  afterAll   -> $disconnect
```

**Evidence:** `tests/setup.ts:1-20`, `tests/helpers/factories.ts:3,12,22,45,65`, `tests/orders.test.ts:9,85,130,150,159`, `vitest.config.ts:4-18`
**Confidence:** High (explicit code)

---

### Business Rule: BR-14 Seed Script Bypasses the Provider with an Independent Client

**Overview**:
`prisma/seed.ts` creates its own `PrismaClient` instead of importing the provider, applies no environment-conditional logging, and destructively resets the database.

**Detailed description**:
`prisma/seed.ts:4` executes `const prisma = new PrismaClient();` without options. It therefore does not inherit the log policy of BR-01, and it does not share the API's singleton because it runs as a separate process (`tsx --env-file=.env prisma/seed.ts`, `package.json:16`, also registered as the Prisma seed hook at `package.json:22-24`). The script deletes all rows from seven tables (`seed.ts:181-187`), then creates two users with hardcoded passwords, ten customers, twenty products and twenty-seven orders with full status history, using its own `reserveOrderNumber()` transaction (`seed.ts:165-176`). It disconnects in a `finally` block (`seed.ts:333-335`).

The deviation is architecturally reasonable for a CLI task that must run standalone, but it means there are two construction sites for the ORM in the repository and two lifecycle policies. The seed has no environment guard either: it deletes data unconditionally, and its hardcoded demo credentials (`admin123`, `operator123`, `seed.ts:6-7`) are committed. If `DATABASE_URL` points at a shared environment, the seed is destructive. The script also does not go through `src/config/env.ts` directly (it relies on `--env-file`), so it does not benefit from the fail-fast validation messages for `JWT_SECRET` and the other variables, though Prisma will fail if `DATABASE_URL` is absent.

For the purposes of this component, the rule defines the boundary: the provider is the application-runtime client, not the universal client. Any future worker, migration helper or operational script must make the same explicit choice between importing the singleton and constructing its own client.

**Rule workflow**:
```
npm run db:seed -> tsx --env-file=.env prisma/seed.ts
  -> new PrismaClient() (local to script, no log options)
  -> deleteMany on 7 tables
  -> create users/customers/products/orders/history
  -> $disconnect in finally
```

**Evidence:** `prisma/seed.ts:4,6-7,165-187,327-335`, `package.json:16,22-24`
**Confidence:** High (explicit code)

---

## 4. Component Structure

```
src/config/
├── database.ts        # PrismaClient factory + process-wide singleton (10 lines)
└── env.ts             # Zod-validated environment, fail-fast (27 lines)

Runtime consumers (production):
src/server.ts:3,7,16   # imports prisma, injects into buildApp, disconnects on shutdown

Runtime consumers (test infrastructure):
tests/setup.ts:2,5,9-15,19        # $connect, truncate, $disconnect
tests/helpers/factories.ts:3,12   # buildApp({ prisma }), fixture inserts
tests/orders.test.ts:9,85,130,150,159  # direct read assertions

Parallel construction site (dev tooling):
prisma/seed.ts:4      # independent `new PrismaClient()` (does not import provider)
```

### Component internals (symbol map)

| Symbol | Kind | Lines | Exported | External importers |
|--------|------|-------|----------|--------------------|
| `createPrismaClient` | function | 4-8 | Yes | 0 (used only at line 10) |
| `prisma` | const (singleton) | 10 | Yes | 4 files (1 production, 3 test) |
| `PrismaClient` (re-exported type) | imported symbol | 1 | No | n/a |

### Notes on organization

- The module is intentionally minimal: no class, no wrapper service, no health-check helper, no transaction helper, no graceful-shutdown helper.
- It is colocated with `env.ts` under `src/config/`, which groups infrastructure configuration; this is the only file in the folder that opens an external resource.
- There is no `src/database/`, `src/infrastructure/` or `src/persistence/` layer. The provider sits beside configuration rather than beside repositories, and repositories import the `PrismaClient` type from `@prisma/client` rather than from this module.
- Because `src/app.ts` imports only the `PrismaClient` type, the database module is not transitively loaded by `buildApp`; it is loaded by the process entry point and by tests. This keeps `app.ts` usable in principle without instantiating a client, provided a client is passed in.

---

## 5. Dependency Analysis

### Internal dependencies

```
src/config/database.ts
  └── src/config/env.ts            (runtime value import: `env`)
        └── zod                    (external)

Consumers of src/config/database.ts:
  src/server.ts            (production entry point)
  tests/setup.ts           (test lifecycle)
  tests/helpers/factories.ts (test fixtures + app under test)
  tests/orders.test.ts     (direct assertions)

Injection chain (no direct import of database.ts):
  src/server.ts -> src/app.ts (buildApp)
    -> buildControllers(prisma)
       -> UserRepository(prisma)     -> prisma.user.*
       -> CustomerRepository(prisma) -> prisma.customer.*
       -> ProductRepository(prisma)  -> prisma.product.*
       -> OrderRepository(prisma)    -> prisma.order.*
       -> OrderService(OrderRepository, prisma)
            -> prisma.$transaction
            -> tx.customer / tx.product / tx.order / tx.orderItem
               / tx.orderStatusHistory / tx.orderNumberSequence
```

### External dependencies

| Dependency | Version | Scope | Role for this component |
|------------|---------|-------|-------------------------|
| `@prisma/client` | 5.22.0 (exact) | `dependencies` | Provides `PrismaClient` constructor, generated model delegates, `Prisma` types and enums |
| `prisma` (CLI) | 5.22.0 (exact) | `devDependencies` | Generates the client from `prisma/schema.prisma`; runs migrations and seed |
| Prisma query engine binary | Bundled with `@prisma/client` 5.22.0 | Transitive | Executes queries over the MySQL wire protocol; no driver adapter is configured |
| `zod` | 3.23.8 | `dependencies` | Indirect: validates `NODE_ENV` before the log branch is evaluated (`env.ts:3-10`) |
| MySQL 8.0 | `mysql:8.0` image | Infrastructure | Target database at `DATABASE_URL` (Compose service `mysql`, port 3306) |

### Dependency characteristics

- **Exact version pinning.** `package.json:26,48` pins both Prisma packages to `5.22.0` with no range, and `package-lock.json` is committed. This makes the generated client reproducible but requires deliberate upgrades.
- **No driver adapter.** Prisma 5 with `prisma-client-js` uses the bundled Rust query engine. There is no `@prisma/adapter-*` dependency, so the connection is managed entirely inside the client binary.
- **Single internal dependency.** The only internal import is `env.ts`, which itself has only `zod`. The fan-out of this component is therefore minimal and stable.
- **No optional or peer dependencies at runtime.** Nothing is lazily required; the client is mandatory for the process to serve data.
- **Coupling to the generated artifact.** The `@prisma/client` package is useless without generation; `node_modules` is absent in the analyzed workspace, so the artifact is produced at install time by the package's postinstall hook (or manually via `prisma generate`).

---

## 6. Afferent and Efferent Coupling

Coupling is measured at file and symbol level, following the methodology of the architectural report (distinct files importing the component). External npm packages are separated from the internal graph.

### File-level coupling

| Component | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Critical |
|-----------|------------------------|------------------------|----------|
| `src/config/database.ts` (production scope) | 1 (`src/server.ts`) | 1 internal (`src/config/env.ts`) + 1 external package (`@prisma/client`) | High |
| `src/config/database.ts` (including tests) | 4 (`src/server.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `tests/orders.test.ts`) | 1 internal + 1 external | High |
| `src/config/env.ts` (dependency of this component) | 5 | 0 internal + 1 external (`zod`) | High |
| `@prisma/client` (external package, internal importers) | 13 source files + 1 test file | external | High |

### Symbol-level coupling

| Symbol | Afferent Coupling | Efferent Coupling | Critical |
|--------|-------------------|-------------------|----------|
| `prisma` (singleton export) | 4 files (1 production, 3 test) | n/a (instance) | High |
| `createPrismaClient` (factory) | 0 external files | 1 (`PrismaClient` constructor) | Low today, Medium if parameterized |
| `PrismaClient` type (via `@prisma/client`) | 12 source files use the type/constructor, 13 import the package | external | High |
| `Prisma.TransactionClient` type | 1 file (`order.service.ts:24`) | external | Medium |

### Interpretation

The component is **stable and central**: low efferent coupling (one internal dependency), low-to-moderate afferent coupling at file level, and very high transitive reach. It sits at the bottom of the dependency graph, close to MySQL, and above it sits the entire application. Instability (`Ce / (Ca + Ce)`) is low for production scope (1 / 2 = 0.5 at file level counting only the production importer; 1 / 5 = 0.2 when tests are included), which matches its role as infrastructure that should not change often.

The asymmetry worth noting is that the **factory is over-exposed and under-used**: `createPrismaClient` is exported but has zero external callers, while the singleton has all the consumers. The effective public contract of the module is therefore the `prisma` constant, not the factory.

Transitive coupling: because every repository and `OrderService` receive the singleton, a failure or configuration change here reaches all five feature modules and all 19 route handlers (plus `/health` indirectly for readiness). The architectural report classifies `src/config` with Ca=4 and Ce=0 at module level; this component is the only member of that folder that opens a resource.

---

## 7. Endpoints

Not applicable. The component exposes no REST, GraphQL, gRPC or other network endpoint. It is an in-process data-access provider consumed by `src/server.ts` and by test infrastructure. The only HTTP surface that reflects its state indirectly is `GET /health`, which returns a static payload and does not query the database (`src/app.ts:62-64`).

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 (`oms` database) | External stateful service | Primary datastore for users, customers, products, orders, order items, status history and order-number sequence | MySQL wire protocol over TCP (`DATABASE_URL`, host port 3306) | Relational rows mapped to generated TypeScript types; `JSON` column for `customers.address` | Errors propagate as Prisma errors; `P2002`/`P2025` mapped to 409/404, everything else to 500 (`error.middleware.ts:37-64`) |
| `@prisma/client` runtime | In-process library | Typed query builder, relation loading, batch and interactive transactions, connection pooling | In-process function calls into the query engine binary | Generated TypeScript types (`Prisma.*` inputs, model outputs, enums) | `PrismaClientKnownRequestError` / `PrismaClientInitializationError` / unknown errors propagated unwrapped |
| Prisma query engine | Bundled binary process/library | Compiles the Prisma query AST and executes SQL against MySQL | Internal engine protocol; SQL to MySQL | SQL + result sets | Engine connection/timeouts surface as Prisma initialization or unknown request errors |
| `DATABASE_URL` | Configuration | Selects the runtime database and DSN parameters (credentials, host, port, database, optional pool/TLS parameters) | Environment variable read by the Prisma schema (`env("DATABASE_URL")`) | Connection string (`mysql://user:pass@host:port/db`) | Missing/empty value stops the process via `env.ts` (`zod` `min(1)`); invalid DSN fails on first query |
| `SHADOW_DATABASE_URL` | Configuration (tooling) | Schema-diff database for `prisma migrate dev` | Environment variable read by Prisma CLI only | Connection string | Not validated by `src/config/env.ts`; failure occurs when running `npm run db:migrate` |
| `prisma` CLI 5.22.0 | Development tooling | Generate client, apply migrations, reset and seed | CLI | `schema.prisma`, SQL migrations | Exit codes only; wired through `db:migrate`, `db:reset`, `db:seed` scripts (`package.json:14-16`) |
| Docker Compose `mysql` service | Local infrastructure | Provides MySQL 8.0 with `utf8mb4`, named volume and healthcheck | Docker network / published port 3306 | n/a | Compose healthcheck (`mysqladmin ping`) is independent of application readiness |
| Test harness | Internal integration | Connect, truncate and disconnect through the singleton | In-process | SQL `DELETE` per table | No environment guard; destructive on whatever `DATABASE_URL` points to |

### Integration observations

- The application has exactly one external stateful dependency: MySQL. Everything else in this component's integration list is in-process or configuration.
- No connection is opened at construction; the first query triggers TCP connect, authentication and pool creation.
- No TLS parameter is present in `.env.example`; MySQL traffic is unencrypted by default in this setup.
- No retry/circuit-breaker/timeout policy is configured at the client level. Availability behaviour is whatever Prisma and MySQL provide by default.
- The Prisma query engine is a native binary shipped with the client; there is no driver adapter, so the component cannot be pointed at a serverless HTTP driver without a client/version change.
- `SHADOW_DATABASE_URL` is declared in the schema but is not provisioned by `docker-compose.yml` and not validated by `env.ts`; the integration is tooling-only and fragile on a fresh machine.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Singleton (module-level instance) | `export const prisma = createPrismaClient()` | `src/config/database.ts:10` | One client and one pool per process; global lifecycle |
| Factory Function | `createPrismaClient(): PrismaClient` | `src/config/database.ts:4-8` | Isolate construction and configuration; expose a seam for future parameterization |
| Constructor Dependency Injection | Repositories and `OrderService` receive `PrismaClient`; composition root wires them | `src/app.ts:26-53`; repository constructors | Inversion of the client dependency without a DI container; testability of consumers |
| Composition Root | `buildControllers(prisma)` and `buildApp({ prisma })` | `src/app.ts:26-76` | Centralize instantiation order and lifetime |
| Lazy Initialization | Prisma opens connections on first use; no `$connect` in production code | `src/config/database.ts:5-7`; `src/server.ts:6-11` | Avoid startup coupling to database availability |
| Configuration Object (validated) | Zod-validated `env` consumed by the provider | `src/config/env.ts:3-27` | Fail-fast configuration and typed access |
| Fail-Fast Startup | `process.exit(1)` on invalid environment | `src/config/env.ts:16-23` | Prevent partially configured processes from serving |
| Unit of Work (consumer-applied) | `$transaction(async (tx) => ...)` and `$transaction([...])` | `src/modules/orders/order.service.ts:58,131`; `*.repository.ts` list methods | Atomic multi-statement operations; transactional client scope |
| Repository (consumer pattern) | `UserRepository`, `CustomerRepository`, `ProductRepository`, `OrderRepository` | `src/modules/*/*.repository.ts` | Encapsulate query construction; provider supplies the handle |
| Shared Kernel (error/type contract) | Generated Prisma types + `PrismaClientKnownRequestError` mapping | `prisma/schema.prisma`; `src/middlewares/error.middleware.ts:37-54` | Common error and type vocabulary across layers |

### Architectural decisions encoded by this component

1. **ORM-first persistence.** There is no handwritten SQL layer; all access is through generated model delegates. Raw queries are absent from the entire repository (no `$queryRaw`/`$executeRaw` usages found).
2. **One database, one client, one pool.** No read replica, tenant routing, sharding or multi-datasource abstraction is anticipated.
3. **Logging is environment-driven, not level-driven.** The existing `LOG_LEVEL` variable does not influence Prisma; `NODE_ENV` does.
4. **Lifecycle ownership belongs to the entry point, not to the provider.** The module exports a handle; `src/server.ts` decides when to disconnect.
5. **No cross-cutting ORM middlewares.** No soft-delete, audit, tenant filter, query timing or retry extension is installed. This keeps behaviour explicit in services and repositories.
6. **Errors are not wrapped.** The provider treats Prisma errors as the canonical error type and delegates HTTP translation to middleware.

---

## 10. Technical Debt & Risks

### 10.1 Risk register

| Risk Level | Component Area | Issue | Impact | Evidence |
|------------|----------------|-------|--------|----------|
| High | Module initialization | `PrismaClient` is instantiated at module import time, so importing the provider has the side effect of creating a client and (transitively) validating/exiting on environment | Any tool importing the module inherits process-exit behaviour and client creation; the future webhook worker cannot import it safely without sharing process state | `src/config/database.ts:10`, `src/config/env.ts:16-23` |
| High | Startup / readiness | No connection probe at startup and `/health` does not query the database | Traffic can be routed to a process that cannot reach MySQL; failures appear as HTTP 500 instead of failing the deploy | `src/server.ts:6-11`, `src/app.ts:62-64` |
| High | Shutdown | `server.close()` is not awaited before `process.exit(0)`; `$disconnect()` runs while HTTP handlers may still be in flight | In-flight requests and open transactions can be terminated; data-affecting operations may be cut mid-flight | `src/server.ts:13-21` |
| High | Error mapping | Only `P2002` and `P2025` are translated; `P2003` (FK), `P2024` (pool timeout), `P2034` (deadlock), `P1001`/`P1002` (connectivity) fall through to 500 | Business-meaningful conditions are reported as server faults; availability issues look like bugs; clients cannot react correctly | `src/middlewares/error.middleware.ts:37-64` |
| High | Dependency lifecycle | `@prisma/client`/`prisma` 5.22.0 are two majors behind (7.10.1); `prisma-client-js` generator is deprecated in Prisma 7 | Upgrade requires changes to this module, every `@prisma/client` import and the generator, plus a driver adapter/Accelerate URL | `package.json:26,48`, `prisma/schema.prisma:1-3`, dependency audit |
| Medium | Build reproducibility | `build` runs only `tsc`; no `prisma generate` in any script; `node_modules` absent in the workspace | A schema change without regeneration yields stale types; fresh checkouts depend on the package postinstall hook | `package.json:12`, `prisma/schema.prisma:1-3` |
| Medium | Transaction policy | No `transactionOptions` (isolation, maxWait, timeout) configured; consumers rely on defaults (5 s interactive timeout) | Long per-item stock loops may time out; deadlocks are not retried; isolation level is implicit | `src/config/database.ts:5-7`, `src/modules/orders/order.service.ts:204-243` |
| Medium | Pool configuration | No explicit `connection_limit`, `pool_timeout` or `connect_timeout`; pool size is the Prisma default derived from CPU count | Capacity is environment-dependent and undocumented; pool exhaustion surfaces as 500 with no metric | `src/config/database.ts:5-7`, `.env.example:5` |
| Medium | Observability | Log channels are limited to `warn`/`error`, and `warn` is disabled outside development; no query logging, slow-query logging or pool metrics | Production loses connection warnings and there is no SQL-level diagnosis path; `LOG_LEVEL` is ignored by Prisma | `src/config/database.ts:6`, `src/config/env.ts:6` |
| Medium | Multi-datasource capability | Factory takes no arguments; singleton bound to one `DATABASE_URL` | Cannot run isolated test DB, read replica, tenant split or a separate worker DB through the provider | `src/config/database.ts:4-10` |
| Medium | Test safety | Test setup truncates all tables through the configured `DATABASE_URL` with no environment/database guard | Running tests against a non-test database destroys data | `tests/setup.ts:4-16` |
| Medium | Seed safety and duplication | Seed uses a second client construction site, hardcoded credentials, destructive deleteMany with no guard | Duplicate lifecycle/logging policy; accidental data loss; committed demo credentials | `prisma/seed.ts:4,6-7,181-187` |
| Medium | Security: transport | No TLS/`sslmode` parameter in the DSN guidance; MySQL traffic unencrypted by default | Credentials and PII in transit are exposed on untrusted networks | `.env.example:5-6`, `docker-compose.yml:11-12` |
| Medium | Security: credentials | Example credentials and Compose defaults (`oms_password`, `root_password`) plus published port 3306 | Credential reuse and direct database exposure in non-hardened environments | `.env.example:5-14`, `docker-compose.yml:7-12` |
| Low | Type annotation redundancy | `export const prisma: PrismaClient = createPrismaClient()` repeats the inferred type | No functional impact; style only | `src/config/database.ts:10` |
| Low | Unused export surface | `createPrismaClient` is exported but imported nowhere | Misleading contract; the factory's intended seam is unrealized | `src/config/database.ts:4` |
| Low | No raw-query escape hatch usage | No `$queryRaw` anywhere; readiness/health probes would need one | Readiness checks would have to be added at the consumer level using the client | `src/app.ts:62-64` |

### 10.2 Security posture relevant to this component

- **SQL injection:** Prisma generates parameterized queries for all model operations; no string-concatenated SQL exists in the repository. This is a positive control attributable to the provider choice.
- **Credential handling:** The DSN carries the database username and password in `DATABASE_URL`. `.env` is gitignored (`.gitignore` lists `.env`), but `.env.example` documents weak samples and Compose supplies defaults. There is no secret manager or vault integration.
- **Least privilege:** Compose creates `oms_user` as the application user for the `oms` database. The migration user and `root` (used by the shadow database URL) are separate, but the application DSN has full DML/DDL rights on its schema by default.
- **Transport:** No TLS enforcement. MySQL 8.0 supports TLS, but the DSN and Compose configuration do not require it.
- **Information disclosure:** Prisma error `meta` can include constraint and column names. The middleware exposes the target of `P2002` in the response message (`error.middleware.ts:39-44`). Other errors are masked with a generic message and logged server-side with the request id.
- **Data at rest:** Customer PII (name, email, phone, document, address) is stored plaintext in MySQL. This is a schema/domain concern rather than a provider concern, but the provider is the access path for all of it.
- **Logging hygiene:** Prisma logs are limited to warnings/errors and do not include query parameters; application-level Pino redaction covers credentials paths. The combination reduces accidental secret leakage in logs.

### 10.3 Resilience and error-handling assessment

- **Atomicity:** Provided correctly; all multi-table writes use `$transaction`.
- **Concurrency control:** Not provided by the client configuration; no row locking, optimistic version column, isolation override or retry policy. Consumer-level read-check-write sequences are exposed to races.
- **Connection resilience:** No retry, no circuit breaker, no pool timeout tuning, no failover DSN. A database restart breaks in-flight requests and subsequent requests until the driver reconnects.
- **Graceful degradation:** None. Every feature depends on the single client; there is no cache or fallback path.
- **Graceful shutdown:** Partial (Prisma disconnect only; HTTP drain not awaited).
- **Observability of failures:** Errors are logged with method, path and request id, but database-specific context (Prisma code, pool state) is only visible in the logged error object.

---

## 11. Test Coverage Analysis

### 11.1 Test assets and locations

| Test asset | Path | Relationship to this component |
|------------|------|--------------------------------|
| Global setup | `tests/setup.ts` | Imports `prisma`; `$connect` (line 5), seven `deleteMany` truncations (lines 9-15), `$disconnect` (line 19) |
| Factories | `tests/helpers/factories.ts` | Imports `prisma`; builds app with `buildApp({ prisma })` (line 12); inserts user (22), customer (45), product (65) |
| Auth integration suite | `tests/auth.test.ts` | 7 tests; no direct import, but every test runs through the setup and factories, therefore through the singleton |
| Orders integration suite | `tests/orders.test.ts` | 8 tests; imports `prisma` (line 9) and asserts product stock directly (lines 85, 130, 150, 159) |
| Test runner config | `vitest.config.ts` | Node environment, `tests/setup.ts` as setup file, 30 s timeouts, serial single fork |
| Seed script (not a test) | `prisma/seed.ts` | Independent client; exercises all models and the sequence transaction |

No unit-test files exist for `src/config/database.ts`, and no test file mocks or stubs the client. All tests are integration tests against a real MySQL instance.

### 11.2 Coverage by behaviour

| Behaviour / unit | Unit Tests | Integration Tests | Estimated Coverage | Test Quality |
|------------------|------------|-------------------|--------------------|--------------|
| Module import and singleton creation | 0 | 15 tests (all, via setup/factories/app bootstrap) | Statement ~100% (every line executed) | No assertion on identity or reuse |
| Environment-conditional log branch (`development` vs other) | 0 | 0 (test run uses `NODE_ENV=test`, so only the `['error']` branch executes) | Branch ~50% of the ternary; `development` branch unexecuted | No assertion on `log` option or emitted logs |
| `createPrismaClient()` factory | 0 | Indirect through module import | Statement 100% | No direct invocation or option assertion |
| `$connect` / `$disconnect` lifecycle | 0 | `tests/setup.ts:4-6,18-20` | Happy path ~100%; failure path 0% | Verifies connectivity implicitly; no assertion on disconnect behaviour |
| CRUD through model delegates | 0 | 7 auth + 8 orders tests through repositories/`OrderService` | High for `user`, `customer`, `product`, `order`, `orderItem`, `orderStatusHistory`, `orderNumberSequence` (all touched) | Good end-to-end assertions on business outcomes; no direct provider-level assertions |
| Direct client reads (test assertions) | 0 | `tests/orders.test.ts:85,130,150,159` | Exercises `product.findUnique` after mutations | Adequate for stock verification |
| Batch transactions (`$transaction([...])`) | 0 | Exercise via list endpoints (customers/products/orders list routes) | Indirect; list paths covered in orders test only | Partial |
| Interactive transactions | 0 | `tests/orders.test.ts` create/status/delete flows | High for success paths | No rollback, deadlock, timeout or concurrent-execution test |
| Error mapping from Prisma codes | 0 | `P2002` exercised indirectly by duplicate-email registration (`tests/auth.test.ts:19-28`) and duplicate SKU paths (not tested) | `P2002` partially; `P2025`, `P2003`, `P2034`, `P2024` untested | Unknown codes degrade to 500; no tests assert this |
| Startup fail-fast on missing `DATABASE_URL` | 0 | 0 | 0% | Untested; would require process spawning |
| Shutdown signal handling | 0 | 0 | 0% | Untested; no signal simulation |
| Database unreachable / reconnection | 0 | 0 | 0% | Untested |
| Seed script | 0 | 0 (requires `npm run db:seed`) | 0% under test | Manual only |

### 11.3 Coverage tooling and quality assessment

- **No coverage provider is installed.** `package.json` does not include `@vitest/coverage-v8` or `@vitest/coverage-istanbul`, and no `coverage` block exists in `vitest.config.ts`. Therefore the project cannot produce a coverage report or enforce a threshold; the percentages above are analytical estimates based on static execution paths, not measured values.
- **Statement coverage is effectively full for the provider file** because every test run imports it and executes lines 1, 2, 4, 5, 6 and 10. Line 6's ternary is evaluated but only one side is taken.
- **Behavioural coverage is near zero for the provider's own policies.** No test asserts that the development branch configures `['warn','error']`, that a single instance is reused, that `$disconnect` is called on shutdown, or that initialization is lazy.
- **Test safety is a concern.** `tests/setup.ts:8-16` deletes all rows in seven tables with no `NODE_ENV` or database-name guard. The suite requires an externally provisioned MySQL with migrations applied; `docker-compose.yml` provides the server but not the schema or the `oms_shadow` database.
- **Serialization.** `fileParallelism: false` and `singleFork: true` (`vitest.config.ts:11-16`) confirm that the shared singleton and shared database prevent parallel execution.
- **Quality strengths.** The existing integration tests assert real outcomes (HTTP status codes, totals, stock quantities, history sequence, pagination totals) and therefore exercise the provider under realistic multi-statement transactions. The gaps are concentrated in infrastructure failure modes and configuration branches, which are exactly the areas this component owns.

### 11.4 Recommended test targets (observations only, not implementation guidance)

- The `development` log branch.
- Process startup with a missing/empty `DATABASE_URL` (fail-fast contract).
- Singleton identity across repeated imports.
- `$disconnect` invocation on `SIGINT`/`SIGTERM` and ordering relative to HTTP drain.
- Prisma error codes `P2003`, `P2025`, `P2034`, `P2024`, `P1001` and their HTTP mapping.
- Transaction rollback behaviour when a statement inside `$transaction` fails.

---

## Appendix A. Assumptions, Ambiguities and Confidence Levels

| Topic | Statement | Confidence |
|-------|-----------|------------|
| BR-01 log policy | Explicit ternary on `NODE_ENV`; only `warn`/`error` or `error` | High |
| BR-02 singleton | Explicit module-level export; ESM caching guarantees one evaluation | High |
| BR-03 factory-only construction in `src/` | Verified by search; seed is a documented exception | High |
| BR-04 fixed options | Explicit zero-argument factory signature | High |
| BR-05 fail-fast | Explicit `process.exit(1)` in `env.ts` imported at `database.ts:2` | High |
| BR-06 datasource | Explicit in `schema.prisma`; URL content varies by environment | High |
| BR-07 lazy connect | No `$connect` in production code; Prisma documented behaviour | High |
| BR-08 shutdown | Explicit signal handlers; ordering weakness observed | High |
| BR-09 shared pool | Consequence of singleton; exact pool size not configured | High / Medium (size) |
| BR-10 transaction defaults | No options configured; numeric defaults come from Prisma 5/MySQL documentation, not repository files | Medium-High |
| BR-11 generated contract | Explicit generator and imports; generation step not automated in scripts | High |
| BR-12 error propagation | Explicit absence of handling; mapping table explicit | High |
| BR-13 test lifecycle | Explicit setup file | High |
| BR-14 seed deviation | Explicit separate `new PrismaClient()` | High |
| Actual runtime behaviour under load | Not measured; no execution performed | Not assessed |
| Coverage percentages | Analytical estimates; no coverage provider installed | Low (indicative only) |

### Ambiguities found in the code

1. **Why `createPrismaClient` is exported** is not documented; it has no external callers. It may be a deliberate seam for tests/workers or a leftover abstraction.
2. **Whether `NODE_ENV=test` intentionally suppresses Prisma warnings** is not stated. The ternary groups `test` with `production`.
3. **Whether the absence of a startup `$connect` is intentional** (lazy start) or an omission is not documented; the only comment in the component is absent entirely.
4. **Whether the seed's independent client is intentional** (standalone CLI) or an oversight is not documented; it is the only second construction site in the repository.
5. **The intended production topology** (single host, container, managed MySQL) is not defined, so the adequacy of the default pool and the absence of TLS cannot be judged against a target environment.
6. **`SHADOW_DATABASE_URL` is declared but not validated or provisioned**, so its operational contract is implicit.

---

*This report is analysis-only. No project source, configuration, schema, migration or test file was modified. The only file created is this report under `docs/agents/component-deep-analyzer/`.*
