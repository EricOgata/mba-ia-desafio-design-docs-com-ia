# Component Deep Analysis Report: Order Status History

**Component:** Order Status History (Prisma audit model `OrderStatusHistory` / table `order_status_history`)

**Analysis date:** 2026-11-09 09:26:46 (UTC-03)

**Scope:** The audit-trail concern only: schema shape, indexes, relation semantics, write sites, read/ordering path, retention, and immutability guarantees. The full Prisma data model and the `OrderService` behavior are covered by sibling analyses and are referenced here only where required to explain the audit trail.

**Source files examined:**

- `prisma/schema.prisma` (`OrderStatusHistory`, lines 116-131; related `Order`, `User`, `OrderStatus`)
- `prisma/migrations/20260519182739_init/migration.sql` (table lines 87-99; indexes 96-97; foreign keys 122-125)
- `prisma/seed.ts` (lines 181, 307-320)
- `src/modules/orders/order.service.ts` (lines 95-120, 126-179)
- `src/modules/orders/order.repository.ts` (lines 1-16, 47-60)
- `src/modules/orders/order.controller.ts` (lines 19-46)
- `src/modules/orders/order.routes.ts` (lines 12-27)
- `src/modules/orders/order.schemas.ts` (lines 18-21)
- `src/modules/orders/order.status.ts` (lines 3-37)
- `src/middlewares/auth.middleware.ts` (lines 27-47)
- `src/middlewares/error.middleware.ts` (lines 14-65)
- `tests/setup.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `vitest.config.ts`, `package.json`

**Exclusions applied:** `node_modules`, `.git`, `dist`, `coverage`.

**Analysis constraints:** No files were modified. No tests were executed. No database was queried. All findings are derived from static inspection of the repository.

---

## 1. Executive Summary

`OrderStatusHistory` is a Prisma persistence model that implements the application's only audit trail: an append-only journal of order status transitions. It has no dedicated TypeScript module, service, repository, controller, or endpoint. It is a data component whose lifecycle is entirely controlled by two write sites in `OrderService` and one read include in `OrderRepository`.

The model records, per row: an order reference, an optional previous status, the resulting status, a database-assigned timestamp, the acting user, and an optional free-text reason. The initial row is written during order creation with `fromStatus = null`, `toStatus = PENDING`, and the fixed reason `order created`. Every subsequent successful status change appends one row with the previous and next status. Failed or rejected transitions do not produce rows.

The audit trail is atomic with the state it describes. Creation uses a Prisma nested write inside `prisma.$transaction` (`src/modules/orders/order.service.ts:58-123`), and status changes update the order and insert the history row in the same interactive transaction (`src/modules/orders/order.service.ts:131-178`). A rollback removes both the status change and its audit row. There is no production code path that creates an order without its initial history row, and no production code path that updates or deletes a history row.

The read path is embedded in order projections rather than exposed as a first-class resource. History is eager-loaded with `orderBy: { changedAt: 'asc' }` in three places: creation response (`order.service.ts:117`), status-change response (`order.service.ts:173`), and get-by-id (`order.repository.ts:56`). The list endpoint does not include history. There is no `GET /orders/:id/history` endpoint, no pagination, no date filtering, and no actor join.

Immutability is a convention, not an enforced property. The database has no trigger, permission, hash chain, signature, unique constraint, or check constraint preventing updates or deletes. The Prisma model exposes the full CRUD surface, and the only retention control is `ON DELETE CASCADE` from `order_status_history.orderId` to `orders.id`. Deleting an order physically deletes all of its audit rows. The service restricts order deletion to `PENDING` or `CANCELLED`, but that still destroys one to four rows per deleted order, and no deletion event is audited.

Key findings:

- **Two production write sites, one read include.** No dedicated history abstraction exists; all writes bypass the repository layer and use `tx.orderStatusHistory.create` or a nested `history.create`.
- **Schema is append-only shaped but not append-only enforced.** No `updatedAt`, no soft delete, no version, no write protection.
- **Audit completeness is partial.** Only successful status transitions and order creation are recorded. Rejected transitions, stock failures, order deletion, field changes, and read access are not recorded.
- **Actor attribution is reliable but shallow.** `changedById` is taken from the verified JWT subject and protected by a `RESTRICT` foreign key, but the row stores no actor name, role, IP, request id, or authorization decision.
- **Ordering has no tie-breaker.** `ORDER BY changedAt ASC` alone; `DATETIME(3)` collisions can produce nondeterministic order.
- **Concurrency can duplicate audit rows.** `changeStatus` reads the order without a row lock or optimistic version check; two concurrent requests can both append a row for the same transition.
- **Retention is destructive.** `ON DELETE CASCADE` removes history with the parent order; no archive, tombstone, or deletion record exists.
- **Read access is broad.** All order routes require only `authenticate`, not `requireRole` or ownership scoping, so any authenticated `ADMIN` or `OPERATOR` can read any order's audit trail.
- **Indexing is minimal.** Single-column indexes on `orderId` and `changedAt`; no composite index for the actual read pattern `WHERE orderId = ? ORDER BY changedAt ASC`.
- **Test coverage is thin.** Two integration assertions cover the presence and `toStatus` sequence of history; actor, reason, timestamps, ordering ties, no-write-on-failure, cascade deletion, immutability, and concurrency are untested.

---

## 2. Data Flow Analysis

### 2.1 Write path 1: order creation (initial audit row)

```
1. POST /api/v1/orders reaches order.routes.ts:18 with authenticate at line 14.
2. validate(createOrderSchema) accepts customerId, items, discountCents, notes (order.schemas.ts:11-16).
3. OrderController.create reads req.user.id and calls OrderService.create(input, userId) (order.controller.ts:28-32).
4. OrderService.create opens prisma.$transaction (order.service.ts:58).
5. Customer, products, stock-independent pricing and discount checks run inside the transaction (order.service.ts:59-92).
6. tx.order.create is called with a nested history.create (order.service.ts:95-120).
7. Prisma emits INSERT into orders, INSERT into order_items, and INSERT into order_status_history.
8. The history row receives fromStatus = NULL, toStatus = PENDING, changedById = JWT user id, reason = 'order created' (order.service.ts:106-113).
9. changedAt is not supplied; MySQL applies DEFAULT CURRENT_TIMESTAMP(3) (migration.sql:92).
10. The transaction commits; the response includes history ordered by changedAt ASC (order.service.ts:115-119).
```

The initial row is part of the same Prisma nested write as the order and item inserts. There is no application-level step that can commit the order without the history row; if the nested history insert fails, the entire order insert fails.

### 2.2 Write path 2: status change (transition audit row)

```
1. PATCH /api/v1/orders/:id/status reaches order.routes.ts:19-23 with authenticate and validation.
2. updateOrderStatusSchema accepts toStatus and optional reason max 500 (order.schemas.ts:18-21).
3. OrderController.changeStatus reads req.user.id and calls OrderService.changeStatus(id, body, userId) (order.controller.ts:38-42).
4. OrderService.changeStatus opens prisma.$transaction (order.service.ts:131).
5. tx.order.findUnique loads the order and items without a row lock (order.service.ts:132-135).
6. Missing order -> 404; same status -> 409; illegal transition -> 409 (order.service.ts:136-149).
7. Stock debit or replenish is applied when the transition predicate matches (order.service.ts:151-156).
8. tx.order.update writes the new status (order.service.ts:158).
9. tx.orderStatusHistory.create appends fromStatus = prior status, toStatus = requested status, changedById = JWT user id, reason = input.reason ?? null (order.service.ts:159-167).
10. tx.order.findUnique re-reads the aggregate with history ordered by changedAt ASC (order.service.ts:169-176).
11. The transaction commits; the response returns the refreshed order with the complete history.
```

The order row update and the audit row insert are ordered and atomic. If the history insert fails, the status update is rolled back; if the status update fails, no history row is created. The `fromStatus` value comes from the order row read at the start of the transaction, not from a client field.

### 2.3 Read path

```
1. GET /api/v1/orders/:id reaches order.routes.ts:17.
2. OrderController.getById calls OrderService.getById (order.controller.ts:19-26).
3. OrderService.getById calls OrderRepository.findByIdWithRelations (order.service.ts:44-48).
4. Repository issues order.findUnique with include.history orderBy changedAt ASC (order.repository.ts:47-60).
5. Prisma returns all scalar OrderStatusHistory fields: id, orderId, fromStatus, toStatus, changedAt, changedById, reason.
6. The order aggregate is serialized as the HTTP response.
```

History is also returned by the create and change-status endpoints through their own include clauses (`order.service.ts:117,173`). It is not returned by the list endpoint: `OrderRepository.list` selects bare `Order` rows (`order.repository.ts:33-45`). No endpoint filters, paginates, aggregates, or joins history with users.

### 2.4 Deletion and retention flow

```
1. DELETE /api/v1/orders/:id reaches order.routes.ts:24.
2. OrderController.delete calls OrderService.delete (order.controller.ts:48-54).
3. Service loads the order and permits deletion only for PENDING or CANCELLED (order.service.ts:181-190).
4. OrderRepository.deleteById issues prisma.order.delete (order.repository.ts:66-68).
5. MySQL applies ON DELETE CASCADE from order_status_history.orderId (migration.sql:122).
6. All history rows for that order are physically removed.
7. No audit row, tombstone, or deletion record is written anywhere.
```

### 2.5 Auxiliary write/delete paths (development and test only)

```
Seed:
1. prisma/seed.ts:181 deletes all order_status_history rows first.
2. For each seeded order, a status path is derived from flowFromStatus (seed.ts:255-262).
3. prisma.orderStatusHistory.create is called once per path element (seed.ts:307-320).
4. Each seed insert is independent; there is no wrapping transaction.

Tests:
1. tests/setup.ts:9 calls prisma.orderStatusHistory.deleteMany() before every test.
2. tests/orders.test.ts reads history only through HTTP responses.
3. tests/helpers/factories.ts creates users, customers and products but no history fixtures.
```

These paths are not production writes, but they demonstrate that the model is fully mutable and deletable through the Prisma client.

### 2.6 Consolidated flow diagram

```
POST /orders
  -> OrderService.create (tx)
     -> INSERT orders
     -> INSERT order_items
     -> INSERT order_status_history (NULL -> PENDING)

PATCH /orders/:id/status
  -> OrderService.changeStatus (tx)
     -> SELECT order + items (no lock)
     -> validate transition
     -> UPDATE orders.status
     -> INSERT order_status_history (from -> to)

GET /orders/:id
  -> OrderRepository.findByIdWithRelations
     -> SELECT order
     -> SELECT items + product summary
     -> SELECT order_status_history ORDER BY changedAt ASC
     -> SELECT customer summary

DELETE /orders/:id
  -> DELETE orders
     -> CASCADE DELETE order_status_history
     -> no audit record
```

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location |
|---|---|---|---|
| BR-H01 | Audit creation | Order creation writes exactly one initial history row: `fromStatus = NULL`, `toStatus = PENDING`, fixed reason `order created` | `src/modules/orders/order.service.ts:106-113` |
| BR-H02 | Audit write | Each successful status change appends one row with the prior and new status | `src/modules/orders/order.service.ts:158-167` |
| BR-H03 | Audit scope | Only successful transitions are recorded; rejected and failed attempts leave no row | `order.service.ts:136-156`; `error.middleware.ts:14-65` |
| BR-H04 | Atomicity | History writes occur in the same transaction as the state change they describe | `order.service.ts:58-123,131-178` |
| BR-H05 | Attribution | `changedById` is the authenticated JWT subject, never a request body field, and is FK-constrained to `users` | `order.controller.ts:30-31,40-41`; `auth.middleware.ts:41-42`; `schema.prisma:122,126` |
| BR-H06 | State origin | `fromStatus` is nullable only to represent creation; no `CREATED` enum value exists | `schema.prisma:119`; `order.service.ts:108` |
| BR-H07 | Reason field | `reason` is optional, free text, max 500 characters, stored as NULL when absent; creation uses a fixed literal | `order.schemas.ts:20`; `order.service.ts:111,165`; `schema.prisma:123` |
| BR-H08 | Timestamp | `changedAt` is assigned by the database default; clients cannot set it through the API; no `updatedAt` exists | `schema.prisma:121`; `migration.sql:92` |
| BR-H09 | Ordering | History is read in ascending `changedAt` order with no secondary tie-breaker | `order.service.ts:117,173`; `order.repository.ts:56` |
| BR-H10 | Read surface | History has no standalone endpoint; it is embedded in order responses, and the list endpoint excludes it | `order.routes.ts:16-24`; `order.repository.ts:33-60` |
| BR-H11 | Immutability | Append-only by application convention only; no database-level protection against UPDATE or DELETE | `schema.prisma:116-131`; absence of update/delete in `src` |
| BR-H12 | Retention | History is physically deleted by cascade when its order is deleted | `schema.prisma:125`; `migration.sql:122`; `order.repository.ts:66-68` |
| BR-H13 | Referential integrity | A user referenced by history cannot be deleted while the row exists (`ON DELETE RESTRICT`) | `schema.prisma:126`; `migration.sql:125` |
| BR-H14 | Audit completeness | Only status transitions and creation are audited; field changes, stock movements, deletion, failed attempts and reads are not | `schema.prisma:116-131`; `order.service.ts` |
| BR-H15 | Uniqueness | No unique or idempotency constraint exists on history rows; duplicates are possible | `schema.prisma:128-130`; `migration.sql:96-97` |
| BR-H16 | Enum domain | `fromStatus` and `toStatus` are restricted to the six `OrderStatus` values, but transition legality is enforced only in application code | `schema.prisma:16-23,119-120`; `order.status.ts:3-14` |
| BR-H17 | Chain consistency | Cross-row consistency (`toStatus` of row N = `fromStatus` of row N+1) is not enforced or validated | `schema.prisma:116-131`; `order.service.ts:159-167` |

---

### Business Rule: BR-H01 - Initial audit entry on order creation

**Overview**:
Every order created through the API receives exactly one initial `OrderStatusHistory` row describing the transition from no state to `PENDING`.

**Location**: `src/modules/orders/order.service.ts:106-113`
**Confidence**: High

**Detailed description**:
`OrderService.create` builds a single `tx.order.create` payload that contains `items.create` and `history.create` (`src/modules/orders/order.service.ts:95-120`). The history literal fixes `fromStatus: null`, `toStatus: OrderStatus.PENDING`, `changedById: userId`, and `reason: 'order created'`. Because the history insert is nested in the order insert, Prisma executes it as part of the same write operation. The order's initial `status` is also explicitly set to `OrderStatus.PENDING` (`order.service.ts:99`), so the materialized order state and the initial audit row agree at creation time.

The `fromStatus` null is the only representation of order creation in the audit model. The `OrderStatus` enum has no `CREATED` member (`prisma/schema.prisma:16-23`), so the first row is distinguishable from all later rows only by its null `fromStatus` and by its fixed reason. No client-supplied field can alter the initial row: `createOrderSchema` has no status, history, actor, or reason field (`order.schemas.ts:11-16`), and `userId` is passed separately by the controller from `req.user.id`.

The rule is absolute for the production HTTP path: there is no alternative production creator. The seed script writes its own history separately and can omit the initial row or use different reason values (`prisma/seed.ts:307-320`), but the seed is development tooling and is not part of the API path. The initial row is inserted without an explicit `changedAt`; the database default supplies the timestamp.

**Rule workflow**:
```
1. OrderService.create receives validated input and userId.
2. A Prisma transaction starts.
3. tx.order.create is called with nested items and nested history.
4. History values: fromStatus = NULL, toStatus = PENDING, changedById = userId, reason = 'order created'.
5. MySQL assigns changedAt = CURRENT_TIMESTAMP(3).
6. Transaction commits order, items, and history together.
7. The create response includes the single initial history row.
```

---

### Business Rule: BR-H02 - Transition audit entry on status change

**Overview**:
Each successful status change appends exactly one history row describing the previous status and the new status.

**Location**: `src/modules/orders/order.service.ts:158-167`
**Confidence**: High

**Detailed description**:
`OrderService.changeStatus` reads the order inside a transaction and captures `const from = order.status` (`order.service.ts:138`). After `canTransition(from, to)` and any stock side effect pass, it calls `tx.order.update` to set the new status (`order.service.ts:158`) and then `tx.orderStatusHistory.create` with `orderId: id`, `fromStatus: from`, `toStatus: to`, `changedById: userId`, and `reason: input.reason ?? null` (`order.service.ts:159-167`). The pair therefore records both the source and destination of the transition, not only the resulting status.

The order update precedes the history insert. Because both statements are in the same interactive transaction, the history insert failing rolls back the status update, and the status update failing prevents the history insert. The audit row does not carry the stock delta or any other transaction detail; it carries only the status pair, actor, timestamp, and reason.

The only caller of `changeStatus` is `OrderController.changeStatus` (`order.controller.ts:38-46`), and the only route is `PATCH /api/v1/orders/:id/status` (`order.routes.ts:19-23`). There is no bulk transition path and no other production writer of transition rows.

**Rule workflow**:
```
1. Transaction starts.
2. Read order and capture from = order.status.
3. Validate toStatus against the state machine.
4. Apply stock debit or replenish if the pair requires it.
5. UPDATE orders SET status = to.
6. INSERT order_status_history (orderId, fromStatus = from, toStatus = to, changedById, reason).
7. Re-read order with history.
8. Commit.
```

---

### Business Rule: BR-H03 - Only successful transitions are recorded

**Overview**:
The audit trail records successful state changes only; a rejected, invalid, or failed transition produces no history row.

**Location**: `src/modules/orders/order.service.ts:136-156`; `src/middlewares/error.middleware.ts:14-65`
**Confidence**: High

**Detailed description**:
Three guards run before the history insert. A missing order throws `NotFoundError` before any write (`order.service.ts:136-137`). A transition to the same status throws `ConflictError` with code `INVALID_STATUS_TRANSITION` (`order.service.ts:140-146`). A transition not present in the state machine throws `InvalidStatusTransitionError` (`order.service.ts:147-149`). Stock insufficiency throws `InsufficientStockError` inside `debitStock` before the order update and history insert (`order.service.ts:151-153,204-231`). In all four cases the interactive transaction aborts and no audit row is inserted.

This behavior means the audit trail is a record of what happened, not a record of what was attempted. A caller who repeatedly tries an illegal transition leaves no trace. A caller who attempts a valid transition but cannot obtain stock leaves no trace. The only evidence of those attempts is the application log emitted by the request logger and, for handled errors, no dedicated audit entry. The centralized error middleware maps the thrown `AppError` to a JSON error response and does not persist anything (`error.middleware.ts:15-24`).

The rule is a deliberate consequence of the write ordering: the audit insert is the last step of a successful transition and is never reached on the failure branches. It also means the audit trail cannot be used to detect abuse patterns such as repeated forbidden transition attempts.

**Rule workflow**:
```
1. Read order.
   -> not found: 404, no history.
2. Compare from and to.
   -> equal: 409, no history.
3. Check canTransition(from, to).
   -> false: 409, no history.
4. Apply stock side effect.
   -> insufficient stock: 422 and rollback, no history.
5. Update order and insert history.
6. Commit.
```

---

### Business Rule: BR-H04 - Audit writes are atomic with the state change

**Overview**:
A history row and the status change it describes commit or roll back together.

**Location**: `src/modules/orders/order.service.ts:58-123,131-178`
**Confidence**: High

**Detailed description**:
Both production write paths use `this.prisma.$transaction(async (tx) => ...)`. Creation wraps the order, items, and nested history in one interactive transaction (`order.service.ts:58-123`). Status change wraps the read, validation, stock adjustment, order update, history insert, and refresh in one interactive transaction (`order.service.ts:131-178`). There is no code that writes history outside the transaction that owns the corresponding order mutation.

The practical effect is that the audit trail cannot contain a row for a state change that did not commit, and a committed state change cannot lack its audit row through the service. If the history insert fails because of a foreign-key error, constraint error, or connection failure, Prisma aborts the transaction and the order update is undone. If the order update fails, the history insert is never attempted.

The seed script is the exception: it calls `prisma.orderStatusHistory.create` in a loop with no transaction (`prisma/seed.ts:307-320`). A failure mid-path can leave a partially reconstructed history. The test setup also deletes history outside any application transaction (`tests/setup.ts:9`). These are development and test paths, not the production command path.

**Rule workflow**:
```
1. BEGIN transaction.
2. Perform all order mutations and the history insert.
3. If any statement fails -> ROLLBACK everything.
4. If all statements succeed -> COMMIT everything.
5. Read-after-write inside the same transaction returns the new row.
```

---

### Business Rule: BR-H05 - Actor attribution from the authenticated principal

**Overview**:
`changedById` is always the authenticated user id from the verified JWT, never a value supplied in the request body.

**Location**: `src/modules/orders/order.controller.ts:30-31,40-41`; `src/middlewares/auth.middleware.ts:41-42`; `prisma/schema.prisma:122,126`
**Confidence**: High

**Detailed description**:
`authenticate` verifies the bearer token with `jwt.verify` and sets `req.user = { id: payload.sub, email: payload.email, role: payload.role }` (`auth.middleware.ts:41-42`). `OrderController.create` and `OrderController.changeStatus` reject a missing user and pass `req.user.id` as the `userId` argument (`order.controller.ts:30-31,40-41`). The service writes that value into `Order.createdById`, the creation history row, and every transition history row (`order.service.ts:104,110,164`). No Zod schema for order creation or status update contains a user, actor, or author field (`order.schemas.ts:11-21`), so a caller cannot impersonate another user through the body.

The database enforces that the id exists: `order_status_history.changedById` is `CHAR(36) NOT NULL` with a foreign key to `users.id` and `ON DELETE RESTRICT` (`migration.sql:93,124-125`). A history row cannot reference a non-existent user at insert time. Attribution survives user record changes because the row stores the immutable user id, not the user name or email.

Attribution is not accompanied by an authorization snapshot. The history row does not record the actor's role at the time, whether the actor owned the order, whether an override was used, or the request id. The service also does not verify that the user is active or that the role permits the operation; any authenticated `ADMIN` or `OPERATOR` can change any order. The audit trail therefore answers "which user id performed the write" but not "was that user authorized to perform it".

**Rule workflow**:
```
1. Bearer token is verified by authenticate.
2. req.user.id is populated from the JWT sub claim.
3. Controller passes req.user.id to the service.
4. Service writes it as changedById on every history insert.
5. FK to users.id rejects an unknown user.
6. ON DELETE RESTRICT prevents removing a user referenced by history.
```

---

### Business Rule: BR-H06 - Nullable fromStatus represents order creation

**Overview**:
`fromStatus` may be `NULL` only to express the absence of a previous state at order creation; the model has no separate creation event type.

**Location**: `prisma/schema.prisma:119`; `src/modules/orders/order.service.ts:108`
**Confidence**: High

**Detailed description**:
The schema declares `fromStatus OrderStatus?` (nullable) and `toStatus OrderStatus` (required) (`schema.prisma:119-120`). The creation path sets `fromStatus: null` and `toStatus: OrderStatus.PENDING` (`order.service.ts:108-109`). No other production path writes a null `fromStatus`; `changeStatus` always supplies `from = order.status`, which is a non-null enum because `Order.status` is required (`schema.prisma:78`). The nullable field is therefore a semantic marker for "the order had no status before this row".

There is no `CREATED` member in `OrderStatus` (`schema.prisma:16-23`). The creation event is reconstructed from the null `fromStatus` and the fixed reason `order created`, not from an event-type column. This keeps the enum limited to business states but couples the meaning of "creation" to a nullable field and a string literal.

The database does not restrict nulls to the first row. It permits a null `fromStatus` on any row inserted by any writer, including the seed script and direct Prisma calls. There is also no constraint that the first row for an order must have a null `fromStatus`, or that only one such row may exist. The semantic rule is enforced only by the two production write sites.

**Rule workflow**:
```
1. Creation nested write sets fromStatus = NULL, toStatus = PENDING.
2. All later transition writes supply a non-null fromStatus.
3. Read consumers interpret a NULL fromStatus as the creation event.
```

---

### Business Rule: BR-H07 - Optional free-text reason

**Overview**:
`reason` is optional, free text, limited to 500 characters at the HTTP boundary, stored as `NULL` when absent; creation uses a fixed literal.

**Location**: `src/modules/orders/order.schemas.ts:20`; `src/modules/orders/order.service.ts:111,165`; `prisma/schema.prisma:123`; `migration.sql:94`
**Confidence**: High

**Detailed description**:
`updateOrderStatusSchema` defines `reason: z.string().max(500).optional()` (`order.schemas.ts:20`). The service writes `input.reason ?? null`, so omission produces SQL `NULL`, not an empty string (`order.service.ts:165`). The database column is `VARCHAR(500) NULL` (`schema.prisma:123`; `migration.sql:94`), matching the validation ceiling. The creation path does not accept a reason from the client; it always writes the literal `order created` (`order.service.ts:111`).

The field has no enumeration, normalization, trimming, minimum length, or format rule. It can contain any Unicode text, including sensitive operational or personal data. There is no required reason for cancellation, even though cancellation is the transition most likely to need a rationale. There is no structured reason code, so aggregation by reason requires free-text parsing or exact matching.

Because the 500-character limit exists only in the HTTP schema, other writers bypass it. The seed script writes generated reasons such as `seed transition PENDING->PAID` (`prisma/seed.ts:316`). A direct Prisma call can supply a longer value; MySQL in strict mode would reject it rather than truncate, producing a database error that the error middleware would render as a generic 500 because only `P2002` and `P2025` are mapped (`error.middleware.ts:37-54`).

**Rule workflow**:
```
1. PATCH body passes updateOrderStatusSchema.
2. reason is either a string of at most 500 characters or absent.
3. Service writes input.reason ?? null.
4. Database stores VARCHAR(500) or NULL.
5. Read path returns the raw string with no transformation.
```

---

### Business Rule: BR-H08 - Database-assigned changedAt

**Overview**:
`changedAt` is assigned by the database default, not by the client, and is the only temporal field on the row.

**Location**: `prisma/schema.prisma:121`; `prisma/migrations/20260519182739_init/migration.sql:92`
**Confidence**: High

**Detailed description**:
The schema declares `changedAt DateTime @default(now())` (`schema.prisma:121`). The migration materializes it as `DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)` (`migration.sql:92`). Neither `changeStatus` nor the creation nested write supplies a value (`order.service.ts:106-113,159-167`), so MySQL assigns the timestamp at statement execution with millisecond precision. The HTTP request cannot set or override it because neither request schema contains a timestamp field.

The model has no `updatedAt`, no `deletedAt`, and no separate event-time versus record-time column. `changedAt` therefore serves simultaneously as the business event time and the persistence time. MySQL `DATETIME(3)` stores a wall-clock value without timezone; the database session timezone determines the stored value, and no timezone offset is stored with the row.

The seed script also relies on the default because it does not set `changedAt` (`prisma/seed.ts:310-318`). A direct Prisma write or raw SQL statement can supply an arbitrary timestamp, including a backdated or future value, because the column accepts explicit values despite the default. There is no check constraint limiting the range.

**Rule workflow**:
```
1. Service inserts a row without changedAt.
2. MySQL evaluates CURRENT_TIMESTAMP(3) at statement execution.
3. The value is stored as DATETIME(3) with millisecond precision.
4. Read path orders by this value ascending.
```

---

### Business Rule: BR-H09 - Ascending chronological read order without a tie-breaker

**Overview**:
History is returned oldest-first by `changedAt`, with no secondary sort key for rows sharing the same timestamp.

**Location**: `src/modules/orders/order.service.ts:117,173`; `src/modules/orders/order.repository.ts:56`
**Confidence**: High

**Detailed description**:
All three read includes use `history: { orderBy: { changedAt: 'asc' } }`: the creation response (`order.service.ts:117`), the status-change response (`order.service.ts:173`), and `findByIdWithRelations` (`order.repository.ts:56`). No include adds `id` or any other field as a secondary sort. The repository list path does not load history at all (`order.repository.ts:33-45`).

`changedAt` has millisecond precision (`migration.sql:92`). Two rows written in the same millisecond, or rows whose timestamps are manually set to the same value, have no defined relative order. MySQL may return them in any order, and the order can change between executions. This matters most for the creation row and an immediate first transition, or for concurrent duplicate rows created under a race.

Given the state machine, the number of rows per order is naturally bounded: the longest forward path produces five rows including creation (`PENDING -> PAID -> PROCESSING -> SHIPPED -> DELIVERED`), and the longest cancellation path produces four (`PENDING -> PAID -> PROCESSING -> CANCELLED`). The missing tie-breaker is therefore usually invisible. It becomes observable if direct writes add rows, if timestamps are backdated, or if concurrent requests produce duplicate rows with the same millisecond.

**Rule workflow**:
```
1. Query history for an order.
2. ORDER BY changedAt ASC.
3. No ORDER BY id or sequence column.
4. Rows with distinct timestamps are oldest-first.
5. Rows with equal timestamps have no guaranteed order.
```

---

### Business Rule: BR-H10 - History is embedded in order responses, not exposed as a standalone resource

**Overview**:
The component has no endpoint of its own; history is a projection on three order operations and is excluded from the list operation.

**Location**: `src/modules/orders/order.routes.ts:16-24`; `src/modules/orders/order.repository.ts:33-60`
**Confidence**: High

**Detailed description**:
The order router exposes `GET /`, `GET /:id`, `POST /`, `PATCH /:id/status`, and `DELETE /:id` (`order.routes.ts:16-24`). There is no `/orders/:id/history` route and no history controller. `GET /orders/:id` returns the aggregate with history (`order.repository.ts:47-60`). `POST /orders` returns the created aggregate with its single initial row (`order.service.ts:115-119`). `PATCH /orders/:id/status` returns the refreshed aggregate with all rows (`order.service.ts:169-176`). `GET /orders` returns bare `Order` rows without relations (`order.repository.ts:33-45`).

The returned history includes all scalar fields: `id`, `orderId`, `fromStatus`, `toStatus`, `changedAt`, `changedById`, and `reason`. It does not include the `changedBy` relation, so the actor is an opaque UUID with no name or email. It does not include any derived fields such as duration between statuses or the current status. There is no field selection, filtering by status or actor, date range, pagination, or total count.

Because the history is loaded as part of the order aggregate, every detail read pays the cost of loading all rows and every create/status response pays the cost of loading history again after the write. For the bounded history size of the state machine this is small, but the read contract is coupled to the order aggregate and cannot be consumed independently.

**Rule workflow**:
```
GET /api/v1/orders/:id
  -> 200 order + items + history (asc) + customer summary
POST /api/v1/orders
  -> 201 order + items + history (one row) + customer summary
PATCH /api/v1/orders/:id/status
  -> 200 order + items + full history (asc) + customer summary
GET /api/v1/orders
  -> 200 bare order rows, no history
DELETE /api/v1/orders/:id
  -> 204, history cascaded away
```

---

### Business Rule: BR-H11 - Append-only by convention, not by enforcement

**Overview**:
Production code only inserts history rows, but neither the application nor the database prevents updates or deletes.

**Location**: `prisma/schema.prisma:116-131`; write sites `src/modules/orders/order.service.ts:106-113,159-167`
**Confidence**: High

**Detailed description**:
A repository-wide search for `orderStatusHistory` in `src/` returns one match: `tx.orderStatusHistory.create` in `changeStatus` (`order.service.ts:159`). The creation path uses a nested `history.create` (`order.service.ts:106`). There is no `orderStatusHistory.update`, `updateMany`, `delete`, or `deleteMany` anywhere in application source. There is no service method, repository method, or route that mutates an existing history row. From the application's perspective, the journal is append-only.

At the database level there is no enforcement. The table has no trigger, no check constraint, no rule, no row-level security, and no separate restricted database role. The Prisma client exposes `update`, `updateMany`, `upsert`, `delete`, and `deleteMany` for the model, and any future code with a `PrismaClient` can call them. The row has no version column, hash, previous-hash pointer, signature, or write-once marker, so even a silent edit would leave no cryptographic evidence.

The only structural properties that support immutability are negative: no `updatedAt` field, no business update path, and no API surface. Those are conventions. They do not survive a direct database connection, a migration script, the seed script, or the test harness, all of which have deleted history rows (`prisma/seed.ts:181`; `tests/setup.ts:9`).

**Rule workflow**:
```
Application path:
  INSERT only -> no UPDATE/DELETE.

Prisma client path:
  update / updateMany / delete / deleteMany are all available.
  No trigger, permission, or hash prevents or detects them.

Deletion path:
  DELETE order -> ON DELETE CASCADE removes history.
```

---

### Business Rule: BR-H12 - History retention is bound to order retention

**Overview**:
History cannot outlive its order; deleting the order physically deletes all of its audit rows.

**Location**: `prisma/schema.prisma:125`; `prisma/migrations/20260519182739_init/migration.sql:122`; `src/modules/orders/order.repository.ts:66-68`
**Confidence**: High

**Detailed description**:
The `order` relation on `OrderStatusHistory` declares `onDelete: Cascade` (`schema.prisma:125`). The migration creates `order_status_history_orderId_fkey` with `ON DELETE CASCADE ON UPDATE CASCADE` (`migration.sql:122`). `OrderRepository.deleteById` performs a hard `prisma.order.delete` (`order.repository.ts:66-68`), so MySQL removes the order's history rows as part of the same delete. No application code deletes history explicitly before the order delete.

The service restricts which orders can be deleted: only `PENDING` or `CANCELLED` (`order.service.ts:184-190`). For a `PENDING` order, deletion destroys the single creation row. For a `CANCELLED` order, deletion destroys between two and four rows depending on the path taken to cancellation. The status restriction protects `PAID`, `PROCESSING`, `SHIPPED`, and `DELIVERED` orders from this destruction through the API, but it does not protect `PENDING` or `CANCELLED` audit data.

There is no tombstone, archive table, soft-delete flag, or deletion audit row. Once the order is deleted, neither the order nor its status history is recoverable through the application, and no record identifies who deleted it or when. The database retains no trace beyond the absence of the rows. The seed script and test setup also delete history rows directly (`prisma/seed.ts:181`; `tests/setup.ts:9`).

**Rule workflow**:
```
DELETE /orders/:id
  -> service permits only PENDING or CANCELLED
  -> prisma.order.delete
  -> InnoDB cascade deletes order_status_history rows
  -> 204 No Content
  -> no audit record of the deletion
```

---

### Business Rule: BR-H13 - Referential integrity with the acting user

**Overview**:
A user referenced by history cannot be deleted while the history row exists; the foreign key is `ON DELETE RESTRICT`.

**Location**: `prisma/schema.prisma:126`; `prisma/migrations/20260519182739_init/migration.sql:125`
**Confidence**: High

**Detailed description**:
`changedById` is required and references `User` through the named relation `StatusChangedBy` (`schema.prisma:122,126`). The migration creates `order_status_history_changedById_fkey` with `ON DELETE RESTRICT ON UPDATE CASCADE` (`migration.sql:125`). Any attempt to delete a user who appears as an actor in at least one history row is rejected by MySQL with a foreign-key error.

This protects attribution integrity: a history row cannot point to a missing user. It also creates a retention coupling in the opposite direction from BR-H12. An order delete can remove history and thereby make the user deletable; a user delete cannot proceed while history remains. Because the Users module exposes no delete operation, the restriction is currently latent, but it is a real schema-level constraint for any future administrative deletion or privacy-erasure operation.

The `ON UPDATE CASCADE` action is effectively inert because `users.id` is a UUID primary key that the application never changes. The row stores only the id, not a snapshot of the user's name, email, or role. If the user's name changes, the historical attribution still resolves to the same user record and displays the current name if a future read joins the relation. There is no record of the actor's attributes at the time of the event.

**Rule workflow**:
```
1. History insert supplies changedById.
2. FK requires a matching users.id.
3. DELETE user with history -> RESTRICT, operation fails.
4. Delete all orders referencing the user -> cascade removes history.
5. User deletion may then succeed if no other references remain.
```

---

### Business Rule: BR-H14 - Audit scope is limited to status transitions and creation

**Overview**:
The journal records order creation and successful status transitions only; it does not record other mutations, failures, deletions, stock movements, or reads.

**Location**: `prisma/schema.prisma:116-131`; `src/modules/orders/order.service.ts`
**Confidence**: High

**Detailed description**:
The model has fields only for order id, status pair, timestamp, actor, and reason. It has no event type, entity type, field name, old value, new value, request id, source, IP address, user agent, or payload. Consequently it cannot represent anything other than a status transition. The two write sites confirm that only creation and status change produce rows. There is no order update endpoint for totals, discount, notes, or customer, so no field-change audit is expected from the current API, but the model could not represent one if such an endpoint were added.

Several operation classes are outside the audit scope. Failed transitions are not recorded (BR-H03). Order deletion is not recorded (BR-H12). Stock decrement and replenishment are not recorded as separate events, even though they are business-relevant side effects of specific transitions; the transition row is the only evidence that stock moved. Read access is not recorded. Authentication and authorization events are outside this component entirely.

The materialized `Order.status` column and the journal can diverge if a writer updates the order without inserting history. The service does not do this, but the database has no trigger linking the two. The journal is therefore a partial history of status changes made through the service, not a database-level change-data-capture stream.

**Rule workflow**:
```
Recorded:
  order created (NULL -> PENDING)
  successful status transition (from -> to)

Not recorded:
  rejected same-status transition
  rejected illegal transition
  insufficient-stock failure
  order deletion
  stock movement as a standalone event
  field changes other than status
  read access
  actor role or authorization decision
```

---

### Business Rule: BR-H15 - No uniqueness or idempotency constraint

**Overview**:
The schema has no unique constraint on history rows, so duplicate entries for the same transition are structurally permitted.

**Location**: `prisma/schema.prisma:128-130`; `prisma/migrations/20260519182739_init/migration.sql:96-97`
**Confidence**: High

**Detailed description**:
The only key is the primary key on `id` (`schema.prisma:117`). The two declared indexes are non-unique single-column indexes on `orderId` and `changedAt` (`schema.prisma:128-129`; `migration.sql:96-97`). There is no unique constraint on `(orderId, changedAt)`, `(orderId, toStatus)`, `(orderId, fromStatus, toStatus)`, or any sequence number. There is no idempotency key on the status-change request or on the history row.

Under normal sequential use, the service produces one row per successful transition because it appends once per call. Under concurrency, two transactions can both read the same prior status and both append a row for the same transition. The transaction does not lock the order row (`order.service.ts:132-135`), and there is no optimistic version check or conditional update. Both transactions can pass `canTransition`, both can update the order, and both can insert history; the last update wins on `orders.status`, and the journal retains two rows that claim the same `fromStatus` and `toStatus`.

The absence of a uniqueness rule also means a retried request after a timeout can append a duplicate if the first transaction actually committed but the client did not receive the response. The same-status guard returns 409 on a later retry, but it cannot prevent a duplicate when the retry overlaps the original before either commits. This is a structural property of the schema, not only of the service.

**Rule workflow**:
```
Sequential:
  PENDING -> PAID (row 1), PAID -> PROCESSING (row 2): distinct.

Concurrent:
  Tx A reads PENDING.
  Tx B reads PENDING.
  Tx A updates to PAID and inserts PENDING->PAID.
  Tx B updates to PAID and inserts PENDING->PAID.
  Result: two identical transition rows.
```

---

### Business Rule: BR-H16 - Enum domain versus application-enforced transition legality

**Overview**:
The database restricts status values to the six enum members, but it does not restrict which `fromStatus`/`toStatus` pairs are legal.

**Location**: `prisma/schema.prisma:16-23,119-120`; `src/modules/orders/order.status.ts:3-14`
**Confidence**: High

**Detailed description**:
`fromStatus` and `toStatus` use the `OrderStatus` enum (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) and are materialized as MySQL `ENUM` columns (`schema.prisma:16-23`; `migration.sql:90-91`). The database rejects any value outside that set, so the journal cannot contain an unknown status. The database does not reject `fromStatus = toStatus`, backward movement, or a pair not present in the state machine.

Transition legality is enforced in application code by `canTransition` against the read-only transition table (`order.status.ts:3-14`), called from `changeStatus` (`order.service.ts:147-149`). The table permits `PENDING -> [PAID, CANCELLED]`, `PAID -> [PROCESSING, CANCELLED]`, `PROCESSING -> [SHIPPED, CANCELLED]`, `SHIPPED -> [DELIVERED]`, and no successors for `DELIVERED` or `CANCELLED`. The same-status case is rejected separately before `canTransition` (`order.service.ts:140-146`).

Because the transition table is code and is not persisted or mirrored in a database constraint, any writer other than `OrderService.changeStatus` can insert an illegal pair. This includes the seed script, which constructs its own paths, and any future direct Prisma or SQL write. The journal therefore records the pair it was given; it does not prove that the pair was legal at the time.

**Rule workflow**:
```
API path:
  toStatus must be an OrderStatus enum value (Zod).
  from === to -> 409.
  canTransition(from, to) false -> 409.
  Only legal pairs are inserted.

Direct path:
  Any enum pair is accepted by the database.
```

---

### Business Rule: BR-H17 - Cross-row chain consistency is not enforced or validated

**Overview**:
Each row stores its own `fromStatus` and `toStatus`, but nothing guarantees that consecutive rows form a continuous chain.

**Location**: `prisma/schema.prisma:116-131`; `src/modules/orders/order.service.ts:159-167`
**Confidence**: High (absence of enforcement); Medium (interpretation of read consumers)

**Detailed description**:
The journal stores a `fromStatus` and `toStatus` on every row. A consistent history has the property that the `toStatus` of one row equals the `fromStatus` of the next row, and that the first row has a null `fromStatus`. The schema has no sequence number, no previous-row id, no unique ordering key, and no check constraint linking rows. The service produces consistent pairs sequentially, but the database does not enforce the chain.

Read consumers receive the rows in `changedAt` order and must infer the chain themselves. There is no application code that verifies the chain, detects gaps, or reconciles the latest history row with `Order.status`. The detail response returns both `Order.status` and the history array, and no code compares them. If a writer updates the order status without history, or inserts a row out of order, the response can contain a materialized status that does not match the last row.

The missing chain constraint is a structural limitation rather than an active bug in the service. It matters for tamper detection, data repair, and any future consumer that treats the journal as the source of truth for current state.

**Rule workflow**:
```
Expected invariant:
  row[0].fromStatus = NULL
  row[0].toStatus = row[1].fromStatus
  row[1].toStatus = row[2].fromStatus
  ...
  row[n].toStatus = orders.status

Actual enforcement:
  none at the database or service read layer.
```

---

## 4. Component Structure

### 4.1 File and artifact structure

```
prisma/
├── schema.prisma                              # OrderStatusHistory model (lines 116-131)
├── migrations/
│   └── 20260519182739_init/
│       └── migration.sql                      # table (87-99), indexes (96-97), FKs (122,125)
└── seed.ts                                    # deleteMany (181), create loop (307-320)

src/modules/orders/
├── order.service.ts                           # nested create (106-113), transition create (159-167)
│                                              # history includes (117,173)
├── order.repository.ts                        # OrderWithRelations type (12-16)
│                                              # history include (56)
├── order.controller.ts                        # userId propagation (30-31,40-41)
├── order.routes.ts                            # order endpoints (16-24)
├── order.schemas.ts                           # reason max 500 (20)
└── order.status.ts                            # legal transition table (3-14)

tests/
├── setup.ts                                   # history deleteMany (9)
├── orders.test.ts                             # history assertions (37-38,80-83)
└── helpers/factories.ts                       # no history fixture
```

There is no `order-history.repository.ts`, `order-history.service.ts`, `order-history.controller.ts`, `order-history.routes.ts`, or domain entity class. The component exists only as a Prisma model plus the call sites that use it.

### 4.2 Schema shape

| Column | Prisma type | SQL type | Null | Default | Constraint / note |
|---|---|---|---|---|---|
| `id` | `String` | `CHAR(36)` | No | None at SQL level; Prisma `uuid()` generates the value client-side | Primary key (`schema.prisma:117`; `migration.sql:88,98`) |
| `orderId` | `String` | `CHAR(36)` | No | None | FK to `orders.id`; `ON DELETE CASCADE ON UPDATE CASCADE` (`schema.prisma:125`; `migration.sql:122`) |
| `fromStatus` | `OrderStatus?` | `ENUM(...)` | Yes | None | Nullable only by convention for creation (`schema.prisma:119`; `migration.sql:90`) |
| `toStatus` | `OrderStatus` | `ENUM(...)` | No | None | Required (`schema.prisma:120`; `migration.sql:91`) |
| `changedAt` | `DateTime` | `DATETIME(3)` | No | `CURRENT_TIMESTAMP(3)` | Database-assigned (`schema.prisma:121`; `migration.sql:92`) |
| `changedById` | `String` | `CHAR(36)` | No | None | FK to `users.id`; `ON DELETE RESTRICT ON UPDATE CASCADE` (`schema.prisma:126`; `migration.sql:125`) |
| `reason` | `String?` | `VARCHAR(500)` | Yes | None | Free text; no enum/normalization (`schema.prisma:123`; `migration.sql:94`) |

### 4.3 Index inventory

| Index | Columns | Type | Declared in | Supports |
|---|---|---|---|---|
| `PRIMARY` | `id` | Unique B-tree | `migration.sql:98` | Primary-key lookup; no application code looks up a history row by id |
| `order_status_history_orderId_idx` | `orderId` | Non-unique B-tree | `schema.prisma:128`; `migration.sql:96` | Filtering history by parent order; the FK requires an index |
| `order_status_history_changedAt_idx` | `changedAt` | Non-unique B-tree | `schema.prisma:129`; `migration.sql:97` | Time-range scans or global chronological reads; no current application query uses it |
| Implicit FK index on `changedById` | `changedById` | Non-unique B-tree | Not declared in Prisma; created by InnoDB to satisfy the FK | Actor-based lookups; no current application query uses it |

The actual read pattern is `WHERE orderId = ? ORDER BY changedAt ASC` on a small per-order set. The `orderId` index can locate the rows; MySQL then sorts them by `changedAt`. There is no composite `(orderId, changedAt)` index that would satisfy both the predicate and the ordering from one index. Because the state machine bounds the row count to at most five through the API, the sort is negligible. The `changedAt` index is unused by the current code and becomes relevant only for a future global history query or retention job.

### 4.4 Relation semantics

```
Order 1 ----- N OrderStatusHistory
  Order.history (schema.prisma:90)
  OrderStatusHistory.order (schema.prisma:125)
  onDelete: Cascade

User 1 ----- N OrderStatusHistory
  User.statusHistoryChanges @relation("StatusChangedBy") (schema.prisma:35)
  OrderStatusHistory.changedBy @relation("StatusChangedBy") (schema.prisma:126)
  onDelete: Restrict (Prisma default for a required relation, confirmed in migration line 125)
```

- A history row belongs to exactly one order because `orderId` is required and is a foreign key. An order can have zero rows only if it was inserted outside the service; the service always writes one at creation.
- A history row belongs to exactly one user because `changedById` is required and is a foreign key.
- The relation to `Order` is the retention boundary. The relation to `User` is an attribution boundary.
- The model has no relation to `Product`, `OrderItem`, or any stock-movement record. Stock effects are not represented in the audit trail.

---

## 5. Dependency Analysis

### 5.1 Internal dependencies

```
OrderService.create
  -> OrderStatusHistory (nested history.create)
  -> OrderStatus enum (PENDING)
  -> Order (nested write parent)
  -> User (changedById value from JWT)

OrderService.changeStatus
  -> OrderStatusHistory (tx.orderStatusHistory.create)
  -> OrderStatus enum (fromStatus/toStatus)
  -> Order (parent status read/update)
  -> User (changedById value from JWT)

OrderRepository.findByIdWithRelations
  -> OrderStatusHistory (include.history)
  -> OrderStatusHistory (OrderWithRelations type)

OrderStatusHistory
  -> Order (required relation, cascade)
  -> User (required relation, restrict)
  -> OrderStatus (fromStatus/toStatus enum)

seed.ts
  -> OrderStatusHistory (deleteMany, create)

tests/setup.ts
  -> OrderStatusHistory (deleteMany)

tests/orders.test.ts
  -> OrderStatusHistory (read indirectly through HTTP JSON)
```

The write path does not pass through `OrderRepository`. `OrderService` holds a `PrismaClient` directly for both transactions (`order.service.ts:27-30`), so history writes are coupled to the ORM and to the transaction client rather than to a repository contract. The read path does use `OrderRepository`, which returns the history as part of `OrderWithRelations`.

The `history` include literal is duplicated in three places: twice in `OrderService` (`order.service.ts:117,173`) and once in `OrderRepository` (`order.repository.ts:56`). All three must stay aligned on ordering; there is no shared projection constant.

### 5.2 External dependencies

| Dependency | Version | Role for this component | Coupling |
|---|---|---|---|
| `@prisma/client` | 5.22.0 (`package.json:26`) | Generates `OrderStatusHistory` type, `tx.orderStatusHistory.create`, `include.history`, query engine | Hard, compile-time and runtime |
| Prisma CLI / migration engine | 5.22.0 (`package.json:48`) | Creates `order_status_history`, indexes, and foreign keys | Build-time / deployment-time |
| MySQL 8.0 | Docker Compose service (`docker-compose.yml`) | Stores rows, applies enum, cascade, restrict, default timestamp | Hard, runtime |
| `zod` | 3.23.8 (`package.json:33`) | Enforces `reason` max 500 at the HTTP boundary; enforces enum for `toStatus` | Indirect, boundary only |
| `express` | 4.21.1 (`package.json:28`) | Serializes history in HTTP responses | Indirect |
| `vitest` and `supertest` | 2.1.4 / 7.0.0 (`package.json:52,49`) | Exercise history through HTTP | Test-time only |

There is no message broker, event bus, cache, search index, or external audit service. The only stateful dependency is MySQL. There is no outbound integration: the component does not emit webhooks, events, or logs of its own.

### 5.3 Runtime dependency chain

```
HTTP request
  -> Express router (authenticate, validate)
  -> OrderController
  -> OrderService (PrismaClient)
  -> Prisma query engine
  -> MySQL InnoDB
  -> order_status_history table
```

The read chain is the same until `OrderRepository`, which uses the same `PrismaClient` and query engine.

---

## 6. Afferent and Efferent Coupling

Coupling is expressed at the artifact level appropriate to a Prisma data component: the model/table, the write call sites, the read projection, and the auxiliary writers.

| Component / artifact | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Criticality | Notes |
|---|---:|---:|---|---|
| `OrderStatusHistory` Prisma model | 5 | 3 | High | Ca: two writes in `OrderService`, one read include in `OrderRepository`, `OrderWithRelations` type, seed/delete paths. Ce: `Order`, `User`, `OrderStatus`. |
| `order_status_history` table | 5 | 3 | High | Ca: ORM writes/reads/deletes. Ce: `orders` FK, `users` FK, MySQL enum type. |
| `history` projection literal | 3 | 2 | Medium | Duplicated in `order.service.ts:117,173` and `order.repository.ts:56`; Ce: `changedAt` ordering. |
| `OrderWithRelations.history` type | 2 | 1 | Medium | Ca: `OrderService.getById`, controller return type. Ce: `OrderStatusHistory` type. |
| `OrderStatusHistory` TS type import | 2 | 1 | Low | `order.repository.ts:1,14`; generated by Prisma. |
| Migration definition | 0 | 3 | High | Ce: `orders`, `users`, enum type; no inbound code references. |
| Seed history writer | 0 | 2 | Low | Ce: `order_status_history`, `OrderStatus`. |
| Test history assertions | 0 | 1 | Low | Ce: HTTP response shape; no direct model coupling. |

Observations:

- The component has no internal outgoing calls. It is a passive data store; all behavior lives in its callers.
- Its afferent coupling is entirely through Prisma, not through a TypeScript interface. There is no abstraction boundary to substitute.
- The model is depended on by `Order` and `User` relation back-references, which are schema-level couplings that static import analysis does not show.
- The duplicated read projection creates a small internal consistency risk: a change to one `orderBy` or include shape can diverge from the others.
- The component's data-level coupling is high because `changedById` and `orderId` are hard foreign keys, while its code-level coupling is low because it has no module of its own.

---

## 7. Endpoints

The component does not own or expose any endpoint. There is no `OrderStatusHistoryController` and no history route. The table below lists the order endpoints that read, write, or delete history, because those are the only HTTP surfaces through which the component is observable.

| Endpoint | Method | Description | History effect | Auth |
|---|---|---|---|---|
| `/api/v1/orders` | POST | Create an order | Inserts one initial row (`NULL -> PENDING`, reason `order created`) and returns it | Bearer JWT (`authenticate`) |
| `/api/v1/orders/{id}` | GET | Get an order by id | Returns full history ordered by `changedAt` ASC | Bearer JWT (`authenticate`) |
| `/api/v1/orders/{id}/status` | PATCH | Change order status | Appends one row for a successful transition and returns full history | Bearer JWT (`authenticate`) |
| `/api/v1/orders/{id}` | DELETE | Delete an order | Cascades deletion of all history rows; no audit record remains | Bearer JWT (`authenticate`) |
| `/api/v1/orders` | GET | List orders | No history included | Bearer JWT (`authenticate`) |

Route definitions: `src/modules/orders/order.routes.ts:16-24`. The router applies only `authenticate` (`order.routes.ts:14`); no route applies `requireRole`, so both `ADMIN` and `OPERATOR` tokens can access every operation.

There is no GraphQL, gRPC, or other protocol surface for this component.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|---|---|---|---|---|---|
| MySQL 8.0 via Prisma | External stateful service | Persist and query `order_status_history` rows | MySQL wire protocol over TCP | Relational rows mapped to `OrderStatusHistory` | Lazy connection; Prisma errors surface at query time. Only `P2002` and `P2025` are mapped (`error.middleware.ts:37-54`); other Prisma errors become 500. No retry, circuit breaker, or deadlock handling. |
| `orders` table | Internal database integration | Parent entity; `orderId` FK with `ON DELETE CASCADE` | SQL foreign key | `CHAR(36)` | Cascade delete removes history; FK violation on insert (missing order) becomes 500 because `P2003` is unmapped. |
| `users` table | Internal database integration | Actor identity for `changedById`; `ON DELETE RESTRICT` | SQL foreign key | `CHAR(36)` | Insert with unknown user becomes 500 (`P2003`); deleting a referenced user is blocked by MySQL. |
| Express HTTP layer | Internal application integration | Serialize history in order responses | HTTP/JSON | JSON array of history objects | Handled errors use the shared error envelope; unexpected serialization errors fall through to the central middleware. |
| Prisma transaction manager | Internal runtime integration | Atomic order + history writes | In-process Prisma API | Prisma query objects | Interactive transaction timeout or write conflict surfaces as an unmapped Prisma error and becomes 500. |
| Seed script | Development tooling | Delete and recreate demo history | Direct Prisma calls | Prisma create payloads | No transaction around the path loop; a failure can leave partial history. |
| Test harness | Test tooling | Truncate history before each test | Direct Prisma calls | `deleteMany` | No environment guard; the configured `DATABASE_URL` is used directly (`tests/setup.ts:8-16`). |
| Outbox / webhook pipeline | Planned, not implemented | Would consume status changes for external notification | None in code | None | No outbox table, event emission, worker, or retry exists in the repository. The component currently produces no event. |

No external HTTP service, message broker, cache, or file system is involved in this component.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---|---|---|---|
| Audit log / append-only journal | `OrderStatusHistory` rows inserted on creation and each successful status change | `order.service.ts:106-113,159-167` | Reconstruct order lifecycle changes |
| State-transition record (delta pair) | `fromStatus` + `toStatus` per row | `schema.prisma:119-120` | Record both source and destination, not only the new state |
| Unit of Work / interactive transaction | Prisma `$transaction` wrapping order and history writes | `order.service.ts:58-123,131-178` | Atomicity between state and audit |
| Nested write | `history.create` inside `tx.order.create` | `order.service.ts:106-113` | Guarantee an order cannot exist without its initial row |
| Eager-loaded aggregate projection | `include.history` with explicit `orderBy` | `order.service.ts:117,173`; `order.repository.ts:56` | Return history with the order without a second request |
| Referential-action cascade | `onDelete: Cascade` on `OrderStatusHistory.order` | `schema.prisma:125`; `migration.sql:122` | Automatic cleanup when the parent order is deleted |
| Referential-action restrict | `onDelete: Restrict` on `OrderStatusHistory.changedBy` | `schema.prisma:126`; `migration.sql:125` | Preserve actor attribution |
| Actor attribution via foreign key | `changedById` from JWT `sub` | `order.controller.ts:30-31,40-41` | Attribute every row to an authenticated user |
| Temporal ordering by timestamp | `orderBy: { changedAt: 'asc' }` | `order.service.ts:117,173`; `order.repository.ts:56` | Present lifecycle in chronological order |
| Anemic data model / no domain repository | No history service, repository, or entity; Prisma model only | `prisma/schema.prisma:116-131` | Minimal implementation; logic lives in `OrderService` |
| Duplicated projection literal | Same `history` include written three times | `order.service.ts:117,173`; `order.repository.ts:56` | No shared constant; small drift risk |
| Optimistic concurrency absent | No version column, conditional update, or row lock | `order.service.ts:131-159`; `schema.prisma:116-131` | None; this is an architectural omission rather than a pattern |

### 9.1 Architectural role

The component is a journal subordinate to the `Order` aggregate. It is not a first-class aggregate of its own: it has no independent lifecycle, no independent transaction boundary, and no independent API. Its identity is derived from the parent order, and its retention is controlled by the parent. This is a deliberate "history as part of the order aggregate" design, consistent with the aggregate being loaded with `history` in `OrderWithRelations`.

The journal uses a delta representation (previous and next state) rather than a full snapshot. It stores no version of the order's totals, items, or customer. This keeps rows small and makes the status narrative explicit, but it cannot reconstruct the full order state at a past point in time; only the status at each transition is known.

### 9.2 Quality attribute assessment

| Attribute | Assessment | Evidence |
|---|---|---|
| Auditability | Strong for successful status transitions; weak for attempts, deletions, and other mutations | Only two write sites; no failure/deletion audit |
| Integrity | Strong within a single transaction; weak under concurrency | `$transaction`; no lock/version |
| Immutability | Not enforced | No trigger, permission, hash, or unique write-once rule |
| Attribution | Strong identity source and FK integrity; shallow context | JWT `sub`; no actor snapshot or authorization record |
| Performance | Low risk at current scale; one index per access path, no composite index | Bounded rows per order; small sort |
| Scalability | Unbounded table growth with no archive; global `changedAt` index unused | No retention policy |
| Security | Broad read access; free-text reason may contain sensitive data; no field-level protection | `authenticate` only |
| Maintainability | Small schema surface; duplicated projection literals; no dedicated abstraction | Three identical include shapes |
| Testability | Integration-testable through HTTP; no unit-level seam | No history repository/service to unit test |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|---|---|---|---|
| Critical | `changeStatus` write path | No row lock, optimistic version, or conditional status update; two concurrent requests can both pass validation and append duplicate history rows | Journal can contain multiple rows for one transition; stock may be debited twice; latest-row reconstruction is ambiguous (`order.service.ts:131-159`) |
| High | Immutability | No database-level protection against UPDATE/DELETE; Prisma exposes full CRUD | Audit rows can be altered or removed without detection; no tamper evidence (`schema.prisma:116-131`) |
| High | Retention | `ON DELETE CASCADE` physically removes history when an order is deleted; deletion is not audited | Audit trail for deleted `PENDING`/`CANCELLED` orders is unrecoverable; no record of who deleted or when (`schema.prisma:125`; `order.service.ts:181-192`) |
| High | Audit completeness | Failed/rejected transitions, stock failures, deletions, field changes, and reads are not recorded | Cannot investigate attempted abuse, failed operations, or data changes; journal is not a complete activity record |
| High | Authorization | All order routes require only `authenticate`; no `requireRole` or ownership scoping | Any authenticated user can read any order's audit trail and change any order; audit actor is reliable but authorization is not (`order.routes.ts:14`) |
| Medium | Ordering | `ORDER BY changedAt` with no secondary key | Rows with equal millisecond timestamps can appear in nondeterministic order; chain reconstruction can vary between reads (`order.service.ts:117,173`; `order.repository.ts:56`) |
| Medium | Uniqueness / idempotency | No unique constraint on transition; no idempotency key on PATCH | Duplicate rows from concurrency or retries; no way to deduplicate reliably (`schema.prisma:128-130`) |
| Medium | Chain integrity | No constraint or read validation that `toStatus` of row N equals `fromStatus` of row N+1; no reconciliation with `Order.status` | Journal can be internally inconsistent or diverge from the materialized order state; no detection (`schema.prisma:116-131`) |
| Medium | Indexing | No composite `(orderId, changedAt)` index; `changedAt` index unused by current queries | Current per-order volume makes this negligible; large histories or global queries would sort or scan more than necessary (`migration.sql:96-97`) |
| Medium | Retention lifecycle | No archive, partition, TTL, or purge strategy; table grows with every transition forever | Unbounded storage and slower retention/cleanup operations; no separation of active and historical data |
| Medium | Actor context | Row stores only `changedById`; no name snapshot, role, IP, request id, source, or correlation id | Cannot reconstruct actor attributes at event time; cannot correlate a history row with a specific HTTP request or log entry |
| Medium | Reason quality | Free text, optional even for cancellation, no taxonomy or normalization; max length enforced only at HTTP boundary | Inconsistent rationale data; sensitive data may be stored unredacted; direct writers can bypass the 500 limit (`order.schemas.ts:20`; `schema.prisma:123`) |
| Medium | Error handling | Only Prisma `P2002` and `P2025` are mapped; FK violations, deadlocks, and transaction conflicts become generic 500 | Audit insert failures are indistinguishable from other server faults; no retry or conflict resolution (`error.middleware.ts:37-64`) |
| Medium | Test coverage | No tests for immutability, cascade deletion, actor attribution, reason persistence, no-write-on-failure, ordering ties, or concurrency | Regressions in audit guarantees would not be detected by the current suite (`tests/orders.test.ts`) |
| Low | Projection duplication | Same history include shape is written in three places | A change to one path can diverge from the others; no shared constant (`order.service.ts:117,173`; `order.repository.ts:56`) |
| Low | Storage type | UUID stored as `CHAR(36)` for every row and foreign key | Larger index and row footprint than binary UUID; not a correctness issue |
| Low | Timestamp semantics | `DATETIME(3)` without timezone; DB session timezone determines stored value | Ambiguity across deployments with different timezones; no event-time versus record-time distinction |
| Low | Creation marker | `fromStatus = NULL` plus fixed string `order created` is the only creation marker | Consumers must know the convention; no event-type column (`order.service.ts:108-111`) |
| Low | Seed consistency | Seed writes history paths without a transaction and can use arbitrary reasons | Development data may not match production invariants (`prisma/seed.ts:307-320`) |
| Low | Test harness safety | `beforeEach` deletes all history using the configured `DATABASE_URL` with no environment guard | Running tests against a non-test database destroys audit data (`tests/setup.ts:8-16`) |

### 10.1 Error handling and resilience details

The audit insert has no dedicated error handling. In `create`, a history insert failure is a Prisma error inside the transaction; the transaction rolls back and the error propagates to the central middleware. In `changeStatus`, the same applies. The middleware maps `P2002` to 409 and `P2025` to 404, logs everything else, and returns a generic 500 (`error.middleware.ts:37-64`). A foreign-key violation (`P2003`) from `changedById` or `orderId`, a transaction conflict, or a deadlock is therefore reported as a server error with no audit-specific code.

There is no retry for transient deadlocks or write conflicts. There is no idempotency key to make a retried status change safe. There is no circuit breaker because the only dependency is the database. There is no fallback persistence path; if the database is unavailable, the status change fails and no audit row is written, which keeps state and audit consistent but reduces availability.

The JWT authentication path does not query the database (`auth.middleware.ts:40-46`). A token for a user row that has been removed outside the application remains valid until expiry. If such a token is used for a status change, the history insert fails on the `changedById` foreign key and becomes a 500. This is latent because the Users module has no delete endpoint, but it is reachable through seed or direct database cleanup.

---

## 11. Test Coverage Analysis

### 11.1 Test inventory

| Test file | Type | History relevance |
|---|---|---|
| `tests/orders.test.ts` | Integration (HTTP + real Prisma/MySQL) | Asserts creation history length and `toStatus`; asserts transition sequence |
| `tests/setup.ts` | Test harness | Deletes all history before each test; connects/disconnects Prisma |
| `tests/helpers/factories.ts` | Test fixtures | Creates users, customers, products; no history fixture |
| `tests/auth.test.ts` | Integration | Does not reference history; shares the harness that truncates it |
| `vitest.config.ts` | Configuration | No coverage provider or coverage thresholds configured |
| `package.json` | Configuration | `test` script is `vitest run`; no coverage script (`package.json:17`) |

No unit test file exists for the schema, the write paths, or the read projection in isolation.

### 11.2 Behavior coverage

| Behavior / rule | Test location | Assertions | Coverage assessment |
|---|---|---|---|
| BR-H01 initial row created | `tests/orders.test.ts:37-38` | `history` has length 1; `history[0].toStatus === 'PENDING'` | Partial: does not assert `fromStatus === null`, `reason`, `changedById`, `changedAt`, or `orderId` |
| BR-H02 transition row appended | `tests/orders.test.ts:80-83` | `history.map(h => h.toStatus)` equals `['PENDING', 'PAID']` | Partial: does not assert `fromStatus`, `changedById`, `reason`, or row count after a longer path |
| BR-H03 no row on invalid transition | `tests/orders.test.ts:89-107` | Asserts 409 and error code only | Not covered: no history count assertion after the failed request |
| BR-H03 no row on stock failure | `tests/orders.test.ts:109-132` | Asserts 422, error details, and unchanged stock | Not covered: no history count assertion |
| BR-H04 atomicity | None | None | Not covered: no rollback/failure-injection test |
| BR-H05 actor attribution | None | None | Not covered: no assertion on `changedById` vs. token subject |
| BR-H06 null `fromStatus` | None | None | Not covered: creation test checks `toStatus` only |
| BR-H07 reason persistence | `tests/orders.test.ts:76` sends `reason: 'payment confirmed'` | No assertion on the stored/returned reason | Not covered: reason is sent but never verified |
| BR-H08 timestamp | None | None | Not covered: no assertion that `changedAt` exists, is server-assigned, or is ordered |
| BR-H09 ascending order | `tests/orders.test.ts:80-83` | Sequence `PENDING`, `PAID` implies ascending for two rows | Partial: no equal-timestamp or multi-row ordering test |
| BR-H10 embedded read surface | Indirectly covered by the assertions above | Detail/create/PATCH responses parsed | Partial: no test for absence from list, no standalone endpoint test, no 404 test for history |
| BR-H11 immutability | None | None | Not covered: no test attempts update/delete of a row |
| BR-H12 cascade deletion | None | None | Not covered: the delete test asserts only the 409 guard (`tests/orders.test.ts:197-218`); no successful delete + history disappearance test |
| BR-H13 user restrict | None | None | Not covered: no user-deletion attempt with history present |
| BR-H14 audit scope | None | None | Not covered: no test for deletion audit, stock movement, or field changes |
| BR-H15 uniqueness/idempotency | None | None | Not covered: no concurrent request test |
| BR-H16 enum/transition domain | `tests/orders.test.ts:89-107` | Illegal `PENDING -> SHIPPED` returns 409 | Partial: no DB-level or direct-write test |
| BR-H17 chain consistency | None | None | Not covered: no validation test |

### 11.3 Coverage metrics

Coverage percentages cannot be reported because no coverage provider is configured in `vitest.config.ts` and no coverage script exists in `package.json`. The suite is integration-only and exercises history through two assertions in one test file. The number of distinct history behaviors asserted is approximately two out of the seventeen documented rules, with partial assertions in both cases.

### 11.4 Test quality assessment

The existing assertions are meaningful for the happy path: they prove that creation produces one row and that a status change produces a second row with the expected resulting statuses. They use a real database through Prisma, so they validate the migration, the enum mapping, the nested write, and the read include end to end.

The suite does not test the properties that distinguish an audit trail from an ordinary log: immutability, completeness, ordering determinism, actor attribution, retention, and tamper resistance. It also does not test negative writes, so a regression that inserts history before validation or after a failed stock check would not be detected. The absence of coverage instrumentation means there is no measured baseline against which to detect drift.

### 11.5 Test file locations

- `tests/orders.test.ts:37-38` - creation history assertions.
- `tests/orders.test.ts:76-83` - status transition with reason and history sequence assertion.
- `tests/orders.test.ts:89-107` - invalid transition, no history assertion.
- `tests/orders.test.ts:109-132` - insufficient stock, no history assertion.
- `tests/orders.test.ts:134-161` - cancellation and stock replenishment, no history assertion.
- `tests/orders.test.ts:197-218` - delete guard, no successful deletion or cascade assertion.
- `tests/setup.ts:8-16` - history truncation and cleanup order.
- `tests/helpers/factories.ts:1-83` - fixtures; no history factory.
- `prisma/seed.ts:181,307-320` - history deletion and reconstruction in seed data.

---

## Appendix A: File Inventory and References

| Path | Lines | Role in this component |
|---|---|---|
| `prisma/schema.prisma` | 16-23 | `OrderStatus` enum used by `fromStatus`/`toStatus` |
| `prisma/schema.prisma` | 34-35 | `User.statusHistoryChanges` back-relation |
| `prisma/schema.prisma` | 90 | `Order.history` back-relation |
| `prisma/schema.prisma` | 116-131 | `OrderStatusHistory` model definition |
| `prisma/migrations/20260519182739_init/migration.sql` | 87-99 | Table creation, indexes, primary key |
| `prisma/migrations/20260519182739_init/migration.sql` | 122 | `ON DELETE CASCADE` to `orders` |
| `prisma/migrations/20260519182739_init/migration.sql` | 125 | `ON DELETE RESTRICT` to `users` |
| `src/modules/orders/order.service.ts` | 95-120 | Creation nested history write and include |
| `src/modules/orders/order.service.ts` | 126-179 | Transition write, history insert, refresh |
| `src/modules/orders/order.repository.ts` | 1-16 | `OrderStatusHistory` type and `OrderWithRelations` |
| `src/modules/orders/order.repository.ts` | 47-60 | Read include with ascending order |
| `src/modules/orders/order.controller.ts` | 19-46 | Read/create/change handlers and user id propagation |
| `src/modules/orders/order.routes.ts` | 12-27 | HTTP surface and authentication |
| `src/modules/orders/order.schemas.ts` | 18-21 | `toStatus` enum and `reason` max 500 |
| `src/modules/orders/order.status.ts` | 3-14 | Legal transition table used before writing |
| `src/middlewares/auth.middleware.ts` | 27-47 | JWT verification and `req.user.id` |
| `src/middlewares/error.middleware.ts` | 14-65 | Error mapping for write failures |
| `prisma/seed.ts` | 181,307-320 | Development delete/recreate path |
| `tests/setup.ts` | 8-16 | Test truncation |
| `tests/orders.test.ts` | 37-38,76-83,89-132,134-161,197-218 | History assertions and related scenarios |
| `tests/helpers/factories.ts` | 1-83 | Fixtures; no history factory |
| `vitest.config.ts` | 1-19 | Test configuration; no coverage provider |
| `package.json` | 17,26,48 | Test script, Prisma client, Prisma CLI |

## Appendix B: Assumptions, Ambiguities, and Confidence

- **No database was inspected.** Index behavior, implicit FK indexes, timestamp precision, and strict-mode behavior are derived from the migration SQL and documented MySQL/Prisma semantics. The actual MySQL session timezone and `sql_mode` are not verifiable without a running database.
- **"Append-only" is an application convention.** It is stated as a business rule because the production code only inserts, but it is explicitly not enforced at the database level. Confidence in the convention is high; confidence in immutability as a guarantee is none.
- **Audit scope is inferred from absence.** The statement that deletions, failures, and field changes are not audited is based on a complete search of `src/` for `orderStatusHistory` and on the absence of other write paths. It is high confidence for the current commit but would change if new code is added.
- **Concurrency duplicates are a structural possibility, not an observed event.** The race is inferred from the read-then-write sequence in `changeStatus` and the absence of a lock or version check. The repository has no concurrency test.
- **`changedAt` default semantics.** The migration uses `CURRENT_TIMESTAMP(3)`, which MySQL evaluates at statement execution. Whether two statements in one transaction can receive the exact same millisecond depends on execution timing; the ordering rule is therefore described as potentially ambiguous, not guaranteed to collide.
- **Implicit FK index on `changedById`.** InnoDB automatically creates an index for a foreign-key column when no suitable index exists. The migration does not declare one explicitly, so the index name and existence are an InnoDB behavior rather than a Prisma declaration.
- **No dedicated endpoint.** The absence of a history route is confirmed by the complete route file (`order.routes.ts`) and by a search of the orders module. No other router mounts a history path.
- **No event emission.** A search of `src/` found no event bus, outbox, queue, or webhook code. The status change and history insert are synchronous database writes only.

---

**End of report.**
