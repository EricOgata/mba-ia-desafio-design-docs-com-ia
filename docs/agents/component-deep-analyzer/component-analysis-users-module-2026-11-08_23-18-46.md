# Component Deep Analysis Report - Users Module

**Component:** Users Module
**Primary location:** `src/modules/users/`
**Analysis scope:** `src/modules/users/` plus directly coupled contracts in `src/modules/auth/`, `src/middlewares/`, `src/shared/errors/`, `src/app.ts`, `src/routes/index.ts`, `prisma/schema.prisma`, `prisma/migrations/`, `prisma/seed.ts`, and the project test suite (`tests/`).
**Excluded paths:** `node_modules`, `.git`, `dist`, `coverage`
**Analyzed revision:** `e7f6311` (working tree)
**Analysis date:** 2026-11-08 23:18:46
**Report confidence:** High for implemented behavior. Requirements documentation is not available: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` and `docs/TRACKER.md` are placeholder templates. Business rules below are reverse-engineered from source code; confidence is stated per rule where inference is involved.

---

## 1. Executive Summary

The Users Module is one of the five feature modules of the Order Management System (OMS) REST API. It owns the persistence and public representation of application users, password hashing, email uniqueness enforcement and user lookup. It is a small component - five files and 119 lines - but it is the data owner for the identity records consumed by the Auth Module, the authentication middleware and the Orders Module audit columns.

The module follows the project's standard layered convention:

- `user.routes.ts` - HTTP binding, authentication, role authorization and request validation.
- `user.controller.ts` - transport adaptation from Express to the service layer.
- `user.service.ts` - business rules: uniqueness, password hashing, public projection, not-found handling.
- `user.repository.ts` - Prisma data access for `findById`, `findByEmail` and `create`.
- `user.schemas.ts` - Zod contracts and the inferred `CreateUserInput` type.

Only one endpoint is owned by the module: `GET /api/v1/users/:id`, restricted to `ADMIN`. User creation is not exposed by this module's router; it is delegated from the Auth Module's public `POST /api/v1/auth/register` route into `UserService.createUser`. The Users Module therefore has two distinct entry paths: a direct admin-only read endpoint and an indirect, public, unauthenticated write path through Auth.

### Key findings

| Severity | Finding | Evidence |
|----------|---------|----------|
| Critical | The public registration path delegates to `UserService.createUser`, which persists the caller-supplied `role` without authorization. Any anonymous caller can create an `ADMIN`. The Users Module is the persistence point of this privilege-escalation path. | `src/modules/users/user.service.ts:21-33`, `src/modules/auth/auth.schemas.ts:7`, `src/modules/auth/auth.routes.ts:10` |
| High | Email uniqueness is enforced by a check-then-insert sequence without a transaction or a domain-specific unique-constraint handler. Concurrent registrations can both pass the pre-check; the database unique index then produces a generic `CONFLICT` instead of `EMAIL_ALREADY_USED`. | `src/modules/users/user.service.ts:22-32`, `src/modules/users/user.repository.ts:10-16`, `src/middlewares/error.middleware.ts:37-47` |
| High | Roles are trusted from the JWT claim for every protected request. The Users Module does not expose a role revalidation or user-state check, so a token keeps its original role for up to 8 hours after a change. | `src/middlewares/auth.middleware.ts:40-46`, `src/modules/users/user.routes.ts:14-15` |
| Medium | The creation contract is duplicated between `src/modules/auth/auth.schemas.ts:3-8` and `src/modules/users/user.schemas.ts:7-12`. The two definitions are currently identical but can drift independently, and `AuthService.register` relies on structural typing compatibility. | `src/modules/auth/auth.service.ts:27-29`, `src/modules/users/user.schemas.ts:7-14` |
| Medium | The UUID path schema is declared twice: inline in `user.routes.ts:7` and exported as `userIdParamSchema` in `user.schemas.ts:3-5`. The exported schema is unused, which is dead code and duplicated validation logic. | `src/modules/users/user.routes.ts:7,16`, `src/modules/users/user.schemas.ts:3-5` |
| Medium | Email identity is not normalized in application code. Uniqueness and lookup behavior depend on MySQL's `utf8mb4_unicode_ci` collation; a collation change would silently change identity semantics. | `prisma/migrations/20260519182739_init/migration.sql:11-13`, `src/modules/users/user.service.ts:22,27` |
| Medium | Password length is validated in characters (`max(72)`) while bcrypt truncates at 72 bytes. Multi-byte passwords can be silently truncated at the hashing boundary. | `src/modules/users/user.schemas.ts:9`, `src/modules/users/user.service.ts:16,26` |
| Low | The Users Module has no dedicated tests and its only owned endpoint (`GET /users/:id`) has no test coverage for success, 404, 401 or 403 paths. | `tests/auth.test.ts`, `tests/orders.test.ts` (no `/users` calls) |
| Low | `PublicUser.role` and the repository `create` parameter redeclare the role union as string literals instead of using the generated Prisma `UserRole` enum, creating a drift risk if the enum changes. | `src/modules/users/user.service.ts:7-14`, `src/modules/users/user.repository.ts:14` |

### Role in the system

- Inbound (owned): `GET /api/v1/users/:id`, mounted by `src/routes/index.ts:25` from `buildUserRouter` (`src/modules/users/user.routes.ts:9-20`).
- Inbound (delegated): `POST /api/v1/auth/register` calls `UserService.createUser` through `AuthService.register` (`src/modules/auth/auth.service.ts:27-29`); `GET /api/v1/auth/me` calls `UserService.getById` (`src/modules/auth/auth.controller.ts:30-38`).
- Outbound: `UserRepository` uses the injected `PrismaClient` for the `users` table (`src/modules/users/user.repository.ts:3-16`); errors use the shared taxonomy (`src/shared/errors/index.ts`); password hashing uses bcrypt (`src/modules/users/user.service.ts:1,26`).
- Data dependency from other modules: `orders.createdById` and `order_status_history.changedById` are foreign keys to `users.id` with `ON DELETE RESTRICT` (`prisma/migrations/20260519182739_init/migration.sql:112-125`). The Orders Module writes `req.user.id` into both columns (`src/modules/orders/order.controller.ts:30-41`, `src/modules/orders/order.service.ts:104,110,164`).

### Assumptions

- No written specification exists for user management. All rules are extracted from code.
- Coverage percentages are not reported as measured values because no coverage provider or coverage script is configured. Section 11 provides a scenario-level assessment instead.
- The Users Module is analyzed as the five files under `src/modules/users/`; Auth Module routes are treated as indirect entry points, not as part of the component boundary.

---

## 2. Data Flow Analysis

### 2.1 Admin user lookup (`GET /api/v1/users/:id`)

```
1. Client sends GET /api/v1/users/{uuid} with Authorization: Bearer <JWT>
2. src/app.ts:59      express.json parses body (1 MB limit; body unused for GET)
3. src/app.ts:60      requestLogger assigns/propagates X-Request-Id and records userId on finish
4. src/app.ts:67      app mounts /api/v1 -> src/routes/index.ts:25 -> buildUserRouter
5. src/modules/users/user.routes.ts:14  authenticate verifies JWT and sets req.user
6. src/modules/users/user.routes.ts:15  requireRole('ADMIN') checks req.user.role
7. src/modules/users/user.routes.ts:16  validate({ params: idParamSchema }) enforces UUID
8. src/modules/users/user.controller.ts:7-14  reads req.params.id and calls UserService.getById
9. src/modules/users/user.service.ts:36-40    UserRepository.findById
10. src/modules/users/user.repository.ts:6-8  prisma.user.findUnique({ where: { id } })
11. MySQL users table (PRIMARY KEY lookup)
12. UserService.toPublic strips passwordHash (user.service.ts:42-51)
13. Controller returns 200 with PublicUser JSON
14. On missing row: NotFoundError('User') -> error middleware -> 404 NOT_FOUND
15. On non-ADMIN: ForbiddenError -> 403 FORBIDDEN; on bad token: 401 UNAUTHORIZED
```

### 2.2 User creation through the Auth Module (`POST /api/v1/auth/register`)

```
1. Anonymous client sends JSON { email, password, name, role? }
2. requestLogger -> /api/v1 router -> auth router
3. src/modules/auth/auth.routes.ts:10  validate({ body: registerSchema })
4. src/modules/auth/auth.controller.ts:12-19  AuthController.register
5. src/modules/auth/auth.service.ts:27-29     AuthService.register -> UserService.createUser(input)
6. src/modules/users/user.service.ts:22-25    UserRepository.findByEmail (pre-check)
7. Existing row -> ConflictError('Email already registered', 'EMAIL_ALREADY_USED')
8. src/modules/users/user.service.ts:26       bcrypt.hash(password, 10)
9. src/modules/users/user.service.ts:27-32    UserRepository.create
10. src/modules/users/user.repository.ts:14-16 prisma.user.create
11. MySQL users table INSERT (unique index on email, UUID default id, timestamps)
12. UserService.toPublic (no passwordHash)
13. 201 Created with public user JSON
14. Race path: DB P2002 -> error middleware -> 409 CONFLICT (generic code)
```

### 2.3 Identity lookup through the Auth Module (`GET /api/v1/auth/me`)

```
1. Authenticated request with Bearer JWT
2. authenticate sets req.user = { id, email, role } from token claims
3. AuthController.me checks req.user exists (401 if missing)
4. UserService.getById(req.user.id)
5. UserRepository.findById -> Prisma -> MySQL
6. Public projection returned with 200
7. Deleted user with still-valid token -> 404 NOT_FOUND (not 401)
```

### 2.4 Cross-module identity reference (Orders)

```
1. OrderController.create/changeStatus read req.user.id (src/modules/orders/order.controller.ts:31,41)
2. OrderService writes createdById/changedById inside transactions
3. MySQL foreign key users.id enforces existence with ON DELETE RESTRICT
4. No Users Module API call occurs; coupling is at the database schema level
```

### 2.5 Error propagation

```
Any module error (ConflictError, NotFoundError, AppError)
  -> controller catch -> next(err)
  -> src/middlewares/error.middleware.ts:14-65
  -> AppError branch: { error: { code, message, details? } } with statusCode
Unexpected Prisma error other than P2002/P2025
  -> logs with requestId/method/path -> 500 INTERNAL_SERVER_ERROR
```

---

## 3. Business Rules & Logic

### Overview of the business rules

| ID | Rule Type | Rule Description | Location |
|----|-----------|------------------|----------|
| BR-01 | Data integrity | Email must be unique across users; pre-check returns 409 `EMAIL_ALREADY_USED` | `src/modules/users/user.service.ts:22-25` |
| BR-02 | Security | Passwords are hashed with bcrypt cost factor 10 before persistence; plaintext is never stored | `src/modules/users/user.service.ts:16,26-32` |
| BR-03 | Data projection | User responses expose only `id`, `email`, `name`, `role`, `createdAt`, `updatedAt`; credentials are never returned | `src/modules/users/user.service.ts:7-14,42-51` |
| BR-04 | Validation | Role is restricted to `ADMIN` or `OPERATOR` and defaults to `OPERATOR` | `src/modules/users/user.schemas.ts:11`, `prisma/schema.prisma:11-14,30` |
| BR-05 | Authorization | User creation trusts the caller-supplied role; the module performs no role authorization or elevation check | `src/modules/users/user.service.ts:27-32`, `src/modules/auth/auth.routes.ts:10` |
| BR-06 | Validation | Creation payload must satisfy email format/max 255, password 8-72, name 2-150, role enum | `src/modules/users/user.schemas.ts:7-12`, `src/modules/auth/auth.schemas.ts:3-8` |
| BR-07 | Validation | The user id path parameter must be a UUID | `src/modules/users/user.routes.ts:7,16`, `src/modules/users/user.schemas.ts:3-5` |
| BR-08 | Error handling | An unknown user id produces 404 `NOT_FOUND` | `src/modules/users/user.service.ts:36-40` |
| BR-09 | Security | User lookup requires a valid Bearer JWT | `src/modules/users/user.routes.ts:14`, `src/middlewares/auth.middleware.ts:27-47` |
| BR-10 | Authorization | User lookup requires the `ADMIN` role | `src/modules/users/user.routes.ts:15`, `src/middlewares/auth.middleware.ts:49-61` |
| BR-11 | Data integrity | `id`, `createdAt` and `updatedAt` are server/database managed and not accepted from input | `prisma/schema.prisma:26-32`, `src/modules/users/user.repository.ts:14-16` |
| BR-12 | Lifecycle constraint | The module exposes no update, delete, list, password-change or deactivation capability | `src/modules/users/user.routes.ts:9-20`, `src/modules/users/user.repository.ts:3-16` |
| BR-13 | Identity semantics | Email comparison and uniqueness are delegated to MySQL `utf8mb4_unicode_ci`; no normalization in code | `prisma/migrations/20260519182739_init/migration.sql:11-13`, `src/modules/users/user.service.ts:22` |
| BR-14 | Referential integrity | Users referenced by orders or status history cannot be deleted (`ON DELETE RESTRICT`) | `prisma/migrations/20260519182739_init/migration.sql:112-125` |

### Detailed breakdown of the business rules

---

### Business Rule: BR-01 Email uniqueness

**Overview**:
A user email address identifies a single account. `UserService.createUser` checks for an existing user before inserting.

**Detailed description**:
`UserService.createUser` calls `this.users.findByEmail(input.email)` before any hashing or insertion (`src/modules/users/user.service.ts:22`). If a row is found, the service throws `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` (`user.service.ts:23-25`), which the centralized error middleware serializes as HTTP 409 with `error.code = "EMAIL_ALREADY_USED"`. This gives callers a stable, domain-specific error code for the common duplicate case. The same check is reached from `POST /api/v1/auth/register` because `AuthService.register` delegates directly to `createUser` (`src/modules/auth/auth.service.ts:27-29`).

The database enforces the invariant independently through `email String @unique` in `prisma/schema.prisma:27` and the unique index `users_email_key` in `prisma/migrations/20260519182739_init/migration.sql:11`. This second line of defense matters because the service-level check is a read-then-write sequence without a transaction, row lock or `upsert`. Two concurrent requests for the same email can both observe no row and both attempt insertion; exactly one insert succeeds and the other raises Prisma `P2002`.

The failure mode of that race is inconsistent with the documented domain error. `src/middlewares/error.middleware.ts:37-47` maps `P2002` to HTTP 409 with `error.code = "CONFLICT"` and the message `Unique constraint violation on: email`, not `EMAIL_ALREADY_USED`. Clients that branch on `EMAIL_ALREADY_USED` may not recognize the race outcome. There is no retry, no transaction, no advisory lock and no `upsert`-based implementation in the module.

**Rule workflow**:
```
createUser(input)
  -> findByEmail(input.email)
     -> row exists: throw ConflictError(409, EMAIL_ALREADY_USED)
     -> row absent: bcrypt.hash -> repository.create
        -> success: return PublicUser
        -> concurrent duplicate: Prisma P2002 -> error middleware -> 409 CONFLICT
```

**Confidence:** High (explicit code). The race behavior is inferred from the absence of transaction/locking and is confirmed by the Prisma error mapping.

---

### Business Rule: BR-02 Password hashing before persistence

**Overview**:
User passwords are never persisted or returned in plaintext. A bcrypt hash with cost factor 10 is stored in `passwordHash`.

**Detailed description**:
`BCRYPT_ROUNDS = 10` is a module-level constant (`src/modules/users/user.service.ts:16`). `createUser` awaits `bcrypt.hash(input.password, BCRYPT_ROUNDS)` (`user.service.ts:26`) and passes only the resulting `passwordHash` to `UserRepository.create` (`user.service.ts:27-32`). The repository signature requires `passwordHash` and has no `password` field (`src/modules/users/user.repository.ts:14-16`), so plaintext cannot reach the Prisma `create` call through the typed repository API.

The hash is deliberately excluded from the public projection. `UserService.toPublic` builds a new object with an explicit field whitelist and does not copy `passwordHash` (`user.service.ts:42-51`). The same projection is used by registration (`user.service.ts:33`) and lookup (`user.service.ts:39`), and therefore by all response paths in the Users and Auth modules. The integration test at `tests/auth.test.ts:14-16` asserts that neither `passwordHash` nor `password` appears in the registration response.

The cost factor is fixed in code rather than configured. Password verification is performed outside the Users Module by `AuthService.login` using `bcrypt.compare(input.password, user.passwordHash)` (`src/modules/auth/auth.service.ts:36`). The Users Module therefore owns hash creation but not hash verification; the two sides agree only through the stored string format. There is no password rehash-on-login path and no configurable work factor.

**Rule workflow**:
```
createUser(input)
  -> plaintext input.password exists only in request memory
  -> bcrypt.hash(password, 10)
  -> repository.create({ passwordHash, ... })
  -> toPublic() copies no passwordHash
  -> response contains no credential fields
```

**Confidence:** High (explicit code and test assertion).

---

### Business Rule: BR-03 Public user projection

**Overview**:
Every user object leaving the Users Module is a `PublicUser` containing only non-sensitive fields.

**Detailed description**:
The `PublicUser` type declares exactly `id`, `email`, `name`, `role`, `createdAt` and `updatedAt` (`src/modules/users/user.service.ts:7-14`). The static `toPublic` method constructs a new literal with those six fields (`user.service.ts:42-51`). Because it returns a fresh object rather than the Prisma entity, adding new sensitive columns to the `User` model later will not automatically leak them through existing endpoints. This is a whitelist projection rather than blacklist deletion.

All response-producing paths in the component use this projection: `createUser` returns `UserService.toPublic(created)` (`user.service.ts:33`) and `getById` returns `UserService.toPublic(user)` (`user.service.ts:39`). The Auth Module reuses the same static method in `AuthService.login` (`src/modules/auth/auth.service.ts:42`) and relies on `UserService.getById` for `/auth/me` (`src/modules/auth/auth.controller.ts:33`). The projection is therefore a cross-module response contract, not just an internal helper.

The projection converts Prisma `Date` values for `createdAt` and `updatedAt` into JSON ISO strings at serialization time by Express. It does not expose relations, even though the `User` model has `createdOrders` and `statusHistoryChanges` relations (`prisma/schema.prisma:34-35`). It also does not expose any `active`, `deletedAt` or last-login field, because none exists in the schema.

**Rule workflow**:
```
Prisma User entity
  -> UserService.toPublic(user)
     -> pick id, email, name, role, createdAt, updatedAt
     -> drop passwordHash and all relations
  -> controller res.status(200|201).json(publicUser)
```

**Confidence:** High (explicit whitelist; validated indirectly by `tests/auth.test.ts:14-16`).

---

### Business Rule: BR-04 Role domain and default

**Overview**:
A user role must be one of `ADMIN` or `OPERATOR`. When omitted on creation, it becomes `OPERATOR`.

**Detailed description**:
The role domain exists in three places. Zod validates `role: z.enum(['ADMIN', 'OPERATOR']).default('OPERATOR')` in the Users Module (`src/modules/users/user.schemas.ts:11`) and again in the Auth Module (`src/modules/auth/auth.schemas.ts:7`). Prisma defines `enum UserRole { ADMIN OPERATOR }` with `role UserRole @default(OPERATOR)` (`prisma/schema.prisma:11-14,30`), and the migration materializes an `ENUM('ADMIN','OPERATOR')` column with a database default (`prisma/migrations/20260519182739_init/migration.sql:7`). A value outside the set is rejected before persistence with HTTP 400 `VALIDATION_ERROR`.

Because the Zod default applies during parsing, an omitted role is materialized as `OPERATOR` in `req.body` before the controller runs (`src/middlewares/validate.middleware.ts:14-16`). The service receives a fully populated input and passes it to the repository unchanged. If the service is called directly with a malformed value, TypeScript narrows the type but does not provide runtime protection; the database enum is the final guard and would reject an invalid value with a Prisma error.

The default is security-relevant because it defines the minimum privilege of self-registered accounts. It does not constrain the maximum, however: an explicit `"ADMIN"` is equally valid under the same schema. The role value is stored as an enum column, not a free string, and is returned as a string in `PublicUser`.

**Rule workflow**:
```
request body role
  -> Zod enum + default('OPERATOR')
     -> invalid: 400 VALIDATION_ERROR
     -> omitted: becomes OPERATOR
  -> service/repository
  -> MySQL ENUM('ADMIN','OPERATOR') DEFAULT 'OPERATOR'
```

**Confidence:** High (schema, service, migration all explicit).

---

### Business Rule: BR-05 Caller-controlled role assignment

**Overview**:
The Users Module persists whatever role the caller supplies; it has no role-elevation check, approval step or server-side role policy.

**Detailed description**:
`createUser` forwards `input.role` directly into `this.users.create({ email, passwordHash, name, role: input.role })` (`src/modules/users/user.service.ts:27-32`). There is no comparison against `req.user`, no allowlist conditioned on an authenticated administrator and no separate administrative creation method. The service signature `CreateUserInput` includes `role` as a caller-provided field (`src/modules/users/user.schemas.ts:7-14`), so role is part of the creation contract rather than an internal decision.

The only production caller is `AuthService.register`, which forwards its input unchanged (`src/modules/auth/auth.service.ts:27-29`). The Auth route is public, with no `authenticate` and no `requireRole` middleware (`src/modules/auth/auth.routes.ts:10`), and its schema accepts `ADMIN`. Consequently `POST /api/v1/auth/register` with `"role": "ADMIN"` creates a full administrator. The Users Module's own admin-only route only governs reads; it does not protect creation.

This rule is documented as a behavior, not as an intended policy, because no requirements are available. The architectural report classifies it as Critical (`docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md`, risk register). From the component perspective, the Users Module is where the privilege is materialized in the database, and it provides no compensating control.

**Rule workflow**:
```
anonymous POST /auth/register { role: "ADMIN" }
  -> registerSchema accepts ADMIN
  -> AuthService.register
  -> UserService.createUser
  -> repository.create({ role: "ADMIN" })
  -> administrator account persisted
```

**Confidence:** High (explicit code path; no hidden role guard was found).

---

### Business Rule: BR-06 User creation payload validation

**Overview**:
User creation accepts a strictly shaped payload: valid email up to 255 characters, password 8-72 characters, name 2-150 characters and a valid role.

**Detailed description**:
`createUserSchema` defines `email: z.string().email().max(255)`, `password: z.string().min(8).max(72)`, `name: z.string().min(2).max(150)` and the role enum with default (`src/modules/users/user.schemas.ts:7-12`). The schema is exported together with `CreateUserInput` (`user.schemas.ts:14`). The Auth Module declares a structurally identical `registerSchema` (`src/modules/auth/auth.schemas.ts:3-8`) and is the schema actually applied to the public registration route, because the Users Module has no creation route. `validate.middleware.ts` replaces `req.body` with the parsed output and converts `ZodError` into `ValidationError` with per-field details (`src/middlewares/validate.middleware.ts:14-33`).

The limits align with database column widths for `email` (255) and `name` (150), preventing truncation on those fields. Password is not stored, so its 72-character limit is a bcrypt compatibility bound rather than a column bound. Zod counts characters, while bcrypt operates on bytes and only uses the first 72 bytes; a multi-byte password can therefore exceed 72 bytes while satisfying the character limit. There is no complexity rule (uppercase, digit, symbol), no breached-password check and no rejection of leading/trailing whitespace.

Validation is repeated at two module boundaries. If `UserService.createUser` were called from another module without schema validation, email format and name length would not be rechecked by the service; only the database would enforce length, and invalid email formats would be accepted. The current code relies entirely on the HTTP middleware for these constraints.

**Rule workflow**:
```
POST /auth/register body
  -> registerSchema.parse (duplicate of createUserSchema)
     -> email invalid/too long: 400
     -> password <8 or >72 chars: 400
     -> name <2 or >150 chars: 400
     -> role not ADMIN/OPERATOR: 400
  -> parsed body reaches UserService.createUser
```

**Confidence:** High (schemas explicit). The byte-vs-character bcrypt nuance is inferred from bcrypt's documented 72-byte behavior.

---

### Business Rule: BR-07 UUID path parameter for user lookup

**Overview**:
`GET /api/v1/users/:id` accepts only a UUID-formatted identifier; any other format is rejected before the controller.

**Detailed description**:
`user.routes.ts` declares a local `idParamSchema = z.object({ id: z.string().uuid() })` and applies it with `validate({ params: idParamSchema })` (`src/modules/users/user.routes.ts:7,16`). The validation middleware parses `req.params` and replaces it with the parsed object (`src/middlewares/validate.middleware.ts:21-23`). A non-UUID value such as `123` or `abc` fails before `UserController.getById` runs and produces HTTP 400 with `VALIDATION_ERROR` and details pointing at `params.id`.

A second, exported schema `userIdParamSchema` exists in `user.schemas.ts:3-5` but is never imported. This is duplicated logic: the route-local schema and the exported schema currently enforce the same rule, but only the local one is active. There is no route-level test asserting the 400 behavior.

The UUID rule is a format check, not an existence check. A syntactically valid UUID that does not exist passes validation and reaches the service, where it produces 404 (BR-08). Prisma `User.id` is `CHAR(36)` with a UUID default (`prisma/schema.prisma:26`), so the format is consistent with the storage type.

**Rule workflow**:
```
GET /users/:id
  -> authenticate
  -> requireRole('ADMIN')
  -> validate params: z.string().uuid()
     -> invalid: 400 VALIDATION_ERROR (controller not called)
     -> valid: controller -> service -> repository
```

**Confidence:** High (explicit code), including the observation that `userIdParamSchema` is unused.

---

### Business Rule: BR-08 Unknown user produces 404

**Overview**:
When a lookup finds no user row, the service throws `NotFoundError('User')`, producing HTTP 404 with code `NOT_FOUND`.

**Detailed description**:
`getById` calls `this.users.findById(id)` and checks for `null` (`src/modules/users/user.service.ts:36-38`). On null it throws `new NotFoundError('User')` (`user.service.ts:38`), whose constructor builds the message `User not found`, status 404 and code `NOT_FOUND` (`src/shared/errors/http-errors.ts:27-31`). The controller does not catch this as a special case; its `catch` forwards to `next(err)` (`src/modules/users/user.controller.ts:11-13`), and the centralized error middleware serializes it (`src/middlewares/error.middleware.ts:15-24`).

The same rule applies to the indirect `/auth/me` path. If a JWT is still valid but the user row was deleted (for example by test truncation or direct database operation), `UserService.getById` throws 404 rather than signalling an invalid credential. This is a cross-module semantic consequence: the caller holds a valid token but receives `NOT_FOUND`. No delete endpoint exists, so this state is reachable only through direct database mutation, cascading operations or external administration.

The repository returns `null` rather than throwing, keeping persistence concerns separate from domain error decisions. The service is the only place that decides 404 semantics for user lookup.

**Rule workflow**:
```
controller.getById(id)
  -> service.getById(id)
     -> repository.findById(id)
        -> row found: toPublic -> 200
        -> null: throw NotFoundError('User')
  -> next(err) -> error middleware -> 404 { code: "NOT_FOUND", message: "User not found" }
```

**Confidence:** High (explicit code).

---

### Business Rule: BR-09 Authentication required for user lookup

**Overview**:
`GET /api/v1/users/:id` rejects requests without a valid Bearer JWT.

**Detailed description**:
The users router places `authenticate` before role checking and validation (`src/modules/users/user.routes.ts:14`). The middleware requires an `Authorization` header beginning with `Bearer ` (case-insensitive), extracts the token, and verifies it with `jwt.verify(token, env.JWT_SECRET)` (`src/middlewares/auth.middleware.ts:27-46`). Missing header, empty token and invalid or expired token all produce `UnauthorizedError`, hence HTTP 401 with code `UNAUTHORIZED`.

On success, the middleware hydrates `req.user` from the JWT claims `sub`, `email` and `role` (`auth.middleware.ts:41-42`). It does not query the database, so authentication proves possession of a signed token, not current existence or current role of the user. Token lifetime defaults to 8 hours (`src/config/env.ts:9`).

The middleware ordering means an unauthenticated request is rejected before the UUID validator runs, so callers receive 401 rather than 400 for a bad id when no token is present. This ordering is consistent across the application's protected routers.

**Rule workflow**:
```
request
  -> Authorization header present and "Bearer <token>"?
     -> no: 401 UNAUTHORIZED
  -> jwt.verify(token, JWT_SECRET)
     -> invalid/expired: 401 UNAUTHORIZED
     -> valid: req.user = { id: sub, email, role }; next()
```

**Confidence:** High (explicit code).

---

### Business Rule: BR-10 User lookup is ADMIN-only

**Overview**:
Even an authenticated `OPERATOR` cannot read arbitrary user records; `GET /api/v1/users/:id` requires `role === 'ADMIN'`.

**Detailed description**:
`requireRole('ADMIN')` is applied after authentication (`src/modules/users/user.routes.ts:15`). The middleware rejects missing `req.user` with 401 and rejects a role not in the allowed list with `ForbiddenError('Insufficient permissions')`, producing HTTP 403 and code `FORBIDDEN` (`src/middlewares/auth.middleware.ts:49-61`). This is the only role-protected route in the entire application according to the architectural report.

The role decision uses the JWT claim only, not a fresh database read. A user whose database role was changed from `ADMIN` to `OPERATOR` remains able to call the endpoint until the token expires (default 8 hours), and a user demoted in the database is not revalidated. Conversely, a newly promoted user must log in again to receive a token carrying `ADMIN`.

The rule protects reads of user records but not creation. `POST /auth/register` is public and can create an `ADMIN` (BR-05), so the endpoint's authorization is weaker than the system's overall privilege model. Within the Users Module itself, no endpoint allows modifying or deleting a user.

**Rule workflow**:
```
authenticate -> req.user.role
  -> role === 'ADMIN': next()
  -> role !== 'ADMIN': 403 FORBIDDEN
  -> no req.user (middleware misordered/direct use): 401 UNAUTHORIZED
```

**Confidence:** High (explicit route composition).

---

### Business Rule: BR-11 Server-managed identity and timestamps

**Overview**:
User identifiers and timestamps are generated by the database/ORM and cannot be supplied by callers.

**Detailed description**:
`User.id` is `String @id @default(uuid()) @db.Char(36)` (`prisma/schema.prisma:26`), so Prisma generates a UUID when the row is created unless an id is explicitly provided. The repository's `create` data type declares only `email`, `passwordHash`, `name` and `role` (`src/modules/users/user.repository.ts:14`), so the service cannot pass an id, `createdAt` or `updatedAt` through the typed API. The HTTP schemas likewise declare only the four creation fields (`src/modules/users/user.schemas.ts:7-12`).

Timestamps are database-managed: `createdAt DateTime @default(now())` and `updatedAt DateTime @updatedAt` (`prisma/schema.prisma:31-32`). `createdAt` is generated once by MySQL (`DEFAULT CURRENT_TIMESTAMP(3)` in the migration), and Prisma manages `updatedAt` on writes. There is currently no update path in the Users Module, so `updatedAt` equals creation time for every user created by this code.

The public projection exposes both timestamps as `Date` values (`src/modules/users/user.service.ts:12-13,48-49`), which Express serializes to ISO-8601 strings. No timezone strategy is encoded in the schema or configuration; MySQL `DATETIME(3)` stores wall-clock values without timezone.

**Rule workflow**:
```
HTTP body (email, password, name, role)
  -> service builds data object (email, passwordHash, name, role)
  -> prisma.user.create
     -> id = uuid() at database/ORM layer
     -> createdAt = now(), updatedAt managed by Prisma
  -> toPublic returns id/createdAt/updatedAt
```

**Confidence:** High (schema and repository signature explicit). The exact timezone interpretation of `DATETIME(3)` is not defined by the code.

---

### Business Rule: BR-12 No user mutation, deletion, listing or password lifecycle

**Overview**:
The module exposes no update, delete, list, password change/reset or deactivation operation. Users are created and read only.

**Detailed description**:
The users router contains exactly one route: `GET /:id` (`src/modules/users/user.routes.ts:9-20`). `UserRepository` contains exactly three methods - `findById`, `findByEmail`, `create` (`src/modules/users/user.repository.ts:3-16`). `UserService` contains exactly `createUser`, `getById` and the static `toPublic` (`src/modules/users/user.service.ts:18-51`). No `update`, `delete`, `list`, `findAll`, `changePassword` or `setRole` symbol exists in the component.

As a consequence, there is no administrative interface to enumerate users, promote or demote them, reset a forgotten password, deactivate an account or remove a compromised identity. Role changes can only be performed by direct database access or by creating a new account with a different role. The absence of a delete route also means the `ON DELETE RESTRICT` foreign keys (BR-14) are never exercised through the API.

The Auth Module provides no password-reset or logout capability either, and tokens are stateless for 8 hours. From a business-lifecycle perspective, the user record is effectively append-only and immutable through the public API, with no status field such as `active`, `disabled` or `emailVerified` in the schema (`prisma/schema.prisma:25-38`).

**Rule workflow**:
```
Available: create (via Auth), findById, findByEmail
Not available: update, delete, list, change password, change role, deactivate
Attempting other HTTP methods on /api/v1/users/:id
  -> no matching route -> app 404 handler -> NotFoundError("Route METHOD /api/v1/users/:id")
```

**Confidence:** High for absence (exhaustive scan of the component). Whether this is intentional is unknown; no requirements exist.

---

### Business Rule: BR-13 Email identity follows database collation

**Overview**:
Email equality and uniqueness are resolved by MySQL under `utf8mb4_unicode_ci`; the application performs no trimming, case folding or normalization.

**Detailed description**:
User input is passed to `findByEmail` and `create` exactly as parsed by Zod (`src/modules/users/user.service.ts:22,27`). Zod's `.email()` validates shape but does not lowercase or trim the value. Prisma parameterizes the value into `WHERE email = ?` and `INSERT`, and MySQL applies the column collation. The `users` table is created with `DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci` (`prisma/migrations/20260519182739_init/migration.sql:13`), a case-insensitive, accent-insensitive collation.

Under that collation, `Admin@Example.com` and `admin@example.com` compare equal for both the unique index and the lookup. This makes login case-insensitive in practice and prevents duplicate addresses that differ only by case. However, the stored value preserves the original casing, so `PublicUser.email` returns whatever the caller supplied at registration. Uniqueness behavior is therefore a database-collation property, not an application invariant.

If the collation were changed to a binary/case-sensitive collation, the same code would allow both variants to coexist and would make login case-sensitive, without any code change or test failure. There is no explicit normalization layer, no `LOWER(email)` index and no documented identity policy in the repository.

**Rule workflow**:
```
input.email (no normalization)
  -> prisma findUnique({ email })
  -> MySQL utf8mb4_unicode_ci comparison (case/accent-insensitive)
  -> unique index uses same collation
```

**Confidence:** Medium-high. The collation is explicit in the migration; the assumption that the deployed database uses the migration's table definition is not independently verifiable from the repository.

---

### Business Rule: BR-14 Referential integrity with orders and audit history

**Overview**:
A user referenced by an order or by order status history cannot be deleted at the database level; `createdById` and `changedById` are mandatory foreign keys.

**Detailed description**:
The `orders` table defines `createdById CHAR(36) NOT NULL` with a foreign key to `users.id` and `ON DELETE RESTRICT ON UPDATE CASCADE` (`prisma/migrations/20260519182739_init/migration.sql:59,112-113`). The `order_status_history` table defines `changedById CHAR(36) NOT NULL` with the same restriction (`migration.sql:93,124-125`). Prisma models declare the relations as `createdBy User` and `changedBy User` (`prisma/schema.prisma:88,126`).

Every order creation writes the authenticated user's id into `createdById` (`src/modules/orders/order.service.ts:104`), and every initial and subsequent history entry writes it into `changedById` (`order.service.ts:110,164`). The Users Module never participates in those writes directly; the dependency is a schema-level contract that requires a valid user row to exist for every order operation. Deleting a user who has authored an order or a status change is blocked by the database.

Because the Users Module exposes no delete operation, this constraint is currently latent. It becomes relevant to any future administrative deletion or data-retention feature and explains why direct SQL cleanup must delete dependent orders/history first. The test setup follows that order: history, items, orders and sequence are deleted before users (`tests/setup.ts:9-15`).

**Rule workflow**:
```
order create/changeStatus
  -> createdById/changedById = req.user.id
  -> FK check against users.id
     -> user absent: FK error (P2003, generic 500 if unhandled)
     -> user present: write succeeds
delete user (not exposed)
  -> FK RESTRICT blocks deletion while orders/history reference the user
```

**Confidence:** High (schema and migration explicit). The runtime 500 outcome is inferred from the error middleware's limited Prisma mapping.

---

## 4. Component Structure

```
src/modules/users/
├── user.routes.ts        (21 lines)  HTTP binding; authenticate -> requireRole('ADMIN') -> UUID params -> controller
├── user.controller.ts    (15 lines)  Express handler; reads req.params.id, returns 200 JSON, forwards errors to next()
├── user.service.ts       (52 lines)  Business logic; email uniqueness, bcrypt hashing, public projection, 404 semantics
├── user.repository.ts    (17 lines)  Prisma access; findById, findByEmail, create
└── user.schemas.ts       (14 lines)  Zod creation schema + UUID param schema + CreateUserInput type
```

### 4.1 File responsibilities and key symbols

| File | Key symbols | Responsibility | Notes |
|------|-------------|----------------|-------|
| `user.routes.ts` | `idParamSchema`, `buildUserRouter` | Route composition and middleware ordering | Local UUID schema duplicates exported `userIdParamSchema` |
| `user.controller.ts` | `UserController`, `getById` | HTTP adaptation only | Uses non-null assertion `req.params.id!`; no response envelope |
| `user.service.ts` | `PublicUser`, `BCRYPT_ROUNDS`, `UserService`, `createUser`, `getById`, `toPublic` | All domain rules | No transaction, no logging, no role policy |
| `user.repository.ts` | `UserRepository`, `findById`, `findByEmail`, `create` | Persistence abstraction | Thin Prisma passthrough; concrete dependency, no interface |
| `user.schemas.ts` | `userIdParamSchema`, `createUserSchema`, `CreateUserInput` | Input contract | `userIdParamSchema` unused; creation schema shadowed by `auth.schemas.ts` on the actual route |

### 4.2 Persistence model used by the component

| Field | Prisma type | Database type | Constraints | Source |
|-------|-------------|---------------|-------------|--------|
| `id` | `String @id @default(uuid())` | `CHAR(36)` | Primary key | `prisma/schema.prisma:26` |
| `email` | `String @unique` | `VARCHAR(255)` | Unique index `users_email_key`, utf8mb4_unicode_ci | `prisma/schema.prisma:27`, migration line 11-13 |
| `passwordHash` | `String` | `VARCHAR(255)` | Not null | `prisma/schema.prisma:28` |
| `name` | `String` | `VARCHAR(150)` | Not null | `prisma/schema.prisma:29` |
| `role` | `UserRole @default(OPERATOR)` | `ENUM('ADMIN','OPERATOR')` | Default `OPERATOR` | `prisma/schema.prisma:30`, migration line 7 |
| `createdAt` | `DateTime @default(now())` | `DATETIME(3)` | Default current timestamp | `prisma/schema.prisma:31` |
| `updatedAt` | `DateTime @updatedAt` | `DATETIME(3)` | Prisma-managed | `prisma/schema.prisma:32` |

Relations owned by the model but not exposed by the module: `createdOrders Order[]` and `statusHistoryChanges OrderStatusHistory[]` (`prisma/schema.prisma:34-35`). Table mapping is `@@map("users")` (`prisma/schema.prisma:37`).

### 4.3 Composition and mounting

```
src/app.ts:27-32
  UserRepository(prisma)
    -> UserService(userRepository)
       -> UserController(userService)
       -> AuthService(userRepository, userService)
          -> AuthController(authService, userService)

src/routes/index.ts:25
  router.use('/users', buildUserRouter(controllers.users))

src/app.ts:67
  app.use('/api/v1', buildApiRouter(controllers))
```

The module has no container, factory or interface. Construction is manual in the composition root, and the same `UserRepository` instance is shared between the Users and Auth modules. There are no module-level configuration values except the hardcoded `BCRYPT_ROUNDS` constant.

---

## 5. Dependency Analysis

### 5.1 Internal dependency graph

```
UserController --> UserService --> UserRepository --> PrismaClient --> MySQL (users)
      ^                |                    ^
      |                +--> shared/errors     |
      |                +--> user.schemas       |
      |                                        |
UserRoutes --> auth.middleware                  |
      |     --> validate.middleware             |
      |     --> UserController                  |
      |                                         |
routes/index --> UserRoutes                     |
app.ts -------> UserRepository, UserService, UserController
                 ^
auth.service ---+ (findByEmail, createUser via UserService)
auth.controller-+ (getById via UserService)
orders DB columns -> users.id (FK RESTRICT)
```

### 5.2 Internal dependencies (outgoing)

| From | To | Kind | Purpose |
|------|----|------|---------|
| `user.routes.ts` | `middlewares/auth.middleware.ts` | Runtime import | `authenticate`, `requireRole` |
| `user.routes.ts` | `middlewares/validate.middleware.ts` | Runtime import | `validate` |
| `user.routes.ts` | `user.controller.ts` | Type-only import | `UserController` type |
| `user.controller.ts` | `user.service.ts` | Type-only import | `UserService` type |
| `user.service.ts` | `user.repository.ts` | Type-only import | `UserRepository` type |
| `user.service.ts` | `user.schemas.ts` | Type-only import | `CreateUserInput` |
| `user.service.ts` | `shared/errors/index.ts` | Runtime import | `ConflictError`, `NotFoundError` |
| `user.repository.ts` | `@prisma/client` | External type import | `PrismaClient`, `User` |

### 5.3 Consumers of the module (incoming)

| Consumer | Uses | Purpose |
|----------|------|---------|
| `src/app.ts:27-32` | `UserRepository`, `UserService`, `UserController` | Composition and wiring |
| `src/routes/index.ts:8,25` | `buildUserRouter` | Mount at `/api/v1/users` |
| `src/modules/auth/auth.service.ts:6-7,22-29,42` | `UserRepository`, `UserService`, `PublicUser` | Registration and login lookup/projection |
| `src/modules/auth/auth.controller.ts:3,9,33` | `UserService` | `/auth/me` identity lookup |
| `prisma` schema/migration | `users` table | FK target for `orders.createdById` and `order_status_history.changedById` |
| `prisma/seed.ts:191-210` | Prisma directly, not the repository | Seeds admin/operator users with bcrypt hashes |
| `tests/helpers/factories.ts:17-31` | Prisma + bcrypt directly | Test user creation bypassing the service |

### 5.4 External dependencies

| Dependency | Version | Used in | Purpose |
|------------|---------|---------|---------|
| `@prisma/client` | 5.22.0 | `user.repository.ts`, `user.service.ts` (type) | ORM client and generated `User`/`UserRole` types |
| `bcrypt` | 5.1.1 | `user.service.ts:1,26` | Password hashing (cost 10) |
| `zod` | 3.23.8 | `user.schemas.ts:1`, `user.routes.ts:2` | Payload and UUID validation |
| `express` | 4.21.1 | `user.routes.ts`, `user.controller.ts` | Router and request handler types |
| `jsonwebtoken` | 9.0.2 | indirect via `auth.middleware.ts` | Bearer-token verification for BR-09 |
| MySQL | 8.0 | runtime | `users` table, unique index, enum, timestamps, FK restrictions |
| Prisma CLI / migrations | 5.22.0 | `prisma/schema.prisma` | DDL ownership of the `users` table |

### 5.5 Dependency observations

- The repository depends on the concrete `PrismaClient` rather than an interface, so substituting persistence requires either a real database or a mocked object with matching methods.
- `UserService` depends on `UserRepository` as a type only; at runtime the instance is injected. This is constructor injection without a port/interface.
- The module imports the shared error barrel rather than individual files, coupling it to the barrel's export surface (13 lines in `src/shared/errors/index.ts`).
- The module has no dependency on `src/config/env.ts`, `src/shared/logger`, `uuid`, or the pagination helper. It does not log, configure or paginate.
- No cyclic dependency was detected between the Users Module and any other module. The Auth Module depends on Users, never the reverse.

---

## 6. Afferent and Efferent Coupling

### 6.1 Method and definitions

Afferent coupling (Ca) is the number of distinct internal source files that import a file. Efferent coupling (Ce) is the number of distinct internal source files that a file imports. External npm packages are excluded. Instability is `I = Ce / (Ca + Ce)`; values near 0 indicate a stable, widely depended-on component, values near 1 indicate an unstable consumer. Counts are static, derived from `import`/`import type` statements in `src` at revision `e7f6311`.

### 6.2 File-level coupling

| Component | Afferent (Ca) | Efferent (Ce) | Instability (I) | Criticality | Afferent sources |
|-----------|---------------|---------------|-----------------|-------------|------------------|
| `UserService` | 4 | 3 | 0.43 | Critical | `user.controller.ts`, `auth.service.ts`, `auth.controller.ts`, `app.ts` |
| `UserRepository` | 3 | 0 | 0.00 | Critical | `user.service.ts`, `auth.service.ts`, `app.ts` |
| `UserController` | 2 | 1 | 0.33 | Medium | `user.routes.ts`, `app.ts` |
| `UserRoutes` | 1 | 3 | 0.75 | Medium | `routes/index.ts` |
| `UserSchemas` | 1 | 0 | 0.00 | Medium | `user.service.ts` |

### 6.3 Module-level coupling

| Boundary | Afferent (Ca) | Efferent (Ce) | Instability (I) | Interpretation |
|----------|---------------|---------------|-----------------|----------------|
| `src/modules/users/` treated as one component | 4 internal files (`app.ts`, `auth.service.ts`, `auth.controller.ts`, `routes/index.ts`) | 3 internal files across 2 folders (`middlewares/auth.middleware.ts`, `middlewares/validate.middleware.ts`, `shared/errors/index.ts`) | 0.43 | Balanced: more depended-on than depending, but not a pure stable core |
| Comparison with architectural report | 4 | 2 module folders | 0.33 | Report counts module folders rather than files |

### 6.4 Coupling interpretation

- `UserRepository` is the most stable file in the module (Ca 3, Ce 0): it depends only on the external Prisma client and is imported by both feature modules and the composition root. Any signature change affects `UserService`, `AuthService` and `app.ts`.
- `UserService` is the behavioral contract of the module and is imported by four files, including the Auth Module's controller. Its instability of 0.43 reflects that it both consumes three internal files and is consumed by four.
- `UserRoutes` is the most unstable file (I 0.75). It composes two middleware files and the controller, and only `routes/index.ts` imports it, which is normal for an adapter at the edge.
- `UserSchemas` has zero efferent coupling and only one afferent (the service, for a type). The route does not use the exported UUID schema, which weakens the intended contract role of the file.
- The module's afferent coupling is inflated by the Auth Module: two of the four incoming files are auth files. This indicates a one-directional feature dependency `auth -> users`, which is acceptable in the current layering but means Users changes can break authentication.
- Database-level coupling is not captured by Ca/Ce. The `orders` and `order_status_history` foreign keys make the Users table a schema-level dependency of the Orders Module, with `ON DELETE RESTRICT`. This is a hidden coupling that static import analysis does not show.

---

## 7. Endpoints

### 7.1 Endpoint owned by the Users Module

| Endpoint | Method | Auth | Role | Request | Success | Error responses |
|----------|--------|------|------|---------|---------|-----------------|
| `/api/v1/users/{id}` | GET | Bearer JWT required | `ADMIN` | Path `id` must be UUID | 200 with `PublicUser` | 400 `VALIDATION_ERROR`; 401 `UNAUTHORIZED`; 403 `FORBIDDEN`; 404 `NOT_FOUND` |

Response shape (200):

```json
{
  "id": "3f1c...",
  "email": "admin@oms.local",
  "name": "Administrador do Sistema",
  "role": "ADMIN",
  "createdAt": "2026-11-08T00:00:00.000Z",
  "updatedAt": "2026-11-08T00:00:00.000Z"
}
```

Error shape:

```json
{ "error": { "code": "FORBIDDEN", "message": "Insufficient permissions" } }
```

### 7.2 Endpoints that invoke the Users Module indirectly

These routes are owned by the Auth Module but execute Users Module code. They are listed for completeness and are not part of the Users router.

| Endpoint | Method | Auth | Role | Users Module code executed | Success | Error responses |
|----------|--------|------|------|----------------------------|---------|-----------------|
| `/api/v1/auth/register` | POST | None (public) | None | `UserService.createUser`, `toPublic` | 201 with `PublicUser` | 400 `VALIDATION_ERROR`; 409 `EMAIL_ALREADY_USED`; 409 `CONFLICT` on race |
| `/api/v1/auth/me` | GET | Bearer JWT required | Any | `UserService.getById`, `toPublic` | 200 with `PublicUser` | 401 `UNAUTHORIZED`; 404 `NOT_FOUND` |

### 7.3 Non-existent endpoints

There is no `GET /api/v1/users` (list), `POST /api/v1/users`, `PATCH /api/v1/users/:id`, `DELETE /api/v1/users/:id`, password-change endpoint or role-change endpoint. Requests to those paths fall through to the application 404 handler (`src/app.ts:69-71`), which returns `NOT_FOUND` with the message `Route METHOD /path not found`.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 `users` table | External stateful dependency | Durable user records, unique email, role enum, timestamps | TCP/MySQL protocol via Prisma | Relational rows (`CHAR(36)`, `VARCHAR`, `ENUM`, `DATETIME(3)`) | `P2002` mapped to 409 `CONFLICT`; `P2025` to 404; all other Prisma errors become 500 after logging (`src/middlewares/error.middleware.ts:37-65`) |
| Prisma Client 5.22 | Data access library | Query building and entity mapping | In-process | Generated TypeScript types / SQL | `findUnique` returns `null`; `create` throws `PrismaClientKnownRequestError` on constraints |
| bcrypt 5.1.1 | Security library | Hash passwords with cost 10 | In-process | UTF-8 string -> bcrypt hash string | No local try/catch; failures bubble to the controller and become 500 |
| Zod 3.23.8 | Validation library | Creation payload and UUID param contracts | In-process | JSON -> typed object | `ZodError` converted to 400 `VALIDATION_ERROR` with field details (`src/middlewares/validate.middleware.ts:25-32`) |
| JWT auth middleware | Inbound security integration | Verifies Bearer tokens and supplies `req.user` | HTTP `Authorization: Bearer` | JWT (HS256) with `sub`, `email`, `role`, `iat`, `exp` | 401 `UNAUTHORIZED` for missing/invalid/expired token (`src/middlewares/auth.middleware.ts:27-47`) |
| Role authorization middleware | Inbound authorization integration | Restricts lookup to `ADMIN` | In-process middleware | `req.user.role` string | 403 `FORBIDDEN` (`src/middlewares/auth.middleware.ts:49-61`) |
| Auth Module (internal service contract) | Internal module integration | Registration creation and identity lookup | In-process method calls | TypeScript objects (`CreateUserInput`, `PublicUser`) | Throws `ConflictError` / `NotFoundError`; Auth controllers forward via `next` |
| Orders Module (schema contract) | Internal data integration | Author attribution in orders and status history | SQL foreign key | `CHAR(36)` ids | `ON DELETE RESTRICT`; unhandled FK violations become 500 |
| Seed script | Development tooling integration | Creates `admin@oms.local` and `operador@oms.local` | Direct Prisma + bcrypt | Prisma create calls | No error handling beyond top-level catch and exit 1 (`prisma/seed.ts:327-335`) |
| Test factories | Test integration | Create users directly via Prisma + bcrypt | In-process | Prisma create calls | Throws on database failure; no cleanup inside factory |

### Integration observations

- MySQL is the only external stateful integration. There are no outbound HTTP calls, message brokers, caches or identity providers.
- Password hashing happens in-process and is CPU-bound (bcrypt cost 10). There is no rate limit in front of the registration path that triggers it.
- The unique email index is the only database-level guard for user identity beyond the primary key and enum.
- The module does not emit logs, metrics or events of its own. Successful user creation and lookup are only visible through the generic HTTP access log (`src/middlewares/request-logger.middleware.ts:12-25`), and only after the response finishes.
- Error translation is centralized, so the module never formats HTTP errors itself; it only chooses the domain error type.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered architecture | routes -> controller -> service -> repository | `src/modules/users/*` | Separation of HTTP, application logic and persistence |
| Repository pattern | `UserRepository` wraps Prisma calls | `src/modules/users/user.repository.ts:3-16` | Isolate query construction from business logic |
| DTO / public projection | `PublicUser` + `UserService.toPublic` | `src/modules/users/user.service.ts:7-14,42-51` | Whitelist response fields and prevent credential leakage |
| Constructor injection / manual DI | Repository passed into service; service into controller | `src/modules/users/user.service.ts:19`, `user.controller.ts:5`, `src/app.ts:27-29` | Explicit dependency wiring without a container |
| Composition root | `buildControllers` instantiates and connects all components | `src/app.ts:26-53` | Single place for object graph construction |
| Transaction script (anemic domain model) | Business logic lives in service methods; `User` is a Prisma data record with no behavior | `src/modules/users/user.service.ts`, `prisma/schema.prisma:25-38` | Simple procedural logic for a small aggregate |
| Schema-first boundary validation | Zod schemas parsed by middleware before controller | `src/modules/users/user.schemas.ts`, `src/middlewares/validate.middleware.ts` | Reject malformed input at the HTTP boundary and infer types |
| Shared error taxonomy | Typed `AppError` subclasses | `src/modules/users/user.service.ts:4`, `src/shared/errors/*` | Consistent HTTP status and machine-readable error codes |
| Centralized error translation | `errorMiddleware` maps errors to JSON | `src/middlewares/error.middleware.ts:14-65` | Uniform error responses and unknown-error masking |
| Static factory method | `UserService.toPublic` | `src/modules/users/user.service.ts:42-51` | Stateless transformation from entity to DTO |
| Fail-fast configuration (indirect) | `env.ts` validates `JWT_SECRET` before auth middleware runs | `src/config/env.ts:3-27` | Prevent startup without a usable signing secret |
| Stateless authentication (indirect) | JWT claim-based `req.user` and role checks | `src/middlewares/auth.middleware.ts:40-46` | No server-side session store |

### Architectural decisions observed

- The Users Module is the only module with a cross-module service dependency from Auth, making it a de facto identity kernel for authentication.
- The module is intentionally thin: no domain entity, no value objects, no repository interface, no use-case classes. Prisma records are used directly inside the service.
- Creation is owned by Auth at the HTTP level but by Users at the persistence level, splitting the user lifecycle across two modules.
- The module uses whitelist projection rather than ORM serialization, which is the strongest data-protection decision in the component.
- Role is treated as a data field at creation and as a trusted token claim at authorization time; there is no server-side role policy enforcement point in the Users Module.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Critical | `UserService.createUser` / Auth route | Caller-supplied role is persisted without authorization; public registration accepts `ADMIN` | Privilege escalation; any anonymous caller can obtain administrative access |
| High | `UserService.createUser` | Email uniqueness is check-then-insert with no transaction, lock or upsert | Concurrent registrations race; one request fails with generic `CONFLICT` instead of `EMAIL_ALREADY_USED` |
| High | Authorization model | Role is read only from JWT claims; no database revalidation of user existence or current role | Demoted/deleted users retain privileges for up to 8 hours (default token lifetime) |
| High | Users Module lifecycle | No deactivation, deletion, password reset or role-management capability | No revocation path for compromised accounts beyond direct database access and token expiry |
| Medium | Creation schemas | `createUserSchema` and `registerSchema` duplicate the same contract; only the Auth copy is applied to the public route | Contract drift; Users Module validation can be bypassed if service is called elsewhere |
| Medium | `user.routes.ts` / `user.schemas.ts` | UUID param schema is declared twice; exported `userIdParamSchema` is dead code | Duplicated validation logic and misleading API of the schemas file |
| Medium | Email identity | No trimming, lowercasing or normalization; behavior depends on MySQL collation | Identity semantics can change with a collation/migration change; stored casing is caller-controlled |
| Medium | Password schema | `max(72)` counts characters while bcrypt truncates at 72 bytes | Multi-byte passwords can be silently truncated, reducing effective entropy |
| Medium | Persistence abstraction | Concrete `PrismaClient` dependency, no repository interface | Unit testing requires database or hand-built mocks; persistence substitution is not possible without changing the service |
| Medium | Error mapping | Prisma `P2003` (FK violation) and other known errors are not mapped | Referential failures surface as 500 `INTERNAL_SERVER_ERROR` instead of a 4xx domain error |
| Medium | Data retention | `ON DELETE RESTRICT` from orders and history means user records cannot be removed without handling dependent data | Blocks privacy erasure and cleanup features; latent constraint if deletion is added |
| Medium | Auditability | No `createdBy`, role-change audit, login audit or module logging for user creation/promotion | Administrative actions cannot be attributed or reconstructed |
| Low | `PublicUser.role` typing | Role union re-declared as string literals instead of Prisma `UserRole` | Type drift if the enum changes; no compile-time link to schema |
| Low | `UserController.getById` | Uses non-null assertion `req.params.id!` after middleware validation | Static type safety is not carried across the middleware boundary; relies on runtime ordering |
| Low | Test coverage | No dedicated unit or integration tests for the Users Module or its endpoint | Regressions in authorization, 404 handling or projection can pass CI |
| Low | Password strength | Only length is enforced; no complexity, common-password or breach checks | Weak passwords are accepted; combined with no rate limiting, guessing is easier |
| Low | Configuration | bcrypt cost hardcoded at 10; no configurable work factor or rehash policy | Cannot tune cost per environment or upgrade hashes on login |
| Low | Seed data | Seeded users share well-known passwords (`admin123`, `operator123`) and are created outside the service | Unsafe if seed is run in a non-development environment; bypasses service rules |

### Risk context

- The module has no transaction boundary because it performs only one write. The absence of a transaction is not a defect by itself, but the two-step read-check/write sequence means the uniqueness decision is not atomic.
- The Users Module is not a source of performance bottlenecks: lookups use the primary key or the unique email index, and creation is a single insert plus one bcrypt hash.
- The most consequential debt is not code size or complexity; it is the absence of a role-assignment policy at the persistence point. Everything else is medium or low severity by comparison.

---

## 11. Test Coverage Analysis

### 11.1 Test inventory and locations

| Test file | Relevant tests | Users Module code exercised | Direct/Indirect |
|-----------|----------------|-----------------------------|-----------------|
| `tests/auth.test.ts` | `registers a new user and returns 201 with public data only` (lines 6-17) | `UserService.createUser`, `toPublic`, `UserRepository.findByEmail/create` | Indirect via Auth |
| `tests/auth.test.ts` | `rejects register with duplicated email` (lines 19-28) | `UserService.createUser` conflict branch | Indirect |
| `tests/auth.test.ts` | `rejects register with invalid payload` (lines 30-39) | Zod registration contract (duplicate of `createUserSchema`) | Indirect |
| `tests/auth.test.ts` | `logs in successfully and returns an access token` (lines 41-52) | `UserService.toPublic`, `UserRepository.findByEmail` | Indirect |
| `tests/auth.test.ts` | `rejects login with wrong password` (lines 54-63) | `UserRepository.findByEmail` | Indirect |
| `tests/auth.test.ts` | `returns the authenticated user on GET /auth/me` (lines 65-77) | `UserService.getById`, `toPublic`, `findById` | Indirect |
| `tests/auth.test.ts` | `rejects /auth/me without token` (lines 79-83) | Auth middleware only; no Users Module execution | No |
| `tests/orders.test.ts` | Order creation and status tests | User ids used as `createdById`/`changedById` via FK; no Users Module method | No (schema coupling only) |
| `tests/helpers/factories.ts` | `createTestUser` (lines 17-31) | None; bypasses service and inserts via Prisma + bcrypt directly | No |
| `tests/helpers/factories.ts` | `bootstrapAuthenticatedUser` (lines 76-83) | None; creates user directly and logs in through Auth | No |
| `tests/setup.ts` | `beforeEach` (lines 8-16) | Deletes `users` rows (last, after dependent tables) | No |

There are no files matching `users.test.ts`, no unit tests under `src/modules/users/`, and no test that calls `GET /api/v1/users/:id`.

### 11.2 Scenario coverage assessment

| Component / behavior | Unit tests | Integration tests | Coverage assessment | Test quality |
|----------------------|------------|-------------------|---------------------|--------------|
| `UserService.createUser` happy path | 0 | 1 indirect (`auth.test.ts:6-17`) | Covered, including hash-at-rest assertion (`passwordHash` absent) | Good: checks status, body shape and credential absence |
| `UserService.createUser` duplicate email | 0 | 1 indirect (`auth.test.ts:19-28`) | Covered for sequential duplicate only | Good assertion on `EMAIL_ALREADY_USED`; race path untested |
| Creation validation | 0 | 1 indirect (`auth.test.ts:30-39`) | Covered at Auth schema, not at `createUserSchema` | Partial: checks 400 and details array, not individual field rules |
| `UserService.getById` existing user | 0 | 1 indirect (`auth.test.ts:65-77`) | Covered via `/auth/me`, not via `/users/:id` | Good identity assertions; endpoint authorization untested |
| `UserService.getById` missing user | 0 | 0 | Not covered | None; 404 branch unverified |
| `UserService.toPublic` projection | 0 | 1 indirect | Covered for registration; login/me also rely on it | Good negative assertions for `passwordHash`/`password` only in registration |
| `UserController.getById` | 0 | 0 | Not covered | None |
| `user.routes.ts` middleware order | 0 | 0 | Not covered | None |
| `GET /users/:id` success | 0 | 0 | Not covered | None |
| `GET /users/:id` 401 missing/invalid token | 0 | 0 | Not covered (401 is tested for `/auth/me` only) | None |
| `GET /users/:id` 403 non-ADMIN | 0 | 0 | Not covered | None |
| `GET /users/:id` 400 invalid UUID | 0 | 0 | Not covered | None |
| `user.schemas.ts` `userIdParamSchema` | 0 | 0 | Not covered and unused | Dead code not exercised |
| `UserRepository.findByEmail` | 0 | Indirect through registration/login | Covered for existing and missing (login wrong email not tested) | Partial |
| `UserRepository.create` | 0 | 1 indirect | Covered | Good |
| FK restriction on user deletion | 0 | 0 | Not covered and not reachable through API | None |
| Role defaulting to `OPERATOR` | 0 | 1 indirect (`auth.test.ts:6-17` sends role explicitly; duplicate test omits role at line 24) | Partially covered by an omitted-role registration in the duplicate test, not asserted | Weak: no explicit default assertion |
| Role escalation via registration | 0 | 0 | Not covered | None; no test sends `role: ADMIN` |

### 11.3 Coverage metrics

No measured coverage is available. `package.json` defines `"test": "vitest run"` with no `--coverage` flag and no coverage provider dependency (`@vitest/coverage-v8` is absent). `vitest.config.ts` configures no `coverage` block and no thresholds. Reporting a numeric percentage would be fabricated; the assessment above is scenario-based.

Static observation: the service and repository are exercised indirectly by `tests/auth.test.ts`, while the router and controller have zero direct execution through their own endpoint. The owned HTTP surface of the component is effectively untested.

### 11.4 Test harness characteristics

- Tests are integration-only against a real MySQL instance configured by `DATABASE_URL`; `tests/setup.ts:4-6` connects Prisma before all tests.
- `beforeEach` truncates all tables in dependency order, with `users` last (`tests/setup.ts:8-16`). There is no environment guard, so running the suite against a non-test database destroys data.
- Tests run serially: `fileParallelism: false`, single fork, 30-second timeouts (`vitest.config.ts:7-17`).
- `getTestApp` caches one Express app across tests (`tests/helpers/factories.ts:8-15`), while the database is reset between tests.
- User fixtures bypass `UserService` entirely and hash passwords directly with bcrypt (`tests/helpers/factories.ts:20-29`), so service-level creation rules are not exercised by fixture construction.

### 11.5 Test coverage risk

Missing coverage is concentrated on the component's authorization boundary. The only ADMIN-protected route in the system has no test proving that `OPERATOR` receives 403, that missing tokens receive 401, that malformed UUIDs receive 400, or that unknown ids receive 404. The indirect tests validate creation and projection well enough to catch credential leakage regressions, but they do not validate the endpoint the Users Module actually exposes.

---

## Appendix A - Source file reference index

| Path | Lines | Role |
|------|-------|------|
| `src/modules/users/user.routes.ts` | 21 | Router, auth, role, param validation |
| `src/modules/users/user.controller.ts` | 15 | HTTP handler |
| `src/modules/users/user.service.ts` | 52 | Business rules and projection |
| `src/modules/users/user.repository.ts` | 17 | Prisma data access |
| `src/modules/users/user.schemas.ts` | 14 | Zod contracts |
| `src/modules/auth/auth.service.ts` | 51 | Indirect creation and login lookup |
| `src/modules/auth/auth.controller.ts` | 39 | Indirect `/auth/me` lookup |
| `src/modules/auth/auth.schemas.ts` | 16 | Duplicated creation contract |
| `src/middlewares/auth.middleware.ts` | 61 | JWT authentication and role guard |
| `src/middlewares/validate.middleware.ts` | 37 | Zod request parsing |
| `src/middlewares/error.middleware.ts` | 65 | AppError/Zod/Prisma translation |
| `src/shared/errors/http-errors.ts` | 63 | `ConflictError`, `NotFoundError`, etc. |
| `src/app.ts` | 76 | Composition and mounting |
| `src/routes/index.ts` | 31 | Router registry |
| `prisma/schema.prisma` | 138 | `User` model and `UserRole` enum |
| `prisma/migrations/20260519182739_init/migration.sql` | 125 | `users` table, unique index, FKs |
| `tests/auth.test.ts` | 84 | Only tests that indirectly execute the service |
| `tests/helpers/factories.ts` | 83 | User fixtures bypassing the service |
| `tests/setup.ts` | 20 | Database truncation including `users` |
| `prisma/seed.ts` | 335 | Demo users created outside the service |
