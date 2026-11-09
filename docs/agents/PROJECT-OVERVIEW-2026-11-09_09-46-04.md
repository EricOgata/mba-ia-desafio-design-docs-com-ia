# order-management-api - Project Overview

**Generated on**: 2026-11-09 09:46:04

## Summary

The repository is a modular-monolith REST API for an Order Management System, written in TypeScript on Node.js 20+ with Express 4.21, Prisma 5.22, MySQL 8.0, Zod 3.23, JWT (HS256), bcrypt, Pino 9, and Vitest 2 + Supertest 7. It contains roughly 1,700 lines of runtime source across 39 files, organized into five feature modules (`auth`, `users`, `customers`, `products`, `orders`) and a shared kernel for errors, pagination, and logging. Composition is manual and centralized in `src/app.ts`; there is no DI container, no repository interfaces, and no asynchronous/messaging substrate.

The dependency audit found 25 direct npm dependencies (8 production, 17 development), all exact-pinned, plus the `mysql:8.0` container image. Three direct packages carry published advisories (`uuid` High, `zod` High, `vitest` two Critical + one Medium), and six vulnerable transitive packages are present in the production resolution (`proxy-addr` Critical, `body-parser` High, `path-to-regexp` High, `tar` High install-time, `qs` Medium, `brace-expansion` tooling-only). The core stack is one to three majors behind current releases, and ESLint 8 is end-of-life.

Architecturally, the code exhibits consistent per-module layering and a stable shared error taxonomy, but the analysis surfaced critical security and concurrency gaps: public registration accepts a client-supplied `role` including `ADMIN`, most resource routers enforce authentication without role or tenant scoping, stock debit and status transitions run without row locking or optimistic version checks, and the order-number sequence serializes all order creation on one row. The repository is a documentation-challenge fork whose graded deliverable is design documentation for a planned outbound webhook feature; that feature has no implementation in the codebase, and only its integration seams exist.

## Architecture Overview

- **Style**: modular monolith, layered per feature (`routes` -> `controller` -> `service` -> `repository` -> Prisma), synchronous HTTP only.
- **Deployable units**: one Node API process plus one infrastructure service (MySQL 8.0). There is no `Dockerfile`, orchestration manifest, reverse proxy, CI workflow, replica set, or process-manager configuration.
- **Composition**: `src/app.ts` is a composition root with manual dependency injection; it imports 18 internal files and wires every repository, service, and controller. `src/routes/index.ts` mounts 19 endpoints under `/api/v1`, with authentication applied at router level.
- **Cross-cutting concerns**: four middlewares (JWT authentication + `requireRole`, Zod validation, request logging with `X-Request-Id`, centralized error translation) and three shared packages (error taxonomy, pagination envelope, Pino logger). `src/shared/errors/index.ts` is imported by 11 source files.
- **Domain core**: the orders module implements an explicit state machine (6 states, 7 legal edges) and transactional commands covering `orders`, `order_items`, `order_status_history`, `products.stockQuantity`, and `order_number_sequence`. `OrderService` also reads and writes `customers` and `products` directly through `Prisma.TransactionClient`, bypassing sibling repositories.
- **Patterns absent**: no CQRS, event sourcing, DDD aggregates, hexagonal ports/adapters, cache layer, outbox, queue, worker, multi-tenancy, or API gateway.
- **Planned scope (not implemented)**: an outbound webhook subsystem described in `TRANSCRICAO.md` (MySQL outbox, separate polling worker, five retries with backoff, DLQ, HMAC-SHA256 signing, at-least-once delivery). No corresponding schema, endpoint, or process exists.

## Dependencies Health

Critical and high-severity items only:

- **Direct advisories**: `vitest@2.1.4` has two Critical CVEs (CVE-2025-24964 WebSocket hijacking/RCE; CVE-2026-47429 arbitrary file read/execution) and one Medium; exposure is limited to environments where the Vitest API/UI server is listening. `uuid@11.0.3` has a High advisory affecting buffer-writing `v3`/`v5`/`v6` APIs, which this project does not use. `zod@3.23.8` has a High advisory for unbounded validation-issue accumulation, relevant to the unbounded `items` array in `createOrderSchema` under the 1 MB body limit.
- **Transitive advisories in the production tree**: `proxy-addr@2.0.7` Critical (only with `trust proxy` enabled, which the app does not set), `body-parser@1.20.3` High (reachable via `express.json`), `path-to-regexp@0.1.10` High ReDoS (two advisories), `tar@6.2.1` High (install-time via bcrypt/node-pre-gyp), `qs@6.13.0` Medium DoS.
- **End-of-life or legacy**: `eslint@8.57.1` (EOL 2024-10-05), `express@4.21.1` (maintenance mode), `@prisma/client`/`prisma@5.22.0` (two majors behind), `zod@3.23.8`, `typescript@5.6.3`, `vitest@2.1.4`, `@types/node@20.17.6` (Node 20 EOL).
- **Infrastructure**: `mysql:8.0` entered Oracle Sustaining Support on 2026-04-21 and the image tag is not digest-pinned; the Compose file passes `--default-authentication-plugin=mysql_native_password`, removed in MySQL 8.4+.
- **Unused / deprecated**: `pino-http` and `@types/uuid` are declared but unused; `@types/uuid` and `supertest@7.0.0` carry deprecation notices.
- **Licensing**: all npm dependencies are MIT or Apache-2.0; no copyleft propagation. MySQL is GPLv2 but runs as a separate process.

## Components Analyzed

1. **Application Composition Root** (`src/app.ts`) - Assembles the Express app and wires all dependencies manually; contains no domain logic, but pipeline ordering makes malformed JSON and oversized bodies return 500, and the health/404/500 branches are untested.
2. **Server Bootstrap** (`src/server.ts`) - Performs env fail-fast, port binding, and signal-driven shutdown; shutdown does not await `server.close`, lacks `server.on('error')` and re-entrancy guards, and is never imported by tests.
3. **API Router Registry** (`src/routes/index.ts`) - Publishes 19 endpoints with router-level authentication; documents the public-role escalation path, missing role guards on customers/products/orders, 401-before-404 behavior on unknown protected paths, and a duplicated UUID schema.
4. **Environment Configuration** (`src/config/env.ts`) - Zod-validated environment contract with fail-fast exit; notes unvalidated `SHADOW_DATABASE_URL`/`MYSQL_*` variables, a permissive development default, a weak JWT minimum, and an untestable `process.exit` path.
5. **Prisma Client Provider** (`src/config/database.ts`) - Factory plus process singleton for PrismaClient; assessed as high criticality with lifecycle and failure-path risks.
6. **Authentication Middleware** (`src/middlewares/auth.middleware.ts`) - JWT verification and `requireRole` RBAC; lacks an algorithm allowlist, per-request role revalidation, and revocation; `requireRole` branch coverage is near zero.
7. **Validation Middleware** (`src/middlewares/validate.middleware.ts`) - Schema-agnostic `parse` step translating `ZodError` to `400 VALIDATION_ERROR`; mutates body/params asymmetrically versus query and has no dedicated unit tests.
8. **Request Logger Middleware** (`src/middlewares/request-logger.middleware.ts`) - Generates/forwards `X-Request-Id` and emits one access log; body-parser rejections bypass it, client-supplied IDs are unvalidated, aborted responses are unlogged, and `pino-http` is unused.
9. **Centralized Error Middleware** (`src/middlewares/error.middleware.ts`) - Maps `AppError`, Zod, and selected Prisma errors to JSON; only P2002/P2025 are handled, body-parser errors become 500, and `headersSent` delegation is missing.
10. **Error Taxonomy** (`src/shared/errors/`) - `AppError` base plus HTTP-specific subclasses shared across the system; 18 rules and 15 debt items documented.
11. **Pagination Response Helper** (`src/shared/http/response.ts`) - Defines the pagination envelope contract used by list endpoints; 15 rules documented.
12. **Logger** (`src/shared/logger/index.ts`) - Pino factory and process singleton with redaction of credential paths; 15 rules documented, no direct tests.
13. **Auth Module** (`src/modules/auth/`) - Implements register/login/me across 22 rules; registration forwards a client-controlled `role`, enabling self-provisioned `ADMIN`.
14. **Users Module** (`src/modules/users/`) - User persistence and `PublicUser` projection; the admin-only lookup is the sole role-gated endpoint in the system.
15. **Customers Module** (`src/modules/customers/`) - Customer CRUD with pagination and search; no role or tenant scoping, full PII exposure, and an unmapped FK error turns blocked deletion into HTTP 500.
16. **Products Module** (`src/modules/products/`) - Product catalog and inventory CRUD; authenticated-only access permits any actor to mutate stock and price.
17. **Orders Module** (`src/modules/orders/`) - Order creation, listing, deletion, and status transitions across five endpoints; stock is not validated at creation, transitions are not locked, creation is not idempotent, and all endpoints are role-agnostic.
18. **Order Service** (`src/modules/orders/order.service.ts`) - Orchestrates pricing, transactions, stock movement, order numbering, and audit history; 33 rules and 20 ranked risks, including read-check-then-write stock handling.
19. **Order Repository** (`src/modules/orders/order.repository.ts`) - Encapsulates order queries, pagination, and history loading; cannot participate in interactive transactions, lacks a `DATETIME(3)` tiebreaker and a `(status, createdAt)` index, and its delete path is untested.
20. **Order State Machine** (`src/modules/orders/order.status.ts`) - Defines 6 states and 7 legal edges with compile-time exhaustiveness; stock policy is separate from the graph, two exports are dead, and the consumer applies transitions without locking.
21. **Order Status History** (`prisma` model) - Append-only audit journal written by two `OrderService` sites and read via one repository include; records previous/next status, actor, timestamp, and optional reason.
22. **Prisma Data Model and Migrations** (`prisma/schema.prisma`, `prisma/migrations/`) - Seven models and two enums in a single monolithic migration; documents relations, constraints, indexes, and 26 rules.
23. **Seed Script** (`prisma/seed.ts`) - Destructive demo seed producing 26 orders, 41 items, and 78 history rows; uses hardcoded credentials, has no environment guard or run-level transaction, duplicates order-number logic, and fabricates transitions the state machine forbids.
24. **Test Harness and Factories** (`tests/setup.ts`, `tests/helpers/factories.ts`) - Runs 15 integration tests against a real MySQL database with serialized execution and per-test truncation; truncation is destructive and unguarded, factories bypass domain validation, and no coverage provider is configured.
25. **MySQL Container** (`docker-compose.yml`) - Single `mysql:8.0` service with published port, named volume, and healthcheck; image is not digest-pinned and the authentication flag is obsolete for newer MySQL lines.

## Critical Findings

### Security Risks

- **Critical - privilege escalation via registration**: `registerSchema` accepts a client-supplied `role` (including `ADMIN`), and `POST /api/v1/auth/register` is public and unthrottled.
- **High - missing authorization granularity**: customers, products, and orders require only authentication, with no role checks and no customer/tenant scoping; any authenticated actor can read or mutate all records and drive any order through its lifecycle.
- **High - transport and abuse controls absent**: no HTTPS/HSTS/security headers, no TLS termination definition, and no rate limiting, lockout, or backoff on login/register.
- **High - secret and infrastructure exposure**: example/default credentials are committed, the MySQL root password defaults in Compose, and port 3306 is published to the host.
- **Medium - token lifecycle**: 8-hour stateless JWTs with no refresh, revocation, or denylist; roles are not revalidated per request; `jwt.verify` has no algorithm allowlist.
- **Medium - PII and logging**: customer PII is stored and returned in plaintext to any authenticated actor, and request logs capture full `originalUrl` outside redaction coverage.
- **Medium - operational safety**: the test harness truncates every table of whatever database `DATABASE_URL` points to without an environment guard; seed credentials are hardcoded.
- **Dependency exposure**: Critical advisories in `vitest` (dev-only, requires a listening API/UI server), High in `uuid` (vulnerable APIs unused) and `zod` (unbounded issue accumulation reachable through order creation), plus vulnerable production transitives (`body-parser`, `path-to-regexp`, `qs`, `proxy-addr`).

### Technical Debt

- **Concurrency and consistency**: status transitions read without row locks or version checks; stock is validated by read-then-decrement without a conditional guard, allowing double debit/replenish, duplicate history rows, and overselling.
- **Hot-path serialization**: order numbering uses a single `order_number_sequence` row (`id = 1`) with an upsert inside every creation transaction.
- **Business-rule gaps**: orders can be created without stock availability; order creation has no idempotency key; there is no database-level transition enforcement.
- **Layering violations**: `OrderService` bypasses repositories for customers, products, and the sequence; repositories depend on concrete Prisma classes with no interfaces.
- **Composition and coupling**: `src/app.ts` statically imports 18 files and is edited for every new module; the error barrel has 11 dependents and no abstraction boundary.
- **Error handling**: only P2002/P2025 Prisma errors are mapped; FK violations (P2003) and body-parser errors surface as 500; no `headersSent` delegation.
- **Lifecycle and readiness**: shutdown does not await connection draining; `/health` does not probe the database; startup fails hard for any invalid environment variable.
- **Dead or duplicated code**: unused `pino-http`/`@types/uuid`, dead state-machine exports, duplicated registration/user schemas, and inline UUID param schema duplication.
- **Testing**: 15 integration tests across two suites, no unit tests, no coverage provider or thresholds, and untested modules (`users`, `customers`, `products`, health, bootstrap, env, logger).
- **Delivery and operations**: no CI, build gate, container image, API contract (OpenAPI), metrics, traces, or alerting.
- **Documentation**: PRD/RFC/FDD/TRACKER and `docs/adrs/` are placeholders; the planned webhook architecture exists only in `TRANSCRICAO.md`.
- **Dependency currency**: ESLint 8 EOL, Express 4 maintenance, Prisma 5/Zod 3/TypeScript 5/Vitest 2 behind current majors, Node 20 EOL, MySQL 8.0 sustaining support.

### Single Points of Failure

1. **`order_number_sequence` row 1** - every order creation contends on the same row.
2. **Single MySQL instance** - all state, including audit history, resides in one non-replicated volume with no backup/PITR configuration.
3. **Single Node process** - HTTP serving, business logic, and ORM share one event loop and connection pool; a crash or deploy stops all traffic.
4. **`src/shared/errors/index.ts`** - 11 importers; breaking the error contract propagates system-wide.
5. **`src/app.ts` composition root** - the only wiring point for all modules and middleware.
6. **`JWT_SECRET`** - one symmetric secret signs and validates every token.
7. **`src/config/env.ts` fail-fast** - any missing or invalid variable terminates the process at startup, including in test and tooling contexts.

## Reports Index

See [MANIFEST.md](./MANIFEST.md) for the complete registry of all 27 generated reports (1 dependency audit, 1 architectural analysis, 25 component analyses).
