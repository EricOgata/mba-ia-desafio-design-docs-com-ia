# Component Deep Analysis Report

**Component:** Server Bootstrap
**Primary file:** `src/server.ts` (27 lines)
**Project:** `order-management-api` (Order Management System REST API)
**Analysis date:** 2026-11-08
**Runtime:** Node.js >= 20, TypeScript 5.6.3 (ESM), Express 4.21.1, Prisma 5.22.0
**Analysis mode:** Static, read-only, component-boundary analysis (no code executed, no files modified)

## Analysis Scope and Method

The component boundary is the process entry point `src/server.ts` plus the direct collaborators it invokes at startup and shutdown:

- `src/app.ts` (application composition, called by bootstrap)
- `src/config/env.ts` (configuration, imported by bootstrap and by the logger/Prisma providers)
- `src/config/database.ts` (Prisma client provider, imported by bootstrap)
- `src/shared/logger/index.ts` (logger singleton, imported by bootstrap)
- `package.json` scripts and `tsconfig*.json` (how the entry point is built and launched)
- `tests/` (to determine whether the component is covered)

Excluded per instruction: `node_modules`, `.git`, `dist`, `coverage`.

Supporting context was taken from the existing architectural report (`docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md`) and dependency audit (`docs/agents/dependency-auditor/dependencies-report-2026-11-08_22-50-33.md`). This report does not restate the full business analysis of the feature modules; it documents rules that the bootstrap owns or activates.

Coupling metrics were computed statically from `import`/`export from` statements across the 39 TypeScript files in `src` (type-only imports included, external npm packages excluded). For a component that is never imported by another source file, afferent coupling is zero, which makes it maximally unstable by definition but also isolates change impact.

---

## 1. Executive Summary

`src/server.ts` is the outermost boundary of the application: the only module that owns the Node.js process lifecycle. It composes the Express application with a process-wide Prisma client (`src/server.ts:7`), binds an HTTP listener to `env.PORT` (`src/server.ts:9`), emits a structured `server_started` log (`src/server.ts:10`), registers `SIGINT`/`SIGTERM` shutdown handlers (`src/server.ts:20-21`), and converts an unhandled bootstrap rejection into a `bootstrap_failed` fatal log plus exit code 1 (`src/server.ts:24-27`). The whole component is 27 lines and imports exactly four internal modules.

Its architectural role is disproportionate to its size. It is the only place where the process boundary, the listening socket, termination signals, and the Prisma connection pool lifetime meet. It also determines the application's startup contract (environment validation is fail-fast and happens at import time), its observability contract (one structured startup event, one shutdown event per signal), and its exit-code contract (0 for signal-driven shutdown, 1 for bootstrap failure). Afferent coupling is 0 for the module itself; efferent coupling is 4 internal files at file level (or 3 modules if grouped as `app`, `config`, `shared`), matching the architectural report.

Key findings:

1. **Startup is fail-fast and mostly deterministic.** Configuration is validated by Zod before `bootstrap()` runs; invalid configuration prints all issues and exits with code 1. There is no retry, no waiting for the database, and no readiness probe before the listener starts.
2. **Shutdown is only partially graceful.** The component initiates `server.close()` but does not await its callback before calling `process.exit(0)` after `prisma.$disconnect()` (`src/server.ts:15-17`). In-flight requests are terminated, the `http_server_closed` log at line 15 is likely never emitted, and Prisma is disconnected while requests may still be executing.
3. **Bind errors escape the bootstrap error policy.** There is no `server.on('error')` handler, so asynchronous listen failures such as `EADDRINUSE` crash the process through Node's uncaught-exception path rather than through the `bootstrap().catch` branch that emits `bootstrap_failed`.
4. **No shutdown re-entrancy or deadline control.** Repeated signals invoke `shutdown()` concurrently; a hang in `prisma.$disconnect()` prevents `process.exit` with no timeout.
5. **The component has zero direct test coverage.** All 15 integration tests import `buildApp` through `tests/helpers/factories.ts:12`; none imports `src/server.ts`. Signal handling, port binding, exit codes, and startup logging are untested.
6. **Security controls in the activated pipeline are limited but deliberate:** `x-powered-by` disabled, 1 MB JSON body cap, Pino redaction of authorization/cookie/password/token paths, and generic 500 masking. Secrets are environment-only, with weak example defaults and a 16-character minimum for `JWT_SECRET`.

The change blast radius is small (no source file imports `src/server.ts`), but the operational blast radius is total: a defect here prevents the entire API from starting, or silently drops traffic during deploys.

---

## 2. Data Flow Analysis

### 2.1 Process startup flow

```
1. Operator/process manager runs: npm run dev  -> tsx watch --env-file=.env src/server.ts
   or production: npm start -> node --env-file=.env dist/server.js   (package.json:11,13)
2. Node.js loads .env into process.env (native --env-file, no dotenv dependency).
3. ESM module graph is evaluated depth-first:
   a. src/server.ts imports src/app.ts
   b. src/app.ts imports middleware -> src/shared/logger/index.ts -> src/config/env.ts
   c. src/config/env.ts runs loadEnv(): Zod safeParse of process.env; on failure
      prints issues to console.error and process.exit(1) BEFORE any logger exists (src/config/env.ts:14-27)
   d. src/config/database.ts instantiates the process-wide PrismaClient (src/config/database.ts:4-10)
   e. src/shared/logger/index.ts instantiates Pino using the validated env (src/shared/logger/index.ts:13-32)
4. bootstrap() executes (src/server.ts:6):
   a. buildApp({ prisma }) (src/server.ts:7) -> buildControllers() constructs 5 repositories,
      5 services and 5 controllers with a shared PrismaClient (src/app.ts:26-53)
   b. buildApiRouter() mounts auth/users/customers/products/orders under /api/v1 (src/routes/index.ts:21-31)
   c. Express pipeline installed in order: x-powered-by off, json({limit:'1mb'}),
      requestLogger, GET /health, /api/v1, 404 fallback, error middleware (src/app.ts:55-75)
5. app.listen(env.PORT) returns an http.Server bound to all interfaces (src/server.ts:9).
6. Listen callback emits logger.info({ port, env }, 'server_started') (src/server.ts:9-11).
7. process.on('SIGINT') and process.on('SIGTERM') register the shutdown closure (src/server.ts:20-21).
```

### 2.2 HTTP request flow through the activated listener

The component does not process requests itself; it starts the listener that hands requests to `src/app.ts`. The end-to-end path is:

```
1. Client connects to PORT (host resolved by Node; no host argument supplied)
2. Express: express.json({ limit: '1mb' }) parses JSON bodies (src/app.ts:59)
3. requestLogger assigns/propagates X-Request-Id, records start time,
   registers res.on('finish') access log (src/middlewares/request-logger.middleware.ts:5-27)
4. Route matching:
   - GET /health -> static { status: 'ok' } (src/app.ts:62-64)
   - /api/v1/* -> module router (src/routes/index.ts:24-28)
5. Module middleware: authenticate (JWT) and/or zod validate (body/query/params)
6. Controller -> Service -> Repository -> PrismaClient (shared instance) -> MySQL 8.0
7. Response serialized as JSON; res 'finish' event emits `http_request` with
   requestId, method, path, statusCode, durationMs, userId (request-logger.middleware.ts:12-25)
8. Unmatched route -> next(new NotFoundError(...)) -> errorMiddleware returns 404 JSON (src/app.ts:69-73)
9. Thrown AppError/ZodError/Prisma P2002/P2025 -> mapped JSON error (src/middlewares/error.middleware.ts:14-54)
   Any other error -> logger.error + generic 500 INTERNAL_SERVER_ERROR (error.middleware.ts:56-64)
```

### 2.3 Shutdown flow

```
1. Kernel/process manager delivers SIGINT or SIGTERM.
2. Node event loop invokes the async shutdown(signal) closure through `void shutdown(...)`
   (src/server.ts:20-21).
3. logger.info({ signal }, 'shutdown_initiated') (src/server.ts:14).
4. server.close(callback) is invoked but NOT awaited; the callback that would log
   'http_server_closed' is scheduled for later (src/server.ts:15).
   - New connections are refused; existing connections continue per Node semantics.
5. await prisma.$disconnect() closes the database connection pool (src/server.ts:16).
6. process.exit(0) terminates the process immediately (src/server.ts:17).
   - Any in-flight request, the server.close callback, and buffered async log writes can be lost.
7. Exit status observed by the process manager: 0 (success), even if draining did not complete.
```

### 2.4 Failure flow

```
A. Invalid environment:
   env.ts loadEnv -> safeParse failure -> console.error(all issues) -> process.exit(1)
   The bootstrap().catch branch is never reached; no structured log, no Prisma cleanup needed.

B. Synchronous bootstrap rejection (e.g., wiring error, invalid port range thrown by listen):
   bootstrap() rejects -> .catch -> logger.fatal({ err }, 'bootstrap_failed') -> process.exit(1)
   (src/server.ts:24-27)

C. Asynchronous listen error (EADDRINUSE, EACCES):
   http.Server emits 'error'; no listener is registered -> Node throws uncaughtException
   -> process terminates without the bootstrap_failed log and without Prisma disconnect.

D. Runtime unhandled rejection/uncaught exception after startup:
   No process-level handler exists; Node default behavior terminates the process.
   Only per-request errors are contained by errorMiddleware.
```

### 2.5 Data ownership and boundaries

The component owns no domain data. It owns three runtime resources only: the environment snapshot (by import side effect), the HTTP listening socket, and the process-wide PrismaClient reference it passes into the composition root. All persistent data flow (orders, products, customers, users, status history) occurs below the boundary established at `src/server.ts:7`. The component's only emitted data are stdout log records (`server_started`, `shutdown_initiated`, `http_server_closed` intent, `bootstrap_failed`) and the process exit status.

---

## 3. Business Rules & Logic

### 3.1 Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location | Confidence |
|---------|-----------|------------------|----------|------------|
| BR-01 | Configuration / Validation | Environment variables are validated as a single Zod schema at import time; any invalid value aborts the process with exit code 1 and a stderr report of all issues | `src/config/env.ts:3-27` | High |
| BR-02 | Configuration / Constraint | `DATABASE_URL` must be non-empty and `JWT_SECRET` must be at least 16 characters; neither has a default, so the process cannot start without them | `src/config/env.ts:7-8` | High |
| BR-03 | Runtime / Validation | `PORT` is coerced to a positive integer with default 3000; the listener binds to that port without an explicit host | `src/config/env.ts:5`, `src/server.ts:9` | High |
| BR-04 | Observability | A successful bind emits exactly one `server_started` info event carrying `port` and `env`; logging is configured from `LOG_LEVEL` | `src/server.ts:9-11`, `src/shared/logger/index.ts:13-32` | High |
| BR-05 | Composition | Exactly one PrismaClient instance is created per process and injected into every repository/service that needs persistence; no per-request client exists | `src/server.ts:7`, `src/config/database.ts:10`, `src/app.ts:26-53` | High |
| BR-06 | HTTP Surface | The listener publishes the Express pipeline in a fixed order: security header removal, JSON parsing, request logging, `/health`, `/api/v1` routers, 404 fallback, centralized error handler | `src/app.ts:55-75`, `src/routes/index.ts:21-31` | High |
| BR-07 | Lifecycle | `SIGINT` and `SIGTERM` initiate shutdown; no other signals are handled | `src/server.ts:20-21` | High |
| BR-08 | Lifecycle / Ordering | Shutdown logs the signal, initiates HTTP listener closure, disconnects Prisma, then exits with code 0; listener closure is not awaited | `src/server.ts:13-18` | High (intent), High (gap) |
| BR-09 | Data Infrastructure | The Prisma connection pool is explicitly disconnected during shutdown | `src/server.ts:16` | High |
| BR-10 | Process Contract | Signal-driven shutdown exits with 0; configuration/bootstrap failure exits with 1 | `src/server.ts:17`, `src/server.ts:26`, `src/config/env.ts:22` | High |
| BR-11 | Error Handling | Any rejection from `bootstrap()` is logged at fatal level as `bootstrap_failed` with the error object, then exits with code 1 | `src/server.ts:24-27` | High |
| BR-12 | Environment Semantics | `NODE_ENV` must be `development`, `test`, or `production`; `development` enables pretty logs and Prisma warn/error logs, all other values produce JSON logs and Prisma error-only logs | `src/config/env.ts:4`, `src/shared/logger/index.ts:22-28`, `src/config/database.ts:6` | High |

### 3.2 Detailed breakdown of the business rules

---

### Business Rule: BR-01 Fail-fast environment validation

**Overview:**
The process refuses to start with an incomplete or malformed environment. Validation is performed once, synchronously, as a side effect of importing `src/config/env.ts`, before the HTTP listener or the logger exist.

**Detailed description:**
`src/config/env.ts:3-10` declares a Zod object schema for six variables: `NODE_ENV` (enum with default `development`), `PORT` (coerced positive integer with default 3000), `LOG_LEVEL` (enum with default `info`), `DATABASE_URL` (non-empty string, no default), `JWT_SECRET` (string of at least 16 characters, no default), and `JWT_EXPIRES_IN` (string with default `8h`). `loadEnv()` calls `safeParse(process.env)` and, on failure, renders every issue as `  - <path>: <message>` and prints the aggregated block to `console.error`, then calls `process.exit(1)` (`src/config/env.ts:14-25`). On success it returns `parsed.data`, which is exported as a module-level constant at `src/config/env.ts:27`.

This rule has an import-time, not a function-call, trigger. Because `src/shared/logger/index.ts:2` and `src/config/database.ts:2` also import `env`, and `src/server.ts` imports all of them, the validation runs during ESM graph evaluation before `bootstrap()` executes. The same is true of tests: `tests/setup.ts:2` imports `src/config/database.ts`, so any test run validates the environment and can terminate the Vitest worker process with exit code 1 if `DATABASE_URL` or `JWT_SECRET` is missing. Validation is all-or-nothing and aggregated: a single invalid variable does not short-circuit discovery of the others, which is useful for operators correcting `.env` in one pass.

Several constraints are weaker than their names suggest. `DATABASE_URL` is only checked for non-empty length, so a syntactically malformed MySQL URL is accepted at boot and fails later at the first Prisma query. `JWT_SECRET` enforces length only, not entropy or disallowed placeholder values, which is why the `.env.example:8` placeholder (`change-me-in-production-please-use-a-long-random-secret`) passes validation. `JWT_EXPIRES_IN` accepts any string, including an empty string, with the `jsonwebtoken` library performing the effective interpretation later. Failure output goes to raw stderr rather than the structured logger because the logger depends on the very configuration being validated, which means environment failures are not visible in JSON log aggregation unless stderr is captured separately.

**Rule workflow:**
1. Node starts with `--env-file=.env` (or the environment is injected by the platform).
2. The ESM loader evaluates `src/config/env.ts` as part of the import graph.
3. `loadEnv()` calls `envSchema.safeParse(process.env)`.
4. Failure: issues are joined into a multi-line message, printed via `console.error`, `process.exit(1)` is called; nothing else in the application starts.
5. Success: `parsed.data` is assigned to the exported `env` constant; all consumers read from this frozen-in-time snapshot.
6. Later changes to `process.env` are not reflected, because the snapshot is taken once.

---

### Business Rule: BR-02 Mandatory stateful and security configuration

**Overview:**
The application has exactly one stateful external dependency (MySQL through Prisma) and one symmetric security secret (JWT HS256). Both must be supplied by the environment; there are no built-in fallbacks.

**Detailed description:**
`DATABASE_URL` (`src/config/env.ts:7`) is the only channel through which the process learns where MySQL lives. Because `src/config/database.ts:4-8` constructs `PrismaClient` with only a `log` option, Prisma resolves the datasource URL from the environment at client construction and connection time. There is no alternate configuration file, no secret manager integration, and no validation that the URL points to MySQL versus another Prisma-supported provider; the schema (`prisma/schema.prisma`) defines the provider separately. If the database is unreachable, startup still succeeds because the connection is lazy, and the failure surfaces on the first query handled by the Express pipeline.

`JWT_SECRET` (`src/config/env.ts:8`) is required with a minimum length of 16 characters. It is consumed by the authentication service and middleware, not by `src/server.ts` itself, but bootstrap cannot proceed without it because `env` is loaded as a unit. This couples startup availability to a security setting that is operationally independent from process startup, meaning a deployment that only needs to serve public health checks cannot start without a valid secret. The 16-character floor is permissive relative to modern HMAC key guidance, and there is no rotation, key-id, or multi-secret support in the configuration surface.

Two related variables exist outside the schema. `SHADOW_DATABASE_URL` (`.env.example:6`) is required by `prisma migrate dev` but never validated by the application, and `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD` are consumed only by `docker-compose.yml`. This means a "valid" application environment can still fail during migrations, and a Compose stack can start with default credentials regardless of the application's own validation.

**Rule workflow:**
1. `envSchema.safeParse` checks presence and minimum length.
2. Missing or too-short values produce issues associated with the `DATABASE_URL` or `JWT_SECRET` path.
3. `loadEnv()` prints the issues and exits 1.
4. Valid values are passed to Prisma (via `process.env` resolution) and to the auth module (via the exported `env` object).
5. No connectivity verification occurs during startup; the first HTTP request that touches the database performs the first connection attempt.

---

### Business Rule: BR-03 Port resolution and listener binding

**Overview:**
The HTTP listener uses a single validated port and binds it without an explicit network interface, so the effective listen address is determined by Node.js defaults (all IPv6/IPv4 interfaces).

**Detailed description:**
`PORT` is declared as `z.coerce.number().int().positive().default(3000)` (`src/config/env.ts:5`). Coercion means string values such as `"3000"` are accepted, and a missing value becomes 3000. Zero and negative numbers are rejected because of `.positive()`, while values above 65535 are not rejected by the schema. An out-of-range port is caught later because Node's `server.listen` throws a `RangeError` synchronously, which propagates out of `bootstrap()` and is handled by the `bootstrap_failed` branch. The schema does not restrict privileged ports, so a deployment running as root could bind port 80 or 443 even though no TLS termination exists in the application.

`src/server.ts:9` calls `app.listen(env.PORT, callback)` with no host argument. Node therefore listens on the unspecified IPv6 address (`::`) when available, accepting IPv4-mapped connections as well, which is equivalent to all interfaces. There is no `HOST`, `BIND_ADDRESS`, or `LISTEN_HOST` configuration, so the component cannot be restricted to loopback through configuration; restriction must come from the container network, firewall, or reverse proxy. No `backlog`, `exclusive`, or `ipv6Only` option is set, and no socket-level timeouts are configured, so Node defaults apply to `headersTimeout`, `requestTimeout`, and `keepAliveTimeout`.

Only one listener is created per process. There is no cluster mode, worker-thread HTTP server, or SO_REUSEPORT configuration, which aligns with the architectural finding that the system is a single Node process and a single point of failure. The listen callback fires only after the bind succeeds, so `server_started` is a reliable signal that the socket is accepting connections, but not that the database is reachable.

**Rule workflow:**
1. `PORT` is read from the validated env snapshot (default 3000).
2. `app.listen(PORT, callback)` requests a bind on all interfaces.
3. Success: the socket is listening; the callback logs `server_started` with `port` and `env`.
4. Synchronous failure (invalid port, permission denied on some platforms): the throw rejects `bootstrap()`, producing `bootstrap_failed` and exit 1.
5. Asynchronous failure (`EADDRINUSE`, `EACCES` emitted as `error`): no `error` listener exists, so the process crashes outside the bootstrap error policy.

---

### Business Rule: BR-04 Structured startup observability

**Overview:**
A successful start must be observable as exactly one structured event that identifies the listening port and the runtime environment.

**Detailed description:**
The listen callback emits `logger.info({ port: env.PORT, env: env.NODE_ENV }, 'server_started')` (`src/server.ts:10`). The logger is a Pino singleton configured at `src/shared/logger/index.ts:13-32` with `level` from `LOG_LEVEL`, base fields `{ service: 'order-management-api', env: NODE_ENV }`, ISO timestamps, and a `development`-only `pino-pretty` transport with colorization and `HH:MM:ss.l` timestamps. In `test` and `production`, output is newline-delimited JSON written to stdout. Pino redaction censors `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, and `*.accessToken` with `[REDACTED]` (`src/shared/logger/index.ts:4-11`).

The startup event is minimal by design: it does not include the resolved bind address, the application version, the database target, the migration state, or the process ID. Because the event is emitted only in the listen callback, an asynchronous bind failure produces no `server_started` event and no alternative structured event. The fatal startup path uses a different event name, `bootstrap_failed` (`src/server.ts:25`), and the environment-failure path emits no event at all because the logger does not yet exist. An operator therefore has three distinct startup failure shapes to correlate: raw stderr from Zod validation, a fatal JSON record from bootstrap rejection, and an uncaught-exception stack trace from bind errors.

Log level is itself a startup configuration rule: an invalid `LOG_LEVEL` terminates the process, and setting `LOG_LEVEL` above `info` suppresses the `server_started` and `shutdown_initiated` events, leaving only warnings and errors. The logger is initialized once at import time, before bootstrap, so the level cannot change at runtime.

**Rule workflow:**
1. Environment supplies `LOG_LEVEL` and `NODE_ENV`.
2. Pino singleton is constructed during module evaluation.
3. Bind succeeds and the listen callback runs.
4. One `server_started` record is written at `info` level with `port` and `env`.
5. In development, the record is rendered by `pino-pretty` on a worker thread; elsewhere it is raw JSON on stdout.
6. If the process exits before the transport flushes, records can be lost; this is most likely during `process.exit` in shutdown.

---

### Business Rule: BR-05 Single process-wide composition and Prisma client

**Overview:**
Bootstrap creates exactly one PrismaClient and injects it into the entire object graph; every HTTP request shares that client and its connection pool.

**Detailed description:**
`src/config/database.ts:4-8` exposes `createPrismaClient()`, which passes a `log` option that depends on `NODE_ENV` (`['warn','error']` in development, `['error']` otherwise), and `src/config/database.ts:10` immediately instantiates and exports a singleton `prisma`. `src/server.ts:3` imports that singleton and `src/server.ts:7` passes it to `buildApp({ prisma })`. Inside `buildApp`, `buildControllers` (`src/app.ts:26-53`) constructs five repositories with the same client, five services around those repositories, and five controllers around the services; `OrderService` additionally receives the raw client (`src/app.ts:43`) because it manages transactions directly. There is no request-scoped client, no connection-per-request pattern, and no alternate data source.

The practical consequence is a single Prisma connection pool sized by Prisma defaults for the MySQL provider, shared by all concurrent requests. Startup does not call `prisma.$connect()`; Prisma connects lazily on the first query, so `server_started` does not imply database readiness. The same singleton is therefore a hard runtime dependency for every stateful route and a single point of failure for throughput and connection exhaustion, as noted in the architectural report. Because the singleton is created at module evaluation, an application instance created by tests or tooling always shares the same client within a process, and `prisma.$disconnect()` in `src/server.ts:16` affects all consumers.

This rule also establishes the composition order: repositories before services before controllers, module routers before the API router, and the API router before the 404 and error middleware. Any failure during these constructors propagates synchronously through `buildApp` and becomes a `bootstrap_failed` fatal log. Constructor injection is manual; there is no DI container, so the wiring in `src/app.ts:26-53` is the only place where module dependencies are resolved.

**Rule workflow:**
1. `src/config/database.ts` evaluates and creates the singleton client from the validated env.
2. `bootstrap()` calls `buildApp({ prisma })`.
3. `buildControllers` instantiates 5 repositories, 5 services, 5 controllers with constructor injection.
4. `buildApiRouter` binds the five controllers into routers under `/api/v1`.
5. Express mounts the composed app.
6. Every request handler uses the same client until the process exits or the client is disconnected.

---

### Business Rule: BR-06 Fixed HTTP pipeline and route publication order

**Overview:**
The listener publishes a fixed Express pipeline in which cross-cutting middleware, the health probe, the versioned API, the 404 fallback, and the error handler have a mandatory order.

**Detailed description:**
`buildApp` applies `app.disable('x-powered-by')` (`src/app.ts:58`), then `express.json({ limit: '1mb' })` (`src/app.ts:59`), then the request logger (`src/app.ts:60`). The health route is registered next (`src/app.ts:62-64`), so it is subject to the JSON parser and request logging but not to authentication or API validation. The versioned router is mounted at `/api/v1` (`src/app.ts:67`), with sub-routers for `/auth`, `/users`, `/customers`, `/products`, and `/orders` (`src/routes/index.ts:24-28`). After the router, a catch-all middleware raises `NotFoundError` for any unmatched method/path (`src/app.ts:69-71`), and the centralized error middleware is registered last (`src/app.ts:73`) so that it can translate `AppError`, `ZodError`, and selected Prisma errors into JSON.

Order is semantically significant. Moving the 404 middleware before the API router would mask every API route; moving the error middleware before the router would prevent it from catching downstream errors. The 1 MB JSON cap applies to all routes, including unauthenticated ones, and is the only request-size control at bootstrap scope. The health probe is unconditional: `{ status: 'ok' }` with HTTP 200 and no database check, so it functions as a liveness indicator only, not readiness. Authentication is per-router, not global: `/health` and `/api/v1/auth/register` and `/api/v1/auth/login` are public, while customers, products, orders, and the user lookup apply `authenticate`, and only the user lookup applies `requireRole('ADMIN')`.

The pipeline also defines the error contract that bootstrap exposes: application errors carry `{ error: { code, message, details? } }`; validation errors return 400 `VALIDATION_ERROR` with an array of `{path,message}`; Prisma `P2002` returns 409 `CONFLICT` with the target columns; `P2025` returns 404 `NOT_FOUND`; unmatched API routes return 404 through the shared error taxonomy; all other errors return 500 `INTERNAL_SERVER_ERROR` after a `logger.error` that includes the raw error object and the request id (`src/middlewares/error.middleware.ts:14-64`).

**Rule workflow:**
1. Request enters the listener started at `src/server.ts:9`.
2. JSON body is parsed with a 1 MB limit.
3. Request id is assigned (`x-request-id` header or UUID) and echoed on the response.
4. `/health` short-circuits with a static 200.
5. `/api/v1/*` is matched; module middleware authenticates and validates.
6. Controller/service/repository handle the request through the shared Prisma client.
7. Unmatched routes fall through to the 404 middleware and then the error middleware.
8. All response paths are logged on `finish` with status and duration.

---

### Business Rule: BR-07 Termination signal handling

**Overview:**
Only `SIGINT` and `SIGTERM` are treated as shutdown requests, and both invoke the same async closure.

**Detailed description:**
`src/server.ts:20-21` registers `process.on('SIGINT', () => void shutdown('SIGINT'))` and the equivalent for `SIGTERM`. Registration happens in the same synchronous execution as the `listen` call, so signals arriving after the module body completes are handled. The `void` operator discards the returned promise, meaning any rejection inside `shutdown` becomes an unhandled rejection rather than a logged fatal error. There is no `SIGHUP`, `SIGQUIT`, or `SIGUSR2` handling, so hot reload or terminal-close semantics depend on the supervisor (for example, `tsx watch` restarts the process rather than triggering application shutdown).

The closure is not guarded by a flag. Two rapid `SIGTERM` deliveries (common with process managers that escalate, or with `docker stop` followed by a forced kill) can run two shutdowns concurrently: two `shutdown_initiated` logs, two `server.close` calls, two `$disconnect` calls, and two `process.exit(0)` calls. The second disconnect is likely to reject because the client is already disconnected, producing the unhandled rejection described above. In container environments where the process is PID 1, Node's default signal disposition would normally terminate the process; registering handlers overrides that, which is why this rule matters for restart latency and exit behavior.

Signal handling is registered after the `listen` invocation but before the listen callback is guaranteed to run. If the bind fails asynchronously, the handlers are still installed, but the process is already on an uncaught-exception path. The component does not remove its listeners on shutdown, does not expose a shutdown state, and does not coordinate with in-flight requests beyond initiating `server.close`.

**Rule workflow:**
1. Process manager sends `SIGTERM` (typical) or the operator presses Ctrl+C (`SIGINT`).
2. Node dispatches to the registered listener.
3. `shutdown(signal)` starts; its promise is intentionally discarded.
4. The shutdown sequence proceeds (BR-08 and BR-09).
5. A repeated signal starts a second, concurrent shutdown sequence with no coordination.
6. The process ends via `process.exit(0)` after the first sequence reaches line 17.

---

### Business Rule: BR-08 Shutdown ordering: listener closure, then database, then exit

**Overview:**
The intended shutdown order is: log the signal, stop accepting HTTP connections, release the database pool, then exit successfully. The implementation initiates but does not complete the HTTP closure before exit.

**Detailed description:**
`src/server.ts:13-18` defines `shutdown(signal)`: it logs `shutdown_initiated` with the signal, calls `server.close(() => logger.info('http_server_closed'))`, awaits `prisma.$disconnect()`, and calls `process.exit(0)`. The `server.close` call is asynchronous and callback-based; its returned value is not awaited and the callback is not promisified. Consequently the sequence that actually completes is: log, initiate close, await disconnect, exit. In-flight requests are not drained, and any handler that performs a query after the disconnect can fail with a disconnected-client error instead of completing normally. The `http_server_closed` log is scheduled in a callback that can never run before `process.exit` unless the disconnect happens to yield long enough and the server has no connections.

The rule is therefore best characterized as "HTTP closure is initiated before database disconnection", not "HTTP traffic is drained before exit". Node's `server.close` stops accepting new connections and waits for existing connections; with keep-alive connections, draining can take as long as the keep-alive timeout, which is one reason awaiting it matters. The code also does not call `server.closeAllConnections()` or `closeIdleConnections()`, and does not impose a deadline, so a correct implementation of the intent would still need a bound on drain time. There is no tracking of in-flight request count, no readiness-state toggle, and no response header or health-state change to tell load balancers to stop routing before shutdown.

The ordering has a second consequence: because Prisma is disconnected before HTTP traffic is guaranteed to be finished, the window between lines 15 and 16 is the only point where the process is in a semantically consistent state for draining, and the code does not wait there. For deployment platforms that send `SIGTERM` and then wait a grace period, the process will usually exit within milliseconds, so the platform's connection draining must be provided externally by a load balancer or reverse proxy. The repository contains neither, per the architectural report.

**Rule workflow:**
1. Signal is received and `shutdown_initiated` is logged.
2. `server.close(callback)` is invoked; new connections are refused from this point.
3. Existing connections and the close callback continue independently of the awaited chain.
4. `await prisma.$disconnect()` resolves (or rejects) and the connection pool is torn down.
5. `process.exit(0)` terminates the event loop immediately.
6. The close callback, any in-flight request, and buffered logs may never complete.
7. The process manager observes exit code 0 and considers the shutdown successful.

---

### Business Rule: BR-09 Prisma pool disconnection on shutdown

**Overview:**
The process must explicitly close the shared Prisma connection pool when terminating, rather than relying on the OS to reclaim sockets.

**Detailed description:**
`src/server.ts:16` awaits `prisma.$disconnect()`. This targets the singleton imported at `src/server.ts:3` and therefore affects every repository, service, and the transactional `OrderService` path. The call happens after `server.close` is initiated and before `process.exit(0)`. It is the only cleanup action in the shutdown path: there is no cache flush, no outbox drain, no queue close, and no telemetry flush because the current system has none of those resources.

The call has no timeout and no try/catch. If MySQL is unreachable or a connection is mid-query, `$disconnect` can take as long as its internal behavior allows, delaying the exit; if it rejects, the rejection is swallowed by `void shutdown(...)` at lines 20-21 and becomes an unhandled rejection. Node's default unhandled-rejection behavior in modern versions is to terminate the process, which would produce a nonzero exit and no `http_server_closed` log, contradicting the intended exit-code contract. Because the process is exiting anyway, the practical benefit of the disconnect is orderly TCP teardown and avoiding connection slots lingering at the database until TCP keepalive detects the close.

There is also an interaction with tests: `tests/setup.ts:18-20` calls `prisma.$disconnect()` in `afterAll` against the same singleton pattern, confirming that the provider is designed for explicit lifecycle management. In the server, that lifecycle is exactly one connect-on-demand and one disconnect-on-signal; there is no reconnect strategy if the pool is disconnected while the process remains alive.

**Rule workflow:**
1. Shutdown reaches line 16 after initiating HTTP closure.
2. `prisma.$disconnect()` closes the shared connection pool.
3. Success: the promise resolves and execution continues to `process.exit(0)`.
4. Failure: the promise rejects; the error is unhandled because the caller used `void`.
5. No retry and no timeout exist; the outcome does not change the exit code when it succeeds.

---

### Business Rule: BR-10 Exit code contract

**Overview:**
The process distinguishes normal signal-driven termination (exit 0) from configuration or bootstrap failure (exit 1), and uses no other explicit exit codes.

**Detailed description:**
Three explicit exits exist in the component boundary: `process.exit(0)` at `src/server.ts:17` after a signal-driven shutdown; `process.exit(1)` at `src/server.ts:26` after a `bootstrap_failed` fatal log; and `process.exit(1)` at `src/config/env.ts:22` after invalid environment output. All other terminations are implicit: Node's default exit code for an uncaught exception or unhandled rejection is 1, and a clean event-loop drain is 0. There is no distinct code for "shutdown timed out", "database disconnect failed", or "bind failed", so supervisors cannot differentiate causes from the exit status alone.

The signal path always exits 0 once it reaches line 17, regardless of whether HTTP draining completed, whether requests were dropped, or whether log records were flushed. If `$disconnect` rejects, line 17 may never be reached and the exit code becomes whatever Node's unhandled-rejection policy produces, which creates an inconsistency between the intended and actual contract. Environment failure exits 1 before any structured logging, so exit code plus stderr is the only diagnostic pair. Bootstrap failure exits 1 after a structured fatal log, so it is the most observable failure shape.

No exit code is reserved for "started but not ready". Since the listener starts before database connectivity is verified, a process that is up but unable to serve stateful routes still reports a successful start and, if later killed, exits 0. This is consistent with the liveness-only health endpoint and with the absence of orchestrator readiness probes in the repository.

**Rule workflow:**
1. Invalid env: print issues, exit 1.
2. Bootstrap rejection: log fatal, exit 1.
3. Bind error or runtime crash: Node default, exit 1 without structured startup/shutdown records.
4. Signal-driven shutdown reaching line 17: exit 0.
5. Signal-driven shutdown rejecting before line 17: unhandled rejection, default nonzero exit.
6. In all cases, the last observed exit status is the only machine-readable shutdown outcome.

---

### Business Rule: BR-11 Fatal bootstrap error policy

**Overview:**
Any rejection from `bootstrap()` is treated as unrecoverable: it is logged at fatal level with the error object and the process exits 1.

**Detailed description:**
`src/server.ts:24-27` attaches `.catch((err) => { logger.fatal({ err }, 'bootstrap_failed'); process.exit(1); })` to the bootstrap promise. The scope covers everything inside `bootstrap()`: `buildApp` and all constructor wiring, `buildControllers`, `buildApiRouter`, the synchronous portion of `app.listen` (including port range validation), and the registration of signal handlers. It does not cover errors thrown later in the listen callback, asynchronous `error` events from the HTTP server, rejections inside `shutdown` (which is detached with `void`), or runtime request errors (which are handled by the Express error middleware).

The policy has no cleanup step. If failure occurs after the Prisma client was constructed but before or during listen, the catch logs and exits without calling `$disconnect`; the process boundary reclaims the sockets, so this is acceptable for a fatal startup, but it means there is no chance for a graceful database handshake teardown. The fatal log includes the full error object, which is useful for diagnostics but bypasses the redaction patterns only in the sense that Pino redaction applies to configured paths; arbitrary fields on an error object are serialized as-is. The log is written immediately before `process.exit(1)`, so asynchronous transport flushing can drop it, particularly in production where the destination is stdout and the exit is immediate.

Because `logger` is imported at `src/server.ts:4`, the catch handler is available even if bootstrap fails early. However, if the failure originates in `src/config/env.ts`, the process has already exited inside module evaluation and this handler never runs. The fatal policy therefore applies to "valid configuration, broken wiring or listen" rather than to all startup failures.

**Rule workflow:**
1. `bootstrap()` is invoked at module top level (`src/server.ts:24`).
2. Rejection propagates to the attached `catch`.
3. `logger.fatal({ err }, 'bootstrap_failed')` is emitted with base fields `service` and `env`.
4. `process.exit(1)` terminates the process without further cleanup.
5. The supervisor observes exit 1 and a fatal JSON record (if the write flushed).
6. Env-validation and asynchronous bind failures do not follow this path.

---

### Business Rule: BR-12 Environment-conditional runtime behavior

**Overview:**
`NODE_ENV` is a closed enum whose value changes logging output and Prisma log categories, with no effect on the bootstrap sequence itself.

**Detailed description:**
`src/config/env.ts:4` restricts `NODE_ENV` to `development`, `test`, or `production`, defaulting to `development`. Any other value, including common deployment values such as `staging`, `qa`, or `prod`, fails validation and exits 1. Within the allowed set, `development` selects two behaviors: the logger uses a `pino-pretty` transport with colorization and short timestamps (`src/shared/logger/index.ts:22-28`), and the Prisma client logs `warn` and `error` events (`src/config/database.ts:6`). `test` and `production` both produce JSON logs on stdout and restrict Prisma logs to `error`, so test runs are not visually distinguishable from production in log format except through the `env` base field.

The bootstrap sequence does not branch on `NODE_ENV`. The same port resolution, composition, listen, signal registration, and shutdown behavior apply in all three environments. This means there is no development-only pretty error page, no test-only listener suppression, no production-only trust-proxy configuration, and no environment-specific shutdown grace period. The only runtime consumers of the value at the component boundary are the logger factory and the Prisma factory; the exported `env.NODE_ENV` is also logged in `server_started` and in every request log via the logger base.

Because validation is import-time and the value is captured once, changing `NODE_ENV` after startup has no effect. The default of `development` means that a deployment which forgets to set the variable will start successfully with pretty-printed logs and warn-level Prisma logging, rather than failing closed. This is a deliberate convenience for local development but changes log volume and format in unintentional deployments.

**Rule workflow:**
1. `NODE_ENV` is read and validated against the enum.
2. Missing value: default `development`.
3. Invalid value: validation failure, exit 1.
4. `development`: Pino uses `pino-pretty`; Prisma logs `warn` and `error`.
5. `test` or `production`: Pino writes JSON; Prisma logs `error` only.
6. Bootstrap behavior is otherwise identical in all environments.

---

## 4. Component Structure

```
src/
├── server.ts                        # Component boundary: bootstrap(), listen, signal handlers, fatal catch
├── app.ts                           # Composition root invoked by bootstrap; Express pipeline + controllers wiring
├── routes/
│   └── index.ts                     # Aggregates five module routers under /api/v1; defines Controllers contract
├── config/
│   ├── env.ts                       # Zod env schema, loadEnv(), import-time process.exit on invalid config
│   └── database.ts                  # createPrismaClient() + process-wide prisma singleton
├── shared/
│   └── logger/
│       └── index.ts                 # createLogger() + pino singleton, redaction, dev transport
└── middlewares/
    ├── request-logger.middleware.ts # X-Request-Id, duration, http_request access log
    └── error.middleware.ts          # AppError/ZodError/Prisma mapping, generic 500 + logging
```

Direct collaborator responsibilities:

| File | Role relative to Server Bootstrap | Key lines |
|------|-----------------------------------|-----------|
| `src/server.ts` | Owns process lifecycle: compose, listen, signal handlers, fatal catch | 6-27 |
| `src/app.ts` | Builds Express app and object graph; defines `/health`, `/api/v1`, 404 and error handlers | 22-75 |
| `src/routes/index.ts` | Mounts module routers; defines `Controllers` type used by composition | 13-31 |
| `src/config/env.ts` | Validates and snapshots configuration at import time | 3-27 |
| `src/config/database.ts` | Creates singleton PrismaClient with environment-dependent log level | 4-10 |
| `src/shared/logger/index.ts` | Creates singleton Pino logger with redaction and dev transport | 4-32 |
| `src/middlewares/request-logger.middleware.ts` | Emits per-request logs through the same logger singleton | 5-28 |
| `src/middlewares/error.middleware.ts` | Emits unhandled-error logs through the same logger singleton | 14-65 |

The component has no subdirectories, no interfaces, no classes, and no exported symbols used elsewhere. Its only exported production-side surface is the implicit module side effect (calling `bootstrap()` at line 24); `bootstrap` and `shutdown` are not exported. In TypeScript terms, `src/server.ts` is a pure entry-point module with two local function declarations and one top-level invocation.

---

## 5. Dependency Analysis

### 5.1 Internal dependency graph

```
package.json scripts (dev/start)
        |
        v
  src/server.ts
   |    |     \----------------------\
   |    |                             \
   v    v                              v
src/app.ts                     src/config/database.ts
   |                                   |
   |                                   v
   |                             src/config/env.ts
   |                                   ^
   v                                   |
src/middlewares/*  ------> src/shared/logger/index.ts
   |                                   |
   v                                   v
src/modules/* ------+          src/config/env.ts
src/routes/index.ts|
src/shared/errors/*|
        |
        v
   PrismaClient -> MySQL 8.0
```

Explicit import chains from the component:

```
src/server.ts:1  -> src/app.ts
src/server.ts:2  -> src/config/env.ts
src/server.ts:3  -> src/config/database.ts
src/server.ts:4  -> src/shared/logger/index.ts

src/app.ts:3     -> src/middlewares/request-logger.middleware.ts (-> shared/logger -> config/env)
src/app.ts:4     -> src/middlewares/error.middleware.ts (-> shared/errors, shared/logger)
src/app.ts:5-19  -> five repositories, five services, five controllers, routes/index, shared/errors
src/app.ts:19    -> src/routes/index.ts (-> five module .routes.ts files)

src/config/database.ts:1 -> @prisma/client
src/config/database.ts:2 -> src/config/env.ts
src/shared/logger/index.ts:1 -> pino
src/shared/logger/index.ts:2 -> src/config/env.ts
src/config/env.ts:1 -> zod
```

ESM note: all internal specifiers use the `.js` extension in TypeScript source (for example `./app.js` at `src/server.ts:1`), which is required for the NodeNext/ESM output used by `node dist/server.js`. The tsconfig uses `moduleResolution: Bundler`, `module: ESNext`, `target: ES2022`, and `tsconfig.build.json` compiles `src` to `dist` with `rootDir: ./src` and source maps disabled.

### 5.2 External dependencies at the component boundary

| Package | Version | Used at | Purpose in this component |
|---------|---------|---------|---------------------------|
| `express` | 4.21.1 | `src/app.ts:1` via bootstrap | HTTP application and listener (`app.listen`) |
| `@prisma/client` | 5.22.0 | `src/config/database.ts:1`, `src/app.ts:2` (type) | Database client singleton and type contract |
| `pino` | 9.5.0 | `src/shared/logger/index.ts:1` | Structured logging for startup/shutdown/fatal events |
| `zod` | 3.23.8 | `src/config/env.ts:1` | Environment schema validation |
| `pino-pretty` | 11.3.0 (dev) | `src/shared/logger/index.ts:25` (development only) | Human-readable log transport |
| `tsx` | 4.19.2 (dev) | `package.json:11` | Runs TypeScript entry point in watch mode |
| `typescript` | 5.6.3 (dev) | `package.json:12` | Builds `dist/server.js` for `npm start` |
| Node.js built-ins | >= 20 | `src/server.ts:9,13-21,24-27` | `http.Server`, `process.on`, `process.exit`, `--env-file` |

Transitive runtime dependencies are not declared by the component directly. Notably, `pino-http` 10.3.0 is declared in `package.json:31` but is not imported anywhere; the request logger is a hand-written Express middleware. This is documented in the dependency audit as dead dependency surface and is relevant to bootstrap because the logger initialization path is where `pino-http` would typically be used.

### 5.3 Build and launch dependency chain

```
npm run dev   -> tsx watch --env-file=.env src/server.ts
npm run build -> tsc -p tsconfig.build.json (src -> dist, source maps off)
npm start     -> node --env-file=.env dist/server.js
```

`--env-file` requires Node.js 20.6 or later, while `package.json:8` declares `"node": ">=20"`. Node 20.0 through 20.5 satisfy the engine declaration but do not support the flag, so the documented start commands can fail before application code runs on those versions. `docker-compose.yml` provides MySQL only; there is no application image, so the runtime host must supply Node, `.env`, and a migrated database.

### 5.4 Dependency lifecycle summary

| Dependency | Initialized when | Disposed when | Failure mode if unavailable |
|------------|------------------|---------------|-----------------------------|
| `env` snapshot | Import of `src/config/env.ts` | Never (immutable snapshot) | `console.error` + exit 1 |
| Pino logger | Import of `src/shared/logger/index.ts` | Process exit (no explicit flush/close) | Log writes silently degrade; no startup failure |
| PrismaClient | Import of `src/config/database.ts` (lazy connect) | `prisma.$disconnect()` at `src/server.ts:16` | Startup succeeds; first query fails |
| Express app | `buildApp` at `src/server.ts:7` | Never (claims no resources) | Constructor throw -> `bootstrap_failed`, exit 1 |
| HTTP server | `app.listen` at `src/server.ts:9` | `server.close` initiated at line 15, not awaited | Async bind error escapes error policy |
| Signal handlers | Lines 20-21 after listen call | Never removed | Repeated signals run concurrent shutdowns |

---

## 6. Afferent and Efferent Coupling

Coupling units for a TypeScript/Node ESM codebase are modules and exported/local functions. Afferent coupling (Ca) counts distinct internal source files that import the unit; efferent coupling (Ce) counts distinct internal source files the unit imports. External npm packages are excluded from totals and listed separately in Section 5. This yields a slightly different file-level count than the architecture report's module grouping: the report lists Server Bootstrap Ce = 3 modules (`app`, `config`, `shared`), while at file level it imports 4 files (`app.ts`, `config/env.ts`, `config/database.ts`, `shared/logger/index.ts`). Both are consistent; the report's inventory table already states Ce = 4.

| Component / Unit | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Instability I = Ce/(Ca+Ce) | Criticality | Notes |
|------------------|------------------------|------------------------|---------------------------|-------------|-------|
| `src/server.ts` (module) | 0 | 4 files | 1.00 | Critical | No source file imports it; only npm scripts reference the path |
| `bootstrap()` function | 1 (top-level call at `src/server.ts:24`) | 5 (buildApp, app.listen, logger, prisma, process) | 0.83 | Critical | Sole owner of startup ordering |
| `shutdown()` closure | 2 (SIGINT, SIGTERM listeners) | 4 (logger, server.close, prisma.$disconnect, process.exit) | 0.67 | Critical | Detached with `void`; no export |
| `bootstrap().catch` handler | 1 | 2 (logger.fatal, process.exit) | 0.67 | High | Covers only bootstrap rejection |
| `src/app.ts` (`buildApp`) | 2 in repo (1 src + 1 test helper: `tests/helpers/factories.ts:12`) | 18 files | 0.90 | High | Composition root triggered by bootstrap |
| `src/config/env.ts` | 5 src files + tests via database/logger | 0 | 0.00 | Critical | Most stable unit; import-time exit |
| `src/config/database.ts` (`prisma` singleton) | 1 src file (`src/server.ts`) | 1 (`src/config/env.ts`) | 0.50 | High | Also consumed transitively via injected parameter |
| `src/shared/logger/index.ts` | 3 src files (`server`, request logger, error middleware) | 1 (`src/config/env.ts`) | 0.25 | High | Startup/shutdown observability channel |
| HTTP server object (`app.listen` result) | 1 (`shutdown` closure) | 0 internal | n/a (runtime object) | High | No error listener, not exposed |
| PrismaClient instance (runtime object) | 5 repositories + 5 services + server pass-through | MySQL socket | n/a | Critical | Shared pool, disconnected on signal |

Interpretation:

- `src/server.ts` is maximally unstable by construction (Ca = 0). Changes to it cannot ripple into other code, but every change affects process startup directly. Its stability must come from review and operational testing, not from internal contracts.
- `src/config/env.ts` is the most stable unit in the system (Ce = 0, Ca distributed across config, logger, auth, middleware, server, tests). It is a leaf dependency that terminates the graph and, unusually, performs a lifecycle action (`process.exit`) as an import side effect.
- The logger is moderately stable and is a single shared observability channel. Any failure in logger construction happens during module evaluation and is not covered by the bootstrap fatal handler.
- The component's total dependency fan-out is small (4 internal files) but includes almost every cross-cutting infrastructure concern in the system, so a change in config, logging, composition, or database infrastructure is likely to intersect the bootstrap path even when `src/server.ts` itself is unchanged.

---

## 7. Endpoints

`src/server.ts` defines no routes, controllers, or protocol handlers. It creates the listener that publishes the Express surface composed in `src/app.ts` and the module routers. The table below records the exact surface activated by `app.listen(env.PORT)` so that the component's externally visible effect is documented; ownership is outside the component boundary.

| Endpoint | Method | Defined in | Description | Auth |
|----------|--------|------------|-------------|------|
| `/health` | GET | `src/app.ts:62-64` | Static liveness response `{ status: 'ok' }`; no database check | Public |
| `/api/v1/auth/register` | POST | `src/modules/auth/auth.routes.ts:9` | Register user, returns 201 public projection | Public |
| `/api/v1/auth/login` | POST | `src/modules/auth/auth.routes.ts:10` | Issue JWT access token | Public |
| `/api/v1/auth/me` | GET | `src/modules/auth/auth.routes.ts:11` | Return authenticated identity | Bearer JWT |
| `/api/v1/users/:id` | GET | `src/modules/users/user.routes.ts:12-18` | Admin-only user lookup | Bearer JWT + ADMIN |
| `/api/v1/customers` | GET | `src/modules/customers/customer.routes.ts:16` | Paginated/searchable customer list | Bearer JWT |
| `/api/v1/customers/:id` | GET | `src/modules/customers/customer.routes.ts:17` | Get customer by UUID | Bearer JWT |
| `/api/v1/customers` | POST | `src/modules/customers/customer.routes.ts:18` | Create customer | Bearer JWT |
| `/api/v1/customers/:id` | PATCH | `src/modules/customers/customer.routes.ts:19-23` | Update customer | Bearer JWT |
| `/api/v1/customers/:id` | DELETE | `src/modules/customers/customer.routes.ts:24` | Delete customer | Bearer JWT |
| `/api/v1/products` | GET | `src/modules/products/product.routes.ts:16` | Paginated/searchable product list | Bearer JWT |
| `/api/v1/products/:id` | GET | `src/modules/products/product.routes.ts:17` | Get product by UUID | Bearer JWT |
| `/api/v1/products` | POST | `src/modules/products/product.routes.ts:18` | Create product | Bearer JWT |
| `/api/v1/products/:id` | PATCH | `src/modules/products/product.routes.ts:19-23` | Update product | Bearer JWT |
| `/api/v1/products/:id` | DELETE | `src/modules/products/product.routes.ts:24` | Delete product | Bearer JWT |
| `/api/v1/orders` | GET | `src/modules/orders/order.routes.ts:16` | Paginated/filtered order list | Bearer JWT |
| `/api/v1/orders/:id` | GET | `src/modules/orders/order.routes.ts:17` | Get order with items/history/customer | Bearer JWT |
| `/api/v1/orders` | POST | `src/modules/orders/order.routes.ts:18` | Create order in transaction | Bearer JWT |
| `/api/v1/orders/:id/status` | PATCH | `src/modules/orders/order.routes.ts:19-23` | Transition order status | Bearer JWT |
| `/api/v1/orders/:id` | DELETE | `src/modules/orders/order.routes.ts:24` | Delete PENDING/CANCELLED order | Bearer JWT |

Protocol: HTTP/1.1 over plain TCP (no TLS in the application). Payload format: JSON (`express.json`, 1 MB limit). Response format: JSON; errors follow `{ error: { code, message, details? } }`. The listener itself adds no API version negotiation, content negotiation, CORS, rate limiting, or compression.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 via Prisma | External stateful service | Persistence for users, customers, products, orders, order items, status history, order number sequence | MySQL wire protocol over TCP (`DATABASE_URL`, default `localhost:3306`) | Relational rows mapped to Prisma models | Lazy connect; failures surface at query time. `P2002` and `P2025` mapped to 409/404; all other Prisma errors become 500. No startup retry, no readiness probe, no shutdown timeout |
| Node.js `--env-file` loader | Process launcher | Populate `process.env` from `.env` before module evaluation | Filesystem read by Node runtime | `KEY=VALUE` dotenv syntax | Handled by Node before application code; missing file aborts startup. Requires Node >= 20.6 while `engines` declares >= 20 |
| Operating system signals | Runtime integration | Termination requests from operator, container runtime, or process manager | POSIX `SIGINT` / `SIGTERM` | Signal number/name | `shutdown` runs with no re-entrancy guard, no timeout, and detached promise; repeated signals cause concurrent shutdown |
| Process supervisor / container runtime | Deployment integration | Start, restart, and stop the Node process | Process spawn + signals | Exit code (0/1) and stdout/stderr | Exit 0 on signal path, 1 on env/bootstrap failure; async bind failure and unhandled rejections use Node defaults. No Dockerfile, unit file, or orchestrator manifest exists |
| HTTP clients | Inbound network integration | Consume API and health endpoint | HTTP/1.1 over TCP on `PORT`, all interfaces | JSON request/response | Express error middleware maps known errors; unmatched routes 404; unknown errors 500; no request timeout configuration |
| stdout (Pino) | Observability sink | Emit startup, shutdown, access, and fatal logs | File descriptor 1; `pino-pretty` worker in development | NDJSON (production/test) or pretty text (development) | No flush/close on exit; `process.exit` may truncate buffered records. Redaction covers six configured paths |
| MySQL Compose healthcheck | Infrastructure integration | Local database availability signal | `mysqladmin ping` every 5 s, 20 retries | Exit status of CLI probe | Consumed by Docker/operator only; the Node process never queries this healthcheck |
| `SHADOW_DATABASE_URL` | Migration tooling integration | Prisma migrate shadow database | MySQL wire protocol | Connection URL | Not validated by `src/config/env.ts`; failures occur only when running `prisma migrate dev` |

### 8.1 Configuration matrix

| Variable | Required | Validation | Default | Consumed by |
|----------|----------|------------|---------|-------------|
| `NODE_ENV` | No | Enum: `development`, `test`, `production` | `development` | Logger transport/base fields, Prisma log categories, `server_started` payload |
| `PORT` | No | Coerced positive integer | `3000` | `app.listen` (`src/server.ts:9`), startup log |
| `LOG_LEVEL` | No | Enum: `fatal`, `error`, `warn`, `info`, `debug`, `trace` | `info` | Pino logger level |
| `DATABASE_URL` | Yes | Non-empty string only | None | Prisma datasource resolution |
| `JWT_SECRET` | Yes | String, minimum 16 characters | None | JWT signing/verification (auth module) |
| `JWT_EXPIRES_IN` | No | Any string | `8h` | JWT expiry (auth module) |
| `SHADOW_DATABASE_URL` | Only for migrations | Not validated by application | None | Prisma CLI (`migrate dev`) |
| `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD` | No | Compose fallback defaults | `root_password`, `oms`, `oms_user`, `oms_password` | `docker-compose.yml` |

Configuration is supplied exclusively through environment variables loaded by `--env-file`. There is no config file, no profile system, no feature-flag service, and no secret manager integration. The application snapshot is taken once at import and never refreshed.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Composition Root | `buildApp` constructs the entire object graph and Express pipeline | `src/app.ts:26-75`, invoked at `src/server.ts:7` | Centralize wiring and make dependency construction explicit |
| Entry Point / Bootstrap Function | `bootstrap()` with top-level invocation and terminal `catch` | `src/server.ts:6-27` | Separate process lifecycle from HTTP composition |
| Singleton (module scope) | `env`, `prisma`, `logger` exported as module-level constants | `src/config/env.ts:27`, `src/config/database.ts:10`, `src/shared/logger/index.ts:32` | One configuration snapshot, one connection pool, one logger per process |
| Factory Function | `createLogger`, `createPrismaClient`, `buildApp`, `buildControllers`, `buildApiRouter`, `build*Router` | `src/shared/logger/index.ts:13`, `src/config/database.ts:4`, `src/app.ts:26,55`, `src/routes/index.ts:21` | Testable construction and explicit dependency passing |
| Constructor Dependency Injection | Repositories, services, controllers receive dependencies as constructor parameters | `src/app.ts:27-44` | Layered decoupling without a DI container |
| Fail-Fast Configuration | Zod `safeParse` plus `process.exit(1)` at module import | `src/config/env.ts:14-27` | Prevent partially configured processes from starting |
| Graceful Shutdown (partial) | Signal handlers initiate HTTP close and Prisma disconnect | `src/server.ts:13-21` | Release resources and exit deterministically |
| Observer / Event Handler | `process.on('SIGINT'/'SIGTERM')`, `res.on('finish')` | `src/server.ts:20-21`, `src/middlewares/request-logger.middleware.ts:12` | React to lifecycle and request-completion events |
| Middleware Chain | Express `app.use` ordering: parser, logger, routes, 404, error | `src/app.ts:58-73` | Cross-cutting concerns without controller duplication |
| Centralized Error Translation | Single `ErrorRequestHandler` maps domain, validation, and Prisma errors | `src/middlewares/error.middleware.ts:14-65` | Uniform error contract and secret-safe 500 responses |
| Layered Architecture | server -> app -> routes -> middleware -> controller -> service -> repository -> Prisma | Whole `src` tree | Separation of transport, domain, and persistence concerns |
| Structured Logging with Redaction | Pino JSON, base fields, `redact` paths | `src/shared/logger/index.ts:4-30` | Machine-readable observability and secret masking |
| Lazy Initialization | Prisma connects on first query, not at boot | `src/config/database.ts:10` + no `$connect` in `src/server.ts` | Faster startup; defers dependency failure to request time |

### 9.1 Architectural decisions evidenced in the component

| Decision | Evidence | Consequence |
|----------|----------|-------------|
| Native `--env-file` instead of `dotenv` | `package.json:11,13`; no `dotenv` dependency | Depends on Node version >= 20.6 for launch scripts |
| Single Node process, single listener | `src/server.ts:9`; no cluster/worker code | Simple lifecycle, but a single point of failure and an event-loop throughput ceiling |
| Process-wide Prisma singleton | `src/config/database.ts:10`, `src/server.ts:3,7` | One pool for all requests; explicit lifetime managed by shutdown |
| Import-time configuration validation | `src/config/env.ts:27` | Fail-fast startup, but untestable side effects and no logger during env failure |
| Exit code 0 for signal shutdown, 1 for fatal startup | `src/server.ts:17,26`, `src/config/env.ts:22` | Coarse-grained but conventional contract |
| ESM with `.js` specifiers in TS sources | `src/server.ts:1-4`, tsconfig `module: ESNext` | Direct `node dist/server.js` execution without a bundler |
| No HTTP drain before exit | `src/server.ts:15-17` | Fast deploys/restarts, at the cost of dropped in-flight requests |
| No host binding configuration | `src/server.ts:9` | Listens on all interfaces; exposure controlled externally |

### 9.2 Component quality attributes

| Attribute | Assessment | Evidence |
|-----------|------------|----------|
| Simplicity | Very high; 27 lines, 4 internal imports, no branching beyond the catch | `src/server.ts` |
| Cohesion | High; every line serves process lifecycle | `src/server.ts:6-27` |
| Coupling | Low internal fan-out (4 files), zero afferent coupling | Section 6 |
| Testability | Low; side-effectful module, no exports, import-time `process.exit` in dependency | `src/server.ts:24`, `src/config/env.ts:22-27` |
| Observability | Moderate; startup/shutdown/fatal events exist, but env failures and async bind failures bypass them | `src/server.ts:10,14,25` |
| Resilience | Low; no retry, no readiness gating, no drain deadline, no error-event handler | `src/server.ts:9-21` |
| Security posture | Baseline controls only; no TLS, no secret manager, all-interface bind, weak example secrets | `src/app.ts:58-59`, `src/config/env.ts:8`, `.env.example:8` |
| Operability | Low-to-moderate; start/stop works, but shutdown drops traffic and `/health` is liveness-only | `src/server.ts:15-17`, `src/app.ts:62-64` |
| Portability | Requires Node >= 20.6 in practice and an externally provisioned MySQL | `package.json:8,11,13` |

---

## 10. Technical Debt & Risks

### 10.1 Risk register

| ID | Risk Level | Component Area | Issue | Impact | Evidence |
|----|------------|----------------|-------|--------|----------|
| TD-01 | High | Shutdown | `server.close()` is not awaited and its callback is not promisified before `process.exit(0)` | In-flight requests are terminated; the `http_server_closed` log is effectively unreachable; traffic is dropped on every deploy/restart | `src/server.ts:15-17` |
| TD-02 | High | Listener error handling | No `server.on('error')` handler; asynchronous bind failures are not covered by `bootstrap().catch` | `EADDRINUSE`/`EACCES` crash via uncaught exception, bypassing the fatal log and Prisma cleanup; exit diagnostics are inconsistent | `src/server.ts:9`, `src/server.ts:24-27` |
| TD-03 | High | Test coverage | No test imports `src/server.ts`; lifecycle behavior has zero direct coverage | Regressions in signal handling, exit codes, and startup ordering cannot be detected by the suite | Section 11; `tests/helpers/factories.ts:12` |
| TD-04 | High | Shutdown ordering | Prisma is disconnected before HTTP connections are guaranteed to be finished | Requests executing during the disconnect window can fail with a disconnected client; partial writes are possible in transactional endpoints | `src/server.ts:15-16` |
| TD-05 | Medium | Shutdown | No re-entrancy guard; repeated signals run concurrent `shutdown()` calls | Duplicate logs, duplicate `$disconnect` (likely rejection), duplicate `process.exit`; nondeterministic exit behavior | `src/server.ts:13-21` |
| TD-06 | Medium | Shutdown | `void shutdown(...)` discards promise rejections; `$disconnect` failure becomes an unhandled rejection | Shutdown failures are unlogged; exit code may diverge from the documented 0 contract | `src/server.ts:16,20-21` |
| TD-07 | Medium | Shutdown | No shutdown deadline or forced-close fallback | A hung `$disconnect` prevents termination until the supervisor's grace period expires and kills the process | `src/server.ts:13-18` |
| TD-08 | Medium | Configuration | `process.exit(1)` inside an import-time module (`env.ts`) bypasses the logger, cleanup, and test control; any importer inherits the exit | Environment validation cannot be unit-tested in-process; failures are unstructured stderr; tests can kill the runner | `src/config/env.ts:14-27`, `tests/setup.ts:2` |
| TD-09 | Medium | Configuration | `DATABASE_URL` validated for non-empty string only; `JWT_EXPIRES_IN` accepts any string; `PORT` not range-checked against 65535 | Malformed URLs and invalid expiry values start successfully and fail later; out-of-range ports surface as a fatal bootstrap error instead of a configuration message | `src/config/env.ts:5,7,9` |
| TD-10 | Medium | Startup readiness | Listener starts without verifying database connectivity or applying migrations | Process and `/health` report success while every stateful route fails; orchestrators cannot gate traffic correctly | `src/server.ts:7-11`, `src/app.ts:62-64` |
| TD-11 | Medium | Process error handling | No `uncaughtException` or `unhandledRejection` handlers | Runtime crashes after startup bypass all structured fatal logging and resource cleanup | `src/server.ts` (absent) |
| TD-12 | Medium | HTTP server hardening | No explicit `headersTimeout`, `requestTimeout`, `keepAliveTimeout`, or `maxRequestsPerSocket`; no drain of keep-alive connections on shutdown | Exposure to slow-client resource retention; shutdown leaves idle sockets until process exit | `src/server.ts:9` |
| TD-13 | Medium | Secrets | `JWT_SECRET` minimum is 16 characters and the shipped example uses a predictable placeholder; no secret manager or rotation mechanism | Weak or reused secrets pass validation; compromise forges all sessions | `src/config/env.ts:8`, `.env.example:8` |
| TD-14 | Medium | Log redaction | Redaction covers six fixed paths; fatal/error logs serialize the full `err` object and request context | Prisma error metadata, payload fragments, or unexpected fields can reach logs unredacted | `src/shared/logger/index.ts:4-11`, `src/middlewares/error.middleware.ts:57-60`, `src/server.ts:25` |
| TD-15 | Medium | Runtime version | Launch scripts use `--env-file`, which requires Node >= 20.6, while `engines` allows any Node >= 20 | Start commands fail on Node 20.0-20.5 before application code executes | `package.json:8,11,13` |
| TD-16 | Low | Network binding | No host configuration; listener binds all interfaces by default | Service is reachable from any network interface unless restricted externally; no TLS at the application layer | `src/server.ts:9` |
| TD-17 | Low | Log delivery | `process.exit` can truncate buffered Pino output, including the fatal startup record | Missing diagnostics precisely during failures, where they matter most | `src/server.ts:17,26`, `src/shared/logger/index.ts:22-28` |
| TD-18 | Low | Startup observability | `server_started` omits resolved address, version, pid, database target, and readiness state | Harder to correlate logs across replicas and deployments | `src/server.ts:10` |
| TD-19 | Low | Signal coverage | Only `SIGINT` and `SIGTERM` handled; no `SIGHUP`/`SIGQUIT` semantics | Terminal/supervisor behavior depends on external tooling | `src/server.ts:20-21` |
| TD-20 | Low | Topology | Single process, single listener, no cluster/worker; repository has no Dockerfile, process manager, or CI | Any process crash or deploy removes API availability; deployment procedure is implicit | `src/server.ts:9`, repository root (no deployment artifacts) |
| TD-21 | Low | Dead dependency | `pino-http` is declared but unused; request logging is hand-written | Extra supply-chain surface; misleading dependency intent at bootstrap/logging setup | `package.json:31`, `src/middlewares/request-logger.middleware.ts` |

### 10.2 Security assessment

Controls present at or activated by the component:

| Control | Implementation | Location |
|---------|----------------|----------|
| Framework fingerprint reduction | `app.disable('x-powered-by')` | `src/app.ts:58` |
| Request body size limit | `express.json({ limit: '1mb' })` | `src/app.ts:59` |
| Secret redaction in logs | `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken` | `src/shared/logger/index.ts:4-11` |
| Error masking for unknown failures | Generic `INTERNAL_SERVER_ERROR` with server-side logging | `src/middlewares/error.middleware.ts:56-64` |
| Request correlation | `X-Request-Id` echo and log field | `src/middlewares/request-logger.middleware.ts:6-8` |
| Configuration failure closed | Exit 1 on invalid environment | `src/config/env.ts:14-23` |

Security-relevant risks:

| Risk | Detail | Evidence |
|------|--------|----------|
| All-interface binding | `app.listen(port)` with no host; reachable on any interface | `src/server.ts:9` |
| No transport security | Plain HTTP listener; no TLS options, no HTTPS bootstrap path, no trust-proxy configuration | `src/server.ts:9`, `src/app.ts:55-75` |
| Environment-only secrets | No vault, encrypted config, or file permissions enforcement; `.env` path is fixed in scripts | `package.json:11,13`, `.env.example` |
| Weak secret policy | 16-character minimum; placeholder secret passes validation | `src/config/env.ts:8`, `.env.example:8` |
| Error-object logging | Full `err` serialized at fatal and error levels; redaction is path-based only | `src/server.ts:25`, `src/middlewares/error.middleware.ts:57-60` |
| No global crash containment | Uncaught exceptions terminate the process; no last-resort logging | `src/server.ts` |
| Shutdown injection surface | Any process able to send a signal can trigger exit 0 and drop in-flight work; no authorization boundary at process level | `src/server.ts:20-21` |

No authentication, authorization, rate limiting, CORS, or CSRF logic exists in the component itself; those are located in middleware and route modules and are out of this component's boundary.

### 10.3 Error handling and resilience summary

| Failure class | Containment | Result |
|---------------|-------------|--------|
| Invalid environment | Fail-fast at import | Exit 1 with aggregated stderr |
| Bootstrap wiring/listen synchronous error | `bootstrap().catch` | Fatal log + exit 1 |
| Async bind error (`EADDRINUSE`) | None | Uncaught exception, exit 1 without fatal log |
| Unhandled rejection in shutdown | None (`void`) | Unhandled rejection; exit behavior depends on Node defaults |
| Runtime uncaught exception | None | Process termination, no structured fatal event |
| Per-request error | Express error middleware | 4xx/5xx JSON; 500 for unknown |
| Database unavailable at query time | Prisma rejection -> error middleware | 500 (or mapped 404/409 for specific codes); no retry/circuit breaker |
| Shutdown hang | None | Process remains until supervisor force-kills |

The resilience model is fail-fast at startup and fail-open-to-500 at request time; there is no retry, no circuit breaker, no bulkhead, and no graceful degradation path at the bootstrap layer.

---

## 11. Test Coverage Analysis

### 11.1 Test inventory

| File | Type | Tests | What it exercises |
|------|------|-------|-------------------|
| `tests/auth.test.ts` | Integration (HTTP through Express in-process) | 7 | Register, duplicate email, validation failure, login success/failure, authenticated `/me`, unauthenticated `/me` |
| `tests/orders.test.ts` | Integration (HTTP + real MySQL) | 8 | Order creation and totals, missing product, PENDING->PAID stock debit, invalid transition, insufficient stock, PAID->CANCELLED replenish, list filtering, delete guard |
| `tests/setup.ts` | Test lifecycle | n/a | `$connect` before all, truncate 7 tables before each, `$disconnect` after all |
| `tests/helpers/factories.ts` | Test harness | n/a | Caches `buildApp({ prisma })`; user/customer/product factories; auth bootstrap |

Total: 15 test cases. Configuration: `vitest.config.ts` includes `tests/**/*.test.ts`, Node environment, 30 s test/hook timeouts, `tests/setup.ts` as setup file, single fork (`fileParallelism: false`, `singleFork: true`), globals disabled.

### 11.2 Coverage by component behavior

| Component / Behavior | Unit Tests | Integration Tests | Coverage | Test Quality |
|----------------------|------------|-------------------|----------|--------------|
| `src/server.ts` module load and `bootstrap()` | 0 | 0 | 0% direct | Untested; never imported by any test |
| HTTP listener creation / port binding | 0 | 0 | 0% direct | Untested; tests bypass the listener with Supertest's in-process app |
| `SIGINT` / `SIGTERM` shutdown sequence | 0 | 0 | 0% | Untested; no signal simulation, no exit-code assertions |
| `prisma.$disconnect()` on shutdown | 0 | 1 indirect (`tests/setup.ts:18-20`) | Indirect only | Test teardown exercises the provider, not the server shutdown path |
| `bootstrap_failed` fatal branch and exit 1 | 0 | 0 | 0% | Untested |
| Environment validation (`env.ts`) | 0 | 0 | 0% for invalid cases; valid env exercised by all 15 tests | No test sets invalid env; import-time `process.exit` makes in-process assertion difficult |
| Port default and coercion | 0 | 0 | 0% | Untested |
| Prisma client provider | 0 | 15 indirect + setup | Indirect | Real MySQL used; no assertion on log-level behavior |
| Logger singleton / redaction | 0 | 15 indirect (log emission) | Indirect, unasserted | No test captures or inspects Pino output; redaction unverified |
| App composition and router mounting | 0 | 15 | High for exercised routes | Good behavioral assertions on status/body; 13 of 20 routes untested |
| `GET /health` | 0 | 0 | 0% | Untested |
| Request logger / `X-Request-Id` | 0 | 0 direct | 0% asserted | Middleware runs but header and duration are never asserted |
| 404 fallback for unknown routes | 0 | 0 | 0% | Untested (the observed 404 assertions are domain NOT_FOUND in `tests/orders.test.ts:55-56`) |
| Centralized error middleware branches | 0 | Partial via route tests | Partial | AppError and ZodError exercised; Prisma P2002/P2025 branches and generic 500 untested |
| Startup log event `server_started` | 0 | 0 | 0% | Untested |
| Shutdown log events | 0 | 0 | 0% | Untested |

### 11.3 Coverage measurement status

No coverage provider is installed or configured. `package.json` has no `coverage` script, no `@vitest/coverage-v8`/`@vitest/coverage-istanbul` dependency, and no `coverage/` directory exists. The `vitest.config.ts` has no `coverage` block. Consequently, the percentages above are behavioral presence/absence assessments from static inspection, not instrumented line coverage. For `src/server.ts`, the direct line coverage is 0%: the module is executed only when the server is started, never by the test suite.

### 11.4 Test quality assessment

Strengths:

- The 15 integration tests exercise the real Express pipeline and a real MySQL database, so middleware, routing, validation, services, repositories, and Prisma contracts are tested end-to-end rather than mocked.
- `tests/helpers/factories.ts` provides reusable fixtures and a cached app instance, keeping tests readable and fast within a single fork.
- Multiple negative cases exist for auth (duplicate email, invalid payload, wrong password, missing token) and orders (missing product, invalid transition, insufficient stock, delete guard).
- Tests assert both HTTP status and error `code`, which validates the error contract that the bootstrap activates.
- `fileParallelism: false` and `singleFork: true` prevent cross-test truncation races on the shared database.

Gaps:

- The process boundary is entirely untested: no test starts a real listener, sends a real signal, or asserts an exit code. Server Bootstrap is the only component in `src` with zero direct execution by the suite.
- Startup failures (invalid env, missing secret, bind conflict) have no tests, so the fail-fast contract is unverified.
- The `/health` endpoint has no test, so liveness behavior and its lack of a database check are unasserted.
- The request logger's `X-Request-Id` propagation and `http_request` fields are not asserted, and Pino redaction is not tested against a payload containing `authorization`, `password`, or `token`.
- 13 of the 20 published routes have no integration test (customers and products modules entirely, user lookup, several auth/order variants), so the composition root's full wiring is only partially validated.
- The truncate-every-table `beforeEach` uses the configured `DATABASE_URL` with no environment guard, which is a test-harness risk documented in the architectural report and directly adjacent to bootstrap configuration.

### 11.5 Test file locations

- `tests/auth.test.ts`
- `tests/orders.test.ts`
- `tests/helpers/factories.ts`
- `tests/setup.ts`
- Test configuration: `vitest.config.ts`

No test file exists under `src/`, no `__tests__` directories exist, and no dedicated `server.test.ts`, `bootstrap.test.ts`, or `shutdown.test.ts` exists anywhere in the repository.

---

## Appendix A: Evidence Index

| Claim | Source |
|-------|--------|
| Bootstrap composition, listen, shutdown, fatal catch | `src/server.ts:1-27` |
| App composition, health route, middleware order, 404 and error wiring | `src/app.ts:22-75` |
| Environment schema, defaults, import-time exit | `src/config/env.ts:3-27` |
| Prisma singleton and environment-dependent log level | `src/config/database.ts:4-10` |
| Logger configuration, redaction, dev transport | `src/shared/logger/index.ts:4-32` |
| Request id, duration logging | `src/middlewares/request-logger.middleware.ts:5-28` |
| Error translation and 500 masking | `src/middlewares/error.middleware.ts:7-65` |
| Router mounting and Controllers contract | `src/routes/index.ts:13-31` |
| Module routes and auth/role guards | `src/modules/*/*.routes.ts` |
| Launch/build scripts and engine constraint | `package.json:7-21,25-53` |
| TypeScript/ESM build configuration | `tsconfig.json:1-30`, `tsconfig.build.json:1-11` |
| Environment template values | `.env.example:1-14` |
| Local MySQL topology | `docker-compose.yml:1-28` |
| Test lifecycle and fixtures | `tests/setup.ts:1-20`, `tests/helpers/factories.ts:1-20` |
| Test cases | `tests/auth.test.ts`, `tests/orders.test.ts` |
| Vitest behavior | `vitest.config.ts:1-19` |
| Prior architectural findings corroborated | `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md:132,169-170,358-363,553-556` |
| Dependency/dead dependency context | `docs/agents/dependency-auditor/dependencies-report-2026-11-08_22-50-33.md:245,297,305` |

---

*End of report.*
