# Component Deep Analysis Report — Authentication Middleware

- **Component name:** Authentication Middleware
- **Primary file:** `src/middlewares/auth.middleware.ts` (61 lines)
- **Component type:** Cross-cutting security middleware (Express request pipeline)
- **Analysis date:** 2026-11-08 23:01:16
- **Project scope:** entire project root
- **Excluded folders:** `node_modules`, `.git`, `dist`, `coverage`
- **Analysis method:** static source reading and dependency mapping. No code was executed, no tests were run, and no project files were modified.

---

## 1. Executive Summary

The Authentication Middleware is a single-file, stateless security component that performs two distinct responsibilities for the Order Management System API:

1. **Authentication (`authenticate`)** — extracts a bearer token from the `Authorization` header, verifies it as a JSON Web Token signed with the shared symmetric secret `env.JWT_SECRET`, and hydrates `req.user` with `{ id, email, role }` derived from the token claims (`sub`, `email`, `role`).
2. **Authorization (`requireRole`)** — a higher-order middleware factory that rejects requests whose hydrated `req.user.role` is not in a caller-supplied allowlist, returning `403 FORBIDDEN`, and rejects requests with no hydrated user with `401 UNAUTHORIZED`.

The component is deliberately thin: 61 lines, zero persistence, zero outbound network calls, and no session state. Its only runtime dependency is `jsonwebtoken@9.0.2`; it depends on the environment configuration module for the signing secret and on the shared `AppError` taxonomy (`UnauthorizedError`, `ForbiddenError`) for error propagation to the centralized error middleware.

**Key findings:**

- The middleware is a **choke point for the entire protected API surface**: it is applied to the `/auth/me` route and to all of `customers`, `products`, `orders`, and `users` (with an additional `ADMIN` guard on the latter). Only `/health`, `/auth/register`, and `/auth/login` are unauthenticated.
- **Verification is cryptographic but not structurally validated.** `jwt.verify` is called without an `algorithms` allowlist and the decoded payload is type-cast to `JwtPayload` rather than parsed/validated (`src/middlewares/auth.middleware.ts:41-42`). A correctly signed token with missing or wrongly typed claims would hydrate an invalid `req.user` without an error.
- **Authorization is entirely token-derived and frozen for the token lifetime.** No database lookup occurs per request, so user deletion, role demotion, or revocation does not take effect until the token expires (default `8h`, `src/config/env.ts:9`). There is no logout, denylist, refresh-token, issuer, or audience check.
- **The role guard has no automated test.** `requireRole` is wired to exactly one endpoint (`GET /api/v1/users/:id`, `src/modules/users/user.routes.ts:12-18`) and neither the allow path nor the `403` deny path is exercised by the test suite. Only the unauthenticated happy paths plus one "no token" case for `authenticate` are covered.
- **Error handling is clean and consistent** with the rest of the codebase: every failure is forwarded via `next(new AppError(...))` and rendered by `errorMiddleware` as `{ error: { code, message } }` with HTTP `401` or `403`.
- **The component is small, highly cohesive, and low-risk to change** but carries meaningful security surface because it is the only enforcement point for authentication and the only implemented role guard in the system.

---

## 2. Data Flow Analysis

### 2.1 Primary flow — successful authenticated request

```
1. HTTP request arrives at the Express app (src/app.ts:55-76); requestLogger assigns req.id (src/middlewares/request-logger.middleware.ts:5-27)
2. Router chain invokes authenticate, either via router.use(authenticate) (customer.routes.ts:14, product.routes.ts:14, order.routes.ts:14)
   or as a per-route guard (auth.routes.ts:12, user.routes.ts:14)
3. authenticate reads req.headers.authorization (auth.middleware.ts:28)
4. Header presence + "bearer " prefix check, case-insensitive (auth.middleware.ts:29)
5. Token extraction: header.slice(7).trim() (auth.middleware.ts:34)
6. Non-empty token check (auth.middleware.ts:35)
7. jwt.verify(token, env.JWT_SECRET) -> signature, exp and nbf validation (auth.middleware.ts:41)
8. Payload cast to JwtPayload and projected into req.user = { id: sub, email, role } (auth.middleware.ts:42)
9. next() -> optional requireRole(...) guard executes (auth.middleware.ts:49-61)
10. Controller executes; reads req.user.id / req.user.role
    (auth.controller.ts:32-33, order.controller.ts:30-31 and 40-41)
11. Response completes; requestLogger emits access log including userId (request-logger.middleware.ts:21)
```

### 2.2 Failure flow — missing, malformed, invalid, or expired credentials

```
1. authenticate detects a failure condition at one of three points:
   a. header absent or not "bearer ..." -> UnauthorizedError('Missing or invalid Authorization header') (auth.middleware.ts:29-32)
   b. token empty after trim          -> UnauthorizedError('Missing bearer token')                     (auth.middleware.ts:35-38)
   c. jwt.verify throws               -> UnauthorizedError('Invalid or expired token')                 (auth.middleware.ts:44-46)
2. Error is passed to next(err); no response is written by the middleware itself
3. errorMiddleware matches `err instanceof AppError` and serializes
   { error: { code: 'UNAUTHORIZED', message: <message> } } with HTTP 401 (error.middleware.ts:15-24)
4. Controller/handler never executes; no side effects occur
```

### 2.3 Failure flow — insufficient role

```
1. authenticate has already populated req.user
2. requireRole(...roles) compares req.user.role against the allowlist (auth.middleware.ts:55)
3. If absent from allowlist -> ForbiddenError('Insufficient permissions') (auth.middleware.ts:56)
4. errorMiddleware serializes { error: { code: 'FORBIDDEN', message: 'Insufficient permissions' } } with HTTP 403
```

### 2.4 Failure flow — requireRole used without authenticate

```
1. requireRole observes req.user === undefined (auth.middleware.ts:51)
2. UnauthorizedError() is raised with the default message 'Unauthorized' (http-errors.ts:15-19)
3. errorMiddleware returns HTTP 401 with code UNAUTHORIZED
```

This is a fail-closed defense: a route that mounts `requireRole` without `authenticate` still cannot be accessed anonymously.

### 2.5 Data transformation summary

| Step | Input shape | Output shape | Location |
|------|-------------|--------------|----------|
| Header parse | `authorization?: string` | raw header string | `auth.middleware.ts:28` |
| Scheme check | raw header | pass / `UnauthorizedError` | `auth.middleware.ts:29-32` |
| Token extraction | `"Bearer <jwt>"` | `<jwt>` (trimmed) | `auth.middleware.ts:34` |
| Empty check | `<jwt>` | pass / `UnauthorizedError` | `auth.middleware.ts:35-38` |
| Verification | `<jwt>` + `JWT_SECRET` | `JwtPayload` object (cast) | `auth.middleware.ts:41` |
| Projection | `JwtPayload` | `req.user: AuthUser` | `auth.middleware.ts:42` |
| Authorization | `req.user.role` + allowlist | pass / `ForbiddenError` | `auth.middleware.ts:55-58` |

---

## 3. Business Rules & Logic

### 3.1 Overview of the business rules

| ID | Rule Type | Rule Description | Location |
|----|-----------|------------------|----------|
| BR-01 | Validation / Authentication | A protected request MUST carry an `Authorization` header whose value starts with `bearer ` (case-insensitive); otherwise `401 UNAUTHORIZED` | `src/middlewares/auth.middleware.ts:28-32` |
| BR-02 | Validation / Authentication | The extracted token MUST be non-empty after removing the 7-character scheme prefix and trimming; otherwise `401 UNAUTHORIZED` | `src/middlewares/auth.middleware.ts:34-38` |
| BR-03 | Security / Authentication | The token MUST pass `jwt.verify` against `env.JWT_SECRET`, which enforces signature integrity, `exp`, and `nbf`; any thrown error becomes `401 UNAUTHORIZED` | `src/middlewares/auth.middleware.ts:40-46` |
| BR-04 | Identity projection | A verified token MUST hydrate the request principal as `req.user = { id: payload.sub, email: payload.email, role: payload.role }` | `src/middlewares/auth.middleware.ts:42` |
| BR-05 | Identity contract | A principal role MUST be one of the vocabulary `ADMIN` or `OPERATOR`; enforced at compile time only, not at runtime | `src/middlewares/auth.middleware.ts:6-10`, `19-25` |
| BR-06 | Authorization (defense in depth) | A role-guarded route MUST have an authenticated principal; if `req.user` is absent the request is rejected with `401 UNAUTHORIZED` | `src/middlewares/auth.middleware.ts:51-54` |
| BR-07 | Authorization | A principal MUST hold at least one role from the allowlist passed to `requireRole`; otherwise `403 FORBIDDEN` | `src/middlewares/auth.middleware.ts:55-58` |
| BR-08 | Authorization semantics | `requireRole` implements an exact-match, case-sensitive, variadic allowlist; an empty allowlist denies every principal (fail-closed) | `src/middlewares/auth.middleware.ts:49-60` |
| BR-09 | Error contract | All middleware failures MUST be propagated as `AppError` subtypes via `next(err)` and MUST NOT be written directly to the response | `src/middlewares/auth.middleware.ts:30, 36, 45, 52, 56`; `src/middlewares/error.middleware.ts:15-24` |
| BR-10 | Pipeline placement | Authentication MUST precede validation and controller execution on every protected route; authorization MUST follow authentication | `src/modules/auth/auth.routes.ts:12`, `src/modules/users/user.routes.ts:12-18`, `src/modules/customers/customer.routes.ts:14`, `src/modules/products/product.routes.ts:14`, `src/modules/orders/order.routes.ts:14` |
| BR-11 | Public surface | `/health`, `POST /api/v1/auth/register`, and `POST /api/v1/auth/login` MUST remain unauthenticated; every other `/api/v1` route MUST be authenticated | `src/app.ts:62-67`, `src/routes/index.ts:21-30`, `src/modules/auth/auth.routes.ts:10-12` |
| BR-12 | Statelessness | Authentication state MUST NOT be persisted or looked up per request; authorization decisions derive solely from token claims (implicit rule, confidence: high — no DB/cache access exists in the file) | `src/middlewares/auth.middleware.ts` (entire file) |
| BR-13 | Request contract | The middleware MUST augment the Express `Request` type with optional `user?: AuthUser` and `id?: string` so downstream layers can read the principal and request id type-safely | `src/middlewares/auth.middleware.ts:12-17` |
| BR-14 | Secret configuration | Token verification MUST use the single application-wide `JWT_SECRET`, which is validated at startup to be at least 16 characters; invalid configuration terminates the process | `src/config/env.ts:8, 14-27`; `src/middlewares/auth.middleware.ts:41` |
| BR-15 | Error message granularity | The middleware MUST distinguish header-level failure, empty-token failure, and verification failure by message, while sharing the same `401`/`UNAUTHORIZED` code | `src/middlewares/auth.middleware.ts:30, 36, 45` |
| BR-16 | Non-disclosure of credentials | The middleware MUST NOT log, echo, or persist the raw token or header value | `src/middlewares/auth.middleware.ts` (no logging); redaction at `src/shared/logger/index.ts:4-11` |

### 3.2 Detailed breakdown of the business rules

---

### Business Rule: BR-01 — Bearer scheme is mandatory

**Overview**

Every request to a protected route must present an `Authorization` header using the HTTP Bearer authentication scheme. Requests without the header, or with a different scheme (for example `Basic`, `Digest`, or a raw token with no scheme), are rejected before any token parsing occurs.

**Detailed description**

The rule is implemented as the first statement of `authenticate` (`src/middlewares/auth.middleware.ts:28-32`). The middleware reads `req.headers.authorization` and performs a combined presence-and-scheme check using `header.toLowerCase().startsWith('bearer ')`. Because the comparison is case-insensitive, `Bearer`, `bearer`, and `BEARER` are all accepted, matching the case-insensitive nature of HTTP authentication scheme names described in RFC 7235. The trailing space in the literal `'bearer '` is significant: a header consisting only of the word `Bearer` with no following space fails this check and is reported as a missing/invalid header rather than an empty token.

When the check fails, the middleware does not write a response. It constructs `UnauthorizedError('Missing or invalid Authorization header')` and passes it to `next`. The centralized error middleware then produces HTTP `401` with `error.code = 'UNAUTHORIZED'`. This separation means the authentication middleware is transport-agnostic in its failure handling and inherits the uniform JSON error envelope used across the API.

This rule is the outermost gate of the whole protected API. Because it runs before `jwt.verify`, malformed or absent headers cost only a string comparison and never reach cryptographic work. The rule also deliberately does not distinguish "no header" from "wrong scheme" in the response, which reduces the amount of actionable information returned to an unauthenticated caller while still allowing operators to diagnose the generic case from the message text.

**Rule workflow**

```
req.headers.authorization
        |
        +-- undefined / empty -----------------> UnauthorizedError('Missing or invalid Authorization header') -> 401
        |
        +-- value does not start with 'bearer ' (case-insensitive)
        |                                        -> UnauthorizedError('Missing or invalid Authorization header') -> 401
        |
        +-- value starts with 'bearer ' -------> continue to BR-02
```

---

### Business Rule: BR-02 — Extracted token must be non-empty

**Overview**

After the scheme is validated, the middleware strips exactly seven characters (the length of `"bearer "`) and trims surrounding whitespace. The remaining credential must be a non-empty string.

**Detailed description**

The extraction is performed with `header.slice(7).trim()` (`src/middlewares/auth.middleware.ts:34`). Slicing by fixed length is safe here because the prefix check in BR-01 has already established that the first seven characters are the scheme plus one space, regardless of letter case. The subsequent `trim()` normalises both accidental leading whitespace and trailing whitespace, so `"Bearer   token"` and `"Bearer token  "` both yield `token`.

If the resulting string is empty, the middleware raises `UnauthorizedError('Missing bearer token')` (`src/middlewares/auth.middleware.ts:35-38`). This branch exists to separate an explicitly empty credential from a structurally wrong header. A caller sending `Authorization: Bearer ` (with the trailing space) or `Authorization: Bearer    ` reaches this branch. In practice the branch is rarely triggered by well-behaved clients but is reachable, which matters for test completeness: no automated test currently exercises it.

The rule guarantees that `jwt.verify` is never invoked with an empty string. Without this guard, `jsonwebtoken` would throw a `JsonWebTokenError` that would be caught by the same `catch` block, producing the less specific "Invalid or expired token" message. The explicit check therefore improves diagnosability without weakening security, since both outcomes are `401`.

**Rule workflow**

```
"bearer " + remainder
        |
        +-- remainder.trim() === '' ----------> UnauthorizedError('Missing bearer token') -> 401
        |
        +-- remainder.trim() !== '' ----------> token -> continue to BR-03
```

---

### Business Rule: BR-03 — Token must verify against the application secret

**Overview**

The bearer token must be a JWT whose signature validates with the single application secret `env.JWT_SECRET`, and whose time-based claims (`exp`, and by library default `nbf`) are currently valid. Any verification exception is converted into a generic `401`.

**Detailed description**

Verification is a single synchronous call: `jwt.verify(token, env.JWT_SECRET)` inside a `try` block (`src/middlewares/auth.middleware.ts:40-46`). Because no options object is supplied, `jsonwebtoken` applies its defaults: for a symmetric string secret it accepts the HMAC family (HS256/HS384/HS512) rather than a single pinned algorithm, it validates the `exp` claim when present, it validates `nbf` when present, and it does not enforce `issuer`, `audience`, or `subject`. The secret itself comes from startup-validated configuration that requires at least 16 characters and terminates the process when missing or too short (`src/config/env.ts:8, 14-27`).

The signing side lives in the Auth module: `AuthService.signToken` signs `{ sub, email, role }` with `expiresIn: env.JWT_EXPIRES_IN` (`src/modules/auth/auth.service.ts:47-50`). The default lifetime is `8h` (`src/config/env.ts:9`). The middleware does not read `JWT_EXPIRES_IN`; it honours whatever `exp` the token itself carries. Consequently, changing the environment variable affects only newly issued tokens, never the validity window of tokens already in circulation. This asymmetry is an important operational property: shortening `JWT_EXPIRES_IN` does not shorten existing sessions.

All verification failures collapse into one `catch` branch that raises `UnauthorizedError('Invalid or expired token')` (`src/middlewares/auth.middleware.ts:44-46`). This deliberately hides whether the failure was a bad signature, an expired token, a malformed compact serialization, or an unsupported algorithm. The catch is intentionally unattributed (`catch {` with no binding), so no exception detail can leak into logs or responses from this component. A downstream consumer, however, cannot distinguish "expired, please log in again" from "forged" — which is a diagnosability trade-off, not a correctness defect.

One structural gap follows from this rule as implemented: because the payload is only cast, not validated, a token that is genuinely signed with the correct secret but lacks `sub`, `email`, or `role` passes BR-03 and proceeds to BR-04 with `undefined` fields. Such a token can only be produced by someone holding the secret (or by a compromised signing service), but it demonstrates that the middleware verifies authenticity without verifying claim conformance.

**Rule workflow**

```
token + env.JWT_SECRET
        |
        +-- jwt.verify throws (signature, exp, nbf, malformed) -> UnauthorizedError('Invalid or expired token') -> 401
        |
        +-- jwt.verify returns payload ------------------------> continue to BR-04
```

---

### Business Rule: BR-04 — Verified claims hydrate the request principal

**Overview**

A successfully verified token must produce a request-scoped principal object with exactly three fields: `id` from `sub`, `email` from `email`, and `role` from `role`. Downstream layers then treat `req.user` as the authenticated actor.

**Detailed description**

The projection is a direct, non-validating assignment: `req.user = { id: payload.sub, email: payload.email, role: payload.role }` where `payload` is the result of `jwt.verify` cast to the local `JwtPayload` type (`src/middlewares/auth.middleware.ts:41-42`). The type declares `sub`, `email`, and `role` as required, plus optional `iat` and `exp`, but TypeScript types are erased at runtime, so the cast asserts a contract rather than enforcing it. The middleware does not look up the user in the database, does not confirm that the user still exists, and does not compare the token role with the stored role.

Once assigned, `req.user` is the single source of actor identity for the remainder of the request. Three consumers read it directly: `AuthController.me` uses `req.user.id` to fetch the caller's own profile (`src/modules/auth/auth.controller.ts:32-33`), `OrderController.create` uses `req.user.id` as `createdById` (`src/modules/orders/order.controller.ts:30-31`), and `OrderController.changeStatus` uses it as the audit actor (`src/modules/orders/order.controller.ts:40-41`). The request logger also reads `req.user?.id` on response finish to correlate access logs with an actor (`src/middlewares/request-logger.middleware.ts:21`). This wide but read-only consumption is what makes the middleware architecturally central despite its size.

Two consequences follow. First, actor identity is immutable for the duration of the request — no layer re-assigns `req.user`, and there is no mechanism to "re-authenticate" mid-request. Second, because the principal is a denormalised snapshot of token claims, any drift between token and database (role change, email change, account deletion) is invisible until the token expires. The Prisma `User` model has no `active`/`deletedAt` field (`prisma/schema.prisma:25-38`), so even a database-aware implementation would currently have no account-status flag to enforce.

**Rule workflow**

```
verified payload { sub, email, role, iat?, exp? }
        |
        +-- req.user = { id: sub, email, role }
        |
        +-- next() -> requireRole (BR-06/07) -> controller -> logger
```

---

### Business Rule: BR-05 — Role vocabulary is ADMIN or OPERATOR

**Overview**

The only recognised principal roles are `ADMIN` and `OPERATOR`. The rule is expressed in the `AuthUser` type and in `JwtPayload`, and is mirrored by the Prisma `UserRole` enum and the Zod registration schema.

**Detailed description**

`AuthUser.role` is declared as the union `'ADMIN' | 'OPERATOR'` (`src/middlewares/auth.middleware.ts:6-10`), and `JwtPayload.role` reuses that union via `AuthUser['role']` (`src/middlewares/auth.middleware.ts:19-25`). This is a compile-time constraint only. At runtime, `jwt.verify` returns whatever the token contains, and the cast at line 41 does not check it. A token signed with the application secret carrying `role: "SUPERUSER"` would hydrate `req.user.role = "SUPERUSER"`; `requireRole('ADMIN')` and `requireRole('OPERATOR')` would then both reject it with `403` (fail-closed by BR-08), but any consumer that branches on role without an allowlist would observe the unrecognised value.

The vocabulary is consistent across the system: `prisma/schema.prisma:11-14` defines `enum UserRole { ADMIN OPERATOR }`, and `src/modules/auth/auth.schemas.ts:7` restricts registration input to the same two values. `src/modules/users/user.service.ts:7-14` exposes the same union in `PublicUser`. This consistency means the type-level contract is unlikely to drift silently; nevertheless, the middleware is the place where trust in the token becomes trust in the application, and it does not re-assert the vocabulary.

Because the role determines authorization outcomes, the practical effect of this rule is to make the role claim a security-critical piece of data that is asserted solely by signature validity. Any process able to sign with `JWT_SECRET` can mint an `ADMIN` principal.

**Rule workflow**

```
token role claim
        |
        +-- 'ADMIN' | 'OPERATOR' (type-level contract) -> usable by requireRole allowlists
        |
        +-- any other string at runtime --------------> requireRole denies (403); consumers without
                                                        allowlist may observe unexpected value
```

---

### Business Rule: BR-06 — Role guard requires an authenticated principal

**Overview**

`requireRole` must not be usable as a bypass when mounted without `authenticate`. If `req.user` is absent, the request is rejected with `401 UNAUTHORIZED`.

**Detailed description**

The returned handler first checks `if (!req.user)` (`src/middlewares/auth.middleware.ts:51-54`) and calls `next(new UnauthorizedError())`. The `UnauthorizedError` constructor defaults its message to `'Unauthorized'` (`src/shared/errors/http-errors.ts:15-19`). This branch is a defensive invariant: in the current routing configuration it is unreachable because every route that mounts `requireRole` also mounts `authenticate` immediately before it — `src/modules/users/user.routes.ts:12-18` is the only such composition. The branch matters for future composition, where a developer might add a role-guarded route and forget the authentication middleware; in that case the failure mode is a rejection rather than an unintentional open route.

The rule also encodes an ordering requirement: authentication (BR-01..04) must precede authorization (BR-07). Express executes middleware in declaration order, and no router in the project reorders them. `requireRole` therefore acts as a guard that presumes a previous stage has run. It does not attempt to parse the header itself, does not fall back to anonymous principals, and does not cache a negative decision.

Because the branch is unreachable in current tests and current routing, its behavior is only verifiable by inspection. This is a testability gap rather than a logic defect: a one-line unit test invoking `requireRole('ADMIN')` with an empty request object would cover it.

**Rule workflow**

```
requireRole(...roles) handler
        |
        +-- req.user undefined -> UnauthorizedError() -> 401
        |
        +-- req.user present ---> continue to BR-07
```

---

### Business Rule: BR-07 — Principal must hold an allowed role

**Overview**

The authenticated principal's role must appear in the allowlist supplied to `requireRole`. If it does not, the request is rejected with `403 FORBIDDEN` and the message `Insufficient permissions`.

**Detailed description**

The check is `roles.includes(req.user.role)` (`src/middlewares/auth.middleware.ts:55`). On failure, `ForbiddenError('Insufficient permissions')` is raised (`src/middlewares/auth.middleware.ts:56`), which the error middleware renders as HTTP `403` with `error.code = 'FORBIDDEN'` (`src/shared/errors/http-errors.ts:21-25`, `src/middlewares/error.middleware.ts:15-24`). A `403` rather than `401` is semantically correct here: the caller is authenticated, but lacks the required privilege. This distinction is preserved end-to-end and is meaningful to API clients, which can react to `401` by re-authenticating and to `403` by abandoning the operation.

In the current system, exactly one route uses the guard: `GET /api/v1/users/:id` requires `ADMIN` (`src/modules/users/user.routes.ts:12-18`). The guard is declared before parameter validation and before the controller, so a forbidden caller learns nothing about whether the requested user id exists or is a valid UUID. This ordering is a small but real information-disclosure control.

The rule does not constrain which resource a role may act upon. There is no ownership check, no tenant scoping, and no per-resource policy; the role allowlist is the entirety of the authorization decision. That is a deliberate scope of the component, but it means the middleware cannot prevent an `ADMIN` from reading any user, or an `OPERATOR` from mutating any customer, product, or order, because those routers only require authentication.

**Rule workflow**

```
req.user.role
        |
        +-- included in roles[] -> next() -> controller
        |
        +-- not included -------> ForbiddenError('Insufficient permissions') -> 403
```

---

### Business Rule: BR-08 — Allowlist semantics are exact, case-sensitive, and fail-closed

**Overview**

`requireRole` implements a variadic allowlist with exact string matching. It rejects when the allowlist is empty, when the role differs in case, and when the role is undefined.

**Detailed description**

The factory signature `requireRole(...roles: AuthUser['role'][])` (`src/middlewares/auth.middleware.ts:49`) accepts zero or more roles and closes over the array. `Array.prototype.includes` performs a strict equality comparison, so `'admin'` does not match `'ADMIN'` and any non-enumerated value does not match. An empty invocation, `requireRole()`, produces an allowlist for which `includes` is always false, so every authenticated request is rejected with `403`. This is a fail-closed default: forgetting to pass a role denies access rather than granting it.

The factory returns a fresh `RequestHandler` per invocation, allowing different routes to carry independent policies without shared mutable state. Because the closure captures the array by reference, a caller could in principle mutate the array after construction; in practice all call sites pass inline literals (`requireRole('ADMIN')`), so no such mutation exists.

The guard also does not normalise whitespace or case in the principal role. Combined with BR-05's lack of runtime validation, an unrecognised role value is simply denied, which is a safe outcome but produces a `403` where a `401` or a data-integrity error might be more informative. The rule is therefore secure by construction and intentionally simple, at the cost of diagnostic precision.

**Rule workflow**

```
roles[] (possibly empty)
        |
        +-- []                     -> every principal denied (403)
        +-- ['ADMIN']              -> only 'ADMIN' passes
        +-- ['ADMIN','OPERATOR']   -> both pass, anything else 403
        +-- case/whitespace differs -> denied (403)
```

---

### Business Rule: BR-09 — Errors are propagated, never rendered, by the middleware

**Overview**

The middleware must not write HTTP responses. Every failure must be expressed as an `AppError` subtype passed to `next`, so that the centralized error middleware owns the HTTP representation.

**Detailed description**

All five failure points call `next(new UnauthorizedError(...))` or `next(new ForbiddenError(...))` (`src/middlewares/auth.middleware.ts:30, 36, 45, 52, 56`), and the success paths call bare `next()` (`:43, :59`). No branch touches `res`. This is consistent with every other middleware and controller in the codebase (`validate.middleware.ts:31`, controllers in all modules). The benefit is a single, uniform error envelope: `{ error: { code, message, details? } }` (`src/middlewares/error.middleware.ts:15-24`), with machine-readable codes `UNAUTHORIZED` and `FORBIDDEN` defined in the shared taxonomy (`src/shared/errors/http-errors.ts:15-25`).

Because error creation happens at the point of detection, each branch can attach a precise message while retaining the shared status/code mapping. The error middleware distinguishes `instanceof AppError` first, so authentication errors never fall through to the generic `500` path or to the Prisma/Zod branches. The `AppError` base class captures a stack trace (`src/shared/errors/app-error.ts:14`), but these are expected control-flow errors, not logged as unhandled: the error middleware's logging branch (`error.middleware.ts:56-60`) is only reached for non-`AppError` errors.

The rule has one testability implication: exercising the middleware in isolation requires either a fake `next` and assertions on the error instance, or integration through the full app. The existing suite always uses the latter, which is why the failure branches are only covered when a route is hit without credentials.

**Rule workflow**

```
failure detected
        |
        +-- new UnauthorizedError | ForbiddenError -> next(err) -> errorMiddleware
                                                                    |
                                                                    +-> 401/403 JSON envelope
                                                                    +-> handler never runs
```

---

### Business Rule: BR-10 — Middleware ordering on protected routes

**Overview**

On every protected route, `authenticate` must run before `validate` and before the controller; `requireRole`, where present, must run after `authenticate` and before validation.

**Detailed description**

The ordering is visible in each router. `auth.routes.ts:12` mounts `authenticate` before `AuthController.me`. `user.routes.ts:12-18` mounts `authenticate`, then `requireRole('ADMIN')`, then `validate({ params: idParamSchema })`, then the controller. `customer.routes.ts:14`, `product.routes.ts:14`, and `order.routes.ts:14` apply `router.use(authenticate)` before any route-specific middleware, so authentication precedes validation on every endpoint in those modules. In `app.ts:59-67`, JSON body parsing and request logging precede the API router, but they are not authorization-relevant.

The consequence is that unauthenticated requests are rejected before Zod parsing and before any database access. This reduces attack surface (no schema errors leaked to anonymous callers, no repository queries performed for unauthenticated traffic) and controls cost (the cheapest checks run first). Placing `requireRole` before `validate` in the users router additionally ensures a non-admin cannot probe id validity through validation errors.

The rule is convention-enforced, not structurally enforced: nothing prevents a future router from mounting `validate` before `authenticate`. The middleware itself cannot detect misordering, except indirectly through BR-06, which catches the specific case of `requireRole` without a principal.

**Rule workflow**

```
router chain: requestLogger -> authenticate -> requireRole? -> validate -> controller
                                    |               |             |
                                    +-- 401         +-- 403       +-- 400
```

---

### Business Rule: BR-11 — Public route surface is explicitly limited

**Overview**

Only the health probe and the two credential-acquisition endpoints are unauthenticated. Every other `/api/v1` route requires a valid bearer token.

**Detailed description**

`GET /health` is registered directly on the app before the API router and has no middleware (`src/app.ts:62-64`). The auth router declares `POST /register` and `POST /login` before the only guarded route, `GET /me` (`src/modules/auth/auth.routes.ts:10-12`). The users, customers, products, and orders routers all apply `authenticate` at router level or per route (`user.routes.ts:14`, `customer.routes.ts:14`, `product.routes.ts:14`, `order.routes.ts:14`). No other router is mounted (`src/routes/index.ts:21-30`), and unmatched routes fall through to a `NotFoundError` handler (`src/app.ts:69-71`), which is itself unauthenticated but returns `404` regardless of credentials.

This rule defines the component's effective blast radius: compromising or disabling `authenticate` exposes the entire business API (customer PII, product catalogue and pricing, order lifecycle), while `/auth/*` and `/health` remain open by design. It also means the unauthenticated surface is small enough to enumerate and review, which is a positive property.

A notable interaction: because `POST /auth/register` is public and accepts a caller-supplied `role` including `ADMIN` (`src/modules/auth/auth.schemas.ts:7`), the authentication boundary can be legitimately crossed by anyone who can create a user. The middleware faithfully authenticates the resulting principal; it cannot compensate for an over-permissive provisioning rule upstream.

**Rule workflow**

```
/health                         -> no authenticate
/api/v1/auth/register, /login   -> no authenticate
/api/v1/auth/me                 -> authenticate
/api/v1/users/:id               -> authenticate + requireRole('ADMIN')
/api/v1/customers/*             -> authenticate
/api/v1/products/*              -> authenticate
/api/v1/orders/*                -> authenticate
unmatched                       -> 404 (no authenticate)
```

---

### Business Rule: BR-12 — Stateless, claim-derived authorization (implicit)

**Overview**

The middleware performs no persistence, cache access, or outbound call. All authentication and authorization decisions derive from the presented token and the in-process secret.

**Detailed description**

Confidence: **high**. The entire file imports only `express` (type-only), `jsonwebtoken`, the env module, and the error barrel (`src/middlewares/auth.middleware.ts:1-4`). There is no repository, no Prisma client, no Redis client, and no HTTP client. Verification is a local HMAC computation, so the per-request cost is CPU-bound and constant relative to external systems.

The benefit is horizontal scalability and resilience: the middleware cannot fail because a database or identity provider is unavailable, and instances share no session state. The cost is the complete absence of revocation: deleting a user, demoting a role, or rotating a compromised credential has no effect until the token's `exp` passes (default `8h`). The system has no logout endpoint and no denylist (`src/modules/auth/auth.routes.ts` contains only register, login, me), and there is a single `JWT_SECRET` for every token, so rotation invalidates all sessions at once.

This rule is implicit in the implementation rather than documented in a comment or ADR. It should be treated as an architectural decision with security consequences, not merely an implementation detail.

**Rule workflow**

```
request -> local HMAC verify -> local claim projection -> local role comparison -> next
(no network, no DB, no cache, no shared state)
```

---

### Business Rule: BR-13 — Express Request type augmentation contract

**Overview**

The component augments the Express `Request` interface with `user?: AuthUser` and `id?: string`, making the principal and request id first-class, type-safe properties for all downstream code.

**Detailed description**

The declaration merge targets `'express-serve-static-core'` (`src/middlewares/auth.middleware.ts:12-17`). Both properties are optional, which accurately models the runtime state: before `authenticate` runs there is no user, and requests that never reach `requestLogger` would have no id. Making them optional forces consumers to narrow before use; `AuthController.me`, `OrderController.create`, and `OrderController.changeStatus` all perform explicit `if (!req.user) throw new UnauthorizedError()` checks (`auth.controller.ts:32`, `order.controller.ts:30, 40`), which is defensive but redundant on routes where `authenticate` already guarantees a user.

A cohesion observation: `id?: string` is declared here but is owned and set by `requestLogger` (`src/middlewares/request-logger.middleware.ts:7`), and consumed by `errorMiddleware` (`error.middleware.ts:56`) and the logger (`request-logger.middleware.ts:21`). The auth middleware is thus the type-host for an unrelated cross-cutting property. Functionally harmless, but it couples the request-id contract to the auth module's declaration file and slightly muddies the component's responsibility boundary.

TypeScript declaration merging is global: once this file is part of the compilation, every `Request` in the project accepts `req.user`. There is no runtime guard that the augmentation matches reality, so a route that reads `req.user` without mounting `authenticate` compiles successfully and fails only at runtime (which the controllers defensively handle with `401`).

**Rule workflow**

```
auth.middleware.ts (declare module 'express-serve-static-core')
        |
        +-- Request.user?: AuthUser  -> consumed by controllers, requireRole, logger
        +-- Request.id?: string      -> set by requestLogger, consumed by errorMiddleware/logger
```

---

### Business Rule: BR-14 — Secret management and fail-fast configuration

**Overview**

Verification must use `env.JWT_SECRET`, a value validated at process start to be at least 16 characters. Missing or invalid configuration aborts the process before the server listens.

**Detailed description**

`src/config/env.ts:3-10` defines a Zod schema requiring `DATABASE_URL` and `JWT_SECRET` (`min(16)`), with defaults for `NODE_ENV`, `PORT`, `LOG_LEVEL`, and `JWT_EXPIRES_IN`. `loadEnv` uses `safeParse`, prints all issues, and calls `process.exit(1)` on failure (`src/config/env.ts:14-27`). Because `env` is a module-level singleton imported by the auth middleware, the validation runs during module loading, before `buildApp` is called and before `app.listen` in `src/server.ts:6-11`. The system therefore cannot start with a missing or short secret.

The 16-character minimum is a floor, not a strength requirement: a 16-character low-entropy string passes. `.env.example:8` ships a human-readable placeholder (`change-me-in-production-please-use-a-long-random-secret`), and `docker-compose.yml` supplies environment values for local development. There is one secret for signing and verification, held in process memory, with no key id (`kid`) header, no key rotation support, and no asymmetric option. If `JWT_SECRET` leaks, any party can mint tokens for any user and role, and the middleware has no secondary signal (issuer, audience, or database check) to reject them.

The middleware itself reads only `JWT_SECRET`; `JWT_EXPIRES_IN` is consumed exclusively by `AuthService.signToken` (`src/modules/auth/auth.service.ts:48-49`). This separation is correct but worth noting operationally: verification duration is governed by the token, not the current configuration.

**Rule workflow**

```
process start -> env.ts loadEnv() -> Zod safeParse
                                        |
                                        +-- failure -> console.error + process.exit(1)
                                        +-- success -> env.JWT_SECRET available to jwt.verify
```

---

### Business Rule: BR-15 — Distinct messages, single error code

**Overview**

The middleware distinguishes three failure messages while returning the same `401` status and `UNAUTHORIZED` code for all authentication failures.

**Detailed description**

The three messages are "Missing or invalid Authorization header" (`:30`), "Missing bearer token" (`:36`), and "Invalid or expired token" (`:45`). The `UnauthorizedError` class fixes status `401` and code `UNAUTHORIZED` regardless of message (`src/shared/errors/http-errors.ts:15-19`). Clients can therefore branch reliably on the code while operators can read the message for diagnosis. The role guard's failure is separately coded as `FORBIDDEN` with a fixed message, so clients can also distinguish authentication from authorization.

The trade-off is a small information leak: a caller can learn whether the failure was at header level, token level, or signature/expiry level. Since all three are trivially distinguishable by an attacker crafting requests, and none reveals secret material, the practical risk is low. A stricter design would return one generic message; the current design favours operability.

**Rule workflow**

```
failure type            -> message                                  -> status/code
header absent/wrong     -> 'Missing or invalid Authorization header' -> 401 UNAUTHORIZED
empty token             -> 'Missing bearer token'                    -> 401 UNAUTHORIZED
verify failure          -> 'Invalid or expired token'                -> 401 UNAUTHORIZED
role mismatch           -> 'Insufficient permissions'                -> 403 FORBIDDEN
no principal in guard   -> 'Unauthorized'                           -> 401 UNAUTHORIZED
```

---

### Business Rule: BR-16 — Credentials are never logged or echoed

**Overview**

The middleware must not emit the raw `Authorization` header, the bearer token, or the decoded payload into logs or responses.

**Detailed description**

There is no logging statement in `src/middlewares/auth.middleware.ts`, and no error message interpolates the token. Failure messages are fixed strings. On the response side, the token is only ever read, never written. The request logger does not log headers; it logs method, path, status, duration, and `req.user?.id` (`src/middlewares/request-logger.middleware.ts:14-24`). As a second layer of defence, the Pino logger configures redaction for `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, and `*.accessToken` (`src/shared/logger/index.ts:4-11`), so accidental structured logging of a request or response object would be censored.

The only credential-adjacent value that crosses into observability is the authenticated user id, which is generally acceptable for audit correlation but does link activity to an identity in logs. Token contents (including email) are not logged. The login flow's response does contain the access token by design (`src/modules/auth/auth.service.ts:40-44`), but that is outside this component.

**Rule workflow**

```
token / header
        |
        +-- middleware: never logged, never interpolated, never returned
        +-- request logger: logs only userId
        +-- pino redaction: authorization/cookie/token paths censored as [REDACTED]
```

---

## 4. Component Structure

### 4.1 File inventory and internal structure

```
src/middlewares/auth.middleware.ts        # 61 lines - the entire component
├── imports (lines 1-4)
│   ├── type { RequestHandler } from 'express'        # type-only, erased at runtime
│   ├── jwt from 'jsonwebtoken'                        # only runtime library dependency
│   ├── { env } from '../config/env.js'                # JWT_SECRET source
│   └── { UnauthorizedError, ForbiddenError }
│       from '../shared/errors/index.js'               # error taxonomy
├── AuthUser type (lines 6-10)
│   └── { id: string; email: string; role: 'ADMIN' | 'OPERATOR' }
├── Express Request augmentation (lines 12-17)
│   ├── user?: AuthUser
│   └── id?: string
├── JwtPayload type (lines 19-25)
│   └── { sub, email, role, iat?, exp? }
├── export const authenticate: RequestHandler (lines 27-47)
│   ├── header presence + scheme check (28-32)
│   ├── token extraction + empty check (34-38)
│   ├── jwt.verify + try/catch (40-46)
│   └── req.user projection + next() (42-43)
└── export function requireRole(...roles): RequestHandler (49-61)
    ├── principal presence check (51-54)
    ├── allowlist membership check (55-58)
    └── next() (59)
```

### 4.2 Position within the project

```
src/
├── app.ts                              # builds Express app, mounts /api/v1, error middleware
├── server.ts                           # HTTP listener, graceful shutdown
├── routes/
│   └── index.ts                        # aggregates the five module routers
├── middlewares/
│   ├── auth.middleware.ts              # <-- ANALYZED COMPONENT
│   ├── validate.middleware.ts          # Zod body/query/params parsing
│   ├── request-logger.middleware.ts    # request id + access logging (reads req.user.id)
│   └── error.middleware.ts             # AppError/Zod/Prisma -> HTTP JSON
├── config/
│   ├── env.ts                          # JWT_SECRET, JWT_EXPIRES_IN, fail-fast validation
│   └── database.ts                     # Prisma singleton
├── shared/
│   ├── errors/                         # AppError + UnauthorizedError + ForbiddenError
│   ├── http/response.ts
│   └── logger/index.ts                 # Pino with credential redaction
└── modules/
    ├── auth/       (controller, service, routes, schemas)
    ├── users/      (controller, service, repository, routes, schemas)   # only requireRole call site
    ├── customers/  (controller, service, repository, routes, schemas)
    ├── products/   (controller, service, repository, routes, schemas)
    └── orders/     (controller, service, repository, routes, schemas)
```

### 4.3 Internal cohesion assessment

The file contains two exported behaviors plus three supporting type contracts. `authenticate` and `requireRole` share the same `AuthUser` vocabulary and the same error taxonomy, and both operate on `req.user`. Cohesion is **functional** for the pair (both are security decision points in the same request pipeline) but the file mixes two responsibilities: credential verification and privilege evaluation. They are composable and independently mountable, which is a recognised and appropriate middleware pattern; the separation is by function rather than by file. The `id?: string` augmentation is the only element that is arguably out of place, since request-id ownership lives in `request-logger.middleware.ts`.

---

## 5. Dependency Analysis

### 5.1 Internal dependencies (efferent — what this component depends on)

```
src/middlewares/auth.middleware.ts
├── src/config/env.ts                     # env.JWT_SECRET (verification key)
│   └── external: zod                     # schema validation of process.env
└── src/shared/errors/index.ts            # barrel
    ├── src/shared/errors/app-error.ts    # AppError base (statusCode, errorCode, details)
    └── src/shared/errors/http-errors.ts  # UnauthorizedError (401), ForbiddenError (403)
```

| Dependency | Kind | Symbols used | Coupling strength | Notes |
|------------|------|--------------|-------------------|-------|
| `src/config/env.ts` | Internal module | `env` (singleton) | High (shared secret contract) | Reading `JWT_SECRET` binds verification to the signing key; any change to config keys breaks auth |
| `src/shared/errors/index.ts` | Internal barrel | `UnauthorizedError`, `ForbiddenError` | Medium | Only two of nine error types used; barrel import pulls a stable contract module |
| `express` | External (`@types/express` 4.17.21) | `RequestHandler` (type-only) | Low at runtime, medium at compile time | `import type` erased by TypeScript; no runtime Express API is used |
| `jsonwebtoken` 9.0.2 | External runtime | `verify` (default import) | High | Sole runtime library; verification semantics (algorithms, exp/nbf) are inherited from it |

### 5.2 Afferent dependencies (who depends on this component)

```
src/modules/auth/auth.routes.ts:3,12        -> authenticate (GET /me)
src/modules/users/user.routes.ts:3,14,15    -> authenticate + requireRole('ADMIN')
src/modules/customers/customer.routes.ts:2,14 -> authenticate (router-level)
src/modules/products/product.routes.ts:2,14   -> authenticate (router-level)
src/modules/orders/order.routes.ts:2,14       -> authenticate (router-level)

Indirect (consume the augmented Request contract):
src/modules/auth/auth.controller.ts:32-33    -> req.user.id
src/modules/orders/order.controller.ts:30,40 -> req.user.id
src/middlewares/request-logger.middleware.ts:21 -> req.user?.id
```

| Dependent | Symbols used | Criticality | Notes |
|-----------|--------------|-------------|-------|
| `src/modules/auth/auth.routes.ts` | `authenticate` | High | Guards identity endpoint |
| `src/modules/users/user.routes.ts` | `authenticate`, `requireRole` | High | Only role-based policy in the system |
| `src/modules/customers/customer.routes.ts` | `authenticate` | High | All customer CRUD depends on it |
| `src/modules/products/product.routes.ts` | `authenticate` | High | All product CRUD depends on it |
| `src/modules/orders/order.routes.ts` | `authenticate` | High | All order lifecycle depends on it; actor id feeds audit history |
| `src/modules/auth/auth.controller.ts` | `req.user` (type) | Medium | Defensive `if (!req.user)` check |
| `src/modules/orders/order.controller.ts` | `req.user` (type) | High | `createdById` and status-change actor |
| `src/middlewares/request-logger.middleware.ts` | `req.user` (type) | Low | Log enrichment only |

### 5.3 Dependency chains (runtime)

```
HTTP request
  -> app.ts -> routes/index.ts -> module router
       -> auth.middleware.authenticate
            -> config/env.ts (env.JWT_SECRET)
            -> jsonwebtoken.verify
            -> shared/errors (UnauthorizedError)
       -> auth.middleware.requireRole (users router only)
            -> shared/errors (UnauthorizedError, ForbiddenError)
       -> error.middleware (renders AppError)
```

No circular dependencies exist. The component sits in the middle of the graph: it is a **stable-ish security contract** with 5 direct consumers and 2 internal dependencies, and it imports only leaf/stable modules (`config`, `shared/errors`), both of which have no dependency on the middleware.

### 5.4 Token lifecycle dependency (cross-component)

The middleware's correctness depends on a producer it does not import: `AuthService.signToken` (`src/modules/auth/auth.service.ts:47-50`) must sign with the same `env.JWT_SECRET` and the same claim names. This is a **contractual (implicit) dependency** enforced only by convention. Renaming the `sub` claim, changing the role claim name, or switching to an asymmetric key would silently break authentication without any compile-time error, because both sides converge only on `env.JWT_SECRET` and on a locally declared payload type.

---

## 6. Afferent and Efferent Coupling

Methodology: files within `src` (excluding `node_modules`, `dist`, tests) that import the component, and internal files the component imports. External npm packages are listed separately and excluded from the counts, consistent with the architectural report baseline (`docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md:125-127`).

### 6.1 File-level coupling

| Component | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Instability (Ce/(Ca+Ce)) | Critical |
|-----------|------------------------|------------------------|--------------------------|----------|
| `src/middlewares/auth.middleware.ts` (whole file) | 5 | 2 | 0.29 | High |

Afferent (5): `auth.routes.ts`, `user.routes.ts`, `customer.routes.ts`, `product.routes.ts`, `order.routes.ts`.
Efferent (2): `config/env.ts`, `shared/errors/index.ts`.

### 6.2 Sub-component (exported symbol) coupling

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|----------|
| `authenticate` (RequestHandler) | 5 route files | 3 internal symbols (`env`, `UnauthorizedError`, `jwt`) + 1 external (`jsonwebtoken`) | High |
| `requireRole` (factory) | 1 route file (`user.routes.ts:15`) | 2 internal symbols (`UnauthorizedError`, `ForbiddenError`) | Medium |
| `AuthUser` (type) | 0 direct imports; 3 indirect consumers via `Request.user` (`auth.controller.ts`, `order.controller.ts`, `request-logger.middleware.ts`) | 0 | Medium |
| `JwtPayload` (type) | 0 | 0 (uses `AuthUser['role']`) | Low |

### 6.3 Interpretation

- **Low efferent coupling (2)** means the middleware is easy to reason about and hard to break by unrelated changes. It depends only on configuration and the error taxonomy, both leaf modules with high stability.
- **Moderate afferent coupling (5)** is essentially the entire routing layer. Because all feature routers depend on `authenticate`, any behavioural change (for example adding an audience check) is a system-wide security change. The instability metric of 0.29 correctly classifies the component as more stable than its consumers, which is appropriate for a security gate.
- **`requireRole` is heavily under-adopted relative to its design.** One call site means its branching logic is exercised on a single endpoint and has no test coverage at all. From a coupling standpoint, the component exports a capability the system barely uses.
- **Type-level coupling is wider than import-level coupling.** The `AuthUser` shape propagates to three additional files through global `Request` augmentation. This is invisible to import-graph analysis but real: changing `AuthUser.role` would produce compile errors in `auth.controller.ts`, `order.controller.ts`, and `request-logger.middleware.ts` indirectly.

---

## 7. Endpoints (Protected Surface)

The Authentication Middleware **does not expose any endpoints of its own**; it is a cross-cutting pipeline component. The table below documents the endpoints on which it is mounted, which defines its effective contract surface. Statuses in the "Auth behavior" column describe what the middleware contributes, not the full endpoint contract.

| Endpoint | Method | Protection Applied | Auth Behavior | Location |
|----------|--------|--------------------|---------------|----------|
| `/api/v1/auth/register` | POST | None (public) | Bypasses the middleware | `src/modules/auth/auth.routes.ts:10` |
| `/api/v1/auth/login` | POST | None (public) | Bypasses the middleware; issues the token the middleware later verifies | `src/modules/auth/auth.routes.ts:11` |
| `/api/v1/auth/me` | GET | `authenticate` | 401 without valid token; sets `req.user` for self-lookup | `src/modules/auth/auth.routes.ts:12` |
| `/api/v1/users/:id` | GET | `authenticate` + `requireRole('ADMIN')` | 401 without valid token; 403 for `OPERATOR` | `src/modules/users/user.routes.ts:12-18` |
| `/api/v1/customers` | GET, POST | `authenticate` (router-level) | 401 without valid token; any role allowed | `src/modules/customers/customer.routes.ts:14-24` |
| `/api/v1/customers/:id` | GET, PATCH, DELETE | `authenticate` (router-level) | 401 without valid token; any role allowed | `src/modules/customers/customer.routes.ts:14-24` |
| `/api/v1/products` | GET, POST | `authenticate` (router-level) | 401 without valid token; any role allowed | `src/modules/products/product.routes.ts:14-24` |
| `/api/v1/products/:id` | GET, PATCH, DELETE | `authenticate` (router-level) | 401 without valid token; any role allowed | `src/modules/products/product.routes.ts:14-24` |
| `/api/v1/orders` | GET, POST | `authenticate` (router-level) | 401 without valid token; `req.user.id` used as creator/actor | `src/modules/orders/order.routes.ts:14-18` |
| `/api/v1/orders/:id` | GET, DELETE | `authenticate` (router-level) | 401 without valid token; any role allowed | `src/modules/orders/order.routes.ts:14-24` |
| `/api/v1/orders/:id/status` | PATCH | `authenticate` (router-level) | 401 without valid token; `req.user.id` recorded in status history | `src/modules/orders/order.routes.ts:19-23` |
| `/health` | GET | None (public) | Bypasses the middleware | `src/app.ts:62-64` |

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| `jsonwebtoken@9.0.2` | In-process library | Signature, `exp`, and `nbf` verification | Local function call (HMAC-SHA) | Compact JWS string -> JS object | All thrown errors caught and mapped to `UnauthorizedError` (`auth.middleware.ts:44-46`) |
| `src/config/env.ts` | Internal configuration module | Supplies `JWT_SECRET` (and validates it at startup) | In-process module import | `Env` object from Zod-parsed `process.env` | Missing/short secret -> `console.error` + `process.exit(1)` (`env.ts:16-23`) |
| `src/shared/errors` (`AppError` taxonomy) | Internal contract | Express failures as HTTP-mappable errors | In-process `next(err)` | `AppError` subclass instances | Rendered by `errorMiddleware` as `{ error: { code, message } }` (`error.middleware.ts:15-24`) |
| Express request pipeline | Framework contract | Middleware composition and `req` mutation | In-process function invocation | `IncomingMessage`-derived `Request` | `next()` for success, `next(err)` for failure; no direct response writes |
| Express `Request` interface | Type-level integration | Exposes `req.user` and `req.id` to all layers | TypeScript declaration merging | Compile-time types only | None (erased at runtime) |
| `AuthService.signToken` (producer) | Implicit cross-component contract | Must sign with same secret and same claim names | Shared `env.JWT_SECRET` | JWT claims `sub`, `email`, `role` | No shared type; drift possible without compile error (`auth.service.ts:47-50`) |
| Pino logger (indirect) | Observability | Access logs include authenticated `userId` | In-process logger call in `requestLogger` | Structured JSON log fields | Redaction of authorization/token paths (`logger/index.ts:4-11`) |

**No database, cache, message broker, or external HTTP service is integrated by this component.** This is a defining property: the integration surface is entirely in-process.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Chain of Responsibility / Middleware | `authenticate` and `requireRole` are Express `RequestHandler`s that either call `next()` or `next(err)` | `src/middlewares/auth.middleware.ts:27-47, 49-61` | Composable request-processing pipeline |
| Guard / Policy (RBAC) | `requireRole(...roles)` allowlist check | `src/middlewares/auth.middleware.ts:49-61` | Role-based access control primitive reused by routes |
| Factory / Higher-Order Function | `requireRole` returns a configured handler per call | `src/middlewares/auth.middleware.ts:49-50` | Per-route policy configuration with closure state |
| Fail-Closed Default | Empty allowlist denies all; missing `req.user` denies; verification failure denies | `src/middlewares/auth.middleware.ts:51-58` | Security-safe behavior on misconfiguration |
| Dependency Injection (module-level) | `env` singleton injected via import; errors via import | `src/middlewares/auth.middleware.ts:3-4` | No constructor wiring; configuration resolved once at startup |
| Centralized Error Translation | Throws `AppError` subtypes, delegates HTTP mapping to `errorMiddleware` | `src/middlewares/auth.middleware.ts:30,36,45,52,56` -> `error.middleware.ts:15-24` | Uniform error envelope, separation of policy from transport |
| Type Augmentation (declaration merging) | `declare module 'express-serve-static-core'` adds `user` and `id` | `src/middlewares/auth.middleware.ts:12-17` | Type-safe request context across layers |
| Stateless Token Authentication | JWT bearer verification, no session store | `src/middlewares/auth.middleware.ts:40-43` | Horizontal scalability, no shared session state |
| Separation of Authentication and Authorization | Two independent middlewares composed per route | `src/middlewares/auth.middleware.ts` + route files | Independent reuse; authorization optional per route |
| Fail-Fast Configuration | Zod-validated env with `process.exit(1)` | `src/config/env.ts:14-27` | Prevents start with weak/missing secret |

### Architectural decisions observed

1. **Symmetric JWT (HS256 family) with a single application secret.** Signing and verification share one secret, so there is no public-key distribution and no `kid`-based rotation. This keeps the component self-contained but makes global secret compromise equal to global identity compromise.
2. **Authentication is a cross-cutting middleware, not per-controller logic.** Enforced by mounting on routers; controllers only read `req.user`.
3. **Authorization is opt-in per route.** Authentication is applied broadly; role checks only where designed. This produces a system where most routes are authenticated-but-unrestricted, a deliberate but coarse posture.
4. **No per-request identity revalidation.** The token is the authoritative identity snapshot; the database is not consulted. Trade-off: latency and resilience versus revocation.
5. **Error policy is centralized.** The middleware contains no HTTP status constants, which keeps status mapping single-sourced.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Evidence | Impact |
|------------|----------------|-------|----------|--------|
| High | Token verification | No `algorithms` allowlist passed to `jwt.verify`; library defaults accept the HMAC family rather than a pinned algorithm | `src/middlewares/auth.middleware.ts:41` | Wider algorithm surface than intended; reduces defence against algorithm-confusion classes of attack |
| High | Identity freshness | No per-request user existence, status, or role revalidation; role changes and deletions are not enforced until `exp` | `src/middlewares/auth.middleware.ts:40-43`; no repository import | Demoted or deleted users retain privileges for up to 8 hours (`src/config/env.ts:9`) |
| High | Session lifecycle | No logout, denylist, refresh token, or revocation mechanism; single `JWT_SECRET` for all tokens | `src/modules/auth/auth.routes.ts:10-12`; `src/middlewares/auth.middleware.ts` | A stolen token cannot be invalidated except by global secret rotation |
| High | Payload integrity | `jwt.verify` result is type-cast, not validated; missing/wrong-typed `sub`/`role` produces an invalid principal silently | `src/middlewares/auth.middleware.ts:41-42` | Malformed but correctly signed tokens yield `req.user` with `undefined` fields; downstream DB errors or silent authorization denial |
| Medium | Token claiming | No `issuer` or `audience` validation | `src/middlewares/auth.middleware.ts:41` | A token minted by any other system sharing the secret would be accepted |
| Medium | Authorization scope | `requireRole` is used on only one route; customers/products/orders require authentication only | `src/modules/users/user.routes.ts:15`; other routers `:14` | Any authenticated `OPERATOR` can mutate all business data; no tenant/ownership scoping exists at any layer |
| Medium | Configuration | `JWT_SECRET` minimum length is 16 characters; `.env.example` ships a placeholder secret | `src/config/env.ts:8`; `.env.example:8` | Low-entropy secrets accepted; copy-paste deployment risk |
| Medium | Test coverage | No unit tests for the middleware; `requireRole` entirely unexercised; invalid/expired/empty-token branches untested | `tests/` (see Section 11) | Regressions in security-critical branches would not be detected |
| Medium | Rate limiting | No throttling or brute-force protection around token presentation or login | `src/middlewares/auth.middleware.ts`; `src/modules/auth/auth.routes.ts` | Unlimited credential/token guessing attempts (login enumeration also possible) |
| Low | HTTP semantics | No `WWW-Authenticate` header on `401` responses | `src/middlewares/error.middleware.ts:15-24` | Does not fully conform to RFC 6750 challenge semantics; clients cannot discover the scheme from the response |
| Low | Observability | All verification failures collapse to one message; no structured reason (expired vs invalid signature) is surfaced | `src/middlewares/auth.middleware.ts:44-46` | Harder operations triage; no metrics distinguishing expiry from tampering |
| Low | Cohesion | `req.id` type augmentation is declared in the auth middleware although owned by `requestLogger` | `src/middlewares/auth.middleware.ts:12-17` vs `request-logger.middleware.ts:7` | Misleading ownership; request-id contract tied to auth module |
| Low | Header typing | `req.headers.authorization` is assumed to be a string; array-shaped header values would throw at `.toLowerCase()` and surface as `500` | `src/middlewares/auth.middleware.ts:28-29` | Theoretical (Node joins duplicate headers for this header), but unguarded |
| Low | Token size | No maximum token length check before cryptographic work | `src/middlewares/auth.middleware.ts:34-41` | Very large tokens incur parsing/HMAC cost; minor DoS surface |
| Low | Contract duplication | JWT claim names are declared independently in `auth.middleware.ts` and `auth.service.ts` with no shared type | `auth.middleware.ts:19-25`; `auth.service.ts:47-50` | Claim rename would break auth silently with no compile-time protection |
| Low | Registration coupling (upstream) | Public registration accepts caller-supplied `role`, including `ADMIN`; middleware then authenticates the resulting principal | `src/modules/auth/auth.schemas.ts:7` | Middleware is correct, but the trust boundary it protects can be crossed legitimately; privilege escalation risk originates upstream |

---

## 11. Test Coverage Analysis

### 11.1 Test inventory and locations

| Test file | Type | Relevance to this component |
|-----------|------|-----------------------------|
| `tests/auth.test.ts` | Integration (Supertest against real Express app + MySQL) | Direct: exercises `authenticate` via `GET /api/v1/auth/me` with and without token; login/register produce tokens |
| `tests/orders.test.ts` | Integration | Indirect: every request carries `Bearer <token>`, exercising the success path of `authenticate` 8 times; `req.user.id` is persisted as creator/actor |
| `tests/helpers/factories.ts` | Test infrastructure | `loginAndGetToken` (`:33-40`) and `bootstrapAuthenticatedUser` (`:76-83`) manufacture valid tokens by calling the real login endpoint; `createTestUser` seeds bcrypt-hashed users |
| `tests/setup.ts` | Test infrastructure | Connects Prisma, truncates all tables before each test (including `user`), disconnects after suite |
| `vitest.config.ts` | Configuration | Serial execution, single fork, 30s timeouts, `tests/**/*.test.ts` include pattern |

There are **no unit tests** targeting `src/middlewares/auth.middleware.ts` directly (no test file imports `authenticate` or `requireRole`). All coverage is therefore obtained indirectly through HTTP integration tests.

### 11.2 Coverage assessment

| Component area | Unit Tests | Integration Tests | Estimated coverage | Test Quality |
|----------------|------------|-------------------|--------------------|--------------|
| `authenticate` — missing header branch (BR-01) | 0 | 1 (`tests/auth.test.ts:79-83`) | Branch covered; asserts only HTTP 401, not `error.code` or message | Fair — the only negative authentication test |
| `authenticate` — wrong scheme / malformed header | 0 | 0 | Not covered | Gap |
| `authenticate` — empty token branch (BR-02) | 0 | 0 | Not covered (`auth.middleware.ts:35-38` never executed) | Gap |
| `authenticate` — valid token success path (BR-03, BR-04) | 0 | 1 explicit (`tests/auth.test.ts:65-77`) + 8 indirect (`tests/orders.test.ts`) | Strong positive coverage; asserts `req.user` indirectly via `/auth/me` body and order actor persistence | Good — verifies end-to-end identity projection |
| `authenticate` — invalid signature / malformed token | 0 | 0 | Not covered (`catch` branch at `:44-46` never executed) | Gap |
| `authenticate` — expired token | 0 | 0 | Not covered | Gap |
| `authenticate` — case-insensitive `bearer` scheme | 0 | 0 | Not covered | Minor gap |
| `requireRole` — allow path (ADMIN) | 0 | 0 | Not covered (`GET /api/v1/users/:id` has no test) | Gap |
| `requireRole` — deny path (OPERATOR -> 403) | 0 | 0 | Not covered | Significant gap |
| `requireRole` — missing principal -> 401 (BR-06) | 0 | 0 | Not covered (branch unreachable in current routing) | Gap (defensive code unverified) |
| `requireRole` — empty allowlist fail-closed (BR-08) | 0 | 0 | Not covered | Gap |
| Module-level wiring (routers mount middleware correctly) | 0 | Implicitly exercised by every authenticated request | Covered for `authenticate`; `requireRole` wiring compiled but unexecuted | Fair |
| `AuthUser` / `Request` augmentation | 0 | Indirectly via controller access to `req.user.id` | Covered at compile time by `tsc`; runtime implicitly | Fair |

### 11.3 Approximate quantitative estimate

No coverage tooling was executed (per analysis constraints). Based on static path analysis:

- **`authenticate` (lines 27-47):** executed lines ≈ 27-32, 34, 40-43, 47 (plus function entry). Unexecuted ≈ 35-38 (empty token) and 44-46 (verification failure). Statement estimate ≈ 70%; branch outcomes covered ≈ 3 of 6 (header present/absent, verify success; missing: empty-token hit, verify failure, non-`bearer` scheme as distinct case).
- **`requireRole` (lines 49-61):** line 49 executes once at app construction (`requireRole('ADMIN')` in `user.routes.ts:15`); the returned handler body (51-60) never executes in the suite. Statement estimate ≈ 10-15%; branch estimate ≈ 0%.
- **Whole file:** statement estimate ≈ **55-60%**, branch estimate ≈ **20-30%**, with all authorization branches at 0%.

These figures are estimates from static inspection and should be treated as indicative, not measured.

### 11.4 Why the gaps matter

The uncovered branches are precisely the negative security paths: tampered signature, expired credential, empty credential, and insufficient role. A regression that, for example, changed the `catch` block to swallow the error and call `next()` (fail-open) would pass the entire existing suite, because no test presents an invalid but well-formed token. Similarly, inverting the `roles.includes` condition would not be detected, because the only role-guarded endpoint is untested. The positive-path coverage is good and the middleware is simple enough that the risk of undetected regression is bounded, but the security-relevant behaviour is verified by inspection only.

---

## 12. Report Metadata

- **Files analyzed:** `src/middlewares/auth.middleware.ts` (primary), `src/config/env.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/app.ts`, `src/server.ts`, `src/routes/index.ts`, `src/modules/auth/*`, `src/modules/users/user.routes.ts`, `src/modules/users/user.service.ts`, `src/modules/users/user.repository.ts`, `src/modules/customers/customer.routes.ts`, `src/modules/products/product.routes.ts`, `src/modules/orders/order.routes.ts`, `src/modules/orders/order.controller.ts`, `prisma/schema.prisma`, `.env.example`, `package.json`, `tsconfig.json`, `vitest.config.ts`, `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `tests/setup.ts`, plus `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md` for architectural context.
- **Symbols:** `authenticate`, `requireRole`, `AuthUser`, `JwtPayload`, Express `Request` augmentation.
- **No project files were modified during this analysis.**
