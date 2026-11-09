# Component Deep Analysis Report — Products Module

**Component:** Products Module
**Primary location:** `src/modules/products/`
**Primary files:** `product.routes.ts`, `product.controller.ts`, `product.service.ts`, `product.repository.ts`, `product.schemas.ts`
**Analyst scope:** the Products Module plus directly coupled contracts and consumers: `src/middlewares/`, `src/shared/`, `src/app.ts`, `src/routes/index.ts`, `src/modules/orders/` (table-level coupling to `products`), `src/modules/auth/` (token issuance consumed through the authentication middleware), `prisma/schema.prisma`, `prisma/migrations/`, `prisma/seed.ts`, `tests/`, `vitest.config.ts`, `package.json`
**Excluded paths:** `node_modules`, `.git`, `dist`, `coverage`
**Analysis date:** 2026-11-08
**Report confidence:** High for implemented behavior. Low for intended behavior: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` and `docs/TRACKER.md` are empty templates (3 lines each, heading plus HTML placeholder comment), so no written requirement states what the product catalog is supposed to do.

---

## 1. Executive Summary

The Products Module is a feature module of the Order Management System (OMS) REST API. It owns the product catalog: creation, retrieval, filtered pagination, partial update and deletion of `Product` records, plus the SKU uniqueness rule and the mutable `active` and `stockQuantity` state that other modules read. It is a small module — 5 files, 237 lines — implemented in the project's standard layered shape: `product.routes.ts` (HTTP binding and middleware composition), `product.controller.ts` (transport adaptation), `product.service.ts` (business rules), `product.repository.ts` (Prisma data access) and `product.schemas.ts` (Zod request contracts). Persistence is owned by the module through `ProductRepository`; the `Product` table is the module's only durable state.

The module is mounted at `/api/v1/products` (`src/routes/index.ts:27`) and exposes five REST operations: list with pagination/search/active filter, read by id, create, partial update and delete. Every route requires a valid bearer JWT through `router.use(authenticate)` (`src/modules/products/product.routes.ts:14`). No route uses `requireRole`, so any authenticated `ADMIN` or `OPERATOR` can change prices, stock, SKU and active state, or delete products.

Architecturally the module is a clean, conventional CRUD service. It contains no transactions for its own writes (each write is a single Prisma statement), no domain entities or value objects (`Product` is the Prisma record), no events, no cache, and no audit trail. Its most important structural characteristic is not internal: it is that the Orders Module reads and writes the `products` table directly through `Prisma.TransactionClient`, bypassing `ProductService` and `ProductRepository` entirely (`src/modules/orders/order.service.ts:62-84, 204-243`). The product catalog therefore has two writers — the Products Module and the Orders Module — with no shared concurrency control.

### Key findings

| Severity | Finding | Evidence |
|----------|---------|----------|
| High | No authorization granularity: any authenticated user, including `OPERATOR`, can create, price, re-stock, deactivate or delete products. The module applies `authenticate` only, never `requireRole`. | `src/modules/products/product.routes.ts:14-24`; `src/middlewares/auth.middleware.ts:49-61` (available but unused here) |
| High | Hard delete is blocked only by the database foreign key `order_items.productId` (`ON DELETE RESTRICT`); `P2003` is not translated by the error middleware, so deleting a referenced product returns `500 INTERNAL_SERVER_ERROR` instead of a 4xx. | `prisma/migrations/20260519182739_init/migration.sql:119`; `src/middlewares/error.middleware.ts:37-54`; `src/modules/products/product.repository.ts:59-61` |
| High | `stockQuantity` and `priceCents` are writable through `PATCH` as absolute values, concurrently with the Orders Module's `decrement`/`increment` calls, with no version column, row lock or optimistic check. Lost updates and stock/order divergence are possible. | `src/modules/products/product.service.ts:45-61`; `src/modules/orders/order.service.ts:225-242`; `prisma/schema.prisma:56-72` |
| Medium | SKU uniqueness uses check-then-write. A concurrent create/update loses the race and returns a generic `409 CONFLICT` instead of the module's `409 SKU_ALREADY_USED`, because only Prisma `P2002` reaches the error middleware. | `src/modules/products/product.service.ts:31-34,47-52`; `src/middlewares/error.middleware.ts:37-47` |
| Medium | SKU uniqueness is case- and trailing-space-insensitive in MySQL (`utf8mb4_unicode_ci`) while application code performs no normalization, so the effective uniqueness rule depends on database collation, not on the Zod contract. | `prisma/migrations/20260519182739_init/migration.sql:29,43`; `src/modules/products/product.schemas.ts:6` |
| Medium | `priceCents` and `stockQuantity` have no upper bound in the schema, while the columns are signed 32-bit `INT`. Values above 2,147,483,647 are accepted by Zod and fail at the database as an unmapped 500. | `src/modules/products/product.schemas.ts:9-10`; `prisma/schema.prisma:61-62` |
| Medium | `PATCH` accepts an empty object and performs a no-op write that still bumps `updatedAt`; the update schema is `createProductSchema.partial()`, so creation defaults (`stockQuantity`, `active`) are not reapplied. | `src/modules/products/product.schemas.ts:14`; `src/modules/products/product.service.ts:53-60` |
| Medium | `description` is nullable in the database and can be set to `null` only at creation; the update schema does not accept `null`, so the field cannot be cleared through the API. | `src/modules/products/product.schemas.ts:8,14`; `src/modules/products/product.service.ts:38,56` |
| Low | `ProductRepository.findManyByIds` has no call sites; the Orders Module performs the equivalent query directly. | `src/modules/products/product.repository.ts:47-49`; `src/modules/orders/order.service.ts:62` |
| Low | No `products` test file exists. The module has zero direct endpoint tests; it is exercised only indirectly through the Orders tests via a Prisma factory. | `tests/orders.test.ts`; `tests/helpers/factories.ts:62-74` |

### Component role in the system

- Inbound: mounted at `/api/v1/products` by `src/routes/index.ts:27`, using the router returned from `buildProductRouter` (`src/modules/products/product.routes.ts:12-27`).
- Outbound: `ProductRepository` to Prisma/MySQL; shared error taxonomy; pagination helper; validation and authentication middleware.
- Lateral (implicit): the `products` table is read and written by `OrderService` and projected by `OrderRepository` (`src/modules/orders/order.repository.ts:53`), so product rows are part of the order lifecycle contract even though no code imports the Products Module.
- Operational: 20 demo products are inserted by `prisma/seed.ts:142-163, 219-222`; tests create products directly with `tests/helpers/factories.ts:62-74` and truncate the table in `tests/setup.ts:13`.

### Scope assumptions

- No written product requirements exist. All rules below are reverse-engineered from code, with a confidence indicator per rule.
- No unit tests exist for any file in the module; behavior is inferred from code plus the indirect coverage in `tests/orders.test.ts`.
- No coverage percentage is reported because `vitest.config.ts` defines no coverage provider and `package.json` has no coverage script or `@vitest/coverage-*` dependency. Any percentage would be fabricated.

---

## 2. Data Flow Analysis

### 2.1 List products (`GET /api/v1/products`)

```
1. Express receives the request; express.json (1 MB limit) and requestLogger run first (src/app.ts:59-60)
2. requestLogger assigns or propagates X-Request-Id and records start time (src/middlewares/request-logger.middleware.ts:5-10)
3. /api/v1 router dispatches to buildProductRouter (src/routes/index.ts:27)
4. router.use(authenticate) verifies the bearer JWT and hydrates req.user (src/modules/products/product.routes.ts:14)
   - missing/invalid header or token -> UnauthorizedError -> 401 (src/middlewares/auth.middleware.ts:27-47)
5. validate({ query: listProductsQuerySchema }) parses req.query (src/modules/products/product.routes.ts:16)
   - page and pageSize are coerced to integers and bounded
   - search is trimmed and must be non-empty when present
   - active must be the literal "true" or "false" and is transformed to a boolean
   - Zod failures -> ValidationError -> 400 with path/message details (src/middlewares/validate.middleware.ts:25-35)
   - parsed values are merged onto the cached req.query object (src/middlewares/validate.middleware.ts:17-20)
6. ProductController.list casts req.query to ListProductsQuery (src/modules/products/product.controller.ts:10)
7. ProductService.list computes skip = (page - 1) * pageSize and take = pageSize (src/modules/products/product.service.ts:15-20)
8. ProductRepository.list builds a Prisma where clause (src/modules/products/product.repository.ts:13-23)
   - active filter is added only when defined
   - search adds OR [ name contains, sku contains ]
9. Prisma batch transaction executes findMany (orderBy createdAt DESC, skip, take) and count in one transaction
   (src/modules/products/product.repository.ts:27-35)
10. ProductService wraps items and total with paginated() (src/shared/http/response.ts:22-24)
11. ProductController responds 200 with { data: Product[], pagination: { page, pageSize, total, totalPages } }
    (src/modules/products/product.controller.ts:12)
12. Request logger emits http_request with statusCode and duration on response finish
    (src/middlewares/request-logger.middleware.ts:12-25)
```

### 2.2 Read product by id (`GET /api/v1/products/:id`)

```
1. authenticate -> 401 on missing/invalid token (src/modules/products/product.routes.ts:14)
2. validate({ params: productIdParamSchema }) requires a UUID string (src/modules/products/product.routes.ts:17)
   - invalid UUID -> 400 VALIDATION_ERROR with details [{ path: "id", ... }]
   - req.params is replaced by the parsed object (src/middlewares/validate.middleware.ts:21-23)
3. ProductController.getById passes req.params.id (non-null assertion) to the service
   (src/modules/products/product.controller.ts:20)
4. ProductService.getById -> ProductRepository.findById -> prisma.product.findUnique({ where: { id } })
   (src/modules/products/product.service.ts:24-28; src/modules/products/product.repository.ts:39-41)
5a. Row missing -> NotFoundError("Product") -> 404 { error: { code: "NOT_FOUND", message: "Product not found" } }
5b. Row found -> the raw Prisma Product is returned to the controller
6. Controller responds 200 with the Product JSON, no envelope (src/modules/products/product.controller.ts:21)
```

### 2.3 Create product (`POST /api/v1/products`)

```
1. authenticate -> 401 on missing/invalid token
2. validate({ body: createProductSchema }) replaces req.body with the parsed object
   (src/modules/products/product.routes.ts:18; src/middlewares/validate.middleware.ts:14-16)
   - sku: 2..64 chars; name: 2..200 chars; description: <=2000 optional
   - priceCents: integer >= 0
   - stockQuantity: integer >= 0, default 0
   - active: boolean, default true
   - unknown keys are stripped by Zod object parsing
3. ProductController.create calls ProductService.create with the validated body
   (src/modules/products/product.controller.ts:29)
4. ProductService.create calls ProductRepository.findBySku(input.sku)
   (src/modules/products/product.service.ts:31; src/modules/products/product.repository.ts:43-45)
5a. Existing SKU -> ConflictError("SKU already in use", "SKU_ALREADY_USED") -> 409
5b. No existing SKU -> ProductRepository.create with an explicit field whitelist
    (id, createdAt, updatedAt and any extra field are not forwarded)
    (src/modules/products/product.service.ts:35-42)
6. Prisma INSERT into products; UUID default, createdAt/updatedAt populated by MySQL/Prisma
   (prisma/schema.prisma:56-72)
7. Unique index products_sku_key is the final invariant; a concurrent insert surfaces as Prisma P2002
   (prisma/migrations/20260519182739_init/migration.sql:43)
8. Controller responds 201 with the created Product (src/modules/products/product.controller.ts:30)
```

### 2.4 Update product (`PATCH /api/v1/products/:id`)

```
1. authenticate -> 401
2. validate({ params: productIdParamSchema, body: updateProductSchema })
   (src/modules/products/product.routes.ts:19-23)
   - updateProductSchema is createProductSchema.partial(); every field is optional
   - no null values accepted; no minimum of one field enforced
3. ProductController.update calls ProductService.update(id, body) (src/modules/products/product.controller.ts:38)
4. ProductService.update first calls getById(id) -> 404 if the product does not exist
   (src/modules/products/product.service.ts:46)
5. If input.sku is truthy, findBySku(input.sku); if a different row owns it -> 409 SKU_ALREADY_USED
   (src/modules/products/product.service.ts:47-52)
6. A new data object is assembled with only the defined fields, using !== undefined checks
   (src/modules/products/product.service.ts:53-60)
7. Prisma UPDATE by id; updatedAt is regenerated by Prisma (@updatedAt)
8. Controller responds 200 with the updated Product (src/modules/products/product.controller.ts:39)
```

### 2.5 Delete product (`DELETE /api/v1/products/:id`)

```
1. authenticate -> 401
2. validate({ params: productIdParamSchema }) -> 400 for a non-UUID id (src/modules/products/product.routes.ts:24)
3. ProductController.delete calls ProductService.delete(id) (src/modules/products/product.controller.ts:47)
4. ProductService.delete first calls getById(id) -> 404 NOT_FOUND if absent
   (src/modules/products/product.service.ts:64)
5. ProductRepository.delete -> prisma.product.delete({ where: { id } })
   (src/modules/products/product.repository.ts:59-61)
6a. Product not referenced by order_items -> row removed; controller responds 204 with no body
    (src/modules/products/product.controller.ts:48)
6b. Product referenced by order_items -> MySQL FK RESTRICT raises Prisma P2003
    -> not handled by error.middleware -> logged and returned as 500 INTERNAL_SERVER_ERROR
    (prisma/migrations/20260519182739_init/migration.sql:119; src/middlewares/error.middleware.ts:56-64)
6c. Row deleted concurrently after step 4 -> Prisma P2025 -> 404 NOT_FOUND "Resource not found"
    (src/middlewares/error.middleware.ts:48-53)
```

### 2.6 Cross-module product flows (not owned by the module but relevant to its data)

```
Order creation (src/modules/orders/order.service.ts:50-124)
  -> tx.product.findMany({ where: { id: { in: productIds } } })
  -> 404 if any product is missing
  -> 422 INACTIVE_PRODUCT if any product has active = false
  -> unitPriceCents copied from product.priceCents into order_items (price snapshot)
  -> stockQuantity is NOT checked at creation

Order PENDING -> PAID (src/modules/orders/order.service.ts:151-153, 204-231)
  -> reads products, compares stockQuantity with requested quantity
  -> 422 INSUFFICIENT_STOCK with skus/requested/available details if short
  -> prisma.product.update({ stockQuantity: { decrement: quantity } })

Order PAID|PROCESSING -> CANCELLED (src/modules/orders/order.service.ts:154-156, 233-243)
  -> prisma.product.update({ stockQuantity: { increment: quantity } })

Order read projections (src/modules/orders/order.repository.ts:53)
  -> items include product { id, sku, name } resolved at read time, not snapshotted
```

### 2.7 Trust boundaries crossed

| Boundary | Crossing point | Validation applied |
|----------|----------------|--------------------|
| Network -> application | Express JSON parser and router | 1 MB body limit (`src/app.ts:59`); Zod body/query/params schemas |
| Anonymous -> authenticated actor | `authenticate` middleware | JWT signature and expiry only; no role or tenant check |
| Application -> database | Prisma Client | Parameterized SQL; unique index on `sku`; FK RESTRICT from `order_items` |
| Products Module -> Orders Module | Shared `products` table | No contract, no lock, no version; only application-level stock checks in Orders |
| Application -> caller | JSON responses | Raw Prisma record serialization; no field-level redaction (no sensitive fields exist) |

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location | Confidence |
|---------|-----------|------------------|----------|------------|
| BR-01 | Security / Access | Every product endpoint requires a valid Bearer JWT | `src/modules/products/product.routes.ts:14` | High |
| BR-02 | Security / Authorization | No role restriction: any authenticated user can read and mutate the catalog | `src/modules/products/product.routes.ts:14-24` | High |
| BR-03 | Validation | Path `id` must be a UUID | `src/modules/products/product.schemas.ts:3` | High |
| BR-04 | Validation / Identity | SKU must be a 2..64 character string; uniqueness is case-insensitive at the database | `src/modules/products/product.schemas.ts:6`; `prisma/schema.prisma:58` | High |
| BR-05 | Validation | Name must be a 2..200 character string | `src/modules/products/product.schemas.ts:7` | High |
| BR-06 | Validation / Data | Description is optional, at most 2000 characters, and cannot be reset to null after creation | `src/modules/products/product.schemas.ts:8,14`; `src/modules/products/product.service.ts:38,56` | High |
| BR-07 | Validation / Money | `priceCents` must be a non-negative integer with no configured upper bound | `src/modules/products/product.schemas.ts:9` | High |
| BR-08 | Validation / Inventory | `stockQuantity` must be a non-negative integer, defaulting to 0 on create | `src/modules/products/product.schemas.ts:10` | High |
| BR-09 | Business Logic | `active` defaults to true on create and is the catalog's availability flag | `src/modules/products/product.schemas.ts:11`; `prisma/schema.prisma:63` | High |
| BR-10 | Validation / Hardening | Unknown input fields are stripped at the body and params boundaries | `src/middlewares/validate.middleware.ts:14-23` | High |
| BR-11 | Business Logic | Update is a partial write; defaults are not reapplied and an empty patch is allowed | `src/modules/products/product.schemas.ts:14`; `src/modules/products/product.service.ts:53-60` | High |
| BR-12 | Business Logic | List pagination defaults to page 1 and pageSize 20, with pageSize capped at 100 | `src/modules/products/product.schemas.ts:17-18` | High |
| BR-13 | Business Logic | `search` filters by name or SKU substring after trimming, and must be non-empty | `src/modules/products/product.schemas.ts:19`; `src/modules/products/product.repository.ts:16-21` | High |
| BR-14 | Business Logic | `active` query filter accepts only the lowercase literals "true" and "false" | `src/modules/products/product.schemas.ts:20-23` | High |
| BR-15 | Business Logic | List ordering is fixed to `createdAt DESC` with no client-selected sort | `src/modules/products/product.repository.ts:30` | High |
| BR-16 | Contract | List responses use the shared `{ data, pagination }` envelope; single resources do not | `src/shared/http/response.ts:8-24`; `src/modules/products/product.controller.ts:12,21` | High |
| BR-17 | Business Logic | SKU uniqueness is pre-checked in the service before create | `src/modules/products/product.service.ts:31-34` | High |
| BR-18 | Business Logic / Integrity | The database unique index is the final SKU invariant; the race path maps to generic `CONFLICT` | `prisma/schema.prisma:58`; `src/middlewares/error.middleware.ts:37-47` | High |
| BR-19 | Business Logic | Read, update and delete require the product to exist (404 otherwise) | `src/modules/products/product.service.ts:24-28,46,64` | High |
| BR-20 | Business Logic | Update writes only defined fields and excludes the row itself from the SKU conflict check | `src/modules/products/product.service.ts:47-60` | High |
| BR-21 | Business Logic | SKU is mutable after creation and is not snapshotted by order history | `src/modules/products/product.service.ts:54`; `src/modules/orders/order.repository.ts:53` | High |
| BR-22 | Business Logic / Integrity | Delete is physical and is implicitly limited by the `order_items` foreign key | `src/modules/products/product.service.ts:63-66`; `prisma/migrations/20260519182739_init/migration.sql:119` | High |
| BR-23 | Cross-module | The module owns absolute price and stock values while Orders mutates stock directly; price is snapshotted into orders | `src/modules/products/product.service.ts:53-60`; `src/modules/orders/order.service.ts:75-84,225-242` | High |
| BR-24 | Cross-module | Inactive products are rejected at order creation but are not re-checked at payment or cancellation | `src/modules/orders/order.service.ts:66-73`; `src/modules/orders/order.service.ts:126-179` | High |
| BR-25 | Audit / Governance | Product price, stock and active changes leave no dedicated audit trail | `prisma/schema.prisma:56-72`; `src/middlewares/request-logger.middleware.ts:12-25` | High |
| BR-26 | Contract | Responses serialize the raw Prisma `Product` record with all columns and no DTO projection | `src/modules/products/product.controller.ts:12,21,30,39` | High |
| BR-27 | Error Contract | Failures use the centralized taxonomy with `VALIDATION_ERROR`, `UNAUTHORIZED`, `NOT_FOUND`, `SKU_ALREADY_USED`, `CONFLICT`, `INTERNAL_SERVER_ERROR` | `src/middlewares/error.middleware.ts:14-65`; `src/shared/errors/http-errors.ts:27-37` | High |
| BR-28 | Routing | Unsupported methods on `/products` return the global 404, not 405; authentication still runs first | `src/app.ts:69-71`; `src/modules/products/product.routes.ts:14` | High |

### Detailed breakdown of the business rules

---

### Business Rule BR-01: Authentication requirement for every product route

**Overview**:
All five product endpoints require a valid JWT bearer token. There is no public product read path and no health-style exception inside the router.

**Detailed description**:
`buildProductRouter` calls `router.use(authenticate)` before declaring any route (`src/modules/products/product.routes.ts:14`). Because it is a router-level `use`, the middleware executes for every request whose path enters the router, including requests to unmatched subpaths and unsupported methods. `authenticate` rejects a missing or non-Bearer `Authorization` header with `UnauthorizedError('Missing or invalid Authorization header')`, rejects an empty bearer value with `UnauthorizedError('Missing bearer token')`, and rejects any `jwt.verify` failure with `UnauthorizedError('Invalid or expired token')` (`src/middlewares/auth.middleware.ts:27-47`). All three become HTTP 401 with code `UNAUTHORIZED` through the centralized error middleware.

On success the middleware assigns `req.user = { id: payload.sub, email: payload.email, role: payload.role }` (`src/middlewares/auth.middleware.ts:42`). The Products Module never reads `req.user`: the controller does not pass the actor id to the service, so product mutations are authenticated but not attributable in the domain layer. Authentication is stateless; there is no session lookup or per-request database check, so a token remains usable until expiry even if the user was deleted or their role changed.

The effective security boundary is therefore "any holder of a valid token issued by this system". Since `POST /api/v1/auth/register` is public and accepts a client-supplied role (`src/modules/auth/auth.schemas.ts:7`), the practical population of token holders includes self-registered users. The Products Module inherits that trust model without adding controls of its own.

**Rule workflow**:
```
Request to any /api/v1/products path
  -> Authorization header present and scheme Bearer?
       no  -> 401 UNAUTHORIZED
  -> token non-empty?
       no  -> 401 UNAUTHORIZED "Missing bearer token"
  -> jwt.verify(token, JWT_SECRET)
       throws -> 401 UNAUTHORIZED "Invalid or expired token"
       ok     -> req.user = { id, email, role } -> route validation and controller
```

---

### Business Rule BR-02: No role-based authorization on the catalog

**Overview**:
The module authenticates but does not authorize. Any `ADMIN` or `OPERATOR` token can list, read, create, update and delete products, including changing prices and stock.

**Detailed description**:
`requireRole` exists in the shared authentication middleware and is used exactly once in the system, on the admin-only user lookup route (`src/middlewares/auth.middleware.ts:49-61`; `src/modules/users/user.routes.ts:14-15`). No product route imports or applies it. The product router exposes `POST`, `PATCH` and `DELETE` to every authenticated actor (`src/modules/products/product.routes.ts:18-24`). There is also no tenant, organization or ownership dimension in the `Product` model (`prisma/schema.prisma:56-72`), so no narrower scoping is possible with the current schema.

This is materially different from a read-only catalog policy. A compromised or self-registered `OPERATOR` token can change `priceCents` (directly affecting the price snapshot of every subsequently created order), change `stockQuantity` (bypassing the order stock machine), rename SKUs (affecting order projections that resolve SKU at read time) or deactivate products (blocking new orders). The module has no approval workflow, no two-person rule and no field-level restriction.

The architectural analysis classified this as a High authorization risk (`docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md`, section 6.1). From the module's perspective the rule is simply absent, not misconfigured: there is no dead role check, no commented guard and no configuration flag. Establishing a policy would be a new behavior, not a correction of existing code.

**Rule workflow**:
```
authenticate -> req.user.role in { ADMIN, OPERATOR }
  -> no requireRole call in product.routes.ts
  -> any role proceeds to controller
  -> GET/POST/PATCH/DELETE all permitted
```

---

### Business Rule BR-03: Product identifier must be a UUID

**Overview**:
Every path parameter named `id` is validated against `z.string().uuid()` before the controller runs.

**Detailed description**:
`productIdParamSchema` is defined as `z.object({ id: z.string().uuid() })` (`src/modules/products/product.schemas.ts:3`). It is applied to `GET /:id`, `PATCH /:id` and `DELETE /:id` (`src/modules/products/product.routes.ts:17,19-23,24`). `validate` parses `req.params` and replaces it with the parsed object (`src/middlewares/validate.middleware.ts:21-23`), so the controller receives a validated string. A non-UUID value produces `ValidationError('Validation failed', [{ path: 'id', message: ... }])` and HTTP 400 with `VALIDATION_ERROR`.

The check is syntactic, not existential. A well-formed UUID that does not exist passes validation and then produces 404 through `NotFoundError('Product')`. Zod's default UUID pattern accepts versions 1 through 5 and the nil UUID, so a client can reach the database with identifiers that Prisma's `uuid()` generator would never produce; the result is still a clean 404. The model uses `CHAR(36)` (`prisma/schema.prisma:57`), and Prisma compares the string exactly.

The controller relies on a non-null assertion (`req.params.id!`) in all three handlers (`src/modules/products/product.controller.ts:20,38,47`). This assertion is safe only because the validation middleware runs first; it is not enforced by the type system. There is no length, format or normalization logic in the service or repository.

**Rule workflow**:
```
:id value
  -> z.string().uuid()
       fail -> 400 VALIDATION_ERROR, details[{ path: "id" }]
       pass -> replace req.params.id -> controller -> service -> Prisma findUnique/update/delete
  -> valid UUID but no row -> 404 NOT_FOUND
```

---

### Business Rule BR-04: SKU length and database-defined uniqueness semantics

**Overview**:
A SKU must be a string of 2 to 64 characters. Uniqueness is enforced by a unique database index and is effectively case-insensitive and trailing-space-insensitive because of the MySQL collation.

**Detailed description**:
The schema declares `sku: z.string().min(2).max(64)` (`src/modules/products/product.schemas.ts:6`). There is no `.trim()`, no `.toLowerCase()`, no character-class restriction and no normalization before persistence. A value consisting of two spaces therefore satisfies the length rule and can be stored. The column is `VARCHAR(64)` with a unique index `products_sku_key` (`prisma/schema.prisma:58`; `prisma/migrations/20260519182739_init/migration.sql:34,43`). The table uses `utf8mb4_unicode_ci` (`prisma/migrations/20260519182739_init/migration.sql:47`), a case-insensitive, accent-insensitive, PAD SPACE collation.

Two consequences follow. First, `findBySku('abc')` can match a stored `ABC`, and the unique index rejects `ABC` when `abc` exists, because collation rather than application code defines equality. Second, because `utf8mb4_unicode_ci` is a PAD SPACE collation, trailing spaces are ignored in comparisons, so `SKU-1` and `SKU-1 ` collide at the index even though both pass different string lengths. The component never documents or tests these semantics; they are inherited from the migration.

SKU uniqueness applies to inactive products as well, because there is no partial or conditional unique index. Deactivating a product does not release its SKU. There is also no SKU format convention (prefix, separator, uppercase) enforced anywhere; the seed uses patterns such as `NB-DELL-001` (`prisma/seed.ts:143-162`) but they are not validated.

**Rule workflow**:
```
Create/update payload sku
  -> Zod: string, length 2..64 (untrimmed, unnormalized)
       fail -> 400 VALIDATION_ERROR
       pass -> service findBySku (collation-based equality)
  -> MySQL unique index products_sku_key (utf8mb4_unicode_ci)
       collision -> P2002 -> 409 CONFLICT
```

---

### Business Rule BR-05: Product name length and storage

**Overview**:
A product name must be a string of 2 to 200 characters and is stored untrimmed in a `VARCHAR(200)` column with a secondary index.

**Detailed description**:
`name: z.string().min(2).max(200)` (`src/modules/products/product.schemas.ts:7`) mirrors the column definition `name String @db.VarChar(200)` (`prisma/schema.prisma:59`). The schema performs no trimming and no whitespace collapsing, so leading and trailing spaces are preserved in responses and in search comparisons. Empty strings and one-character names are rejected with a 400 `VALIDATION_ERROR` whose detail path is `name`.

The column has an index `products_name_idx` (`prisma/migrations/20260519182739_init/migration.sql:45`), but the list search uses `contains` (`src/modules/products/product.repository.ts:17-20`), which Prisma translates to a leading-wildcard `LIKE '%term%'` pattern in MySQL. A leading wildcard prevents a normal B-tree index range scan, so the `name` index does not make the search cheap. There is no full-text index in the schema.

Names are not unique and no canonical form is enforced. Two products can share the same name with different SKUs, and the seed contains both `Monitor LG UltraWide 29"` and `Monitor Dell 27" 4K` in the same category-like naming style without any category field to disambiguate (`prisma/seed.ts:145-146`).

**Rule workflow**:
```
name input
  -> Zod string length 2..200 (no trim)
       fail -> 400 VALIDATION_ERROR
       pass -> stored verbatim in VARCHAR(200)
  -> used by list search via LIKE '%name%' (index not effective for substring)
```

---

### Business Rule BR-06: Description is optional and cannot be cleared

**Overview**:
`description` is optional on create, capped at 2000 characters, stored in a nullable `TEXT` column, and cannot be reset to `null` through the update endpoint.

**Detailed description**:
Creation declares `description: z.string().max(2000).optional()` (`src/modules/products/product.schemas.ts:8`) and the service maps an omitted value to `null` with `input.description ?? null` (`src/modules/products/product.service.ts:38`). The column is `description String? @db.Text` (`prisma/schema.prisma:60`). An empty string is valid (length 0 satisfies `max(2000)`) and is stored as `""`, which is indistinguishable from "no description" for many clients but is a different database value from `null`.

Update uses `createProductSchema.partial()` (`src/modules/products/product.schemas.ts:14`), which makes the field optional but does not make it nullable. Sending `"description": null` fails Zod's string check and returns 400. Sending a string replaces the value; omitting the field leaves it untouched (`src/modules/products/product.service.ts:56`). There is therefore no API path that returns the column to `null` after creation.

The 2000-character cap is a Zod-only rule; the database column is `TEXT` and can hold far more. Any code path that writes products without the HTTP schema (seed, tests, direct ORM use) can exceed the contract. Responses return the value verbatim, with no HTML sanitization or Markdown handling; the API treats it as opaque text.

**Rule workflow**:
```
Create: description undefined -> stored NULL
        description "" or "text" -> stored as given
Update: description omitted -> unchanged
        description string  -> replaced
        description null    -> 400 VALIDATION_ERROR
        >2000 chars        -> 400 VALIDATION_ERROR
```

---

### Business Rule BR-07: Price is a non-negative integer of cents with no upper bound

**Overview**:
`priceCents` is required on create, must be a JSON number that is an integer and non-negative, and is persisted in a signed 32-bit `INT` column.

**Detailed description**:
The schema is `priceCents: z.number().int().nonnegative()` (`src/modules/products/product.schemas.ts:9`). Because there is no `z.coerce`, string values such as `"1000"` are rejected rather than converted. Floats, `NaN`, `Infinity`, negative values and `null` all fail with 400. Zero is explicitly allowed, so the catalog can contain free products. The column is `priceCents Int` (`prisma/schema.prisma:61`), which MySQL creates as a signed `INT` (`prisma/migrations/20260519182739_init/migration.sql:37`) with a maximum of 2,147,483,647; the schema sets no matching maximum.

A value between 2,147,483,648 and `Number.MAX_SAFE_INTEGER` passes Zod and fails at the database. The error middleware recognizes only `P2002` and `P2025` (`src/middlewares/error.middleware.ts:37-54`), so an out-of-range write becomes a logged 500 rather than a 400. The same absence of an upper bound applies to multiplication in order pricing: `OrderItem.totalCents` is `unitPriceCents * quantity` (`src/modules/orders/order.service.ts:82`) using JavaScript numbers and then stored in `INT` columns, so large prices combined with large quantities can also overflow the order tables.

The unit is implicitly cents and the currency is implicitly single and unspecified. Nothing in the schema, module or configuration records a currency code, rounding policy or tax treatment. Order items copy `priceCents` at creation time (`src/modules/orders/order.service.ts:75-84`), so a later product price change never retroactively changes an order; that snapshot rule is implemented in the Orders Module, not here.

**Rule workflow**:
```
priceCents input
  -> must be JSON number, integer, >= 0
       fail -> 400 VALIDATION_ERROR
       pass (<= 2^31-1) -> stored in INT
       pass (> 2^31-1)  -> database range error -> unmapped 500
```

---

### Business Rule BR-08: Stock quantity is a non-negative integer defaulting to zero

**Overview**:
`stockQuantity` must be a non-negative integer, defaults to 0 when omitted on create, and is an absolute field that both the Products Module and the Orders Module write.

**Detailed description**:
Creation declares `stockQuantity: z.number().int().nonnegative().default(0)` (`src/modules/products/product.schemas.ts:10`), matching the column default `stockQuantity Int @default(0)` (`prisma/schema.prisma:62`). On create, an omitted field becomes `0` through Zod and is then passed explicitly by the service (`src/modules/products/product.service.ts:40`). On update, `.partial()` wraps the field as optional and the Zod optional branch returns `undefined` before the inner default runs, so an omitted `stockQuantity` is not reset to 0; the service omits it from the update (`src/modules/products/product.service.ts:58`). This difference between create and update is an emergent property of schema composition, not an explicitly written rule.

The schema prevents negative values through the HTTP API, but the database column is a signed `INT` with no `CHECK` constraint or unsigned attribute. The Orders Module decrements and increments the same column directly (`src/modules/orders/order.service.ts:225-242`) and relies on an application-level comparison before decrementing (`src/modules/orders/order.service.ts:212-224`). Any direct SQL, migration script, seed or future consumer can therefore write a negative value.

`PATCH /products/:id` sets an absolute quantity. It does not accept a delta, does not reference an order or reason, and does not coordinate with in-flight order payments. This makes the field a shared mutable counter with two writers and no versioning, which is analyzed further in BR-23.

**Rule workflow**:
```
Create: stockQuantity omitted -> 0; provided -> int >= 0; negative/float/string -> 400
Update: stockQuantity omitted -> unchanged; provided -> int >= 0 replaces absolute value
Orders: PENDING->PAID decrements; PAID|PROCESSING->CANCELLED increments (direct Prisma update)
```

---

### Business Rule BR-09: Active flag defaults to true and represents catalog availability

**Overview**:
`active` is a boolean that defaults to `true` on creation, can be changed only through `PATCH`, and is used by the Orders Module to reject inactive products at order creation.

**Detailed description**:
Creation declares `active: z.boolean().default(true)` (`src/modules/products/product.schemas.ts:11`), matching `active Boolean @default(true)` (`prisma/schema.prisma:63`). The service forwards the resolved boolean (`src/modules/products/product.service.ts:41`), so an omitted field is stored as `true`. The column is indexed (`products_active_idx`, `prisma/migrations/20260519182739_init/migration.sql:44`) to support the list filter. Update uses the partial schema, so `active` is optional and, for the same Zod reason as `stockQuantity`, is not redefaulted.

The flag is a soft-state mechanism, not a soft delete. A product with `active = false` remains physically present, is returned by default list queries unless `active=false` is supplied, and keeps its order references. There is no dedicated activation endpoint, no activation timestamp, no actor or reason field, and no rule preventing deactivation of a product that appears in pending orders. Setting `active` back to `true` restores the product to the catalog immediately.

The functional consumer of the flag is `OrderService.create`, which queries products, filters `!p.active`, and throws `UnprocessableEntityError('One or more products are inactive', 'INACTIVE_PRODUCT', { skus })` (`src/modules/orders/order.service.ts:66-73`). Status transitions do not re-check the flag, so a pending order containing a product that was deactivated after order creation can still be paid if stock is available. The Products Module itself never reads or enforces the flag; it only stores it.

**Rule workflow**:
```
Create active omitted -> true; active false -> inactive product
Update active false -> product hidden from active-filtered lists; blocked at order creation
Order create -> active=false -> 422 INACTIVE_PRODUCT with skus
Order status change -> no active re-check
```

---

### Business Rule BR-10: Unknown fields are stripped at the request boundary

**Overview**:
Zod object parsing strips unknown keys from `req.body` and `req.params`, preventing clients from injecting identifiers, timestamps or other columns that are not part of the contract.

**Detailed description**:
Both `createProductSchema` and `updateProductSchema` are `ZodObject` instances with default `strip` behavior, and `validate` assigns the parse result back: `req.body = schemas.body.parse(req.body)` (`src/middlewares/validate.middleware.ts:14-16`). A payload containing `id`, `createdAt`, `updatedAt`, `stockQuantity` in unexpected places, `__proto__`, or arbitrary extra fields therefore reaches the controller without those keys. The service also builds an explicit field whitelist on create (`src/modules/products/product.service.ts:35-42`), providing a second line of defense against mass assignment.

Query handling differs: `validate` merges parsed values into the existing object with `Object.assign(req.query, parsedQuery)` (`src/middlewares/validate.middleware.ts:17-20`) rather than replacing it. Unknown query keys remain on `req.query`, are ignored by the service, and appear in request logs because the logger records `req.originalUrl` (`src/middlewares/request-logger.middleware.ts:18`). This is not a mass-assignment risk because the repository builds the `where` clause from named fields only, but it means the query object is not a pristine contract object.

Params are replaced (`src/middlewares/validate.middleware.ts:21-23`), so an extra path-derived property cannot be injected. There is no global validation pipe, so the stripping guarantee holds only for routes that call `validate`; the product routes all do.

**Rule workflow**:
```
Body: parse -> known keys only -> req.body replaced -> service whitelist -> Prisma
Params: parse -> { id } only -> req.params replaced
Query: parse -> Object.assign onto existing req.query -> unknown keys remain but are ignored
```

---

### Business Rule BR-11: Partial update semantics

**Overview**:
`PATCH /products/:id` accepts any subset of the create fields, including the empty subset; creation defaults are not reapplied, and every accepted call bumps `updatedAt`.

**Detailed description**:
`updateProductSchema` is exactly `createProductSchema.partial()` (`src/modules/products/product.schemas.ts:14`). `.partial()` wraps each field in `ZodOptional`, so absent fields parse as `undefined`. For fields that carried `.default()` in the create schema, the optional wrapper returns `undefined` before the default is evaluated, which is why an update omitting `stockQuantity` or `active` does not reset them. The service then assembles the Prisma payload with individual `!== undefined` checks (`src/modules/products/product.service.ts:53-60`), so only supplied fields are written.

No refinement requires at least one field. `PATCH` with `{}` passes validation, passes the existence check, and executes `prisma.product.update({ where: { id }, data: {} })`. Prisma permits an empty update object and still applies `@updatedAt` (`prisma/schema.prisma:65`), so the call returns 200 with a new `updatedAt` and no business change. This makes the endpoint non-idempotent in its observable metadata even when the client sends nothing.

The same constraints as create apply to supplied fields: `sku` length, `name` length, `description` length, non-negative integers for price and stock, boolean for active. No field is nullable, no field can be explicitly cleared, and there is no distinction between "not provided" and "provided with the current value". There is also no ETag, `If-Match`, `updatedAt` precondition or version column, so two concurrent partial updates both succeed and the last writer wins for each field.

**Rule workflow**:
```
PATCH body subset
  -> Zod partial: absent -> undefined, supplied -> validated
  -> existence check (404 if absent)
  -> SKU conflict check when sku supplied
  -> data object with only defined fields (may be {})
  -> Prisma update -> updatedAt changes
  -> 200 full Product
```

---

### Business Rule BR-12: Pagination defaults and bounds

**Overview**:
List pagination uses `page` (default 1, minimum 1, no maximum) and `pageSize` (default 20, minimum 1, maximum 100), both coerced from strings and translated into `skip`/`take`.

**Detailed description**:
`listProductsQuerySchema` declares `page: z.coerce.number().int().min(1).default(1)` and `pageSize: z.coerce.number().int().min(1).max(100).default(20)` (`src/modules/products/product.schemas.ts:17-18`). Coercion means `"2"` becomes `2`, while `"2.5"` fails `int()`, `"abc"` becomes `NaN` and fails, and an empty string becomes `0` and fails the minimum. Every failure is a 400 with a field-path detail. `ProductService.list` computes `skip = (query.page - 1) * query.pageSize` and `take = query.pageSize` (`src/modules/products/product.service.ts:18-19`).

The page itself has no upper bound. A request with `page=100000000` produces a very large `OFFSET`, which MySQL must scan and discard; the response is an empty `data` array with `page` echoed and `total`/`totalPages` calculated normally. There is no clamp of `page` to `totalPages`, no cursor pagination and no maximum offset. The `pageSize` cap of 100 is the only protection against oversized result sets.

Because pagination is offset-based, concurrent inserts or deletes shift the window. Combined with the fixed `createdAt DESC` ordering and the absence of a tie-breaker (BR-15), a client paging through the catalog can see duplicates or miss rows. The module does not expose `totalPages` as an input, only as output.

**Rule workflow**:
```
GET /products?page=&pageSize=
  -> coerce to number, require integer
  -> page >= 1 (default 1); 1 <= pageSize <= 100 (default 20)
       fail -> 400 VALIDATION_ERROR
  -> skip = (page - 1) * pageSize; take = pageSize
  -> findMany(skip, take) + count in one batch transaction
```

---

### Business Rule BR-13: Search matches name or SKU substring after trimming

**Overview**:
When `search` is present it is trimmed, must be non-empty, and filters products whose name or SKU contains the term; matching semantics are delegated to MySQL collation and `LIKE`.

**Detailed description**:
The schema is `search: z.string().trim().min(1).optional()` (`src/modules/products/product.schemas.ts:19`). A value of `"   "` is trimmed to `""` and rejected with 400 rather than treated as "no filter". There is no maximum length, so an arbitrarily long term is accepted and sent to the database. The repository adds `where.OR = [{ name: { contains: search } }, { sku: { contains: search } }]` (`src/modules/products/product.repository.ts:16-21`), which Prisma executes as `LIKE '%term%'` against both columns.

Case and accent sensitivity are not specified in code. They follow the column collation, `utf8mb4_unicode_ci`, which is case-insensitive and accent-insensitive, so `notebook` matches `Notebook`. The component does not set Prisma's `mode: 'insensitive'` (a PostgreSQL-only option); the behavior is a MySQL inheritance. Escaping of `%` and `_` is not implemented in the component; pattern metacharacters inside the term are handled, or not, by Prisma's parameterization and MySQL's pattern semantics, which the code does not document or test.

Performance is a known consequence of the pattern: leading-wildcard `LIKE` cannot use `products_name_idx` or the unique SKU index for a range scan, so a search scans the filtered table. The OR across two columns further prevents index-only access. There is no full-text index, no trigram index, no search-result cap and no relevance ranking; results keep the default `createdAt DESC` order.

**Rule workflow**:
```
search value
  -> trim
  -> length >= 1 (else 400)
  -> no length cap
  -> where OR (name LIKE %term%, sku LIKE %term%)
  -> collation-based case/accent behavior
  -> combined with active filter and pagination
```

---

### Business Rule BR-14: Active filter accepts only lowercase literal strings

**Overview**:
The `active` query parameter is a string union of `"true"` and `"false"` that is transformed to a boolean; any other spelling is rejected.

**Detailed description**:
`active` is declared as `z.union([z.literal('true'), z.literal('false')]).transform((v) => v === 'true').optional()` (`src/modules/products/product.schemas.ts:20-23`). Because query values arrive as strings, this is an explicit string comparison, not a boolean coercion. `?active=true` and `?active=false` work; `TRUE`, `True`, `1`, `0`, `yes`, empty string and repeated parameters (`?active=true&active=false`, parsed as an array by Express's default query parser) all fail with 400 `VALIDATION_ERROR` at path `active`.

The transformed boolean is stored back on `req.query` by `Object.assign` (`src/middlewares/validate.middleware.ts:19`) and then read by the controller through a cast. `ProductRepository.buildWhere` applies it only when `!== undefined` (`src/modules/products/product.repository.ts:15`), so an omitted parameter means "all products" rather than "active only". There is no default filter, which is why inactive products appear in the default list.

The contract is asymmetric with typical REST boolean parsing and is not documented in any API specification. The same pattern is not necessarily used by other modules, so a client cannot generalize. The strictness is total: there is no fallback, no case folding and no lenient parsing.

**Rule workflow**:
```
active query value
  -> exactly "true" -> boolean true
  -> exactly "false" -> boolean false
  -> absent -> undefined -> no where condition
  -> anything else -> 400 VALIDATION_ERROR at path "active"
```

---

### Business Rule BR-15: Fixed ordering by creation time descending

**Overview**:
Lists are always ordered by `createdAt DESC`; clients cannot choose a sort field or direction, and there is no deterministic tie-breaker.

**Detailed description**:
`ProductRepository.list` hardcodes `orderBy: { createdAt: 'desc' }` inside `findMany` (`src/modules/products/product.repository.ts:30`). The list schema has no `sort`, `order`, `orderBy` or `sortBy` field (`src/modules/products/product.schemas.ts:16-24`), and the service forwards only search/active/skip/take (`src/modules/products/product.service.ts:15-20`). Newest products therefore appear first, which matches the seed's narrative of a catalog loaded once and browsed by recency.

`createdAt` is a `DATETIME(3)` column with millisecond precision (`prisma/schema.prisma:64`; `prisma/migrations/20260519182739_init/migration.sql:40`). Rows created in the same millisecond have equal sort keys, and MySQL does not guarantee a stable order among ties. The seed creates products with `Promise.all` (`prisma/seed.ts:219-222`), making ties plausible in demo data. Under offset pagination, an unstable order can cause the same row to appear on two pages or be skipped entirely.

There is no index on `createdAt` for the `products` table; the schema indexes only `active` and `name` plus the unique SKU (`prisma/schema.prisma:69-70`). Sorting the whole table therefore requires a filesort in MySQL unless the filtered set is small. For large catalogs, list and search queries both pay for sorting without an index.

**Rule workflow**:
```
GET /products (filtered)
  -> where from search/active
  -> ORDER BY createdAt DESC (no tie-breaker)
  -> SKIP/OFFSET then LIMIT pageSize
  -> ties and concurrent writes can reorder/shift the window
```

---

### Business Rule BR-16: List envelope versus raw single-resource responses

**Overview**:
List responses use the shared `{ data, pagination }` envelope; single-resource responses return the bare product object.

**Detailed description**:
`ProductService.list` returns `paginated(items, page, pageSize, total)` (`src/modules/products/product.service.ts:21`), which builds `{ data, pagination: { page, pageSize, total, totalPages } }` (`src/shared/http/response.ts:22-24`). `totalPages` is `Math.ceil(total / pageSize)` with a defensive `pageSize === 0 ? 0` branch (`src/shared/http/response.ts:13-19`). That zero branch is unreachable through HTTP because `pageSize` has a minimum of 1, but it protects other callers.

`getById`, `create` and `update` respond with the Prisma `Product` object directly (`src/modules/products/product.controller.ts:21,30,39`). There is no `{ data: ... }` wrapper, no `meta`, and no `links`. `delete` responds 204 with an empty body (`src/modules/products/product.controller.ts:48`). The asymmetry is consistent with the rest of the project (customers and orders behave similarly) but is not documented in an OpenAPI contract.

`pagination.total` counts the filtered set, not the page, and is produced by a `count` call inside the same Prisma batch transaction as `findMany` (`src/modules/products/product.repository.ts:27-35`). Using a transaction gives a consistent snapshot for `items` and `total`. The envelope's `page` and `pageSize` echo the validated request values, including a `page` beyond `totalPages`.

**Rule workflow**:
```
List -> 200 { data: Product[], pagination: { page, pageSize, total, totalPages } }
Get   -> 200 Product
Create-> 201 Product
Update-> 200 Product
Delete-> 204 (empty)
```

---

### Business Rule BR-17: SKU uniqueness is pre-checked in the service

**Overview**:
Before inserting a product, the service looks up the SKU and rejects a duplicate with `409 SKU_ALREADY_USED`.

**Detailed description**:
`ProductService.create` calls `this.products.findBySku(input.sku)` and throws `new ConflictError('SKU already in use', 'SKU_ALREADY_USED')` when a row is returned (`src/modules/products/product.service.ts:31-34`). `ConflictError` maps to HTTP 409 with the supplied machine code (`src/shared/errors/http-errors.ts:33-37`). This gives clients a specific, actionable code and a stable message, which the database error path does not (BR-18).

The check is a plain `prisma.product.findUnique({ where: { sku } })` (`src/modules/products/product.repository.ts:43-45`) executed outside any transaction. It reads through the same collation semantics described in BR-04, so it detects case and trailing-space collisions. It does not consume the row or reserve the SKU, and there is no advisory lock or `SELECT ... FOR UPDATE`; another request can insert the same SKU between the check and the insert.

The rule is implemented only for create. Update has an analogous check (BR-20), but no other code path — seed, tests, direct Prisma use — calls the service. The invariant that actually holds is the unique index; the service check is a UX and error-code improvement layered on top.

**Rule workflow**:
```
POST /products
  -> findBySku(sku)
       found -> 409 SKU_ALREADY_USED
       not found -> create
  -> race window until INSERT commits
```

---

### Business Rule BR-18: Database unique index is the final SKU invariant

**Overview**:
The unique index `products_sku_key` guarantees SKU uniqueness regardless of application logic, and its violation is translated to a generic `409 CONFLICT` rather than the module's specific code.

**Detailed description**:
The migration creates `UNIQUE INDEX products_sku_key(sku)` (`prisma/migrations/20260519182739_init/migration.sql:43`), and the Prisma model declares `sku String @unique` (`prisma/schema.prisma:58`). Any concurrent pair of writes that both pass the pre-check will still leave exactly one row: the second `INSERT` or `UPDATE` fails with Prisma `PrismaClientKnownRequestError` code `P2002`. Because the product service does not catch Prisma errors, the exception propagates to `next(err)` and is handled centrally.

`errorMiddleware` maps `P2002` to HTTP 409 with code `CONFLICT` and a message built from `err.meta.target`, for example `Unique constraint violation on: sku` (`src/middlewares/error.middleware.ts:37-47`). This differs from the sequential path, which returns `409 SKU_ALREADY_USED`. A client that keys behavior on the specific code sees two different shapes for the same business condition depending on timing. The `target` field is database-driver dependent and may be absent, in which case the message falls back to the generic string.

The same index also protects against duplicate SKUs introduced outside the API (seed scripts, migrations, direct SQL). There is no retry, no upsert and no idempotency key anywhere in the create path, so a client that retries after an ambiguous failure must handle the conflict itself. The index is not partial, so uniqueness covers inactive and deleted-adjacent rows equally.

**Rule workflow**:
```
Pre-check passes
  -> INSERT/UPDATE
       success -> row stored
       P2002   -> errorMiddleware -> 409 { code: "CONFLICT", message: "Unique constraint violation on: sku" }
  -> client sees a different code than the sequential 409 SKU_ALREADY_USED
```

---

### Business Rule BR-19: Existence requirement for read, update and delete

**Overview**:
`GET`, `PATCH` and `DELETE` by id return 404 when the product does not exist; update and delete perform an extra read before mutating.

**Detailed description**:
`getById` queries `findById` and throws `NotFoundError('Product')` when the result is `null` (`src/modules/products/product.service.ts:24-28`). `NotFoundError` builds the message `${resource} not found` with status 404 and code `NOT_FOUND` (`src/shared/errors/http-errors.ts:27-31`), so the response is `{ error: { code: "NOT_FOUND", message: "Product not found" } }`. The same method is called at the start of `update` and `delete` (`src/modules/products/product.service.ts:46,64`), which makes the read endpoint effectively the existence gate for all writes.

The existence check and the subsequent write are separate statements, not one transaction. A row can disappear between them, in which case the write raises Prisma `P2025`; the error middleware maps that to 404 with the generic message `Resource not found` (`src/middlewares/error.middleware.ts:48-53`). Clients therefore see two variants of the 404 message depending on timing: `Product not found` from the service and `Resource not found` from Prisma.

The read is a full-row `findUnique` with no `select`, so it fetches all columns only to confirm existence and then discards them (except in `getById`, where the row is the response). Update then performs a second query to load the updated row implicitly through Prisma's `update` return value. There is no `RETURNING`-style optimization or single-statement existence guard such as `updateMany` with a count check.

**Rule workflow**:
```
GET/PATCH/DELETE :id
  -> findUnique(id)
       null -> 404 NOT_FOUND "Product not found"
       row  -> GET returns it; PATCH/DELETE continue
  -> concurrent delete -> P2025 -> 404 NOT_FOUND "Resource not found"
```

---

### Business Rule BR-20: Update applies only defined fields and excludes self from SKU conflict

**Overview**:
The update builds a sparse data object from defined inputs and treats the current row as a non-conflict when the SKU is unchanged.

**Detailed description**:
After the existence check, `update` checks `if (input.sku)` and calls `findBySku` (`src/modules/products/product.service.ts:47-48`). If a row is found and its `id` differs from the target id, it throws `SKU_ALREADY_USED` (`src/modules/products/product.service.ts:49-51`). The self-exclusion is by id, not by value equality, so re-sending the same SKU is allowed and produces a normal update. Because the schema enforces `min(2)`, the truthiness guard and an `!== undefined` guard are equivalent in practice; the inconsistency is stylistic.

The data object is assembled with six separate conditional spreads (`src/modules/products/product.service.ts:53-60`). Fields not present are omitted, so Prisma leaves their columns untouched. Fields present are written as absolute values, not deltas. There is no field-level diff, no change log, no validation that the new value differs from the old one, and no handling for a `description` set to null (impossible at the schema layer). `updatedAt` is regenerated by Prisma because the model declares `@updatedAt` (`prisma/schema.prisma:65`).

The repository method signature accepts `Prisma.ProductUncheckedUpdateInput` (`src/modules/products/product.repository.ts:55-57`), a broad type that would permit writing `id`, `createdAt` and all scalars. The service's whitelist keeps the HTTP path narrow, but the repository itself does not enforce the restriction for future callers. There is no transaction around the read-check-write sequence.

**Rule workflow**:
```
PATCH payload
  -> getById (404)
  -> if sku supplied: findBySku; different id -> 409 SKU_ALREADY_USED
  -> build { sku?, name?, description?, priceCents?, stockQuantity?, active? }
  -> Prisma update by id (absolute values) -> updatedAt refreshed
  -> 200 full Product
```

---

### Business Rule BR-21: SKU is mutable and not snapshotted by order history

**Overview**:
A product's SKU can be changed at any time, and order read projections resolve the current SKU and name rather than the values at order creation.

**Detailed description**:
`updateProductSchema` includes `sku` (through the partial of the create schema) and the service writes it whenever supplied (`src/modules/products/product.service.ts:54`). There is no immutability rule, no history table for SKU changes and no event emitted. The only protection is uniqueness. A client can therefore rename `NB-DELL-001` to an arbitrary new identifier and the old identifier becomes immediately available for reuse (subject to no other row holding it).

Order items store `productId`, `quantity`, `unitPriceCents` and `totalCents` (`prisma/schema.prisma:99-114`), not SKU or name. When orders are read, the repository includes `product: { select: { id: true, sku: true, name: true } }` (`src/modules/orders/order.repository.ts:53`; also `src/modules/orders/order.service.ts:116,172`). The SKU and name shown for a historical order are therefore the product's current values, not the values at the time of sale. Price, by contrast, is snapshotted (`unitPriceCents` is copied at creation), so the model intentionally mixes snapshotted and live data.

This is a significant semantic for invoices, exports and audit reports: renaming a product retroactively changes how past orders are described, while changing its price does not. The Products Module has no knowledge of this consequence and no rule preventing a rename when open or historical orders exist.

**Rule workflow**:
```
PATCH { sku: "NEW-SKU" }
  -> uniqueness check excluding self
  -> UPDATE products.sku
  -> next order read: items[].product.sku returns "NEW-SKU" for all past orders
  -> order_items.unitPriceCents remains the original price
```

---

### Business Rule BR-22: Delete is physical and limited by the order_items foreign key

**Overview**:
`DELETE /products/:id` permanently removes the row and returns 204; if any `order_items` row references it, the database blocks the delete and the API returns an unmapped 500.

**Detailed description**:
`ProductService.delete` checks existence and then calls `prisma.product.delete` (`src/modules/products/product.service.ts:63-66`; `src/modules/products/product.repository.ts:59-61`). On success the controller sends 204 with an empty body (`src/modules/products/product.controller.ts:48`). There is no soft-delete column, no `deletedAt`, no archival table and no way to recover the row. The `active` flag is the only reversible alternative, but the module does not enforce or suggest it as a delete substitute.

The `order_items` table declares `FOREIGN KEY (productId) REFERENCES products(id) ON DELETE RESTRICT ON UPDATE CASCADE` (`prisma/migrations/20260519182739_init/migration.sql:119`; `prisma/schema.prisma:109`). Any product that has ever appeared in an order therefore cannot be deleted. Prisma raises `P2003` (foreign key constraint failed), which `errorMiddleware` does not translate (`src/middlewares/error.middleware.ts:37-54`); the request is logged as an unhandled error and returns 500 `INTERNAL_SERVER_ERROR`. The client receives no indication that the correct action is to deactivate instead.

The pre-check does not count references, so the failure is discovered only at the database. Deleting an unreferenced product succeeds even if it is present in a shopping cart or external system, because no such concept exists in this codebase. There is also no cascade to anything other than `order_items` (which is restrictive, not cascading), so the deletion is safe from orphan creation but not from client surprise.

**Rule workflow**:
```
DELETE /products/:id
  -> getById (404)
  -> prisma.product.delete
       no order_items reference -> 204, row gone
       order_items reference    -> P2003 -> 500 INTERNAL_SERVER_ERROR (logged)
       row already gone         -> P2025 -> 404 NOT_FOUND
```

---

### Business Rule BR-23: Shared ownership of price and stock with the Orders Module

**Overview**:
The Products Module owns absolute `priceCents` and `stockQuantity`, while the Orders Module reads price at order creation and mutates stock directly during status transitions; there is no shared service, lock or version.

**Detailed description**:
Every order item copies the current product price into `unitPriceCents` and computes `totalCents` at creation (`src/modules/orders/order.service.ts:75-84`). After that point, price changes in the Products Module have no effect on existing orders. This is a deliberate snapshot pattern, although it is implemented in the Orders Module and is invisible to the Products Module's contract. The product `priceCents` is a reference price for future orders only.

Stock is different: it is a live counter mutated by both modules. The Products Module writes absolute values through `PATCH` (`src/modules/products/product.service.ts:58`). The Orders Module reads the product rows inside a transaction, validates `stockQuantity >= quantity`, then issues `decrement` updates (`src/modules/orders/order.service.ts:204-231`), and issues `increment` updates on cancellation (`src/modules/orders/order.service.ts:233-243`). It does this through `Prisma.TransactionClient` rather than `ProductRepository`, so the two write paths share no code.

Because there is no row lock (`SELECT ... FOR UPDATE`), no conditional update (`where: { stockQuantity: { gte: quantity } }`), no version column and no optimistic concurrency check, a `PATCH stockQuantity` concurrent with a payment can overwrite the decrement, and two concurrent payments can both pass the stock check (the latter risk is documented in the architectural report). The Products Module also allows setting stock to an arbitrary absolute number with no record of the adjustment, so a manual correction is indistinguishable from a sale or return.

**Rule workflow**:
```
Products Module: PATCH stockQuantity = N   (absolute write, no version)
Orders Module:   PENDING -> PAID            (read, check, decrement)
Orders Module:   PAID|PROCESSING -> CANCELLED (increment)
                 no shared lock, no shared repository, no audit row
```

---

### Business Rule BR-24: Inactive products are blocked at order creation only

**Overview**:
The `active` flag is enforced when an order is created (`422 INACTIVE_PRODUCT`), but not when the order later moves through payment, shipping or cancellation.

**Detailed description**:
`OrderService.create` loads the referenced products and filters those with `active === false`, throwing `UnprocessableEntityError` with code `INACTIVE_PRODUCT` and the offending SKUs (`src/modules/orders/order.service.ts:66-73`). This is the only enforcement point in the codebase. The Products Module provides the flag and the `PATCH` path that changes it, but contains no check for pending orders, reservations or downstream references.

A product deactivated after an order is created does not invalidate that order. `changeStatus` loads the order and its items and applies the transition table and stock predicates (`src/modules/orders/order.service.ts:126-179`); it never reloads product `active` state. A pending order for a now-inactive product can therefore be paid if stock is available, and can proceed to shipping and delivery. Conversely, a product reactivated later makes new orders possible again with no trace of the inactive interval.

This deferred enforcement is consistent with the architectural finding that stock is validated only at payment, not at order creation. It means the catalog's availability flag is a point-in-time gate rather than a reservation or lifecycle constraint. No written requirement clarifies whether that is intended.

**Rule workflow**:
```
Order create: active=false -> 422 INACTIVE_PRODUCT
Product deactivated after order creation
  -> order remains valid
  -> PENDING -> PAID allowed if stock sufficient
  -> no re-check of active on any transition
```

---

### Business Rule BR-25: No domain audit trail for product mutations

**Overview**:
Product creation and every price, stock, SKU, name and active-state change leave only `createdAt`/`updatedAt`; there is no product history, actor field or domain event.

**Detailed description**:
The `Product` model has no `createdBy`, `updatedBy`, `version`, history table or soft-delete metadata (`prisma/schema.prisma:56-72`). The module receives `req.user` through the authentication middleware but the controller never passes it to the service (`src/modules/products/product.controller.ts:27-52`), so the service cannot attribute a change even if it wanted to. There is no event emission, outbox row or message broker anywhere in the project.

The only forensic trace is the HTTP access log emitted by `requestLogger`, which records `requestId`, `method`, `path`, `statusCode`, `durationMs` and `userId` on response finish (`src/middlewares/request-logger.middleware.ts:12-25`). It does not record the request body or the before/after values, and Pino redaction targets credentials only (`src/shared/logger/index.ts`). A price change is therefore visible as `PATCH /api/v1/products/<id> 200` by a user id, but the old and new prices are not recoverable from application logs. A `DELETE` is even less recoverable because the row is gone.

This is a governance gap rather than a functional bug. It means incidents such as a wrong price, an accidental stock overwrite or an unexplained SKU rename cannot be reconstructed from the system of record. The Orders Module has the opposite property: `order_status_history` is an append-only audit table (`prisma/schema.prisma:116-131`). The asymmetry between order audit and product audit is a design characteristic of the current codebase.

**Rule workflow**:
```
Any product mutation
  -> single UPDATE/INSERT/DELETE
  -> updatedAt changes (or row disappears)
  -> access log records actor + route + status, not payload or diff
  -> no product history row, no event, no version
```

---

### Business Rule BR-26: Response contract is the raw Prisma Product

**Overview**:
Product responses contain every column of the `products` table serialized directly, with no DTO, field selection or envelope for single resources.

**Detailed description**:
`ProductRepository` returns `Product` records from `findMany`, `findUnique`, `create` and `update` without `select` (`src/modules/products/product.repository.ts:25-61`). Controllers pass those objects to `res.json` (`src/modules/products/product.controller.ts:12,21,30,39`). The JSON therefore contains `id`, `sku`, `name`, `description`, `priceCents`, `stockQuantity`, `active`, `createdAt` and `updatedAt`. `Date` values are serialized as ISO-8601 strings by `JSON.stringify`. There are no computed fields such as `price` in currency units, `totalPages` per item, or `inStock` booleans.

Because the ORM record is the contract, adding a column to `prisma/schema.prisma` automatically changes the API response. There is no response schema, no OpenAPI document and no contract test asserting the field set. Removing or renaming a column is a breaking API change with no versioning beyond the path-level `/api/v1`. Clients cannot request sparse fieldsets, and there is no `Content-Location` header on create or `ETag` on read/update.

`description` can be `null`, so consumers must handle nullability. `priceCents` and `stockQuantity` are numbers within the JavaScript safe range but bounded by the database `INT` range. `createdAt`/`updatedAt` are millisecond-precision timestamps; no timezone policy is encoded, and Prisma returns them in UTC as `Date` objects.

**Rule workflow**:
```
Repository returns Prisma Product (all scalars)
  -> controller res.json(product)
  -> JSON: id, sku, name, description|null, priceCents, stockQuantity, active, createdAt, updatedAt
  -> list wraps an array of the same shape in { data, pagination }
```

---

### Business Rule BR-27: Centralized error contract with a specific SKU code

**Overview**:
The module throws shared error types and relies on one middleware to map them to HTTP status and a machine-readable code; only the SKU conflict has a module-specific code.

**Detailed description**:
Validation failures become `400 VALIDATION_ERROR` with a `details` array of `{ path, message }` (`src/middlewares/validate.middleware.ts:25-35`; `src/middlewares/error.middleware.ts:26-35`). Authentication produces `401 UNAUTHORIZED`. Missing products produce `404 NOT_FOUND`. Duplicate SKUs produce `409 SKU_ALREADY_USED` from the service or `409 CONFLICT` from the database race path. Any other error is logged with request id, method and path and returned as `500 INTERNAL_SERVER_ERROR` with no internal details (`src/middlewares/error.middleware.ts:56-64`). No product path produces 403 or 422.

The envelope is `{ error: { code, message, details? } }`. The `details` member is present only when the error carries it, and its shape differs by producer: Zod validation yields an array of paths/messages, while `ConflictError` and `UnauthorizedError` generally have none. There is no `requestId` echoed in the error body even though the logger records it, which makes support correlation dependent on the `X-Request-Id` response header set by `requestLogger` (`src/middlewares/request-logger.middleware.ts:8`).

The mapping is the only translation layer; controllers just `next(err)` (`src/modules/products/product.controller.ts:13-15,22-24,31-33,40-42,49-51`). This keeps the module consistent with customers, orders and users, but it also means unmapped Prisma codes (`P2003`, value-out-of-range) leak as generic 500s, as noted in BR-07 and BR-22.

**Rule workflow**:
```
AppError (ValidationError/UnauthorizedError/NotFoundError/ConflictError) -> statusCode + errorCode + details
ZodError (outside validate) -> 400 VALIDATION_ERROR
Prisma P2002 -> 409 CONFLICT ; P2025 -> 404 NOT_FOUND
anything else -> log + 500 INTERNAL_SERVER_ERROR
```

---

### Business Rule BR-28: Unsupported methods and paths fall through to the global 404

**Overview**:
The product router defines only five route/method combinations; any other method or subpath is not method-not-allowed but route-not-found, and authentication still runs first.

**Detailed description**:
`buildProductRouter` declares `GET /`, `GET /:id`, `POST /`, `PATCH /:id` and `DELETE /:id` (`src/modules/products/product.routes.ts:16-24`). Express does not synthesize 405 responses, so `PUT /api/v1/products/:id`, `POST /api/v1/products/:id` and `GET /api/v1/products/:id/anything` pass through the router without matching a handler. The application-level catch-all then throws `NotFoundError(\`Route ${req.method} ${req.originalUrl}\`)` (`src/app.ts:69-71`), producing 404 with message `Route PUT /api/v1/products/<id> not found`. Clients cannot distinguish "wrong method" from "wrong path" without reading the message.

Because `router.use(authenticate)` is registered first (`src/modules/products/product.routes.ts:14`), an unauthenticated request to an unsupported product path returns 401 rather than 404. This ordering protects route-topology information from anonymous callers but is an implicit rule: the catch-all itself is outside the module, and a future refactor that moves `authenticate` to individual routes would change the observable behavior.

There is no `OPTIONS` handler, no CORS preflight support in the app, no trailing-slash normalization beyond Express defaults, and no API version negotiation. The 404 body follows the standard error envelope with code `NOT_FOUND`.

**Rule workflow**:
```
Request to /api/v1/products...
  -> authenticate (401 if no valid token)
  -> match one of 5 method/path pairs?
       yes -> validate -> controller
       no  -> next() -> app catch-all -> 404 NOT_FOUND "Route <method> <url> not found"
```

---

## 4. Component Structure

```
src/modules/products/                     Products Module boundary (5 files, 237 lines)
├── product.routes.ts                     # Express Router factory; authenticate + validate composition; 5 routes (27 lines)
├── product.controller.ts                 # Transport adapter; list/getById/create/update/delete handlers; next(err) forwarding (53 lines)
├── product.service.ts                    # Business rules; SKU uniqueness, pagination, existence checks, sparse update (67 lines)
├── product.repository.ts                 # Prisma data access; where builder, batch list+count, CRUD primitives (62 lines)
└── product.schemas.ts                    # Zod contracts; create/update/list/param schemas and inferred types (28 lines)
```

Directly coupled files outside the boundary:

```
src/
├── app.ts                                # Composition root: new ProductRepository/ProductService/ProductController (lines 38-40)
├── routes/index.ts                       # Mounts buildProductRouter at /api/v1/products (line 27)
├── middlewares/
│   ├── auth.middleware.ts                # authenticate (JWT verification) applied to the whole router
│   ├── validate.middleware.ts            # Zod parse for body/query/params -> ValidationError
│   └── error.middleware.ts               # AppError/Zod/Prisma -> HTTP JSON; P2002/P2025 mapping
├── shared/
│   ├── errors/index.ts                   # ConflictError, NotFoundError barrel
│   └── http/response.ts                  # paginated() envelope and totalPages computation
├── modules/orders/
│   ├── order.service.ts                  # Direct tx.product reads/writes, active check, price snapshot, stock debit/replenish
│   └── order.repository.ts               # Includes product { id, sku, name } in order projections
└── modules/auth/
    └── auth.schemas.ts                   # Role is client-supplied at registration, widening the authenticated population

prisma/
├── schema.prisma                         # Product model (lines 56-72), OrderItem relation (99-114)
├── migrations/20260519182739_init/migration.sql  # products DDL (32-47), FK RESTRICT (119)
└── seed.ts                               # 20 demo products (142-163, 219-222)

tests/
├── setup.ts                              # prisma.product.deleteMany() before each test (line 13)
├── helpers/factories.ts                  # createTestProduct (lines 62-74)
└── orders.test.ts                        # Indirect product usage in 8 order tests
```

File-level responsibilities:

| File | Public surface | Internal dependencies | Notes |
|------|----------------|-----------------------|-------|
| `product.routes.ts` | `buildProductRouter(controller)` | `express`, `authenticate`, `validate`, product schemas, `ProductController` type | Factory, no module-level state; `router.use(authenticate)` applies to all routes |
| `product.controller.ts` | `ProductController` with five arrow-function `RequestHandler` fields | `express` types, `ProductService` type, `ListProductsQuery` type | No business logic; `req.params.id!` and `req.query as unknown as ListProductsQuery` casts |
| `product.service.ts` | `ProductService` with `list`, `getById`, `create`, `update`, `delete` | `Product` type, `ProductRepository` type, shared errors, schemas types, `paginated` | Owns SKU rules, pagination math, existence checks; no transaction |
| `product.repository.ts` | `ProductRepository`, `ProductListFilters` type | `@prisma/client` types | `buildWhere` private; `findManyByIds` is unused |
| `product.schemas.ts` | `productIdParamSchema`, `createProductSchema`, `updateProductSchema`, `listProductsQuerySchema`, three input types | `zod` | Contracts only; `updateProductSchema` derived via `.partial()` |

Boundary observations:

- Layering is respected: controller imports no Prisma, service imports no Express, repository contains no business decisions.
- The module uses asynchronous handlers with manual `try/catch` and `next(err)` in every controller method, so no error is swallowed.
- The module exports classes, not interfaces; there is no port/adapter abstraction, so Prisma is the only persistence seam and tests require a real database.
- The `ProductListFilters` type is exported from the repository but imported by nobody; the service constructs the object structurally.
- No barrel file (`index.ts`) exists for the module; consumers import concrete files.

---

## 5. Dependency Analysis

### Internal dependency graph

```
src/app.ts (composition root)
  ├── new ProductRepository(prisma)
  ├── new ProductService(productRepository)
  ├── new ProductController(productService)
  └── buildApiRouter({ products: productController })

src/routes/index.ts
  └── buildProductRouter(controllers.products)
        ├── authenticate (src/middlewares/auth.middleware.ts)
        │     ├── env (src/config/env.ts)
        │     └── UnauthorizedError (src/shared/errors/index.ts)
        ├── validate (src/middlewares/validate.middleware.ts)
        │     └── ValidationError (src/shared/errors/index.ts)
        ├── productIdParamSchema / createProductSchema / updateProductSchema / listProductsQuerySchema
        │     └── zod (external)
        └── ProductController
              ├── ProductService (type)
              │     ├── ProductRepository (type)
              │     │     └── @prisma/client (external) -> MySQL products table
              │     ├── ConflictError / NotFoundError (src/shared/errors/index.ts)
              │     ├── paginated / PaginatedResponse (src/shared/http/response.ts)
              │     └── CreateProductInput / UpdateProductInput / ListProductsQuery (types)
              └── ListProductsQuery (type)

Cross-module (no import, table-level coupling):
src/modules/orders/order.service.ts -> Prisma tx.product.findMany / update (read price, active, stock)
src/modules/orders/order.repository.ts -> Prisma include product { id, sku, name }
prisma/seed.ts -> Prisma product.create
tests/helpers/factories.ts -> Prisma product.create
```

### Dependency direction and cycles

- Direction is strictly downward: routes -> controller -> service -> repository -> Prisma. The controller's dependencies are type-only imports, so runtime coupling is limited to the composition root and the router factory.
- No cycle exists at file or module level. The Products Module imports no feature module; it depends only on cross-cutting middleware and shared utilities. `products` does not import `orders`, and `orders` does not import `products` (it reaches the table through Prisma).
- The composition root is the only place where concrete instances are constructed (`src/app.ts:38-40`), and `src/routes/index.ts:27` is the only place the router is mounted. Both are outside the module boundary, which is why the module's afferent coupling is 2.
- The module inherits a dependency on `src/config/env.ts` transitively through `authenticate`, because JWT verification reads `env.JWT_SECRET`. There is no direct configuration import in the module.
- `shared/http/response.ts` is a leaf shared utility with no dependencies, imported by product, customer and order services; it is a stable contract.

### External dependencies used by the component

| Dependency | Version | Used by | Purpose |
|------------|---------|---------|---------|
| `express` | 4.21.1 (`@types/express` 4.17.21 dev) | `product.routes.ts:1`, `product.controller.ts:1` | Router and request handler types |
| `zod` | 3.23.8 | `product.schemas.ts:1`; via `validate.middleware.ts:2` | Request contract definition and parsing |
| `@prisma/client` | 5.22.0 | `product.repository.ts:1`, `product.service.ts:1` | Typed ORM client, `Product` model and `Prisma.*` input types |
| MySQL | 8.0 (Compose image) | Indirect through Prisma | Durable product storage, unique index, FK RESTRICT, collation semantics |
| `jsonwebtoken` | 9.0.2 | Indirect through `auth.middleware.ts:2` | Bearer token verification gating all product routes |
| `uuid` | 11.0.3 | Indirect through `request-logger.middleware.ts:2` | Request id generation for logs (not product ids; those are Prisma defaults) |

### Dependency risk notes

- `@prisma/client` 5.22.0 must stay aligned with the `prisma` CLI 5.22.0 in devDependencies; both are pinned exactly, so upgrades are deliberate but also synchronized manually.
- The module's runtime dependency on `express@4` includes its async-error behavior: rejected promises in handlers are not caught automatically, which is why every controller method uses `try/catch` + `next(err)`.
- There is no HTTP client, cache client, message broker, search engine or file storage dependency. The catalog is database-only.
- Database portability is limited by MySQL-specific collation behavior (case-insensitive unique SKU, PAD SPACE comparisons) and by `LIKE` semantics; moving to PostgreSQL would change the effective SKU and search rules even though TypeScript still compiles.
- Prisma is imported as a type in `product.service.ts` (`import type { Product }`), so the service has no runtime ORM dependency; only the repository does.
- The transitive dependency on `env.ts` means importing the router in a tool or test context triggers environment validation and can call `process.exit(1)` if `JWT_SECRET` or `DATABASE_URL` is missing.

---

## 6. Afferent and Efferent Coupling

Coupling is measured statically from `import` / `import type` / `export ... from` statements across the 39 TypeScript files in `src`, resolving relative `.js` specifiers to `.ts` sources. External npm packages are excluded from internal counts and reported separately. For the module boundary, only distinct files outside the boundary are counted, so intra-module imports are not double-counted.

| Component | Type | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Critical |
|-----------|------|------------------------|------------------------|----------|
| Products Module (directory boundary) | Feature module | 2 (`src/app.ts`, `src/routes/index.ts`) | 4 internal files (`middlewares/auth.middleware.ts`, `middlewares/validate.middleware.ts`, `shared/errors/index.ts`, `shared/http/response.ts`) | High |
| `ProductService` (class) | Service | 2 (`product.controller.ts` type-only, `app.ts`) | 4 internal files (`product.repository.ts`, `shared/errors/index.ts`, `product.schemas.ts`, `shared/http/response.ts`) + `@prisma/client` type | High |
| `ProductController` (class) | Controller | 3 (`product.routes.ts`, `app.ts`, `routes/index.ts` type-only) | 2 internal files (`product.service.ts`, `product.schemas.ts`) + `express` types | Medium |
| `ProductRepository` (class) | Repository | 2 (`product.service.ts` type-only, `app.ts`) | 0 internal + `@prisma/client` | Medium |
| `product.schemas.ts` (schemas and types) | Contract | 3 (`product.routes.ts`, `product.service.ts`, `product.controller.ts`) | 0 internal + `zod` | High |
| `buildProductRouter` (function) | Router factory | 1 (`routes/index.ts`) | 4 internal files (`auth.middleware.ts`, `validate.middleware.ts`, `product.schemas.ts`, `product.controller.ts`) + `express` | Medium |
| `ProductListFilters` (type) | Type contract | 0 outside `product.repository.ts` | 0 | Low |
| `products` table (data component) | Data model | 4 write/read paths outside the module (`order.service.ts` x2, `order.repository.ts`, `seed.ts`) plus test factories | 0 | High |
| `authenticate` middleware (dependency) | Security middleware | 5 routers (`auth`, `customers`, `orders`, `products`, `users`) | 2 internal files (`config/env.ts`, `shared/errors/index.ts`) + `jsonwebtoken` | High |
| `paginated` helper (dependency) | Shared utility | 3 services (`customer`, `order`, `product`) | 0 | Low |

### Interpretation

- The Products Module has low afferent coupling at the code level (Ca=2): only the composition root and the router registry import it. This is healthy isolation for a CRUD module and means a signature change is caught by the compiler in exactly those two places.
- Its efferent coupling (Ce=4 internal files) is moderate and almost entirely cross-cutting: authentication, validation, the error taxonomy and the pagination envelope. It has exactly one domain-facing dependency, `ProductRepository`, and no feature-module dependency. This is the cleanest module-level dependency profile in the system alongside `customers`.
- `product.schemas.ts` is the most depended-on file inside the boundary (Ca=3) and is also the most stable (Ce=0). It is the module's contract anchor: router, service and controller all consume its inferred types.
- `ProductRepository` is a leaf in the internal graph (Ce=0). `findManyByIds` is exported but unused, which makes it dead surface area rather than coupling.
- The real coupling pressure on the module is invisible to import analysis. `OrderService` reads and writes `products` through Prisma (`Ca` at the table level = 4 non-test paths), which means schema and semantic changes to the product catalog can break order behavior without any compile-time signal from the Products Module. This is data coupling, not code coupling.
- `authenticate` has the highest fan-in among the module's dependencies (Ca=5). Any change to token verification affects every business router, including this one, even though no product file imports it directly.

---

## 7. Endpoints

The module exposes five REST endpoints under the `/api/v1` prefix applied in `src/app.ts:67` and the `/products` mount in `src/routes/index.ts:27`. All require a bearer JWT; none requires a role.

| Endpoint | Method | Description | Auth | Success | Main errors |
|----------|--------|-------------|------|---------|-------------|
| `/api/v1/products` | GET | List products with pagination, substring search and active filter; ordered by `createdAt DESC` | Bearer JWT | `200` `{ data: Product[], pagination }` | `400 VALIDATION_ERROR` (page/pageSize/search/active); `401 UNAUTHORIZED`; `500 INTERNAL_SERVER_ERROR` |
| `/api/v1/products/{id}` | GET | Get one product by UUID | Bearer JWT | `200` `Product` | `400 VALIDATION_ERROR` (non-UUID id); `401 UNAUTHORIZED`; `404 NOT_FOUND`; `500 INTERNAL_SERVER_ERROR` |
| `/api/v1/products` | POST | Create a product; SKU must be unique | Bearer JWT | `201` `Product` | `400 VALIDATION_ERROR`; `401 UNAUTHORIZED`; `409 SKU_ALREADY_USED` or `409 CONFLICT`; `500 INTERNAL_SERVER_ERROR` |
| `/api/v1/products/{id}` | PATCH | Partially update a product; only supplied fields change | Bearer JWT | `200` `Product` | `400 VALIDATION_ERROR`; `401 UNAUTHORIZED`; `404 NOT_FOUND`; `409 SKU_ALREADY_USED` / `409 CONFLICT`; `500 INTERNAL_SERVER_ERROR` |
| `/api/v1/products/{id}` | DELETE | Permanently delete a product | Bearer JWT | `204` empty body | `400 VALIDATION_ERROR`; `401 UNAUTHORIZED`; `404 NOT_FOUND`; `500 INTERNAL_SERVER_ERROR` (FK restriction on referenced product) |

Route implementations: `src/modules/products/product.routes.ts:16` (list), `:17` (get by id), `:18` (create), `:19-23` (patch), `:24` (delete).

### Request examples

Create (`POST /api/v1/products`):

```json
{
  "sku": "NB-DELL-001",
  "name": "Notebook Dell Inspiron 15",
  "description": "Notebook 15.6\", Intel i5, 16GB RAM, 512GB SSD",
  "priceCents": 459900,
  "stockQuantity": 25,
  "active": true
}
```

Minimal create (defaults applied: `stockQuantity = 0`, `active = true`):

```json
{
  "sku": "CABO-USC-001",
  "name": "Cabo USB-C 2m Anker",
  "priceCents": 12900
}
```

Partial update (`PATCH /api/v1/products/{id}`):

```json
{
  "priceCents": 439900,
  "active": false
}
```

List with filters (`GET /api/v1/products?page=2&pageSize=10&search=notebook&active=true`):

```
page     -> 2
pageSize -> 10
search   -> "notebook" (trimmed, matched against name OR sku)
active   -> true (only literal "true"/"false" accepted)
```

### Response examples

List success (`200`):

```json
{
  "data": [
    {
      "id": "0f0f0f0f-0f0f-4f0f-8f0f-0f0f0f0f0f0f",
      "sku": "NB-DELL-001",
      "name": "Notebook Dell Inspiron 15",
      "description": "Notebook 15.6\", Intel i5, 16GB RAM, 512GB SSD",
      "priceCents": 459900,
      "stockQuantity": 25,
      "active": true,
      "createdAt": "2026-11-08T23:00:00.000Z",
      "updatedAt": "2026-11-08T23:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

Create success (`201`): the bare `Product` object shown above, without the `pagination` member.

Delete success (`204`): no body.

Validation error (`400`):

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      { "path": "priceCents", "message": "Number must be greater than or equal to 0" },
      { "path": "sku", "message": "String must contain at least 2 character(s)" }
    ]
  }
}
```

Duplicate SKU, sequential path (`409`):

```json
{
  "error": {
    "code": "SKU_ALREADY_USED",
    "message": "SKU already in use"
  }
}
```

Duplicate SKU, race path (`409`):

```json
{
  "error": {
    "code": "CONFLICT",
    "message": "Unique constraint violation on: sku"
  }
}
```

Not found (`404`):

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Product not found"
  }
}
```

Missing token (`401`):

```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Missing or invalid Authorization header"
  }
}
```

Foreign-key failure on delete (`500`, logged server-side):

```json
{
  "error": {
    "code": "INTERNAL_SERVER_ERROR",
    "message": "Internal server error"
  }
}
```

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol / Mechanism | Data Format | Error Handling |
|-------------|------|---------|----------------------|-------------|----------------|
| MySQL `products` table | Internal database | Durable catalog storage, unique SKU index, active/name indexes, FK target for `order_items` | Prisma Client 5.22 over MySQL wire protocol | Relational rows mapped to `Product` (UUID `CHAR(36)`, `VARCHAR`, `TEXT`, `INT`, `BOOLEAN`, `DATETIME(3)`) | Prisma errors propagate to `next(err)`; `P2002` -> 409, `P2025` -> 404, `P2003`/range errors -> 500 with Pino log |
| `authenticate` middleware | Internal security middleware | Gate every product route behind a valid JWT and hydrate `req.user` | In-process Express middleware | Bearer JWT (HMAC-SHA256), claims `sub`, `email`, `role` | Missing/invalid token -> `UnauthorizedError` -> 401; no fallback or anonymous mode |
| `validate` middleware | Internal request pipeline | Parse and validate body, query and params with Zod | In-process middleware | Parsed TypeScript objects; Zod `ZodError` mapped to field paths | `ZodError` -> `ValidationError` -> 400 with `details`; non-Zod errors rethrown to `next` |
| Centralized error middleware | Internal cross-cutting translator | Convert `AppError`, `ZodError`, Prisma `P2002`/`P2025` to HTTP JSON; log and mask unknowns | In-process Express error handler | `{ error: { code, message, details? } }` | Unknown errors -> 500; logs `err`, `requestId`, `method`, `path` |
| Pagination helper | Internal shared utility | Build `{ data, pagination }` envelope and compute `totalPages` | In-process function call | TypeScript object | None; pure computation, `pageSize === 0` guarded |
| Orders Module (`OrderService`) | Internal module, table-level | Reads price/active/stock at order creation; decrements stock on payment; increments on cancellation | Direct Prisma `TransactionClient` queries | Prisma `Product` rows and scalar updates | Its own errors (`NOT_FOUND`, `INACTIVE_PRODUCT`, `INSUFFICIENT_STOCK`); no call back into `ProductService` |
| Orders Module (`OrderRepository`) | Internal module, read projection | Includes `product { id, sku, name }` in order responses | Prisma relation `include`/`select` | Nested JSON in order payloads | None specific; a missing product is impossible while FK RESTRICT holds |
| Prisma seed script | Operational bootstrap | Populate 20 demo products before demo orders | Prisma Client with a separate `PrismaClient` instance | `products` rows with fixed SKUs/prices/stock | Process exits non-zero on failure (`prisma/seed.ts:327-332`) |
| Test factories and setup | Test infrastructure | Create products for order tests; truncate products before each test | Prisma Client directly | `Product` rows | Test helper throws if creation fails; setup has no environment guard |
| Express HTTP server | Inbound HTTP | Expose `/api/v1/products/*` and the global 404 fallback | HTTP/1.1 JSON | JSON request/response | Centralized error middleware; 404 catch-all in `src/app.ts:69-71` |
| Pino request logger | Cross-cutting observability | Log method, path, status, duration and `userId` per request | In-process logger | Structured JSON logs with request id | Not explicitly handled; no payload logging |

### Integration notes

- The Products Module has exactly one external stateful integration: MySQL. There is no cache (Redis or in-memory), no search engine, no object storage for product images, no pricing service, no tax service, no inventory service, no ERP integration and no event publisher.
- All internal integrations are in-process method calls. There is no network failure mode between the module and its collaborators, but there is also no transaction that spans the Products Module and the Orders Module.
- The `products` table is a shared integration surface: `OrderService` reads it to price and validate orders and writes it to move stock. This is the only place in the system where two feature modules share a writable aggregate without going through one module's service boundary.
- The module consumes `req.user` only through middleware; it never forwards the actor id. Any future audit, approval or per-role rule requires threading that value through the controller and service.
- `Zod` validates the HTTP boundary only. Seed and test factories bypass it, which is why demo and test data can contain values that the API would reject (for example, a product without a description is fine, but nothing prevents a test from inserting a negative stock).
- The `X-Request-Id` response header is the only correlation identifier exposed to clients; it is set before authentication, so 401 and 400 responses also carry it.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered architecture (3-tier per module) | routes -> controller -> service -> repository | `src/modules/products/*` | Separation of HTTP adaptation, business rules, persistence |
| Repository pattern | `ProductRepository` wraps all Prisma `Product` operations; `buildWhere` is private | `src/modules/products/product.repository.ts:10-61` | Keeps Prisma query construction out of the service |
| Constructor dependency injection | `new ProductService(productRepository)`, `new ProductController(productService)` | `src/app.ts:38-40` | Explicit dependencies, no service locator or DI container |
| Composition root | `buildControllers(prisma)` wires every module | `src/app.ts:26-53` | Single place for object graph construction |
| Router factory | `buildProductRouter(controller)` returns a configured `Router` | `src/modules/products/product.routes.ts:12-27` | Avoids module-level singletons; enables controller injection |
| Schema-first boundary validation | Zod schemas for body, query and params; parsed output replaces input | `src/modules/products/product.schemas.ts`; `src/middlewares/validate.middleware.ts` | Declarative contracts and inferred TypeScript types |
| Middleware chain | `authenticate` -> `validate` -> controller, in that order | `src/modules/products/product.routes.ts:14-24` | Composable cross-cutting concerns; auth before input parsing |
| Centralized error taxonomy | `AppError` subclasses thrown by service, translated by one error handler | `src/shared/errors/*`; `src/middlewares/error.middleware.ts` | Uniform error envelope and safe 500 masking |
| Pagination envelope | `paginated(items, page, pageSize, total)` | `src/shared/http/response.ts:22-24` | Consistent list contract across modules |
| Batch transaction for read consistency | `prisma.$transaction([findMany, count])` | `src/modules/products/product.repository.ts:27-35` | Consistent snapshot of page items and total |
| Anemic domain model | `Product` is the Prisma record; no entity behavior or invariants | `prisma/schema.prisma:56-72` | Simple CRUD; all rules live in service/schema |
| Soft-state flag (partial soft delete) | `active` boolean with `PATCH` toggle | `src/modules/products/product.schemas.ts:11`; `src/modules/products/product.service.ts:59` | Reversible availability without deletion |
| Sparse update assembly | Conditional spreads with `!== undefined` | `src/modules/products/product.service.ts:53-60` | Avoid overwriting unspecified fields |
| Arrow-function handler fields | `RequestHandler` class properties | `src/modules/products/product.controller.ts:8-52` | Preserves `this` when passed as route callbacks |
| Guard-clause validation | Existence checks throw before mutation | `src/modules/products/product.service.ts:24-28,46,64` | Simple, readable preconditions |

### Architectural decisions observed

1. **The module is a conventional CRUD slice over a single table.** There is no aggregate root, no domain layer and no use-case abstraction; the service is a thin set of rules over Prisma.
2. **Validation is split between Zod and the service.** Shape, type, bounds and defaults live in schemas; uniqueness, existence and partial-application semantics live in the service. The split is consistent but means neither layer alone describes the full contract.
3. **Authentication is module-wide, authorization is absent.** The router applies `authenticate` once and never narrows further. This is the project-wide pattern; only the users route uses `requireRole`.
4. **Writes are single-statement and non-transactional.** Create, update and delete do not open a transaction. Atomicity is per row; cross-row operations (the SKU pre-check plus insert, the existence check plus update) are deliberately not atomic.
5. **The catalog is a shared table, not a shared service.** Orders bypass the module's repository, so the Products Module is not the authoritative gateway for its own data. This is the most consequential architectural decision affecting the component.
6. **Responses expose ORM records directly.** There is no DTO layer, so the database schema is the API schema for products. This favors speed of development over contract stability.
7. **Search is delegated to the database.** There is no search index or application-side filtering; `LIKE` semantics and collation define matching, and pagination is offset-based.
8. **Deletion is physical, with `active` as the only reversible state.** The design offers two distinct lifecycle mechanisms without a rule connecting them.

### Quality attributes

| Attribute | Assessment | Evidence |
|-----------|------------|----------|
| Security | Weak at the authorization layer; authentication is solid but undifferentiated | No `requireRole` on any product route (`src/modules/products/product.routes.ts:14-24`); UUID param validation and parameterized Prisma queries mitigate injection |
| Reliability | Simple and predictable for single-row operations; failure modes at the database edge are poorly mapped | No retries needed for in-process calls; `P2003` and range errors become 500 (`src/middlewares/error.middleware.ts:37-54`) |
| Maintainability | Good layering and small files; contract duplication with Prisma types and two writers to the same table | 5 files, 237 lines, clear responsibilities; `findManyByIds` dead code; `Prisma.ProductUnchecked*` broad repository types |
| Testability | Poor: no unit seams exercised and no direct tests | Concrete repository dependency, no interfaces, no mocks; tests require a live MySQL |
| Performance | Adequate for small catalogs; list/search scale poorly | Offset pagination, leading-wildcard `LIKE`, no `createdAt` index, no cache, batch transaction overhead |
| Scalability | Stateless HTTP layer scales horizontally; database is the shared bottleneck | No server-side state in the module; all reads/writes hit one MySQL instance |
| Observability | Request-level logging only; no product-specific metrics or audit | `src/middlewares/request-logger.middleware.ts:12-25`; no counters, traces or history table |
| Consistency | Row-level only; cross-module stock consistency is best-effort | No transaction spanning Products and Orders; no version column; last-writer-wins on `PATCH` |
| API ergonomics | Consistent envelope for lists, raw objects for singles; strict filter parsing | `src/shared/http/response.ts`; `src/modules/products/product.schemas.ts:16-24` |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact | Evidence |
|------------|----------------|-------|--------|----------|
| High | Authorization | No role checks on create, update or delete; any authenticated `OPERATOR` can change prices, stock or SKUs and delete products | Financial and catalog integrity depend on every token holder behaving correctly; self-registered users obtain tokens | `src/modules/products/product.routes.ts:14-24`; `src/middlewares/auth.middleware.ts:49-61`; `src/modules/auth/auth.schemas.ts:7` |
| High | Cross-module stock consistency | Products writes absolute `stockQuantity` while Orders decrements/increments directly, with no lock, version or shared repository | Lost updates, stock drift, oversell under concurrency | `src/modules/products/product.service.ts:58`; `src/modules/orders/order.service.ts:204-243` |
| High | Delete path | `P2003` from the `order_items` FK restriction is not mapped; deleting a referenced product returns 500 | Clients cannot distinguish a permanent business restriction from a server fault; misleading retries | `prisma/migrations/20260519182739_init/migration.sql:119`; `src/middlewares/error.middleware.ts:37-54` |
| Medium | SKU uniqueness race | Check-then-write in create and update; concurrent duplicates surface as `CONFLICT` instead of `SKU_ALREADY_USED` | Inconsistent client error handling; no idempotent create | `src/modules/products/product.service.ts:31-34,47-52`; `src/middlewares/error.middleware.ts:37-47` |
| Medium | Identity normalization | No trim/case fold on SKU or name; uniqueness and search depend on `utf8mb4_unicode_ci` and PAD SPACE | Environment-dependent duplicates and lookups; whitespace SKUs accepted | `src/modules/products/product.schemas.ts:6-7`; `prisma/migrations/20260519182739_init/migration.sql:29,47` |
| Medium | Numeric bounds | `priceCents` and `stockQuantity` accept values beyond signed `INT`; Zod has no maximum | Out-of-range writes become 500; order totals can also overflow `INT` | `src/modules/products/product.schemas.ts:9-10`; `prisma/schema.prisma:61-62`; `src/modules/orders/order.service.ts:82` |
| Medium | Empty patch | `PATCH {}` passes validation and performs a no-op write that changes `updatedAt` | Useless writes, misleading audit timestamps, cache invalidation churn | `src/modules/products/product.schemas.ts:14`; `src/modules/products/product.service.ts:53-60` |
| Medium | Nullability asymmetry | `description` can be `null` on create but not on update | Clients cannot clear a description; schema and API diverge | `src/modules/products/product.schemas.ts:8,14`; `src/modules/products/product.service.ts:38,56` |
| Medium | SKU mutability | SKU can be renamed; order projections resolve current SKU/name, not the sold values | Historical orders and invoices change retroactively; external references break | `src/modules/products/product.service.ts:54`; `src/modules/orders/order.repository.ts:53` |
| Medium | Order active rule | Deactivation is blocked only at order creation; pending orders can still be paid for inactive products | Catalog withdrawal is not a lifecycle control; operational surprise | `src/modules/orders/order.service.ts:66-73,126-179` |
| Medium | Auditability | No product history, actor fields, change diff or domain event | Price/stock/SKU incidents are not reconstructable; deletions are irreversible | `prisma/schema.prisma:56-72`; `src/middlewares/request-logger.middleware.ts:12-25` |
| Medium | Search scalability | Leading-wildcard `LIKE '%term%'` over two columns, no full-text index, no term length cap | Full scans and slow queries on large catalogs; possible wildcard-pattern abuse | `src/modules/products/product.repository.ts:16-21`; `src/modules/products/product.schemas.ts:19` |
| Medium | Pagination model | Offset pagination with no tie-breaker and `createdAt` unindexed; `page` unbounded | Duplicate/missing rows during concurrent writes; expensive deep offsets | `src/modules/products/product.repository.ts:28-34`; `src/modules/products/product.schemas.ts:17-18` |
| Medium | Test coverage | No `products` test file; no endpoint, validation, uniqueness, update or delete tests | Regressions in the whole module are undetected; only compile errors are caught | `tests/` contains only `auth.test.ts` and `orders.test.ts` |
| Low | Dead code | `ProductRepository.findManyByIds` has no callers; Orders duplicates the query | Maintenance noise; misleading API surface | `src/modules/products/product.repository.ts:47-49`; `src/modules/orders/order.service.ts:62` |
| Low | Dead branch | `buildPagination` handles `pageSize === 0`, unreachable through HTTP because the schema requires >= 1 | Untested defensive branch; minor complexity | `src/shared/http/response.ts:18`; `src/modules/products/product.schemas.ts:18` |
| Low | Controller casts | `req.params.id!` and `req.query as unknown as ListProductsQuery` bypass type safety | Validated data is not statically guaranteed at the handler boundary; refactors can silently break | `src/modules/products/product.controller.ts:10,20,38,47` |
| Low | Repository type breadth | `Prisma.ProductUncheckedCreateInput` / `UpdateInput` allow writing any scalar, including `id` and timestamps | The service whitelist is the only mass-assignment barrier for future callers | `src/modules/products/product.repository.ts:51-57` |
| Low | Validation asymmetry | `if (input.sku)` truthiness check for conflict versus `!== undefined` for the write | Fragile if the minimum length ever changes; stylistic inconsistency | `src/modules/products/product.service.ts:47,54` |
| Low | Boolean parsing | `active` accepts only lowercase `"true"`/`"false"` | Client friction and 400s for common boolean spellings | `src/modules/products/product.schemas.ts:20-23` |
| Low | API contract artifacts | No OpenAPI/Swagger document, no response schema, no contract tests | Consumers depend on source reading; ORM changes silently change the API | No `openapi`/`swagger` files; raw Prisma returns in `product.controller.ts` |
| Low | Transport security | No helmet, HSTS, CORS policy or HTTPS enforcement in the app | Catalog mutations and tokens depend on external TLS termination | `src/app.ts:55-75`; `package.json:25-33` |
| Low | Test data safety | Test setup truncates `products` with no environment guard | Running tests against a non-test database destroys catalog data | `tests/setup.ts:8-16` |

### Code smells observed

- **Dead code**: `findManyByIds` (`src/modules/products/product.repository.ts:47-49`) is never called; the Orders Module re-implements it inline.
- **Primitive obsession**: `priceCents` is a bare `number` with no `Money` type, currency or rounding concept; `stockQuantity` is a bare integer shared by two modules.
- **Anemic domain model**: `Product` is a Prisma record; all invariants are enforced outside the data type, and nothing prevents constructing an invalid product in application code.
- **Magic numbers**: `pageSize` maximum 100, `description` maximum 2000, `name` 200 and `sku` 64 appear as literals in schemas and are duplicated implicitly in `prisma/schema.prisma`; no shared constants.
- **Feature envy / divergent change**: the Orders Module knows product pricing, active filtering and stock arithmetic, which are catalog concerns, and duplicates access that `ProductRepository` already exposes.
- **Inappropriate intimacy with the schema**: the controller/service contract is the ORM record, so any column addition changes the HTTP response without a deliberate API decision.
- **Type-unsafe boundary**: non-null assertions and a double cast in the controller defeat strict TypeScript at the exact point where validated data should be guaranteed (`tsconfig.json` enables `strict` and `noUncheckedIndexedAccess`).
- **Temporal coupling**: `authenticate` must run before `validate` and before the controller; `validate` must run before the non-null assertion is safe. This ordering is enforced only by line order in the router.
- **Duplicated where-building knowledge**: `buildWhere` lives in the repository, but Orders builds equivalent product filters inline.
- **Inconsistent truthiness**: `if (input.sku)` versus `!== undefined` in the same method for the same value.
- **Speculative generality**: exported `ProductListFilters` type and unused `findManyByIds` add surface without consumers.

---

## 11. Test Coverage Analysis

### Test inventory

| Test file / helper | Location | Scope | Count |
|--------------------|----------|-------|-------|
| Auth integration suite | `tests/auth.test.ts` | Register/login/me; issues the tokens used by protected routes | 7 `it` blocks |
| Orders integration suite | `tests/orders.test.ts` | Order lifecycle; creates products through a Prisma factory and reads `products.stockQuantity` | 8 `it` blocks |
| Test factories | `tests/helpers/factories.ts` | `getTestApp`, `createTestUser`, `createTestCustomer`, `createTestProduct`, `createTestOrder`-style helpers, `bootstrapAuthenticatedUser` | Shared helpers |
| Global setup | `tests/setup.ts` | Prisma connect/disconnect; deletes products (among other tables) before each test | Hooks |
| Vitest configuration | `vitest.config.ts` | `tests/**/*.test.ts`, node environment, serial single fork, 30 s timeouts, setup file | N/A |
| Seed script (not a test) | `prisma/seed.ts` | Inserts 20 products; not executed by `npm test` | N/A |

There is **no `tests/products.test.ts`** and no unit test for `product.service.ts`, `product.repository.ts`, `product.schemas.ts`, `product.controller.ts` or `product.routes.ts`. No test issues an HTTP request to `/api/v1/products`.

### Coverage table

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| `product.routes.ts` | 0 | 0 direct; route file is loaded by every order test through `buildApp` | Not instrumented; no product route is ever invoked | None; middleware order, path matching and method handling unverified |
| `ProductController` (all 5 handlers) | 0 | 0 direct | Not instrumented; handlers never execute in tests | None; status codes, envelopes and `next(err)` paths unverified |
| `ProductService.list` | 0 | 0 direct | Not instrumented | None; pagination math, filters and empty results unverified |
| `ProductService.getById` | 0 | 0 direct | Not instrumented | None; 200 and 404 branches unverified |
| `ProductService.create` | 0 | 0 direct (factories bypass it) | Not instrumented | None; SKU conflict, defaults and whitelist unverified |
| `ProductService.update` | 0 | 0 direct | Not instrumented | None; sparse application and self-exclusion unverified |
| `ProductService.delete` | 0 | 0 direct | Not instrumented | None; 204, 404 and FK 500 paths unverified |
| `ProductRepository` | 0 | 0 direct; `prisma.product` is used in orders tests for assertions | Not instrumented; `findUnique` exercised indirectly | Weak for the module: list/where/search/create/update/delete are unexercised |
| `product.schemas.ts` | 0 | 0 direct | Not instrumented; every schema branch untested | None; no boundary values, no invalid `active`, no page bounds |
| `createTestProduct` factory | 0 | 8 orders tests create at least one product | Indirect; bypasses the API and all product validation | Good as fixture setup; provides no coverage of the module under test |
| Product table constraints | 0 | 0 | Unique SKU and FK RESTRICT never exercised by a product test | The most important integrity rules are untested at the API level |

### Qualitative coverage assessment

`vitest.config.ts` defines no `coverage` block, `package.json` has no `@vitest/coverage-*` dependency and no `test:coverage` script. There is therefore no line, branch or function coverage number for this module, and none can be computed from the repository without adding tooling (which is out of scope for this analysis). The table records which code paths are demonstrably executed, not measured percentages.

What is demonstrable: `tests/helpers/factories.ts:62-74` inserts products directly via `prisma.product.create`, so the `products` table is exercised as a dependency of the Orders tests, and `tests/setup.ts:13` deletes rows before each test. `tests/orders.test.ts:85-86,130-131,150-151,159-160` reads `stockQuantity` to assert order-driven decrement/replenish, which indirectly validates that the column is the shared stock counter. Nothing exercises the Products Module's HTTP surface, Zod contracts, service rules or repository filters.

### Tested behaviors (with evidence)

| Behavior | Test | Location |
|----------|------|----------|
| Products can be inserted with price and stock fixtures | `creates an order...` and every order test using `createTestProduct` | `tests/helpers/factories.ts:62-74`; `tests/orders.test.ts:16-17` |
| Product price is used as `unitPriceCents` at order creation | `creates an order in PENDING status and computes totals on the server` | `tests/orders.test.ts:33-35` |
| Missing product id during order creation returns 404 | `rejects an order with non-existent product` | `tests/orders.test.ts:42-57` |
| Payment decrements `products.stockQuantity` | `transitions PENDING -> PAID and decrements stock` | `tests/orders.test.ts:85-86` |
| Insufficient stock blocks payment and leaves stock untouched | `returns 422 when transitioning PENDING -> PAID without enough stock` | `tests/orders.test.ts:109-132` |
| Cancellation replenishes `products.stockQuantity` | `replenishes stock when going PAID -> CANCELLED` | `tests/orders.test.ts:134-161` |
| Products are truncated between tests | Global `beforeEach` | `tests/setup.ts:8-16` |

Note: none of these tests asserts product API behavior. They assert order behavior whose side effects are visible on product rows.

### Uncovered behaviors and branches

| Gap | Risk | Relevant code |
|-----|------|---------------|
| `GET /api/v1/products` list, pagination, search, active filter | Entire read surface untested | `src/modules/products/product.routes.ts:16`; `product.service.ts:14-22`; `product.repository.ts:13-37` |
| `GET /api/v1/products/:id` success and 404 | Read-by-id untested | `src/modules/products/product.service.ts:24-28` |
| `POST /api/v1/products` success and 201 shape | Creation contract untested | `src/modules/products/product.controller.ts:27-34` |
| Duplicate SKU sequential (`SKU_ALREADY_USED`) and race (`CONFLICT`) | The module's central uniqueness rule is unverified | `src/modules/products/product.service.ts:31-34`; `src/middlewares/error.middleware.ts:37-47` |
| Zod boundaries: 1/2/63/64/65-char SKU, 1/2/199/200/201-char name, 2000/2001-char description | Off-by-one and rejection behavior unverified | `src/modules/products/product.schemas.ts:5-12` |
| Negative, float, string and oversized `priceCents` | Numeric validation and overflow behavior unverified | `src/modules/products/product.schemas.ts:9` |
| Negative `stockQuantity`; default 0 on create | Defaults and bounds unverified | `src/modules/products/product.schemas.ts:10` |
| Default `active=true` and explicit `active=false` | Default behavior unverified | `src/modules/products/product.schemas.ts:11` |
| `PATCH` partial semantics: omitted fields unchanged, `{}` accepted, defaults not reapplied | The most subtle service rule is unverified | `src/modules/products/product.schemas.ts:14`; `product.service.ts:53-60` |
| `PATCH` SKU self-exclusion (same SKU accepted) | Conflict logic unverified | `src/modules/products/product.service.ts:47-52` |
| `description: null` rejection on update | Nullability asymmetry unverified | `src/modules/products/product.schemas.ts:8,14` |
| `DELETE` success 204 | Delete contract unverified | `src/modules/products/product.controller.ts:45-52` |
| `DELETE` referenced product -> 500 from `P2003` | The highest-impact delete failure mode is unverified | `src/middlewares/error.middleware.ts:37-54`; `prisma/migrations/20260519182739_init/migration.sql:119` |
| Missing/invalid/expired token on product routes | Authentication gate on this router unverified | `src/middlewares/auth.middleware.ts:27-47` |
| PAGE bounds (`page=0`, `pageSize=0`, `pageSize=101`, `page=abc`) | Validation boundaries unverified | `src/modules/products/product.schemas.ts:17-18` |
| `active=TRUE`, `active=1`, repeated `active` parameters | Strict filter parsing unverified | `src/modules/products/product.schemas.ts:20-23` |
| `search` whitespace-only rejection and special characters (`%`, `_`) | Trim and LIKE semantics unverified | `src/modules/products/product.schemas.ts:19`; `product.repository.ts:16-21` |
| Unknown body fields stripped | Mass-assignment protection unverified | `src/middlewares/validate.middleware.ts:14-16` |
| Non-UUID path id -> 400 | Parameter validation unverified | `src/modules/products/product.schemas.ts:3` |
| Unsupported methods -> 404 (not 405) | Route topology unverified | `src/app.ts:69-71` |
| Empty pagination result (`total=0`, `totalPages=0`) | Envelope edge case unverified | `src/shared/http/response.ts:13-19` |
| Collation-dependent SKU and search behavior | Environment-dependent semantics unverified | `prisma/migrations/20260519182739_init/migration.sql:29,47` |
| Concurrent stock update versus order payment | The most serious consistency risk has no test | `src/modules/products/product.service.ts:58`; `src/modules/orders/order.service.ts:204-243` |

### Test infrastructure observations

- Tests are integration-only and require a reachable MySQL with migrations applied; `vitest.config.ts` sets node environment, 30 s test and hook timeouts, serial forks and a single fork to avoid cross-test interference.
- `tests/setup.ts:8-16` truncates `orderStatusHistory`, `orderItem`, `order`, `orderNumberSequence`, `product`, `customer` and `user` before every test, with no check that `DATABASE_URL` points to a disposable database. A misconfigured environment would destroy real catalog data.
- `tests/helpers/factories.ts:62-74` creates products directly through Prisma, bypassing `ProductService.create`. This is why product validation, defaults and SKU uniqueness have no coverage even though many tests contain products.
- `tests/helpers/factories.ts:8-15` caches a single Express app built with the real Prisma client. There are no mocks, no in-memory repository and no dependency substitution seam, because the module depends on a concrete class.
- There is no fixture that creates an `OrderItem` referencing a product outside the order flow, so the delete-restriction path can only be reached by first creating an order.
- No test asserts product response shape, status codes, `pagination` fields or error codes; the only product-related assertions are on `stockQuantity` and `sku` inside order responses.

---

## 12. Assumptions, Ambiguities and Evidence Quality

| Topic | Status | Basis |
|-------|--------|-------|
| Who is allowed to manage the catalog | Unknown | No written requirements; code applies authentication only, so the implemented rule is "any authenticated user" |
| Intended delete semantics (hard vs soft) | Ambiguous | Hard delete implemented; `active` flag exists but is not enforced as a delete replacement |
| Intended SKU format and immutability | Unknown | Only length is validated; SKU is mutable; no convention or immutability rule in code |
| Intended SKU case sensitivity | Ambiguous | No normalization in code; MySQL `utf8mb4_unicode_ci` makes uniqueness case-insensitive; production collation is not asserted anywhere |
| Intended currency and price precision | Unknown | Column and field are integer cents; no currency code or rounding policy |
| Whether zero-price products are valid | Unknown | `nonnegative()` permits 0; no business rule states a minimum |
| Whether `description` should be clearable | Unknown | Database allows NULL; update schema does not accept null |
| Whether `PATCH {}` should be rejected | Unknown | Accepted and produces a no-op write with a new `updatedAt` |
| Whether stock should be writable via the catalog API | Ambiguous | Implemented as absolute write; Orders mutates the same field; no requirement resolves ownership |
| Intended behavior when deleting a referenced product | Ambiguous | FK RESTRICT blocks it; API returns 500 with no documented business code |
| Whether inactive products should block already-created orders | Unknown | Only order creation checks `active`; transitions do not |
| Sorting and filter requirements | Unknown | `createdAt DESC` is hardcoded; no sort contract |
| Coverage percentages | Unavailable | No coverage provider or script configured |
| Test environment expectations | Known constraint | Tests require external `DATABASE_URL`/`JWT_SECRET` and a live MySQL; `npm test` does not load `.env` |
| Zod `.partial()` default behavior | Verified from library semantics | `.partial()` wraps fields in `ZodOptional`; optional returns `undefined` before inner defaults, so create defaults are not reapplied on update |
| Effective LIKE wildcard escaping | Ambiguous | Delegated to Prisma/MySQL; the component performs no escaping and has no test for `%` or `_` |
| Severity ratings | Analyst judgment | Based on exploitability, blast radius and consistency with the architectural risk register |

### Evidence quality notes

- Every business rule above is traceable to a file and line in the current working tree. Rules marked "High" confidence are directly observable in code or schema; rules about intended behavior are explicitly marked unknown or ambiguous.
- The architectural report (`docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md`) was used only as corroborating context for the risk register and coupling methodology; all product-specific claims were re-verified against source.
- No file in `src/`, `prisma/`, `tests/` or configuration was modified. The only file created is this report.

---

## 13. Report Metadata

- **Component analyzed:** Products Module (`src/modules/products/`)
- **Files read for this report:** `src/modules/products/product.routes.ts`, `src/modules/products/product.controller.ts`, `src/modules/products/product.service.ts`, `src/modules/products/product.repository.ts`, `src/modules/products/product.schemas.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/config/env.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts`, `src/shared/http/response.ts`, `src/routes/index.ts`, `src/app.ts`, `src/modules/orders/order.service.ts`, `src/modules/orders/order.repository.ts`, `src/modules/customers/customer.service.ts` (layering comparison), `src/modules/auth/auth.schemas.ts` (trust-model context), `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql`, `prisma/seed.ts`, `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `tests/setup.ts`, `vitest.config.ts`, `package.json`, `tsconfig.json`, `README.md`, `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, `docs/agents/MANIFEST.md`, and the architectural report under `docs/agents/architectural-analyzer/`.
- **Excluded from analysis:** `node_modules`, `.git`, `dist`, `coverage`.
- **No project files were modified.** This report is analysis-only.
