# Component Deep Analysis Report — Auth Module

**Component:** Auth Module
**Primary location:** `src/modules/auth/`
**Analyst scope:** `src/modules/auth/` plus directly coupled contracts in `src/modules/users/`, `src/middlewares/`, `src/config/`, `src/shared/`, `src/app.ts`, `src/routes/index.ts`, `prisma/schema.prisma`, and the project test suite
**Excluded paths:** `node_modules`, `.git`, `dist`, `coverage`
**Analysis date:** 2026-11-08
**Report confidence:** High for implemented behavior; low for intended behavior because `docs/PRD.md`, `docs/FDD.md`, `docs/RFC.md` and `docs/TRACKER.md` are empty templates (3 lines each, only a heading and an HTML placeholder comment).

---

## 1. Executive Summary

The Auth Module is a feature module of the Order Management System (OMS) REST API. It owns three externally visible capabilities:

1. Public self-service user registration.
2. Credential-based login that issues a stateless JWT access token.
3. Authenticated identity lookup for the current bearer-token holder.

The module is implemented as four files following the project's standard layered convention: `auth.routes.ts` (HTTP binding and middleware composition), `auth.controller.ts` (transport adaptation), `auth.service.ts` (authentication logic) and `auth.schemas.ts` (Zod request contracts). Persistence is not owned by the module: registration and credential lookup are delegated to the Users Module (`UserService`, `UserRepository`) and Prisma/MySQL. Token verification is not owned by the module either: it lives in `src/middlewares/auth.middleware.ts`, which the Auth Module's `/me` route consumes and which every other protected router in the system also consumes.

The module is small (4 files, 121 lines total) but security-critical. It sits on the system's primary trust boundary: it converts an email/password pair into a bearer credential that grants access to customers, products, orders, and (for `ADMIN`) user records.

### Key findings

| Severity | Finding | Evidence |
|----------|---------|----------|
| Critical | Public registration accepts a client-supplied role, including `ADMIN`, with no authentication or approval step. Any anonymous caller can self-provision an administrator account. | `src/modules/auth/auth.schemas.ts:7`, `src/modules/auth/auth.routes.ts:10`, `src/modules/auth/auth.service.ts:27-29`, `src/modules/users/user.service.ts:21-33` |
| High | No rate limiting, account lockout, CAPTCHA or login backoff. Password guessing and registration abuse are unthrottled. | No rate-limit dependency or middleware; `src/modules/auth/auth.routes.ts:10-12` |
| High | Stateless tokens live for 8 hours by default with no refresh, logout, revocation, denylist or server-side session. Role changes and user deletion remain invisible to issued tokens until expiry. | `src/config/env.ts:8-9`, `src/modules/auth/auth.service.ts:47-50`, `src/middlewares/auth.middleware.ts:40-46` |
| High | JWT verification does not pin an `algorithms` allowlist and does not validate `iss`, `aud`, or the payload structure. The decoded payload is a TypeScript cast only. | `src/middlewares/auth.middleware.ts:19-25,40-46` |
| Medium | Email uniqueness has a check-then-write race. The database constraint is the real invariant; the race path surfaces as generic `CONFLICT` rather than the documented `EMAIL_ALREADY_USED` code. | `src/modules/users/user.service.ts:22-26`, `src/middlewares/error.middleware.ts:37-47` |
| Medium | No email normalization or case folding in application code. Identity semantics depend on the MySQL collation (`utf8mb4_unicode_ci` in `docker-compose.yml:23`). | `src/modules/auth/auth.schemas.ts:4`, `src/modules/auth/auth.service.ts:32` |
| Medium | The registration contract is duplicated between `auth.schemas.ts` and `user.schemas.ts`; the two can drift independently. | `src/modules/auth/auth.schemas.ts:3-8`, `src/modules/users/user.schemas.ts:7-12` |
| Low | `/auth/me` returns `404 NOT_FOUND` if the token is valid but the user row was deleted, instead of treating the credential as no longer valid (`401`). | `src/modules/auth/auth.controller.ts:33`, `src/modules/users/user.service.ts:36-40` |
| Low | The password maximum is expressed in characters (`max(72)`) while bcrypt truncates at 72 bytes; multi-byte passwords can be silently truncated. | `src/modules/auth/auth.schemas.ts:5,12`, `src/modules/users/user.service.ts:16,26` |

### Component role in the system

- Inbound: mounted at `/api/v1/auth` by `src/routes/index.ts:24`, which receives the router from `buildAuthRouter` (`src/modules/auth/auth.routes.ts:7-15`).
- Outbound: delegates to Users Module services/repositories (`src/modules/auth/auth.service.ts:22-25`) and to the shared error taxonomy (`src/modules/auth/auth.service.ts:5`, `src/modules/auth/auth.controller.ts:4`).
- Lateral: the authentication middleware it depends on is reused by customers, products, orders and users routers (`src/modules/customers/customer.routes.ts:14`, `src/modules/products/product.routes.ts:14`, `src/modules/orders/order.routes.ts:14`, `src/modules/users/user.routes.ts:14-15`).

### Scope assumptions

- No written requirements exist for authentication. All rules below are reverse-engineered from code, with confidence levels stated where behavior is implicit.
- `AuthService` has no unit tests of its own; behavior is inferred from code and from integration tests in `tests/auth.test.ts`.
- Coverage percentages are not available because no coverage provider is configured. See Section 11.

---

## 2. Data Flow Analysis

### 2.1 Registration flow (`POST /api/v1/auth/register`)

```
1. Express app receives JSON body (1 MB limit) in src/app.ts:59
2. requestLogger assigns/propagates X-Request-Id in src/middlewares/request-logger.middleware.ts:5-8
3. API router dispatches /api/v1/auth to buildAuthRouter in src/routes/index.ts:24
4. Route-level validate({ body: registerSchema }) parses and replaces req.body in src/modules/auth/auth.routes.ts:10
   - Zod validates email, password, name, role
   - Missing role becomes "OPERATOR" via .default()
   - Unknown keys are stripped by Zod object parsing
5. AuthController.register receives the validated body in src/modules/auth/auth.controller.ts:12-19
6. AuthService.register delegates unchanged to UserService.createUser in src/modules/auth/auth.service.ts:27-29
7. UserService.createUser queries UserRepository.findByEmail in src/modules/users/user.service.ts:22
8. UserRepository.findByEmail executes prisma.user.findUnique({ where: { email } }) in src/modules/users/user.repository.ts:10-12
9a. Existing user -> ConflictError("Email already registered", "EMAIL_ALREADY_USED") -> next(err) -> errorMiddleware -> 409
9b. No existing user -> bcrypt.hash(password, 10) in src/modules/users/user.service.ts:26
10. UserRepository.create persists { email, passwordHash, name, role } in src/modules/users/user.repository.ts:14-16
11. Prisma INSERT into MySQL "users" table; UUID default, role enum, timestamps populated
12. UserService.toPublic strips passwordHash in src/modules/users/user.service.ts:42-51
13. AuthController responds 201 with the PublicUser JSON in src/modules/auth/auth.controller.ts:15
14. On unexpected error -> next(err) -> errorMiddleware logs and returns 500 (or maps Prisma P2002 to 409)
```

### 2.2 Login flow (`POST /api/v1/auth/login`)

```
1. Express JSON body parsing and request logging
2. validate({ body: loginSchema }) in src/modules/auth/auth.routes.ts:11
3. AuthController.login calls AuthService.login in src/modules/auth/auth.controller.ts:21-28
4. AuthService.login calls UserRepository.findByEmail(input.email) in src/modules/auth/auth.service.ts:32
5. Prisma SELECT by unique email index
6a. User not found -> UnauthorizedError("Invalid credentials") -> 401
6b. User found -> bcrypt.compare(input.password, user.passwordHash) in src/modules/auth/auth.service.ts:36
7a. Mismatch -> UnauthorizedError("Invalid credentials") -> 401
7b. Match -> signToken(user.id, user.email, user.role) in src/modules/auth/auth.service.ts:40
8. jwt.sign({ sub, email, role }, env.JWT_SECRET, { expiresIn: env.JWT_EXPIRES_IN }) in src/modules/auth/auth.service.ts:47-50
9. Build LoginResult { user: PublicUser, tokens: { accessToken, expiresIn, tokenType: "Bearer" } } in src/modules/auth/auth.service.ts:41-44
10. AuthController responds 200 with LoginResult in src/modules/auth/auth.controller.ts:24
11. No token, session or audit row is persisted anywhere
```

### 2.3 Current identity flow (`GET /api/v1/auth/me`)

```
1. Router applies authenticate before the controller in src/modules/auth/auth.routes.ts:12
2. authenticate reads Authorization header in src/middlewares/auth.middleware.ts:28
3a. Header absent or not "Bearer ..." -> UnauthorizedError("Missing or invalid Authorization header") -> 401
3b. Bearer token empty -> UnauthorizedError("Missing bearer token") -> 401
4. jwt.verify(token, env.JWT_SECRET) in src/middlewares/auth.middleware.ts:41
5a. Signature/expiry failure -> UnauthorizedError("Invalid or expired token") -> 401
5b. Success -> req.user = { id: payload.sub, email: payload.email, role: payload.role } in src/middlewares/auth.middleware.ts:42
6. AuthController.me verifies req.user exists (defensive) in src/modules/auth/auth.controller.ts:32
7. UserService.getById(req.user.id) in src/modules/auth/auth.controller.ts:33
8. UserRepository.findById -> prisma.user.findUnique({ where: { id } }) in src/modules/users/user.repository.ts:6-8
9a. Row missing -> NotFoundError("User") -> 404
9b. Row found -> toPublic projection -> 200 PublicUser
10. Request logger emits userId on response finish in src/middlewares/request-logger.middleware.ts:14-24
```

### 2.4 Trust boundaries crossed

| Boundary | Crossing point | Validation applied |
|----------|----------------|--------------------|
| Anonymous network -> application | Express JSON parser and route handlers | Zod body schemas; 1 MB body limit |
| Application -> identity store | Prisma MySQL client | Parameterized queries via Prisma; unique email constraint |
| Client bearer credential -> request identity | `authenticate` middleware | JWT signature and expiry only; no algorithm pin, issuer, audience, or payload schema |
| Auth Module -> Users Module | `UserService` and `UserRepository` calls | None beyond method contracts; no ACL |
| Application -> password hashing | `bcrypt` in-process | Cost factor 10; 72-byte truncation |

---

## 3. Business Rules & Logic

### Overview of the business rules

| ID | Rule Type | Rule Description | Location |
|----|-----------|------------------|----------|
| BR-01 | Validation | Registration email must be a syntactically valid email of at most 255 characters | `src/modules/auth/auth.schemas.ts:4` |
| BR-02 | Validation | Registration password must contain between 8 and 72 characters | `src/modules/auth/auth.schemas.ts:5` |
| BR-03 | Validation | Registration name must contain between 2 and 150 characters | `src/modules/auth/auth.schemas.ts:6` |
| BR-04 | Domain constraint | Role must be one of `ADMIN` or `OPERATOR`; omission defaults to `OPERATOR` | `src/modules/auth/auth.schemas.ts:7`; `prisma/schema.prisma:11-14,30` |
| BR-05 | Access control | Registration is public and unauthenticated; the caller may choose any allowed role | `src/modules/auth/auth.routes.ts:10`; `src/modules/auth/auth.service.ts:27-29` |
| BR-06 | Uniqueness | An email address may identify at most one user | `src/modules/users/user.service.ts:22-25`; `prisma/schema.prisma:27` |
| BR-07 | Security | Passwords are persisted only as bcrypt hashes with cost factor 10 | `src/modules/users/user.service.ts:16,26`; `prisma/schema.prisma:28` |
| BR-08 | Data exposure | Auth responses never include `passwordHash` or plaintext password | `src/modules/users/user.service.ts:42-51`; `src/modules/auth/auth.controller.ts:15,24,34` |
| BR-09 | Authentication | Login requires an existing account and a matching password; both failure modes return the same generic 401 | `src/modules/auth/auth.service.ts:32-39` |
| BR-10 | Validation | Login accepts any non-empty password up to 72 characters | `src/modules/auth/auth.schemas.ts:10-13` |
| BR-11 | Token issuance | Successful login issues an HS256 JWT containing `sub`, `email`, `role`, `iat`, `exp`, with lifetime from `JWT_EXPIRES_IN` (default `8h`) | `src/modules/auth/auth.service.ts:40-44,47-50`; `src/config/env.ts:9` |
| BR-12 | API contract | Login returns `{ user, tokens: { accessToken, expiresIn, tokenType: "Bearer" } }`; `expiresIn` is the raw configured duration string | `src/modules/auth/auth.service.ts:10-19,41-44` |
| BR-13 | Authentication | Protected auth routes require a syntactically valid `Authorization: Bearer <token>` header | `src/middlewares/auth.middleware.ts:27-38`; `src/modules/auth/auth.routes.ts:12` |
| BR-14 | Authentication | A token is accepted only if its signature and expiry validate against `JWT_SECRET`; token claims are trusted as identity | `src/middlewares/auth.middleware.ts:40-46` |
| BR-15 | Identity resolution | `/auth/me` returns the current database user, not the token payload alone | `src/modules/auth/auth.controller.ts:30-38`; `src/modules/users/user.service.ts:36-40` |
| BR-16 | Authorization | Only `ADMIN` may call `GET /api/v1/users/:id`; other business routers require authentication but no role | `src/modules/users/user.routes.ts:12-18`; `src/modules/customers/customer.routes.ts:14`; `src/modules/products/product.routes.ts:14`; `src/modules/orders/order.routes.ts:14` |
| BR-17 | Session lifecycle | Tokens are stateless single-access tokens: no refresh, logout, revocation, denylist, or server-side session exists | `src/modules/auth/auth.service.ts:47-50`; `src/config/env.ts:8-9` |
| BR-18 | Abuse control | Auth endpoints have no rate limiting, lockout, CAPTCHA, or retry backoff | `src/modules/auth/auth.routes.ts:10-12`; `package.json:25-33` |
| BR-19 | Configuration | Startup fails unless `JWT_SECRET` is at least 16 characters; `JWT_EXPIRES_IN` defaults to `8h` | `src/config/env.ts:8-10,14-24` |
| BR-20 | Error contract | Auth failures map to `400 VALIDATION_ERROR`, `401 UNAUTHORIZED`, `404 NOT_FOUND`, or `409 EMAIL_ALREADY_USED`/`CONFLICT` in a uniform error envelope | `src/middlewares/error.middleware.ts:15-47`; `src/shared/errors/http-errors.ts:9-37` |
| BR-21 | Identity semantics | Email comparison is exact at the application layer and inherits database collation case rules; no normalization is performed | `src/modules/users/user.repository.ts:10-12`; `docker-compose.yml:23` |
| BR-22 | Account lifecycle | Accounts are immediately active; there is no email verification, activation flag, suspension, or password reset | `prisma/schema.prisma:25-38`; no corresponding route or service method exists |

---

### Detailed breakdown of the business rules

---

### Business Rule BR-01: Registration email format and length

**Overview**:
Every registration request must carry an email string that is a valid email address and no longer than 255 characters.

**Detailed description**:
The rule is implemented exclusively in `registerSchema` (`src/modules/auth/auth.schemas.ts:4`) with `z.string().email().max(255)`. The validation middleware parses the request body before the controller is invoked, and replaces `req.body` with the parsed output (`src/middlewares/validate.middleware.ts:14-16`). A failure produces `ValidationError` with per-field details, which the centralized error middleware renders as HTTP 400 with code `VALIDATION_ERROR` (`src/middlewares/error.middleware.ts:26-34`). The rule applies only to the transport boundary; no service-level revalidation exists, so a caller that bypasses Express and invokes `AuthService.register` directly would bypass it.

The 255-character ceiling mirrors `User.email @db.VarChar(255)` (`prisma/schema.prisma:27`). The rule does not trim whitespace, lowercase the address, or perform DNS/MX validation. Zod's `.email()` checks syntax only. Consequently, an address with leading or trailing whitespace is rejected rather than normalized, and an address that is syntactically valid but undeliverable is accepted. This is consistent with the absence of email verification (BR-22).

The rule is the first line of defense for the `users` table's unique email invariant. It prevents oversized payloads from reaching MySQL, but it does not prevent duplicate registration, which is BR-06.

**Rule workflow**:
```
POST /auth/register body.email
  -> validate middleware parses with z.string().email().max(255)
     -> invalid: 400 VALIDATION_ERROR with path "email"
     -> valid: req.body.email forwarded to AuthController -> AuthService -> UserService
```

---

### Business Rule BR-02: Registration password policy

**Overview**:
A registration password must be at least 8 characters and at most 72 characters.

**Detailed description**:
`registerSchema` declares `password: z.string().min(8).max(72)` (`src/modules/auth/auth.schemas.ts:5`). This is the only password policy in the system: there is no requirement for uppercase, lowercase, digits, symbols, or a blocklist. The minimum reduces trivially guessable passwords but does not enforce entropy. The maximum exists because bcrypt processes at most 72 bytes of input; the schema expresses it in characters, which is not equivalent for multi-byte UTF-8 input and can allow silent truncation above 72 bytes.

Enforcement occurs before hashing in `UserService.createUser` (`src/modules/users/user.service.ts:26`). The raw password is never persisted; only the bcrypt digest is written. Because `registerSchema` is the transport contract for registration, the policy is not re-applied inside `UserService.createUser` or `AuthService.register`; `CreateUserInput` is a compile-time type only (`src/modules/users/user.schemas.ts:14`). Any future non-HTTP caller must enforce the policy itself.

The policy applies to registration only. Login deliberately uses a weaker contract (BR-10) so that seeded or legacy accounts with shorter passwords can still authenticate, although `prisma/seed.ts:6-7` uses 8+-character passwords (`admin123`, `operator123`).

**Rule workflow**:
```
Register body.password
  -> min 8 and max 72 check in registerSchema
     -> fail: 400 VALIDATION_ERROR details[]
     -> pass: bcrypt.hash(password, 10) -> passwordHash persisted
```

---

### Business Rule BR-03: Registration name length

**Overview**:
A registration name must be between 2 and 150 characters.

**Detailed description**:
`registerSchema` applies `z.string().min(2).max(150)` (`src/modules/auth/auth.schemas.ts:6`). The upper bound matches the database column `User.name @db.VarChar(150)` (`prisma/schema.prisma:29`). The lower bound prevents empty or one-character display names. No character-set restriction, trimming, or normalization is applied, so whitespace-only names of sufficient length (for example two spaces) pass both Zod and MySQL.

The value is stored verbatim by `UserService.createUser` (`src/modules/users/user.service.ts:27-32`) and projected by `toPublic` (`src/modules/users/user.service.ts:42-51`). It is not used for authentication or authorization decisions; it is display data only. No uniqueness constraint exists on name.

Because the rule is transport-only, direct service calls can persist names that violate the database length, which would then fail at the MySQL layer with a Prisma error and surface as HTTP 500 unless the error matches a handled Prisma code.

**Rule workflow**:
```
Register body.name
  -> 2..150 character check
     -> fail: 400 VALIDATION_ERROR
     -> pass: persisted as User.name
```

---

### Business Rule BR-04: Role enumeration and default

**Overview**:
A user role is restricted to the enum values `ADMIN` and `OPERATOR`, and defaults to `OPERATOR` when omitted.

**Detailed description**:
The rule exists in three places. At the transport layer, `registerSchema` uses `z.enum(['ADMIN', 'OPERATOR']).default('OPERATOR')` (`src/modules/auth/auth.schemas.ts:7`), so an omitted or `undefined` role becomes `OPERATOR` during parsing. At the domain/persistence layer, Prisma defines `enum UserRole { ADMIN OPERATOR }` and `role UserRole @default(OPERATOR)` (`prisma/schema.prisma:11-14,30`). At the type layer, `PublicUser.role` and the JWT role claim are typed as the union `'ADMIN' | 'OPERATOR'` (`src/modules/users/user.service.ts:11`; `src/modules/auth/auth.service.ts:47`). A value outside the enum is rejected with `400 VALIDATION_ERROR` before any database work.

The default is applied by Zod before the controller runs, and again by MySQL if a row were inserted without a role. The service passes `input.role` through unchanged (`src/modules/users/user.service.ts:31`), so the schema default is the effective business default for HTTP registration. There is no way to represent a disabled or unprivileged role; every account is either an operator or an administrator.

Role is a security-relevant attribute because `requireRole` reads it from the JWT (`src/middlewares/auth.middleware.ts:49-61`) and the only currently role-guarded route is `GET /api/v1/users/:id` (`src/modules/users/user.routes.ts:12-18`). The enum therefore defines the complete authorization vocabulary of the system.

**Rule workflow**:
```
Register body.role (optional)
  -> enum check ADMIN|OPERATOR
     -> invalid: 400 VALIDATION_ERROR
     -> omitted: replaced with "OPERATOR"
     -> valid: persisted; later embedded in JWT claim "role"
```

---

### Business Rule BR-05: Public registration with caller-selected role

**Overview**:
`POST /api/v1/auth/register` is reachable without authentication and accepts the role from the request body, including `ADMIN`.

**Detailed description**:
The route composes only `validate({ body: registerSchema })` and the controller (`src/modules/auth/auth.routes.ts:10`); there is no `authenticate`, no `requireRole`, no invitation token, no domain allowlist, and no administrative approval. `AuthService.register` is a direct delegation to `UserService.createUser(input)` (`src/modules/auth/auth.service.ts:27-29`), and `UserService.createUser` persists `input.role` unchanged (`src/modules/users/user.service.ts:27-32`). Therefore an anonymous caller can submit `{ "role": "ADMIN" }` and receive a 201 response for a fully privileged account.

This is the implemented behavior, not a hypothetical. It is corroborated by the integration test that registers with an explicit `OPERATOR` role (`tests/auth.test.ts:6-17`) and by the architectural risk register. The business consequence is that the role-based guard on `GET /api/v1/users/:id` and any future `requireRole` usage can be satisfied by self-registration. The intended rule, if one exists, cannot be determined because the PRD/FDD are empty templates.

The only practical constraints on this rule are email uniqueness (BR-06), payload validation (BR-01 to BR-04), and the absence of rate limiting (BR-18). There is no audit trail recording who created an account or with which role; `User` has only `createdAt`/`updatedAt`.

**Rule workflow**:
```
Anonymous POST /auth/register { role: "ADMIN" }
  -> no authenticate / requireRole middleware
  -> schema accepts ADMIN
  -> UserService.createUser persists role ADMIN
  -> 201 PublicUser { role: "ADMIN" }
  -> subsequent login issues JWT with role ADMIN
```

---

### Business Rule BR-06: Email uniqueness per user

**Overview**:
At most one user row may exist for a given email address.

**Detailed description**:
The invariant is enforced twice. First, `UserService.createUser` calls `users.findByEmail(input.email)` and throws `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` when a row exists (`src/modules/users/user.service.ts:22-25`). Second, MySQL enforces `email String @unique @db.VarChar(255)` (`prisma/schema.prisma:27`), which is the authoritative constraint. The pre-check provides a friendly, specific error code; the database constraint protects against concurrent registrations that both pass the pre-check.

There is a time-of-check/time-of-use window between the `SELECT` and the `INSERT`. If two requests race, one insert succeeds and the other raises Prisma `P2002`. The centralized error middleware maps `P2002` to HTTP 409 with code `CONFLICT` and a message naming the constrained column (`src/middlewares/error.middleware.ts:37-47`), not to `EMAIL_ALREADY_USED`. Clients must therefore handle both `EMAIL_ALREADY_USED` and `CONFLICT` for the same business condition. The test suite covers the sequential duplicate path only (`tests/auth.test.ts:19-28`).

Email uniqueness is the account identity key used by login (`findByEmail`) and by the database. Because no normalization is performed (BR-21), whether `User@Example.com` and `user@example.com` collide depends on the column collation. In the provided Compose configuration the server collation is `utf8mb4_unicode_ci` (`docker-compose.yml:23`), which is case-insensitive, so they collide in that environment but might not under a different deployment configuration.

**Rule workflow**:
```
Register
  -> SELECT user by email
     -> found: 409 EMAIL_ALREADY_USED
     -> not found: bcrypt hash -> INSERT
        -> unique violation (race): 409 CONFLICT (Prisma P2002 mapping)
        -> success: 201
```

---

### Business Rule BR-07: Password hashing at rest

**Overview**:
Plaintext passwords are never stored; the persisted value is a bcrypt hash with cost factor 10.

**Detailed description**:
`UserService` defines `const BCRYPT_ROUNDS = 10` (`src/modules/users/user.service.ts:16`) and hashes with `bcrypt.hash(input.password, BCRYPT_ROUNDS)` before calling the repository (`src/modules/users/user.service.ts:26-32`). The database column is named `passwordHash` and typed `VarChar(255)` (`prisma/schema.prisma:28`), making the storage contract explicit. The plaintext exists only in the request body and in the function argument; it is not logged by the application's request logger, which records only method, path, status, duration and `userId` (`src/middlewares/request-logger.middleware.ts:14-24`).

Hashing is performed in the Users Module, not in the Auth Module. The Auth Module's registration path delegates to `UserService.createUser` precisely so that there is one hashing implementation. Login never hashes; it uses `bcrypt.compare` against the stored digest (`src/modules/auth/auth.service.ts:36`), which is a constant-time comparison inside the library. Cost 10 is a deliberate trade-off between resistance to offline brute force and login latency; it is not configurable through environment variables.

The bcrypt algorithm truncates input beyond 72 bytes. The schemas cap at 72 characters (BR-02), which is stricter than 72 bytes for ASCII but not for multi-byte characters. Two passwords sharing the first 72 bytes hash identically. No pepper, per-user salt management, or hash upgrade-on-login logic exists; bcrypt generates and embeds a random salt per hash.

**Rule workflow**:
```
Registration password (plaintext)
  -> bcrypt.hash(password, 10)
  -> digest stored in users.passwordHash
  -> plaintext discarded
Login password (plaintext)
  -> bcrypt.compare(plaintext, users.passwordHash)
  -> boolean result drives 200 or 401
```

---

### Business Rule BR-08: No credential material in responses

**Overview**:
Auth endpoints expose only the `PublicUser` projection and tokens; they never return `passwordHash` or the plaintext password.

**Detailed description**:
`UserService.toPublic` constructs an explicit object with `id`, `email`, `name`, `role`, `createdAt`, `updatedAt` (`src/modules/users/user.service.ts:42-51`). Both `AuthService.login` (`src/modules/auth/auth.service.ts:42`) and `AuthController.me` (`src/modules/auth/auth.controller.ts:33-34`) return that projection. `register` returns the result of `AuthService.register`, which is the same projection (`src/modules/auth/auth.service.ts:27-29`). No controller returns a raw Prisma `User` object.

The rule is a whitelist, not a blacklist: new sensitive columns added to `User` would not leak automatically because `toPublic` enumerates fields. However, the projection is duplicated in the type declaration `PublicUser` and the function body, so a schema change requires both to be updated. The integration suite explicitly asserts absence of `passwordHash` and `password` in the registration response (`tests/auth.test.ts:15-16`), which is the strongest automated protection for this rule.

The token itself is credential material and is intentionally returned by login. It is not returned by `/me`. There is no response field for refresh tokens because none exist. There is also no cache-control or no-store header on auth responses, so intermediary caching behavior is unspecified.

**Rule workflow**:
```
Prisma User row
  -> UserService.toPublic whitelist
  -> { id, email, name, role, createdAt, updatedAt }
  -> response JSON (register 201 / login user field / me 200)
```

---

### Business Rule BR-09: Credential verification with indistinguishable failures

**Overview**:
Login succeeds only when the email belongs to an existing user and the supplied password matches the stored hash. Unknown email and wrong password both yield `401 UNAUTHORIZED` with the message `Invalid credentials`.

**Detailed description**:
`AuthService.login` first calls `users.findByEmail(input.email)` (`src/modules/auth/auth.service.ts:32`). If no row is found it throws `new UnauthorizedError('Invalid credentials')` (`src/modules/auth/auth.service.ts:33-35`). If a row is found it calls `bcrypt.compare(input.password, user.passwordHash)` and throws the same error type and message on mismatch (`src/modules/auth/auth.service.ts:36-39`). The shared `UnauthorizedError` maps to status 401 and code `UNAUTHORIZED` (`src/shared/errors/http-errors.ts:15-19`), which `errorMiddleware` serializes into the standard error envelope (`src/middlewares/error.middleware.ts:15-24`).

The identical external response prevents straightforward account enumeration through status code or message. It does not equalize timing: the unknown-email path returns before any bcrypt work, while the known-email path performs a cost-10 comparison. An observer able to measure response latency could infer whether an account exists. There is no dummy hash comparison to normalize this. The integration test covers the wrong-password branch (`tests/auth.test.ts:54-63`) but not the unknown-email branch.

The rule does not check account status because there is no status field (BR-22). Any row that exists is eligible to log in, regardless of role, age, or administrative intent. It also does not invalidate previous tokens or record the login event.

**Rule workflow**:
```
Login email/password
  -> findByEmail
     -> null: 401 UNAUTHORIZED "Invalid credentials"
     -> user: bcrypt.compare
        -> false: 401 UNAUTHORIZED "Invalid credentials"
        -> true: issue JWT -> 200
```

---

### Business Rule BR-10: Login password acceptance range

**Overview**:
Login accepts any non-empty password string up to 72 characters, independent of the registration minimum.

**Detailed description**:
`loginSchema` declares `password: z.string().min(1).max(72)` (`src/modules/auth/auth.schemas.ts:12`). The looser minimum avoids rejecting credentials created outside the registration flow, such as seeded users or migrated accounts, before bcrypt can evaluate them. The 72-character maximum matches the registration ceiling and the bcrypt input limit. A password shorter than 8 characters is accepted by validation and then fails at `bcrypt.compare` unless it genuinely matches a stored hash.

Because `loginSchema` and `registerSchema` are separate declarations, the two policies can drift. The login policy intentionally reveals nothing about the stored password's complexity; it is an acceptance envelope, not a policy statement. An empty password is rejected with 400 before any database lookup, so it cannot be used to probe password hashes.

This rule interacts with BR-09: schema-valid but incorrect credentials always produce 401, while schema-invalid credentials produce 400. That distinction lets a caller differentiate malformed requests from failed authentication, which is normal HTTP behavior but is also a small oracle about request shape, not account existence.

**Rule workflow**:
```
Login body.password
  -> 1..72 character check
     -> fail: 400 VALIDATION_ERROR
     -> pass: bcrypt.compare against stored hash
```

---

### Business Rule BR-11: JWT issuance and lifetime

**Overview**:
A successful login signs a stateless HS256 JSON Web Token containing the user id, email and role, expiring after the configured `JWT_EXPIRES_IN` duration (default `8h`).

**Detailed description**:
`AuthService.signToken` builds the payload `{ sub: userId, email, role }` and calls `jwt.sign(payload, env.JWT_SECRET, options)` with `expiresIn: env.JWT_EXPIRES_IN` (`src/modules/auth/auth.service.ts:47-50`). `jsonwebtoken` adds `iat` and `exp` automatically. Because the secret is a string, the library selects HMAC; the effective algorithm is HS256 by default. The token is returned as `accessToken` together with `expiresIn` and `tokenType: 'Bearer'` (`src/modules/auth/auth.service.ts:43`). No `jti`, `iss`, `aud`, `nbf`, or custom session identifier is added.

Lifetime is controlled by `JWT_EXPIRES_IN`, whose schema default is `'8h'` (`src/config/env.ts:9`). The value is cast to `SignOptions['expiresIn']` rather than parsed or validated as a duration, so an invalid duration string is accepted by configuration loading and only fails at signing time, producing an unhandled error and HTTP 500 on login. The response exposes the raw duration string, not an absolute expiry timestamp or a number of seconds, so clients must know how to interpret values such as `8h`.

The token is a bearer credential: possession alone grants access. It is not bound to a device, IP, user agent, or TLS channel. Since there is no server-side session record (BR-17), the signature and expiry are the only validity signals the system can check.

**Rule workflow**:
```
Valid credentials
  -> payload { sub, email, role }
  -> jwt.sign(payload, JWT_SECRET, { expiresIn: JWT_EXPIRES_IN })
  -> token with iat/exp
  -> response { accessToken, expiresIn: "8h", tokenType: "Bearer" }
```

---

### Business Rule BR-12: Login response contract

**Overview**:
Login returns HTTP 200 with a `user` object and a `tokens` object containing `accessToken`, `expiresIn`, and the literal `tokenType: "Bearer"`.

**Detailed description**:
The shapes are declared as `AuthTokens` and `LoginResult` (`src/modules/auth/auth.service.ts:10-19`) and constructed in `login` (`src/modules/auth/auth.service.ts:41-44`). The controller serializes the object directly with `res.status(200).json(result)` (`src/modules/auth/auth.controller.ts:24`), without an envelope such as `{ data: ... }`. The `user` member is a `PublicUser`, so it includes timestamps serialized as ISO strings by Express. `tokenType` is a literal type, not configurable.

The integration test asserts the token is a string, `tokenType` is `Bearer`, and the user email matches (`tests/auth.test.ts:48-51`). It does not assert `expiresIn`, the user id, role, or the absence of extra fields. Because the contract is implicit in code and not documented in the empty FDD, consumers have no schema artifact to validate against.

This response is the only place the access token is emitted. There is no `Set-Cookie`, so browser clients must store the token themselves and send it in the `Authorization` header. No CORS configuration exists in `src/app.ts`, which constrains browser-based consumption.

**Rule workflow**:
```
200 OK
{
  "user": { id, email, name, role, createdAt, updatedAt },
  "tokens": { "accessToken": "<jwt>", "expiresIn": "8h", "tokenType": "Bearer" }
}
```

---

### Business Rule BR-13: Bearer header requirement

**Overview**:
`/auth/me` and every router that applies `authenticate` require an `Authorization` header beginning with `Bearer ` followed by a non-empty token.

**Detailed description**:
`authenticate` reads `req.headers.authorization` and rejects a missing header or any scheme other than Bearer, case-insensitively, with `UnauthorizedError('Missing or invalid Authorization header')` (`src/middlewares/auth.middleware.ts:28-32`). It then extracts `header.slice(7).trim()` and rejects an empty result with `UnauthorizedError('Missing bearer token')` (`src/middlewares/auth.middleware.ts:34-38`). Both cases produce 401 through `errorMiddleware`. The check is purely syntactic at this stage; cryptographic validation happens next.

The middleware is applied per route or router. The Auth Module uses it only on `/me` (`src/modules/auth/auth.routes.ts:12`). Customers, products and orders apply it with `router.use(authenticate)` (`src/modules/customers/customer.routes.ts:14`, `src/modules/products/product.routes.ts:14`, `src/modules/orders/order.routes.ts:14`), and users applies it before the role guard (`src/modules/users/user.routes.ts:14-15`). This makes the middleware a system-wide dependency even though the Auth Module does not own it.

The scheme check accepts unusual but valid forms such as `bearer` in lowercase and extra whitespace inside the token after trimming. It does not validate the token format (three dot-separated segments) before calling `jwt.verify`; malformed values fail verification and return the generic invalid-token error. There is no support for cookies, query-string tokens, or other schemes.

**Rule workflow**:
```
Authorization header
  -> absent or scheme != Bearer: 401 "Missing or invalid Authorization header"
  -> token empty after slicing: 401 "Missing bearer token"
  -> token present: jwt.verify
```

---

### Business Rule BR-14: Token verification and claim trust

**Overview**:
A bearer token is accepted only if `jwt.verify(token, JWT_SECRET)` succeeds; on success, the decoded `sub`, `email` and `role` become `req.user` without further validation.

**Detailed description**:
`jwt.verify(token, env.JWT_SECRET)` is called inside a try/catch (`src/middlewares/auth.middleware.ts:40-46`). Any verification error, including bad signature, malformed token, or expiry, is collapsed into `UnauthorizedError('Invalid or expired token')`. On success, the payload is cast to the local `JwtPayload` type (`src/middlewares/auth.middleware.ts:19-25`) and assigned to `req.user` as `{ id: payload.sub, email: payload.email, role: payload.role }`. TypeScript casting performs no runtime checks.

The verification call supplies no options object, so there is no explicit `algorithms` allowlist, no `issuer`, no `audience`, no clock tolerance, and no `maxAge`. The library's default behavior with a string secret is to accept HMAC algorithms; the system assumes HS256 because that is what `sign` produces, but the verifier does not assert it. There is also no structural validation that `sub` is a UUID, that `email` is an email, or that `role` is one of the two enum values. A token signed with the same secret but missing claims would hydrate `req.user` with `undefined` fields, and `/me` would then pass `undefined` to Prisma.

The claims are trusted for the lifetime of the token. No database lookup occurs during authentication, so a deleted user or a downgraded role remains effective until expiry. This is a deliberate stateless-design consequence (BR-17), not an oversight in the middleware alone.

**Rule workflow**:
```
Bearer token
  -> jwt.verify(token, JWT_SECRET)
     -> throws (signature/expiry/format): 401
     -> succeeds: req.user = { id: sub, email, role }
        -> no schema check, no algorithm pin, no DB check
```

---

### Business Rule BR-15: Current identity resolved from the database

**Overview**:
`GET /api/v1/auth/me` returns the `PublicUser` row identified by the token's `sub`, not the token payload itself.

**Detailed description**:
After `authenticate` populates `req.user`, `AuthController.me` defensively checks `req.user` and throws `UnauthorizedError` if absent (`src/modules/auth/auth.controller.ts:32`). It then calls `this.userService.getById(req.user.id)` (`src/modules/auth/auth.controller.ts:33`). `UserService.getById` queries `UserRepository.findById` and throws `NotFoundError('User')` when the row is gone (`src/modules/users/user.service.ts:36-40`), which maps to HTTP 404 with code `NOT_FOUND`. On success it returns the `toPublic` projection with status 200.

This rule gives `/me` fresher data than the token for `email`, `name`, `role` and timestamps, but only when the row still exists. It does not re-check authorization or account status; it is a read of the identity row, not a session validation endpoint. Deleted users therefore produce a 404, which is semantically different from an invalid credential (401) and may surprise clients that use `/me` to test whether their token is still usable.

The controller's direct dependency on `UserService` (rather than `AuthService`) is the reason the Auth Module has a compile-time dependency on the Users Module in two directions: `AuthService` depends on `UserRepository` and `UserService`, and `AuthController` depends on `UserService`. This is documented in the coupling analysis.

**Rule workflow**:
```
GET /auth/me with valid token
  -> req.user.id = payload.sub
  -> UserService.getById(id)
     -> row missing: 404 NOT_FOUND "User not found"
     -> row found: 200 PublicUser
```

---

### Business Rule BR-16: Role-based authorization boundary

**Overview**:
Authentication is required for all customer, product and order routes, but only `GET /api/v1/users/:id` additionally requires the `ADMIN` role.

**Detailed description**:
`requireRole(...roles)` returns middleware that rejects an absent `req.user` with 401 and a non-matching role with `ForbiddenError('Insufficient permissions')`, mapped to 403 (`src/middlewares/auth.middleware.ts:49-61`; `src/shared/errors/http-errors.ts:21-25`). It is used exactly once, on the user lookup route, after `authenticate` and before parameter validation (`src/modules/users/user.routes.ts:12-18`). Role comes exclusively from the JWT claim, never from a database read.

The Auth Module itself exposes no role-protected endpoint. Its registration endpoint is public (BR-05), login is public, and `/me` requires only authentication. The broader system rule is therefore "authenticated but not necessarily authorized": any valid token can create, read, update and delete customers, products and orders. There is no tenant, owner, or customer scoping in those routers.

Because registration can mint `ADMIN` tokens (BR-05), the role guard does not currently provide a meaningful separation between administrators and self-registered users. The rule is enforced correctly at the middleware level; its practical strength depends on the integrity of role assignment.

**Rule workflow**:
```
Request to protected route
  -> authenticate -> req.user
  -> requireRole('ADMIN') on /users/:id
     -> no user: 401
     -> role != ADMIN: 403 FORBIDDEN
     -> role == ADMIN: controller
  -> other business routers: no requireRole -> any authenticated role
```

---

### Business Rule BR-17: Stateless session lifecycle

**Overview**:
Sessions are stateless JWTs with a fixed lifetime; there is no refresh token, logout, revocation, denylist, or server-side session store.

**Detailed description**:
`signToken` is the only token-creation path (`src/modules/auth/auth.service.ts:47-50`), and no code persists or tracks the resulting token. There is no `POST /auth/logout`, no `POST /auth/refresh`, no `jti` claim, and no Redis or database table for sessions. `JWT_EXPIRES_IN` defaults to `8h` (`src/config/env.ts:9`), so a stolen token is usable for up to eight hours unless the secret is rotated globally.

The consequences are architectural. Horizontal scaling is easy because any instance with the secret can verify tokens. Conversely, individual revocation is impossible: deleting a user does not invalidate their token (though `/me` returns 404), changing a role does not affect an already-issued token, and password changes (which have no endpoint) would not either. `JWT_SECRET` is a single symmetric key with a minimum length of 16 characters (`src/config/env.ts:8`); rotating it invalidates every token at once.

There is no logout endpoint to clear client state server-side, so clients must discard tokens themselves. The absence of refresh tokens also means users must re-authenticate every eight hours. These are product decisions that cannot be confirmed against written requirements because the PRD/FDD are placeholders.

**Rule workflow**:
```
Login -> signed JWT (no server record)
Every request -> signature + exp check only
Logout/refresh/revoke -> no endpoint, no state change
Expiry -> token rejected by jwt.verify
Secret rotation -> all tokens invalid
```

---

### Business Rule BR-18: No abuse controls on auth endpoints

**Overview**:
Registration and login have no rate limiting, account lockout, CAPTCHA, IP throttling, or exponential backoff.

**Detailed description**:
The routes are declared directly on an Express `Router` with only validation and, for `/me`, authentication (`src/modules/auth/auth.routes.ts:10-12`). `src/app.ts:55-75` installs JSON parsing, request logging, the API router, a 404 handler and the error middleware, but no rate-limiting or security-header middleware. `package.json` contains no rate-limit, helmet, or CAPTCHA dependency (`package.json:25-33`). No counter, lock flag, or `failedLoginAttempts` column exists on `User` (`prisma/schema.prisma:25-38`).

This means an attacker can attempt passwords indefinitely, subject only to network and bcrypt CPU cost, and can create unlimited accounts, including `ADMIN` accounts (BR-05). The cost-10 bcrypt comparison provides some slowdown per attempt, but no threshold or lockout. The login error is generic (BR-09), which prevents easy enumeration but does not slow brute force. Registration performs a bcrypt hash per successful unique email, so it is also a CPU amplification vector.

No rule expresses an intended limit, so this is documented as an absent control rather than a violated threshold. The practical impact is higher because the system also lacks TLS termination, WAF, or reverse proxy configuration in the repository (`docker-compose.yml` exposes MySQL directly; no proxy service is defined).

**Rule workflow**:
```
Any number of POST /auth/login or /auth/register
  -> no counter, no delay, no block
  -> each request processed independently
```

---

### Business Rule BR-19: Authentication configuration preconditions

**Overview**:
The process refuses to start unless `DATABASE_URL` is present and `JWT_SECRET` is at least 16 characters; `JWT_EXPIRES_IN` defaults to `8h`.

**Detailed description**:
`envSchema` validates `JWT_SECRET: z.string().min(16)` and `JWT_EXPIRES_IN: z.string().default('8h')` (`src/config/env.ts:8-9`). `loadEnv` uses `safeParse` and calls `process.exit(1)` after printing a human-readable list of issues when validation fails (`src/config/env.ts:14-24`). The module executes at import time, so any entry point that imports `env` (including the Auth Module, the auth middleware, the database client, and the server) inherits fail-fast behavior. This prevents the application from signing or verifying tokens with an empty or trivially short secret.

The 16-character minimum is a floor, not a strength guarantee. The example configuration supplies a long placeholder (`change-me-in-production-please-use-a-long-random-secret`, `.env.example:8`), but there is no check for entropy, no key version, and no support for key rotation. A 16-character low-entropy string passes. The same secret signs and verifies all tokens across all environments where it is reused.

`JWT_EXPIRES_IN` is typed as a free string, so invalid duration syntax is not caught at startup; it surfaces when `jwt.sign` is called during login. The schema does not import a dotenv loader; environment variables are supplied by the npm scripts (`--env-file=.env` in `package.json:11,13,16`) or by the test/runtime environment. Under `npm test`, no `--env-file` flag is present (`package.json:17`), so the test process depends on externally exported variables.

**Rule workflow**:
```
Process start / module import
  -> envSchema.safeParse(process.env)
     -> DATABASE_URL missing or JWT_SECRET < 16: print issues, exit(1)
     -> valid: export frozen env object
  -> login later uses JWT_EXPIRES_IN as-is in jwt.sign
```

---

### Business Rule BR-20: Uniform auth error contract

**Overview**:
Auth failures are serialized into a consistent error envelope with a machine-readable code, a human message, and optional details; status codes are 400, 401, 404, 409 or 500.

**Detailed description**:
Domain errors extend `AppError`, which carries `statusCode`, `errorCode` and `details` (`src/shared/errors/app-error.ts:3-15`). Relevant subclasses are `ValidationError` (400, `VALIDATION_ERROR`), `UnauthorizedError` (401, `UNAUTHORIZED`), `ForbiddenError` (403, `FORBIDDEN`), `NotFoundError` (404, `NOT_FOUND`) and `ConflictError` (409, configurable code) (`src/shared/errors/http-errors.ts:9-37`). `errorMiddleware` checks `instanceof AppError` and emits `{ error: { code, message, details? } }` (`src/middlewares/error.middleware.ts:15-24`). Zod errors that somehow bypass the validation middleware are also mapped to 400 with per-issue details (`src/middlewares/error.middleware.ts:26-34`).

The module uses specific codes for its business conditions: `EMAIL_ALREADY_USED` for the sequential duplicate path (`src/modules/users/user.service.ts:24`) and `UNAUTHORIZED` for both unknown email and wrong password (`src/modules/auth/auth.service.ts:34,38`). The concurrent duplicate path produces generic `CONFLICT` via the Prisma `P2002` mapping (`src/middlewares/error.middleware.ts:37-47`), which is an exception to the otherwise specific contract. Unhandled errors are logged with a request id and returned as `500 INTERNAL_SERVER_ERROR` without leaking internals (`src/middlewares/error.middleware.ts:56-64`).

The envelope is not documented in an OpenAPI specification or the FDD. The tests assert `error.code` and the presence of `details` for validation (`tests/auth.test.ts:36-38`), establishing a de facto contract. Error responses do not include the request id in the body, only in the `X-Request-Id` response header set by the logger.

**Rule workflow**:
```
Thrown error
  -> AppError: statusCode + code + message (+ details)
  -> ZodError: 400 VALIDATION_ERROR with issue paths
  -> Prisma P2002: 409 CONFLICT
  -> Prisma P2025: 404 NOT_FOUND
  -> other: log + 500 INTERNAL_SERVER_ERROR
```

---

### Business Rule BR-21: Email identity semantics

**Overview**:
Email is the login identifier; the application performs exact-string lookup and no case folding or trimming, so effective case sensitivity is delegated to the database collation.

**Detailed description**:
`AuthService.login` calls `users.findByEmail(input.email)` (`src/modules/auth/auth.service.ts:32`), which executes `prisma.user.findUnique({ where: { email } })` (`src/modules/users/user.repository.ts:10-12`). The same method is used for duplicate detection during registration (`src/modules/users/user.service.ts:22`). Neither layer normalizes the address: `registerSchema` validates syntax but does not `.trim().toLowerCase()`, and `UserService.createUser` stores `input.email` verbatim.

The database column is `@unique` with `VarChar(255)` (`prisma/schema.prisma:27`). In the provided `docker-compose.yml`, MySQL runs with `--collation-server=utf8mb4_unicode_ci` (`docker-compose.yml:23`), a case-insensitive collation. Under that configuration, `User@Example.com` and `user@example.com` are the same key: the second registration is rejected and login succeeds regardless of case. If the deployment uses a case-sensitive or binary collation, the same inputs become distinct accounts. This environment-dependent behavior is not documented in application code.

Whitespace is not normalized either: `" user@example.com "` is rejected by Zod's `.email()` rather than trimmed, so users must submit a clean address. Unicode normalization (NFC/NFKC) is not applied, so visually identical addresses with different code-point sequences can theoretically be distinct.

**Rule workflow**:
```
Registration/login email string
  -> Zod syntax check (no trim/lowercase)
  -> Prisma equality lookup
  -> MySQL collation decides case sensitivity
  -> unique index enforces one row per collation-equivalent value
```

---

### Business Rule BR-22: Immediate account activation with no lifecycle controls

**Overview**:
Every created account is immediately usable; there is no email verification, activation flag, suspension, password reset, or account deletion endpoint in the Auth Module.

**Detailed description**:
The `User` model has only `id`, `email`, `passwordHash`, `name`, `role`, `createdAt` and `updatedAt` (`prisma/schema.prisma:25-38`). There is no `emailVerified`, `status`, `disabled`, `lastLoginAt`, or `failedLoginAttempts` field. `UserService.createUser` writes the row and returns it without any verification step (`src/modules/users/user.service.ts:21-34`). `AuthService.login` checks only existence and password (BR-09), so a newly registered account can log in immediately.

The module exposes no password change or reset route. The only routes are register, login and me (`src/modules/auth/auth.routes.ts:10-12`). There is no email-sending dependency in `package.json`, no token table for reset links, and no administrative user-management endpoints beyond `GET /users/:id`. Account deletion is not exposed at all in the Users Module (`src/modules/users/user.routes.ts:12-18`), so the only way a user row disappears is direct database manipulation or test cleanup.

This rule is a significant determinant of the security posture: it makes registration abuse immediately exploitable, removes the recovery path for locked-out users, and means "deactivated" users cannot exist. Because DR requirements are blank, it is impossible to say whether this is intentional scope reduction or an unimplemented requirement.

**Rule workflow**:
```
Register -> user row created -> active immediately
Login -> existence + password check -> token
Lifecycle actions (verify, suspend, reset, delete) -> no endpoints or fields
```

---

## 4. Component Structure

```
src/modules/auth/                      Auth Module boundary (4 files, 121 lines)
├── auth.routes.ts                     # Express Router factory; composes validate/authenticate middleware; 3 routes (15 lines)
├── auth.controller.ts                 # Transport adapter; register/login/me handlers; delegates to services (39 lines)
├── auth.service.ts                    # Authentication core; credential check, bcrypt compare, JWT signing, DTO types (51 lines)
└── auth.schemas.ts                    # Zod request contracts; registerSchema, loginSchema, inferred input types (16 lines)
```

Directly coupled files outside the boundary:

```
src/modules/users/
├── user.service.ts                    # createUser, getById, toPublic, bcrypt cost 10 (auth registration dependency)
├── user.repository.ts                 # findByEmail, findById, create via Prisma
├── user.schemas.ts                    # Duplicate createUserSchema contract
└── user.routes.ts                     # Only consumer of requireRole('ADMIN')

src/middlewares/
├── auth.middleware.ts                 # authenticate (JWT verify) and requireRole; consumed by auth.routes and 4 other routers
├── validate.middleware.ts             # Zod parse -> req.body/query/params; converts ZodError to ValidationError
└── error.middleware.ts                # AppError/Zod/Prisma error translation to HTTP envelope

src/config/env.ts                      # JWT_SECRET (min 16), JWT_EXPIRES_IN (default 8h), fail-fast parsing
src/shared/errors/                     # AppError, UnauthorizedError, ConflictError, NotFoundError, ...
src/routes/index.ts                    # Mounts buildAuthRouter at /api/v1/auth
src/app.ts                             # Composition root: instantiates repository, service, controller
prisma/schema.prisma                   # User model, UserRole enum, unique email index
tests/auth.test.ts                     # 7 integration tests for the module
tests/helpers/factories.ts             # createTestUser, loginAndGetToken, bootstrapAuthenticatedUser
tests/orders.test.ts                   # 8 order tests that obtain tokens through /auth/login
```

File-level responsibilities:

| File | Public surface | Internal dependencies | Notes |
|------|----------------|-----------------------|-------|
| `src/modules/auth/auth.routes.ts` | `buildAuthRouter(controller)` | `validate`, `authenticate`, `registerSchema`, `loginSchema`, `AuthController` | Factory function, no module-level state |
| `src/modules/auth/auth.controller.ts` | `AuthController` class with `register`, `login`, `me` `RequestHandler` fields | `AuthService` (type), `UserService` (type), `UnauthorizedError` | Arrow-function handlers preserve `this`; each method forwards errors to `next` |
| `src/modules/auth/auth.service.ts` | `AuthService` class, `AuthTokens`, `LoginResult` types | `bcrypt`, `jsonwebtoken`, `env`, `UnauthorizedError`, `UserRepository`, `UserService`, `PublicUser`, `LoginInput`, `RegisterInput` | Owns credential verification and token signing; only file with cryptographic logic |
| `src/modules/auth/auth.schemas.ts` | `registerSchema`, `loginSchema`, `RegisterInput`, `LoginInput` | `zod` | Contracts only, no runtime side effects |

---

## 5. Dependency Analysis

### Internal dependency graph

```
src/app.ts (composition root)
  ├── new UserRepository(prisma)
  ├── new UserService(userRepository)
  ├── new AuthService(userRepository, userService)
  ├── new AuthController(authService, userService)
  └── buildApiRouter({ auth: authController, ... })

src/routes/index.ts
  └── buildAuthRouter(controllers.auth)
        ├── validate (src/middlewares/validate.middleware.ts)
        │     └── ValidationError (src/shared/errors)
        ├── authenticate (src/middlewares/auth.middleware.ts)
        │     ├── env (src/config/env.ts)
        │     └── UnauthorizedError / ForbiddenError (src/shared/errors)
        ├── registerSchema / loginSchema (src/modules/auth/auth.schemas.ts)
        └── AuthController
              ├── AuthService
              │     ├── bcrypt (external)
              │     ├── jsonwebtoken (external)
              │     ├── env (src/config/env.ts)
              │     ├── UnauthorizedError (src/shared/errors)
              │     ├── UserRepository
              │     │     └── @prisma/client (external) -> MySQL
              │     ├── UserService
              │     │     └── bcrypt (external) + ConflictError/NotFoundError
              │     └── auth.schemas types
              ├── UserService (direct, for /me)
              └── UnauthorizedError (src/shared/errors)
```

### Dependency direction and cycles

- The Auth Module depends on the Users Module; the Users Module does not import the Auth Module. There is no cycle at module level (confirmed by static import inspection).
- The Auth Module depends on middleware and shared error modules, which depend on `config` and `shared`. Those lower layers do not depend on feature modules.
- `src/app.ts` is the only place where concrete instances are wired; the module itself uses constructor injection and type-only imports for its collaborators.
- The authentication middleware is a lateral dependency: the Auth Module uses it, but customers, products, orders and users routers depend on it independently.

### External dependencies used by the component

| Dependency | Version | Used by | Purpose |
|------------|---------|---------|---------|
| `bcrypt` | 5.1.1 (direct), `@types/bcrypt` 5.0.2 (dev) | `src/modules/auth/auth.service.ts:1,36`; `src/modules/users/user.service.ts:1,16,26` | Password comparison and hashing |
| `jsonwebtoken` | 9.0.2 (direct), `@types/jsonwebtoken` 9.0.7 (dev) | `src/modules/auth/auth.service.ts:2-3,49`; `src/middlewares/auth.middleware.ts:2,41` | JWT signing and verification |
| `zod` | 3.23.8 | `src/modules/auth/auth.schemas.ts:1`; `src/middlewares/validate.middleware.ts:2` | Request schema validation |
| `express` | 4.21.1 | `src/modules/auth/auth.routes.ts:1`, `src/modules/auth/auth.controller.ts:1` (types) | Router and request handlers |
| `@prisma/client` | 5.22.0 | `src/modules/users/user.repository.ts:1`, `src/modules/users/user.service.ts:2`, `prisma/schema.prisma` | User persistence and enum types |
| MySQL | 8.0 (Compose image) | Indirect through Prisma | Durable identity store, unique email index, enum role |

### Dependency risk notes

- `jsonwebtoken` 9.0.2 and `bcrypt` 5.1.1 are pinned exact versions in `package.json`, which makes builds reproducible but leaves upgrade behavior to explicit changes.
- `bcrypt` is a native module; it depends on a prebuilt binary or a local toolchain. No engine constraint beyond Node >= 20 (`package.json:7-9`).
- `@prisma/client` version must match the `prisma` CLI version in devDependencies; both are 5.22.0.
- No email provider, identity provider, cache, message broker, or external HTTP client is present. Authentication is fully local.
- No secret manager is integrated; secrets arrive through environment variables.

---

## 6. Afferent and Efferent Coupling

Coupling is measured statically from `import`/`import type`/`export ... from` statements. External npm packages are excluded from internal counts and reported separately. For the module boundary, only distinct external files are counted.

| Component | Type | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Criticality |
|-----------|------|------------------------|------------------------|-------------|
| Auth Module (directory boundary) | Feature module | 2 (`src/app.ts`, `src/routes/index.ts`) | 6 internal files across 4 directories (`config/env.ts`, `middlewares/auth.middleware.ts`, `middlewares/validate.middleware.ts`, `shared/errors/index.ts`, `users/user.repository.ts`, `users/user.service.ts`) | High |
| `AuthService` (class) | Service | 2 (`auth.controller.ts`, `app.ts`) | 5 internal files (`config/env.ts`, `shared/errors/index.ts`, `users/user.repository.ts`, `users/user.service.ts`, `auth.schemas.ts`) + 2 external packages (`bcrypt`, `jsonwebtoken`) | High |
| `AuthController` (class) | Controller | 3 (`auth.routes.ts`, `routes/index.ts` type-only, `app.ts`) | 3 internal files (`auth.service.ts`, `users/user.service.ts`, `shared/errors/index.ts`) | Medium |
| `auth.schemas.ts` (`registerSchema`, `loginSchema`, `RegisterInput`, `LoginInput`) | Contract | 2 (`auth.routes.ts`, `auth.service.ts`) | 0 internal + `zod` | High |
| `buildAuthRouter` (function) | Router factory | 1 (`routes/index.ts`) | 4 internal files (`validate.middleware.ts`, `auth.middleware.ts`, `auth.schemas.ts`, `auth.controller.ts`) + `express` | Medium |
| `AuthTokens` / `LoginResult` (types) | Type contract | 0 outside `auth.service.ts` | 0 | Low |
| `authenticate` middleware (dependency) | Security middleware | 5 (`auth.routes.ts`, `customers/customer.routes.ts`, `orders/order.routes.ts`, `products/product.routes.ts`, `users/user.routes.ts`) | 2 internal files (`config/env.ts`, `shared/errors/index.ts`) + `jsonwebtoken` | High |
| `requireRole` middleware (dependency) | Authorization middleware | 1 (`users/user.routes.ts`) | 1 internal file (`shared/errors/index.ts`) | High |

### Interpretation

- The Auth Module has low afferent coupling (Ca=2) because only the composition root and router registry import it. This is healthy isolation but also means changes are not widely detected by compilation outside those two files.
- It has moderate efferent coupling for its size (Ce=6 files): it leans on the Users Module for persistence and on shared middleware for validation and identity. The dependency on `UserRepository` at the service layer and on `UserService` at both service and controller layers is the module's main structural coupling concern.
- `auth.schemas.ts` has Ca=2 and no internal dependencies, making it the most stable file in the module. Its contract is consumed by both the router (for validation) and the service (for input types).
- `AuthTokens` and `LoginResult` are not imported outside the service; they are local documentation of the response shape rather than a shared contract. Consumers rely on structural typing.
- The `authenticate` middleware is the highest-fan-in security component (Ca=5) and therefore the highest-leverage point for breaking changes. It lives outside the Auth Module boundary, so Auth Module analysis alone would miss its blast radius.

---

## 7. Endpoints

The Auth Module exposes three HTTP endpoints under the `/api/v1` prefix applied by `src/app.ts:67` and the `/auth` mount in `src/routes/index.ts:24`.

| Endpoint | Method | Auth required | Request body | Success | Error responses | Implementation |
|----------|--------|---------------|--------------|---------|-----------------|----------------|
| `/api/v1/auth/register` | POST | No | `{ email: string, password: string, name: string, role?: "ADMIN" \| "OPERATOR" }` | `201` `PublicUser` | `400 VALIDATION_ERROR`; `409 EMAIL_ALREADY_USED` (sequential duplicate) or `409 CONFLICT` (race); `500 INTERNAL_SERVER_ERROR` | `src/modules/auth/auth.routes.ts:10` |
| `/api/v1/auth/login` | POST | No | `{ email: string, password: string }` | `200` `{ user: PublicUser, tokens: { accessToken, expiresIn, tokenType: "Bearer" } }` | `400 VALIDATION_ERROR`; `401 UNAUTHORIZED`; `500 INTERNAL_SERVER_ERROR` | `src/modules/auth/auth.routes.ts:11` |
| `/api/v1/auth/me` | GET | Yes, `Authorization: Bearer <JWT>` | None | `200` `PublicUser` | `401 UNAUTHORIZED` (missing/malformed/invalid/expired token); `404 NOT_FOUND` (token user deleted); `500 INTERNAL_SERVER_ERROR` | `src/modules/auth/auth.routes.ts:12` |

### Request examples

Registration:

```json
{
  "email": "user@example.com",
  "password": "strongPass123",
  "name": "New User",
  "role": "OPERATOR"
}
```

Login:

```json
{
  "email": "user@example.com",
  "password": "strongPass123"
}
```

### Response examples

Registration success (`201`):

```json
{
  "id": "3f1c2a9e-...-...",
  "email": "user@example.com",
  "name": "New User",
  "role": "OPERATOR",
  "createdAt": "2026-11-08T23:00:00.000Z",
  "updatedAt": "2026-11-08T23:00:00.000Z"
}
```

Login success (`200`):

```json
{
  "user": {
    "id": "3f1c2a9e-...-...",
    "email": "user@example.com",
    "name": "New User",
    "role": "OPERATOR",
    "createdAt": "2026-11-08T23:00:00.000Z",
    "updatedAt": "2026-11-08T23:00:00.000Z"
  },
  "tokens": {
    "accessToken": "<jwt>",
    "expiresIn": "8h",
    "tokenType": "Bearer"
  }
}
```

Error envelope:

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid credentials"
  }
}
```

Validation error envelope:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      { "path": "email", "message": "Invalid email" },
      { "path": "password", "message": "String must contain at least 8 character(s)" }
    ]
  }
}
```

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol / Mechanism | Data Format | Error Handling |
|-------------|------|---------|----------------------|-------------|----------------|
| MySQL `users` table | Internal database | Persist users, unique email lookup, password hash storage | Prisma Client 5.22 over MySQL wire protocol | Relational rows; `User` model with `UserRole` enum | Prisma errors bubble to `next`; `P2002` -> 409, `P2025` -> 404, others -> 500 with Pino log |
| Users Module (`UserService`, `UserRepository`) | Internal module | Registration, duplicate check, password hashing, public projection, identity lookup | Direct in-process method calls | TypeScript objects (`CreateUserInput`, `PublicUser`, Prisma `User`) | `ConflictError`, `NotFoundError`; no retries or fallbacks |
| `bcrypt` library | In-process library | Hash passwords on registration, compare on login | Native async API (`hash`, `compare`) | String password / bcrypt digest | Throws on internal failure; propagates to 500; no explicit handling |
| `jsonwebtoken` library | In-process library | Sign access tokens, verify bearer tokens | HMAC-SHA256 via string secret | JWT compact serialization; claims `sub`, `email`, `role`, `iat`, `exp` | Signing errors -> 500; verification errors collapsed to 401 |
| `zod` validation | In-process library | Validate register/login bodies and query/params | Schema parsing | Parsed TypeScript objects | `ZodError` converted to `ValidationError` with field paths |
| Environment configuration | Process configuration | Provide `JWT_SECRET`, `JWT_EXPIRES_IN`, `DATABASE_URL` | `process.env` at module import | Strings validated by Zod | Invalid config prints issues and exits process with code 1 |
| Express HTTP server | Inbound HTTP | Expose `/api/v1/auth/*` | HTTP/1.1 JSON | Request/response JSON | Centralized `errorMiddleware`; 404 fallback in `src/app.ts:69-71` |
| Pino request logger | Cross-cutting logging | Correlate requests, log `userId` and metadata | In-process logger | Structured JSON logs | Logger failure not handled explicitly; no auth payload logging |
| Downstream business routers | Internal consumers | Reuse tokens issued by login and middleware for auth | `Authorization` header + in-process middleware | Bearer JWT | 401/403 via shared error types |
| Seed script | Operational bootstrap | Create `admin@oms.local` and `operador@oms.local` with hashed passwords | Prisma + bcrypt | User rows | Script exits non-zero on failure (`prisma/seed.ts` tail) |

### Integration notes

- There is no outbound HTTP integration, no external identity provider (OAuth/OIDC/SAML), no email delivery, no SMS, no Redis, no message broker, and no audit sink.
- The Auth Module does not call the Users Module over HTTP; all integration is in-process, so there is no network failure mode between them, but there is also no transaction boundary beyond individual Prisma calls.
- The database is the only stateful integration. If it is unavailable, both registration and login fail; unhandled Prisma connection errors become 500.
- The token contract is consumed by middleware, not by the module, so integration testing of the module requires the middleware and the Users Module to be present. The integration tests build the full app (`tests/helpers/factories.ts:10-15`).

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered architecture | routes -> controller -> service -> repository | `src/modules/auth/*`, `src/modules/users/*` | Separation of HTTP, orchestration, domain logic, persistence |
| Repository pattern | `UserRepository` wraps Prisma `User` operations | `src/modules/users/user.repository.ts:3-16` | Abstracts data access and keeps Prisma out of services except types |
| Dependency injection (constructor) | `AuthService(users, userService)`, `AuthController(authService, userService)` | `src/modules/auth/auth.service.ts:22-25`, `src/modules/auth/auth.controller.ts:7-10` | Testability and explicit dependencies |
| Composition root | `buildControllers(prisma)` wires all repositories, services, controllers | `src/app.ts:26-53` | Centralizes object graph construction |
| Factory function for routers | `buildAuthRouter(controller)` returns a configured `Router` | `src/modules/auth/auth.routes.ts:7-15` | Avoids module-level singletons and enables injected controllers |
| DTO / whitelist projection | `PublicUser` + `UserService.toPublic` | `src/modules/users/user.service.ts:7-14,42-51` | Prevents sensitive field leakage |
| Schema validation middleware | `validate({ body })` parses and replaces request data | `src/middlewares/validate.middleware.ts:11-37` | Declarative input contracts at the boundary |
| Middleware chain | `authenticate` -> `validate` -> controller | `src/modules/auth/auth.routes.ts:12`, `src/modules/users/user.routes.ts:12-18` | Composable cross-cutting concerns |
| Centralized error translation | `AppError` hierarchy + `errorMiddleware` | `src/shared/errors/*`, `src/middlewares/error.middleware.ts` | Uniform API error contract and safe 500s |
| Stateless authentication | JWT bearer tokens, no server session | `src/modules/auth/auth.service.ts:47-50`, `src/middlewares/auth.middleware.ts:40-46` | Horizontal scalability, no session store |
| Fail-fast configuration | Zod `safeParse` + `process.exit(1)` | `src/config/env.ts:14-24` | Prevent startup with missing or weak secrets |
| Functional handler fields | Arrow-function `RequestHandler` class properties | `src/modules/auth/auth.controller.ts:12-38` | Preserves `this` when passed as route callbacks |
| Static mapper | `UserService.toPublic` as a static method | `src/modules/users/user.service.ts:42-51` | Reusable projection without an instance |

### Architectural decisions observed

1. **Auth is a thin feature module over Users.** It does not own the user model or hashing; it composes authentication-specific behavior (credential check, token issuance) over the Users Module's persistence.
2. **Verification is centralized in middleware, not in the module.** Any router can opt into authentication by importing `authenticate`, which keeps the Auth Module from becoming a mandatory gateway in code.
3. **Authorization is coarse and sparse.** Only one route uses `requireRole`; the system primarily relies on "authenticated" as the access decision.
4. **Tokens are stateless and self-contained.** No session store means no per-token revocation, a trade-off that is not documented in ADRs (the `docs/adrs/` directory contains only a README).
5. **Validation contracts are duplicated.** `registerSchema` and `createUserSchema` restate the same field rules instead of sharing a base schema.
6. **The module is synchronous request/response.** There are no events, queues, or side effects on registration or login beyond database writes.

### Quality attributes

| Attribute | Assessment | Evidence |
|-----------|------------|----------|
| Security | Weak overall despite correct password hashing and generic login errors | Role escalation, no rate limiting, no token revocation, no algorithm pinning, weak secret floor |
| Reliability | Simple and predictable; no retries needed for in-process calls | Fail-fast env, centralized errors, Prisma transactions not used (single-row operations only) |
| Maintainability | Good separation and small files; duplicated contracts and cross-module coupling | 4 small files, clear layering; `registerSchema` duplication; `AuthController` depends on `UserService` |
| Testability | Integration-testable via HTTP; no unit seams exercised | Constructor injection present; tests use a real database and full app |
| Performance | bcrypt cost 10 dominates login latency; JWT operations cheap; indexed email lookup | `src/modules/users/user.service.ts:16,26`; unique email index; no caching |
| Scalability | Stateless tokens scale horizontally; bcrypt CPU and database are shared bottlenecks | No session store; no rate limiting |
| Observability | Request-level logging with request id and user id; no auth-specific metrics or audit events | `src/middlewares/request-logger.middleware.ts`, `src/shared/logger/index.ts` |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact | Evidence |
|------------|----------------|-------|--------|----------|
| Critical | Registration / authorization | Public registration accepts `role: "ADMIN"` from the client | Anonymous privilege escalation to administrator; undermines all role checks | `src/modules/auth/auth.schemas.ts:7`, `src/modules/auth/auth.service.ts:27-29`, `src/modules/users/user.service.ts:31`, `src/modules/auth/auth.routes.ts:10` |
| High | Login abuse prevention | No rate limiting, lockout, CAPTCHA, or backoff on login or registration | Unlimited password guessing and account creation; CPU exhaustion via bcrypt | `src/modules/auth/auth.routes.ts:10-12`, `package.json:25-33` |
| High | Token lifecycle | No refresh, logout, revocation, denylist, or `jti`; 8-hour default lifetime | Stolen tokens remain valid; role changes and deletions are not enforced until expiry | `src/modules/auth/auth.service.ts:47-50`, `src/config/env.ts:9` |
| High | Token verification | No `algorithms` allowlist, issuer, audience, or payload schema validation | Algorithm confusion hardening absent; malformed but correctly signed claims can hydrate invalid identity | `src/middlewares/auth.middleware.ts:19-25,40-46` |
| High | Authorization coverage | Customers, products and orders require only authentication; no tenant/owner scoping | Any authenticated user can read or mutate all business data | `src/modules/customers/customer.routes.ts:14`, `src/modules/products/product.routes.ts:14`, `src/modules/orders/order.routes.ts:14` |
| Medium | Uniqueness race | Check-then-create is not atomic; concurrent duplicate registration returns generic `CONFLICT` instead of `EMAIL_ALREADY_USED` | Client error handling inconsistent; no transaction or upsert | `src/modules/users/user.service.ts:22-26`, `src/middlewares/error.middleware.ts:37-47` |
| Medium | Identity normalization | Emails are not trimmed, lowercased, or Unicode-normalized; uniqueness depends on collation | Environment-dependent duplicate accounts and login behavior | `src/modules/auth/auth.schemas.ts:4`, `src/modules/users/user.repository.ts:10-12`, `docker-compose.yml:23` |
| Medium | Contract duplication | `registerSchema` duplicates `createUserSchema`; `user.routes.ts` redeclares a UUID schema | Rules can drift; inconsistent validation across entry points | `src/modules/auth/auth.schemas.ts:3-8`, `src/modules/users/user.schemas.ts:7-12`, `src/modules/users/user.routes.ts:7` |
| Medium | Password input semantics | `max(72)` characters vs bcrypt's 72-byte limit | Multi-byte passwords silently truncated; users may believe characters beyond the byte limit matter | `src/modules/auth/auth.schemas.ts:5,12`, `src/modules/users/user.service.ts:26` |
| Medium | Deleted-user semantics | `/auth/me` returns 404 for a valid token whose user was deleted | Clients cannot distinguish invalid credential from missing profile; token remains valid elsewhere for unauthenticated-by-identity routes | `src/modules/auth/auth.controller.ts:32-34`, `src/modules/users/user.service.ts:36-40` |
| Medium | Account lifecycle | No email verification, activation, suspension, password reset, or password change | Registration abuse is immediately effective; no recovery path; no deactivation | `prisma/schema.prisma:25-38`, `src/modules/auth/auth.routes.ts:10-12` |
| Medium | Login timing | Unknown email returns before bcrypt; known email performs cost-10 compare | Account existence may be inferred from timing despite identical messages | `src/modules/auth/auth.service.ts:32-39` |
| Low | Configuration robustness | `JWT_EXPIRES_IN` is an unvalidated string cast; invalid syntax fails at login with 500 | Misconfiguration detected late and only on the login path | `src/config/env.ts:9`, `src/modules/auth/auth.service.ts:48` |
| Low | Secret strength | `JWT_SECRET` minimum is only 16 characters; no entropy check or rotation | Weak secrets pass startup validation; global rotation invalidates all sessions | `src/config/env.ts:8`, `.env.example:8` |
| Low | Boundary coupling | `AuthController` calls `UserService` directly rather than going through `AuthService` | Two service dependencies in one controller; Auth Module has duplicate paths to Users | `src/modules/auth/auth.controller.ts:7-10,33` |
| Low | Response metadata | `expiresIn` is a raw duration string, not seconds or an absolute timestamp | Client implementations must parse provider-specific duration syntax | `src/modules/auth/auth.service.ts:43` |
| Low | Test coverage | No unit tests and no coverage provider; several branches untested | Regressions in token verification, role escalation, default role, and race path go undetected | `tests/auth.test.ts` only; no coverage config in `vitest.config.ts`/`package.json` |
| Low | Security headers / transport | No helmet, CORS, HSTS, or HTTPS enforcement in the app | Bearer tokens could be exposed by browser or network misconfiguration | `src/app.ts:55-75`, `package.json:25-33` |
| Low | Auditability | No login success/failure events, no account-creation audit fields beyond timestamps | Security incidents cannot be reconstructed from application logs alone | `prisma/schema.prisma:25-38`, `src/middlewares/request-logger.middleware.ts` |

### Code smells observed

- **Duplicated validation knowledge**: registration rules exist twice (`auth.schemas.ts` and `user.schemas.ts`).
- **Inappropriate intimacy / cross-module reach**: Auth controller reaches into Users' service; Auth service reaches into Users' repository and service.
- **Magic constants**: bcrypt cost 10 is a module constant with no configuration path; default expiry `8h` is in env schema.
- **Primitive obsession**: roles are string unions repeated in multiple files instead of importing the Prisma enum or a shared domain type.
- **Dead flexibility**: `AuthTokens` and `LoginResult` are exported but never imported elsewhere, providing no shared contract.
- **Anemic delegation**: `AuthService.register` is a one-line pass-through, adding a layer without adding behavior.
- **Defensive redundancy**: `AuthController.me` checks `req.user` although `authenticate` already guarantees it on that route.
- **Implicit environment coupling**: module import executes `process.exit(1)` via `env`, making isolated unit testing harder.

---

## 11. Test Coverage Analysis

### Test inventory

| Test file / helper | Location | Scope | Count |
|--------------------|----------|-------|-------|
| Auth integration suite | `tests/auth.test.ts` | Full HTTP flows for register, login, me | 7 `it` blocks |
| Test factories | `tests/helpers/factories.ts` | `getTestApp`, `createTestUser`, `loginAndGetToken`, `bootstrapAuthenticatedUser` | Shared helper |
| Global test setup | `tests/setup.ts` | Prisma connect/disconnect, truncation between tests | Hooks |
| Orders integration suite | `tests/orders.test.ts` | Uses `bootstrapAuthenticatedUser` to obtain tokens for all 8 tests | 8 `it` blocks, indirect auth coverage |
| Vitest configuration | `vitest.config.ts` | Node environment, `tests/**/*.test.ts`, serial single fork, 30s timeouts | N/A |

There are no test files for `users`, `customers`, `products`, or the health endpoint. No unit tests exist for `AuthService`, `AuthController`, `auth.schemas.ts`, or the authentication middleware in isolation.

### Coverage table

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| `auth.routes.ts` (route wiring) | 0 | 7 (all auth tests traverse the router) | Not instrumented; route paths and middleware order exercised for register/login/me | Good happy-path and two negative paths; no unmounted-route or method-mismatch tests |
| `AuthController.register` | 0 | 3 (success, duplicate, invalid payload) | Not instrumented; 201 and 409/400 paths exercised | Strong assertion that `passwordHash`/`password` are absent |
| `AuthController.login` | 0 | 2 (success, wrong password) | Not instrumented; 200 and 401 paths exercised | Asserts token type and user email; does not assert `expiresIn` or token claims |
| `AuthController.me` | 0 | 2 (valid token, missing token) | Not instrumented; 200 and 401 paths exercised | Asserts id and email; no deleted-user 404 or malformed-token case |
| `AuthService.register` | 0 | 3 (via controller) | Not instrumented; delegation path exercised | No direct test that role is forwarded or defaulted |
| `AuthService.login` | 0 | 2 (via controller) | Not instrumented; success and wrong-password branches exercised | Unknown-email branch untested; no timing or expired-token tests |
| `AuthService.signToken` | 0 | 2 (via login and `/me`) | Not instrumented; token is proven usable | Claims, expiry, algorithm, and `JWT_EXPIRES_IN` values not asserted |
| `auth.schemas.ts` | 0 | 1 invalid payload + 5 valid payloads | Not instrumented; email/password/name failures in one request | No boundary tests (7/8/72/73/255/256 chars), no invalid role, no default-role test, no unknown-field behavior test |
| `authenticate` middleware (dependency) | 0 | 2 (missing header, valid token) | Not instrumented; one positive and one negative | No expired token, wrong signature, malformed token, wrong scheme, empty bearer, or lowercase `bearer` tests |
| `requireRole` middleware (dependency) | 0 | 0 | Not instrumented | Completely untested; `GET /users/:id` has no tests at all |
| Users Module registration path | 0 | 3 (via auth register) | Not instrumented | Duplicate check tested; bcrypt hash not asserted directly; race path untested |

### Qualitative coverage assessment

Because `vitest.config.ts` has no `coverage` block and `package.json` has no `@vitest/coverage-*` dependency or coverage script, no line, branch, or function coverage number can be reported. The table above records which branches are demonstrably exercised, not measured percentages. Any percentage claim would be fabrication.

### Tested behaviors (with evidence)

| Behavior | Test | Location |
|----------|------|----------|
| Register succeeds with 201 and public fields | `registers a new user and returns 201 with public data only` | `tests/auth.test.ts:6-17` |
| Register response omits `passwordHash` and `password` | Same test, assertions | `tests/auth.test.ts:14-16` |
| Duplicate email returns 409 `EMAIL_ALREADY_USED` | `rejects register with duplicated email` | `tests/auth.test.ts:19-28` |
| Invalid register payload returns 400 `VALIDATION_ERROR` with details array | `rejects register with invalid payload` | `tests/auth.test.ts:30-39` |
| Login returns a string token, `Bearer` type, and user email | `logs in successfully and returns an access token` | `tests/auth.test.ts:41-52` |
| Wrong password returns 401 `UNAUTHORIZED` | `rejects login with wrong password` | `tests/auth.test.ts:54-63` |
| Valid token returns current user id and email | `returns the authenticated user on GET /auth/me` | `tests/auth.test.ts:65-77` |
| Missing token returns 401 | `rejects /auth/me without token` | `tests/auth.test.ts:79-83` |
| Login issues usable tokens for other modules | All 8 order tests use `bootstrapAuthenticatedUser` | `tests/orders.test.ts:14,44,61,91,111,136,165,199`; helper at `tests/helpers/factories.ts:76-83` |

### Uncovered behaviors and branches

| Gap | Risk | Relevant code |
|-----|------|---------------|
| Unknown email login (401 same as wrong password) | Account enumeration behavior unverified | `src/modules/auth/auth.service.ts:33-35` |
| Expired token | Token lifetime enforcement unverified | `src/middlewares/auth.middleware.ts:41-45` |
| Token signed with wrong secret / malformed token | Verification failure branch only partially covered | `src/middlewares/auth.middleware.ts:44-46` |
| Malformed `Authorization` schemes (`Basic`, empty `Bearer`) | Header parsing branches untested | `src/middlewares/auth.middleware.ts:28-38` |
| Role escalation via `role: "ADMIN"` on register | The most severe security behavior has no regression test | `src/modules/auth/auth.schemas.ts:7`, `src/modules/auth/auth.service.ts:27-29` |
| Default role when omitted | Default behavior untested | `src/modules/auth/auth.schemas.ts:7` |
| Invalid role value | Enum rejection untested | `src/modules/auth/auth.schemas.ts:7` |
| Password/name boundary values | Off-by-one behavior untested | `src/modules/auth/auth.schemas.ts:4-6,12` |
| `/auth/me` with deleted user | 404 path untested | `src/modules/auth/auth.controller.ts:33`, `src/modules/users/user.service.ts:36-40` |
| `requireRole('ADMIN')` allow/deny | Entire authorization branch untested | `src/middlewares/auth.middleware.ts:49-61`, `src/modules/users/user.routes.ts:12-18` |
| Concurrent duplicate registration | Race and `P2002` mapping untested | `src/modules/users/user.service.ts:22-26`, `src/middlewares/error.middleware.ts:37-47` |
| Token claims (`sub`, `role`, `exp`, algorithm) | Token content unverified | `src/modules/auth/auth.service.ts:47-50` |
| `expiresIn` response value | Contract field unverified | `src/modules/auth/auth.service.ts:43` |
| Case-insensitive email behavior | Collation-dependent identity untested | `src/modules/users/user.repository.ts:10-12` |
| Rate limiting / lockout | No control exists to test | `src/modules/auth/auth.routes.ts:10-12` |

### Test infrastructure observations

- Tests are integration-only and require a running MySQL instance plus environment variables. `tests/setup.ts:4-6` connects Prisma before all tests and `tests/setup.ts:8-16` deletes all domain rows before each test. There are no mocks or in-memory repositories.
- `tests/setup.ts:12` references `prisma.orderNumberSequence`, which matches `OrderNumberSequence` in `prisma/schema.prisma:133-138`.
- Execution is forced serial: `fileParallelism: false`, `pool: 'forks'`, `singleFork: true` (`vitest.config.ts:11-17`). This avoids cross-test data interference but increases runtime.
- `tests/helpers/factories.ts:22-29` creates users directly through Prisma with bcrypt cost 10, mirroring production hashing. This bypasses `UserService.createUser`, so duplicate-check logic is not exercised by helper-created users.
- `tests/helpers/factories.ts:8-15` caches a single Express app instance across tests; environment and dependency state are shared.
- No test asserts that passwords are hashed at rest (for example, by querying the row and checking the digest prefix), and no test asserts that tokens expire.

---

## 12. Assumptions, Ambiguities and Evidence Quality

| Topic | Status | Basis |
|-------|--------|-------|
| Intended registration authorization model | Unknown | `docs/PRD.md`, `docs/FDD.md`, `docs/RFC.md` are empty templates; the implemented behavior is public registration with caller-selected role |
| Intended token lifetime and refresh policy | Unknown | Only `JWT_EXPIRES_IN` default `8h` is encoded; no requirements document |
| Intended email case sensitivity | Ambiguous | Application performs exact lookup; Compose MySQL collation is case-insensitive; production collation unspecified |
| Intended account lifecycle | Unknown | No status/verification fields or endpoints; likely out of scope, but unconfirmed |
| Intended authorization granularity | Partially known | Only one `requireRole` call; architectural report flags coarse authorization as a risk |
| Coverage percentages | Unavailable | No coverage provider configured; no `test:coverage` script |
| Test environment | Known constraint | Tests require external `DATABASE_URL`/`JWT_SECRET` and a live MySQL; `npm test` does not load `.env` |
| Business rules with implicit confidence | Medium | Rules inferred from Zod schemas, service code, and Prisma constraints; no written specification to confirm intent |
| Security severity ratings | Analyst judgment | Based on exploitability and blast radius, consistent with the architectural risk register |

---

## 13. Report Metadata

- **Component analyzed:** Auth Module (`src/modules/auth/`)
- **Files read for this report:** `src/modules/auth/auth.routes.ts`, `src/modules/auth/auth.controller.ts`, `src/modules/auth/auth.service.ts`, `src/modules/auth/auth.schemas.ts`, `src/modules/users/user.service.ts`, `src/modules/users/user.repository.ts`, `src/modules/users/user.schemas.ts`, `src/modules/users/user.routes.ts`, `src/modules/users/user.controller.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/config/env.ts`, `src/config/database.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts`, `src/routes/index.ts`, `src/app.ts`, `prisma/schema.prisma`, `prisma/seed.ts`, `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `tests/setup.ts`, `vitest.config.ts`, `package.json`, `tsconfig.json`, `.env.example`, `docker-compose.yml`, `.eslintrc.json`, `README.md`, and the existing architectural report under `docs/agents/architectural-analyzer/`.
- **No files were modified.** This report is analysis-only.
