# Component Deep Analysis Report: Environment Configuration

**Component:** Environment Configuration
**Primary files:** `src/config/env.ts`, `.env.example`
**Project:** `order-management-api` (Order Management System REST API)
**Analysis date:** 2026-11-08 23:00:51
**Scope:** entire project root, excluding `node_modules`, `.git`, `dist`, `coverage`
**Output:** `docs/agents/component-deep-analyzer/component-analysis-environment-configuration-2026-11-08_23-00-51.md`

---

## 1. Executive Summary

The Environment Configuration component is the startup configuration gate of the application. It is implemented as a single TypeScript module, `src/config/env.ts` (27 lines), backed by the Zod schema-validation library, plus a companion template file `.env.example` (14 lines) that documents the environment variables expected by the runtime, the Prisma CLI, and the local Docker Compose stack.

The component's role in the system is to convert the untyped, string-only `process.env` map into a typed, validated `Env` object that the rest of the application consumes. It is the first application-owned module evaluated during process startup: `src/server.ts:2` imports it, and through `src/config/database.ts:2`, `src/shared/logger/index.ts:2`, `src/modules/auth/auth.service.ts:4`, and `src/middlewares/auth.middleware.ts:3`, five internal source files depend on the exported `env` singleton. It has zero internal dependencies and one external dependency (`zod` 3.23.8).

The component is small but architecturally central: it controls four defaulted values (`NODE_ENV`, `PORT`, `LOG_LEVEL`, `JWT_EXPIRES_IN`) and two mandatory values (`DATABASE_URL`, `JWT_SECRET`). Its validation policy is fail-fast: any invalid value aggregated across the schema causes a formatted message on `console.error` and an immediate `process.exit(1)` (`src/config/env.ts:16-23`). There is no fallback, retry, partial initialization, or degraded mode.

Key findings:

- The component validates only 6 of the 11 variables documented in `.env.example`. `SHADOW_DATABASE_URL` is consumed directly by Prisma (`prisma/schema.prisma:8`) and the four `MYSQL_*` variables are consumed by Docker Compose (`docker-compose.yml:7-10`); neither path passes through `env.ts`.
- There is no dedicated test file for the component anywhere in `tests/`. Its unit-level coverage of validation branches is 0. It is exercised only indirectly by integration suites that import `src/config/database.ts` and therefore transitively evaluate `env.ts`.
- `JWT_EXPIRES_IN` and `DATABASE_URL` are validated only as strings. Invalid expiration syntax and malformed connection strings pass startup validation and fail later at token signing time or first database connection.
- `NODE_ENV` defaults to `development` when absent, so a deployment that forgets to set it runs with development logging behavior (pretty transport) and the development Prisma log level, not production behavior.
- The exported `env` object is a mutable singleton: it is not frozen, and its name and shape are visible to every consumer.
- No `.env` file exists in the repository workspace (it is gitignored at `.gitignore:4-6`), and the test script does not use Node's `--env-file` flag (`package.json:17`), so tests depend entirely on variables already present in the invoking shell.

---

## 2. Data Flow Analysis

### 2.1 Startup data flow

```
1. Operator or orchestrator starts a process:
   - npm run dev      -> tsx watch --env-file=.env src/server.ts      (package.json:11)
   - npm start        -> node --env-file=.env dist/server.js           (package.json:13)
   - npm run db:seed  -> tsx --env-file=.env prisma/seed.ts            (package.json:16)
   - npx prisma ...   -> Prisma CLI auto-loads .env for DATABASE_URL
                         and SHADOW_DATABASE_URL                        (prisma/schema.prisma:5-9)
   - docker compose   -> Compose reads .env for MYSQL_* interpolation   (docker-compose.yml:7-10)
2. Node's --env-file loader injects file entries into process.env before
   any application module is evaluated.
3. src/server.ts:2 imports ./config/env.js, which triggers ES module
   evaluation of src/config/env.ts.
4. The module defines envSchema (src/config/env.ts:3-10).
5. loadEnv() calls envSchema.safeParse(process.env) (src/config/env.ts:15).
6. Zod applies, per field: enum restriction, string-to-number coercion,
   integer/positive checks, minimum-length checks, and defaults; unknown
   keys are stripped from the result.
7. Failure path: parsed.success === false -> issues are formatted as
   "  - <path>: <message>" (src/config/env.ts:17-19) -> console.error
   (src/config/env.ts:20-21) -> process.exit(1) (src/config/env.ts:22).
   No env object is ever exported; the process terminates before Express,
   Prisma, or Pino are constructed.
8. Success path: parsed.data is returned (src/config/env.ts:24) and bound
   to the exported const env (src/config/env.ts:27).
9. Consumers read the singleton at their own module-evaluation time:
   - src/server.ts:9-10                 -> env.PORT, env.NODE_ENV
   - src/config/database.ts:6           -> env.NODE_ENV (Prisma log level)
   - src/shared/logger/index.ts:15,20,23 -> env.LOG_LEVEL, env.NODE_ENV
   - src/modules/auth/auth.service.ts:43,48-49 -> env.JWT_EXPIRES_IN,
                                                 env.JWT_SECRET
   - src/middlewares/auth.middleware.ts:41     -> env.JWT_SECRET
10. Independent configuration paths that never see env.ts:
   - Prisma CLI reads DATABASE_URL / SHADOW_DATABASE_URL from .env.
   - Docker Compose reads MYSQL_ROOT_PASSWORD / MYSQL_DATABASE / MYSQL_USER /
     MYSQL_PASSWORD from .env, with hardcoded fallbacks.
   - The seed script constructs its own PrismaClient (prisma/seed.ts:4) and
     inherits DATABASE_URL from process.env supplied by --env-file.
```

### 2.2 Test-time data flow

```
1. vitest run (package.json:17) starts the runner with the include pattern
   tests/**/*.test.ts and setup file tests/setup.ts (vitest.config.ts:7,10).
2. Vitest does not load .env automatically and the test script has no
   --env-file flag, so process.env must already contain DATABASE_URL and
   JWT_SECRET from the invoking shell.
3. tests/setup.ts:2 imports prisma from src/config/database.js, which
   imports env from src/config/env.js -> validation runs during test setup.
4. Invalid configuration aborts the entire test process with exit code 1
   before beforeAll (tests/setup.ts:4-6) executes.
5. Valid configuration allows beforeEach (tests/setup.ts:8-16) to delete all
   rows in seven tables of whatever database DATABASE_URL points at.
6. tests/helpers/factories.ts:3-5 imports the same prisma singleton and
   buildApp; tokens created during login use env.JWT_SECRET and
   env.JWT_EXPIRES_IN at src/modules/auth/auth.service.ts:48-49.
```

---

## 3. Business Rules & Logic

## Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location |
|---------|-----------|------------------|----------|
| BR-01 | Validation | `NODE_ENV` must be one of `development`, `test`, `production`; defaults to `development` when absent | `src/config/env.ts:4` |
| BR-02 | Validation | `PORT` is coerced from string to a positive integer; defaults to `3000` | `src/config/env.ts:5` |
| BR-03 | Validation | `LOG_LEVEL` must be one of `fatal`, `error`, `warn`, `info`, `debug`, `trace`; defaults to `info` | `src/config/env.ts:6` |
| BR-04 | Validation | `DATABASE_URL` is mandatory and must be a non-empty string | `src/config/env.ts:7` |
| BR-05 | Validation | `JWT_SECRET` is mandatory and must contain at least 16 characters | `src/config/env.ts:8` |
| BR-06 | Validation / Default | `JWT_EXPIRES_IN` is a free-form string with default `8h`; its format is not validated | `src/config/env.ts:9` |
| BR-07 | Validation | Defaults apply only to absent (`undefined`) variables; empty strings are validated, not defaulted | `src/config/env.ts:4-9` (Zod `.default()` semantics) |
| BR-08 | Error Handling | All schema violations are aggregated, printed to stderr, and the process exits with code 1 | `src/config/env.ts:15-23` |
| BR-09 | Data Shaping | Only the six declared variables are exposed; unknown environment variables are stripped from the typed result | `src/config/env.ts:3-10,24,27` |
| BR-10 | Lifecycle | Validation runs exactly once per process at first module import; the resulting object is a shared singleton | `src/config/env.ts:27` |
| BR-11 | Configuration Boundary | `.env.example` documents 11 variables, but only 6 are validated by this component; `SHADOW_DATABASE_URL` and `MYSQL_*` follow separate, unvalidated paths | `.env.example:1-14`, `prisma/schema.prisma:8`, `docker-compose.yml:7-10` |
| BR-12 | Diagnostics / Security | Failure output lists field path and validator message only; secret values are not echoed, except invalid enum values | `src/config/env.ts:17-21` |
| BR-13 | Environment-driven Behavior | `NODE_ENV` selects development-only logging transport and Prisma log verbosity in consumers | `src/shared/logger/index.ts:22-28`, `src/config/database.ts:6` |

## Detailed breakdown of the business rules

---

### Business Rule: BR-01 - NODE_ENV must be a declared environment name

**Overview**:
`NODE_ENV` is constrained to the closed set `development`, `test`, `production`, with `development` as the default when the variable is absent (`src/config/env.ts:4`). The typed result is a union of three string literals, not a general string.

**Detailed description**:
The rule establishes the runtime mode of the process before any other component initializes. Because the schema uses `z.enum`, Zod rejects any value outside the three allowed literals, including common variants such as `prod`, `Production`, `staging`, `local`, or an empty string. The check is exact and case-sensitive. Rejection is not repaired by normalization: there is no lowercasing or alias mapping, so `NODE_ENV=PROD` is a hard startup failure. In a correctly configured process, the value flows to three consumers that branch on the literal `development`: the logger selects a `pino-pretty` transport only when the value equals `development` (`src/shared/logger/index.ts:23`), the Prisma client selects `['warn','error']` versus `['error']` logging (`src/config/database.ts:6`), and the server logs the value at startup (`src/server.ts:10`).

The default is significant from a business-behavior perspective. A deployment that simply omits `NODE_ENV` does not fail; it silently runs in `development` mode. In that mode the logger attempts to load the `pino-pretty` transport and emits human-formatted, colorized logs rather than structured JSON, and Prisma emits warnings in addition to errors. The system therefore has a "permissive default" rather than a "production-safe default". There is no rule anywhere in the component that requires `production` to be set explicitly, and no rule that cross-checks `NODE_ENV` against other values (for example, forbidding the sample JWT secret in production).

The value is also consumed implicitly by the test harness decision, because test execution normally expects `NODE_ENV=test`, but nothing in the component or in `tests/setup.ts` asserts that state. The `Env` type exported at `src/config/env.ts:12` narrows the field to the three literals, but no source file imports that type; consumers only read the runtime property. Consequently, TypeScript prevents assigning other literals inside the codebase, but the guarantee originates from startup validation, not from the type system at the boundary.

**Rule workflow**:
```
process.env.NODE_ENV
  |-- undefined -> default "development" -> accepted
  |-- "development" | "test" | "production" -> accepted, passed through unchanged
  |-- any other value (including "", "prod", "Production", "staging")
        -> Zod invalid_enum_value issue for path "NODE_ENV"
        -> aggregated into failure output
        -> console.error + process.exit(1)
```

---

### Business Rule: BR-02 - PORT must be a positive integer and defaults to 3000

**Overview**:
`PORT` is declared as `z.coerce.number().int().positive().default(3000)` (`src/config/env.ts:5`). String input from the environment is converted to a number, must be an integer, must be greater than zero, and defaults to `3000` when absent.

**Detailed description**:
Environment variables are always strings, so this rule is the component's only numeric coercion boundary. `z.coerce.number()` applies JavaScript `Number()` semantics before the numeric refinements run. That means accepted forms include plain decimal text (`"8080"`), exponent notation (`"3e3"` becomes 3000), hexadecimal (`"0x10"` becomes 16), and text with surrounding whitespace (`" 3000 "` becomes 3000). The `.int()` refinement then rejects fractional results such as `"3000.5"`, and `.positive()` rejects zero and negative numbers. A non-numeric string such as `"http"` becomes `NaN` and fails the numeric check. An empty string becomes `0` and is rejected by `.positive()` before any default can apply, because the variable is present.

The upper bound is not constrained. Values above the TCP port range, for example `70000`, pass validation and only fail later when `app.listen(env.PORT)` is invoked at `src/server.ts:9`. Node throws a `RangeError` in that situation, which is caught by the bootstrap rejection handler at `src/server.ts:24-27`, logged as `bootstrap_failed` through the Pino logger, and followed by `process.exit(1)`. The failure is therefore still fatal and observable, but it occurs after logger and Prisma initialization rather than at the configuration gate. Port `0`, which Node treats as "assign an ephemeral port", is not permitted by this rule because zero fails `.positive()`.

The default of `3000` is applied only when the variable is absent. Downstream, the value is used once for the HTTP listener and once for the startup log line (`src/server.ts:9-10`). No other component reads `PORT`, and no documentation file in the repository records the effective default other than the schema and `.env.example:2`.

**Rule workflow**:
```
process.env.PORT
  |-- undefined -> default 3000 -> accepted
  |-- numeric string convertible to positive integer -> accepted as number
  |-- "" -> Number("") = 0 -> positive() fails -> startup failure
  |-- "3000.5" -> int() fails -> startup failure
  |-- "-1" or "0" -> positive() fails -> startup failure
  |-- "abc" -> NaN -> number check fails -> startup failure
  |-- "70000" -> validation passes -> listen() RangeError at src/server.ts:9
                 -> bootstrap catch -> logger.fatal -> process.exit(1)
```

---

### Business Rule: BR-03 - LOG_LEVEL must be a valid Pino level

**Overview**:
`LOG_LEVEL` is restricted to the six standard Pino levels `fatal`, `error`, `warn`, `info`, `debug`, `trace`, with `info` as the default (`src/config/env.ts:6`). The value is passed unchanged to `pino({ level: env.LOG_LEVEL })` at `src/shared/logger/index.ts:15`.

**Detailed description**:
The rule aligns configuration validation with the logger library's expected vocabulary. Because the check is a strict enum, Pino-specific values that are not in the list, such as `silent`, are rejected even though Pino itself supports a `silent` level. Likewise, numeric levels (`"30"`), uppercase levels (`"INFO"`), and comma-separated multi-level strings are rejected. This guarantees that a successfully started process can construct its logger without Pino throwing on an unknown level, and it makes the set of operational verbosity settings explicit and enumerable.

The default `info` applies when the variable is absent and represents the component's chosen baseline verbosity. There is no environment-aware adjustment: the schema does not derive a stricter level for production or a more verbose level for development, and `NODE_ENV` does not influence `LOG_LEVEL`. An operator who sets `LOG_LEVEL=debug` in production passes validation and obtains debug-level output. Conversely, an operator who needs to silence logs entirely cannot do so through this variable, because `silent` is outside the enum and the component has no alternative switch.

The value is a cross-cutting operational control rather than a domain value. It affects every log statement emitted through the shared logger, including request logs, startup logs, shutdown logs, and fatal bootstrap errors. When validation fails for this field, the failure itself is reported through `console.error`, not through the logger, because the logger module cannot be constructed until `env` exists. This ordering is a direct consequence of the dependency chain `logger -> env`.

**Rule workflow**:
```
process.env.LOG_LEVEL
  |-- undefined -> default "info" -> accepted
  |-- one of fatal|error|warn|info|debug|trace -> accepted
  |-- "" | "INFO" | "silent" | "30" -> invalid_enum_value
        -> aggregated failure output
        -> console.error + process.exit(1)
  |-- accepted -> pino({ level }) at src/shared/logger/index.ts:15
```

---

### Business Rule: BR-04 - DATABASE_URL is mandatory and non-empty

**Overview**:
`DATABASE_URL` is declared as `z.string().min(1, 'DATABASE_URL is required')` (`src/config/env.ts:7`). It has no default, so its absence is always fatal, and it must contain at least one character.

**Detailed description**:
This rule is the component's only gate for database connectivity configuration. Because the field has no `.default()`, an unset variable produces a required-field issue, and because `.min(1)` is applied, an explicitly empty string produces a minimum-length issue. Both messages are effectively equivalent to the operator: the database location must be supplied. The value itself is not parsed, normalized, trimmed, or checked against a connection-string grammar. Any non-empty text passes, including text that is not a URL at all.

The practical consequence is that semantic validation of the connection string is deferred to Prisma. `prisma/schema.prisma:7` reads `env("DATABASE_URL")` for the datasource URL, and the generated client validates or uses the value when the first connection is attempted. In the normal server bootstrap, `PrismaClient` is constructed at `src/config/database.ts:10` during module evaluation, but the actual connection happens lazily on first query; the test harness explicitly calls `prisma.$connect()` in `beforeAll` (`tests/setup.ts:4-6`). A malformed URL therefore produces a startup-time or first-query failure attributed to Prisma, not a configuration-gate failure attributed to `env.ts`. This weakens the fail-fast promise for this specific variable: only presence is fail-fast, not correctness.

The variable is also the most security-sensitive string in the configuration because MySQL connection URLs conventionally embed username and password. The component never logs the value, and the failure messages for this field never echo it. However, the value remains resident in `process.env` and in the exported `env` object for the process lifetime, and it is available to any code with access to either. The `.env.example` template supplies a concrete example URL with credentials (`mysql://oms_user:oms_password@localhost:3306/oms`, `.env.example:5`), which demonstrates the expected format but also establishes reusable default credentials for local environments.

**Rule workflow**:
```
process.env.DATABASE_URL
  |-- undefined -> "DATABASE_URL is required" -> startup failure
  |-- "" -> min(1) failure -> startup failure
  |-- any non-empty string (including malformed URLs) -> accepted
  |-- accepted -> consumed by PrismaClient (src/config/database.ts) and by
                  the Prisma CLI datasource (prisma/schema.prisma:7)
```

---

### Business Rule: BR-05 - JWT_SECRET is mandatory with a 16-character minimum

**Overview**:
`JWT_SECRET` is declared as `z.string().min(16, 'JWT_SECRET must be at least 16 characters')` (`src/config/env.ts:8`). It has no default, must be present, and must meet a minimum length of 16 UTF-16 code units.

**Detailed description**:
This rule governs the symmetric key used for both signing and verification of access tokens. `src/modules/auth/auth.service.ts:49` signs with `env.JWT_SECRET`, and `src/middlewares/auth.middleware.ts:41` verifies with the same value. A mismatch between the value used at signing time and the value used at verification time is not detectable by this component; it manifests as every protected request returning `401` with code `UNAUTHORIZED`. Within a single process the singleton guarantees consistency, but across multiple processes or deployments the rule only guarantees that each process has some 16-plus-character string.

The validation is purely length-based. There is no entropy check, no character-class requirement, no blocklist of known placeholder values, and no comparison against the sample value published in `.env.example:8` (`change-me-in-production-please-use-a-long-random-secret`, which is 51 characters and passes). A value of 16 spaces, if quoted, passes; a repeated character such as `aaaaaaaaaaaaaaaa` passes; a low-entropy human phrase of sufficient length passes. The minimum is also quite low relative to the HS256 algorithm used by `jsonwebtoken`, where the security of the HMAC depends on the key's entropy rather than its character count.

Because there is no default, the component cannot start without this variable. This is deliberate and distinct from the permissive defaults on `NODE_ENV`, `PORT`, `LOG_LEVEL`, and `JWT_EXPIRES_IN`. The failure message does not reveal the supplied value or its actual length, so an operator can distinguish "missing or too short" from other configuration failures without leaking the secret into logs. The value is never written by the component to any log, error message, or response; the logger's redaction paths (`src/shared/logger/index.ts:4-11`) additionally cover token-bearing fields, though they do not cover an environment dump, which the application does not perform.

**Rule workflow**:
```
process.env.JWT_SECRET
  |-- undefined -> "JWT_SECRET must be at least 16 characters" -> exit 1
  |-- length < 16 -> same failure -> exit 1
  |-- length >= 16 (any content) -> accepted
  |-- accepted -> jwt.sign(..., env.JWT_SECRET)   (auth.service.ts:49)
                  jwt.verify(token, env.JWT_SECRET) (auth.middleware.ts:41)
```

---

### Business Rule: BR-06 - JWT_EXPIRES_IN is a free-form duration string defaulting to 8h

**Overview**:
`JWT_EXPIRES_IN` is declared as `z.string().default('8h')` (`src/config/env.ts:9`). It is not validated against any duration grammar, and it is cast to `SignOptions['expiresIn']` when used (`src/modules/auth/auth.service.ts:48`). It is also returned verbatim to API clients in the login response (`src/modules/auth/auth.service.ts:43`).

**Detailed description**:
This rule controls token lifetime, which is a security and user-experience parameter rather than a connectivity parameter. The schema accepts any string, including values that `jsonwebtoken` cannot interpret, such as `"8 hours"` with a space, `"forever"`, or an empty string. The cast at `src/modules/auth/auth.service.ts:48` suppresses the TypeScript distinction between the broad `string` type and the library's narrower `StringValue | number` type, so the compiler does not flag the mismatch. When an invalid value is supplied, the failure occurs at login time inside `jwt.sign`, not during startup, and it surfaces through the centralized error middleware rather than the configuration gate.

The default `8h` is the only duration the component guarantees to be valid, because it is a literal chosen by the authors and known to be accepted by the library's `ms`-style parser. There is no validation that the duration is positive, bounded, or expressed in a specific unit. There is no cross-field rule tying `JWT_EXPIRES_IN` to `NODE_ENV`, and there is no refresh-token or revocation configuration in this component. The value is also exposed to clients as `tokens.expiresIn` in the login payload (`src/modules/auth/auth.service.ts:43`), so an invalid or surprising value becomes part of the public API contract for that response.

The empty-string case is the sharpest edge of this rule. Because a variable that is present with an empty value is not `undefined`, Zod's `.default('8h')` does not apply, and `z.string()` accepts `""`. The resulting `env.JWT_EXPIRES_IN` is an empty string that passes startup validation, is returned to clients as `expiresIn: ""`, and causes `jwt.sign` to throw when a user attempts to log in. This is a case where presence semantics (BR-07) and weak validation combine to move a configuration error from startup to runtime.

**Rule workflow**:
```
process.env.JWT_EXPIRES_IN
  |-- undefined -> default "8h" -> accepted
  |-- "" -> accepted by z.string() (default not applied)
  |-- any other string ("15m", "1d", "forever", "8 hours") -> accepted
  |-- accepted -> returned in login response (auth.service.ts:43)
                  -> cast and passed to jwt.sign (auth.service.ts:48-49)
  |-- invalid at signing time -> runtime error during POST /api/v1/auth/login
```

---

### Business Rule: BR-07 - Defaults apply only to absent variables, never to empty strings

**Overview**:
The `.default()` modifiers on `NODE_ENV`, `PORT`, `LOG_LEVEL`, and `JWT_EXPIRES_IN` (`src/config/env.ts:4-6,9`) take effect only when the corresponding property is `undefined`. A variable that exists with an empty string is treated as a supplied value and is validated as such.

**Detailed description**:
This rule is inherited from Zod's default semantics and is not written explicitly in the code, but it determines observable behavior at startup. When Node loads an environment file containing `PORT=`, the property exists and its value is the empty string; `undefined` is not produced. The same applies to shell exports such as `export LOG_LEVEL=`. Under those conditions, `PORT` is coerced to `0` and rejected by `.positive()`, `NODE_ENV` and `LOG_LEVEL` fail their enum checks, and `JWT_EXPIRES_IN` passes as an empty string. The asymmetry between these outcomes is a property of the individual field schemas, not of a single uniform policy.

For operators, the practical effect is that clearing a value in `.env` is not equivalent to deleting the line. Deleting the line yields the documented default; leaving the key with no value yields a startup failure for three fields and a latent runtime failure for the fourth. `.env.example` does not state this distinction, and there is no comment in the file or in `env.ts` describing default behavior for absent versus empty values.

The same rule applies to whitespace. No field is trimmed. A `PORT` value of `"   "` becomes `NaN` and fails, while a `NODE_ENV` value of `" development"` fails the enum because of the leading space. `DATABASE_URL` and `JWT_SECRET` with leading or trailing whitespace are accepted as-is if they meet the length rule, and the whitespace becomes part of the value passed to Prisma or to the JWT library. The component performs no normalization of any kind.

**Rule workflow**:
```
for each defaulted field:
  property === undefined
    -> schema default is used (development / 3000 / info / "8h")
  property === ""
    -> default is NOT used
    -> field-specific validation decides:
         NODE_ENV       -> invalid enum -> exit 1
         PORT           -> coerced 0 -> positive() -> exit 1
         LOG_LEVEL      -> invalid enum -> exit 1
         JWT_EXPIRES_IN -> accepted as "" -> latent runtime failure
```

---

### Business Rule: BR-08 - Validation failures are aggregated and terminate the process

**Overview**:
`loadEnv()` uses `envSchema.safeParse(process.env)` rather than `parse`, collects all issues, formats them, writes them to stderr with `console.error`, and calls `process.exit(1)` (`src/config/env.ts:15-23`). No exception is thrown to a caller and no partial configuration is returned.

**Detailed description**:
The aggregation behavior is driven by Zod's issue collection. A single `safeParse` call reports every failing field, so an operator who is missing `DATABASE_URL` and has an invalid `PORT` sees both problems in one run instead of discovering them sequentially. Each issue is rendered as two spaces, a hyphen, a space, the dot-joined issue path, a colon, a space, and the validator message (`src/config/env.ts:17-19`). Because the schema is flat, paths are always single field names. The resulting block is prefixed with `Invalid environment configuration:` and a newline (`src/config/env.ts:21`).

Termination is immediate and unconditional on failure. `process.exit(1)` is called synchronously inside the module top-level expression `loadEnv()`, so the process ends during module evaluation. At that point the Express application has not been constructed (`src/app.ts`), the Prisma client singleton has not been created (`src/config/database.ts:10`), and the Pino logger has not been created (`src/shared/logger/index.ts:32`). There is consequently no resource to close, which makes the abrupt exit safe in practice, but it also means the failure is not recorded in the application's structured log stream. The only consumer of the message is whatever reads the process's stderr.

Because the failure mechanism is `process.exit` rather than a thrown error, the module is effectively untestable in-process: importing it with invalid `process.env` terminates the test worker. There is no exported function that returns a result or error, no injection point for a custom environment map, and no way to reset the module-level state. The `loadEnv` function is not exported; only the fully evaluated `env` constant is. This design choice is the direct cause of the unit-test gap documented in section 11. Additionally, the `// eslint-disable-next-line no-console` comment at `src/config/env.ts:20` is redundant with the project's ESLint configuration, which already allows `console.warn` and `console.error` (`.eslintrc.json:28`).

**Rule workflow**:
```
envSchema.safeParse(process.env)
  |-- success: true
  |     -> parsed.data returned (env.ts:24)
  |     -> assigned to exported env (env.ts:27)
  |     -> consumers proceed
  |-- success: false
        -> parsed.error.issues.map(issue => "  - path: message").join("\n")
        -> console.error("Invalid environment configuration:\n" + issues)
        -> process.exit(1)
        -> no further application code executes
```

---

### Business Rule: BR-09 - Only declared variables are exposed; unknown variables are stripped

**Overview**:
The schema is a plain `z.object` without `.passthrough()` or `.strict()` (`src/config/env.ts:3-10`). Zod strips unknown keys by default, so the exported `env` object contains exactly six properties and none of the many other variables present in `process.env`.

**Detailed description**:
This rule defines the effective configuration surface visible to application code. `process.env` in a typical process contains dozens of entries, including `PATH`, `HOME`, `SHELL`, `SHADOW_DATABASE_URL`, the four `MYSQL_*` variables, and any CI or orchestrator metadata. None of them appear on `env`. Consumers therefore cannot accidentally read an undeclared variable through the typed object, and TypeScript autocompletion reflects only the six declared fields. This is a deliberate encapsulation of configuration: the schema is the single source of truth for what the application considers configuration.

Stripping also means that a typo in a variable name is not silently ignored at the level of the typed object. If an operator sets `DATABSE_URL` and omits `DATABASE_URL`, the misspelled variable is stripped and the required-field check for `DATABASE_URL` fails, producing a startup error. The error message names the correct variable, which is operationally helpful, though it does not name the misspelled one. There is no rule that warns about unknown or suspiciously similar keys, and `.strict()` is not used, so extra variables never cause failure.

The rule has a second consequence for the boundary between this component and the Prisma CLI. `SHADOW_DATABASE_URL` is present in `.env` and in `.env.example`, but it is not part of the schema, so it is stripped from `env`. Prisma reads it independently from the environment when the CLI runs (`prisma/schema.prisma:8`). Application code that needed the shadow URL would not find it on `env`; it would have to read `process.env` directly, which no source file does.

**Rule workflow**:
```
process.env (all variables)
  -> z.object({ six declared fields }).safeParse
  -> declared fields validated and retained
  -> undeclared fields (SHADOW_DATABASE_URL, MYSQL_*, PATH, HOME, ...) dropped
  -> exported env has exactly:
       NODE_ENV, PORT, LOG_LEVEL, DATABASE_URL, JWT_SECRET, JWT_EXPIRES_IN
```

---

### Business Rule: BR-10 - Configuration is validated once at module import and shared as a singleton

**Overview**:
The exported binding is a constant initialized by a top-level call: `export const env: Env = loadEnv();` (`src/config/env.ts:27`). ES module evaluation caches the module, so validation runs once per process and every importer receives the same object reference.

**Detailed description**:
The rule fixes the time and cardinality of configuration loading. Validation happens at the moment the module graph first reaches `src/config/env.ts`, which in the production entry point is during the import of `src/server.ts:2`. Because ES modules are evaluated once, the second through fifth importers do not re-run validation; they read the already-populated binding. This guarantees intra-process consistency: the secret used to sign a token and the secret used to verify it are the same value, and the `PORT` used for `listen` is the same value reported in the startup log.

The singleton is a mutable object, however. `export const env` prevents rebinding the name, but it does not freeze the object or make its properties read-only. Any code holding the reference can assign `env.PORT = 9999` or `env.JWT_SECRET = 'x'` at runtime, and TypeScript would reject the assignment only if it knows the property type and the object is not widened; at runtime nothing prevents it. No source file currently mutates `env`, and there is no `Object.freeze`. The rule is therefore "loaded once and shared", not "immutable after load".

Import order matters for the failure experience but not for correctness. In `src/server.ts`, `app.js` is imported at line 1 before `env.js` at line 2; because `app.ts` does not import `env` directly, the first evaluation of `env.ts` still occurs at line 2. Through `src/config/database.ts:2` and `src/shared/logger/index.ts:2`, the environment module would be reached even if `server.ts` did not import it directly. In tests, the first reach is `tests/setup.ts:2` through the Prisma provider. In every case, invalid configuration aborts the process before the dependent module finishes evaluating.

**Rule workflow**:
```
first import of src/config/env.ts in the process
  -> module evaluation
  -> loadEnv() runs
  -> env binding assigned (or process exits)
subsequent imports
  -> cached module namespace returned
  -> same env object reference, no re-validation
runtime
  -> properties are readable and writable (no freeze)
  -> no reload or refresh mechanism exists
```

---

### Business Rule: BR-11 - The documented configuration surface is larger than the validated surface

**Overview**:
`.env.example` documents 11 variables across 14 lines (`.env.example:1-14`), while `src/config/env.ts:3-10` validates 6. `SHADOW_DATABASE_URL` is read by Prisma CLI, and `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, and `MYSQL_PASSWORD` are read by Docker Compose with fallback defaults.

**Detailed description**:
The component is one of three independent consumers of the same `.env` file. The application runtime uses Node's `--env-file` flag (`package.json:11,13,16`) and then validates six fields through Zod. The Prisma CLI auto-loads `.env` and resolves `env("DATABASE_URL")` and `env("SHADOW_DATABASE_URL")` directly from the schema (`prisma/schema.prisma:5-9`). Docker Compose reads `.env` for interpolation and applies its own fallbacks: `${MYSQL_ROOT_PASSWORD:-root_password}`, `${MYSQL_DATABASE:-oms}`, `${MYSQL_USER:-oms_user}`, `${MYSQL_PASSWORD:-oms_password}` (`docker-compose.yml:7-10`).

The consequence is an uneven validation boundary. `DATABASE_URL` is present-checked by this component and also consumed by Prisma; `SHADOW_DATABASE_URL` is consumed only by Prisma and is not present-checked anywhere, so `prisma migrate dev` fails at CLI level if it is absent. The `MYSQL_*` variables are never seen by application code and are not validated by any schema; they configure the container that the application expects to connect to. No rule verifies that the database named in `MYSQL_DATABASE` matches the database path in `DATABASE_URL`, that `MYSQL_USER` and `MYSQL_PASSWORD` match the credentials embedded in `DATABASE_URL`, or that `SHADOW_DATABASE_URL` points to a distinct database. These relationships are maintained by convention in `.env.example:5-6,11-14`.

The template also carries concrete default credentials and a sample JWT secret. Those values are not enforced by `env.ts`; the secret simply happens to exceed the 16-character minimum, and the credentials happen to match the Compose fallbacks. The component therefore validates the presence and shape of a subset of configuration while leaving the coherence of the overall local stack to the operator.

**Rule workflow**:
```
.env / process environment
  |-- six validated fields -> src/config/env.ts -> application runtime
  |       NODE_ENV, PORT, LOG_LEVEL, DATABASE_URL, JWT_SECRET, JWT_EXPIRES_IN
  |-- SHADOW_DATABASE_URL -> Prisma CLI only (prisma/schema.prisma:8)
  |       no presence check anywhere
  |-- MYSQL_ROOT_PASSWORD, MYSQL_DATABASE, MYSQL_USER, MYSQL_PASSWORD
  |       -> Docker Compose interpolation only (docker-compose.yml:7-10)
  |       -> fallbacks applied when absent
  |-- no cross-variable consistency checks exist
```

---

### Business Rule: BR-12 - Failure diagnostics expose field names but not secret values

**Overview**:
The failure output is constructed exclusively from `issue.path` and `issue.message` (`src/config/env.ts:17-19`). It is written with `console.error` and then the process exits. The component never interpolates the raw environment value into a message.

**Detailed description**:
For most validators the message is static text defined in the schema: `DATABASE_URL is required` and `JWT_SECRET must be at least 16 characters`. Those two messages are the ones attached to the sensitive fields, and neither echoes the supplied value or its length. For `z.enum` failures, however, Zod's `invalid_enum_value` message includes the received value, so an invalid `NODE_ENV` or `LOG_LEVEL` is printed verbatim. Those two values are operational labels rather than credentials, so the disclosure is low impact, but it is still a case where operator-supplied text reaches stderr unescaped. Numeric coercion failures likewise describe the expected type rather than reproducing arbitrary content.

The diagnostics path is deliberately independent of the application logger. `src/shared/logger/index.ts:2` imports `env`, so the logger cannot exist until validation succeeds; using it for configuration errors would create a circular dependency. The component therefore falls back to `console.error`, which writes to stderr without JSON structure, correlation identifiers, service name, or timestamps. In a container environment that collects stdout only, a configuration failure may be less visible than other startup logs. The chosen exit code, `1`, is the only machine-readable signal available to a supervisor.

There is no redaction layer in this component because there is no intentional value logging. The risk of secret exposure through this path is limited to future changes: the formatting code prints `issue.message`, and if a custom validator were written to include the input in its message, that input would be printed. No such custom validator exists today. The logger's own redaction paths (`src/shared/logger/index.ts:4-11`) cover authorization headers, cookies, passwords, password hashes, tokens, and access tokens, but they are not involved in configuration failure output.

**Rule workflow**:
```
safeParse failure
  -> for each issue: "  - " + path.join(".") + ": " + message
  -> messages for DATABASE_URL / JWT_SECRET: static, no value echo
  -> messages for NODE_ENV / LOG_LEVEL enums: include received value
  -> single console.error block to stderr
  -> process.exit(1) with no structured log record
```

---

### Business Rule: BR-13 - NODE_ENV selects development-only runtime behavior in consumers

**Overview**:
Although the branch logic lives outside this component, the component supplies the only value that controls it. `NODE_ENV === 'development'` enables the `pino-pretty` transport (`src/shared/logger/index.ts:22-28`) and the broader Prisma log set (`src/config/database.ts:6`).

**Detailed description**:
This rule describes the business effect of the configuration value, not a validation rule. The logger constructs its transport conditionally: in development it loads `pino-pretty` with colorization and local time formatting, and in all other environments the `transport` property is `undefined`, producing default JSON output. The Prisma provider similarly selects `['warn','error']` in development and `['error']` otherwise. The server records the value in its startup log, giving operators a way to confirm which mode was selected.

No validation rule ensures that the development branches are safe in production or that the production branches are exercised in test. `NODE_ENV=test` follows the non-development path for both consumers, so tests run with JSON logs and error-only Prisma logging. `NODE_ENV` does not alter database selection, secrets, or security posture; the same `DATABASE_URL` and `JWT_SECRET` are used regardless of mode. The test harness does not assert `NODE_ENV === 'test'` before truncating tables (`tests/setup.ts:8-16`), so the value's only protective effect is indirect and absent.

The default interacts with this rule in a way that matters operationally: because an absent `NODE_ENV` becomes `development`, a production process that omits the variable will attempt to load `pino-pretty`. That package is declared in `devDependencies` (`package.json:46`) and may not be installed in a production build environment, which could turn a missing variable into a logger-construction failure or a silently different logging configuration depending on installation strategy. No rule in the component prevents this combination.

**Rule workflow**:
```
NODE_ENV value (from BR-01)
  |-- "development"
  |     -> logger transport = pino-pretty (logger/index.ts:22-28)
  |     -> prisma log = ["warn","error"] (database.ts:6)
  |-- "test" | "production"
  |     -> logger transport = undefined (JSON output)
  |     -> prisma log = ["error"]
  |-- absent -> default "development" -> development behavior
```

---

## 4. Component Structure

```
<project root>/
├── src/
│   └── config/
│       ├── env.ts                      # Component under analysis (27 lines)
│       │                               #   envSchema      : zod object, 6 fields (lines 3-10)
│       │                               #   Env            : inferred type export (line 12)
│       │                               #   loadEnv()      : safeParse + fail-fast (lines 14-25)
│       │                               #   env            : evaluated singleton export (line 27)
│       └── database.ts                 # Consumer: env.NODE_ENV -> Prisma log level (line 6)
├── .env.example                        # Companion template, 11 variables (lines 1-14)
├── .env                                # Absent in workspace; gitignored (.gitignore:4-6)
├── package.json                        # --env-file wiring in dev/start/db:seed (lines 11,13,16)
├── prisma/
│   └── schema.prisma                   # Independent consumer: DATABASE_URL, SHADOW_DATABASE_URL
├── docker-compose.yml                  # Independent consumer: MYSQL_* with fallbacks (lines 7-10)
├── vitest.config.ts                    # Test runner: does not load .env (lines 7,10)
├── tests/
│   ├── setup.ts                        # Indirect consumer via config/database.ts (line 2)
│   ├── helpers/
│   │   └── factories.ts                # Indirect consumer via config/database.ts (line 3)
│   ├── auth.test.ts                    # Indirect: login exercises JWT_SECRET/EXPIRES_IN
│   └── orders.test.ts                  # Indirect: imports config/database.ts (line 9)
├── src/server.ts                       # Consumer: env.PORT, env.NODE_ENV (lines 2,9-10)
├── src/shared/logger/index.ts          # Consumer: env.LOG_LEVEL, env.NODE_ENV (lines 2,15,20,23)
├── src/modules/auth/auth.service.ts    # Consumer: env.JWT_SECRET, env.JWT_EXPIRES_IN (lines 4,43,48-49)
└── src/middlewares/auth.middleware.ts  # Consumer: env.JWT_SECRET (lines 3,41)
```

The component boundary is tight: one implementation file, one documentation template, and no helper modules, constants file, or barrel export. The `.env.example` template is part of the component's contract even though it is not executable; it is the only artifact that tells an operator which variables the system as a whole expects, including the five that `env.ts` never sees.

The component has no barrel file. Consumers import the concrete path `config/env.js` (with the `.js` extension required by the ESM/`moduleResolution: Bundler` setup in `tsconfig.json:5`). There is no `src/config/index.ts`, so the module cannot be swapped or re-exported without editing each consumer.

### 4.1 Interface surface

| Export | Kind | Signature / Shape | Consumed by |
|--------|------|-------------------|-------------|
| `envSchema` | const (module-private) | ZodObject with 6 fields | `loadEnv()` only |
| `Env` | type | `z.infer<typeof envSchema>` = `{ NODE_ENV: 'development' \| 'test' \| 'production'; PORT: number; LOG_LEVEL: 'fatal' \| 'error' \| 'warn' \| 'info' \| 'debug' \| 'trace'; DATABASE_URL: string; JWT_SECRET: string; JWT_EXPIRES_IN: string }` | No importer (exported but unused) |
| `loadEnv` | function (module-private) | `(): Env` | Top-level initializer only |
| `env` | const | `Env` (singleton) | 5 files (see section 6) |

Neither `envSchema` nor `loadEnv` is exported. The only public surface is the `env` value plus the unused `Env` type.

### 4.2 Input surface (.env.example)

| Variable | Line | Validated by `env.ts` | Consumed by | Default in code | Default in template/Compose |
|----------|------|-----------------------|-------------|-----------------|-----------------------------|
| `NODE_ENV` | 1 | Yes (enum) | logger, database, server | `development` | `development` |
| `PORT` | 2 | Yes (positive int) | server | `3000` | `3000` |
| `LOG_LEVEL` | 3 | Yes (enum) | logger | `info` | `info` |
| `DATABASE_URL` | 5 | Yes (min 1) | Prisma client, Prisma CLI | none (required) | sample MySQL URL |
| `SHADOW_DATABASE_URL` | 6 | No | Prisma CLI | none | sample shadow URL |
| `JWT_SECRET` | 8 | Yes (min 16) | auth service, auth middleware | none (required) | sample 51-char string |
| `JWT_EXPIRES_IN` | 9 | Yes (string) | auth service | `8h` | `8h` |
| `MYSQL_ROOT_PASSWORD` | 11 | No | Docker Compose | n/a | `root_password` |
| `MYSQL_DATABASE` | 12 | No | Docker Compose | n/a | `oms` |
| `MYSQL_USER` | 13 | No | Docker Compose | n/a | `oms_user` |
| `MYSQL_PASSWORD` | 14 | No | Docker Compose | n/a | `oms_password` |

---

## 5. Dependency Analysis

### 5.1 Internal dependencies

The component imports nothing from the project. Its internal dependency graph is empty:

```
src/config/env.ts
  └── (no internal imports)
```

### 5.2 Internal dependents (compile-time and runtime)

```
src/config/env.ts (env)
  ├── src/server.ts                     (line 2)   runtime: PORT, NODE_ENV
  ├── src/config/database.ts            (line 2)   runtime: NODE_ENV
  ├── src/shared/logger/index.ts        (line 2)   runtime: LOG_LEVEL, NODE_ENV
  ├── src/modules/auth/auth.service.ts  (line 4)   runtime: JWT_SECRET, JWT_EXPIRES_IN
  └── src/middlewares/auth.middleware.ts(line 3)   runtime: JWT_SECRET
```

Transitive dependents (files that import a direct consumer and therefore load `env.ts` during evaluation):

```
src/server.ts
  ├── src/app.ts
  │     ├── src/routes/index.ts
  │     │     ├── src/modules/auth/auth.routes.ts
  │     │     ├── src/modules/users/user.routes.ts
  │     │     ├── src/modules/customers/customer.routes.ts
  │     │     ├── src/modules/products/product.routes.ts
  │     │     └── src/modules/orders/order.routes.ts
  │     ├── src/middlewares/request-logger.middleware.ts   -> shared/logger -> env
  │     ├── src/middlewares/error.middleware.ts            -> shared/logger -> env
  │     └── all controllers/services/repositories
  └── src/shared/logger/index.ts

src/config/database.ts
  ├── src/app.ts (type import of PrismaClient only)
  ├── tests/setup.ts
  ├── tests/helpers/factories.ts
  └── tests/orders.test.ts
```

The transitive fan-in is effectively the whole application: any code path that reaches the logger, the Prisma provider, or authentication triggers environment validation.

### 5.3 External dependencies

| Dependency | Version | Type | Used for | Coupling |
|------------|---------|------|----------|----------|
| `zod` | 3.23.8 (`package.json:33`) | Runtime | Schema definition, coercion, enum validation, defaults, issue formatting, type inference | Direct, compile-time and runtime |
| Node.js `process.env` | Node >= 20 (`package.json:8`) | Platform | Source of all input values | Direct, runtime |
| Node.js `process.exit` | Node >= 20 | Platform | Fail-fast termination | Direct, runtime |
| Node.js `console.error` | Node >= 20 | Platform | Failure diagnostics | Direct, runtime |
| Node.js `--env-file` | Node >= 20 | Platform / CLI | Loads `.env` into `process.env` for dev, start, seed scripts | Indirect, via `package.json:11,13,16` |
| `pino` (transitive) | 9.5.0 | Runtime | Consumer: `level` must match `LOG_LEVEL` enum; development transport | Indirect, via `src/shared/logger/index.ts` |
| `jsonwebtoken` (transitive) | 9.0.2 | Runtime | Consumer: `JWT_SECRET` and `JWT_EXPIRES_IN` | Indirect, via `auth.service.ts` and `auth.middleware.ts` |
| Prisma CLI | 5.22.0 | Build/tooling | Independent consumer of `DATABASE_URL`, `SHADOW_DATABASE_URL` | Indirect, via `prisma/schema.prisma:7-8` |
| Docker Compose | external | Infrastructure | Independent consumer of `MYSQL_*` | Indirect, via `docker-compose.yml:7-10` |

### 5.4 Dependency characteristics

- **Zero internal efferent coupling.** The component is a leaf in the internal import graph, which makes it trivially replaceable in isolation but also means all of its behavior is self-contained and untestable through seams.
- **Single external runtime dependency (`zod`).** The dependency is pinned exactly and is also used by the HTTP validation middleware and all `*.schemas.ts` files, so it is not a component-specific cost.
- **No configuration library.** There is no `dotenv` dependency; `.env` loading is delegated to the Node runtime flag. This makes loading behavior differ between scripts that use `--env-file` and scripts that do not (`npm test` at `package.json:17`).
- **Platform coupling to Node >= 20.** The `--env-file` flag is the only loading mechanism in scripts, and `process.exit` semantics are relied on for termination.
- **Implicit contract with three consumers outside the module system.** Prisma CLI and Docker Compose read the same file but not the same variables and apply their own resolution rules, so the "configuration component" is really a three-way convention.

---

## 6. Afferent and Efferent Coupling

Metrics are computed at the TypeScript symbol level for the four declarations in `src/config/env.ts`. Afferent coupling (Ca) counts distinct source files that import the symbol from outside the file; efferent coupling (Ce) counts distinct internal files the symbol depends on. External npm packages are counted separately. Method: static inspection of `import` statements across `src/` and `tests/`.

| Component (symbol) | Kind | Afferent Coupling (Ca) | Efferent Coupling (Ce) | External Dependencies | Critical | Notes |
|--------------------|------|------------------------|------------------------|-----------------------|----------|-------|
| `env` | const singleton | 5 (`server.ts`, `database.ts`, `logger/index.ts`, `auth.service.ts`, `auth.middleware.ts`) | 0 | `zod` (via module init) | High | Every consumer reads at least one field; `JWT_SECRET` is the only field with two consumers |
| `envSchema` | const ZodObject | 1 (`loadEnv` in same file) | 1 (`zod`) | `zod` | Medium | Not exported; internal single use |
| `loadEnv` | function | 1 (top-level initializer, same file) | 1 (`envSchema`) | none | Medium | Not exported; not injectable; contains `process.exit` |
| `Env` | type alias | 0 | 1 (`envSchema` type inference) | `zod` (type-level) | Low | Exported but never imported by any file |

Aggregate view at file and module level:

| Boundary | Ca | Ce | Stability (Ce / (Ca + Ce)) | Interpretation |
|----------|----|----|----------------------------|----------------|
| `src/config/env.ts` (file) | 5 | 0 internal (1 external) | 0.00 | Maximally stable; no internal reasons to change |
| `src/config` (module, incl. `database.ts`) | 4 (module-level, per architecture report) | 0 | 0.00 | Leaf infrastructure module consumed by auth, middlewares, server, shared |
| `.env.example` (artifact) | 0 importers | 0 | n/a | Documentation contract; coupled by convention to `env.ts`, Prisma CLI, Compose |

Field-level fan-in within the `env` object:

| Field | Consumers | Files |
|-------|-----------|-------|
| `NODE_ENV` | 3 | `server.ts:10`, `database.ts:6`, `logger/index.ts:20,23` |
| `PORT` | 1 | `server.ts:9-10` |
| `LOG_LEVEL` | 1 | `logger/index.ts:15` |
| `DATABASE_URL` | 0 direct (consumed by Prisma through `process.env`, not through `env`) | none |
| `JWT_SECRET` | 2 | `auth.service.ts:49`, `auth.middleware.ts:41` |
| `JWT_EXPIRES_IN` | 1 | `auth.service.ts:43,48` |

Two coupling anomalies are notable. First, `DATABASE_URL` is validated by the component but never read from the component: Prisma reads the raw environment directly (`prisma/schema.prisma:7`), so the validated value and the used value are separate reads of the same source. Second, the `Env` type has zero afferent coupling, meaning the type-level contract is not used to constrain any consumer; consumers rely on property access inference from the `env` value instead.

---

## 7. Endpoints

Not applicable. The Environment Configuration component does not expose REST, GraphQL, gRPC, CLI, or message-based endpoints. It is a startup-time library module with a single exported value. The HTTP endpoints that depend on it belong to the application modules (`src/routes/index.ts`, `src/modules/*/*.routes.ts`) and are outside this component's boundary.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol / Mechanism | Data Format | Error Handling |
|-------------|------|---------|----------------------|-------------|----------------|
| Host / container environment | Inbound platform | Source of all configuration values | `process.env` in-memory string map | `KEY=value` strings, all values strings | Missing/invalid values collected by Zod; process exits 1 |
| `.env` file | Inbound file | Local development and deployment defaults | Node `--env-file=.env` (`package.json:11,13,16`) | Line-based `KEY=value`, quoted values supported by Node | File absent or unreadable -> Node fails before `env.ts` runs; no component-level handling |
| Prisma CLI datasource | Outbound / parallel consumer | Database and shadow-database resolution for migrations | Prisma `env()` resolution, CLI auto-loads `.env` | MySQL connection URL strings | No validation in `env.ts`; Prisma CLI reports its own error; `SHADOW_DATABASE_URL` absence fails `migrate dev` |
| MySQL container configuration | Outbound / parallel consumer | Provisions the database the app connects to | Docker Compose variable interpolation | Plain strings | Compose fallbacks `${VAR:-default}` (lines 7-10); no cross-check against `DATABASE_URL` |
| Pino logger | Outbound consumer | Sets log verbosity and development transport | Direct property read `env.LOG_LEVEL`, `env.NODE_ENV` | String enum | Invalid value blocked at startup; invalid transport package surfaces at logger construction |
| `jsonwebtoken` | Outbound consumer | HMAC signing and verification key and token lifetime | Function arguments to `jwt.sign` / `jwt.verify` | String secret, `ms`-style duration string | Invalid duration is not caught at startup; throws at login time; verify failure mapped to 401 |
| Prisma client logging | Outbound consumer | Selects log verbosity by environment | Conditional on `env.NODE_ENV === 'development'` | String literal comparison | No error path; value already validated |
| HTTP listener | Outbound consumer | Binds the server port | `app.listen(env.PORT)` | Number | Out-of-range port throws `RangeError`, caught by bootstrap handler (`server.ts:24-27`) |
| Test harness | Inbound/outbound consumer | Provides DB connection and JWT settings for integration tests | Inherited `process.env`; no `.env` loading by Vitest | Strings | Invalid config aborts the whole test process during setup; no test-specific guard |

### 8.1 Configuration channels diagram

```
                         +---------------------------+
   .env file  ---------> | Node --env-file (dev,     | ---> process.env ---+
   (optional)            | start, db:seed)           |                     |
                         +---------------------------+                     |
                                                                           v
                         +---------------------------+          +------------------------+
   .env file  ---------> | Prisma CLI auto-load      |          | src/config/env.ts      |
                         | (DATABASE_URL,            |          | zod safeParse          |
                         |  SHADOW_DATABASE_URL)     |          | 6 vars, fail-fast      |
                         +---------------------------+          +-----------+------------+
                                                                           |
                                                                           v
                         +---------------------------+          +------------------------+
   .env file  ---------> | Docker Compose            |          | exported env singleton |
                         | interpolation (MYSQL_*)   |          +-----------+------------+
                         +---------------------------+                      |
                                                                            v
                                              server / prisma provider / logger / auth
```

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Schema-first validation | Single Zod object defines types, constraints, defaults, and coercion | `src/config/env.ts:3-10` | One declarative source of truth for the configuration contract |
| Type inference from schema | `export type Env = z.infer<typeof envSchema>` | `src/config/env.ts:12` | Keeps the TypeScript type and runtime validator from drifting |
| Fail-fast / fail-silent-at-startup | `safeParse` -> aggregate issues -> `console.error` -> `process.exit(1)` | `src/config/env.ts:15-23` | Prevents the process from serving traffic with incomplete or invalid configuration |
| Error aggregation | Iterate `parsed.error.issues`, format as a bullet list | `src/config/env.ts:17-19` | Reports every misconfiguration in one run instead of one per restart |
| Boundary coercion | `z.coerce.number()` for the only numeric variable | `src/config/env.ts:5` | Bridges the string-only environment model to a numeric runtime type |
| Declarative defaults | `.default()` on four fields | `src/config/env.ts:4-6,9` | Allows local development with a minimal environment |
| Module singleton | Top-level `export const env = loadEnv()` | `src/config/env.ts:27` | Guarantees one validated configuration instance per process; leverages ESM evaluation cache |
| Twelve-Factor Config | All configuration supplied through environment variables; no config files parsed at runtime | Whole component; `package.json:11,13` | Deployment-portable configuration without code changes |
| Guard clause / early exit | Validation precedes every consumer's initialization | `src/config/env.ts:27` -> `src/server.ts:2` | Guarantees consumers never observe invalid values |
| Static service locator | Consumers import the concrete singleton rather than receiving configuration by injection | `src/server.ts:2`, `database.ts:2`, `logger/index.ts:2` | Simplicity and zero wiring; trades away testability and substitutability |
| Separation of concerns at the boundary | Runtime schema validation kept separate from CLI (Prisma) and infrastructure (Compose) configuration | `env.ts` vs `prisma/schema.prisma:7-8` vs `docker-compose.yml:7-10` | Each consumer owns its own concerns, at the cost of a shared-file convention |

### 9.1 Architectural decisions evidenced by the code

1. **Validation belongs at the process boundary, not at each use site.** Consumers read `env.PORT` without re-checking; correctness is assumed because module initialization would have terminated otherwise.
2. **Configuration is treated as infrastructure, not domain.** The component lives under `src/config` alongside the Prisma provider and has no domain types or business vocabulary.
3. **Permissive defaults over mandatory production values.** Four fields default; only the two with security/connectivity consequences are mandatory. `NODE_ENV` defaults to the least restrictive mode.
4. **No runtime reconfiguration.** There is no watcher, reload, or setter; the configuration is a startup snapshot.
5. **No dependency injection for configuration.** The static import makes the value globally reachable and removes the need to thread it through constructors, at the cost of a hidden global dependency.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Test coverage | No test file exists for `env.ts`; `loadEnv` and `envSchema` are private and the module exits the process on invalid input | Validation branches (13 rules) have 0 direct coverage; regressions are invisible until startup or runtime |
| High | Test safety | `NODE_ENV` is validated but never enforced; `tests/setup.ts:8-16` deletes all rows in the database pointed to by `DATABASE_URL`, with no check that it is a test database | Running tests against a non-test environment destroys production or shared data |
| High | Configuration completeness | `SHADOW_DATABASE_URL` is documented (`.env.example:6`) and required by `prisma/schema.prisma:8` but validated nowhere | `prisma migrate dev` fails with a Prisma-level error; the failure is not caught by the startup gate |
| High | Environment default | `NODE_ENV` defaults to `development` (`src/config/env.ts:4`) | A production deployment that omits the variable runs with development logging and attempts to load `pino-pretty`, which is a devDependency (`package.json:46`) |
| Medium | Secret strength | `JWT_SECRET` minimum is 16 characters with no entropy, character-class, or placeholder check (`src/config/env.ts:8`) | Low-entropy keys and the published sample secret (`.env.example:8`) pass validation; HMAC-SHA256 strength depends on entropy, not length |
| Medium | Value validation | `DATABASE_URL` accepts any non-empty string (`src/config/env.ts:7`); no URL, scheme, or host validation | Malformed connection strings pass startup and fail at first Prisma connection with a non-configuration error |
| Medium | Value validation | `JWT_EXPIRES_IN` accepts any string (`src/config/env.ts:9`), including `""`; the cast at `auth.service.ts:48` suppresses the library's type constraint | Invalid or empty duration passes startup and throws at login, turning a startup error into a runtime 500 |
| Medium | Fail-fast diagnostics | `console.error` + synchronous `process.exit(1)` at module evaluation (`src/config/env.ts:20-22`) | Failure bypasses the structured logger and produces no JSON record, correlation id, or timestamp; untestable in-process |
| Medium | Configuration coherence | No consistency checks between `DATABASE_URL`, `SHADOW_DATABASE_URL`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD` | Drift between Compose provisioning and application connection settings is only discovered at connection time |
| Medium | Secrets in template | `.env.example:5-14` commits usable passwords and a sample JWT secret; Compose provides the same values as fallbacks (`docker-compose.yml:7-10`) | Credential reuse and copy-paste of defaults into real environments; MySQL port also published on the host (`docker-compose.yml:12`) |
| Low | Numeric bounds | `PORT` has no upper bound (`src/config/env.ts:5`); port 0 is rejected although Node treats it as an ephemeral port | Out-of-range values fail later at `listen()` (`src/server.ts:9`) rather than at the gate |
| Low | Mutability | `env` is not frozen (`src/config/env.ts:27`) | Any consumer can reassign fields at runtime; no code currently does, but the guarantee is by convention only |
| Low | Type surface | `Env` is exported but imported by no file (`src/config/env.ts:12`) | The type contract provides no enforcement at consumer boundaries; dead export surface |
| Low | Log level vocabulary | `LOG_LEVEL` enum excludes Pino's `silent` (`src/config/env.ts:6`) | Fully silent operation is not representable through configuration |
| Low | Loading consistency | `npm test` does not use `--env-file` (`package.json:17`) while dev/start/seed do | Test configuration depends on the caller's shell; behavior diverges from runtime loading |
| Low | Lint noise | `// eslint-disable-next-line no-console` at `src/config/env.ts:20` is redundant because `console.error` is allowed (`.eslintrc.json:28`) | Misleading suppression comment; suggests a stricter rule than actually configured |
| Low | Documentation gap | `.env.example` has no comments and no `.env` bootstrap step; no `.env` file exists in the workspace | New operators must infer required values and the absent-versus-empty default semantics |
| Low | Cohesion of NODE_ENV checks | Environment string comparisons are duplicated in `database.ts:6` and `logger/index.ts:23` | No central predicate; adding a fourth environment requires touching each comparison |

### 10.1 Security assessment

- **Secret handling:** `JWT_SECRET` and `DATABASE_URL` are never logged or echoed by the component; failure messages for those fields are static. Enum failure messages echo the received `NODE_ENV`/`LOG_LEVEL` value, which is non-sensitive.
- **Secret validation:** only presence and length are enforced. There is no rotation support, no key identifier, no versioning, and no check against published sample values.
- **Secret storage:** values live in `.env` (gitignored at `.gitignore:4-6`) and in `process.env`; there is no secret manager integration.
- **Fail-closed behavior:** the component is fail-closed for invalid configuration (process exits), which is the correct posture, but it is fail-open for missing `NODE_ENV` (defaults to development).
- **Injection surface:** values are used as library arguments, never interpolated into shell commands, SQL, or templates by this component. `PORT` is numeric after coercion, reducing injection surface.
- **Error message hygiene:** paths and validator messages only; no stack traces, no environment dump.
- **Least privilege:** the component validates credentials but cannot constrain the privileges of the MySQL user named in `DATABASE_URL`; Compose creates `oms_user` with the database's default grants.

### 10.2 Resilience assessment

| Aspect | Implementation | Observation |
|--------|----------------|-------------|
| Retry | None | Configuration is read once; no re-read on failure |
| Fallback | Defaults for 4 of 6 fields | Fallback is permissive, not environment-aware |
| Circuit breaker | None | Not applicable to startup configuration |
| Timeout | None | Reading `process.env` is synchronous and local |
| Degraded mode | None | Any invalid required field terminates the process |
| Observability | `console.error` only | No metrics, no structured event, no health signal beyond exit code |
| Recovery | Operator restarts the process | No hot reload or dynamic reconfiguration |
| Idempotence | Module-level singleton | Re-import within a process cannot re-run validation |

---

## 11. Test Coverage Analysis

### 11.1 Test inventory relevant to this component

| Test file | Relationship to Environment Configuration | Evidence |
|-----------|-------------------------------------------|----------|
| `tests/setup.ts` | Indirect: imports `prisma` from `src/config/database.js`, which imports `env`; triggers validation during setup | `tests/setup.ts:2` |
| `tests/helpers/factories.ts` | Indirect: imports the same `prisma` singleton and `buildApp` | `tests/helpers/factories.ts:3-5` |
| `tests/auth.test.ts` | Indirect: login/register flows exercise `JWT_SECRET` (signing) and `JWT_EXPIRES_IN` (signing/response); token verification exercises `JWT_SECRET` | `tests/auth.test.ts:41-52,65-77` |
| `tests/orders.test.ts` | Indirect: imports `prisma` directly; authenticated requests exercise JWT verification | `tests/orders.test.ts:9` |
| `vitest.config.ts` | Harness: includes `tests/**/*.test.ts` and loads `tests/setup.ts`; no env file loading | `vitest.config.ts:7,10` |

No file matching `env.test.ts`, `config.test.ts`, or `environment.test.ts` exists anywhere in the repository. A search for `process.env` and `NODE_ENV` in `tests/` returns no matches, confirming that no test manipulates or asserts environment configuration.

### 11.2 Coverage assessment

Coverage was not measured. There is no coverage script in `package.json:10-21` and no coverage provider package in `devDependencies` (`package.json:35-53`), so no numeric line or branch coverage exists for the project or for this component. The percentages below express the analysis assessment of exercised behavior, not a tool report.

| Component artifact | Unit Tests | Integration Tests | Coverage Assessment | Test Quality |
|--------------------|------------|-------------------|---------------------|--------------|
| `envSchema` field constraints (BR-01 to BR-06) | 0 | 0 direct; only the happy path is implicitly exercised | 0% of validation branches; happy path ~100% incidental | No negative tests for any field; enum, coercion, min-length, and default behaviors unverified |
| `loadEnv` success path | 0 | Implicitly exercised by every suite | Executed, not asserted | No assertion on returned shape, types, or values |
| `loadEnv` failure path (BR-08) | 0 | 0 | 0% | Not testable in-process because the module calls `process.exit`; would require a child-process harness that does not exist |
| Defaults (BR-01, BR-02, BR-03, BR-06) | 0 | 0 | 0% | No test runs with the variables unset; CI behavior depends on ambient environment |
| Absent vs empty semantics (BR-07) | 0 | 0 | 0% | No test distinguishes unset from empty |
| Unknown-key stripping (BR-09) | 0 | 0 | 0% | Not asserted |
| Singleton semantics (BR-10) | 0 | 0 | 0% | Implicitly relied upon, never verified |
| `.env.example` parity (BR-11) | 0 | 0 | 0% | No test or lint checks documentation/validation drift |
| Failure message format (BR-12) | 0 | 0 | 0% | Not asserted |
| Environment-driven branches (BR-13) | 0 | 0 | Partial: `NODE_ENV=test` path is what the suites run under, but nothing asserts it | Development branch (`pino-pretty`, warn logs) is untested |
| `JWT_SECRET` wiring | 0 | 5 auth tests rely on signing/verifying | Indirect only | Tests would fail at setup if the secret were missing, but there is no assertion on minimum length or behavior at 15 characters |
| `JWT_EXPIRES_IN` wiring | 0 | 1 assertion on `tokenType`, none on `expiresIn` value | Indirect only | `tests/auth.test.ts:49-50` checks that a token exists and `tokenType` is `Bearer`; it does not assert `expiresIn === '8h'` |
| `DATABASE_URL` wiring | 0 | All suites via `$connect` and truncation | Indirect only | Failure mode is a connection error, not a configuration assertion |
| `PORT` wiring | 0 | 0 | 0% | Tests use `buildApp` with supertest and never call `listen`, so `PORT` is never read in tests |
| `LOG_LEVEL` wiring | 0 | 0 | 0% | Logger is constructed but level behavior is not asserted |

### 11.3 Testability observations

- **The module has no injection seam.** `loadEnv` is private, takes no argument, reads `process.env` directly, and returns or exits. There is no exported factory such as `parseEnv(source)` that a test could call with a fixture object.
- **Failure paths are process-fatal.** Testing a missing `DATABASE_URL` or a 15-character `JWT_SECRET` in-process would terminate the Vitest worker because the module calls `process.exit(1)` at evaluation time. Any test would need a subprocess or a child-process runner, neither of which exists.
- **Module caching prevents revalidation.** Even if a test mutated `process.env`, a second import of `src/config/env.ts` in the same worker returns the cached binding. The suite runs with `singleFork: true` and `fileParallelism: false` (`vitest.config.ts:11-17`), so all tests share one process and one `env` instance.
- **Ambient environment dependency.** `vitest.config.ts` does not load `.env`, and `package.json:17` runs `vitest run` without `--env-file`. Tests therefore require `DATABASE_URL` and `JWT_SECRET` to be exported by the caller. There is no `.env.test`, no `setupFiles` that assigns defaults, and no `.env` file in the workspace, so a fresh clone cannot run `npm test` without manual environment preparation.
- **Indirect coverage masks configuration regressions.** The integration suites verify that a valid configuration results in a working application, but they cannot detect a relaxed constraint (for example, lowering `JWT_SECRET` minimum to 1) because no test supplies an invalid value.
- **The test harness has the highest configuration risk.** Because `tests/setup.ts:8-16` truncates seven tables and never checks `NODE_ENV` or the database name, the absence of an environment guard is the most consequential untested behavior associated with this component.

### 11.4 Coverage by business rule

| Rule | Direct unit coverage | Indirect integration coverage | Risk if regressed |
|------|----------------------|-------------------------------|-------------------|
| BR-01 `NODE_ENV` enum | None | Happy path only (`test`) | Wrong mode accepted; dev logging in production |
| BR-02 `PORT` coercion | None | None (`listen` never called) | Invalid port reaches Node `listen` |
| BR-03 `LOG_LEVEL` enum | None | None | Invalid level reaches Pino |
| BR-04 `DATABASE_URL` required | None | Setup fails if absent, not asserted | Late Prisma failure |
| BR-05 `JWT_SECRET` minimum | None | Signing/verifying depends on presence | Weak secret accepted |
| BR-06 `JWT_EXPIRES_IN` | None | Token issued, duration not asserted | Runtime signing failure at login |
| BR-07 absent vs empty | None | None | Defaults silently not applied |
| BR-08 fail-fast aggregation | None | None | Partial startup or unreadable diagnostics |
| BR-09 unknown-key stripping | None | None | Accidental config exposure or type drift |
| BR-10 singleton | None | Relied upon implicitly | Inconsistent config within a process |
| BR-11 surface parity | None | None | Missing `SHADOW_DATABASE_URL`, credential drift |
| BR-12 diagnostics hygiene | None | None | Misleading or leaking errors |
| BR-13 environment branches | None | Test branch only, not asserted | Untested development behavior |

---

## 12. Report Metadata

| Item | Value |
|------|-------|
| Component analyzed | Environment Configuration |
| Primary files | `src/config/env.ts` (27 lines), `.env.example` (14 lines) |
| Files inspected | `src/config/env.ts`, `.env.example`, `package.json`, `tsconfig.json`, `tsconfig.build.json`, `.eslintrc.json`, `.gitignore`, `vitest.config.ts`, `docker-compose.yml`, `prisma/schema.prisma`, `prisma/seed.ts`, `src/server.ts`, `src/app.ts`, `src/config/database.ts`, `src/shared/logger/index.ts`, `src/modules/auth/auth.service.ts`, `src/middlewares/auth.middleware.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `tests/auth.test.ts`, `tests/orders.test.ts`, `docs/agents/MANIFEST.md`, `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md` |
| Excluded per parameters | `node_modules`, `.git`, `dist`, `coverage` |
| Lines of production code in component | 27 (`src/config/env.ts`) |
| Business rules documented | 13 |
| External runtime dependencies | 1 (`zod` 3.23.8) |
| Internal dependents | 5 direct files, application-wide transitive |
| Endpoints exposed | 0 |
| Direct tests for component | 0 |
| Ambiguities encountered | None that block analysis. Two behaviors are implicit rather than documented: Zod's absent-versus-empty default semantics (BR-07) and Zod's default stripping of unknown keys (BR-09). Both are confirmed by the library's documented v3 behavior and are marked as implicit rules in this report. |
