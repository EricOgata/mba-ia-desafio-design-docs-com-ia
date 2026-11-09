# Component Deep Analysis Report - Orders Module

**Component:** Orders Module
**Primary location:** `src/modules/orders/`
**Files analyzed in depth:** `order.routes.ts`, `order.controller.ts`, `order.schemas.ts`; `order.service.ts`, `order.repository.ts` and `order.status.ts` analyzed at module-behavior level only (sibling analyses cover their internals)
**Directly coupled contracts inspected:** `src/app.ts`, `src/routes/index.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/shared/errors/`, `src/shared/http/response.ts`, `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql`, `prisma/seed.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `tests/setup.ts`, `vitest.config.ts`, `package.json`
**Excluded paths:** `node_modules`, `.git`, `dist`, `coverage`
**Analysis date:** 2026-11-08
**Report confidence:** High for implemented behavior (all claims traced to source). Low for intended behavior: `docs/PRD.md`, `docs/FDD.md`, `docs/RFC.md` and `docs/TRACKER.md` are 3-line empty templates, so no written order requirements exist to compare against.

---

## 1. Executive Summary

The Orders Module is the transactional core of the Order Management System (OMS) REST API. It owns the full order aggregate: creation from a customer and a set of products, server-side price and total calculation, a six-state lifecycle (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`), stock debit and replenishment side effects, a human-readable order number, and an append-only status audit trail. It is mounted at `/api/v1/orders` by `src/routes/index.ts:28` and receives its controller through the composition root in `src/app.ts:42-44`.

The module follows the project's five-file layered convention: `order.routes.ts` (HTTP binding plus middleware composition), `order.controller.ts` (transport adapter), `order.schemas.ts` (Zod request contracts), `order.service.ts` (business rules and transaction orchestration) and `order.repository.ts` (read/delete persistence). A sixth file, `order.status.ts`, isolates the transition table and the two stock side-effect predicates. Persistence is Prisma over MySQL 8.0 (`prisma/schema.prisma:5-9`); there is no interface or port between the domain layer and Prisma.

The module exposes five operations, all behind JWT authentication and all available to every authenticated role. Its critical write paths (`create`, `changeStatus`) run inside Prisma interactive transactions that cover order rows, order items, status history, product stock and the order-number sequence. Read paths are split: the list operation returns bare order rows with a pagination envelope, while the get-by-id operation eager-loads items with a product summary, full history and a customer summary.

### Key findings

| Severity | Finding | Evidence |
|----------|---------|----------|
| High | Stock is neither checked nor reserved when an order is created. Stock is verified only at the `PENDING -> PAID` transition, so accepted orders can be unfulfillable and oversell exposure is deferred to payment time. | `src/modules/orders/order.service.ts:50-124` (no `stockQuantity` read); `tests/orders.test.ts:109-119` creates quantity 5 against stock 1 and expects 201 |
| High | Status transitions have no row lock, version column or conditional update. Two concurrent `PATCH /:id/status` requests can both read the same `from` status, both pass the transition check and both apply stock and history effects. | `src/modules/orders/order.service.ts:132-167`; no `FOR UPDATE`, no `version` field in `prisma/schema.prisma:74-97` |
| High | Order creation is not idempotent: no idempotency key, no client reference, and no uniqueness constraint other than the server-generated order number. A retried `POST` creates a second order and consumes a second sequence value. | `src/modules/orders/order.routes.ts:18`; `src/modules/orders/order.service.ts:95-120`; `prisma/schema.prisma:76` |
| Medium | `OrderService` reads and writes `customers`, `products` and `order_number_sequence` directly through Prisma, bypassing `CustomerRepository` and `ProductRepository`. | `src/modules/orders/order.service.ts:59-73,93,204-254` |
| Medium | Every order creation serializes on the single `order_number_sequence` row (`id = 1`), making numbering a contention point and a single point of failure. | `src/modules/orders/order.service.ts:245-254`; `prisma/schema.prisma:133-138` |
| Medium | Order invariants (non-negative totals, discount not exceeding subtotal, legal status transitions) exist only in application code. The migration defines enums, foreign keys, unique indexes and not-null columns, but no `CHECK` constraints. | `prisma/migrations/20260519182739_init/migration.sql:44-67`; `src/modules/orders/order.service.ts:87-92,140-149` |
| Medium | All five endpoints accept any authenticated role (`ADMIN` or `OPERATOR`). There is no `requireRole` usage, so an `OPERATOR` can delete orders and drive payment/shipping transitions. | `src/modules/orders/order.routes.ts:14`; contrast `src/modules/users/user.routes.ts:12-18` |
| Medium | Integration test coverage is concentrated on happy paths and three failure paths; `GET /:id`, successful `DELETE`, inactive products, discount overflow, duplicate item aggregation, same-status conflicts and several transitions are untested. No unit tests exist for `order.status.ts`. | `tests/orders.test.ts` (8 tests); no other order test file exists |
| Low | The list endpoint returns only flat `Order` rows, while get-by-id returns the full aggregate; the two read shapes are inconsistent and undocumented. | `src/modules/orders/order.repository.ts:33-45` vs `47-60` |
| Low | Deletion is a hard delete. `order_items` and `order_status_history` rows are removed by cascade, so the audit trail of a deleted order is destroyed, and the deletion actor is not recorded anywhere. | `prisma/schema.prisma:108,125`; `src/modules/orders/order.service.ts:181-192` |

### Role in the system

- Inbound: `/api/v1` (`src/app.ts:67`) to `/orders` (`src/routes/index.ts:28`) to `buildOrderRouter` (`src/modules/orders/order.routes.ts:12`).
- Outbound: Prisma/MySQL for `orders`, `order_items`, `order_status_history`, `order_number_sequence`, plus direct reads/writes of `customers` and `products`.
- Lateral: depends on `authenticate` and `validate` middleware, the shared `AppError` taxonomy and the pagination helper. It is consumed by no other business module at runtime; only the composition root and router registry import it.
- Data ownership: owns the order aggregate and its audit history; does not own customer or product data, but mutates product stock as a lifecycle side effect.

### Scope assumptions

- The module boundary is `src/modules/orders/`. Service, repository and state-machine internals are described only to the depth required to explain module-level route, controller and cross-file business behavior.
- No written requirements exist for orders. All rules below are reverse-engineered from code; where a rule exists only implicitly (for example, that the caller cannot supply prices), this is stated.
- Coverage percentages are not reported because no coverage provider is configured in `vitest.config.ts` or `package.json`.

---

## 2. Data Flow Analysis

### 2.1 Order creation (`POST /api/v1/orders`)

```
1. Express parses JSON with a 1 MB limit in src/app.ts:59
2. requestLogger assigns or propagates X-Request-Id in src/middlewares/request-logger.middleware.ts:5-8
3. API router dispatches /api/v1/orders to buildOrderRouter in src/routes/index.ts:28
4. router.use(authenticate) validates the Bearer JWT and sets req.user in src/modules/orders/order.routes.ts:14
   -> missing/invalid header or token: 401 UNAUTHORIZED, controller never invoked
5. validate({ body: createOrderSchema }) parses and replaces req.body in src/modules/orders/order.routes.ts:18
   -> customerId UUID, items[] min 1, each { productId UUID, quantity positive int },
      discountCents non-negative int default 0, notes <= 1000, unknown keys stripped
   -> failure: 400 VALIDATION_ERROR with per-path details, controller never invoked
6. OrderController.create guards req.user, then calls orders.create(req.body, req.user.id) in src/modules/orders/order.controller.ts:28-36
7. OrderService.create applies a second empty-items guard in src/modules/orders/order.service.ts:51-53
8. Duplicate productIds are aggregated by summing quantities in src/modules/orders/order.service.ts:55,194-202
9. BEGIN Prisma interactive transaction in src/modules/orders/order.service.ts:58
   a. SELECT customer by id; missing -> 404 NOT_FOUND ("Customer not found")
   b. SELECT products WHERE id IN (aggregated ids); count mismatch -> 404 NOT_FOUND ("Product not found")
   c. Any inactive product -> 422 INACTIVE_PRODUCT with { skus: [...] }
   d. Compute unitPriceCents from product.priceCents, line total, subtotal, total (server-authoritative)
   e. discountCents > subtotalCents -> 400 VALIDATION_ERROR with path "discountCents"
   f. UPSERT order_number_sequence id=1, increment nextValue, derive "ORD-000001" style number
   g. INSERT order (status PENDING, createdById = JWT user) with nested items and one history row (null -> PENDING)
10. COMMIT
11. Controller responds 201 with the order including items+product summary, history ascending, customer summary
12. Any thrown AppError -> next(err) -> errorMiddleware envelope; unknown error -> logged 500
```

### 2.2 Status change (`PATCH /api/v1/orders/:id/status`)

```
1. authenticate -> req.user (src/modules/orders/order.routes.ts:14)
2. validate({ params: orderIdParamSchema, body: updateOrderStatusSchema }) in src/modules/orders/order.routes.ts:19-23
   -> id must be UUID-shaped; toStatus must be an OrderStatus enum; reason optional <= 500
3. OrderController.changeStatus guards req.user and calls orders.changeStatus(id, body, req.user.id) in src/modules/orders/order.controller.ts:38-46
4. BEGIN Prisma interactive transaction in src/modules/orders/order.service.ts:131
   a. SELECT order with items; missing -> 404 NOT_FOUND ("Order not found")
   b. from === to -> 409 INVALID_STATUS_TRANSITION ("Order is already in <status> status")
   c. canTransition(from, to) false -> 409 INVALID_STATUS_TRANSITION with { from, to }
   d. If PENDING -> PAID:
        SELECT all products for the order items
        collect every item whose product is missing or stockQuantity < quantity
        any collected -> 422 INSUFFICIENT_STOCK with { unavailable: [{sku, requested, available}] }
        otherwise decrement each product stockQuantity (atomic per-row decrement)
   e. If (PAID or PROCESSING) -> CANCELLED: increment each product stockQuantity
   f. UPDATE order.status
   g. INSERT order_status_history { orderId, fromStatus, toStatus, changedById = JWT user, reason }
   h. Re-SELECT order with items+product summary, history ascending, customer summary
5. COMMIT -> 200 with refreshed order
6. Any error rolls back stock changes, status update and history insert
```

### 2.3 Order listing (`GET /api/v1/orders`)

```
1. authenticate -> req.user (response never filters by user or role)
2. validate({ query: listOrdersQuerySchema }) coerces page, pageSize, from, to and merges parsed values into req.query
3. OrderController.list casts req.query to ListOrdersQuery and calls orders.list(query) in src/modules/orders/order.controller.ts:9-17
4. OrderService.list computes skip = (page - 1) * pageSize and take = pageSize in src/modules/orders/order.service.ts:32-42
5. OrderRepository.list builds a where clause from status, customerId and createdAt range in src/modules/orders/order.repository.ts:21-31
6. Prisma $transaction([findMany(orderBy createdAt desc, skip, take), count(where)]) returns items and total
7. paginated(items, page, pageSize, total) builds { data, pagination: { page, pageSize, total, totalPages } }
8. 200 with the envelope; no relations are loaded
```

### 2.4 Order retrieval (`GET /api/v1/orders/:id`)

```
1. authenticate -> req.user
2. validate({ params: orderIdParamSchema }); non-UUID-shaped id -> 400 VALIDATION_ERROR
3. OrderController.getById calls orders.getById(req.params.id) in src/modules/orders/order.controller.ts:19-26
4. OrderRepository.findByIdWithRelations includes:
   - items with product { id, sku, name }
   - history ordered by changedAt asc
   - customer { id, name, email }
5. Missing row -> 404 NOT_FOUND ("Order not found"); found -> 200 full aggregate
```

### 2.5 Order deletion (`DELETE /api/v1/orders/:id`)

```
1. authenticate -> req.user
2. validate({ params: orderIdParamSchema })
3. OrderController.delete calls orders.delete(id) in src/modules/orders/order.controller.ts:48-55
4. OrderRepository.findById loads the order; missing -> 404 NOT_FOUND
5. status not in {PENDING, CANCELLED} -> 409 INVALID_ORDER_STATE_FOR_DELETE with { status }
6. OrderRepository.deleteById executes prisma.order.delete
   - MySQL cascades order_items and order_status_history
   - products are not touched and stock is not reconciled
7. 204 with empty body
8. Race between findById and delete (row removed concurrently) surfaces as Prisma P2025 -> 404 NOT_FOUND from errorMiddleware
```

### 2.6 Error propagation path

```
Controller catch -> next(err)
  -> errorMiddleware
     - err instanceof AppError -> statusCode + { error: { code, message, details? } }
     - ZodError (defensive; normal path is validate middleware) -> 400 VALIDATION_ERROR
     - Prisma P2002 -> 409 CONFLICT ("Unique constraint violation on: <target>")
     - Prisma P2025 -> 404 NOT_FOUND ("Resource not found")
     - anything else -> logger.error({ err, requestId, method, path }) -> 500 INTERNAL_SERVER_ERROR
```

### 2.7 Trust boundaries crossed

| Boundary | Crossing point | Validation applied |
|----------|----------------|--------------------|
| Network -> application | Express JSON parser and route handlers | Zod body/query/params schemas; 1 MB body limit (`src/app.ts:59`) |
| Bearer credential -> request identity | `authenticate` | JWT signature and expiry against `JWT_SECRET`; payload cast to `AuthUser` without schema validation (`src/middlewares/auth.middleware.ts:19-46`) |
| Orders Module -> orders tables | Prisma | Parameterized SQL; enum, foreign key, unique and not-null constraints in MySQL |
| Orders Module -> customers/products tables | Direct Prisma in `OrderService` | Existence checks in code; no repository contract; no referential check beyond FK |
| Orders Module -> stock | `product.update` decrement/increment | Application-level quantity check only on `PENDING -> PAID`; no DB check that stock remains non-negative |
| Orders Module -> HTTP error contract | `AppError` subclasses | Typed status code and error code; details conditionally serialized |

---

## 3. Business Rules & Logic

### Overview of the business rules

| ID | Rule Type | Rule Description | Location |
|----|-----------|------------------|----------|
| BR-01 | API surface | The module is mounted at `/api/v1/orders` and exposes five operations: list, get by id, create, change status, delete | `src/routes/index.ts:28`; `src/modules/orders/order.routes.ts:16-24` |
| BR-02 | Authentication | Every order route requires a valid `Authorization: Bearer <JWT>` header; missing or invalid credentials return 401 before validation or controller logic | `src/modules/orders/order.routes.ts:14`; `src/middlewares/auth.middleware.ts:27-47` |
| BR-03 | Authorization | Any authenticated role (`ADMIN` or `OPERATOR`) may perform every order operation; no role check or ownership check exists | `src/modules/orders/order.routes.ts:14`; no `requireRole` import |
| BR-04 | Validation | Path `id` must be a UUID-shaped string on get, status change and delete | `src/modules/orders/order.schemas.ts:4` |
| BR-05 | Validation | Create body requires a UUID `customerId`, a non-empty `items` array, a non-negative integer `discountCents` defaulting to 0, and optional `notes` of at most 1000 characters | `src/modules/orders/order.schemas.ts:11-16` |
| BR-06 | Validation | Each order item requires a UUID `productId` and a strictly positive integer `quantity` | `src/modules/orders/order.schemas.ts:6-9` |
| BR-07 | Validation | Status change body requires `toStatus` in the `OrderStatus` enum and an optional `reason` of at most 500 characters | `src/modules/orders/order.schemas.ts:18-21` |
| BR-08 | Validation | List query accepts optional `status`, `customerId` (UUID), `from` and `to` dates, with `page` default 1 and `pageSize` default 20 capped at 100 | `src/modules/orders/order.schemas.ts:23-30` |
| BR-09 | API contract | Controllers are thin adapters returning 200 for list/get/status, 201 for create, 204 for delete, and forwarding all errors to the centralized middleware | `src/modules/orders/order.controller.ts:9-55` |
| BR-10 | Business logic | An order must contain at least one item; the rule is enforced in the schema and re-checked in the service | `src/modules/orders/order.schemas.ts:13`; `src/modules/orders/order.service.ts:51-53` |
| BR-11 | Business logic | Repeated `productId` entries in the request are merged into one order line with quantities summed | `src/modules/orders/order.service.ts:55,194-202` |
| BR-12 | Referential integrity | The referenced customer must exist; otherwise creation fails with 404 | `src/modules/orders/order.service.ts:59-60` |
| BR-13 | Referential integrity | Every referenced product must exist; otherwise creation fails with 404 | `src/modules/orders/order.service.ts:62-65` |
| BR-14 | Business logic | Only active products may be ordered; any inactive product fails the whole order with 422 `INACTIVE_PRODUCT` and the offending SKUs | `src/modules/orders/order.service.ts:66-73` |
| BR-15 | Pricing | Prices are server-authoritative: `unitPriceCents` is copied from `Product.priceCents`, line totals and subtotal are computed, and no client-supplied price is accepted | `src/modules/orders/order.service.ts:75-86`; `src/modules/orders/order.schemas.ts:6-16` |
| BR-16 | Pricing | `discountCents` cannot exceed the computed subtotal; equal to subtotal is allowed (zero total) | `src/modules/orders/order.service.ts:87-91` |
| BR-17 | Pricing | `totalCents = subtotalCents - discountCents` and is persisted with the order | `src/modules/orders/order.service.ts:92,100-102` |
| BR-18 | Lifecycle | New orders always start in `PENDING`; the request contract has no status field and the service hardcodes the status | `src/modules/orders/order.service.ts:99`; `prisma/schema.prisma:78` |
| BR-19 | Numbering | `orderNumber` is server-generated from a singleton sequence row and formatted `ORD-` plus a zero-padded 6-digit value | `src/modules/orders/order.service.ts:93,245-254` |
| BR-20 | Attribution | `createdById` is taken from the authenticated JWT subject, never from the request body | `src/modules/orders/order.controller.ts:30-31`; `src/modules/orders/order.service.ts:104` |
| BR-21 | Audit | Creation writes an initial history row `null -> PENDING` with reason `order created` and the creating user as actor | `src/modules/orders/order.service.ts:106-113` |
| BR-22 | Consistency | Order, items, initial history and sequence increment are committed in one interactive transaction | `src/modules/orders/order.service.ts:58-123` |
| BR-23 | Inventory | Creation does not read, validate, reserve or debit stock; stock is only affected at lifecycle transitions | `src/modules/orders/order.service.ts:50-124`; `tests/orders.test.ts:109-119` |
| BR-24 | Pagination | List computes `skip = (page - 1) * pageSize` and `take = pageSize`, with defaults page 1 and pageSize 20, maximum 100 | `src/modules/orders/order.service.ts:38-39`; `src/modules/orders/order.schemas.ts:24-25` |
| BR-25 | Query | List filters by exact `status`, exact `customerId` and inclusive `createdAt` range (`from` uses gte, `to` uses lte), sorted `createdAt` descending | `src/modules/orders/order.repository.ts:21-45` |
| BR-26 | API contract | List returns `{ data, pagination: { page, pageSize, total, totalPages } }` with `totalPages = ceil(total / pageSize)` | `src/shared/http/response.ts:13-24`; `src/modules/orders/order.service.ts:41` |
| BR-27 | API contract | Get by id returns the order with item product summaries, history ascending and customer summary; missing order returns 404 | `src/modules/orders/order.repository.ts:47-60`; `src/modules/orders/order.service.ts:44-48` |
| BR-28 | State machine | Only the whitelisted transitions are legal: PENDING to PAID or CANCELLED, PAID to PROCESSING or CANCELLED, PROCESSING to SHIPPED or CANCELLED, SHIPPED to DELIVERED; DELIVERED and CANCELLED are terminal | `src/modules/orders/order.status.ts:3-14` |
| BR-29 | State machine | A transition from a status to itself is rejected with 409 `INVALID_STATUS_TRANSITION` before the whitelist is evaluated | `src/modules/orders/order.service.ts:140-146` |
| BR-30 | State machine | Any non-whitelisted transition is rejected with 409 `INVALID_STATUS_TRANSITION` and `{ from, to }` details | `src/modules/orders/order.service.ts:147-149`; `src/shared/errors/http-errors.ts:45-53` |
| BR-31 | Inventory | `PENDING -> PAID` validates stock for every line first; if any line lacks stock the whole transition fails with 422 `INSUFFICIENT_STOCK` and no stock is changed | `src/modules/orders/order.service.ts:151-153,204-231` |
| BR-32 | Inventory | `PAID -> CANCELLED` and `PROCESSING -> CANCELLED` return each line quantity to stock | `src/modules/orders/order.status.ts:33-37`; `src/modules/orders/order.service.ts:154-156,233-243` |
| BR-33 | Inventory | No other transition changes stock: `PENDING -> CANCELLED` never debited, `PAID -> PROCESSING`, `PROCESSING -> SHIPPED` and `SHIPPED -> DELIVERED` move no stock | `src/modules/orders/order.status.ts:24-37`; `src/modules/orders/order.service.ts:151-156` |
| BR-34 | Audit | Every successful status change appends a history row with `fromStatus`, `toStatus`, actor from the JWT and optional reason; history is never updated or deleted through the API | `src/modules/orders/order.service.ts:159-167`; `src/modules/orders/order.schemas.ts:20` |
| BR-35 | Consistency | Stock changes, status update and history insert for a transition commit atomically in one transaction | `src/modules/orders/order.service.ts:131-178` |
| BR-36 | Deletion | An order may be deleted only while `PENDING` or `CANCELLED`; any other status returns 409 `INVALID_ORDER_STATE_FOR_DELETE` with the current status | `src/modules/orders/order.service.ts:181-192` |
| BR-37 | Deletion | Deletion is hard, returns 204, cascades to items and history, does not touch product stock and records no deletion audit | `src/modules/orders/order.repository.ts:66-68`; `prisma/schema.prisma:108,125` |
| BR-38 | Error contract | Domain failures map to a uniform envelope with codes `VALIDATION_ERROR` 400, `UNAUTHORIZED` 401, `NOT_FOUND` 404, `INVALID_STATUS_TRANSITION`/`INVALID_ORDER_STATE_FOR_DELETE`/`CONFLICT` 409, `INACTIVE_PRODUCT`/`INSUFFICIENT_STOCK` 422, and `INTERNAL_SERVER_ERROR` 500 | `src/middlewares/error.middleware.ts:14-64`; `src/shared/errors/http-errors.ts:9-63` |
| BR-39 | Concurrency | Order state and stock use read-then-write without row locks, version columns or conditional updates; interactive transactions use Prisma defaults (5 s timeout, 2 s max wait) | `src/modules/orders/order.service.ts:132-167,204-231`; `prisma/schema.prisma:74-114` |
| BR-40 | Query semantics | Date filters are inclusive (`gte`/`lte`), parsed with `z.coerce.date()`, with no timezone strategy and no check that `from <= to` | `src/modules/orders/order.schemas.ts:28-29`; `src/modules/orders/order.repository.ts:25-29` |
| BR-41 | Data constraints | Financial and lifecycle invariants are enforced only in application code; MySQL has no `CHECK` constraints, and all money/quantity columns are 32-bit signed integers | `prisma/migrations/20260519182739_init/migration.sql:44-67,69-85` |
| BR-42 | Architecture | The module accesses `customers`, `products` and `order_number_sequence` directly through `Prisma.TransactionClient`, not through their owning repositories | `src/modules/orders/order.service.ts:59-73,93,204-254` |

---

### Detailed breakdown of the business rules

---

### Business Rule BR-01: Fixed route surface under `/api/v1/orders`

**Overview**:
The module publishes exactly five HTTP operations beneath a single mount point: list, retrieve by id, create, change status and delete.

**Detailed description**:
`buildOrderRouter` registers `GET /`, `GET /:id`, `POST /`, `PATCH /:id/status` and `DELETE /:id` (`src/modules/orders/order.routes.ts:16-24`). The router is mounted by the API registry at `/orders` (`src/routes/index.ts:28`), and the registry is mounted at `/api/v1` (`src/app.ts:67`), so the effective paths are `/api/v1/orders` and its subpaths. There is no nested resource, no bulk operation, no search-by-order-number endpoint and no endpoint that exposes allowed transitions or stock availability.

The shape is uniform CRUD-plus-command: status changes are modelled as a sub-resource command (`PATCH /:id/status`) rather than a general order update. No `PUT /:id` or `PATCH /:id` exists, which means customer, notes, items, discount and totals are immutable after creation. That immutability is not stated in a requirements document; it is a consequence of the route table and the absence of any update method in the controller or service.

Because `PATCH /:id/status` is separate from order updates, the only mutable order field exposed by the API is `status`. Items, prices and discounts can only change by deleting and recreating an order, which is itself restricted by BR-36. This is the module's effective mutation contract.

**Rule workflow**:
```
/api/v1 (app.ts:67)
  -> /orders (routes/index.ts:28)
     -> GET /            list
     -> GET /:id         getById
     -> POST /           create
     -> PATCH /:id/status changeStatus
     -> DELETE /:id      delete
```

---

### Business Rule BR-02: Authentication on every order endpoint

**Overview**:
No order operation is reachable without a valid bearer token.

**Detailed description**:
`router.use(authenticate)` is the first registration in `buildOrderRouter` (`src/modules/orders/order.routes.ts:14`), so Express applies it to every subsequent route in that router. `authenticate` requires an `Authorization` header beginning with `Bearer ` (case-insensitive), extracts the token, verifies its signature and expiry with `JWT_SECRET`, and populates `req.user` with `{ id: payload.sub, email, role }` (`src/middlewares/auth.middleware.ts:27-47`). Failure produces `UnauthorizedError` with a 401 and the request never reaches Zod validation or the controller.

The middleware distinguishes three failure messages (missing or malformed header, empty bearer token, invalid or expired token) but all map to the same 401 `UNAUTHORIZED` code. No cookie, API key or mTLS path exists. Token claims are cast to the `AuthUser` type without runtime schema validation, so `req.user.role` is trusted as delivered by the token.

For creation and status change, the controller adds a defensive `if (!req.user) throw new UnauthorizedError()` (`src/modules/orders/order.controller.ts:30,40`). That guard is unreachable through the router because authentication runs first; it only matters for direct controller invocation. List, get and delete have no such guard and rely entirely on the middleware.

**Rule workflow**:
```
Any /api/v1/orders request
  -> authenticate
     -> no header / not "Bearer ..." -> 401 UNAUTHORIZED
     -> empty token                   -> 401 UNAUTHORIZED
     -> bad signature / expired       -> 401 UNAUTHORIZED
     -> valid                         -> req.user set -> route validation -> controller
```

---

### Business Rule BR-03: Role-agnostic authorization

**Overview**:
Authentication is required, but authorization is flat: `ADMIN` and `OPERATOR` have identical order capabilities.

**Detailed description**:
The orders router imports only `authenticate`; it never imports `requireRole` (`src/modules/orders/order.routes.ts:2`). `requireRole` exists in the authentication middleware (`src/middlewares/auth.middleware.ts:49-60`) and is used by the users router (`src/modules/users/user.routes.ts:12-18`), but not here. Consequently an `OPERATOR` token can create orders, read any order, drive `PENDING -> PAID`, and delete orders in a deletable state.

There is also no ownership or tenant scoping. The list operation does not filter by `createdById`, and get/change/delete accept any order id regardless of who created it. A user can therefore act on another user's order. Creation records the actor in `createdById` (`src/modules/orders/order.service.ts:104`) and status changes record the actor in history (`159-167`), so attribution exists, but it is not used for access decisions.

The practical authorization model is "any authenticated internal user" which is consistent with the other business routers (customers, products) but weaker than the users router. Because order deletion and payment confirmation are commercially sensitive, this flat model is the module's principal access-control characteristic.

**Rule workflow**:
```
Bearer token valid -> req.user.role in {ADMIN, OPERATOR}
  -> all order routes allowed (no role gate, no ownership gate)
```

---

### Business Rule BR-04: UUID-shaped path identifiers

**Overview**:
The `:id` parameter on get, status change and delete must look like a UUID; malformed ids are rejected before any database access.

**Detailed description**:
`orderIdParamSchema` is `z.object({ id: z.string().uuid() })` (`src/modules/orders/order.schemas.ts:4`) and is applied by the route table on three endpoints (`src/modules/orders/order.routes.ts:17,21,24`). The validation middleware replaces `req.params` with the parsed object and converts a `ZodError` into `ValidationError` with a path/message list (`src/middlewares/validate.middleware.ts:21-33`), producing 400 `VALIDATION_ERROR`. A non-UUID id therefore never reaches Prisma and never produces a 404.

Zod 3.23.8's `uuid()` check validates hexadecimal shape only (`8-4-4-4-12`) and does not enforce RFC 4122 version or variant bits. A value such as `11111111-1111-1111-1111-111111111111` passes validation, which is why the non-existent-product test can use such an id and still reach the database layer (`tests/orders.test.ts:52`). The rule protects against arbitrary strings and SQL-shaped input but not against all RFC-invalid UUIDs.

The schema is shared across all three param routes, so there is a single id contract. The list route has no path parameter. The repository and service assume a valid UUID and pass it straight to `findUnique`/`findById`; there is no second format check below the transport layer.

**Rule workflow**:
```
:id on GET /:id, PATCH /:id/status, DELETE /:id
  -> orderIdParamSchema.safe parse
     -> not UUID-shaped -> 400 VALIDATION_ERROR (path "id")
     -> UUID-shaped     -> req.params.id forwarded
```

---

### Business Rule BR-05: Order creation request contract

**Overview**:
Order creation accepts exactly four logical inputs: customer, items, discount and notes, with defaults and limits defined by Zod.

**Detailed description**:
`createOrderSchema` requires `customerId: z.string().uuid()`, `items: z.array(orderItemInputSchema).min(1)`, `discountCents: z.number().int().nonnegative().default(0)` and optional `notes: z.string().max(1000)` (`src/modules/orders/order.schemas.ts:11-16`). Because Zod object parsing strips unknown keys by default, a client that sends `status`, `totalCents`, `subtotalCents`, `orderNumber` or `createdById` has those fields silently removed before the controller. This is the mechanism behind BR-15, BR-18, BR-19 and BR-20.

`discountCents` is a required-by-type but defaulted-by-schema field: omission becomes 0. There is no maximum and no currency field; amounts are integer cents. `notes` is capped at 1000 characters even though the database column is `TEXT` (`prisma/schema.prisma:82`), so the cap is purely a transport rule. `customerId` is only format-checked here; existence is checked inside the transaction (BR-12).

Unknown or misspelled keys do not cause errors; they are dropped. Combined with the absence of a general update endpoint, the schema is the complete create contract and the only place where the set of accepted inputs is defined.

**Rule workflow**:
```
POST body
  -> createOrderSchema
     customerId: UUID required
     items: array min 1 (item schema BR-06)
     discountCents: int >= 0, default 0
     notes: string <= 1000, optional
     unknown keys stripped
  -> req.body replaced with parsed output
```

---

### Business Rule BR-06: Order item input contract

**Overview**:
Every item line must reference a UUID product and request a positive whole quantity.

**Detailed description**:
`orderItemInputSchema` declares `productId: z.string().uuid()` and `quantity: z.number().int().positive()` (`src/modules/orders/order.schemas.ts:6-9`). The array wrapper in `createOrderSchema` allows one or more entries, so an empty array is rejected at the transport layer (BR-10). Quantity must be at least 1, may be arbitrarily large, and is not bounded by any stock or business maximum. Because JSON numbers are accepted directly, `quantity: 1.5`, `quantity: 0`, `quantity: -2` and string values all fail validation.

The item contract contains no price field. A client cannot propose `unitPriceCents` or `totalCents`; any such fields are stripped by the schema and ignored by the service, which resolves prices from the product table (BR-15). The product UUID is not validated for existence at this stage; that happens in the creation transaction (BR-13).

Duplicate `productId` entries are not rejected by the schema. They are legal input and are merged later by the service (BR-11), so the contract allows a client to express the same product in multiple lines.

**Rule workflow**:
```
items[] entry
  -> productId: UUID-shaped string
  -> quantity: integer > 0
     -> any violation: 400 VALIDATION_ERROR with path "items.<index>.<field>"
```

---

### Business Rule BR-07: Status change request contract

**Overview**:
A status change carries the target status and an optional free-text reason; the source status is derived from stored state.

**Detailed description**:
`updateOrderStatusSchema` requires `toStatus: z.nativeEnum(OrderStatus)` and optional `reason: z.string().max(500)` (`src/modules/orders/order.schemas.ts:18-21`). The enum is generated from Prisma and contains exactly the six lifecycle values (`prisma/schema.prisma:16-23`). Any other string is rejected with 400 `VALIDATION_ERROR` before the controller. The client cannot specify `fromStatus`; it is read from the order row inside the transaction (`src/modules/orders/order.service.ts:138`).

The `reason` ceiling of 500 matches `OrderStatusHistory.reason @db.VarChar(500)` (`prisma/schema.prisma:123`), so validation prevents a database truncation error. Reason is optional and stored as `null` when absent (`src/modules/orders/order.service.ts:165`). It is free text with no enumeration, normalization or required-length rule.

The schema allows any target enum value, including the current status and terminal statuses. Legality is decided by the service state machine (BR-28 to BR-30) rather than by the request schema, so a syntactically valid request can still fail with 409.

**Rule workflow**:
```
PATCH /:id/status body
  -> updateOrderStatusSchema
     toStatus: one of PENDING|PAID|PROCESSING|SHIPPED|DELIVERED|CANCELLED
     reason: string <= 500, optional
  -> controller -> service transition checks
```

---

### Business Rule BR-08: List query contract

**Overview**:
The list endpoint accepts pagination, status, customer and date-range filters, all optional except the pagination defaults.

**Detailed description**:
`listOrdersQuerySchema` defines `page` as a coerced integer with minimum 1 and default 1, `pageSize` as a coerced integer between 1 and 100 with default 20, `status` as the optional order enum, `customerId` as an optional UUID, and `from`/`to` as optional coerced dates (`src/modules/orders/order.schemas.ts:23-30`). Because HTTP query values are strings, `z.coerce.number()` and `z.coerce.date()` perform the conversion. `page=0`, `page=-1`, `pageSize=0`, `pageSize=101` and `pageSize=abc` are rejected with 400.

The validation middleware applies the parsed object with `Object.assign(req.query, parsedQuery)` (`src/middlewares/validate.middleware.ts:17-20`). Defaults are therefore written back into `req.query`, and extra unknown query parameters remain present because assignment merges rather than replaces. The controller then casts `req.query` to `ListOrdersQuery` (`src/modules/orders/order.controller.ts:11`), so the static type is asserted rather than inferred.

There is no `search`, `orderNumber`, `createdById` or sorting parameter. Filters are limited to equality on status and customer and a createdAt range. The date contract is described further in BR-40.

**Rule workflow**:
```
GET /api/v1/orders?...
  -> listOrdersQuerySchema
     page: int >= 1 (default 1)
     pageSize: int 1..100 (default 20)
     status: OrderStatus optional
     customerId: UUID optional
     from, to: coerced Date optional
  -> parsed values merged into req.query -> controller
```

---

### Business Rule BR-09: Thin controller response contract

**Overview**:
Controllers perform no domain logic beyond an identity guard; they translate service results into fixed HTTP statuses and delegate all failures.

**Detailed description**:
Each handler is an arrow-function `RequestHandler` with a `try/catch` whose catch block calls `next(err)` (`src/modules/orders/order.controller.ts:9-55`). `list` and `getById` return 200 with the service result; `create` returns 201 with the created aggregate; `changeStatus` returns 200 with the refreshed aggregate; `delete` returns 204 with an empty body. The controller never constructs error responses itself, so all error shape decisions remain in `errorMiddleware`.

The only domain-adjacent checks are the two `if (!req.user) throw new UnauthorizedError()` guards in `create` and `changeStatus` (`30,40`), plus non-null assertions on `req.params.id!` (`21,41,50`). `list` uses a double cast `req.query as unknown as ListOrdersQuery` (`11`), so validation results are trusted but not statically proven. These choices mean the controller cannot be type-checked to guarantee that params and query were validated.

Because the controller holds no transaction, retry or mapping logic, its behavior is entirely determined by the service and the error middleware. It also does not set `Location` headers, pagination headers or cache headers; all metadata is in the JSON body.

**Rule workflow**:
```
handler
  -> optional req.user guard
  -> service call
     -> success: res.status(200|201|204).json(...) or .send()
     -> failure: catch -> next(err) -> errorMiddleware
```

---

### Business Rule BR-10: At least one order item

**Overview**:
An order cannot be created without items.

**Detailed description**:
The rule is expressed twice. `createOrderSchema` uses `.min(1, 'order must contain at least one item')` on the items array (`src/modules/orders/order.schemas.ts:13`), producing a 400 with a field-level message. `OrderService.create` independently checks `input.items.length === 0` and throws `ValidationError('Order must contain at least one item')` (`src/modules/orders/order.service.ts:51-53`), producing the same 400 code without field details.

Through HTTP the service guard is unreachable because the schema rejects the empty array first. The duplicate exists to protect non-HTTP callers and to keep the service invariant local. It is the only duplicated validation between schema and service in the module.

The rule is about line count, not quantity: an array with one item of quantity 1 passes. An array with ten items where nine are duplicates also passes and is aggregated to two lines (BR-11). There is no maximum number of distinct lines.

**Rule workflow**:
```
POST items
  -> schema: array length >= 1
     -> 0 -> 400 VALIDATION_ERROR
  -> service: items.length === 0 -> 400 VALIDATION_ERROR (defensive)
```

---

### Business Rule BR-11: Duplicate product lines are aggregated

**Overview**:
If the same product appears in multiple request lines, the module creates one order item with the summed quantity.

**Detailed description**:
`aggregateItems` iterates the request items and accumulates quantities in a `Map<productId, quantity>` (`src/modules/orders/order.service.ts:194-202`). The map output is used both to fetch products and to build the persisted item rows, so a request with `[{A,1},{B,2},{A,3}]` produces two order items: A with quantity 4 and B with quantity 2. Line totals are computed once on the aggregated quantity, which yields the same subtotal as separate lines because multiplication distributes over addition.

Aggregation occurs before the product existence and active checks, so duplicate references do not cause false 404s or duplicated inactive-product reports. It also means the service's product-id list contains unique values, making the `products.length !== productIds.length` existence test meaningful (`62-65`). Without aggregation that comparison would misclassify requests with duplicates.

The response therefore does not mirror the request line structure. Clients that send three lines for two products receive two items. No back-reference to the original lines is stored, and there is no per-line discount or price override, so nothing is lost by merging. The rule is implicit product-line normalization rather than an explicit documented requirement.

**Rule workflow**:
```
request items
  -> Map accumulation by productId
  -> unique productIds for product lookup
  -> one persisted OrderItem per unique product, quantity = sum
```

---

### Business Rule BR-12: Customer must exist

**Overview**:
Order creation fails with 404 if the customer id does not resolve to a customer row.

**Detailed description**:
Inside the creation transaction the service executes `tx.customer.findUnique({ where: { id: input.customerId } })` and throws `NotFoundError('Customer')` when the result is null (`src/modules/orders/order.service.ts:59-60`). The message rendered to the client is `Customer not found` with code `NOT_FOUND` and status 404 (`src/shared/errors/http-errors.ts:27-31`).

The check is a read-then-insert rather than a database-only constraint. The `orders.customerId` foreign key with `ON DELETE RESTRICT` (`prisma/migrations/20260519182739_init/migration.sql:89`) would also reject an unknown customer at insert time, but that failure would surface as an unmapped Prisma error and therefore a 500. The explicit lookup converts the common case into a domain 404. A customer deleted between the lookup and the insert cannot happen within the same transaction under MySQL locking semantics for the referenced row, but the code does not rely on that explicitly.

The customer is not locked or versioned, and customer fields are not copied into the order. The order stores only `customerId`; the response joins the current customer name and email. A later customer rename therefore changes what get-by-id returns for historical orders.

**Rule workflow**:
```
BEGIN
  SELECT customer WHERE id = customerId
    -> null -> 404 NOT_FOUND
    -> found -> continue
  ...
COMMIT
```

---

### Business Rule BR-13: All referenced products must exist

**Overview**:
Every distinct product referenced by the order must resolve to a product row; a single missing product rejects the whole order with 404.

**Detailed description**:
After aggregation, the service fetches products with `tx.product.findMany({ where: { id: { in: productIds } } })` and compares `products.length` with `productIds.length` (`src/modules/orders/order.service.ts:62-65`). Because ids are unique after aggregation, an inequality means at least one product is missing, and the service throws `NotFoundError('Product')` with message `Product not found` and status 404.

The rule is all-or-nothing: the error does not identify which product is missing, unlike the inactive-product and insufficient-stock errors which include details. A client must compare its request against product existence to discover the culprit. The check is inside the transaction but before any write, so nothing is persisted on failure.

Existence does not imply orderability; the active check (BR-14) follows immediately. A product that exists but is inactive produces 422 rather than 404, so the two failure modes are distinguishable. The `OrderItem.productId` foreign key with `ON DELETE RESTRICT` (`prisma/migrations/20260519182739_init/migration.sql:97`) protects historical items from product deletion; that constraint acts at insert time and is separate from this application check.

**Rule workflow**:
```
productIds = unique ids from aggregated items
  SELECT products WHERE id IN productIds
    -> count mismatch -> 404 NOT_FOUND ("Product not found")
    -> count match    -> active check
```

---

### Business Rule BR-14: Only active products can be ordered

**Overview**:
Any inactive product in the request rejects the entire order with 422 `INACTIVE_PRODUCT` and the list of offending SKUs.

**Detailed description**:
`const inactive = products.filter((p) => !p.active)` runs after the existence check, and a non-empty list throws `UnprocessableEntityError('One or more products are inactive', 'INACTIVE_PRODUCT', { skus })` (`src/modules/orders/order.service.ts:66-73`). The error maps to 422 with code `INACTIVE_PRODUCT` and details containing the SKUs, not the product ids. The check covers all requested products at once, so the client learns every inactive SKU in one response.

The rule is a creation-time gate only. There is no check at status transition time; if a product is deactivated after an order is created, the order can still be paid and shipped. There is also no rule that reactivating a product affects existing orders. `active` is a boolean on `Product` with a default of `true` (`prisma/schema.prisma:63`), and the product module controls it.

The 422 status distinguishes a semantically valid request from a forbidden business state, which is consistent with the module's use of 422 for stock shortage. Inactive products are treated as orderable-domain violations rather than validation errors because the request itself is well-formed.

**Rule workflow**:
```
products loaded
  -> filter !active
     -> non-empty -> 422 INACTIVE_PRODUCT { skus: [...] }
     -> empty     -> proceed to pricing
```

---

### Business Rule BR-15: Server-authoritative pricing and totals

**Overview**:
All monetary values on the order are computed by the server from the current product price; the client cannot supply or influence unit prices.

**Detailed description**:
For each aggregated item the service resolves the product row and sets `unitPriceCents = product.priceCents`, then computes `totalCents = unitPriceCents * quantity` (`src/modules/orders/order.service.ts:75-84`). The subtotal is the sum of line totals (`86`). No request field carries a price: `orderItemInputSchema` has only `productId` and `quantity`, and unknown keys are stripped by Zod (BR-05, BR-06). This closes the obvious tampering path of posting a lower price.

The price is a snapshot. Changing `Product.priceCents` later does not alter existing order items because `OrderItem.unitPriceCents` and `totalCents` are persisted (`prisma/schema.prisma:103-105`). The order therefore preserves the price at creation time, but there is no currency field, no tax line, no shipping line and no historical price table beyond the item snapshot.

All values are integers in cents; there is no rounding because multiplication is integral. The absence of a maximum quantity or maximum total means the computed value can exceed the 32-bit signed range of the MySQL `INTEGER` columns, a risk detailed in BR-41.

**Rule workflow**:
```
for each aggregated item:
  unitPriceCents = product.priceCents
  lineTotal = unitPriceCents * quantity
subtotalCents = sum(lineTotal)
totalCents = subtotalCents - discountCents
```

---

### Business Rule BR-16: Discount cannot exceed subtotal

**Overview**:
A discount greater than the computed subtotal is rejected with 400; a discount equal to the subtotal is accepted.

**Detailed description**:
After computing the subtotal, the service checks `if (input.discountCents > subtotalCents)` and throws `ValidationError('Discount cannot exceed subtotal', [{ path: 'discountCents', message: 'Discount exceeds subtotal' }])` (`src/modules/orders/order.service.ts:87-91`). The error is a 400 `VALIDATION_ERROR` with a field-level detail, even though Zod already validated the discount's type and non-negativity (BR-05). The comparison is strict, so `discountCents === subtotalCents` is legal and produces a zero total.

The check is inside the transaction but before any write, so a rejected discount persists nothing and does not consume an order number. Because the subtotal depends on server-side prices, the client cannot know the exact maximum discount unless it reads current product prices; a stale client estimate can be rejected. There is no partial discount, percentage discount, coupon or per-item discount model.

The discount is stored as an absolute amount (`discountCents`) and is not attributed to items. Refunds or cancellations do not use it: cancellation replenishes full item quantities and does not compute a refund amount. The rule therefore governs only the arithmetic relationship between subtotal and total at creation.

**Rule workflow**:
```
subtotalCents computed
  -> discountCents > subtotalCents -> 400 VALIDATION_ERROR (path "discountCents")
  -> discountCents <= subtotalCents -> totalCents = subtotalCents - discountCents
```

---

### Business Rule BR-17: Total equals subtotal minus discount

**Overview**:
`totalCents` is derived, never accepted from the client, and stored alongside the subtotal and discount.

**Detailed description**:
The service computes `const totalCents = subtotalCents - input.discountCents` (`src/modules/orders/order.service.ts:92`) and writes all three values into the order (`100-102`). No request field can override them because the schema strips `subtotalCents` and `totalCents` and the service does not read them. The invariant `total = subtotal - discount` holds by construction at creation and is never recomputed afterward, since there is no order update endpoint.

Because discount is bounded by subtotal (BR-16), the total is always non-negative at creation. Nothing prevents later out-of-band changes to these columns, and no database `CHECK` enforces the invariant (BR-41). The API also does not expose a way to change items, prices or discount, so through the module the values remain as created until deletion.

The response returns all three amounts as integers in cents. There is no formatting, currency symbol, tax computation or rounding layer; currency is implicit and unrecorded.

**Rule workflow**:
```
totalCents = subtotalCents - discountCents
  -> persisted with subtotalCents and discountCents
  -> returned unchanged on read
```

---

### Business Rule BR-18: New orders always start in PENDING

**Overview**:
The lifecycle always begins at `PENDING`; the caller cannot choose an initial status.

**Detailed description**:
`createOrderSchema` has no `status` field, so a posted `status` is stripped as an unknown key (BR-05). The service hardcodes `status: OrderStatus.PENDING` in the insert (`src/modules/orders/order.service.ts:99`), and the database column also defaults to `PENDING` (`prisma/schema.prisma:78`). The three layers agree, but only the service assignment is exercised through the API.

Starting at `PENDING` means no stock is debited at creation (BR-23) and payment is the first stock-affecting event (BR-31). It also means the first legal transitions are only to `PAID` or `CANCELLED` (BR-28). The initial history row records `fromStatus: null` and `toStatus: PENDING`, which is the only history entry with a null source.

There is no draft, quote, on-hold or backorder state. The six-state enum is fixed in Prisma and mirrored in MySQL, and no API operation can create an order in any state other than `PENDING`.

**Rule workflow**:
```
create
  -> status forced to PENDING
  -> history row null -> PENDING inserted in same transaction
```

---

### Business Rule BR-19: Server-generated sequential order number

**Overview**:
Each order receives a unique, human-readable `ORD-` number allocated from a singleton counter inside the creation transaction.

**Detailed description**:
`reserveOrderNumber` performs `upsert` on `order_number_sequence` with `id: 1` (`src/modules/orders/order.service.ts:245-254`). On first use it creates the row with `nextValue: 2` and reads 2, yielding current = 1; on later use it increments `nextValue` and subtracts one. The result is formatted with `String(current).padStart(6, '0')` and prefixed with `ORD-`, producing values such as `ORD-000001`. The service returns that string and writes it to `Order.orderNumber`, which has a unique index (`prisma/schema.prisma:76`).

Because the upsert runs inside the same interactive transaction as the order insert, a failed creation rolls back the increment and the number can be reused; committed creations consume numbers monotonically. Concurrent creators contend on the single row: MySQL locks it for the duration of each transaction, serializing the numbering step. That is the intended uniqueness mechanism, not an advisory lock. The counter has no cycle, upper bound or reset.

The format is fixed at six digits but `padStart` does not truncate; after 999999 the value naturally grows to seven digits, still within `VARCHAR(20)`. The seed script duplicates the same algorithm independently (`prisma/seed.ts:165-177`), so there are two implementations that must stay aligned.

**Rule workflow**:
```
BEGIN
  UPSERT order_number_sequence id=1
    insert: nextValue=2 -> read 2
    update: nextValue=nextValue+1 -> read new value
  current = nextValue - 1
  orderNumber = "ORD-" + zeroPad(current, 6)
  INSERT order(orderNumber)
COMMIT
```

---

### Business Rule BR-20: Creator identity is taken from the token

**Overview**:
`createdById` always records the authenticated user and cannot be spoofed through the body.

**Detailed description**:
The controller passes `req.user.id` as the second argument to `OrderService.create` (`src/modules/orders/order.controller.ts:31`), and the service writes it to `createdById` (`src/modules/orders/order.service.ts:104`). `req.user.id` comes from the verified JWT `sub` claim (`src/middlewares/auth.middleware.ts:42`). The create schema has no actor field, so a body-supplied `createdById` is stripped.

`Order.createdById` is a foreign key to `users.id` with `ON DELETE RESTRICT` (`prisma/migrations/20260519182739_init/migration.sql:92`), so a token referencing a deleted user would fail the insert. Token issuance does not check user existence at request time, so a still-valid token for a deleted user can reach the insert and produce an unmapped 500 rather than a 401 or 404. This edge depends on deletion occurring inside the token lifetime.

The same actor-stamping pattern applies to status history (BR-34). There is no `updatedBy` field on the order itself, so the most recent actor is only discoverable through the latest history row.

**Rule workflow**:
```
JWT verified -> req.user.id = payload.sub
  -> controller passes userId
  -> order.createdById = userId
  -> history.changedById = userId
```

---

### Business Rule BR-21: Initial audit history on creation

**Overview**:
Creation writes a first history row representing the transition from no status to `PENDING`.

**Detailed description**:
The nested create includes `history.create({ fromStatus: null, toStatus: PENDING, changedById: userId, reason: 'order created' })` (`src/modules/orders/order.service.ts:106-113`). This produces an audit entry even though no external transition occurred. `fromStatus` is nullable in the schema specifically to represent creation (`prisma/schema.prisma:119`), and `reason` is the fixed English string `order created`, not client input.

The row is inserted in the same transaction as the order, so an order can never exist without its creation history. The read projection orders history by `changedAt` ascending (`src/modules/orders/order.repository.ts:56`; `src/modules/orders/order.service.ts:117,173`), making the creation row first unless timestamps collide within the same millisecond. There is no explicit sequence column, so ordering among same-millisecond rows is not guaranteed.

History rows are never modified or deleted by the module. The only way a history row disappears is cascade deletion of the parent order (BR-37). The initial reason string is not internationalized and not derived from configuration.

**Rule workflow**:
```
INSERT order
  + INSERT order_status_history {
      fromStatus: null,
      toStatus: PENDING,
      changedById: userId,
      reason: "order created"
    }
```

---

### Business Rule BR-22: Creation is atomic

**Overview**:
All creation reads and writes occur in one Prisma interactive transaction and either all persist or none do.

**Detailed description**:
The service wraps customer lookup, product lookup, active check, arithmetic, sequence allocation and the nested order insert in `this.prisma.$transaction(async (tx) => ...)` (`src/modules/orders/order.service.ts:58-123`). Any thrown `AppError` aborts the callback, Prisma rolls back, and the error propagates to the controller and error middleware. Consequences include no orphan items, no history row without an order, and no sequence consumption on failure.

The transaction mixes reads used for validation with writes used for persistence. Under MySQL's default isolation (REPEATABLE READ), the reads see a consistent snapshot, but they do not lock the customer or product rows for update. A product's `active` flag or price can change between the read and the commit by another transaction; the created order uses the values observed by this transaction. There is no explicit isolation-level override in the code.

Prisma interactive transactions have default limits of 5 seconds timeout and 2 seconds max wait. Exceeding them raises transaction errors that are not `AppError` instances, so they fall through to the generic 500 handler. The sequence row is held for the remainder of the transaction after the upsert, which lengthens the critical section.

**Rule workflow**:
```
$transaction(async tx => {
  read customer, products
  validate active, discount
  upsert sequence
  insert order + items + history
})
  -> success: COMMIT
  -> any throw: ROLLBACK, propagate error
```

---

### Business Rule BR-23: No stock check at creation

**Overview**:
Creating an order never verifies availability, never reserves inventory and never changes stock.

**Detailed description**:
The creation path reads products for existence, active flag and price only (`src/modules/orders/order.service.ts:62-84`). `stockQuantity` is not read, compared or updated. Consequently an order for quantity 5 can be created against a product with stock 1; the integration test at `tests/orders.test.ts:109-119` asserts exactly that behavior by expecting 201 and later a 422 at payment time.

Availability is enforced only on `PENDING -> PAID` (BR-31). Between creation and payment, stock can be consumed by other orders, so a created order is a request, not a reservation. There is no backorder flag, no partial fulfillment and no allocation record. The business consequence is deferred oversell exposure rather than prevention.

Because no stock is touched at creation, `PENDING -> CANCELLED` requires no replenishment (BR-33), and creation cannot fail due to inventory. The rule also means stock levels are never validated against pending demand, so the product module's stock number reflects only paid-order debits and manual changes.

**Rule workflow**:
```
create:
  read product existence, active, price
  do NOT read stockQuantity
  do NOT reserve or decrement stock
  -> order may be unfulfillable at payment time
```

---

### Business Rule BR-24: Offset pagination with bounded page size

**Overview**:
Listing is page-based, defaults to page 1 and 20 rows, and cannot return more than 100 rows per request.

**Detailed description**:
The service converts the validated query into `skip: (query.page - 1) * query.pageSize` and `take: query.pageSize` (`src/modules/orders/order.service.ts:38-39`). The schema guarantees page >= 1 and 1 <= pageSize <= 100 (`src/modules/orders/order.schemas.ts:24-25`), so skip is non-negative and take is bounded. There is no cursor pagination, no maximum page and no total-row cap.

Offset pagination has the usual stability characteristics: rows inserted or deleted between page requests shift the window, and deep offsets require the database to scan preceding rows. The repository applies `skip` and `take` after the `where` filter and `createdAt desc` sort (`src/modules/orders/order.repository.ts:36-41`), so the page contents are determined by the current data at query time. There is no snapshot token.

If `page` is beyond the last page, the response is an empty `data` array with the real `total` and a `totalPages` smaller than `page`. This is not treated as an error. The maximum of 100 limits response size but not the number of pages a client can request.

**Rule workflow**:
```
page >= 1, 1 <= pageSize <= 100
skip = (page - 1) * pageSize
take = pageSize
SELECT ... ORDER BY createdAt DESC LIMIT take OFFSET skip
```

---

### Business Rule BR-25: List filtering and ordering

**Overview**:
Orders can be filtered by status, customer and creation date, and are always returned newest first.

**Detailed description**:
`buildWhere` conditionally sets `status`, `customerId`, and a `createdAt` object with `gte` from `from` and `lte` from `to` (`src/modules/orders/order.repository.ts:21-31`). Empty filters produce an empty `where`, so the endpoint lists all orders. Filters are combined with AND semantics because they are fields of one Prisma `where` object. There is no OR, text search, id list, amount range or creator filter.

`orderBy` is fixed to `{ createdAt: 'desc' }` (`38`), with no client-selectable sort or secondary key. Orders created in the same millisecond have no deterministic relative order. The count query uses the same `where`, and both queries are issued together in a Prisma `$transaction` array (`35-43`), giving the response a consistent total relative to the page query within that transaction.

No authorization filter is applied: the where clause never includes `createdById`, so a user sees every order matching the filters. That is a consequence of BR-03. Pagination metadata is produced from the filtered count, not the global count.

**Rule workflow**:
```
where = {}
  if status     -> where.status = status
  if customerId -> where.customerId = customerId
  if from/to    -> where.createdAt = { gte: from?, lte: to? }
findMany({ where, orderBy: { createdAt: desc }, skip, take })
count({ where })
```

---

### Business Rule BR-26: Pagination response envelope

**Overview**:
List responses always wrap results in `data` plus a `pagination` object with page, pageSize, total and totalPages.

**Detailed description**:
`paginated(items, page, pageSize, total)` returns `{ data, pagination }`, and `buildPagination` computes `totalPages = pageSize === 0 ? 0 : Math.ceil(total / pageSize)` (`src/shared/http/response.ts:13-24`). The service calls it with the validated page and pageSize and the count from the repository (`src/modules/orders/order.service.ts:41`). The wrapper is used only by the list operation; get, create and status responses are bare order objects.

The envelope does not include `hasNext`, `hasPrevious`, links, cursors or the applied filters. `pageSize === 0` is impossible through HTTP because the schema enforces a minimum of 1; the branch is defensive in the shared helper. `totalPages` can be 0 when no rows match, with `page` still echoing the request.

Consumers must infer empty pages from `data.length === 0` and compare `page` with `totalPages`. The shape is consistent across modules that use the helper (products, customers), so it functions as a shared list contract even though it is not described in an OpenAPI document.

**Rule workflow**:
```
items, total -> paginated()
  -> { data: items, pagination: { page, pageSize, total, totalPages } }
  -> 200
```

---

### Business Rule BR-27: Get-by-id returns the full aggregate

**Overview**:
Retrieving one order eager-loads items with product summaries, the full status history and a customer summary, or returns 404.

**Detailed description**:
`findByIdWithRelations` includes `items` with `product { id, sku, name }`, `history` ordered by `changedAt asc`, and `customer { id, name, email }` (`src/modules/orders/order.repository.ts:47-60`). The service throws `NotFoundError('Order')` when the result is null (`src/modules/orders/order.service.ts:44-48`), producing 404 `NOT_FOUND` with message `Order not found`. The projection includes all scalar order fields, including `createdById`, and all scalar item fields.

The customer projection excludes `phone`, `document` and `address`, and no user projection is included, so `createdById` is an opaque id in the response. History rows include `changedById` but not the actor's name or email. Items include the product's current `sku` and `name`; if a product is renamed, historical items display the new name while retaining the original price snapshot.

The aggregate is returned for the detail endpoint only. The list endpoint returns flat order rows (BR-25), so clients cannot obtain items, history or customer data without one detail call per order. This asymmetry is a module-level read design choice rather than a business constraint.

**Rule workflow**:
```
SELECT order WHERE id
  include items.product(id, sku, name)
  include history ORDER BY changedAt ASC
  include customer(id, name, email)
    -> null -> 404 NOT_FOUND
    -> found -> 200 aggregate
```

---

### Business Rule BR-28: Whitelisted order state transitions

**Overview**:
The order lifecycle is a directed graph with six states, four forward paths, three cancellation paths and two terminal states.

**Detailed description**:
`transitions` is a read-only record mapping each `OrderStatus` to its allowed successors (`src/modules/orders/order.status.ts:3-10`): `PENDING -> [PAID, CANCELLED]`, `PAID -> [PROCESSING, CANCELLED]`, `PROCESSING -> [SHIPPED, CANCELLED]`, `SHIPPED -> [DELIVERED]`, `DELIVERED -> []`, `CANCELLED -> []`. `canTransition` returns whether the target is in the source's array (`12-14`). The table is the single source of truth for legal movement and is consulted only by `changeStatus`.

The graph permits cancellation from the three pre-shipment active states and permits no cancellation after shipment. It permits no skipping: `PENDING -> PROCESSING`, `PENDING -> SHIPPED`, `PAID -> SHIPPED` and `PROCESSING -> DELIVERED` are all illegal. Backward movement, reopening a cancelled order and editing a delivered order are impossible. `DELIVERED` and `CANCELLED` are terminal by having empty successor arrays; `isTerminal` encodes that fact (`20-22`) but is not called anywhere in `src`.

The transition table is not persisted and not exposed by an endpoint. It is applied after a same-status check and before any stock effect, so an illegal target has no side effects. Because the table is code, a schema enum addition would require a matching code change; otherwise TypeScript's record type would fail compilation.

**Rule workflow**:
```
from, to
  -> transitions[from].includes(to)?
     PENDING    -> PAID | CANCELLED
     PAID       -> PROCESSING | CANCELLED
     PROCESSING -> SHIPPED | CANCELLED
     SHIPPED    -> DELIVERED
     DELIVERED  -> (none)
     CANCELLED  -> (none)
```

---

### Business Rule BR-29: Same-status transitions are rejected

**Overview**:
Requesting the status an order already has returns 409 rather than succeeding idempotently.

**Detailed description**:
Before consulting the transition table, `changeStatus` compares `from === to` and throws `ConflictError('Order is already in <status> status', 'INVALID_STATUS_TRANSITION', { from, to })` (`src/modules/orders/order.service.ts:140-146`). The response is 409 with the shared `INVALID_STATUS_TRANSITION` code, the same code used for illegal transitions (BR-30). The message is more specific than the generic transition message.

This makes `PATCH /:id/status` non-idempotent from the client's perspective: a retry of a successful `PENDING -> PAID` after the first response is lost returns 409, not 200. The order is not modified, no history row is added and no stock is touched because the comparison precedes all effects. There is no "no-op if already there" mode and no `If-Match`/version precondition.

The same-status rule also covers terminal states: setting a `DELIVERED` order to `DELIVERED` produces the "already in" 409 rather than the "invalid transition" 409. Both are 409, so clients that branch on status code cannot distinguish them except by message or details.

**Rule workflow**:
```
from = stored status
to   = requested status
  from === to -> 409 INVALID_STATUS_TRANSITION ("already in")
  from !== to -> canTransition check
```

---

### Business Rule BR-30: Illegal transitions are rejected

**Overview**:
Any target not present in the source status's whitelist is a 409 conflict with structured details.

**Detailed description**:
When `canTransition(from, to)` is false, the service throws `InvalidStatusTransitionError(from, to)` (`src/modules/orders/order.service.ts:147-149`), a subclass of `ConflictError` that sets the code `INVALID_STATUS_TRANSITION`, the message `Invalid status transition from <from> to <to>`, and details `{ from, to }` (`src/shared/errors/http-errors.ts:45-53`). The error is an `AppError`, so `errorMiddleware` serializes it as 409 with the envelope and details.

The rejection occurs before stock evaluation. An illegal `PENDING -> SHIPPED` therefore cannot debit stock, and an illegal `SHIPPED -> CANCELLED` cannot replenish it. No order row, history row or product row is written. The transaction still rolls back cleanly, although for this path no write had occurred.

The rule assumes the stored status is one of the six enum values. If the database contained an out-of-band value, `transitions[from]` would be `undefined` and `.includes` would throw a `TypeError`, producing a 500. MySQL's enum constraint prevents this through normal writes.

**Rule workflow**:
```
canTransition(from, to) === false
  -> InvalidStatusTransitionError(from, to)
  -> 409 { code: INVALID_STATUS_TRANSITION, details: { from, to } }
```

---

### Business Rule BR-31: Stock is debited on payment, all-or-nothing

**Overview**:
`PENDING -> PAID` verifies every line's stock and then decrements all lines; if any line is short, nothing is debited and 422 is returned.

**Detailed description**:
`shouldDebitStock` is true only for the exact pair `PENDING -> PAID` (`src/modules/orders/order.status.ts:24-31`). `debitStock` loads all products for the order's items, builds an `unavailable` array for any product that is missing or has `stockQuantity < quantity`, and throws `InsufficientStockError(unavailable)` before any update (`src/modules/orders/order.service.ts:204-224`). The error detail includes `{ sku, requested, available }` per failing line, where a missing product falls back to the product id as SKU and available 0.

Only after the full precheck passes does the method decrement each product with Prisma's atomic `{ decrement: quantity }` (`225-230`). Because the precheck and the decrements are in the same transaction, a failure during the decrement loop rolls back earlier decrements. There is no partial fulfillment and no reservation of the short lines.

The precheck is read-then-write, not a conditional update (`UPDATE ... WHERE stock >= qty`). Two concurrent payments can both read sufficient stock and both decrement, potentially driving stock below zero. This is a property of the rule as implemented, described further in BR-39. Also note that quantities come from the stored `order.items`, not from the transition request.

**Rule workflow**:
```
PENDING -> PAID
  SELECT products for order items
  for each item:
     missing or stock < quantity -> unavailable.push({sku, requested, available})
  unavailable non-empty -> 422 INSUFFICIENT_STOCK { unavailable }
  otherwise:
     for each item: UPDATE product SET stock = stock - quantity
     UPDATE order SET status = PAID
     INSERT history
```

---

### Business Rule BR-32: Stock is replenished on cancellation after payment

**Overview**:
Cancelling a `PAID` or `PROCESSING` order returns the ordered quantities to stock.

**Detailed description**:
`shouldReplenishStock` returns true when `to === CANCELLED` and `from` is `PAID` or `PROCESSING` (`src/modules/orders/order.status.ts:33-37`). `replenishStock` issues an atomic `{ increment: quantity }` per order item (`src/modules/orders/order.service.ts:233-243`). This mirrors the debit that occurred at `PENDING -> PAID`, restoring the exact ordered quantities regardless of subsequent price or catalogue changes.

The predicate deliberately excludes `PENDING -> CANCELLED`, because nothing was debited at creation (BR-23). It also excludes `SHIPPED -> CANCELLED`, which is not a legal transition anyway. Because `CANCELLED` is terminal, a cancelled order cannot be re-paid and therefore cannot double-replenish through the API. Deleting a cancelled order afterward does not adjust stock again (BR-37).

The increment is unconditional: it does not cap at an original stock value and does not check whether the stock was modified manually after the debit. If an operator manually reduced stock while the order was `PAID`, cancellation still adds the full quantity, which may exceed pre-order levels. There is no stock movement ledger; only the current integer is changed.

**Rule workflow**:
```
(PAID -> CANCELLED) or (PROCESSING -> CANCELLED)
  for each order item: UPDATE product SET stock = stock + quantity
  UPDATE order SET status = CANCELLED
  INSERT history
```

---

### Business Rule BR-33: No stock effect on other transitions

**Overview**:
Only payment and post-payment cancellation move stock; all other legal transitions are stock-neutral.

**Detailed description**:
The service calls `debitStock` only when `shouldDebitStock` is true and `replenishStock` only when `shouldReplenishStock` is true (`src/modules/orders/order.service.ts:151-156`). The remaining legal transitions are `PENDING -> CANCELLED`, `PAID -> PROCESSING`, `PROCESSING -> SHIPPED` and `SHIPPED -> DELIVERED`. None of them reads or writes `stockQuantity`. Fulfilment progression therefore does not allocate, reserve or consume additional inventory beyond the payment debit.

This design treats stock as debited at payment, not at shipment. Shipping does not verify that the debited units still exist; if stock was manually reduced after payment, `PROCESSING -> SHIPPED` still succeeds. Conversely, cancellation from `PENDING` does not restore anything because nothing was taken, which keeps the ledger consistent with BR-23.

Because the predicates are pure functions of `(from, to)`, the stock policy is centralized and testable independently of the service. There is no configuration switch or feature flag to change when stock is debited. `STOCK_DEBIT_TRANSITION` is exported as a constant but only the predicate uses it inside the module.

**Rule workflow**:
```
shouldDebitStock(from,to)     -> only PENDING -> PAID
shouldReplenishStock(from,to) -> only PAID|PROCESSING -> CANCELLED
all other legal transitions   -> status + history only
```

---

### Business Rule BR-34: Every status change is audited

**Overview**:
A successful transition appends a history row containing source, target, actor and optional reason.

**Detailed description**:
After updating the order status, `changeStatus` inserts `orderStatusHistory` with `orderId`, `fromStatus`, `toStatus`, `changedById: userId` and `reason: input.reason ?? null` (`src/modules/orders/order.service.ts:159-167`). The actor comes from the JWT, not the body (BR-20). `reason` is optional and capped at 500 characters by the request schema (BR-07), matching the column width (`prisma/schema.prisma:123`).

The history is append-only from the module's perspective: there is no update or delete method, no endpoint, and no service path that modifies an existing row. The only removal path is cascade deletion of the order (BR-37). Reads order ascending by `changedAt`, so the sequence of changes is reconstructable, subject to same-millisecond tie ambiguity because there is no monotonic per-order sequence column.

Because the history insert is in the same transaction as the status update, an order's current status and its latest history target cannot diverge through the module. A history row is written for every successful transition, including cancellations and delivery, but not for rejected transitions or same-status requests. The creation row (BR-21) is the only row with a null `fromStatus`.

**Rule workflow**:
```
status update success
  -> INSERT order_status_history {
       orderId,
       fromStatus: previous,
       toStatus: new,
       changedById: JWT user,
       reason: body.reason ?? null
     }
  -> commit with status update and stock changes
```

---

### Business Rule BR-35: Status change is atomic

**Overview**:
Stock movement, status update and history insertion for a transition commit or roll back together.

**Detailed description**:
`changeStatus` wraps the entire operation in `this.prisma.$transaction(async (tx) => ...)` (`src/modules/orders/order.service.ts:131-178`). The order read, transition checks, stock debit or replenish, status update, history insert and final re-read all use the same `tx` client. If any step throws, Prisma rolls back the writes, including any stock decrements or increments already applied in the loop.

The final re-read occurs inside the transaction and is returned with the non-null assertion `refreshed!` (`169-177`). Because the row was just updated in the same transaction, it must exist; the assertion documents that assumption. If the order were deleted concurrently, the update would affect zero rows and the re-read would return null, which would surface as a runtime error rather than a domain 404.

The transaction does not lock the order row at read time. Atomicity guarantees that a single request's changes are all-or-nothing, but it does not serialize competing requests. Isolation between concurrent transitions is the subject of BR-39.

**Rule workflow**:
```
$transaction(async tx => {
  read order + items
  validate transition
  optional stock debit / replenish
  update status
  insert history
  re-read aggregate
})
  -> success COMMIT -> 200
  -> any throw ROLLBACK -> mapped error
```

---

### Business Rule BR-36: Deletion is restricted by lifecycle state

**Overview**:
Only `PENDING` and `CANCELLED` orders may be deleted; every other status returns 409.

**Detailed description**:
`delete` calls `findById`, throws `NotFoundError('Order')` if absent, and then checks `order.status !== PENDING && order.status !== CANCELLED` (`src/modules/orders/order.service.ts:181-190`). A violation throws `ConflictError('Order can only be deleted while in PENDING or CANCELLED status', 'INVALID_ORDER_STATE_FOR_DELETE', { status })`, which maps to 409 with the current status in details. `PAID`, `PROCESSING`, `SHIPPED` and `DELIVERED` are all protected.

The permitted set aligns with stock accounting: a `PENDING` order never debited stock, and a `CANCELLED` order has either never debited or has already been replenished. Deletion therefore does not create a stock imbalance under normal module behavior. Allowing deletion of `CANCELLED` also means the record of a cancelled paid order and its audit trail can disappear.

The check is read-then-delete outside a transaction. If another request changes the status between the read and the delete, the delete proceeds based on the stale status; if another request deletes the row first, Prisma raises P2025 and the centralized middleware returns a generic 404. There is no soft-delete flag, no `deletedAt`, and no retention window.

**Rule workflow**:
```
DELETE /:id
  SELECT order
    -> missing -> 404 NOT_FOUND
    -> status in {PENDING, CANCELLED} -> DELETE
    -> otherwise -> 409 INVALID_ORDER_STATE_FOR_DELETE { status }
```

---

### Business Rule BR-37: Hard delete with cascade, 204 response

**Overview**:
Deletion physically removes the order and its dependent rows and returns an empty 204 response.

**Detailed description**:
`deleteById` executes `prisma.order.delete({ where: { id } })` (`src/modules/orders/order.repository.ts:66-68`), and the controller responds with `res.status(204).send()` (`src/modules/orders/order.controller.ts:50-51`). MySQL removes `order_items` and `order_status_history` through `ON DELETE CASCADE` (`prisma/schema.prisma:108,125`; migration lines 95, 103). Products, customers and users are untouched, and the `orders.createdById`/`customerId` foreign keys are `RESTRICT`, which only affects deletion of those parents, not the order.

No stock adjustment occurs during deletion. As noted in BR-36, the allowed states are exactly those where stock is balanced, so the module relies on that invariant rather than reconciling at delete time. If an order reached `CANCELLED` through manual database changes without replenishment, deleting it would leave stock short.

Deletion writes no audit row and there is no tombstone. After a 204, the order number remains consumed in `order_number_sequence` and is never reused, while the order itself is unrecoverable through the API. The response has no body and no `Content-Type` payload.

**Rule workflow**:
```
allowed status -> DELETE order
  -> CASCADE delete order_items, order_status_history
  -> no stock change, no audit row
  -> 204 empty body
```

---

### Business Rule BR-38: Uniform error mapping

**Overview**:
All module failures are expressed through the shared `AppError` hierarchy and a single JSON error envelope.

**Detailed description**:
The service throws typed errors: `ValidationError` for empty items and discount overflow, `NotFoundError` for customer, product and order, `ConflictError`/`InvalidStatusTransitionError` for lifecycle conflicts, `UnprocessableEntityError` for inactive products and `InsufficientStockError` for stock shortage. Each carries an HTTP status and a stable code (`src/shared/errors/http-errors.ts:9-63`). `errorMiddleware` serializes `AppError` as `{ error: { code, message, details? } }` with `details` omitted when undefined (`src/middlewares/error.middleware.ts:15-24`).

Unhandled Prisma known errors are mapped: P2002 to 409 `CONFLICT` with the unique target, and P2025 to 404 `NOT_FOUND` with a generic message (`37-53`). Other errors, including Prisma transaction timeouts and 32-bit overflow errors, are logged with request id, method and path, then returned as 500 `INTERNAL_SERVER_ERROR` with no internal detail (`56-64`). A raw `ZodError` reaching the middleware (only if validation middleware is bypassed) is also mapped to 400.

The envelope is consistent across modules because the same middleware and error classes are shared. The module adds no module-specific error codes beyond `INACTIVE_PRODUCT`, `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION` and `INVALID_ORDER_STATE_FOR_DELETE`. A unique-constraint race on `orderNumber` would surface as generic `CONFLICT`, not as an order-specific code.

**Rule workflow**:
```
throw AppError subclass
  -> errorMiddleware
     -> statusCode + { error: { code, message, details? } }
Prisma P2002 -> 409 CONFLICT
Prisma P2025 -> 404 NOT_FOUND
other        -> log + 500 INTERNAL_SERVER_ERROR
```

---

### Business Rule BR-39: Concurrency is not guarded at the row level

**Overview**:
Status transitions and stock updates rely on read-then-write inside transactions, with no row lock, version column or conditional update.

**Detailed description**:
`changeStatus` reads the order and items with `findUnique` (`src/modules/orders/order.service.ts:132-135`), evaluates the transition and stock in application memory, then issues unconditional updates (`158`, `226-229`, `238-241`). There is no `SELECT ... FOR UPDATE`, no `version` field on `Order` (`prisma/schema.prisma:74-97`), and no `WHERE status = from` guard on the update. Prisma uses the database's default isolation, which for MySQL InnoDB is REPEATABLE READ; consistent reads therefore see a snapshot rather than the latest committed row.

Under concurrent requests, two `PENDING -> PAID` transitions can both read `PENDING`, both pass `canTransition`, both pass the stock precheck against the same snapshot, and both decrement. The result is two history rows, a final status of `PAID`, and stock reduced twice, potentially below zero. Similar interleavings apply to `PAID -> CANCELLED` versus `PAID -> PROCESSING`: both may commit, with the last status update winning and both side effects applied. The stock precheck is not a reservation, so even a single expensive transaction increases the window.

Order creation has a different concurrency profile: the `order_number_sequence` upsert locks one row and serializes the numbering step, but customer and product reads are unlocked and can observe changing prices or active flags. The delete path reads and deletes outside a transaction, so a concurrent status change can invalidate the deletion guard. These behaviors are properties of the implementation, not of an explicitly documented isolation policy; no code comment or configuration states an intended level.

**Rule workflow**:
```
TX A: read order(status=PENDING), validate, read stock, decrement, update, history, commit
TX B: read order(status=PENDING), validate, read stock, decrement, update, history, commit
  -> both may pass; no lock/version prevents double side effect
```

---

### Business Rule BR-40: Date-range filtering semantics

**Overview**:
`from` and `to` bound `createdAt` inclusively, are parsed as generic dates, and are not validated against each other.

**Detailed description**:
The query schema uses `z.coerce.date()` for both bounds (`src/modules/orders/order.schemas.ts:28-29`), which accepts ISO strings, date-only strings and values a JavaScript `Date` can parse. The repository applies `gte` for `from` and `lte` for `to` (`src/modules/orders/order.repository.ts:25-29`), so both endpoints are inclusive. There is no default range, no maximum span and no requirement that both be supplied.

A date-only `to` such as `2026-11-08` is parsed as midnight UTC, so orders created later that day are excluded. No timezone is configured in the schema, the environment or the Prisma datasource; MySQL `DATETIME(3)` stores values without timezone, and Prisma sends JavaScript `Date` values. The resulting day boundaries depend on the process timezone and the client's string format, which is not documented.

If `from > to`, the query is still executed and returns an empty set rather than a 400. Because the filters are ANDed, only rows satisfying both bounds are returned. There is no range validation against `updatedAt`, no `changedAt` history filtering and no relative-date syntax.

**Rule workflow**:
```
from? -> where.createdAt.gte = Date(from)
to?   -> where.createdAt.lte = Date(to)
  -> inclusive bounds, no from<=to check, no timezone normalization
```

---

### Business Rule BR-41: Domain invariants are application-only

**Overview**:
The database enforces types, enums, foreign keys, uniqueness and not-null, but not pricing arithmetic, status transitions or non-negative stock.

**Detailed description**:
The migration creates the six-value `OrderStatus` enum, `CHAR(36)` ids, `VARCHAR` limits, unique indexes on `orderNumber` and product `sku`, foreign keys with `RESTRICT`/`CASCADE`, and indexes for customer, status, createdAt and createdById (`prisma/migrations/20260519182739_init/migration.sql:44-105`). It defines no `CHECK` constraints. Consequently `subtotalCents`, `discountCents`, `totalCents` and `stockQuantity` can be written with values that violate the service's arithmetic, including negative stock, and `status` can be changed by any SQL client without consulting the transition table.

All monetary columns are MySQL `INTEGER`, which Prisma maps to a 32-bit signed integer. `priceCents` is non-negative at product creation (`src/modules/products/product.schemas.ts:9`), but `quantity` has no maximum (`src/modules/orders/order.schemas.ts:8`), so a large quantity times a large price can exceed 2,147,483,647 cents. The database rejects such a value, the error is not a mapped Prisma code, and the client receives a 500. The same range applies to the sequence's `nextValue`.

Because the invariants live in code, they hold only for writes that pass through `OrderService`. The seed script writes orders and history directly with Prisma (`prisma/seed.ts:226-260+`), constructing plausible states without invoking the service, which is acceptable for fixtures but demonstrates that the database is not the invariant owner.

**Rule workflow**:
```
DB constraints: enum, FK, unique, NOT NULL, VARCHAR/INT ranges
DB absent:      CHECK(total >= 0), CHECK(discount <= subtotal),
                CHECK(stock >= 0), status transition constraint
  -> invariants enforced only by OrderService
```

---

### Business Rule BR-42: Direct cross-module table access

**Overview**:
The module treats `customers`, `products` and `order_number_sequence` as directly accessible tables rather than as other modules' domains.

**Detailed description**:
Inside transactions, `OrderService` calls `tx.customer.findUnique`, `tx.product.findMany`, `tx.product.update` and `tx.orderNumberSequence.upsert` (`src/modules/orders/order.service.ts:59-73,93,204-254`). The module's own `OrderRepository` covers only `orders` reads and deletes. `CustomerRepository` and `ProductRepository` exist but are not imported by the Orders Module, so customer and product persistence rules are duplicated or bypassed.

The consequence is schema-level coupling: any rename or semantic change to `priceCents`, `active`, `stockQuantity`, `Product.id` or the customer table propagates directly into order logic. It also means customer and product modules cannot enforce their own access or invariant policies for order-driven writes; for example, stock updates bypass any future stock-adjustment logic in `ProductService`. The transaction client type `Prisma.TransactionClient` is the de facto port.

This rule is architectural rather than user-facing, but it shapes the module's dependency graph: the Orders Module's efferent coupling includes the Prisma schema entities of two other feature modules even though there is no TypeScript import from those modules. The architectural report records the same observation at `src/modules/orders/order.service.ts:58-123,204-254`.

**Rule workflow**:
```
OrderService (inside tx)
  -> tx.customer (customers table)
  -> tx.product (products table, including stock writes)
  -> tx.orderNumberSequence (sequence table)
  -> OrderRepository only for orders reads/deletes
```

---

## 4. Component Structure

```
src/modules/orders/
├── order.routes.ts        # 27 lines - Express Router factory: authenticate + validate + controller binding
├── order.controller.ts    # 56 lines - Transport adapter: 5 handlers, status codes, next(err)
├── order.service.ts       # 255 lines - Business rules: creation transaction, status transaction,
│                          #             pricing, item aggregation, stock debit/replenish,
│                          #             order-number reservation, delete guard
├── order.repository.ts    # 69 lines - Order persistence: list/count, findById, findByIdWithRelations,
│                          #             deleteById; where-clause builder
├── order.status.ts        # 37 lines - Transition table, canTransition, allowedTransitions,
│                          #             isTerminal, stock side-effect predicates
└── order.schemas.ts       # 34 lines - Zod contracts: id param, item input, create, status update, list query
```

Supporting files outside the boundary that define module behavior:

```
src/
├── app.ts                              # Composition root: constructs OrderRepository, OrderService, OrderController (42-44)
├── routes/index.ts                     # Mounts buildOrderRouter at /orders (28)
├── middlewares/
│   ├── auth.middleware.ts              # authenticate, requireRole (requireRole unused by orders)
│   ├── validate.middleware.ts          # Zod parsing, ValidationError conversion, req.query merge
│   ├── error.middleware.ts             # AppError/Prisma/Zod -> HTTP envelope
│   └── request-logger.middleware.ts    # X-Request-Id and completion log used in 500 path
├── shared/
│   ├── errors/                         # AppError + domain error subclasses
│   └── http/response.ts                # PaginatedResponse and totalPages computation
└── config/
    ├── env.ts                          # DATABASE_URL, JWT_SECRET, NODE_ENV (fail-fast startup)
    └── database.ts                     # Singleton PrismaClient

prisma/
├── schema.prisma                       # OrderStatus enum, Order, OrderItem, OrderStatusHistory, OrderNumberSequence
├── migrations/20260519182739_init/     # Tables, enum, FKs, indexes, cascade rules
└── seed.ts                             # Destructive demo data; duplicates order-number reservation (165-177)

tests/
├── orders.test.ts                      # 8 HTTP integration tests against real MySQL
├── helpers/factories.ts                # getTestApp, createTestCustomer, createTestProduct, auth bootstrap
└── setup.ts                            # DB connect, truncate before each test, disconnect after all
```

File-level responsibilities and boundary notes:

| File | Responsibility | Owns domain rules | Touches DB | HTTP awareness |
|------|----------------|-------------------|------------|----------------|
| `order.routes.ts` | Path table, middleware order, schema binding | No | No | Yes |
| `order.controller.ts` | Request/response adaptation, identity guard | No | No | Yes |
| `order.schemas.ts` | Transport contracts, enum membership, defaults, bounds | Partly (input shape only) | No | No |
| `order.service.ts` | All order business rules, transactions, stock side effects, numbering | Yes | Yes (direct Prisma) | No |
| `order.status.ts` | Transition whitelist and stock policy predicates | Yes | No | No |
| `order.repository.ts` | Order read/delete queries and projections | No | Yes (Prisma) | No |

---

## 5. Dependency Analysis

### Internal dependencies

```
buildOrderRouter (order.routes.ts)
  ├── express Router
  ├── authenticate .......................... src/middlewares/auth.middleware.ts
  ├── validate .............................. src/middlewares/validate.middleware.ts
  ├── order.schemas.ts (4 schemas)
  └── OrderController (type only)

OrderController (order.controller.ts)
  ├── OrderService (type only)
  ├── ListOrdersQuery (type from order.schemas.ts)
  ├── UnauthorizedError ..................... src/shared/errors
  └── express types

OrderService (order.service.ts)
  ├── OrderRepository + OrderWithRelations (type)
  ├── CreateOrderInput, ListOrdersQuery, UpdateOrderStatusInput (types)
  ├── canTransition, shouldDebitStock, shouldReplenishStock ... order.status.ts
  ├── paginated, PaginatedResponse .......... src/shared/http/response.ts
  ├── ConflictError, InsufficientStockError, InvalidStatusTransitionError,
  │   NotFoundError, UnprocessableEntityError, ValidationError ... src/shared/errors
  └── PrismaClient, Prisma.TransactionClient, Order, OrderStatus ... @prisma/client

OrderRepository (order.repository.ts)
  ├── PrismaClient, Prisma.OrderWhereInput
  └── Order, OrderItem, OrderStatus, OrderStatusHistory types

order.status.ts
  └── OrderStatus enum ...................... @prisma/client

order.schemas.ts
  ├── zod
  └── OrderStatus enum ...................... @prisma/client

Composition root (src/app.ts)
  └── OrderRepository -> OrderService -> OrderController -> Controllers.orders

Router registry (src/routes/index.ts)
  └── buildOrderRouter(controllers.orders)

Error middleware (src/middlewares/error.middleware.ts)
  └── consumes AppError subclasses thrown by the module

Tests (tests/orders.test.ts)
  ├── tests/helpers/factories.ts -> src/app.ts, src/config/database.ts
  └── src/config/database.ts (direct stock assertions)
```

### External dependencies

| Dependency | Version | Role in the module | Coupling point |
|------------|---------|--------------------|----------------|
| `express` | 4.21.1 | Router, RequestHandler, request/response objects | `order.routes.ts:1`, `order.controller.ts:1` |
| `zod` | 3.23.8 | Request contracts, coercion, defaults, enum validation, UUID check | `order.schemas.ts:1-30` |
| `@prisma/client` | 5.22.0 | ORM client, transaction API, `OrderStatus` enum, generated model types | `order.service.ts:1-6`, `order.repository.ts:1`, `order.status.ts:1`, `order.schemas.ts:2` |
| MySQL | 8.0 (`docker-compose.yml:3`) | Persistent store for orders, items, history, sequence, customers, products | Prisma datasource `prisma/schema.prisma:5-9` |
| `jsonwebtoken` | 9.0.2 | Indirect: verifies the bearer token before the module runs | `src/middlewares/auth.middleware.ts:2,41` |
| `pino` | 9.5.0 | Indirect: errors reaching the 500 path are logged | `src/middlewares/error.middleware.ts:5,57-60` |
| `uuid` | 11.0.3 | Indirect: X-Request-Id generation for the request log | `src/middlewares/request-logger.middleware.ts:2,6` |
| `vitest` | 2.1.4 | Test runner | `vitest.config.ts` |
| `supertest` | 7.0.0 | HTTP assertions in integration tests | `tests/orders.test.ts:2` |

### Dependency notes

- All imports are ES modules with explicit `.js` extensions; TypeScript is compiled to `dist` for production and run via `tsx` in development (`package.json:10-13`).
- `OrderController` imports `OrderService` and `ListOrdersQuery` as `import type`, so those edges disappear at runtime; the runtime edge is the instance injected by `app.ts`.
- The module never imports another feature module's service or repository. Its cross-module coupling to customers and products is expressed only through Prisma table names and generated types.
- `requireRole` exists in the auth middleware but is not imported by the orders router, confirming BR-03.
- `allowedTransitions` and `isTerminal` are exported by `order.status.ts` but have no importer in `src`; only `canTransition`, `shouldDebitStock` and `shouldReplenishStock` are used.

---

## 6. Afferent and Efferent Coupling

Counts are static, within `src` only, and exclude test files. "Internal" counts distinct TypeScript modules under `src`; "external" counts distinct npm packages. Type-only imports are counted because they are compile-time dependencies.

| Component | Afferent Coupling (internal dependents) | Efferent Coupling (internal modules) | Efferent Coupling (external packages) | Critical |
|-----------|------------------------------------------|--------------------------------------|----------------------------------------|----------|
| `OrderService` (class) | 2 (`OrderController` type import, `app.ts` instantiation) | 5 (`order.repository`, `order.schemas`, `order.status`, `shared/errors`, `shared/http/response`) | 1 (`@prisma/client`) | High |
| `order.status.ts` (module) | 1 (`order.service.ts`) | 0 | 1 (`@prisma/client`) | High |
| `OrderRepository` (class) | 2 (`OrderService` type import, `app.ts` instantiation) | 0 | 1 (`@prisma/client`) | Medium |
| `order.schemas.ts` (module) | 3 (`order.routes`, `OrderController` type, `OrderService` type) | 0 | 2 (`zod`, `@prisma/client`) | Medium |
| `OrderController` (class) | 2 (`buildOrderRouter` type import, `app.ts` instantiation) | 2 (`order.service` type, `order.schemas` type) plus `shared/errors` | 1 (`express` types) | Medium |
| `buildOrderRouter` (function) | 1 (`routes/index.ts`) | 4 (`auth.middleware`, `validate.middleware`, `order.schemas`, `order.controller` type) | 1 (`express`) | Low |
| `OrderStatusHistory` (data entity) | 3 data-access sites (2 writes in `OrderService`, 1 read include in `OrderRepository`) | 0 | 0 | Medium |
| `Order` aggregate (data entity) | 4 (`OrderService`, `OrderRepository`, `order.schemas` via enum, `order.status` via enum) | 0 | 0 | High |
| Orders Module (folder boundary) | 2 (`src/app.ts`, `src/routes/index.ts`) | 2 modules (`middlewares`, `shared`) plus implicit `customers`/`products` tables | 3 (`express`, `zod`, `@prisma/client`) | High |

Observations:

- `OrderService` is the module's hub: highest efferent coupling and the only component that imports five internal modules plus Prisma. It is also the only place where business rules concentrate, giving the module low internal cohesion at the service level while keeping the folder boundary cohesive around the order aggregate.
- `order.status.ts` has the cleanest dependency profile (one external enum, zero internal) and is the most stable rule holder. Its low afferent count (one) reflects that only the service consumes it; `allowedTransitions` and `isTerminal` have zero consumers.
- `order.schemas.ts` has the highest afferent coupling inside the module (3) because routes, controller and service all depend on its inferred types. It is the module's internal contract.
- `OrderRepository` is a narrow adapter with zero internal efferent edges; its coupling is entirely to the ORM. It is bypassed for cross-table access, so its low coupling does not mean low module coupling.
- The folder boundary's afferent count of 2 hides the fact that every HTTP consumer of the API depends on its route contract; static imports understate runtime/external coupling.
- The implicit coupling to `customers` and `products` does not appear as internal TypeScript edges, but it is the module's most change-sensitive dependency because those tables are owned elsewhere.

---

## 7. Endpoints

All endpoints are mounted at `/api/v1/orders` and require `Authorization: Bearer <JWT>`. All are available to any authenticated role.

| Endpoint | Method | Description | Auth | Validation | Success | Failure modes |
|----------|--------|-------------|------|------------|---------|---------------|
| `/api/v1/orders` | GET | List orders with pagination and optional filters | Bearer JWT | `listOrdersQuerySchema` (page, pageSize, status, customerId, from, to) | 200 `{ data, pagination }` | 400 `VALIDATION_ERROR`, 401 `UNAUTHORIZED` |
| `/api/v1/orders/{id}` | GET | Get one order with items, product summaries, history and customer summary | Bearer JWT | UUID id param | 200 order aggregate | 400 `VALIDATION_ERROR`, 401 `UNAUTHORIZED`, 404 `NOT_FOUND` |
| `/api/v1/orders` | POST | Create an order in `PENDING` with server-computed totals | Bearer JWT | `createOrderSchema` | 201 order aggregate | 400 `VALIDATION_ERROR`, 401 `UNAUTHORIZED`, 404 `NOT_FOUND` (customer/product), 422 `INACTIVE_PRODUCT` |
| `/api/v1/orders/{id}/status` | PATCH | Change order status and apply stock/audit side effects | Bearer JWT | UUID id param + `updateOrderStatusSchema` (`toStatus`, `reason`) | 200 refreshed order aggregate | 400 `VALIDATION_ERROR`, 401 `UNAUTHORIZED`, 404 `NOT_FOUND`, 409 `INVALID_STATUS_TRANSITION`, 422 `INSUFFICIENT_STOCK` |
| `/api/v1/orders/{id}` | DELETE | Hard-delete an order in `PENDING` or `CANCELLED` | Bearer JWT | UUID id param | 204 empty body | 400 `VALIDATION_ERROR`, 401 `UNAUTHORIZED`, 404 `NOT_FOUND`, 409 `INVALID_ORDER_STATE_FOR_DELETE` |

Request and response shapes (derived from schemas and projections):

```jsonc
// POST /api/v1/orders request
{
  "customerId": "uuid",
  "items": [{ "productId": "uuid", "quantity": 3 }],
  "discountCents": 1000,
  "notes": "optional, <= 1000 chars"
}

// Order aggregate response (create, get by id, status change)
{
  "id": "uuid",
  "orderNumber": "ORD-000001",
  "customerId": "uuid",
  "status": "PENDING",
  "subtotalCents": 18000,
  "discountCents": 1000,
  "totalCents": 17000,
  "notes": null,
  "createdById": "uuid",
  "createdAt": "ISO-8601",
  "updatedAt": "ISO-8601",
  "items": [{ "id": "uuid", "productId": "uuid", "quantity": 3,
              "unitPriceCents": 5000, "totalCents": 15000,
              "product": { "id": "uuid", "sku": "SKU", "name": "Name" } }],
  "history": [{ "id": "uuid", "fromStatus": null, "toStatus": "PENDING",
                "changedAt": "ISO-8601", "changedById": "uuid",
                "reason": "order created" }],
  "customer": { "id": "uuid", "name": "Name", "email": "email" }
}

// GET /api/v1/orders response
{
  "data": [ /* flat order rows without items/history/customer */ ],
  "pagination": { "page": 1, "pageSize": 20, "total": 42, "totalPages": 3 }
}

// Error envelope
{ "error": { "code": "INSUFFICIENT_STOCK", "message": "...", "details": { } } }
```

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 via Prisma | Database | Persist orders, items, history, sequence; read/write customers and products | MySQL wire protocol through Prisma | Relational rows; Prisma model mapping; JSON address column unused by orders | Domain errors thrown before/inside transaction; Prisma P2002 mapped to 409, P2025 to 404, others to logged 500; transaction rollback on throw |
| Authentication middleware | Internal in-process middleware | Convert bearer JWT into `req.user` for every order route | Function call; JWT HS256 verification | `AuthUser { id, email, role }` attached to request | 401 `UNAUTHORIZED` for missing, malformed or expired token |
| Validation middleware | Internal in-process middleware | Parse and coerce params/query/body before controller | Function call with Zod schemas | Replaces `req.body`/`req.params`, merges `req.query`; errors as `{ path, message }[]` | 400 `VALIDATION_ERROR` with field details |
| Centralized error middleware | Internal in-process middleware | Serialize domain and infrastructure failures | Express error pipeline (`next(err)`) | JSON envelope `{ error: { code, message, details? } }` | Maps `AppError`, `ZodError`, Prisma P2002/P2025; logs unknown errors with request id |
| Pagination helper | Internal shared function | Build list envelope and `totalPages` | Function call | `{ data, pagination }` | No error path; guards `pageSize === 0` defensively |
| Request logger | Internal middleware | Assign/propagate `X-Request-Id`, log completion with user id and duration | Express middleware | Pino JSON logs | Non-blocking; not involved in response semantics |
| Order-number sequence | Internal database table | Allocate monotonic `ORD-` numbers | SQL via Prisma upsert | Integer `nextValue` on singleton row `id = 1` | Transaction rollback on failure; unique constraint on `orderNumber` as backstop |
| Seed script | Development tooling | Populate demo orders and history; truncate tables | Prisma direct writes | Same schema, duplicated numbering algorithm | Aborts on any Prisma error; destructive delete order respects FKs |
| Test harness | Test infrastructure | Integration tests against real MySQL via supertest | HTTP over in-process Express app | JSON requests/responses; direct Prisma assertions | Test failure surfaces assertion mismatch; DB truncation before each test |

Integration characteristics:

- There is no external payment, shipping, messaging or webhook integration in the current module. All side effects are local database writes.
- The module has no outbox, event emitter or message broker. Status changes do not publish anything; consumers must poll `GET /orders` or `GET /orders/:id`.
- Customer and product modules are integrated only at the table level, not through service calls (BR-42).
- The 1 MB JSON body limit in `src/app.ts:59` applies to order creation payloads; a large `items` array or `notes` string is bounded by that limit in addition to schema limits.
- HTTP CORS, compression, rate limiting and authentication cookies are absent at the application level; the module inherits that.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered feature module | routes -> controller -> service -> repository per folder | `src/modules/orders/` | Separate transport, orchestration and persistence concerns |
| Thin controller / transport adapter | Handlers only call service and set status codes; `next(err)` for failures | `order.controller.ts:9-55` | Keep HTTP concerns out of domain logic |
| Constructor injection / composition root | `OrderRepository`, `OrderService`, `OrderController` constructed in `buildControllers` and injected | `src/app.ts:42-44` | Explicit wiring; no service locator or decorators |
| Repository pattern (partial) | `OrderRepository` encapsulates order list/count/find/delete | `order.repository.ts:18-68` | Abstract order read/delete queries; bypassed for cross-table access |
| Service layer / transaction script | `OrderService` sequences validation, pricing, persistence, stock and audit inside transactions | `order.service.ts:26-255` | Centralize business rules and atomicity |
| Table-driven state machine | Read-only transition record plus pure predicates | `order.status.ts:3-37` | Single source of truth for lifecycle legality and stock policy |
| Unit of Work via Prisma interactive transaction | `$transaction(async (tx) => ...)` with shared `tx` client | `order.service.ts:58,131` | Atomic order + items + history + stock + sequence changes |
| DTO / schema validation | Zod schemas inferred into TypeScript input types | `order.schemas.ts:4-34` | Runtime transport validation and compile-time types |
| Error taxonomy | `AppError` hierarchy with status code, error code, details | `src/shared/errors/` | Uniform failure contract consumed by middleware |
| Guard clauses / fail-fast | Early throws for missing rows, invalid state, discount and stock | `order.service.ts:51-91,136-149,184-190,222-224` | Keep happy path linear; avoid nested conditionals |
| Pagination envelope | `PaginatedResponse` helper used by list | `src/shared/http/response.ts` | Consistent list contract across modules |
| Audit trail / event-log style history | Append-only `OrderStatusHistory` with actor and reason | `order.service.ts:106-113,159-167`; `prisma/schema.prisma:116-131` | Reconstruct lifecycle changes |
| Sequence table (singleton counter) | Upsert on `order_number_sequence.id = 1` inside creation transaction | `order.service.ts:245-254` | Gap-free, serialized human-readable numbering |
| Snapshot pricing | `unitPriceCents` copied to `OrderItem` at creation | `order.service.ts:75-84`; `prisma/schema.prisma:104` | Preserve price at order time despite catalogue changes |
| Null-object style initial history | `fromStatus: null` represents creation | `prisma/schema.prisma:119`; `order.service.ts:108` | Allow a uniform history model for the first state |

Architectural decisions observed (not evaluated):

- The module is a vertical slice with its own schemas and repository; it does not share DTOs with other modules.
- Domain rules are implemented as functions and service methods, not as rich entities; `Order` is a Prisma data type, not a class with behavior.
- Transactions are owned by the service, not by the repository, so a single service method can coordinate multiple tables.
- Read models are not separated from write models: the same `Order` type and Prisma client serve both, with different `include` projections.
- The state machine is code, not configuration or database data; it is not exposed through an API.
- Cross-module data access is deliberate enough to be repeated in three places (customer, product, sequence) rather than accidental in one helper.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact | Evidence |
|------------|----------------|-------|--------|----------|
| High | Inventory | No stock validation or reservation at creation; stock is checked only at `PENDING -> PAID` | Orders can be accepted and paid for against unavailable stock; oversell is detected late and only for the first payer | `order.service.ts:50-124`; `tests/orders.test.ts:109-119` |
| High | Concurrency | No row lock, `version` column or conditional update on status or stock | Concurrent transitions can double-apply stock and write duplicate history; stock can go negative | `order.service.ts:132-167,204-231`; `prisma/schema.prisma:74-114` |
| High | Idempotency | No idempotency key or client reference on create; no replay protection | Client retries create duplicate orders and consume order numbers | `order.routes.ts:18`; `order.service.ts:95-120` |
| Medium | Numbering | Every creation serializes on one `order_number_sequence` row | Contention and lock waits under concurrent checkout; numbering is a single point of failure | `order.service.ts:245-254`; `prisma/schema.prisma:133-138` |
| Medium | Architecture | Direct Prisma access to `customers`, `products`, `order_number_sequence` | Bypasses repository contracts; schema changes in other domains propagate into order logic | `order.service.ts:59-73,93,204-254` |
| Medium | Data integrity | Status transitions, discount/subtotal arithmetic and non-negative stock are not enforced in MySQL | Out-of-band SQL or future code paths can create illegal states; no database-level safety net | `prisma/migrations/20260519182739_init/migration.sql:44-105` |
| Medium | Authorization | All authenticated roles can create, transition and delete any order | Any operator token can confirm payment, cancel orders or delete records; no separation of duties | `order.routes.ts:14`; no `requireRole` |
| Medium | Concurrency | Delete guard is read-then-delete outside a transaction | Status can change between check and delete; deletion may proceed on stale state | `order.service.ts:181-192` |
| Medium | Numeric bounds | `quantity` has no maximum; money columns are 32-bit signed integers | Large quantity x price can overflow `INTEGER`, producing an unmapped 500 instead of a validation error | `order.schemas.ts:8`; `prisma/schema.prisma:79-81,104-105` |
| Medium | Transaction limits | Prisma interactive transaction defaults (5 s timeout, 2 s max wait) are not configured or handled | Slow products/stock operations or lock contention surface as generic 500s, not domain errors | `order.service.ts:58,131` |
| Medium | Test coverage | No unit tests for `order.status.ts` or the service; several branches untested | Regressions in transition or stock predicates are caught only if an HTTP test covers the exact path | `tests/orders.test.ts` (8 tests); no other order tests |
| Low | Read consistency | List returns flat orders; detail returns the full aggregate | Clients need one detail call per order and cannot see item counts or totals composition in list results | `order.repository.ts:33-45` vs `47-60` |
| Low | Auditing | Hard delete cascades away status history; deletion actor is not recorded | Audit trail for deleted orders is unrecoverable; no record of who deleted | `prisma/schema.prisma:108,125`; `order.service.ts:181-192` |
| Low | Error semantics | Same-status and illegal transition share `INVALID_STATUS_TRANSITION`; unique order-number collision maps to generic `CONFLICT` | Clients cannot distinguish retry-safe conflicts from illegal requests by code alone | `order.service.ts:140-149`; `error.middleware.ts:37-47` |
| Low | Date filtering | `to` is inclusive at the parsed instant; date-only values mean midnight; no `from <= to` validation | End-of-day orders can be excluded; inverted ranges silently return empty results | `order.schemas.ts:28-29`; `order.repository.ts:25-29` |
| Low | Dead API surface | `allowedTransitions` and `isTerminal` are exported but unused; `STOCK_DEBIT_TRANSITION` is only used internally | Untested exports can drift from the table without detection | `order.status.ts:16-27` |
| Low | Type safety at boundary | Controller uses `req.params.id!` and `req.query as unknown as ListOrdersQuery` | Validated values are asserted, not proven; future middleware changes could break handlers silently | `order.controller.ts:11,21,41,50` |
| Low | Documentation | No OpenAPI/Swagger file; route contract exists only in code and Zod schemas | External consumers must read source; contract drift is unverifiable | No API spec file in repository |
| Low | Hard delete lifecycle | Deleting `CANCELLED` orders removes the commercial record of paid-then-cancelled orders | Loss of historical reporting data and audit evidence | `order.service.ts:184-191`; `prisma/schema.prisma:108,125` |
| Low | Numbering format | `padStart(6)` does not cap; over 999999 the format grows to seven digits | Cosmetic/format contract drift; still fits `VARCHAR(20)` | `order.service.ts:253`; `prisma/schema.prisma:76` |
| Low | Duplicate implementation | Seed script reimplements order-number reservation | Two algorithms can diverge if the production format changes | `prisma/seed.ts:165-177`; `order.service.ts:245-254` |
| Low | Test environment | Integration tests require a live MySQL instance; no in-memory or mocked repository tests | Tests are environment-dependent and slower; pure rule modules are not tested in isolation | `tests/setup.ts:4-20`; `tests/helpers/factories.ts:10-15` |

---

## 11. Test Coverage Analysis

### Test inventory

| Test file | Type | Tests | Scope | Database |
|-----------|------|-------|-------|----------|
| `tests/orders.test.ts` | HTTP integration (supertest + real Express app) | 8 `it` blocks | Orders module end-to-end through routes, middleware, controller, service, repository and Prisma | Real MySQL 8.0; tables truncated in `beforeEach` |
| `tests/helpers/factories.ts` | Test fixtures (no assertions) | 5 helpers | App composition, user/customer/product creation, login/token bootstrap | Real MySQL |
| `tests/setup.ts` | Test lifecycle | 3 hooks | Connect before all, truncate 7 tables before each, disconnect after all | Real MySQL |
| `tests/auth.test.ts` | HTTP integration (adjacent) | Not order-specific | Provides login via `/api/v1/auth/login` used by `bootstrapAuthenticatedUser` | Real MySQL |

No unit test file exists for `order.status.ts`, `OrderService`, `OrderRepository` or the validation schemas. No test uses mocks, stubs or an in-memory database. `vitest.config.ts` defines no coverage provider, reporter or thresholds, so no line/branch coverage percentage is produced by the configured tooling.

### Test-by-test behavior coverage

| # | Test (line) | Behavior asserted | Business rules exercised |
|---|-------------|-------------------|--------------------------|
| 1 | `tests/orders.test.ts:12` | Create returns 201, `PENDING`, server-computed subtotal/discount/total, 2 items, 1 history row, `ORD-\d{6}` number | BR-05, BR-09, BR-10, BR-11 (implicitly), BR-15, BR-17, BR-18, BR-19, BR-21 |
| 2 | `tests/orders.test.ts:42` | Non-existent product returns 404 `NOT_FOUND` | BR-13, BR-38 |
| 3 | `tests/orders.test.ts:59` | `PENDING -> PAID` returns 200, status `PAID`, history `[PENDING, PAID]`, stock 5 -> 2 for quantity 3 | BR-28, BR-31, BR-34, BR-35 |
| 4 | `tests/orders.test.ts:89` | `PENDING -> SHIPPED` returns 409 `INVALID_STATUS_TRANSITION` | BR-28, BR-30, BR-38 |
| 5 | `tests/orders.test.ts:109` | Create succeeds with quantity 5 against stock 1 (201), then `PAID` returns 422 `INSUFFICIENT_STOCK` with SKU, stock unchanged | BR-23, BR-31, BR-38 |
| 6 | `tests/orders.test.ts:134` | `PAID -> CANCELLED` replenishes stock from 6 to 10 after quantity-4 debit | BR-28, BR-32, BR-35 |
| 7 | `tests/orders.test.ts:163` | List filtered by `status=PAID` returns 1 row; `status=PENDING` returns 2 rows and `pagination.total = 2` | BR-25, BR-26, BR-09 |
| 8 | `tests/orders.test.ts:197` | Delete of a `PAID` order returns 409 `INVALID_ORDER_STATE_FOR_DELETE` | BR-36, BR-38 |

### Endpoint and rule coverage matrix

| Area | Covered | Not covered |
|------|---------|-------------|
| `POST /orders` success | Total arithmetic, item persistence, initial history, order-number format | `notes`, discount 0 default, duplicate product aggregation, single-item order, large item count |
| `POST /orders` failure | Missing product 404 | Missing customer 404, inactive product 422, discount > subtotal 400, empty items 400, invalid UUID 400, notes > 1000, unauthenticated 401 |
| `GET /orders` | Status filter, `pagination.total` | `page`/`pageSize` defaults, max 100, invalid values, `customerId` filter, `from`/`to` filters, ordering, `totalPages`, empty page, unauthenticated 401 |
| `GET /orders/:id` | None | 200 aggregate shape, 404 missing, 400 malformed UUID, unauthenticated 401 |
| `PATCH /orders/:id/status` | `PENDING -> PAID` debit, `PAID -> CANCELLED` replenish, `PENDING -> SHIPPED` 409, insufficient stock 422 | Same-status 409, `PAID -> PROCESSING`, `PROCESSING -> SHIPPED`, `SHIPPED -> DELIVERED`, `PENDING -> CANCELLED`, `PROCESSING -> CANCELLED` replenish, `DELIVERED`/`CANCELLED` terminal rejection, reason persistence, reason > 500, missing order 404, malformed UUID, unauthenticated 401 |
| `DELETE /orders/:id` | 409 for `PAID` | Successful 204 for `PENDING`, successful 204 for `CANCELLED`, 404 missing, malformed UUID, unauthenticated 401, cascade verification, stock-after-delete |
| State machine unit level | Indirectly through two legal and one illegal path | All 36 ordered pairs; `allowedTransitions`; `isTerminal`; `shouldDebitStock`/`shouldReplenishStock` truth tables |
| Concurrency | None | Parallel `PATCH` requests, parallel `POST` numbering, delete-vs-status race |
| Data constraints | Implicit via MySQL | Overflow behavior, out-of-band status writes, negative stock |

### Qualitative assessment

| Aspect | Assessment | Evidence |
|--------|------------|----------|
| Test level | End-to-end HTTP integration only; no isolated unit or repository tests | `tests/orders.test.ts`; `tests/helpers/factories.ts` |
| Assertion quality | Strong on computed totals, status, history sequence and stock deltas; uses direct Prisma reads to verify side effects | `tests/orders.test.ts:33-39,80-86,128-131,150-160` |
| Failure-path depth | Three domain failure modes covered (missing product, illegal transition, insufficient stock, delete conflict) | Tests 2, 4, 5, 8 |
| Edge cases | Absent for validation boundaries, optional fields, pagination metadata, successful delete and duplicate aggregation | Matrix above |
| Isolation | Each test uses unique generated emails/SKUs and truncates tables in `beforeEach`, so tests are independent but share one database | `tests/setup.ts:8-16`; `tests/helpers/factories.ts:24,67` |
| Determinism | Order listing relies on `createdAt desc`; tests avoid asserting order of the three created orders except by filtering status | `tests/orders.test.ts:163-195` |
| Environment dependence | Requires MySQL reachable through `DATABASE_URL`; no test container is started by the test config | `tests/setup.ts:4-6`; `vitest.config.ts` |
| Coverage measurement | Not configured; no coverage provider in `vitest.config.ts` or `package.json` | `vitest.config.ts:3-18`; `package.json:35-53` |
| Missing regression net | Status machine exports and most transition pairs have no direct tests | `order.status.ts:16-22` unused and untested |

### Test file locations

- Orders tests: `tests/orders.test.ts`
- Shared fixtures and app bootstrap: `tests/helpers/factories.ts`
- Global setup/teardown: `tests/setup.ts`
- Adjacent auth integration tests used for token bootstrap: `tests/auth.test.ts`
- Runner configuration: `vitest.config.ts` (Node environment, `tests/**/*.test.ts`, 30 s timeouts, single fork, no parallelism)

---

## 12. Report Metadata

| Field | Value |
|-------|-------|
| Component analyzed | Orders Module |
| Primary path | `src/modules/orders/` |
| Files in boundary | 6 TypeScript files, 478 lines combined |
| Endpoints | 5 |
| Business rules extracted | 42 |
| Integration tests located | 8 order tests in `tests/orders.test.ts` |
| Coverage measurement | Not available (no coverage provider configured) |
| Analysis mode | Read-only; no project files modified |
