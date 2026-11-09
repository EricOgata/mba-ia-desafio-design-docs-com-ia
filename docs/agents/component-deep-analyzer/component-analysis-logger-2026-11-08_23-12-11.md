# Component Deep Analysis Report

**Component:** Logger
**Primary file:** `src/shared/logger/index.ts` (32 lines)
**Project:** `order-management-api` (Order Management System REST API)
**Analysis date:** 2026-11-08 23:12:11
**Scope:** Entire project root, excluding `node_modules`, `.git`, `dist`, `coverage`
**Output:** `docs/agents/component-deep-analyzer/component-analysis-logger-2026-11-08_23-12-11.md`
**Analysis mode:** Static code analysis only. No project files were modified, no code was executed, no tests were run.

---

## Analysis Scope and Method

This report treats the Logger component as the module `src/shared/logger/index.ts` plus the observable logging contract it creates for its three direct consumers: `src/middlewares/request-logger.middleware.ts`, `src/middlewares/error.middleware.ts`, and `src/server.ts`. The call sites are included because the component's practical behavior (record schema, event names, correlation fields, which values reach the redactor) is defined jointly by the Pino configuration and the objects those call sites pass to `logger.info`, `logger.error`, and `logger.fatal`.

Library semantics are taken from the installed-version contract in the lockfile (`pino` 9.5.0, `pino-pretty` 11.3.0, `pino-std-serializers` 7.1.0, `thread-stream` 3.1.0) and from the Pino documentation. Where a behavior is a library contract rather than an explicit statement in project code, the confidence level is stated in the business-rule table. `node_modules` is not present in the workspace, so no library source was read locally; Pino documentation was used for redaction path semantics, transport flush behavior, and binding defaults.

No file matching `*logger*.test.ts` or `*log*.test.ts` exists in `tests/`. The test harness, factories, and the two integration suites were inspected to determine indirect exercise.

---

## 1. Executive Summary

The Logger component is a 32-line shared-infrastructure module that wraps Pino (`pino` 9.5.0) in a factory function and exports a single process-wide logger instance. It has exactly one internal dependency, the validated environment singleton `env` (`src/config/env.ts`), from which it reads `LOG_LEVEL` and `NODE_ENV`. It exposes two values: `createLogger()` (`src/shared/logger/index.ts:13-30`) and the eagerly constructed `logger` (`src/shared/logger/index.ts:32`). The configuration is small and declarative: level, a six-path redaction deny-list with a `[REDACTED]` censor, two base bindings (`service`, `env`), ISO-8601 timestamps, and a `pino-pretty` transport that is activated only when `NODE_ENV === 'development'`.

Its role in the system is cross-cutting observability for the HTTP and process lifecycle layers. Three files import it directly: `src/server.ts:4` (startup, shutdown, fatal bootstrap), `src/middlewares/request-logger.middleware.ts:3` (one access record per completed HTTP response), and `src/middlewares/error.middleware.ts:5` (unhandled errors). It is mounted into every request path through `src/app.ts:3` and `src/app.ts:60`, so the logger is constructed during the evaluation of the application module graph, before the HTTP listener is created. No domain module (`src/modules/**`) imports the logger; application logging is therefore operational, not an audit trail of business events.

Key findings:

- **Very small surface, high architectural reach.** Two exports, one private constant, one conditional expression, no loops, and no branching beyond the `NODE_ENV` transport selection. Complexity is effectively constant, yet every HTTP response produces one log record through it.
- **Configuration is frozen at import time.** `env` is read once (`src/shared/logger/index.ts:15,20,23`). There is no runtime level change, no per-module level, no child loggers, no `logger.flush` call, and no shutdown hook; the only way to change verbosity is to restart the process with a different `LOG_LEVEL`.
- **The component's security value depends entirely on a six-entry deny-list.** `redactPaths` (`src/shared/logger/index.ts:4-11`) censors `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, and `*.accessToken`. Redaction is applied to structured fields only; the `msg` string and the request `path` value are unaffected (see BR-10).
- **Two of the six redaction paths are currently dormant.** `req.headers.authorization` and `req.headers.cookie` target the `req` object shape produced by `pino-http`, but `pino-http` 10.3.0 is declared in `package.json:31` and imported nowhere. The custom request middleware logs only `method`, `path`, `statusCode`, `durationMs`, and `userId` (`src/middlewares/request-logger.middleware.ts:14-24`), so no header object ever reaches the redactor today.
- **Output format is environment-selected, not runtime-selected.** `development` switches to an asynchronous `pino-pretty` worker with colorization and `HH:MM:ss.l` timestamps (`src/shared/logger/index.ts:22-27`); `test` and `production` use Pino's default newline-delimited JSON on stdout. `pino-pretty` is declared only as a devDependency (`package.json:46`), while `NODE_ENV` defaults to `development` when absent (`src/config/env.ts:4`).
- **No dedicated tests exist.** There is no unit test, no mock of `pino`, no stdout capture, and no assertion on any log field, level, redaction path, or event name anywhere in `tests/`. The module is executed indirectly by both integration suites because building the app builds the logger, but its output is never inspected.
- **The logger is fire-and-forget.** In development, delivery is asynchronous through a worker thread, and neither `src/server.ts:26` (fatal bootstrap exit) nor `src/server.ts:17` (shutdown exit) flushes before `process.exit`. This creates a documented class of log loss that the component does not mitigate.
- **The log record contract is implicit.** Pino's defaults supply `msg` and the numeric `level`; the base object supplies `service` and `env`; the ISO timestamp function supplies `time`. There is no declared schema, no serializer customization, and no TypeScript type describing the record fields that call sites must provide.

---

## 2. Data Flow Analysis

### 2.1 Import-time construction flow

```
1. Process starts through one of the package scripts:
   - dev:   tsx watch --env-file=.env src/server.ts          (package.json:11)
   - start: node --env-file=.env dist/server.js               (package.json:13)
   - test:  vitest run                                       (package.json:17)
2. Node's --env-file loader (or the invoking shell / platform) populates
   process.env before any application module is evaluated.
3. ESM evaluation starts at src/server.ts:1 -> src/app.ts.
4. src/app.ts:3 imports ./middlewares/request-logger.middleware.js, whose
   line 3 imports ../shared/logger/index.js. This is the first import of the
   logger module in the graph.
5. src/shared/logger/index.ts:2 imports env from ../../config/env.js.
   src/config/env.ts validates process.env against its Zod schema and either
   terminates the process with exit code 1 or exports a typed snapshot.
6. src/shared/logger/index.ts:4-11 evaluates the redactPaths array literal.
7. src/shared/logger/index.ts:32 evaluates createLogger() as a module-level
   side effect:
     - level: env.LOG_LEVEL                                    (line 15)
     - redact: { paths: redactPaths, censor: '[REDACTED]' }    (lines 16-19)
     - base: { service: 'order-management-api', env: NODE_ENV }(line 20)
     - timestamp: pino.stdTimeFunctions.isoTime                (line 21)
     - transport: pino-pretty descriptor iff NODE_ENV === 'development'
                  (lines 22-27)
8. pino() compiles the redaction matcher and creates the logger instance.
   If transport is set, pino.transport spawns a thread-stream worker for
   pino-pretty during process startup; startup is asynchronous.
9. The instance is assigned to the exported const logger.
10. Later imports (src/server.ts:4, src/middlewares/error.middleware.ts:5)
    receive the same instance from the ESM module cache. No second logger is
    constructed anywhere in the project, and createLogger() has no call sites
    other than the module's own initializer.
```

### 2.2 Per-request logging flow (access record)

```
1. HTTP request enters Express (src/app.ts:56-60):
   express.json({ limit: '1mb' }) -> requestLogger.
2. requestLogger (src/middlewares/request-logger.middleware.ts:5-27):
   2a. requestId = inbound x-request-id (verbatim) ?? uuid v4   (line 6)
   2b. req.id = requestId; response header X-Request-Id = requestId
       (lines 7-8)
   2c. startedAt = process.hrtime.bigint()                       (line 10)
3. Request proceeds through routing, optional authenticate
   (src/middlewares/auth.middleware.ts:27-47), validation, controller,
   service, repository.
4. On the response 'finish' event (line 12):
   4a. durationMs = (hrtime delta) / 1e6, rounded to 2 decimals (line 13)
   4b. logger.info({
         requestId, method, path: req.originalUrl, statusCode,
         durationMs, userId: req.user?.id
       }, 'http_request')                                      (lines 14-24)
5. Pino compares the record level (info = 30) with the configured level
   (src/shared/logger/index.ts:15). Records below the threshold are discarded
   before serialization.
6. The redaction matcher walks the structured object against the six paths
   (src/shared/logger/index.ts:4-11,16-19) and replaces matching values with
   the string '[REDACTED]'.
7. Serialization composes, in order: bindings {service, env}, time (ISO),
   level (numeric), msg, then the merged log-object fields.
8. Destination:
   - development: thread-stream worker -> pino-pretty (colorize: true,
     singleLine: false, translateTime: 'HH:MM:ss.l') -> stdout.
   - test/production: one newline-delimited JSON line -> stdout (fd 1).
9. No callback, promise, acknowledgement, or flush is awaited by the
   middleware. The response is not delayed by logging.
```

### 2.3 Error logging flow

```
1. An error reaches the centralized handler via next(err)
   (mounted last at src/app.ts:73).
2. The handler checks known error families first
   (src/middlewares/error.middleware.ts:15-54):
   - AppError -> JSON response, no log
   - ZodError -> 400 VALIDATION_ERROR, no log
   - Prisma P2002 -> 409 CONFLICT, no log
   - Prisma P2025 -> 404 NOT_FOUND, no log
3. Any other error:
   3a. requestId = req.id ?? 'unknown'                          (line 56)
   3b. logger.error(
         { err, requestId, method: req.method, path: req.originalUrl },
         'Unhandled error in request')                         (lines 57-60)
4. Pino's standard error serializer expands the `err` key into
   type/message/stack (plus cause when present). No custom serializers are
   configured, so Error objects placed under any other key are not expanded.
5. The client receives 500 with { code: 'INTERNAL_SERVER_ERROR',
   message: 'Internal server error' }                          (lines 62-64).
6. Redaction applies to the object graph before serialization, but only for
   the six configured paths; stack text and other error properties are
   emitted verbatim.
```

### 2.4 Startup, shutdown, and fatal flows

```
Startup:
  src/server.ts:9 app.listen(env.PORT, callback)
    -> callback: logger.info({ port, env: NODE_ENV }, 'server_started')
                                                                 (line 10)

Shutdown (SIGINT/SIGTERM):
  src/server.ts:14 logger.info({ signal }, 'shutdown_initiated')
  src/server.ts:15 server.close(() => logger.info('http_server_closed'))
  src/server.ts:16 await prisma.$disconnect()
  src/server.ts:17 process.exit(0)
  Note: the close callback is not awaited and process.exit can run first;
  the http_server_closed record is emitted only if the callback wins the race.

Fatal bootstrap:
  src/server.ts:24-27 bootstrap().catch(err => {
    logger.fatal({ err }, 'bootstrap_failed');
    process.exit(1);
  })
  Note: in development the fatal record is handed to an asynchronous worker
  immediately before process.exit(1); Pino documents that logs can be lost in
  exactly this pattern.
```

### 2.5 Test-time flow

```
1. vitest run (package.json:17) starts the runner using vitest.config.ts:
   include tests/**/*.test.ts, setupFiles tests/setup.ts, single fork,
   fileParallelism false.
2. tests/setup.ts:2 imports prisma -> src/config/env.ts is evaluated, so the
   environment gate runs before any test body.
3. tests/helpers/factories.ts:4 imports buildApp from src/app.js; app.ts:3
   imports the request logger, which imports and constructs the logger once.
   getTestApp() caches the Express app (factories.ts:8-15), so one logger
   instance serves every suite in the single fork.
4. Vitest sets NODE_ENV=test by default when the variable is not already set
   (environment assumption; not asserted by any project file). Under that
   value the transport branch in src/shared/logger/index.ts:23 is false, so
   the logger writes newline-delimited JSON to stdout instead of launching
   pino-pretty.
5. Every HTTP request issued by tests/auth.test.ts (7 cases) and
   tests/orders.test.ts (8 cases) triggers one `http_request` record at info
   level when the response finishes. No test captures, asserts, or silences
   that output.
6. Error-path logging is not exercised: neither suite asserts a 500 response,
   and the AppError/Zod/Prisma branches return without logging.
```

### 2.6 Record composition and filtering pipeline

```
log call (object, msg)
   |
   +--> level check (configured env.LOG_LEVEL, line 15)
   |        |-- below threshold -> dropped, no serialization
   |        `-- at/above threshold -> continue
   |
   +--> redaction matcher (six compiled paths, lines 4-11)
   |        `-- matching values -> '[REDACTED]' (keys retained)
   |
   +--> serialization
   |        +-- bindings: service, env (line 20)
   |        +-- time: ISO-8601 string (line 21)
   |        +-- level: numeric (Pino default)
   |        +-- msg: default message key (Pino default)
   |        +-- err: standard error serializer when key is `err`
   |        `-- all other fields: shallow JSON merge
   |
   +--> destination
            |-- development: worker thread -> pino-pretty -> stdout
            `-- test/production: NDJSON line -> stdout (fd 1)
```

---

## 3. Business Rules & Logic

## Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location | Confidence |
|---------|-----------|------------------|----------|------------|
| BR-01 | Configuration / Filtering | The effective log threshold is `env.LOG_LEVEL`; only the six Pino levels `fatal`, `error`, `warn`, `info`, `debug`, `trace` are permitted and the default is `info`; records below the threshold are discarded before serialization | `src/shared/logger/index.ts:15`, `src/config/env.ts:6` | High |
| BR-02 | Security / Data Masking | A six-path deny-list is censored with the literal string `[REDACTED]`: `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken` | `src/shared/logger/index.ts:4-11,16-19` | High |
| BR-03 | Data Shaping | Redaction replaces values but preserves keys; there is no `remove` mode, hashing, truncation, or partial masking | `src/shared/logger/index.ts:16-19` | High |
| BR-04 | Data Shaping | Every record carries the base bindings `service: 'order-management-api'` and `env: <NODE_ENV>`; supplying `base` replaces Pino's default `pid`/`hostname` bindings | `src/shared/logger/index.ts:20`, `package.json:2` | High (bindings), Medium (replacement semantics) |
| BR-05 | Data Shaping | Timestamps use `pino.stdTimeFunctions.isoTime`, producing UTC ISO-8601 strings with millisecond precision under the `time` key | `src/shared/logger/index.ts:21` | High |
| BR-06 | Environment-driven Behavior | `NODE_ENV === 'development'` selects a `pino-pretty` transport (`colorize: true`, `singleLine: false`, `translateTime: 'HH:MM:ss.l'`); every other environment writes raw newline-delimited JSON | `src/shared/logger/index.ts:22-28` | High |
| BR-07 | Lifecycle | Exactly one logger instance exists per process, created at module evaluation; every consumer shares it; no close, flush, or dispose operation is exposed by the component | `src/shared/logger/index.ts:13-32` | High |
| BR-08 | Output Contract | The sink is stdout (file descriptor 1) in both formats; there is no file output, rotation, remote shipping, multi-stream routing, or separation of error records onto stderr | `src/shared/logger/index.ts:14-29` (absence of options) | High |
| BR-09 | Error Serialization | Only values passed under the `err` key receive Pino's standard error serializer (type, message, stack); no custom serializers are configured | `src/middlewares/error.middleware.ts:57-60`, `src/server.ts:25` | High |
| BR-10 | Security Boundary | Redaction applies to structured fields on the configured paths only; `msg` strings, URL/path values, request bodies, and nesting deeper than the configured wildcard depth are outside its scope | `src/shared/logger/index.ts:4-11`; `src/middlewares/request-logger.middleware.ts:18` | High (scope), Medium (wildcard depth) |
| BR-11 | Lifecycle / Configuration | Configuration is captured once at import; there is no runtime reconfiguration, no per-module level, no child logger, and no signal-driven reload | `src/shared/logger/index.ts:2,15,20,23` | High |
| BR-12 | Deployment Dependency | The development format requires the `pino-pretty` module to be resolvable by name in the runtime install; it is declared only as a devDependency and not checked at startup | `src/shared/logger/index.ts:22-27`, `package.json:46` | High (declaration), Medium (failure mode) |
| BR-13 | Record Conventions | Call sites follow an implicit schema: snake_case lifecycle event names, prose messages for request/error events, structured fields first, and level selection by severity (info for lifecycle/access, error for unhandled, fatal for bootstrap); `warn` and `debug` are never used | `src/server.ts:10,14,15,25`; `src/middlewares/request-logger.middleware.ts:14-24`; `src/middlewares/error.middleware.ts:57-60` | High |
| BR-14 | Architectural Boundary (implicit) | The logger is used only by infrastructure and process-lifecycle code; no controller, service, repository, or domain module imports it, so business events are not represented in the log stream | `src/modules/**` (no imports); only 3 direct consumers | High |
| BR-15 | Trust Boundary | An inbound `x-request-id` header is accepted verbatim, stored on the request, echoed in the response, and written into access and error records; a UUID v4 is generated only when the header is absent | `src/middlewares/request-logger.middleware.ts:6-8,16`; `src/middlewares/error.middleware.ts:56` | High |

## Detailed breakdown of the business rules

---

### Business Rule: BR-01 - Log level is configured once from the validated environment and filters records

**Overview**:
The logger's threshold is the `LOG_LEVEL` value of the environment snapshot (`src/shared/logger/index.ts:15`). The environment schema restricts it to the six standard Pino levels with `info` as the default (`src/config/env.ts:6`). Pino drops any record whose numeric level is lower than the configured threshold.

**Detailed description**:
The rule has two enforcement points. At startup, Zod rejects any `LOG_LEVEL` value outside `fatal`, `error`, `warn`, `info`, `debug`, `trace`; this excludes Pino's `silent` level, numeric levels, uppercase variants, and comma-separated multi-level strings. At runtime, Pino compares each call's level (fatal 60, error 50, warn 40, info 30, debug 20, trace 10) with the configured threshold and serializes only records at or above it. Because the threshold is read from the environment snapshot once, a deployed process cannot be made more or less verbose without a restart. The component never reads `LOG_LEVEL` again and never calls `logger.level = ...`.

The practical effect is that operational visibility is a deployment-time decision. With the default `info`, the three info-level access/lifecycle call sites and the error/fatal call sites are visible. Setting `warn` suppresses the startup record, every access record, and both shutdown records, leaving only unhandled errors and fatal bootstrap failures. Setting `debug` or `trace` adds no records, because the codebase contains no `logger.debug` or `logger.trace` calls; only library-level output at those levels could appear, and Pino itself emits none. Setting `error` or `fatal` similarly suppresses information that operators normally use to confirm that the server started and that requests completed.

The `silent` exclusion is a deliberate consequence of the environment enum rather than the logger module. There is no alternative switch in the component, so fully disabling output is not representable through configuration. There is also no per-module threshold: the single threshold applies equally to access logs, error logs, and process lifecycle logs. This uniformity is simple but means that increasing verbosity for troubleshooting simultaneously increases access-log volume for every request.

**Rule workflow**:
```
process.env.LOG_LEVEL
  |-- absent -> default "info" (env.ts:6) -> pino level = info
  |-- one of fatal|error|warn|info|debug|trace -> pino level = value
  |-- "silent" | "INFO" | "30" | "" -> env validation failure -> process exit 1
                                        (no logger is constructed)
at each logger.<level>(...) call:
  record level < configured threshold -> discarded before redaction/serialization
  record level >= configured threshold -> continue to redaction/serialization
```

---

### Business Rule: BR-02 - Sensitive-field redaction is a fixed six-path deny-list

**Overview**:
The component declares exactly six redaction paths (`src/shared/logger/index.ts:4-11`): `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, and `*.accessToken`. Each path is compiled into the Pino redaction matcher at construction time and applied to every structured log object thereafter.

**Detailed description**:
The list is a deny-list: any field not matching one of the six paths is serialized normally. The first two entries are expressed against a `req` object with a `headers` map, the shape produced by `pino-http`'s request serializer. The remaining four use the `*.` prefix, which applies the named field to every key at one level below the record root. The six entries cover the credential-bearing names used by this codebase's authentication flow: `authorization` and `cookie` headers, the plaintext `password` input, the stored `passwordHash` (`src/modules/users/user.service.ts:26-29`), and the token names used by the login response (`accessToken`) and by generic token payloads (`token`). `*.token` and `*.accessToken` are distinct patterns; the former does not match `accessToken` because Pino paths are exact, case-sensitive key matches rather than substring matches.

The redaction list is static and has no derivation from a shared constant, schema, or type. Adding a new sensitive field elsewhere in the application creates no compile-time or runtime obligation to extend this list; the protection is only as complete as the list plus the discipline of call sites. At present, the custom request middleware logs no headers and no bodies, so the four wildcard entries are the only ones with a plausible future match; the two `req.headers.*` entries have no reachable trigger while `pino-http` remains unused (`package.json:31` vs no imports).

Pino applies redaction per log object, including objects passed as the first argument to `logger.info`, `logger.error`, and `logger.fatal`. It does not apply redaction to the message string that follows the object, nor to values embedded in a string field such as a URL. The matcher is compiled once at startup, so path syntax errors would surface at construction; the six literals are valid Pino path expressions, and all six are evaluated without error in any successful process start.

**Rule workflow**:
```
logger.<level>(structuredObject, msg)
  -> for each own property path in structuredObject:
       exact match against one of:
         req.headers.authorization
         req.headers.cookie
         <oneLevelKey>.password
         <oneLevelKey>.passwordHash
         <oneLevelKey>.token
         <oneLevelKey>.accessToken
     |-- match -> value replaced with "[REDACTED]" (key remains)
     `-- no match -> value serialized as provided
  -> msg string is appended without path matching
```

---

### Business Rule: BR-03 - Redaction replaces values with a fixed literal and preserves keys

**Overview**:
The `redact` option uses `censor: '[REDACTED]'` (`src/shared/logger/index.ts:18`), and no `remove` flag is set. Matching values become the string `[REDACTED]`; the property names remain in the record.

**Detailed description**:
The rule changes the value's type as well as its content: a numeric or object password becomes a string, so consumers of the log stream cannot infer the original type. The censor is a constant, not a function, so the redaction record contains no information about the original value's length, prefix, suffix, hash, or field path. This is a strong confidentiality posture but a weak forensic one: an operator cannot distinguish an empty password from a long one, cannot correlate a redacted token with a token identifier, and cannot verify that the value was non-empty.

Because the keys are retained, the presence and location of a sensitive field are visible in the log. A record `{ user: { password: '[REDACTED]' } }` confirms that a password field was part of the logged object. In this codebase that is desirable for detecting accidental logging of credentials, but it also means that the record shape is not fully sanitized: only the values are. Setting `remove: true` would have deleted both key and value; the component does not do so.

The same censor applies to every path; there is no per-path policy. A field such as `accessToken` is treated identically to a header. There is also no structural validation that the censored value was actually a credential; the matcher is purely name-based, so a benign field named `token` (for example a pagination token) is censored and its diagnostic value is lost. That trade-off is implicit in a name-based deny-list and is not encoded anywhere in the component.

**Rule workflow**:
```
matching path found
  -> original value discarded
  -> value := "[REDACTED]" (string literal, serialized as JSON string)
  -> property name preserved in the record
  -> no length/prefix/hash information emitted
  -> no per-path variation, no remove mode
```

---

### Business Rule: BR-04 - Every record carries service and environment base bindings

**Overview**:
`base: { service: 'order-management-api', env: env.NODE_ENV }` (`src/shared/logger/index.ts:20`) adds two fields to every record, identifying the producing service and the runtime environment. The service name matches `package.json:2` (`"name": "order-management-api"`).

**Detailed description**:
The base bindings are the only automatic provenance in the log stream. They allow a collector to separate records from different services and from development, test, and production instances even when the record itself carries no other context. `env` is the literal value of `NODE_ENV`, which is constrained by `src/config/env.ts:4` to `development`, `test`, or `production`; it is also the same value that selects the transport in BR-06, so the record's `env` field always agrees with the format used to emit it.

Supplying a `base` object replaces Pino's default bindings, which are `{ pid, hostname }`. Consequently, records do not contain the process ID or the host name unless a call site adds them, and no call site does. In a multi-replica deployment, records from different containers are therefore distinguishable only by the platform's own metadata (container name, pod name, log stream), not by a field in the record. The component also does not include an application version, commit, or instance identifier.

The `server_started` call site passes `{ port, env: env.NODE_ENV }` (`src/server.ts:10`). The `env` field duplicates the base binding with the same value, so the resulting record contains one `env` key (the log-object value is applied to the same binding name) rather than two. The `port` field is the only startup-specific field. No rule prevents a call site from overriding `service` or `env` with a different value; the binding is not protected by a type or formatter.

**Rule workflow**:
```
every log record:
  -> merge base bindings { service: "order-management-api", env: NODE_ENV }
  -> merge call-site object fields (same-named fields take the call-site value)
  -> serialize
  -> record contains service and env on every line, with no pid/hostname by default
```

---

### Business Rule: BR-05 - Timestamps are UTC ISO-8601 strings with millisecond precision

**Overview**:
`timestamp: pino.stdTimeFunctions.isoTime` (`src/shared/logger/index.ts:21`) writes an ISO-8601 UTC timestamp into the default `time` field of every record.

**Detailed description**:
The chosen function produces a string such as `2026-11-08T23:12:11.123Z`, with three fractional digits and a `Z` suffix indicating UTC. Unlike Pino's default epoch-millisecond integer, this is human-readable and lexicographically sortable within a single time zone, which simplifies ad-hoc inspection of raw JSON. It also increases each record's size relative to an integer timestamp and loses no precision relative to the default (both are millisecond resolution).

The timestamp is generated by the application host's clock at serialization time. It is not a monotonic counter, not correlated with `process.hrtime`, and not synchronized by the component. Clock skew between replicas or between a host and its log collector can reorder records, and because the base bindings omit host identity (BR-04), cross-host ordering cannot be reconstructed from the record alone. The access record's `durationMs` is computed separately from `process.hrtime.bigint()` (`src/middlewares/request-logger.middleware.ts:10,13`), so durations are monotonic even if wall-clock timestamps jump.

The `time` key is Pino's default and is not overridden by a `timestampKey` or formatter. There is no local-time mode, no timezone configuration, and no alternative format. `pino-pretty` reformats the timestamp for display using `translateTime: 'HH:MM:ss.l'` (BR-06), which is a presentation concern only; the underlying numeric/string field is consumed by the transport before formatting, so the pretty output shows local wall-clock time with milliseconds rather than the raw ISO string.

**Rule workflow**:
```
record serialization
  -> pino.stdTimeFunctions.isoTime(new Date())
  -> "YYYY-MM-DDTHH:MM:SS.sssZ" (UTC, millisecond precision)
  -> written under key "time"
  -> development transport may reformat for display as HH:MM:ss.l
```

---

### Business Rule: BR-06 - Development uses a pretty transport; all other environments use raw JSON

**Overview**:
The transport is selected by a strict equality check on `env.NODE_ENV` (`src/shared/logger/index.ts:22-28`). Only `development` activates `pino-pretty` with `colorize: true`, `singleLine: false`, and `translateTime: 'HH:MM:ss.l'`. `test` and `production` fall through to `transport: undefined`, which means Pino's default destination and JSON serialization.

**Detailed description**:
The rule creates two materially different output formats from one API. In development, each log call crosses a thread boundary to a `thread-stream` worker that runs `pino-pretty`; the worker colorizes the level, renders nested objects across multiple lines (`singleLine: false`), and converts the ISO timestamp to local `HH:MM:ss.l`. This is optimized for a human reading a terminal. In test and production, records are single-line newline-delimited JSON written to stdout, optimized for machine parsing and log aggregation. The `env` field inside each record identifies which format was used, so a collector can distinguish the two.

The selection is one-way and import-time. There is no runtime switch, no environment variable to force JSON in development, and no way to force pretty output in production through configuration. `NODE_ENV` is validated by the environment component before the logger exists, so no fourth branch is reachable. Because the transport is created during module evaluation, the worker starts before the HTTP listener; Pino documents that this startup is asynchronous, so the earliest log lines can be delivered after the process has already begun serving traffic.

The pretty transport changes delivery guarantees as well as appearance. Without a transport, Pino writes serialized lines through its default destination; with a transport, records are handed to a worker and formatted asynchronously. Pino documents that `process.exit()` before the transport is ready, and `logger.flush()` with `pino-pretty`, do not guarantee delivery of buffered formatted output. The component does not call `flush`, does not listen for transport errors, and does not wait for the worker during shutdown, so development-mode record loss is a property of this rule rather than an incidental bug in a call site.

**Rule workflow**:
```
env.NODE_ENV
  |-- "development"
  |     -> transport = { target: "pino-pretty",
  |                      options: { colorize: true, singleLine: false,
  |                                 translateTime: "HH:MM:ss.l" } }
  |     -> worker thread formats each record, writes to stdout
  `-- "test" | "production"
        -> transport = undefined
        -> pino default destination: NDJSON line per record on stdout
```

---

### Business Rule: BR-07 - One logger instance per process, created at import and never disposed

**Overview**:
`export const logger: Logger = createLogger()` (`src/shared/logger/index.ts:32`) constructs the instance as a module-evaluation side effect. Every importer receives the same object from the ESM cache. The component exposes no `close`, `flush`, or reset operation, and no consumer calls one.

**Detailed description**:
The singleton is established before any consumer executes its own module body: the logger module is evaluated while `src/app.ts` is loading, which is before `src/server.ts` reaches `bootstrap()`. All three direct consumers and all transitive consumers therefore share one Pino instance, one redaction matcher, one configured level, and one destination or worker. There is no per-request, per-module, or per-test logger. In the test harness this is reinforced by `fileParallelism: false` and `singleFork: true` (`vitest.config.ts:11-17`), so both suites reuse one instance and one destination.

The factory `createLogger()` is exported separately, which technically allows a caller to build additional instances with the same configuration. No file does so: a project-wide search finds `createLogger` only at its definition and at the module's own initializer. The export is consequently an unused seam, and the effective lifecycle rule is "exactly one instance, created eagerly". Because creation is eager, an invalid transport target or an invalid level would fail during import rather than on first use; the level is pre-validated by the environment component, and the transport target is not validated at all (BR-12).

The logger has no shutdown protocol. `src/server.ts:13-18` logs `shutdown_initiated`, initiates `server.close`, disconnects Prisma, and exits, but never flushes or ends the logger. Pino exposes `logger.flush()` and the transport stream exposes `end()`, and neither is used. In JSON mode this is less significant because writes are handled by the destination stream; in development mode it is the mechanism by which the `bootstrap_failed` and `http_server_closed` records can be lost. The instance remains referenced by the module graph until process termination, so there is no garbage-collection or double-close concern.

**Rule workflow**:
```
ESM evaluation of src/shared/logger/index.ts
  -> createLogger() executes once
  -> pino instance stored in exported binding `logger`
  -> all importers resolve to the same binding (ESM cache)
  -> no second construction in project code
process lifetime:
  -> level, redaction, bindings, destination fixed at construction
  -> shutdown path performs no flush/end/close on the logger
  -> instance dies with the process
```

---

### Business Rule: BR-08 - Stdout is the only sink and records are newline-delimited

**Overview**:
The component configures no destination, no file, no stream, and no multi-target transport. In non-development environments Pino's default destination is stdout (file descriptor 1), and each record is written as one line of JSON terminated by a newline. Fatal and error records use the same sink and framing as info records.

**Detailed description**:
All severity levels share stdout. There is no convention that errors go to stderr, and no second stream for auditing or alerting. A container platform that captures stdout therefore receives the complete record stream; a platform that captures stdout and stderr separately sees nothing on stderr from the logger. The record's numeric `level` field, not the file descriptor, is the only severity discriminator, so downstream filtering must parse JSON or use a collector that understands Pino levels.

There is no persistence in the process: no log file, no rotation, no retention policy, no compression, no remote HTTP/syslog shipping, and no buffering configuration beyond the library default. Durability and retention are delegated to the execution environment (Docker, systemd, a Kubernetes log driver, or a sidecar). The component also does not attach a service version, build identifier, trace ID, or span ID, so integration with a distributed tracing system is not provided.

`singleLine: false` in the development transport is the only exception to one-record-per-line; the pretty renderer may print a record across multiple terminal lines. That formatting exists only in development and only in the worker's output, not in the JSON stream. Because the default destination is stdout and no `sync` option is set, write behavior follows Pino's destination defaults rather than an explicit project choice.

**Rule workflow**:
```
record serialized
  |-- development -> worker -> human-readable text (possibly multi-line) -> stdout
  `-- test/production -> JSON.stringify(record) + "\n" -> stdout (fd 1)
no file sink, no rotation, no remote transport, no stderr routing, no level-based stream split
```

---

### Business Rule: BR-09 - Errors are serialized only through the `err` key

**Overview**:
The component configures no `serializers` option. Pino's built-in behavior expands an `Error` passed under the key `err` into a structured object; all other error-bearing fields are serialized by the default JSON conversion. Both logging call sites that emit errors use the `err` key (`src/middlewares/error.middleware.ts:58`, `src/server.ts:25`).

**Detailed description**:
At the centralized error handler, any error that is not an `AppError`, `ZodError`, or handled Prisma code reaches `logger.error({ err, requestId, method, path }, 'Unhandled error in request')`. The standard serializer emits the error's type, message, and stack, and includes a `cause` chain when present. This gives operators the stack trace and the request correlation fields in one record, while the HTTP client receives only the generic `INTERNAL_SERVER_ERROR` message (`src/middlewares/error.middleware.ts:62-64`). At bootstrap, `logger.fatal({ err }, 'bootstrap_failed')` uses the same key and therefore the same serializer (`src/server.ts:24-27`).

Because no custom serializers are declared, no request or response serializer exists. `pino-http` would normally supply `req`/`res` serializers and the `req.headers.*` redaction paths would then have a matching shape; without it, those paths are inert. An `Error` placed under any key other than `err` is serialized as a plain object, which for a JavaScript `Error` typically yields `{}` because `message` and `stack` are non-enumerable. The component does not normalize error shapes, redact stack traces, truncate them, or attach error codes explicitly; any code or metadata on a Prisma error that is enumerable is emitted.

Error records are not sampled, deduplicated, or rate-limited. A repeating failure produces one record per occurrence, each carrying the full stack. The only suppression is the level threshold: if `LOG_LEVEL` is set to `fatal`, `error`-level records (including all unhandled errors) are discarded while fatal bootstrap records remain.

**Rule workflow**:
```
logger.error({ err: someError, ...fields }, msg)
  -> Pino detects key "err"
  -> standard error serializer: { type, message, stack, ...cause? }
  -> other fields merged normally
  -> redaction paths applied to the serialized graph
  -> record emitted at level 50 (suppressed if threshold > error)
logger.fatal({ err: someError }, "bootstrap_failed")
  -> same serialization, level 60
any Error under another key -> default JSON conversion (typically {})
```

---

### Business Rule: BR-10 - Redaction covers structured fields on configured paths only

**Overview**:
Redaction is a field-path filter over the structured object passed to the logger. It does not sanitize the message string, URL values, object keys, or nested structures that do not match a declared path. This rule defines the boundary of the component's privacy guarantee.

**Detailed description**:
The access record includes `path: req.originalUrl` (`src/middlewares/request-logger.middleware.ts:18`), which contains the full request target including the query string. If a client sends `?email=...` or another sensitive value in a query parameter, that value is written verbatim because `path` is not among the six redaction paths. The `requestId` field is likewise written verbatim (BR-15). The `msg` argument is never redacted, so a future call such as `logger.info(\`login for ${email}\`)` would emit the value regardless of the deny-list.

Wildcard depth is another boundary. The `*.` paths match a field named `password`, `passwordHash`, `token`, or `accessToken` one level below the record root. A root-level field `{ password: ... }` does not match `*.password`; a nested field `{ user: { credentials: { token: ... } } }` is two levels below the root and also does not match. Pino's redaction documentation and its redaction implementation establish that a wildcard denotes one path segment; array elements require explicit bracket wildcards such as `items[*].token`. No such paths are configured, and no current call site logs a root-level password or a deep credential object, so the gap is latent rather than observed.

The two `req.headers.*` paths are coupled to a serialization shape that the project does not produce. If a future call site passes an object literally named `req` containing `headers`, the paths would censor `authorization` and `cookie`. Without that shape, an object such as `{ request: { headers: { authorization: ... } } }` is not covered, and the authorization header is never logged by the current middleware anyway. The deny-list also has no entries for common personal data fields present in this domain (`email`, `phone`, `document`, `address`) and no entry for `password` at the root, so the guarantee is limited to the six named locations.

**Rule workflow**:
```
log object -> path matcher
  |-- exact path match, one-level wildcard -> value censored
  |-- root-level credential name -> not matched by "*.name"
  |-- depth >= 2 credential name -> not matched
  |-- msg string -> never inspected
  |-- path/url/query value -> never inspected
  |-- headers under a non-"req" key -> never inspected
  `-- personal-data fields -> no configured path
```

---

### Business Rule: BR-11 - Logger configuration is static for the process lifetime

**Overview**:
Every configuration input is read once during module evaluation: `env.LOG_LEVEL` at line 15, `env.NODE_ENV` at lines 20 and 23, and the literal redaction/format options at lines 4-28. No function in the component accepts overrides, and no consumer changes the logger after construction.

**Detailed description**:
The only exported configurable unit is the factory `createLogger()`, which takes no parameters and always reads the same environment snapshot; it cannot be used to build a logger with a different level, destination, or redaction list. The exported `logger` is a Pino instance, so its methods and mutable properties (including `level`) are reachable by any importer, but no project file mutates them. Configuration stability is therefore a convention rather than an enforced invariant.

There is no configuration reload path. `src/server.ts:20-21` handles `SIGINT` and `SIGTERM` only, and those handlers shut the process down; no `SIGHUP` handler, admin endpoint, or watcher re-reads `LOG_LEVEL`. Changing verbosity requires a restart. This also means an incident cannot be diagnosed by temporarily raising verbosity in a running process, and a noisy deployment cannot be quieted without a restart.

The environment snapshot itself is taken before the logger exists, so a configuration error never produces a structured log record. The failure is written to stderr by `console.error` inside `src/config/env.ts:20-21`, followed by `process.exit(1)`. The logger module does not wrap the environment import in a try/catch and could not meaningfully do so, because it needs a validated level and environment name to build its options.

**Rule workflow**:
```
module import
  -> env snapshot read (LOG_LEVEL, NODE_ENV)
  -> options object built from literals
  -> pino instance constructed
runtime
  -> no re-read of env
  -> no reload signal, endpoint, or watcher
  -> direct mutation of the instance is possible but unused
  -> restart with new LOG_LEVEL required to change verbosity
```

---

### Business Rule: BR-12 - Development formatting depends on a devDependency being resolvable at runtime

**Overview**:
The transport target is the module name `'pino-pretty'` (`src/shared/logger/index.ts:25`). Pino resolves that name inside a worker thread at logger construction. The package is declared in `devDependencies` (`package.json:46`), not in `dependencies`.

**Detailed description**:
There is no import statement for `pino-pretty` in the source, so TypeScript, ESLint, and the build do not verify its presence. Resolution occurs at runtime in the transport worker, which means a successful `tsc` build says nothing about whether the development format can be produced. In a full development or test install, `pino-pretty` is present and the worker starts. In an install that omits dev dependencies, the target cannot be resolved; Pino's transport contract requires the target module to be installed, and the component has no fallback to JSON, no try/catch around construction, and no transport error listener.

The risk is amplified by the environment default. `NODE_ENV` defaults to `development` when absent (`src/config/env.ts:4`), so a production process that does not set `NODE_ENV` selects the pretty transport. A production install with `--omit=dev` therefore combines an omitted module with a configuration that requires it. Conversely, an install with dev dependencies present would start successfully but emit human-readable, multi-line output instead of JSON, which breaks machine parsing downstream even though the process runs.

The transport also introduces asynchronous startup. Pino documents that transports boot asynchronously and that calling `process.exit()` before the transport is ready loses logs. The `bootstrap_failed` path (`src/server.ts:24-27`) and the shutdown path (`src/server.ts:13-18`) both call `process.exit` without waiting. Errors inside the worker are not observed by project code, so a failure can manifest as missing output or as an unhandled worker error rather than as a controlled startup failure.

**Rule workflow**:
```
NODE_ENV === "development"
  -> pino constructs transport { target: "pino-pretty", options: {...} }
  -> thread-stream worker resolves module "pino-pretty" by name
     |-- resolvable -> worker formats records asynchronously -> stdout
     `-- not resolvable -> worker/target failure; no fallback, no error handler
                            in project code
NODE_ENV !== "development"
  -> no dependency on pino-pretty at runtime
```

---

### Business Rule: BR-13 - Call sites follow an implicit record schema and naming convention

**Overview**:
The component does not enforce a schema, but its three consumers use a consistent pattern: a structured object followed by a message, snake_case lifecycle event names, and severity chosen by the nature of the event. `warn` and `debug` are never called anywhere in the project.

**Detailed description**:
Lifecycle events use short snake_case identifiers: `server_started` with `{ port, env }` (`src/server.ts:10`), `shutdown_initiated` with `{ signal }` (`src/server.ts:14`), and `http_server_closed` with no fields (`src/server.ts:15`). The access record uses `http_request` with six fields (`src/middlewares/request-logger.middleware.ts:14-24`). The fatal event uses `bootstrap_failed` with `{ err }` (`src/server.ts:25`). The error event breaks the convention by using a prose sentence, `'Unhandled error in request'` (`src/middlewares/error.middleware.ts:59`). There is no shared enum, constant, or type for event names, so a typo would silently create a new event name that collectors would treat as distinct.

Field naming mixes styles: `requestId` is camelCase, `statusCode` and `durationMs` are camelCase, `service` and `env` are lowercase, and `port`/`signal` are single words. `durationMs` is a rounded floating-point number (`Number(durationMs.toFixed(2))`), which is convenient for display but produces non-integer values in JSON. `userId` is `req.user?.id` and is `undefined` for unauthenticated requests; JSON serialization omits undefined properties, so the key is absent rather than null, which makes consumers distinguish "no user" from "field not present". `path` carries `req.originalUrl`, including the query string.

Level selection is consistent with severity: `info` for startup, access, and shutdown; `error` for unhandled errors; `fatal` for bootstrap failure. Because `warn`, `debug`, and `trace` are never invoked, changing `LOG_LEVEL` to one of those values does not add application records. The single-string call at `src/server.ts:15` is valid Pino usage (message only, no fields), so the `http_server_closed` record has no structured context beyond the base bindings and timestamp.

**Rule workflow**:
```
infrastructure event emission:
  server start      -> info  { port, env }        "server_started"
  request finish    -> info  { requestId, method, path, statusCode,
                               durationMs, userId } "http_request"
  shutdown signal   -> info  { signal }            "shutdown_initiated"
  listener closed   -> info  (no fields)           "http_server_closed"
  unhandled error   -> error { err, requestId, method, path }
                                                   "Unhandled error in request"
  bootstrap failure -> fatal { err }               "bootstrap_failed"
```

---

### Business Rule: BR-14 - Logging is confined to infrastructure; domain events are not logged

**Overview**:
Only three files import the logger, all of them outside `src/modules/`: the server entry point, the request-logger middleware, and the centralized error middleware. No controller, service, repository, state machine, or route module emits a log record.

**Detailed description**:
The import graph confirms the boundary: `src/server.ts:4`, `src/middlewares/request-logger.middleware.ts:3`, and `src/middlewares/error.middleware.ts:5` are the only importers. `src/app.ts` depends on the logger only transitively through the request-logger middleware. The five feature modules, the order state machine, and the repositories have no logging statements at all. As a consequence, the log stream contains one access record per HTTP response and a small number of process-lifecycle records, but no record for domain transitions such as an order status change, stock decrement, authentication failure, or authorization denial.

This is a deliberate-looking separation rather than an accident of omission: the project documentation records the decision that Pino is already present and that no new logging technology will be introduced (`TRANSCRICAO.md:174`), and the error middleware is described there as the central handler for `AppError`, Zod, and Prisma errors. The system's business audit trail is instead persisted in the database (`OrderStatusHistory`, per the architectural report), not in logs. The Logging component therefore cannot be used to reconstruct business history, and log-based business analytics or compliance auditing would have no data source in this stream.

The boundary also means that the logger's field conventions are optimized for HTTP and process diagnostics, not for domain correlation. There is no `orderId`, `customerId`, `userId` on business events (only `userId` on access records), and no event taxonomy beyond the six call sites. Any future domain logging would have to extend the implicit schema of BR-13 and would inherit the redaction limits of BR-10.

**Rule workflow**:
```
src/modules/** (controllers, services, repositories, state machine)
  -> no logger import, no log calls
  -> domain facts persisted through Prisma / OrderStatusHistory
infrastructure layer only:
  server.ts, request-logger.middleware.ts, error.middleware.ts
  -> operational records for lifecycle, access, unhandled errors
log stream != business audit trail
```

---

### Business Rule: BR-15 - Inbound request identifiers are trusted and propagated verbatim

**Overview**:
The access record's correlation key is derived from the `x-request-id` request header when present, with a UUID v4 fallback (`src/middlewares/request-logger.middleware.ts:6`). The value is stored on `req.id`, echoed in the response header, and logged by the access and error records.

**Detailed description**:
Header presence is tested with a nullish check, not a validation. Any string value is accepted, including very long values, control characters that survive header parsing, structured prefixes, and values that collide across clients. Duplicate headers are collapsed by Node into a comma-joined string or, depending on the client and parser, exposed as an array; the code casts the header to `string | undefined` (`line 6`), so an array value would be assigned and logged as an array despite the declared type. There is no length cap, character whitelist, UUID format check, or trust distinction between internally generated and client-supplied identifiers. When the header is absent, `uuidv4()` produces a canonical UUID.

The value is written to the response with `res.setHeader('X-Request-Id', requestId)` (`line 8`). Node rejects header values containing CR/LF, which limits classic header injection, but the same value is serialized into the log record. In JSON mode, control characters are escaped by the serializer, so a newline cannot forge a new JSON record. In development, `pino-pretty` renders field values into terminal output with colorization enabled, so ANSI escape sequences or terminal control characters in a client-supplied request ID could affect the displayed line. The log record still remains a single logical record, but the human-readable rendering is influenced by untrusted input.

Correlation coverage is also partial. The access record includes `requestId`, and the unhandled-error record includes it, but startup, shutdown, and bootstrap-failure records do not (they have no request context). The logger itself has no automatic request context binding, no `AsyncLocalStorage`, and no child logger per request; every call site must pass the field explicitly. Errors handled as `AppError`, `ZodError`, or known Prisma codes are not logged at all, so a client can receive a 400/401/404/409 response with no corresponding log record.

**Rule workflow**:
```
inbound request
  -> header x-request-id present?
       |-- yes -> use verbatim (no format/length/charset validation)
       `-- no  -> uuidv4()
  -> req.id = value
  -> response header X-Request-Id = value
  -> access record field requestId = value (on response finish)
  -> unhandled-error record field requestId = req.id ?? "unknown"
  -> AppError/Zod/Prisma-handled responses: no record
```

---

## 4. Component Structure

```
<project root>/
├── src/
│   ├── shared/
│   │   └── logger/
│   │       └── index.ts                  # Component under analysis (32 lines)
│   │                                     #   redactPaths  : private const, 6 paths (4-11)
│   │                                     #   createLogger : exported factory (13-30)
│   │                                     #     level      : env.LOG_LEVEL (15)
│   │                                     #     redact     : paths + censor (16-19)
│   │                                     #     base       : service, env (20)
│   │                                     #     timestamp  : isoTime (21)
│   │                                     #     transport  : dev-only pino-pretty (22-28)
│   │                                     #   logger       : exported singleton (32)
│   ├── config/
│   │   └── env.ts                        # Efferent dependency: LOG_LEVEL, NODE_ENV (2,15,20,23)
│   ├── middlewares/
│   │   ├── request-logger.middleware.ts  # Afferent consumer: access records (3,14-24)
│   │   └── error.middleware.ts           # Afferent consumer: unhandled errors (5,57-60)
│   ├── server.ts                         # Afferent consumer: lifecycle/fatal (4,10,14,15,25)
│   └── app.ts                            # Transitive consumer via requestLogger (3,60)
├── tests/
│   ├── setup.ts                          # No logger import; environment gate only (2)
│   ├── helpers/
│   │   └── factories.ts                  # Transitive consumer via buildApp (4,10-15)
│   ├── auth.test.ts                      # Indirect exercise: 7 HTTP cases
│   └── orders.test.ts                    # Indirect exercise: 8 HTTP cases
├── package.json                          # pino 9.5.0 (30), pino-pretty 11.3.0 (46),
│                                         # pino-http 10.3.0 declared unused (31)
├── package-lock.json                     # pino 9.5.0, pino-pretty 11.3.0,
│                                         # pino-std-serializers 7.1.0, thread-stream 3.1.0
├── tsconfig.json                         # strict, moduleResolution Bundler, ESM (.js imports)
├── tsconfig.build.json                   # compiles src only into dist
├── vitest.config.ts                      # single fork, no logger configuration
├── .eslintrc.json                        # no-console warn (allows warn/error), type-imports
└── .prettierrc                           # 100 columns, single quotes, 2 spaces
```

### 4.1 Interface surface

| Export | Kind | Signature / Shape | Consumed by |
|--------|------|-------------------|-------------|
| `redactPaths` | module-private const | `string[]` of 6 paths | `createLogger()` only |
| `createLogger` | exported function | `(): Logger` (Pino `Logger` type) | No external call site; module initializer only |
| `logger` | exported const | Pino `Logger` instance | `src/server.ts`, `request-logger.middleware.ts`, `error.middleware.ts` |
| `Logger` type | imported type (not re-exported) | Pino's type, used as the return and const annotation | Internal only (`src/shared/logger/index.ts:1,13,32`) |

The public surface is one value. There is no custom `AppLogger` interface, no wrapper methods, no typed event enum, and no barrel file (`src/shared/logger/` contains only `index.ts`). Consumers import the concrete path with the `.js` extension required by the ESM/Bundler configuration (`src/shared/logger/index.js`).

### 4.2 Configuration options effective at construction

| Option | Value | Source line | Effect |
|--------|-------|-------------|--------|
| `level` | `env.LOG_LEVEL` (default `info`) | 15 | Filters records below the threshold |
| `redact.paths` | 6 literal paths | 4-11, 17 | Deny-list applied to structured fields |
| `redact.censor` | `'[REDACTED]'` | 18 | Replacement string for matched values |
| `redact.remove` | not set (false) | 16-19 | Keys are retained |
| `base` | `{ service: 'order-management-api', env: env.NODE_ENV }` | 20 | Replaces default `pid`/`hostname` bindings |
| `timestamp` | `pino.stdTimeFunctions.isoTime` | 21 | ISO-8601 UTC string in `time` |
| `transport` | `pino-pretty` descriptor iff `NODE_ENV === 'development'`, otherwise `undefined` | 22-28 | Selects async pretty output vs NDJSON |
| `messageKey` | not set | - | Pino default `msg` |
| `errorKey` | not set | - | Pino default `err` |
| `serializers` | not set | - | Pino defaults only |
| destination | not set | - | Pino default stdout (fd 1) |
| `sync` / buffering | not set | - | Library default per destination |
| `enabled` | not set | - | Logger always enabled |
| `formatters` | not set | - | Default bindings/level formatting |

### 4.3 Size and complexity

| Metric | Value |
|--------|-------|
| Production lines in component | 32 |
| Exported symbols | 2 (`createLogger`, `logger`) |
| Private constants | 1 (`redactPaths`) |
| Functions | 1 (`createLogger`) |
| Decision points | 1 (`NODE_ENV` ternary, lines 23-28) |
| Loops | 0 |
| Exception handling | 0 |
| Estimated cyclomatic complexity | 2 |
| External runtime packages directly referenced | 2 (`pino`, `pino-pretty` by name) |
| Internal imports | 1 (`env`) |

---

## 5. Dependency Analysis

### 5.1 Internal dependencies (efferent)

```
src/shared/logger/index.ts
  └── src/config/env.ts  (line 2)
        ├── env.LOG_LEVEL   (line 15)
        ├── env.NODE_ENV    (line 20)
        └── env.NODE_ENV    (line 23)
```

`env` is the only project import. The logger does not import the error taxonomy, HTTP response helper, Prisma client, or any module code, which keeps the shared-infrastructure layer free of upward dependencies and free of cycles.

### 5.2 Internal dependents (afferent)

```
src/server.ts:4                          -> logger.info / logger.fatal
src/middlewares/request-logger.middleware.ts:3 -> logger.info
src/middlewares/error.middleware.ts:5    -> logger.error
        ^                     ^
        |                     |
   src/app.ts:3 (requestLogger)  src/app.ts:73 (errorMiddleware)
        ^
        |
   src/server.ts:1 (buildApp)
        ^
        |
   tests/helpers/factories.ts:4 (buildApp) -> both integration suites
```

Direct importers: 3. Transitive application importers: `src/app.ts`. Transitive test importers: `tests/helpers/factories.ts`, `tests/auth.test.ts`, `tests/orders.test.ts`. The logger module is evaluated once per process regardless of how many of these paths load it.

### 5.3 External dependencies

| Dependency | Version | Declared as | Role in this component | Failure behavior |
|------------|---------|-------------|------------------------|------------------|
| `pino` | 9.5.0 | production dependency (`package.json:30`) | Logger construction, level filtering, redaction, serialization, default stdout destination | Construction-time throw for invalid options; no project-level error handling |
| `pino-pretty` | 11.3.0 | devDependency (`package.json:46`) | Development transport target, loaded by module name in a worker thread | Resolution failure inside worker; no fallback or listener in project code |
| `thread-stream` | 3.1.0 (transitive) | via `pino` transport | Worker-thread transport stream | Async startup; logs can be lost on early `process.exit` |
| `pino-std-serializers` | 7.1.0 (transitive) | via `pino` | Standard `err` serializer | Library-managed |
| Node.js `process` | Node >= 20 (`package.json:8`) | runtime | `process.stdout` (fd 1), `process.hrtime` at call sites | Platform-managed |
| `pino-http` | 10.3.0 | production dependency (`package.json:31`) | **Not imported anywhere** | Not applicable; the custom middleware replaces it |

Version pinning is exact (no ranges) for all direct dependencies. The component uses only Pino's default export and the `Logger` type import; it does not call `pino.transport` directly, does not import `pino/file` or `pino.destination`, and does not use `pino.multistream`.

### 5.4 Dependency graph and ordering

```
env.ts (validated environment snapshot)
   |
   v
logger/index.ts  ---- pino 9.5.0 ---- pino-std-serializers 7.1.0
   |                    |
   |                    +---- (development only) thread-stream 3.1.0
   |                                              |
   |                                              +---- pino-pretty 11.3.0
   |
   +--> request-logger.middleware.ts --> app.ts --> server.ts
   +--> error.middleware.ts -----------------------^
   `--> server.ts
```

Evaluation order is deterministic: `env` validates and exits or returns before `pino()` is called, so the logger never observes a partially validated environment. There is no circular dependency in the graph. The conditional dependency on `pino-pretty` is a runtime string reference, invisible to static analysis and to the TypeScript compiler.

### 5.5 Unused and orphaned dependency surface

`pino-http` 10.3.0 is declared as a production dependency but has no import in `src/` or `tests/`. The project implements request logging manually (`src/middlewares/request-logger.middleware.ts`), and the logger's `req.headers.*` redaction paths reference the object shape that `pino-http` would have produced. This creates an orphaned coupling: the redaction configuration anticipates a serializer that does not exist, while the actual logger never receives a `req` object.

`@types/uuid` is also declared unused, but it belongs to the request-logger middleware rather than to this component.

---

## 6. Afferent and Efferent Coupling

Coupling is counted at the TypeScript module/symbol level. Afferent coupling (Ca) counts project modules that depend on the element; efferent coupling (Ce) counts project modules the element depends on. External library dependencies are listed separately and are not folded into Ca/Ce, to keep the internal architecture metric interpretable.

| Component / Element | Type | Afferent (Ca) | Efferent (Ce) | Instability (Ce / (Ca + Ce)) | Criticality |
|---------------------|------|---------------|---------------|------------------------------|-------------|
| Logger module (`src/shared/logger/index.ts`) | module | 3 direct (server, request logger, error middleware); 4 including app and test factory transitively | 1 internal (`env`) | 0.25 (internal only) | High |
| `logger` singleton | const / Pino instance | 3 direct | 0 internal (Pino methods are external) | 0.00 | High |
| `createLogger` | function | 0 external callers | 2 (env read, pino construction) | 1.00 | Low (dead export) |
| `redactPaths` | private const | 1 (`createLogger`) | 0 | 0.00 | High (security policy) |
| `Logger` type annotation | type import | 0 (internal use only) | 1 external (`pino`) | 1.00 | Low |
| `env` (as consumed here) | dependency | 5 project importers total, 1 from logger | 0 | 0.00 | High |
| Pino configuration object | inline literal | 1 (`pino`) | 0 | 0.00 | High |
| Log record schema (implicit) | convention | 3 call sites | 0 | 0.00 | Medium |

Observations:

- **Low efferent coupling, moderate afferent coupling.** The component knows about exactly one project module and two libraries, while three infrastructure modules know about it. That is the profile of a stable shared utility.
- **Instability is low for the module (0.25) and zero for the exported singleton.** The component is unlikely to change in response to changes in its dependents, which is appropriate for infrastructure.
- **`createLogger` has zero afferent coupling and maximal instability.** It is exported but unused, so it contributes no architectural value while widening the module's public surface.
- **Conceptual coupling extends beyond imports.** The record schema is coupled to three call sites through untyped field names; the redaction list is coupled to credential naming conventions; the base `service` string is coupled to the package identity; and the `req.headers.*` paths are coupled to an unused library's serializer shape that the project does not produce.
- **Privacy criticality is concentrated in one constant.** `redactPaths` is a private, 8-line array that determines whether credentials are masked. It has one internal dependent and no test dependent.

---

## 7. Endpoints

Not applicable. The Logger component exposes no REST, GraphQL, gRPC, CLI, messaging, or scheduled endpoints, and it registers no Express routes. It is an in-process library module. The HTTP endpoints whose requests and errors it records belong to the five feature modules and the router registry, which are outside this component's boundary.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol / Mechanism | Data Format | Error Handling |
|-------------|------|---------|----------------------|-------------|----------------|
| stdout (file descriptor 1) | Outbound sink | Delivery of every record in test and production | Pino default destination write (SonicBoom/stream) | Newline-delimited JSON, one record per line | None in component; no flush, no write-error listener, no backpressure handling |
| `pino-pretty` worker | Outbound transport | Human-readable output in development | `pino.transport` + `thread-stream` worker thread; target resolved by module name | JSON records in, colorized multi-line text out | No `error` listener, no fallback to JSON, no readiness wait; failures surface outside project code |
| `pino` library | Embedded dependency | Level filtering, redaction, serialization, destination management | In-process function calls | JavaScript objects -> JSON strings | Construction-time throw for invalid options; runtime write errors library-managed |
| `thread-stream` | Transitive runtime | Asynchronous transport plumbing | Worker thread + shared buffer | Serialized records | Flush semantics documented as insufficient for `pino-pretty`; logs lost on early exit |
| Environment configuration (`env`) | Inbound configuration | Level, environment name, format switch | ESM import of typed singleton (`src/config/env.ts`) | Validated enum/string values | Fail-fast: invalid config exits the process before the logger exists; no structured record |
| Application call sites | Inbound API | Emit records | `logger.info`, `logger.error`, `logger.fatal` | Object + message string | Fire-and-forget; no call site checks delivery or handles logger exceptions |
| Container / host log collector | Indirect platform integration | Capture and ship stdout | Docker/Kubernetes/systemd log driver, sidecar, or terminal | NDJSON (test/production) or pretty text (development) | Outside the codebase; no configuration, retention, or rotation defined in the project |
| `pino-http` | Declared but unused | Intended HTTP request serialization (presumed) | No import, no call | None | Not applicable |

There are no outbound network calls, no database writes, no message-broker publishes, and no file writes in this component. The logger's integrations are the process's own standard output, a conditional worker thread, and the environment snapshot.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Singleton | `export const logger = createLogger()` at module scope | `src/shared/logger/index.ts:32` | One shared logger for the whole process |
| Factory Function | `createLogger()` wraps `pino(...)` and returns a `Logger` | `src/shared/logger/index.ts:13-30` | Centralize configuration; theoretical test seam |
| Facade | Consumers call only `info`, `error`, `fatal` on a Pino instance; no library types or options leak to call sites | Consumer files | Hide Pino's API surface from application code |
| Options Object | A single configuration literal passed to `pino()` | `src/shared/logger/index.ts:14-29` | Declarative configuration |
| Conditional Strategy (configuration-time) | `NODE_ENV` selects pretty transport vs no transport | `src/shared/logger/index.ts:22-28` | Environment-appropriate output format |
| Deny-list Policy / Data Sanitization | `redact.paths` + `censor` | `src/shared/logger/index.ts:4-11,16-19` | Credential masking at serialization |
| Twelve-Factor Configuration | Level and environment read from environment variables | `src/shared/logger/index.ts:15,20,23`; `src/config/env.ts` | Deployment-specific behavior without code changes |
| Cross-Cutting Concern via Shared Module | Logger imported by middleware and server only | `src/shared/logger/index.ts` | Avoid duplicating logging setup |
| Ambient / Static Dependency | `env` imported directly rather than injected | `src/shared/logger/index.ts:2` | Simplicity at the cost of isolation |
| Fire-and-Forget / No Acknowledgment | Log calls return void; no delivery contract | Consumer files | Non-blocking request path |

### 9.1 Architectural decisions observed

1. **Pino is the single logging technology.** The project transcript records the decision that Pino is already used across the project and that no new logging technology will be added (`TRANSCRICAO.md:174`). The component is the single configuration point for that technology.
2. **JSON by default, pretty only in development.** Machine-readable output is the production contract; readability is confined to `development`. The switch is compile-time fixed by environment.
3. **A single shared instance rather than dependency injection.** The logger is imported directly, not passed through constructors. This keeps controller/service/repository signatures free of logging parameters and keeps the shared kernel out of the domain layer (BR-14), at the cost of a hidden global dependency.
4. **Security at the serialization layer.** Redaction is configured in the logger rather than implemented at each call site. This centralizes the policy but limits it to the configured paths (BR-10).
5. **Stdout as the deployment contract.** The process does not own log files or shipping; the platform is expected to capture stdout. No rotation, retention, or aggregation logic exists in the application.
6. **No custom serializers.** Pino's defaults define the record shape for `err`, `msg`, `time`, and `level`. The project does not define `req`/`res` serializers, which is consistent with not using `pino-http`.
7. **No separation of operational and audit logging.** The same logger and stream carry lifecycle, access, and error records; business audit is delegated to the database.

### 9.2 Architectural boundary assessment

The component sits in the shared-infrastructure layer with a strictly downward dependency on configuration. It has no dependency on Express, Prisma, Zod, the error taxonomy, or any feature module, so it can be imported by any layer without creating a cycle. Its afferents are all infrastructure or process-lifecycle code, which confirms that the boundary between "operational logging" and "domain logic" is respected. The one cross-boundary artifact is the implicit record schema (BR-13): field names and event names are agreed by convention across three files and are not typed or validated, so the boundary is clean at the import level but informal at the data-contract level.

### 9.3 Quality attributes

| Attribute | Assessment | Evidence |
|-----------|------------|----------|
| Cohesion | High: single responsibility (configure and provide a logger), 32 lines, one concern | `src/shared/logger/index.ts` |
| Coupling | Low efferent (1 internal import), moderate afferent (3 direct) | Section 6 |
| Complexity | Very low: one ternary, no loops, no error handling | `src/shared/logger/index.ts:23-28` |
| Observability | Structured JSON, ISO timestamps, request correlation on access/error records; no tracing, metrics, or business events | `src/middlewares/request-logger.middleware.ts`, `src/middlewares/error.middleware.ts` |
| Security | Credential deny-list and censor; gaps for root/deep fields, URLs, messages, and unvalidated request IDs | Sections 3, 10 |
| Performance | One serialization per request; no sampling or buffering configuration; development path crosses a worker thread | Section 10 |
| Testability | Low: module-level side effect, static env import, no destination injection; factory exported but unused by tests | Section 11 |
| Operability | Level controlled by env; format controlled by env; no runtime reload, no flush, no transport error handling | Sections 3, 10 |
| Portability | Node >= 20 required; Pino and worker threads are Node-specific | `package.json:8` |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Shutdown / fatal delivery | `logger.fatal(...)` is immediately followed by `process.exit(1)` (`src/server.ts:24-27`) while development transport is asynchronous; Pino documents log loss on early exit | The root cause of a bootstrap failure can be missing from logs precisely when it is most needed |
| High | Shutdown delivery | `process.exit(0)` follows a non-awaited `server.close` callback and `$disconnect` (`src/server.ts:13-18`); `http_server_closed` may never be emitted and buffered records are not flushed | Incomplete shutdown observability and lost final records |
| High | Redaction coverage | `*.password` does not match a root-level `password` nor nesting two or more levels deep; no paths for `email`, `phone`, `document`, `address`, or error metadata | A future call site that logs a user, customer, or credentials object can emit secrets or PII unmasked |
| High | Redaction scope | `path: req.originalUrl` is logged including the query string (`src/middlewares/request-logger.middleware.ts:18`); `msg` strings are never redacted | Sensitive query parameters or interpolated values appear verbatim in access logs |
| Medium | Deployment dependency | Development transport requires `pino-pretty`, declared only in devDependencies (`package.json:46`); `NODE_ENV` defaults to `development` (`src/config/env.ts:4`) | A production install without dev dependencies and without `NODE_ENV` can fail to construct the transport; no fallback |
| Medium | Transport error handling | No `error` listener on the transport, no try/catch around `pino()`, no readiness wait | Worker/target failures are unmanaged and can surface as missing output or an unhandled worker error |
| Medium | Log loss / no flush | The component exposes no flush/close and no consumer calls `logger.flush()`; Pino documents that `pino-pretty` output is not flushed by `logger.flush()` | Buffered development records are lost on crash or exit |
| Medium | Correlation | No child logger, `AsyncLocalStorage`, or automatic request binding; `requestId` is passed manually by two call sites | Any new log call outside those sites lacks request correlation, fragmenting incident analysis |
| Medium | Handled-error invisibility | `AppError`, `ZodError`, `P2002`, and `P2025` responses are not logged (`src/middlewares/error.middleware.ts:15-54`) | Authentication failures, validation abuse, and conflicts leave no server-side trace beyond the access record status code |
| Medium | Inbound identifier trust | `x-request-id` is accepted without length, charset, or format validation and echoed to the response and logs (`src/middlewares/request-logger.middleware.ts:6-8`) | Log pollution, unbounded field size, terminal escape injection in pretty mode, unreliable correlation across clients |
| Medium | Binding loss | Custom `base` replaces Pino's default `{pid, hostname}` (`src/shared/logger/index.ts:20`) | Records cannot be attributed to a process or host from their own content; multi-replica correlation relies entirely on the platform |
| Medium | Test coverage | No direct tests, no output capture, no redaction assertion, no mock of Pino | Privacy and format regressions are invisible until production logs are inspected |
| Low | Level vocabulary | `LOG_LEVEL` enum excludes `silent` (`src/config/env.ts:6`); no per-module level; no runtime change | Full silence is not configurable; troubleshooting verbosity requires a restart and increases all log volume |
| Low | Dead export | `createLogger` has no callers; `pino-http` is declared but unused; the `Logger` type is not re-exported | Wider public surface and an unused dependency without architectural benefit |
| Low | Record schema | Event names and field names are untyped conventions; one prose message breaks the snake_case pattern | Typos create accidental event types; collectors must handle inconsistent naming |
| Low | Hardcoded identity | `service: 'order-management-api'` and no version/commit/instance field | Cannot distinguish releases or instances within one service name |
| Low | Sink robustness | No handling for closed stdout / EPIPE; default destination options not set | A broken output stream can surface as an unhandled stream error or silently dropped logs |
| Low | Cost of development format | Multi-line, colorized output with `singleLine: false` | Development logs are not machine-parseable and are harder to grep or diff |
| Low | `env` field duplication | `server_started` passes `env` although it is already a base binding (`src/server.ts:10`) | Redundant field in one record; no behavioral impact |
| Low | Timestamp semantics | ISO strings are host-clock based and host identity is absent | Cross-host ordering can be misleading under clock skew |
| Low | Documentation | No README, comment, or type documents the record schema, redaction policy, or environment format switch | Operators and developers must read the implementation to know what is guaranteed |

### 10.1 Security assessment

- **Positive controls:** six credential paths are censored with a constant token; the censor preserves keys so accidental credential logging is detectable; request bodies are never logged; headers are never logged by the custom middleware; the HTTP client receives generic 500 messages while details stay server-side; JSON serialization escapes control characters in structured values; secrets are never placed in base bindings.
- **Policy gaps:** the deny-list does not cover root-level or deeply nested credentials, personal data fields, query strings, message strings, or error metadata; stack traces are emitted in full; the request ID is untrusted input.
- **Dormant controls:** the `req.headers.authorization` and `req.headers.cookie` paths have no reachable trigger because `pino-http` is unused and the custom middleware logs no headers. The apparent header protection is therefore not exercised.
- **Least privilege / integrity:** logs are written to stdout without signing, access control, or tamper evidence; confidentiality and integrity depend entirely on the platform's log pipeline.
- **Denial-of-service surface:** a client can inflate record size with an arbitrarily long `originalUrl` and `x-request-id`; there is no truncation, sampling, or rate limit in the component.
- **Data retention:** no retention or deletion policy exists in code; persistence is platform-defined.

### 10.2 Resilience assessment

| Aspect | Implementation | Observation |
|--------|----------------|-------------|
| Retry | None | Log delivery is fire-and-forget |
| Fallback | None | A failed `pino-pretty` transport has no JSON fallback |
| Circuit breaker | None | Not applicable to an in-process logger; a failing sink is not isolated |
| Timeout | None | No bound on destination or worker delivery |
| Buffering / flush | Library default; no explicit flush | Development worker buffers formatted output; loss on exit |
| Degraded mode | None | Output failure does not degrade gracefully or surface to the application |
| Error isolation | None | Logger construction or transport errors are not caught by consumers |
| Backpressure | Not configured | Destination behavior is library default |
| Sampling / rate limiting | None | One record per event, full stack per error |
| Level-based suppression | `LOG_LEVEL` threshold | The only volume control available |
| Recovery | Process restart with different `LOG_LEVEL`/`NODE_ENV` | No dynamic recovery or reconfiguration |

---

## 11. Test Coverage Analysis

### 11.1 Test inventory relevant to this component

| Test file | Relationship to Logger | Evidence |
|-----------|------------------------|----------|
| `tests/setup.ts` | Indirect environment gate only: imports `prisma`, which imports `env`; no logger import | `tests/setup.ts:2` |
| `tests/helpers/factories.ts` | Transitive construction: `buildApp` imports the request logger, which constructs the singleton; `getTestApp` caches the app | `tests/helpers/factories.ts:4,8-15` |
| `tests/auth.test.ts` | Indirect exercise: 7 test cases issue HTTP requests, each emitting one `http_request` info record; no assertions on output | `tests/auth.test.ts:5-84` |
| `tests/orders.test.ts` | Indirect exercise: 8 test cases issue HTTP requests through the same cached app; no assertions on output | `tests/orders.test.ts:1-219` |
| `vitest.config.ts` | Harness: single fork, no logger mock, no `NODE_ENV` assignment, no setup for log capture | `vitest.config.ts:4-18` |
| `package.json` test script | `vitest run`; no coverage provider or coverage script | `package.json:17` |

There is no `tests/logger.test.ts`, `tests/shared/logger.test.ts`, or any file importing `src/shared/logger` directly. A search for `vi.`, `mock`, `stub`, or `spy` in `tests/` returns no matches, confirming that Pino, the logger module, and `process.stdout` are never mocked or observed. A search for `x-request-id`, `X-Request-Id`, `console`, `stdout`, or `stderr` in `tests/` returns no matches, confirming that no test asserts correlation or output behavior.

### 11.2 Coverage assessment

Coverage was not measured. `package.json:10-21` has no coverage script, and `devDependencies` contains no coverage provider (Vitest requires an additional provider package for coverage). No coverage directory or report exists in the workspace. The assessment below is an analysis estimate of exercised behavior, not a tool-generated number.

| Artifact / behavior | Unit Tests | Integration Tests | Coverage Assessment | Test Quality |
|---------------------|------------|-------------------|---------------------|--------------|
| Module import and singleton construction | None | Executed once per test process via `buildApp` | Lines 1-32 execute; no assertion | Execution only, no verification |
| `createLogger` factory | None | Executed only through the module initializer | Executed; return value never inspected | No isolation seam used |
| Level configuration (BR-01) | None | Default path only; no threshold assertion | Filtering never observed | No test sets or checks `LOG_LEVEL` |
| Redaction (BR-02, BR-03, BR-10) | None | No sensitive field is ever logged by call sites | Matcher compiles; no redaction path is triggered | Zero behavioral coverage |
| Base bindings (BR-04) | None | Every record carries them, but no capture | Not asserted | No field-level assertion |
| ISO timestamp (BR-05) | None | Records emitted but not captured | Not asserted | No format assertion |
| Development transport (BR-06) | None | Branch not taken under `NODE_ENV=test` | Development branch unexecuted | Zero coverage of pretty path |
| JSON branch (BR-06) | None | Taken; output not captured | Executed, not asserted | No format assertion |
| Singleton lifecycle (BR-07) | None | Relied upon implicitly by cached app | Implicitly executed | No identity or reuse assertion |
| Stdout sink (BR-08) | None | Output goes to the test process stdout; not captured | Executed, not asserted | No sink assertion |
| Error serialization (BR-09) | None | No 500 response is asserted in either suite | Logging call site unexecuted | Zero coverage |
| Static configuration (BR-11) | None | None | Not exercised | No reload test |
| `pino-pretty` dependency (BR-12) | None | Not taken in `NODE_ENV=test` | Untested | Deployment risk unverified |
| Call-site conventions (BR-13) | None | Access records emitted for every request | Executed, not asserted | Event names/fields unchecked |
| Domain-logging boundary (BR-14) | None | Implicit; no domain log calls | No coverage needed | Structural, untested |
| Request ID propagation (BR-15) | None | No header is set in tests; UUID fallback is used | Generated and logged, not asserted | Correlation untested; no header injection test |

### 11.3 Testability observations

- **The factory is exported but not used as a seam.** `createLogger()` takes no arguments, reads the static `env` import, and always targets stdout or the pretty worker. A test cannot inject a destination, a level, or a transport without mutating `process.env` before module import, and `env` is evaluated once with `process.exit` on failure.
- **The singleton is created as an import side effect.** Any test that imports `app`, `server`, or either middleware constructs the logger, and ESM caching prevents reconfiguration within the same process. The Vitest configuration deliberately uses a single fork (`vitest.config.ts:11-17`), so all tests share that instance.
- **No output capture exists.** A test could attach `vi.spyOn(process.stdout, 'write')` before importing the module and inspect the serialized lines, but no test does. Because Pino serializes to stdout and may buffer through SonicBoom, naive spying would need to account for stream behavior.
- **No mock of `pino` exists.** Redaction and serialization could be unit-tested by mocking `pino` and asserting the options object, but the project has no mock infrastructure at all.
- **Environment coupling blocks branch testing.** Covering the development transport branch requires `process.env.NODE_ENV=development` before module evaluation and a resolvable `pino-pretty`; covering the JSON branch requires a non-development value. No test manipulates `NODE_ENV`, and no test isolates module state per environment.
- **Error-path coverage requires a forced 500.** Neither suite asserts a 500 response, so the `logger.error` call in the centralized handler is never executed. Even if it were, the record would not be inspected.

### 11.4 Coverage by business rule

| Rule | Direct test | Indirect exercise | Gap |
|------|-------------|-------------------|-----|
| BR-01 Level filtering | None | Default `info` only; no thresholds varied | All non-default levels and suppression behavior unverified |
| BR-02 Redaction paths | None | Matcher compiled; no matching field logged | Every path is behaviorally unverified |
| BR-03 Censor semantics | None | None | Replacement shape and key retention unverified |
| BR-04 Base bindings | None | Records emitted with bindings, not captured | Field presence and duplicate-field behavior unverified |
| BR-05 Timestamp | None | Records emitted, not captured | ISO format and key unverified |
| BR-06 Environment format | None | JSON branch only, not captured | Pretty branch and its options unexecuted |
| BR-07 Singleton | None | Shared instance assumed via cached app | Identity and single-construction unverified |
| BR-08 Stdout sink | None | Writes occur, not captured | Sink, framing, and stderr separation unverified |
| BR-09 Error serialization | None | Call site not executed | No 500 test; serializer output unverified |
| BR-10 Redaction boundary | None | None | URL/PII/msg leakage behavior unverified |
| BR-11 Static configuration | None | None | No reload or mutation test |
| BR-12 Dev dependency | None | Branch not taken | Missing-module failure mode unverified |
| BR-13 Record conventions | None | Event names emitted, not asserted | Naming and field schema unverified |
| BR-14 Domain boundary | None | Structural absence | No assertion preventing future domain imports |
| BR-15 Request ID trust | None | UUID fallback used; no header set | Header propagation, validation, and echo unverified |

---

## 12. Report Metadata

| Item | Value |
|------|-------|
| Component analyzed | Logger |
| Primary file | `src/shared/logger/index.ts` (32 lines) |
| Files inspected | `src/shared/logger/index.ts`, `src/config/env.ts`, `src/server.ts`, `src/app.ts`, `src/middlewares/request-logger.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/config/database.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts`, `src/shared/http/response.ts`, `package.json`, `package-lock.json`, `tsconfig.json`, `tsconfig.build.json`, `vitest.config.ts`, `.eslintrc.json`, `.prettierrc`, `.env.example`, `docker-compose.yml`, `tests/setup.ts`, `tests/helpers/factories.ts`, `tests/auth.test.ts`, `tests/orders.test.ts`, `docs/agents/MANIFEST.md`, `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md`, `docs/agents/dependency-auditor/dependencies-report-2026-11-08_22-50-33.md`, `docs/FDD.md`, `docs/PRD.md`, `docs/RFC.md`, `docs/TRACKER.md`, `TRANSCRICAO.md` |
| Excluded per parameters | `node_modules`, `.git`, `dist`, `coverage` |
| Lines of production code in component | 32 |
| Business rules documented | 15 |
| Direct internal dependencies | 1 (`src/config/env.ts`) |
| External runtime dependencies | 2 direct (`pino` 9.5.0, conditionally `pino-pretty` 11.3.0); 2 transitively relevant (`thread-stream` 3.1.0, `pino-std-serializers` 7.1.0) |
| Internal dependents | 3 direct files; application-wide transitively |
| Endpoints exposed | 0 |
| Direct tests for component | 0 |
| Test files exercising the component indirectly | 2 suites (15 cases), 1 factory, 1 setup file |
| Coverage measurement available | No (no coverage script or provider) |
| Ambiguities encountered | Redaction wildcard depth and binding-replacement semantics are Pino library contracts, confirmed from Pino documentation and its redaction implementation rather than from project code, and are marked Medium confidence where applicable. The effective `NODE_ENV` during `npm test` is not set by any project file; Vitest's documented default of `test` is assumed. The precise failure mode of a missing `pino-pretty` transport (worker error versus process crash) is library-internal and is described as a resolution failure without asserting the exact exception surface. |
