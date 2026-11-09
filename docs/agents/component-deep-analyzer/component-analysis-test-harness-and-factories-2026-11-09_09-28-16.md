# Component Deep Analysis Report

**Component:** Test Harness and Factories
**Primary files:** `tests/setup.ts`, `tests/helpers/factories.ts`, `vitest.config.ts`, `tests/auth.test.ts`, `tests/orders.test.ts`
**Project:** `mba-ia-desafio-design-docs-com-ia` (Order Management System REST API)
**Analysis date:** 2026-11-09 09:28:16
**Scope:** Entire project root, excluding `node_modules`, `.git`, `dist`, `coverage`
**Analyzed revision:** `e7f6311` (working tree)
**Analysis mode:** Static code analysis only. No project files were modified, no code was executed, and no tests were run.

---

## 1. Executive Summary

The Test Harness and Factories component is the verification infrastructure of the repository. It is composed of one Vitest configuration file (`vitest.config.ts`, 19 lines), one lifecycle setup file (`tests/setup.ts`, 20 lines), one shared factory/helper module (`tests/helpers/factories.ts`, 83 lines), and two integration suites (`tests/auth.test.ts`, 84 lines; `tests/orders.test.ts`, 219 lines). The five files total 425 lines, of which 406 lines are test code and 19 lines are runner configuration (the architectural report's source-volume line of "4 files / 406 lines in `tests`" excludes `vitest.config.ts`).

The harness is deliberately thin and fully integration-oriented. There are no mocks, stubs, fakes, spies, in-memory databases or dependency overrides. Every test drives the real Express application in-process through Supertest, which executes the real middleware chain, controllers, services, Prisma repositories and interactive transactions against a real MySQL 8.0 database. The only test double is the database lifecycle itself: all seven application tables are emptied before every test, and the test database is whatever `DATABASE_URL` points to.

There are exactly two suites and **15 tests**: 7 in `tests/auth.test.ts` and 8 in `tests/orders.test.ts`. The architectural report refers to "10 integration tests" in `tests/orders.test.ts`; the file actually contains eight `it` blocks (`tests/orders.test.ts:12, 42, 59, 89, 109, 134, 163, 197`). There are no unit tests and no meta-tests for the harness itself.

Key findings:

- **Deterministic isolation is achieved by serialization plus truncation, not by transaction rollback or schema-per-run.** `fileParallelism: false`, `pool: 'forks'` and `poolOptions.forks.singleFork: true` (`vitest.config.ts:11-17`) force all test files through one child process, so the destructive `deleteMany` sequence in `tests/setup.ts:8-16` cannot race across files. Tests inside a file run sequentially because no `.concurrent` modifier is used.
- **The cleanup lifecycle is per test file, not once per run.** `setupFiles` are executed before each test file (`vitest.config.ts:10`), so `beforeAll` connect, `beforeEach` truncate and `afterAll` disconnect run once per file. The database is left populated after the final test because there is no trailing cleanup in `afterAll` (`tests/setup.ts:18-20` only disconnects).
- **The truncation is destructive and unguarded.** `tests/setup.ts:8-16` deletes all rows from `order_status_history`, `order_items`, `orders`, `order_number_sequence`, `products`, `customers` and `users` on the database selected by `DATABASE_URL`. There is no `NODE_ENV`, database-name, hostname or allow-list check. This is the highest-severity safety risk in the component.
- **The harness depends on externally supplied environment variables.** `vitest.config.ts` does not use `loadEnv`, the `test.env` option or `dotenv`, and `package.json:17` runs `vitest run` without `--env-file=.env`. `src/config/env.ts:7-8` requires `DATABASE_URL` and `JWT_SECRET`, and `src/config/env.ts:15-23` calls `process.exit(1)` when either is missing. A fresh checkout with only `.env.example` present cannot import the harness successfully unless those variables are exported by the shell or CI.
- **Factories write directly through Prisma and bypass all domain validation.** `createTestUser`, `createTestCustomer` and `createTestProduct` (`tests/helpers/factories.ts:17-74`) insert rows directly, so arrangement can produce states that the HTTP API would reject. This keeps setup fast and focused but means the factories do not verify production rules and are coupled to the Prisma schema.
- **The app is a lazy module-level singleton reused for the whole module lifetime.** `getTestApp()` (`tests/helpers/factories.ts:10-15`) builds the Express app once per module instance with the production Prisma singleton and returns the cached instance. There is no reset hook and no injection seam beyond `buildApp({ prisma })`.
- **Coverage is partial and unmeasured.** Customers, products, the admin-only user lookup, `/health`, server bootstrap, environment validation, logger behavior and many order branches have no tests. No coverage provider, coverage script or threshold exists in `vitest.config.ts` or `package.json`, so no quantitative coverage figure can be reported from the repository.
- **The suites verify real domain behavior rather than harness behavior.** Auth tests assert public projections, status codes and error codes; order tests assert server-computed totals, PENDING initialization, order-number format, stock debit/replenish, transition rejection, insufficient-stock rollback, filtering and delete guards, including direct database reads for stock verification.

Component boundary used in this analysis: `vitest.config.ts`, `tests/setup.ts`, `tests/helpers/factories.ts` and the two consumer suites. Production files (`src/**`, `prisma/**`) are examined only as dependencies and as the API surface exercised by the suites; their internal business logic is not re-analyzed here.

---

## 2. Data Flow Analysis

### 2.1 Run-level lifecycle flow

```
1. `npm test` invokes `vitest run` (package.json:17).
2. Vitest sets NODE_ENV=test when it is not already set, loads vitest.config.ts.
3. The include pattern `tests/**/*.test.ts` selects only the two suite files;
   setup.ts and helpers/factories.ts are not collected as test files (vitest.config.ts:7).
4. For each test file, Vitest executes tests/setup.ts before the file's tests (vitest.config.ts:10).
5. setup.ts imports the process-wide Prisma singleton from src/config/database.ts:10.
   That import first evaluates src/config/env.ts, which validates DATABASE_URL and JWT_SECRET
   and terminates the process on failure (src/config/env.ts:15-23).
6. beforeAll: prisma.$connect() opens the connection pool (tests/setup.ts:4-6).
7. beforeEach: seven sequential deleteMany operations empty all application tables
   (tests/setup.ts:8-16).
8. The test file executes its `it` blocks sequentially in one fork.
9. afterAll: prisma.$disconnect() releases the pool (tests/setup.ts:18-20).
10. The next test file repeats steps 4-9 in the same single fork.
11. No afterAll truncation occurs, so the rows created by the last executed test remain
    in the database after the run.
```

### 2.2 Per-test execution flow

```
1. Test calls getTestApp() (factory).
   -> first call: buildApp({ prisma }) (src/app.ts:55-75)
   -> buildControllers(prisma) constructs repositories, services and controllers (src/app.ts:26-53)
   -> Express app is cached in module state (tests/helpers/factories.ts:8-15)
2. Test arranges state:
   - createTestUser(): bcrypt.hash(password, 10) -> prisma.user.create() -> { user, password }
   - createTestCustomer(): prisma.customer.create() -> Customer
   - createTestProduct(): prisma.product.create() -> Product
   - bootstrapAuthenticatedUser(): createTestUser() -> POST /api/v1/auth/login -> accessToken
3. Test drives the API with Supertest:
   request(app).post(...)/get(...)/patch(...)/delete(...)
   -> requestLogger middleware: X-Request-Id + finish log (src/middlewares/request-logger.middleware.ts:5-28)
   -> express.json 1 MB parser (src/app.ts:59)
   -> /api/v1 router (src/routes/index.ts:24-32)
   -> module router middleware: authenticate -> validate -> controller
   -> controller -> service -> repository or prisma.$transaction
   -> MySQL 8.0
4. Controller serializes JSON; error middleware serializes AppError/Zod/Prisma failures
   (src/middlewares/error.middleware.ts:14-65).
5. Supertest parses the response; assertions check status, body shape, error codes and,
   in order tests, re-read rows through prisma to verify side effects (tests/orders.test.ts:85-86,
   130-131, 150-151, 159-160).
6. No afterEach cleanup runs; the next test's beforeEach truncation removes the rows.
```

### 2.3 API surface exercised by the suites

The harness exposes no endpoints of its own, so no endpoint section is included in this report. The following production endpoints are exercised by the suites:

| Endpoint | Method | Suite | Purpose |
|----------|--------|-------|---------|
| `/api/v1/auth/register` | POST | `tests/auth.test.ts:9,22,32` | Registration happy path, duplicate email, invalid payload |
| `/api/v1/auth/login` | POST | `tests/auth.test.ts:44,57`; `tests/helpers/factories.ts:35` | Successful login, wrong password, token acquisition for every authenticated test |
| `/api/v1/auth/me` | GET | `tests/auth.test.ts:70,81` | Authenticated identity, missing token |
| `/api/v1/orders` | POST | `tests/orders.test.ts:19,47,65,95,115,140,171,203` | Order creation, pricing, persistence, sequence |
| `/api/v1/orders/:id/status` | PATCH | `tests/orders.test.ts:73,100,121,145,153,178,207` | Transitions, stock movement, audit history |
| `/api/v1/orders` | GET | `tests/orders.test.ts:183,190` | Status filtering and pagination envelope |
| `/api/v1/orders/:id` | DELETE | `tests/orders.test.ts:212` | Delete guard for non-PENDING/non-CANCELLED orders |

### 2.4 Input/output transformations

| Stage | Input | Operation | Output |
|-------|-------|-----------|--------|
| `createTestUser` | Optional `{ email, password, name, role }` | Per-field defaulting, bcrypt hash (cost 10), direct insert | `{ user: User, password: string }` |
| `createTestCustomer` | Optional `{ name, email, phone, document }` | Per-field defaulting, fixed address literal, direct insert | `Customer` |
| `createTestProduct` | Optional `{ sku, name, priceCents, stockQuantity, active }` | Per-field defaulting, direct insert | `Product` |
| `loginAndGetToken` | `email`, `password` | In-process POST to login, status check, body extraction | `string` access token |
| `bootstrapAuthenticatedUser` | Optional role | User factory + login helper composition | `{ user, token }` |
| `getTestApp` | none | Lazy construction of the full Express app | Cached `Express` instance |
| Setup `beforeEach` | none | Seven unqualified `deleteMany` calls | Empty application tables |
| Supertest request | HTTP method/path/body/token | Real middleware/controller/service/DB execution | HTTP status + JSON body |

### 2.5 Failure and edge paths

- **Invalid environment:** `src/config/env.ts:15-23` prints the Zod issues and calls `process.exit(1)` during module evaluation, before any hook runs. This terminates the Vitest worker rather than producing a normal test failure.
- **Database unreachable:** `prisma.$connect()` in `beforeAll` rejects; the file's tests fail at the hook boundary. There is no retry, wait-for-database loop or readiness probe. The configured `hookTimeout` of 30 s (`vitest.config.ts:9`) bounds hook execution.
- **Truncation failure:** the seven deletes are not wrapped in a transaction. A failure mid-sequence leaves partially removed data; no compensating logic exists. Subsequent `beforeEach` runs attempt the full sequence again.
- **Fixture collision:** generated emails/SKUs can collide in theory; the database unique constraints then raise a Prisma error from the factory before the request is sent. No retry or regeneration occurs.
- **Login failure inside a helper:** `loginAndGetToken` throws `Error("login failed: <status> <body>")` for any non-200 response (`tests/helpers/factories.ts:36-38`). A 200 response with a missing `tokens.accessToken` returns `undefined` cast to `string`; the failure then appears later as a 401 in the test, not at the helper.
- **HTTP business failure:** `AppError` subclasses become their mapped status and `{ error: { code, message, details? } }`; Zod failures become `400 VALIDATION_ERROR`; unmatched errors become `500 INTERNAL_SERVER_ERROR` after logging (`src/middlewares/error.middleware.ts:14-65`).
- **Test timeout:** any test exceeding 30 s is failed by the runner (`vitest.config.ts:8`). A hanging database operation consumes the full timeout.
- **Cleanup on abrupt termination:** if the process is killed (SIGKILL, CI cancellation), `afterAll` does not execute and the connection pool is abandoned by the OS; the database retains all rows from the run.

---

## 3. Business Rules & Logic

### Overview of the business rules

In this component, "business rules" are the enforceable invariants of the testing infrastructure: lifecycle constraints, isolation guarantees, fixture semantics and the verification contracts the suites impose on the system under test.

| ID | Rule Type | Rule Description | Location | Confidence |
|----|-----------|------------------|----------|------------|
| TR-01 | Lifecycle | Connect once before each test file and disconnect once after it | `tests/setup.ts:4-6, 18-20` | High (explicit) |
| TR-02 | Isolation | Delete all rows from all seven tables before every test | `tests/setup.ts:8-16` | High (explicit) |
| TR-03 | Isolation | Truncation must follow a fixed child-to-parent order to satisfy foreign keys | `tests/setup.ts:9-15` | High (order explicit; rationale inferred) |
| TR-04 | Execution | Test files run serially in a single fork; no parallel truncation is possible | `vitest.config.ts:11-17` | High (explicit) |
| TR-05 | Discovery/Configuration | Only `tests/**/*.test.ts` are collected; the run uses the Node environment and explicit Vitest imports | `vitest.config.ts:5-10`; `tests/auth.test.ts:1`; `tests/orders.test.ts:1` | High (explicit) |
| TR-06 | Configuration | Each test and hook has a 30-second execution budget | `vitest.config.ts:8-9` | High (explicit) |
| TR-07 | Fixture | The Express app is built once and reused from module state, wired to the production Prisma singleton | `tests/helpers/factories.ts:8-15` | High (explicit); cross-file lifetime ambiguous |
| TR-08 | Integration | All tests use the real app, real Prisma and real MySQL; no test doubles are permitted | All five files; no mocks present | High (absence verified) |
| TR-09 | Fixture | Factory overrides replace individual default fields via nullish coalescing; defaults apply otherwise | `tests/helpers/factories.ts:20-30, 45-59, 65-73` | High (explicit) |
| TR-10 | Fixture | Default identities for users, customers and products must be unique per creation | `tests/helpers/factories.ts:24, 48, 67` | High (explicit); uniqueness probabilistic |
| TR-11 | Fixture/Auth | Test users receive `password123` hashed with bcrypt cost 10, role `OPERATOR` by default, and tokens are obtained only through the real login endpoint | `tests/helpers/factories.ts:20-27, 33-40, 76-83` | High (explicit) |
| TR-12 | Isolation | Every test arranges its own aggregate graph; no shared fixtures or seed data are used | `tests/auth.test.ts`; `tests/orders.test.ts` | High (observed) |
| TR-13 | Verification | Auth suite oracle: public projections, status codes and error codes for register/login/me | `tests/auth.test.ts:6-83` | High (explicit assertions) |
| TR-14 | Verification | Orders suite oracle: server-side totals, PENDING initialization, numbering, stock movement, transition guards, filtering, delete guard | `tests/orders.test.ts:12-218` | High (explicit assertions) |

---

### Business Rule TR-01: Per-file connection and disconnection lifecycle

**Overview:**
The harness opens the database connection before a test file's tests and closes it after they finish. The lifecycle is registered through Vitest hooks in the shared setup file, so every collected test file inherits it.

**Detailed description:**
`tests/setup.ts:4-6` registers `beforeAll(async () => { await prisma.$connect(); })`, and `tests/setup.ts:18-20` registers `afterAll(async () => { await prisma.$disconnect(); })`. The imported `prisma` is the process-wide singleton created at module evaluation in `src/config/database.ts:10` through `createPrismaClient()`. Because Vitest executes setup files before each test file (documented behavior of the `setupFiles` option), the connect/disconnect pair is applied per test file rather than once per run. This detail matters because the repository contains two suites: the pool is opened and closed twice in a full run when isolation creates a fresh module graph for each file.

The connect call is eager: importing the database module already instantiates `PrismaClient`, but the physical pool is only opened by `$connect()`. There is no retry, backoff, health probe or wait-for-MySQL loop. If the database is not accepting connections, the hook rejects and the suite fails; the 30-second `hookTimeout` bounds how long a hanging attempt can consume. There is also no ping or schema/migration check, so a reachable database with missing tables passes `beforeAll` and fails later inside the first test or the first truncation.

The disconnection path is minimal and not protected by `try/finally` or a timeout. It runs only when the file reaches its `afterAll` phase; an abrupt process termination, a runner crash or a CI cancellation leaves the pool to be reclaimed by process exit. The lifecycle does not delete data on the way out (see TR-02), so disconnection and data cleanup are separate concerns with different guarantees.

**Rule workflow:**
```
setup.ts evaluated
  -> import src/config/database.ts (client created)
  -> import src/config/env.ts (validated, may exit)
beforeAll -> prisma.$connect()
  -> test 1, test 2, ... test N (with beforeEach truncation)
afterAll -> prisma.$disconnect()
```

---

### Business Rule TR-02: Full-table truncation before every test

**Overview:**
Before each individual test, the harness removes every row from every application table so that each test starts from a known empty state.

**Detailed description:**
The `beforeEach` block at `tests/setup.ts:8-16` calls `deleteMany()` without a `where` clause on seven Prisma models in this exact order: `orderStatusHistory`, `orderItem`, `order`, `orderNumberSequence`, `product`, `customer`, `user`. Because each call is a separate statement, the cleanup is seven auto-committed DELETE statements rather than a single transaction or a `TRUNCATE` of all tables. The choice avoids foreign-key disabling and table-lock semantics, but it means the intermediate states are visible to other connections and a crash between statements leaves a partial cleanup.

The rule is "before each test", not "after each test". Therefore the rows produced by one test are visible to the next test only until that next test's `beforeEach` completes. There is no `afterEach` cleanup and no final truncation in `afterAll`. Consequently the database retains the data of the last executed test after the run and between the two test files; the first test of the next file is protected only because its own `beforeEach` runs first. Any external observer of the database (a developer using the same schema, a manual query, a concurrently running job) sees committed test writes during and after the run.

The cleanup has no environment or target guard. It does not assert that `DATABASE_URL` contains `test`, that the database name differs from the development database, that `NODE_ENV === 'test'`, or that the host is local. The only precondition is that the imported Prisma client points at whatever URL the environment provides. The truncation also does not reset auto-increment counters because the schema uses UUID primary keys; the only sequence reset comes from deleting the `order_number_sequence` row, which is covered in TR-03.

**Rule workflow:**
```
for each `it` block:
  beforeEach
    -> deleteMany order_status_history
    -> deleteMany order_items
    -> deleteMany orders
    -> deleteMany order_number_sequence
    -> deleteMany products
    -> deleteMany customers
    -> deleteMany users
  -> test executes and commits its own writes
  -> next it -> beforeEach truncates again
```

---

### Business Rule TR-03: Foreign-key-safe truncation order and sequence reset

**Overview:**
The deletion order is fixed and child-first, and it explicitly includes the order-number sequence table, which resets order numbering to `ORD-000001` for every test.

**Detailed description:**
The schema defines foreign keys from `order_status_history` to `orders` and `users`, from `order_items` to `orders` and `products`, and from `orders` to `customers` and `users` (`prisma/schema.prisma:87-88, 108-109, 125-126`). The harness deletes `orderStatusHistory` and `orderItem` first, then `order`, which satisfies the default `RESTRICT` behavior for parent rows still referenced by children. `orderNumberSequence` has no foreign keys and its position in the list has no constraint effect. `product` is deleted after `orderItem` so that no order line still references it; `customer` and `user` are deleted last so that no order or history row still references them.

The sequence reset is a behavioral rule rather than an FK necessity. `OrderService.reserveOrderNumber` upserts the single row `id = 1` and formats `ORD-${String(nextValue - 1).padStart(6, '0')}` (`src/modules/orders/order.service.ts:245-254`). Deleting the sequence row in `beforeEach` means each test's first created order receives `ORD-000001`, the second `ORD-000002`, and so on. This makes assertions such as `/^ORD-\d{6}$/` (`tests/orders.test.ts:39`) and any implicit expectation of deterministic numbering valid per test file. It also means numbering is not globally monotonic across tests and cannot be used to infer ordering between tests or runs.

The list is static and manually maintained. There is no schema introspection, dependency-graph truncation or shared constant enumerating tables. Adding a new table with a foreign key to an existing table requires editing this block in the correct position; omitting it produces either a foreign-key failure or data leakage into subsequent tests. The current list covers all seven tables present in `prisma/schema.prisma:25-138`, so completeness holds for the analyzed revision.

**Rule workflow:**
```
delete history (children of orders/users)
  -> delete items (children of orders/products)
  -> delete orders (children of customers/users)
  -> delete sequence (independent; resets numbering)
  -> delete products (referenced by items, now removed)
  -> delete customers (referenced by orders, now removed)
  -> delete users (referenced by orders/history, now removed)
  -> first order created in the test receives ORD-000001
```

---

### Business Rule TR-04: Serial, single-fork execution

**Overview:**
The harness forbids parallel test-file execution and runs every file in one fork, which is the precondition that makes global table deletion safe.

**Detailed description:**
`vitest.config.ts:11-17` sets `fileParallelism: false`, `pool: 'forks'` and `poolOptions.forks.singleFork: true`. `fileParallelism: false` prevents multiple test files from running at the same time. `singleFork: true` reuses one child process for the whole run instead of creating one process per file. Tests within a file are sequential because the suites use plain `it` calls and no `it.concurrent`. The result is a total ordering of all 15 tests across the run, with no two `beforeEach` truncations able to interleave.

This rule is what turns a globally destructive cleanup into a workable isolation strategy. Without serialization, one suite's `beforeEach` could delete rows another suite had just created, producing flaky 404s, empty lists or foreign-key races even though each suite is individually correct. The configuration therefore encodes an invariant that is not visible from the setup file alone: the correctness of TR-02 depends on TR-04.

The same choice has operational consequences. The suite cannot use multiple CPU cores, and total wall time is the sum of both files. Two CI jobs pointed at the same database will still interfere because serialization is process-local; there is no per-run schema, database name suffix, advisory lock or unique tenant. The `isolate` option is not set, so Vitest's default isolation applies; whether module-level state such as `cachedApp` is shared across files in single-fork mode is a runner-internal detail and is not asserted by any test (see Appendix A).

**Rule workflow:**
```
vitest run
  -> one fork
    -> file A: setup, connect, tests 1..7, disconnect
    -> file B: setup, connect, tests 1..8, disconnect
  -> exit
(no concurrent file execution; no concurrent truncation)
```

---

### Business Rule TR-05: Test discovery, environment and explicit imports

**Overview:**
Only TypeScript files matching `tests/**/*.test.ts` are executed; the runner uses the Node environment, does not inject globals, and loads one setup file for every test file.

**Detailed description:**
`vitest.config.ts:7` sets `include: ['tests/**/*.test.ts']`, which collects `tests/auth.test.ts` and `tests/orders.test.ts` and excludes `tests/setup.ts` and `tests/helpers/factories.ts` from test collection. The helper module is still type-checked because `tsconfig.json:28` includes `tests/**/*.ts`; it is excluded from the production build by `tsconfig.build.json:10`. This separation lets the helper export functions without being treated as a suite.

`environment: 'node'` (`vitest.config.ts:6`) provides Node APIs (`process`, `Buffer`, timers) and no DOM. `globals: false` (`vitest.config.ts:5`) means `describe`, `it` and `expect` must be imported explicitly, which both suites do at line 1. The setup file also imports `afterAll`, `beforeAll` and `beforeEach` explicitly (`tests/setup.ts:1`). This avoids reliance on ambient globals and keeps TypeScript aware of the symbols without adding `vitest/globals` to the `types` array.

The configuration does not declare `test.env`, does not call Vite's `loadEnv`, and does not load `dotenv`. Vitest autoloads only `VITE_`-prefixed variables from `.env` files into `import.meta.env`; arbitrary variables such as `DATABASE_URL` and `JWT_SECRET` are not placed in `process.env` by the runner. `package.json:17` uses plain `vitest run` with no `--env-file`. The harness therefore requires the invoking shell or CI to provide the application environment. `src/config/env.ts:3-10` validates `NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET` and `JWT_EXPIRES_IN`; `DATABASE_URL` and `JWT_SECRET` are mandatory, and `JWT_SECRET` must be at least 16 characters. When Vitest itself is invoked with `vitest run`, Vitest sets `NODE_ENV` to `test` only if it was not already set; if the shell exports `development`, the application logger selects the `pino-pretty` transport and Prisma logs `warn` and `error` instead of only `error` (`src/config/database.ts:6`, `src/shared/logger/index.ts:20-26`).

**Rule workflow:**
```
vitest run
  -> NODE_ENV = test (if unset)
  -> config include filters to *.test.ts
  -> environment = node, globals off
  -> setupFiles = tests/setup.ts (executed per test file)
  -> suites import { describe, expect, it } from 'vitest'
  -> application env must already be present in process.env
```

---

### Business Rule TR-06: Thirty-second test and hook time budget

**Overview:**
Every test and every lifecycle hook is allowed at most 30 seconds. No test overrides the limit and no retry policy exists.

**Detailed description:**
`testTimeout: 30000` and `hookTimeout: 30000` (`vitest.config.ts:8-9`) are the only timing controls in the harness. They apply to each `it` body and to `beforeAll`, `beforeEach` and `afterAll`. The suites do not call `it(..., { timeout })` or `test.setTimeout`, so the global values are authoritative. Because there is no `retry` setting, a failed test is not automatically re-executed; flakiness would surface directly.

The 30-second ceiling accommodates real infrastructure: a MySQL round trip, one bcrypt hash at cost 10 per created user, one bcrypt comparison per login, JWT signing, and several sequential HTTP requests through the full Express stack. It also bounds the damage from a hung connection or an unresponsive database, but not tightly: a stuck operation can consume the entire budget for each affected test. The connect hook can also wait up to the hook timeout before the file fails. No performance assertion or duration threshold is encoded, so the tests do not detect gradual slowdowns.

The timeouts do not include a teardown-specific value; Vitest's default teardown timeout applies to any asynchronous cleanup, but the harness has no explicit teardown work beyond `$disconnect()`. A disconnect that hangs would be governed by the runner's teardown behavior, not by `hookTimeout` as such, which is a runner detail outside the repository's configuration.

**Rule workflow:**
```
beforeAll: max 30 s -> connect
for each test:
  beforeEach: max 30 s -> 7 deletes
  it body: max 30 s -> one or more HTTP requests + assertions
afterAll: disconnect
any overrun -> test/hook failure, no retry
```

---

### Business Rule TR-07: Lazily cached application instance wired to the real Prisma singleton

**Overview:**
`getTestApp()` constructs the complete Express application once and returns the same instance on every subsequent call, using the production Prisma client as the only injected dependency.

**Detailed description:**
`tests/helpers/factories.ts:8` declares `let cachedApp: Express | null = null`; `getTestApp()` checks the cache, calls `buildApp({ prisma })` on the first invocation, stores the result and returns it (`tests/helpers/factories.ts:10-15`). `buildApp` installs the JSON parser, request logger, `/health`, the `/api/v1` router, the 404 handler and the error middleware (`src/app.ts:55-75`). `buildControllers(prisma)` wires every repository, service and controller exactly as in production (`src/app.ts:26-53`). The harness therefore exercises the real composition root and cannot silently substitute a different database or a fake service.

The cache creates a single application graph per module instance. Within a test file, every call to `getTestApp()` returns the same object, so any state stored in a controller, service or middleware would persist across tests; the current services are stateless, but the harness provides no reset hook if that changes. The app is never bound to a fixed port: Supertest starts an ephemeral listener per request and closes it, so there is no port conflict and no external network surface. The app is not rebuilt between tests, which saves the cost of constructing 18 internal dependencies per test.

The only injection point is `AppDependencies.prisma`. There is no way to pass a transaction client, a mock repository, a different logger or an alternate environment. This is consistent with TR-08 (no doubles) but means the harness is coupled to the production composition API and to the Prisma singleton created in `src/config/database.ts`. If a test file runs in a fresh module graph, it gets a fresh `cachedApp` and a fresh Prisma client; if module state is shared, both are shared. The harness does not assert which behavior occurs.

**Rule workflow:**
```
test 1 -> getTestApp()
  -> cache miss
  -> buildApp({ prisma })
     -> buildControllers(prisma)
     -> mount middleware/routes/error handler
  -> cache = app
test 2..N -> getTestApp() -> cache hit -> same Express instance
```

---

### Business Rule TR-08: Full-stack integration only, with no test doubles

**Overview:**
The harness uses the real Express app, real middleware, real services, real Prisma and real MySQL. There are no mocks, stubs, fakes, spies, in-memory repositories or HTTP interception libraries.

**Detailed description:**
A repository-wide scan of `tests/` finds imports only from `vitest`, `supertest`, `bcrypt`, `@prisma/client` (types), `express` (types), the application's Prisma singleton and `buildApp`. There is no `vi.mock`, `vi.fn`, `vi.spyOn`, `msw`, `nock`, `sinon` or fixture JSON. The only indirection is the factory layer, which writes through the same Prisma client the application uses. This means every successful assertion has traversed the actual router, authentication middleware, Zod validation, controller, service, transaction and SQL statement.

The benefit is high fidelity: the suites detect contract drift across layers, real unique-constraint behavior, transaction rollback, stock arithmetic and JWT/bcrypt integration. The costs are environmental dependence and slower execution. The tests cannot simulate database outages, slow queries, Prisma error codes that are hard to trigger, or external network behavior; they also cannot run without a provisioned MySQL and applied migrations. Because the application uses a single Prisma client and one database, there is no seam for fault injection.

The absence of doubles also means test failures are usually attributable to real behavior rather than to mock drift. However, the direct Prisma writes in factories bypass service validation, so "arrangement" and "act" use different code paths. This is a deliberate split: factories create state, suites exercise behavior. It does mean a factory can create a row that the API would never accept (for example, a malformed email or an arbitrary role string at runtime), and the tests would not notice unless a subsequent assertion depends on that state.

**Rule workflow:**
```
arrange via factories -> direct Prisma INSERT (bypasses services)
act via Supertest -> real Express -> real middleware/controllers/services
                   -> real Prisma queries/transactions -> real MySQL
assert via response body and direct Prisma SELECT
(no mock boundary anywhere)
```

---

### Business Rule TR-09: Per-field factory defaults and override precedence

**Overview:**
Each factory merges caller-supplied overrides over a fixed set of defaults using `??` (nullish coalescing), so a field is overridden only when a non-null, non-undefined value is supplied.

**Detailed description:**
`createTestUser` accepts `Partial<{ email; password; name; role }>` and resolves each field with `overrides.x ?? default` (`tests/helpers/factories.ts:17-30`). `createTestCustomer` accepts `Partial<{ name; email; phone; document }>` and fills the address with a fixed literal (`tests/helpers/factories.ts:42-59`). `createTestProduct` accepts `Partial<{ sku; name; priceCents; stockQuantity; active }>` (`tests/helpers/factories.ts:62-74`). The merge is shallow and per-field; there is no deep merge, no builder chaining and no validation. A caller cannot set a field to `null` because `null ?? default` yields the default, and can only bypass that by casting. Optional schema fields not present in the override types (`description` on products, `notes` on orders, address components on customers) cannot be set through the factory at all.

The defaults are meaningful test constants rather than arbitrary values. A user is `Test User` with `OPERATOR` role; a customer is `Test Customer` in Sao Paulo with a fixed phone, document and address; a product costs 10000 cents, has 10 units in stock and is active. These defaults make tests concise and let order tests override only the fields relevant to the scenario, such as `priceCents: 5000, stockQuantity: 20` (`tests/orders.test.ts:16`) or `stockQuantity: 1` (`tests/orders.test.ts:113`). Because the factories return the full Prisma record, tests can assert against generated IDs and values without re-querying.

The direct-insert design means factory behavior is governed by the database schema, not by service rules. Email uniqueness is enforced by the schema, so two users with the same explicit email fail with a Prisma unique-constraint error; the customer `document` is only indexed, not unique, so every default customer shares `000.000.000-00` without conflict. Addresses are identical for all default customers, which is adequate for current assertions but would matter for any address-sensitive behavior. The factories also do not hash or compare through the production `UserService`, so the bcrypt cost is duplicated as a literal (10) rather than imported from `BCRYPT_ROUNDS`.

**Rule workflow:**
```
call factory(overrides)
  -> for each field: value = overrides.field ?? DEFAULT
  -> build Prisma data object
  -> prisma.model.create({ data })
  -> return full record (and password for users)
caller override wins only when value is neither null nor undefined
```

---

### Business Rule TR-10: Unique default identities for every created aggregate

**Overview:**
When the caller does not supply an identity, each factory generates an email or SKU intended to be unique within the test database.

**Detailed description:**
The user factory builds `user-${Date.now()}-${Math.random().toString(36).slice(2, 8)}@test.local` (`tests/helpers/factories.ts:24`). The customer factory uses the same timestamp-plus-six-base36-character pattern with a `customer-` prefix (`tests/helpers/factories.ts:48`). The product factory builds `SKU-${Date.now()}-${Math.random().toString(36).slice(2, 6).toUpperCase()}` (`tests/helpers/factories.ts:67`). The timestamp gives millisecond granularity and the random suffix adds 6 or 4 base36 characters. `Math.random` is not cryptographically strong and no UUID generator is used.

Uniqueness is therefore probabilistic, not guaranteed. Two factories called in the same millisecond with the same random suffix would collide; the database's unique constraints on `users.email`, `customers.email` and `products.sku` would then reject the insert. In practice, the random suffix makes collisions very unlikely, and the per-test truncation removes prior rows, so cross-test collisions do not occur. A test that needs a stable identity supplies it explicitly, as the duplicate-email test does with `dup@test.local` (`tests/auth.test.ts:21`). The registration test uses `register-${Date.now()}@test.local` without a random suffix (`tests/auth.test.ts:8`); this relies on truncation and on the test not repeating within the same millisecond, which is a weaker guarantee than the factories provide.

The uniqueness rule is an implicit contract for factory callers: omit identity fields only when the test does not care about the value. If a test needs to reference a user by email across steps, it should pass the email explicitly. The generated values are not deterministic across runs, so snapshot testing or golden files based on them would be unstable. The uniqueness guarantee also depends on serial execution; parallel writers to the same database could still collide on the same millisecond, although the 4-6 random characters make that improbable.

**Rule workflow:**
```
no identity override:
  email = prefix + Date.now() + "-" + randomBase36Suffix + "@test.local"
  sku   = "SKU-" + Date.now() + "-" + randomBase36Suffix.toUpperCase()
explicit identity override:
  use caller value verbatim
  -> DB unique constraint is the final arbiter
```

---

### Business Rule TR-11: Fixed test credentials, default role, bcrypt cost, and real-login token acquisition

**Overview:**
Test users are created with a known password (`password123`) hashed at bcrypt cost 10 and, by default, the `OPERATOR` role. Authenticated tests obtain a JWT only by calling the real login endpoint; no token is forged or cached.

**Detailed description:**
`createTestUser` resolves `password = overrides.password ?? 'password123'`, hashes it with `bcrypt.hash(password, 10)` and writes the hash directly into `passwordHash` (`tests/helpers/factories.ts:20-27`). The cost factor matches the production constant `BCRYPT_ROUNDS = 10` in `src/modules/users/user.service.ts:16,26`, so login comparison exercises the same work factor. The plaintext password is returned to the caller in the `{ user, password }` tuple so it can be sent to the login endpoint. The role defaults to `OPERATOR`; the override type permits `ADMIN`, but no current test creates an admin, so the admin-only branch of the system is not exercised through the harness.

`loginAndGetToken` calls `getTestApp()`, posts `{ email, password }` to `/api/v1/auth/login`, rejects any non-200 response with an error containing the status and serialized body, and returns `res.body.tokens.accessToken` (`tests/helpers/factories.ts:33-40`). The login path performs a real bcrypt comparison and a real JWT signature using `JWT_SECRET` and `JWT_EXPIRES_IN` (`src/modules/auth/auth.service.ts:31-50`). The helper does not validate that the extracted token is a non-empty string at runtime; a malformed 200 response would yield `undefined` and fail later as a 401.

`bootstrapAuthenticatedUser` composes the two steps: it creates a user with the requested role (default `OPERATOR`), logs in, and returns `{ user, token }` (`tests/helpers/factories.ts:76-83`). All eight order tests call it, so each order test pays for one bcrypt hash and one bcrypt comparison. There is no token cache, no cookie handling and no refresh flow. The helper hardcodes the `/api/v1/auth/login` path, so a route change would break every authenticated test at arrangement time with the descriptive login error rather than at the assertion.

**Rule workflow:**
```
createTestUser({ role })
  -> password = override ?? "password123"
  -> bcrypt.hash(password, 10)
  -> prisma.user.create({ email, name, passwordHash, role: role ?? OPERATOR })
  -> return { user, password }
loginAndGetToken(email, password)
  -> POST /api/v1/auth/login
  -> require status 200
  -> return body.tokens.accessToken
bootstrapAuthenticatedUser(role = OPERATOR)
  -> createTestUser({ role }) -> loginAndGetToken(email, password) -> { user, token }
```

---

### Business Rule TR-12: Fresh aggregate graph per test, no shared fixtures

**Overview:**
Each test creates all the state it needs inside its own body; there are no `beforeAll` domain fixtures, no shared constants for entities and no use of `prisma/seed.ts`.

**Detailed description:**
The auth suite creates users inline or through `createTestUser` only in the tests that need a pre-existing user (`tests/auth.test.ts:21, 43, 56, 67`). The order suite calls `bootstrapAuthenticatedUser`, `createTestCustomer` and `createTestProduct` at the start of every `it` block (`tests/orders.test.ts:13-17, 43-45, 60-63, 90-93, 110-113, 135-138, 164-167, 198-201`). No entity is declared in a module-level variable or reused across tests. Orders themselves are always created through the HTTP API rather than through a factory, so each order test also exercises the creation pipeline.

This rule complements TR-02: truncation makes fresh state possible, and per-test arrangement makes each test self-contained. It gives order independence (even though Vitest's default sequence is deterministic and no shuffle is configured) and makes failures local: a test's data cannot be a leftover from a previous test. It also means there is no shared fixture that could drift from the schema or encode assumptions across suites. The only shared state is the cached app and the Prisma connection.

The absence of a seed or shared fixtures also means each test pays the full arrangement cost. Order tests create at least one user, log in, create a customer and create one or more products before the first assertion. The first order test creates two products; the rest create one. This is deliberate full-stack coverage, but it increases runtime and makes the suite dependent on bcrypt and HTTP latency. Tests never verify that the database is empty at the start; they rely on `beforeEach`. If the truncation block were removed or a table were omitted, tests would still pass individually but could fail in sequence or observe stale rows.

**Rule workflow:**
```
beforeEach truncates all tables
  -> it body:
     bootstrapAuthenticatedUser()
     createTestCustomer()
     createTestProduct(...)
     create order / perform scenario via HTTP
     assert response and direct DB side effects
  -> next it repeats with a fresh graph
(no beforeAll fixtures, no seed, no shared entity variables)
```

---

### Business Rule TR-13: Auth suite verification contract

**Overview:**
The auth suite encodes seven verification rules about registration, login, identity lookup and public data exposure.

**Detailed description:**
The first rule is that registration returns HTTP 201 and only public fields: the test posts a fresh timestamped email, password `strongPass123`, name and `OPERATOR` role, then asserts a subset match and explicitly asserts that `passwordHash` and `password` are absent (`tests/auth.test.ts:6-17`). This verifies the `PublicUser` projection in `src/modules/users/user.service.ts:42-51` and the absence of accidental serialization of the Prisma record. The second rule is duplicate-email rejection: a factory-created user with `dup@test.local` is followed by a registration attempt for the same email, expecting 409 and `EMAIL_ALREADY_USED` (`tests/auth.test.ts:19-28`). The third rule is payload validation: an invalid email, a six-character password and an empty name produce 400 `VALIDATION_ERROR` with an array of details (`tests/auth.test.ts:30-39`), exercising the Zod schema in `src/modules/auth/auth.schemas.ts:3-8` and the validation middleware.

The fourth rule is successful login: a factory user's email and plaintext password produce 200, a string `accessToken`, `tokenType: 'Bearer'` and the same user email (`tests/auth.test.ts:41-52`). The fifth rule is wrong-password rejection: 401 with `UNAUTHORIZED` (`tests/auth.test.ts:54-63`). The sixth rule is authenticated identity: a token obtained through the real login endpoint produces 200 on `GET /auth/me` with the same id and email (`tests/auth.test.ts:65-77`). The seventh rule is missing-token rejection: the same endpoint without an `Authorization` header returns 401 (`tests/auth.test.ts:79-83`). Together these cover the happy path and the main negative paths of the auth module.

The contract does not cover login with an unknown email (only an existing user with a wrong password), registration without an explicit role (default `OPERATOR`), registration with `ADMIN`, password length boundaries, email length boundaries, malformed or expired tokens, token signature tampering, or the admin-only user lookup. It also does not assert response headers or the absence of stack traces. The assertions are explicit about status and error code, which is stronger than checking status alone, and the public-projection assertion is a direct check of a security-relevant property.

**Rule workflow:**
```
POST /auth/register (valid)      -> 201 + public fields only, no passwordHash/password
POST /auth/register (duplicate)  -> 409 EMAIL_ALREADY_USED
POST /auth/register (invalid)    -> 400 VALIDATION_ERROR + details[]
POST /auth/login (valid)         -> 200 + string accessToken + Bearer + user email
POST /auth/login (wrong password)-> 401 UNAUTHORIZED
GET  /auth/me (valid token)      -> 200 + id/email match
GET  /auth/me (no token)         -> 401
```

---

### Business Rule TR-14: Orders suite verification contract

**Overview:**
The orders suite encodes eight verification rules covering creation, pricing, numbering, transition legality, stock movement, filtering and deletion.

**Detailed description:**
The first rule is server-side order creation: with a customer, two products priced 5000 and 1500, quantities 3 and 2, and a 1000-cent discount, the response must be 201 with `status: PENDING`, `subtotalCents: 18000`, `discountCents: 1000`, `totalCents: 17000`, two items, one history entry whose `toStatus` is `PENDING`, and an `orderNumber` matching `/^ORD-\d{6}$/` (`tests/orders.test.ts:12-40`). This verifies aggregation of distinct products, price lookup from the database, discount subtraction, initial history creation and sequence formatting. The second rule is non-existent product rejection: a valid-format but absent UUID produces 404 `NOT_FOUND` (`tests/orders.test.ts:42-57`). The third rule is the PENDING to PAID transition: with stock 5 and quantity 3, the PATCH returns 200, status `PAID`, history equals `[PENDING, PAID]`, and a direct database read shows stock 2 (`tests/orders.test.ts:59-87`).

The fourth rule is illegal transition rejection: PENDING to SHIPPED returns 409 `INVALID_STATUS_TRANSITION` (`tests/orders.test.ts:89-107`). The fifth rule is insufficient-stock handling: an order for quantity 5 can be created even when stock is 1 (confirming creation does not check stock), but paying it returns 422 `INSUFFICIENT_STOCK` with `details.unavailable[0].sku` equal to the product SKU, and a direct read shows stock remains 1, which verifies transaction rollback of the failed debit (`tests/orders.test.ts:109-132`). The sixth rule is stock replenishment: an order for quantity 4 from stock 10 reaches stock 6 after payment and returns to 10 after PAID to CANCELLED (`tests/orders.test.ts:134-161`). The seventh rule is status filtering: three orders are created, one is paid, and `GET /orders?status=PAID` returns exactly that order while `status=PENDING` returns two with `pagination.total === 2` (`tests/orders.test.ts:163-195`). The eighth rule is the delete guard: a PAID order cannot be deleted and returns 409 `INVALID_ORDER_STATE_FOR_DELETE` (`tests/orders.test.ts:197-218`).

Several assertions are performed by re-reading through Prisma after the HTTP call, which is a strong oracle for side effects: stock decrement, stock replenishment and the no-debit-on-failure case. The suite does not cover inactive products, discount greater than subtotal, duplicate product IDs that require aggregation, same-status transitions, PENDING to CANCELLED (which must not replenish), PAID to PROCESSING, PROCESSING to SHIPPED or CANCELLED, SHIPPED to DELIVERED, terminal-state behavior, successful deletion of PENDING or CANCELLED orders, `GET /orders/:id`, pagination fields other than `total`, list filters by customer or date, role authorization, or concurrency. Some setup requests do not assert their status (for example the PATCH at `tests/orders.test.ts:145-148` and the three POSTs at lines 171-176), so a setup failure would surface as a later assertion failure with less direct diagnosis.

**Rule workflow:**
```
POST /orders (valid)                 -> 201, PENDING, server totals, 1 PENDING history, ORD-NNNNNN
POST /orders (missing product)       -> 404 NOT_FOUND
PATCH PENDING -> PAID                -> 200, stock decremented, history appended
PATCH PENDING -> SHIPPED             -> 409 INVALID_STATUS_TRANSITION
PATCH PENDING -> PAID (short stock)  -> 422 INSUFFICIENT_STOCK, stock unchanged
PATCH PAID -> CANCELLED              -> 200, stock replenished
GET /orders?status=PAID|PENDING      -> filtered data + pagination.total
DELETE PAID order                    -> 409 INVALID_ORDER_STATE_FOR_DELETE
```

---

## 4. Component Structure

```
project-root/
├── vitest.config.ts                    # 19 lines: runner configuration
│                                       #   node env, globals off, *.test.ts include
│                                       #   30 s test/hook timeouts
│                                       #   setupFiles = tests/setup.ts
│                                       #   serial single-fork execution
├── tests/
│   ├── setup.ts                        # 20 lines: per-file DB lifecycle
│   │                                   #   beforeAll  -> prisma.$connect()
│   │                                   #   beforeEach -> deleteMany x7 (FK-safe)
│   │                                   #   afterAll   -> prisma.$disconnect()
│   ├── helpers/
│   │   └── factories.ts                # 83 lines: shared arrangement utilities
│   │                                   #   cachedApp / getTestApp
│   │                                   #   createTestUser (bcrypt cost 10)
│   │                                   #   loginAndGetToken (real HTTP login)
│   │                                   #   createTestCustomer (fixed address)
│   │                                   #   createTestProduct (defaults)
│   │                                   #   bootstrapAuthenticatedUser
│   ├── auth.test.ts                    # 84 lines: 7 integration tests
│   │                                   #   register x3, login x2, me x2
│   └── orders.test.ts                  # 219 lines: 8 integration tests
│                                       #   create, invalid product, pay, invalid
│                                       #   transition, insufficient stock,
│                                       #   cancel replenish, list filter, delete guard
├── package.json                        # test scripts and dev dependencies
│                                       #   "test": "vitest run" (line 17)
│                                       #   "test:watch": "vitest" (line 18)
├── tsconfig.json                       # includes tests in type-checking (line 28)
├── tsconfig.build.json                 # excludes tests from production build (line 10)
├── .env.example                        # environment contract for the app and harness
├── docker-compose.yml                  # MySQL 8.0 prerequisite (no test profile)
└── prisma/
    ├── schema.prisma                   # seven tables targeted by truncation
    └── seed.ts                         # not used by the harness
```

Internal organization observations:

- There is no dedicated `tests/fixtures/`, `tests/factories/` per domain, `tests/global-setup.ts` or `tests/utils/` layer. The entire shared surface is one 83-line module plus one 20-line setup file.
- Both suites import from `./helpers/factories.js` (ESM `.js` specifier for a TypeScript source, consistent with the project's `moduleResolution: "Bundler"` and ESM output convention). `tests/orders.test.ts` additionally imports the Prisma singleton directly for side-effect assertions (`tests/orders.test.ts:9`).
- Setup is not exported as a module with initialization functions; it relies entirely on Vitest hooks at import time. Vitest ignores exports from setup files, so this is the only viable structure.
- The harness has no own configuration module, no table list constant and no environment validation of its own; it reuses the application's `env.ts` by importing the database module.

---

## 5. Dependency Analysis

### 5.1 Internal dependency graph

```
vitest.config.ts
  └── (setupFiles path reference) ──> tests/setup.ts

tests/setup.ts
  ├── vitest (hooks)
  └── src/config/database.ts
        └── src/config/env.ts
              └── zod

tests/helpers/factories.ts
  ├── src/config/database.ts          (prisma singleton)
  ├── src/app.ts                      (buildApp)
  │     ├── src/middlewares/*         (requestLogger, errorMiddleware)
  │     ├── src/modules/{auth,users,customers,products,orders}/*
  │     ├── src/routes/index.ts
  │     └── src/shared/errors/index.ts
  ├── bcrypt
  ├── supertest
  ├── @prisma/client (type-only: Customer, Product, User)
  └── express (type-only: Express)

tests/auth.test.ts
  ├── vitest
  ├── supertest
  └── tests/helpers/factories.ts

tests/orders.test.ts
  ├── vitest
  ├── supertest
  ├── tests/helpers/factories.ts
  └── src/config/database.ts          (direct DB side-effect assertions)
```

No cyclic imports exist inside the component. The dependency direction is acyclic: suites -> factories -> application composition -> modules -> shared/config. The setup file depends only on the database provider. `vitest.config.ts` references the setup file by path, not by import.

### 5.2 Internal dependencies

| Consumer | Dependency | Nature | Coupling consequence |
|----------|------------|--------|----------------------|
| `vitest.config.ts` | `tests/setup.ts` | Path reference in `setupFiles` | Renaming or moving setup breaks the run |
| `tests/setup.ts` | `src/config/database.ts` | Value import (`prisma`) | Truncation targets the production singleton and whatever `DATABASE_URL` selects |
| `tests/setup.ts` (indirect) | `src/config/env.ts` | Import-time evaluation | Missing/invalid env terminates the process before hooks |
| `tests/helpers/factories.ts` | `src/config/database.ts` | Value import (`prisma`) | Fixtures use the same connection as the app |
| `tests/helpers/factories.ts` | `src/app.ts` | Value import (`buildApp`) | Harness is bound to the composition-root signature and dependency set |
| `tests/auth.test.ts` | `tests/helpers/factories.ts` | Value imports (3 functions) | Shared fixture behavior affects auth tests |
| `tests/orders.test.ts` | `tests/helpers/factories.ts` | Value imports (4 functions) | Shared fixture behavior affects order tests |
| `tests/orders.test.ts` | `src/config/database.ts` | Value import (`prisma`) | Direct oracle reads couple assertions to Prisma model shape |
| Factories (indirect) | `prisma/schema.prisma` | Generated client types and constraints | Schema changes alter factory fields, return types and unique failures |
| Setup (indirect) | `prisma/schema.prisma` | Table names and FK graph | New tables require manual truncation updates |

### 5.3 External dependencies

| Package / Service | Version | Role in the harness | Used by |
|-------------------|---------|---------------------|---------|
| `vitest` | 2.1.4 (exact) | Test runner, hooks, assertions, worker/pool management | config, setup, both suites |
| `supertest` | 7.0.0 (exact) | In-process HTTP requests against the Express app | factories (`loginAndGetToken`), both suites |
| `@types/supertest` | 6.0.2 | Type declarations for Supertest | Type-checking only |
| `bcrypt` | 5.1.1 (exact) | Hash test passwords at cost 10 | factories |
| `@prisma/client` | 5.22.0 (exact) | DB access, transactions, generated types | factories, setup, both suites (via singleton) |
| `prisma` (CLI) | 5.22.0 | Schema/migration tooling required before tests | External prerequisite, not imported |
| `express` | 4.21.1 (exact) | Application under test; type import in factories | `src/app.ts`, factories type |
| `@types/express` | 4.17.21 | Type declarations | Type-checking only |
| `jsonwebtoken` | 9.0.2 (exact) | Token signing/verification exercised through login and auth middleware | Transitive via app |
| `zod` | 3.23.8 (exact) | Validation exercised through HTTP payloads and env parsing | Transitive via app and env |
| `uuid` | 11.0.3 (exact) | `X-Request-Id` generation in the request logger | Transitive via app |
| `pino` | 9.5.0 (exact) | Access/error logging during tests; no assertions on output | Transitive via app |
| `pino-pretty` | 11.3.0 (exact) | Development log transport; inactive when `NODE_ENV=test` | Transitive via logger |
| `typescript` | 5.6.3 (exact) | Type-checks tests via `tsconfig.json:28`; excludes them from build | Tooling |
| `tsx` | 4.19.2 (exact) | Runs `prisma/seed.ts`; not used by the harness | Tooling |
| MySQL | 8.0 (`mysql:8.0` image) | Real persistence for fixtures and the application | `docker-compose.yml:2-24` |
| Node.js | `>=20` (`package.json:7-9`); observed v24.13.0 in the analysis environment | Runtime for Vitest and the app | All |

Version pins are exact for all direct dependencies, and `package-lock.json` is committed. `pino-http` is declared in `package.json:31` but is not imported by the harness or the application.

---

## 6. Afferent and Efferent Coupling

### 6.1 Methodology

Coupling is measured at two levels appropriate to this TypeScript codebase: **module level** (files importing files) and **export level** (functions invoked by suites or helpers). Afferent coupling (Ca) counts distinct internal project modules outside the unit that depend on it. Efferent coupling (Ce) counts distinct internal project modules the unit imports. External npm packages and services are excluded from Ca/Ce and reported separately in Section 5.3. Type-only imports are counted because they create compile-time coupling. Configuration path references (for example `setupFiles`) are reported separately from import-based Ca so the distinction is explicit.

### 6.2 Module-level coupling

| Component (module) | Afferent Coupling (Ca) | Efferent Coupling (Ce) | External dependencies | Criticality |
|--------------------|------------------------|------------------------|-----------------------|-------------|
| `tests/helpers/factories.ts` | 2 (`tests/auth.test.ts`, `tests/orders.test.ts`) | 2 (`src/config/database.ts`, `src/app.ts`) | `bcrypt`, `supertest`, `@prisma/client` (type), `express` (type) | High |
| `tests/setup.ts` | 0 import-based; 1 configuration reference (`vitest.config.ts:10`) | 1 (`src/config/database.ts`) | `vitest` | High |
| `vitest.config.ts` | 0 import-based (loaded by the runner) | 0 internal | `vitest/config` | High |
| `tests/auth.test.ts` | 0 | 1 (`tests/helpers/factories.ts`) | `vitest`, `supertest` | Low |
| `tests/orders.test.ts` | 0 | 2 (`tests/helpers/factories.ts`, `src/config/database.ts`) | `vitest`, `supertest` | Medium |

Dependencies inside the component boundary but outside the harness files:

| Dependency | Ca from harness | Ce | Note |
|------------|-----------------|----|------|
| `src/config/database.ts` | 3 (`setup.ts`, `factories.ts`, `orders.test.ts`) | 1 (`src/config/env.ts`) | Most depended-on internal module in the component |
| `src/app.ts` | 1 (`factories.ts`) | 18 (per architectural report) | Single composition root; highest fan-out in the project |
| `src/config/env.ts` | 1, indirect (`database.ts`) | 0 | Import-time fail-fast affects every test file |
| `prisma/schema.prisma` | 0 imports, 7 model usages | 0 | Schema is the ultimate contract for fixtures and truncation |
| `prisma/seed.ts` | 0 | 0 | Not used by the harness |

### 6.3 Export-level coupling

| Export | Direct call sites in suites | Internal call sites | Total | Criticality |
|--------|-----------------------------|---------------------|-------|-------------|
| `getTestApp` | 15 (7 auth + 8 orders) | 1 (`loginAndGetToken`) | 16 | High |
| `createTestProduct` | 8 | 0 | 8 | High |
| `createTestCustomer` | 8 | 0 | 8 | High |
| `bootstrapAuthenticatedUser` | 8 | 0 | 8 | High |
| `createTestUser` | 4 | 1 (`bootstrapAuthenticatedUser`) | 5 | Medium |
| `loginAndGetToken` | 1 | 1 (`bootstrapAuthenticatedUser`) | 2 | Medium |

Observations:

- `factories.ts` is the only shared test module; both suites depend on it, which gives it maximum afferent coupling within the component and makes its defaults a de facto contract for all tests.
- `orders.test.ts` has the widest efferent coupling because it reaches both the factory module and the Prisma singleton. This is consistent with its role as the broader integration suite.
- `src/config/database.ts` is the single highest-afferent internal dependency in the harness (three direct importers). Any change to client creation, logging configuration or singleton lifetime affects setup, arrangement and oracle reads simultaneously.
- The harness has no afferent coupling to production modules: no `src` file imports `tests`, and `tsconfig.build.json:10` excludes tests from the production build. The component is a pure consumer at the edge of the dependency graph.
- There is no circular dependency. The cached app does create a runtime back-reference from tests to the composition root, but it is not an import cycle.

---

## 7. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 (`docker-compose.yml:2-24`) | External database | Persistence for fixtures, application writes and oracle reads | MySQL wire protocol over TCP (Prisma) | Relational rows, SQL | No retry or wait; connection failures surface as hook/test failures; destructive truncation has no guard |
| Prisma Client 5.22 (`src/config/database.ts:4-10`) | ORM / transaction manager | CRUD, nested writes, interactive transactions (`OrderService`) | In-process client calling MySQL | Typed model objects | Prisma errors propagate; recognized `P2002`/`P2025` mapped by the app error middleware, others become 500 |
| Express 4.21 application (`src/app.ts:55-75`) | Internal system under test | Full HTTP pipeline execution | In-process Supertest request (ephemeral listener) | JSON request/response | Supertest rejects on transport errors; HTTP errors are serialized by `errorMiddleware` |
| `buildApp({ prisma })` composition root (`src/app.ts:22-55`) | Internal dependency injection | Construct all repositories/services/controllers | Function call | Typed `Express` instance | Construction errors abort `getTestApp()`; cache remains `null` and the next call retries |
| Vitest 2.1.4 (`vitest.config.ts`) | Test runner | Discovery, file isolation, hooks, assertions, timeouts | Node child process (fork) | TypeScript modules, JSON results | Fail-fast; 30 s test/hook timeouts; no retries |
| bcrypt 5.1.1 (`tests/helpers/factories.ts:21`) | Library | Hash default/explicit test passwords | In-process native module | String hash | Throws on failure; no fallback |
| JWT via `jsonwebtoken` (`src/modules/auth/auth.service.ts:47-50`) | Auth library exercised through login | Issue access tokens for authenticated tests | In-process | Signed JWT string | Login helper throws on non-200; token shape not runtime-validated |
| `src/config/env.ts` (`env.ts:3-25`) | Configuration provider | Supply `DATABASE_URL`, `JWT_SECRET`, `NODE_ENV`, `JWT_EXPIRES_IN`, `LOG_LEVEL` | `process.env` | Strings | `process.exit(1)` on invalid/missing required variables |
| Pino logger / request logger | Observability | Emit one info log per HTTP request and error logs | stdout | JSON (or pretty in development) | Logging failure not handled; no assertions on logs |
| Docker Compose MySQL service | Infrastructure provisioning | Local database availability | Docker engine | Container/volume | Healthcheck defined in Compose (`docker-compose.yml:15-20`); harness does not consult it |

Integration observations:

- MySQL is the only external stateful dependency and the only network boundary. Supertest does not leave the process for HTTP; the database connection is the sole real I/O.
- The harness does not use a separate test database, test container, schema suffix or migration step. It connects to `DATABASE_URL` as-is. The repository's Compose file provisions one database (`oms` by default) with one volume (`oms_mysql_data`), so development and test data share the same target unless the operator supplies a different URL.
- The Compose healthcheck is not integrated into the harness; Vitest starts immediately and relies on the database being ready. There is no global setup or wait loop.
- `pino-http` is declared but unused; request logging is the custom middleware, which emits a log for every test request.
- The login integration is the only path that generates a JWT. Tests never call `jsonwebtoken` directly, so token creation and verification are exercised end to end.

---

## 8. Design Patterns & Architecture

### 8.1 Patterns identified

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| xUnit fixture lifecycle | `beforeAll` connect, `beforeEach` truncate, `afterAll` disconnect | `tests/setup.ts:4-20` | Deterministic setup/teardown around each test file |
| Table-truncation isolation | Unqualified `deleteMany()` across seven models | `tests/setup.ts:8-16` | Empty state before every test without transactions |
| Test Data Builder / Object Mother hybrid | Factory functions with per-field defaults and optional overrides | `tests/helpers/factories.ts:17-74` | Concise, consistent aggregate creation |
| Lazy singleton | Module-level `cachedApp` with null check | `tests/helpers/factories.ts:8-15` | Build the expensive app graph once per module lifetime |
| In-process integration testing | Supertest against the Express app without `listen` | `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts:35` | Exercise the real HTTP stack without network setup |
| Composition-root reuse / constructor injection | `buildApp({ prisma })` wires all dependencies | `src/app.ts:22-55` called from `tests/helpers/factories.ts:12` | Test the same wiring as production |
| Direct DB oracle | `prisma.product.findUnique` after HTTP mutation | `tests/orders.test.ts:85-86, 130-131, 150-151, 159-160` | Verify side effects not fully visible in the response |
| Serial test execution | `fileParallelism: false`, `singleFork: true` | `vitest.config.ts:11-17` | Make global truncation race-free |
| No test doubles | No `vi.mock`, stubs or fakes anywhere | Entire harness | Maximum fidelity to production behavior |
| Convention over configuration | Defaults for every entity field and a single setup file | `tests/helpers/factories.ts`, `vitest.config.ts:10` | Low ceremony for new tests |

### 8.2 Architectural decisions encoded in the harness

- **The suite is a black-box integration suite, not a unit-test suite.** Assertions are made at the HTTP boundary plus direct database reads. There is no per-class or per-function test layer, no coverage instrument, and no test for isolated utility functions.
- **Isolation is state-destructive rather than transactional.** A transaction-per-test or schema-per-worker design would avoid deleting committed rows; this harness instead deletes before each test and depends on serial execution. The trade-off is simplicity and full commit visibility at the cost of safety and parallelism.
- **Fixtures are data builders, not domain services.** Factories bypass service validation to make arrangement direct and fast. The "act" phase alone exercises the business rules.
- **The application is not modified for testability.** There is no test-only entry point, no environment switch, no in-memory repository and no overridable clock. The harness adapts to the production composition root rather than the reverse.
- **Environment is externalized to the process.** The harness has no `.env` bootstrap, no `globalSetup` and no child-process environment configuration. It inherits whatever the invoking shell or CI provides.
- **The runner is configured for determinism, not speed.** Serial forks, fixed timeouts and no retries favor reproducibility over throughput.

### 8.3 Configuration and environment handling

| Setting | Source | Required / default | Effect on the harness |
|---------|--------|--------------------|-----------------------|
| `NODE_ENV` | `process.env`; Vitest sets `test` if unset | Optional; `test` under `vitest run` when unset | Selects Pino transport and Prisma log level in the app; `test` disables pretty logs and limits Prisma logs to errors |
| `DATABASE_URL` | `process.env` via `src/config/env.ts:7` | Required | Selects the database that is truncated and asserted against. No test-specific value is defined in the repository |
| `JWT_SECRET` | `process.env` via `src/config/env.ts:8` | Required, minimum 16 characters | Signs and verifies tokens used by authenticated tests |
| `JWT_EXPIRES_IN` | `process.env` via `src/config/env.ts:9` | Optional, default `8h` | Token lifetime; no test asserts expiry |
| `LOG_LEVEL` | `process.env` via `src/config/env.ts:6` | Optional, default `info` | Controls log volume during the run; no test asserts logging |
| `PORT` | `src/config/env.ts:5` | Optional, default `3000`; unused | The app is never `listen`ed; Supertest uses ephemeral ports |
| `SHADOW_DATABASE_URL` | `.env.example:6`; Prisma schema | Not read by `env.ts` or the harness | Only relevant to `prisma migrate dev`, not to test execution |
| `MYSQL_*` | `.env.example:11-14`; Compose | Optional Compose fallbacks | Provision the database container; not read by Vitest |
| `include` | `vitest.config.ts:7` | `tests/**/*.test.ts` | Excludes helpers and setup from collection |
| `setupFiles` | `vitest.config.ts:10` | `tests/setup.ts` | Runs lifecycle hooks before each test file |
| `environment` | `vitest.config.ts:6` | `node` | No DOM; Node APIs available |
| `globals` | `vitest.config.ts:5` | `false` | Explicit imports required |
| `testTimeout` / `hookTimeout` | `vitest.config.ts:8-9` | `30000` ms each | Per-test and per-hook ceiling |
| `fileParallelism` | `vitest.config.ts:11` | `false` | One test file at a time |
| `pool` / `singleFork` | `vitest.config.ts:12-17` | `forks` / `true` | One child process for the run |
| `test.env` / `loadEnv` / `dotenv` | Absent | Not configured | Non-`VITE_` `.env` variables are not loaded into `process.env` by the runner; caller must export them |
| Coverage provider / thresholds | Absent | Not configured | No coverage collection or enforcement |
| `isolate` | Not set | Vitest default | Per-file isolation is not explicitly chosen or documented |
| `sequence.shuffle` | Not set | Default sequential | Test order is deterministic but tests are written to be independent |
| `retry` | Not set | Default zero | No automatic re-execution of flaky tests |

---

## 9. Technical Debt & Risks

### 9.1 Risk register

| Risk Level | Component Area | Issue | Evidence | Impact |
|------------|----------------|-------|----------|--------|
| High | `tests/setup.ts` | Destructive truncation runs against whatever `DATABASE_URL` points to, with no environment or database-name guard | `tests/setup.ts:8-16`; no `NODE_ENV`/URL assertion anywhere in the file | Running the suite against a development or shared database deletes all application data |
| High | Configuration | No separate test database, schema or container profile is defined | `vitest.config.ts` has no `env`; `docker-compose.yml` defines one MySQL service and one `oms` database; `.env.example` has a single `DATABASE_URL` | Test data and development data share a target by default; reproducibility depends on the operator |
| High | Environment bootstrap | `npm test` does not load `.env`, and `env.ts` exits the process when `DATABASE_URL`/`JWT_SECRET` are missing | `package.json:17`; `vitest.config.ts` has no `loadEnv`; `src/config/env.ts:7-8, 15-23` | A fresh checkout cannot run tests without externally exported variables; failure is an abrupt process exit, not a test error |
| Medium | `tests/setup.ts` | No cleanup after the final test; the database is left populated after the run | `tests/setup.ts:18-20` only disconnects; no `afterAll` deletes | Stale test rows remain for inspection or for a subsequent manual run; the next run's first `beforeEach` is the only cleanup |
| Medium | `tests/setup.ts` | Truncation is seven independent auto-committed statements, not one atomic operation | `tests/setup.ts:9-15` | A crash or error mid-cleanup leaves partial state; concurrent observers can see mixed states |
| Medium | `tests/setup.ts` / `prisma/schema.prisma` | The truncation list is manual and static; it is not derived from the schema | `tests/setup.ts:9-15`; seven models in `prisma/schema.prisma:25-138` | New tables added to the schema will not be cleaned unless the list is edited; ordering mistakes cause FK failures or leakage |
| Medium | `src/config/database.ts` / `src/config/env.ts` | Import-time side effects: client creation and `process.exit(1)` happen before any hook | `src/config/database.ts:10`; `src/config/env.ts:15-23, 27`; imported by `tests/setup.ts:2` | Configuration problems terminate the worker instead of producing a diagnosable test failure; the harness cannot catch or report them |
| Medium | `tests/helpers/factories.ts` | Factories bypass all service/domain validation | Direct `prisma.*.create` at `tests/helpers/factories.ts:22-29, 45-59, 65-73` | Arrangement can create states the API forbids; tests may rely on data that production cannot produce |
| Medium | `tests/helpers/factories.ts` | Identity generation uses `Date.now()` plus non-cryptographic `Math.random()` | `tests/helpers/factories.ts:24, 48, 67` | Collisions are improbable but possible; unique constraints surface them as failures without regeneration |
| Medium | `tests/auth.test.ts` | Registration test uses a timestamp-only email | `tests/auth.test.ts:8` | Same-millisecond reruns without truncation can collide; weaker than the factory pattern |
| Medium | `tests/helpers/factories.ts` | Every default customer shares the same document and address literal | `tests/helpers/factories.ts:50-57` | Future uniqueness or address-sensitive rules break every default customer fixture |
| Medium | Coverage | Customers, products, users admin lookup, `/health`, server bootstrap and many order branches are untested; no coverage provider or threshold exists | Section 10; no `coverage` block in `vitest.config.ts`; no coverage package in `package.json:35-53` | Regressions in untested modules are invisible to `npm test`; coverage cannot be measured from the repository |
| Medium | `tests/helpers/factories.ts` | `cachedApp` has no reset hook and no dependency override | `tests/helpers/factories.ts:8-15` | State added to controllers/services would leak across tests in the same module instance; the harness cannot isolate per test beyond DB truncation |
| Low | `tests/helpers/factories.ts` | Login helper assumes a fixed route and an unvalidated response shape | `tests/helpers/factories.ts:35-39` | Route or response changes break every authenticated test; a missing token becomes a later 401 rather than a clear helper error |
| Low | `tests/orders.test.ts` | Several setup requests do not assert their status | `tests/orders.test.ts:145-148, 171-176, 207-210` | A setup failure produces a less direct failure at the next assertion |
| Low | `vitest.config.ts` | 30-second timeouts and no retry | `vitest.config.ts:8-9` | A hung dependency can consume the full budget; transient failures are not retried (intentional for determinism) |
| Low | Runtime behavior | Every request emits a Pino access log; no log suppression in tests | `src/middlewares/request-logger.middleware.ts:12-25` | Noisy output and minor I/O overhead; no assertions depend on logs |
| Low | `package.json` / `tsconfig` | Tests are linted and type-checked, but `*.config.ts` is ignored by ESLint | `.eslintrc.json` `ignorePatterns`; `tsconfig.json:28` | `vitest.config.ts` is not linted; test code is type-checked but excluded from build |
| Low | CI topology | Serial execution and a single shared database make concurrent CI jobs unsafe | `vitest.config.ts:11-17`; no per-job database provisioning | Two runs against the same database interfere; the suite cannot scale horizontally |

### 9.2 Security and data-safety assessment

- **Primary security concern: unintended data destruction.** The harness performs unqualified deletes on seven tables using credentials from `DATABASE_URL`. There is no technical control that distinguishes a test database from a production or shared development database. The only protection is operational discipline (exporting a test URL before running).
- **Credential handling.** Test credentials are hardcoded by design (`password123`) and are not production secrets. The application's `JWT_SECRET` is read from the environment and is not logged by the harness; Pino redaction covers `authorization`, `cookie`, `password`, `passwordHash`, `token` and `accessToken` (`src/shared/logger/index.ts:4-12`). No test writes secrets to disk.
- **Public-data verification.** The auth suite explicitly asserts that `passwordHash` and `password` are absent from registration responses (`tests/auth.test.ts:14-16`), which is a positive security contract. It does not test role escalation, admin-only access, expired/malformed tokens or tenant scoping; the known production risks in those areas are outside this component's test coverage.
- **Test data nature.** Fixtures use synthetic names, a fixed `+5511999999999` phone, a fixed `000.000.000-00` document and a Sao Paulo address. No real personal data is present.
- **Network exposure.** Supertest binds only ephemeral in-process listeners; the only outbound traffic is the MySQL connection. No test calls an external HTTP service.
- **Supply chain.** Direct dependencies are exactly pinned and a lockfile is committed. The harness adds `vitest`, `supertest` and `bcrypt` to the test path; `bcrypt` is a native module used from both production and test code.

### 9.3 Execution and performance characteristics

- **Serial execution:** total wall time is the sum of both files; no worker-level parallelism. The configuration intentionally trades throughput for isolation safety.
- **Per-test fixed cost:** seven DELETE statements, one or more HTTP requests through the full middleware stack, and for order tests one bcrypt hash (cost 10) plus one bcrypt comparison plus JWT signing.
- **Amortized cost:** the Express app is built once per module instance, avoiding repeated construction of 18 internal dependencies. Supertest avoids a fixed listening port.
- **No caching of fixtures:** every test rebuilds its aggregate graph after truncation. This is a direct consequence of TR-12.
- **No performance assertions:** no test measures latency, query count or response size, so performance regressions are not detected by the suite.
- **No retry or sharding:** a flaky test fails the run; there is no `retry`, `sequence.shuffle` or project-level sharding.

---

## 10. Test Coverage Analysis

### 10.1 Suite inventory

| Test file | Suite | Tests | Type | Lines | Execution model |
|-----------|-------|-------|------|-------|-----------------|
| `tests/auth.test.ts` | `Auth flow` | 7 | Integration (HTTP + real DB) | 84 | Sequential, single fork |
| `tests/orders.test.ts` | `Orders` | 8 | Integration (HTTP + real DB) | 219 | Sequential, single fork |
| **Total** | 2 suites | **15** | Integration only | **303 suite lines** | Serial |

Test-level inventory:

| # | Test | File:Line | Primary verification |
|---|------|-----------|----------------------|
| 1 | registers a new user and returns 201 with public data only | `tests/auth.test.ts:6` | 201, public projection, no secrets |
| 2 | rejects register with duplicated email | `tests/auth.test.ts:19` | 409 `EMAIL_ALREADY_USED` |
| 3 | rejects register with invalid payload | `tests/auth.test.ts:30` | 400 `VALIDATION_ERROR`, details array |
| 4 | logs in successfully and returns an access token | `tests/auth.test.ts:41` | 200, token type/string, user |
| 5 | rejects login with wrong password | `tests/auth.test.ts:54` | 401 `UNAUTHORIZED` |
| 6 | returns the authenticated user on GET /auth/me | `tests/auth.test.ts:65` | 200, id/email match |
| 7 | rejects /auth/me without token | `tests/auth.test.ts:79` | 401 |
| 8 | creates an order in PENDING status and computes totals on the server | `tests/orders.test.ts:12` | 201, totals, history, order number |
| 9 | rejects an order with non-existent product | `tests/orders.test.ts:42` | 404 `NOT_FOUND` |
| 10 | transitions PENDING -> PAID and decrements stock | `tests/orders.test.ts:59` | 200, history, DB stock |
| 11 | returns 409 on invalid status transition (PENDING -> SHIPPED) | `tests/orders.test.ts:89` | 409 `INVALID_STATUS_TRANSITION` |
| 12 | returns 422 when transitioning PENDING -> PAID without enough stock | `tests/orders.test.ts:109` | 422 `INSUFFICIENT_STOCK`, DB stock unchanged |
| 13 | replenishes stock when going PAID -> CANCELLED | `tests/orders.test.ts:134` | DB stock 10 -> 6 -> 10 |
| 14 | lists orders filtered by status | `tests/orders.test.ts:163` | Filtered data, `pagination.total` |
| 15 | refuses to delete an order that is not PENDING or CANCELLED | `tests/orders.test.ts:197` | 409 `INVALID_ORDER_STATE_FOR_DELETE` |

### 10.2 Coverage by production area

| Production area | Exercised by | Depth | Notable gaps |
|-----------------|--------------|-------|--------------|
| Auth routes/controller/service | Tests 1-7 | High for register/login/me | Unknown email, default role, ADMIN registration, expiry/tampering, password/email boundaries |
| Users module | Tests 1, 4, 6 | Partial (create, findByEmail, findById, public projection) | `GET /users/:id`, admin role gate, duplicate-race behavior |
| Orders routes/controller/service | Tests 8-15 | Medium-high for create/pay/cancel/list/delete | Inactive product, discount > subtotal, duplicate product aggregation, same-status, PAID->PROCESSING, PROCESSING->SHIPPED/CANCELLED, SHIPPED->DELIVERED, terminal states, successful delete, getById |
| Order state machine | Tests 10, 11, 13 | Partial (PENDING->PAID, PENDING->SHIPPED rejection, PAID->CANCELLED) | All other transitions and terminal predicates |
| Order repository | Tests 8, 14 | Partial (create paths, status filter, list) | customer/date filters, pagination window, getById |
| Pagination helper | Test 14 | Partial (`pagination.total` only) | `page`, `pageSize`, `totalPages`, defaults, bounds |
| Validation middleware | Tests 3, 9, 11, 12 | Medium (body and params/body paths) | Query validation, malformed UUIDs, unknown routes |
| Error middleware | Tests 2, 3, 5, 7, 9, 11, 12, 15 | Medium (`AppError`, Zod, integration paths) | `P2002`/`P2025` mapping, unhandled 500 branch |
| Auth middleware | Tests 6, 7, and all order tests | Medium (valid and missing token) | Malformed/expired/wrong-signature tokens, role authorization |
| Request logger | All HTTP tests (implicit) | Indirect | Header propagation, redaction, duration/status fields |
| Logger | Indirect | Indirect | Redaction, environment transport selection |
| Customers module | None | None | Full CRUD, uniqueness, pagination |
| Products module | None | None | Full CRUD, SKU uniqueness, stock ownership |
| `GET /health` | None | None | Liveness contract |
| Server bootstrap / shutdown | None | None | Process lifecycle |
| Environment configuration | Indirect successful load only | Minimal | Invalid/missing variable behavior, exit path |
| Prisma schema constraints | Tests 2 (unique email), 12 (FK/transaction behavior) | Partial | Unique SKU, document index, cascade/restrict behaviors |
| Seed script | None | None | Not part of the harness |
| Test harness itself | None | None | No meta-tests for truncation order, factory defaults or lifecycle |

### 10.3 Quantitative coverage

No quantitative coverage figure is available from the repository. `vitest.config.ts` has no `coverage` block, `package.json` declares no coverage provider (`@vitest/coverage-v8` or `@vitest/coverage-istanbul`), and no `test:coverage` script exists. Per the analysis constraints, no test run or coverage command was executed, so no line, branch or function percentages can be reported. Any numeric coverage claim would be fabricated. The qualitative depth table in Section 10.2 is the available substitute.

### 10.4 Test quality assessment

| Dimension | Assessment | Evidence |
|-----------|------------|----------|
| Assertion strength | Good on primary paths: status, body fields, error codes and direct DB side effects | `tests/orders.test.ts:31-39, 78-86, 126-131, 157-160` |
| Negative coverage | Moderate: invalid payload, duplicate email, wrong password, missing token, missing product, illegal transition, insufficient stock, delete guard | Tests 2, 3, 5, 7, 9, 11, 12, 15 |
| Edge coverage | Limited: many state-machine and pricing branches are not exercised | Section 10.2 |
| Isolation | Strong per test via truncation; weak per run because the database is shared and unguarded | `tests/setup.ts:8-16` |
| Determinism | Mostly deterministic; generated identities use time/random; no shuffle or retry | `tests/helpers/factories.ts:24, 48, 67`; `vitest.config.ts` |
| Test independence | High by construction; each test arranges its own graph and truncation runs before each test | TR-12 |
| Setup diagnostics | Mixed: helper failures are descriptive; several setup requests ignore status | `tests/helpers/factories.ts:36-38`; `tests/orders.test.ts:145-148` |
| Readability | High: descriptive test names, typed overrides, explicit imports, small helper surface | Both suites; `tests/helpers/factories.ts` |
| Maintainability | Moderate: factory defaults and the truncation list are duplicated contracts; hardcoded API paths | Sections 9.1 and Appendix A |
| Failure artifacts | No custom reporters, screenshots, saved responses or log capture | `vitest.config.ts` |
| Performance awareness | No timing or query-count assertions; serial execution | `vitest.config.ts:8-17` |
| Concurrency coverage | None, despite known concurrency risks in the order write path | No concurrent tests; production risk noted in the architectural report |

### 10.5 Test file locations

- `tests/setup.ts` - lifecycle hooks (20 lines).
- `tests/helpers/factories.ts` - factories, app cache, login/bootstrap (83 lines).
- `tests/auth.test.ts` - `Auth flow` suite, tests at lines 6, 19, 30, 41, 54, 65, 79.
- `tests/orders.test.ts` - `Orders` suite, tests at lines 12, 42, 59, 89, 109, 134, 163, 197.
- `vitest.config.ts` - runner configuration (19 lines).
- `package.json:17-18` - `test` and `test:watch` scripts.
- `tsconfig.json:28` - includes tests in type-checking.
- `tsconfig.build.json:10` - excludes tests from the production build.

---

## Appendix A: Assumptions, Ambiguities, and Confidence Notes

1. **Interpretation of "business rules".** The component is test infrastructure and has no domain of its own. Business rules are documented as enforceable testing invariants (TR-01 to TR-12) plus the verification contracts the suites impose on the application (TR-13, TR-14). Production business rules are referenced only as oracles and are not re-analyzed.
2. **Setup-file execution frequency.** Vitest documentation states that `setupFiles` run before each test file, so the connect/truncate/disconnect lifecycle is applied per file. This is treated as High confidence. The exact module-cache sharing across files under `singleFork` with default isolation is a runner-internal detail; the harness does not configure `isolate` and does not assert it, so any statement about `cachedApp` or the Prisma client surviving across files is marked as ambiguous.
3. **Environment availability.** No `.env` file exists in the analyzed working tree (only `.env.example`), and the analysis shell had no `DATABASE_URL`, `JWT_SECRET` or `NODE_ENV` set. The harness's behavior with externally exported variables is inferred from `src/config/env.ts`; tests were not executed, so the actual pass/fail status of `npm test` in any environment is unknown.
4. **`NODE_ENV` behavior.** Vitest sets `NODE_ENV=test` only when it is unset. If a shell exports `development`, the application logger uses the `pino-pretty` transport and Prisma logs warnings, which changes test output but not assertions. The repository itself does not set a test-specific environment.
5. **Test count discrepancy.** The architectural report states that `tests/orders.test.ts` contains 10 integration tests. The file contains 8 `it` blocks. The report in this document uses the verified count of 15 total tests.
6. **Quantitative coverage.** No coverage provider, report, threshold or CI artifact exists. No percentages are reported because none can be derived statically without execution and instrumentation.
7. **Failure-mode claims.** Statements about partial cleanup, abrupt termination and missing after-run cleanup are derived directly from the absence of `try/finally`, `afterEach` cleanup and `afterAll` deletes. They do not depend on runtime observation.
8. **Severity ratings.** Risk levels are analytical judgments based on data-loss potential, reproducibility impact and coverage impact; they are not derived from a formal risk standard.
9. **Production dependencies.** `src/app.ts` efferent coupling (18) and the overall module coupling figures are taken from the existing architectural report; harness-specific counts were computed from the five component files.
10. **Analysis-only confirmation.** No project file was modified. The only file created by this analysis is this report.

---

*This report is analysis-only. No project files were modified, no tests were executed, and no code was refactored. The only file created is this report under `docs/agents/component-deep-analyzer/`.*
