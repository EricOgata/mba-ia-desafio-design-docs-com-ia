# Component Deep Analysis Report

**Component:** Order Service
**Primary file:** `src/modules/orders/order.service.ts`
**Project:** `mba-ia-desafio-design-docs-com-ia` (`order-management-api`)
**Analysis date:** 2026-11-08 23:17:49
**Scope:** The `OrderService` class and the runtime collaborators required to explain its behaviour: `src/modules/orders/order.schemas.ts` (input contract), `src/modules/orders/order.status.ts` (transition predicates), `src/modules/orders/order.repository.ts` (persistence entry points used by three of its methods), `src/shared/errors/` (error taxonomy), `src/shared/http/response.ts` (pagination envelope), the `Order`/`OrderItem`/`OrderStatusHistory`/`OrderNumberSequence` models in `prisma/schema.prisma`, and the tests that exercise the service through HTTP.
**Excluded from analysis:** `node_modules`, `.git`, `dist`, `coverage`.
**Explicitly out of scope (covered by sibling analyses):** the HTTP route surface (`order.routes.ts`, `order.controller.ts` as a component), the full repository component, and the full state-machine component. They are referenced here only where `OrderService` depends on them.

---

## 1. Executive Summary

`OrderService` (`src/modules/orders/order.service.ts`, 255 lines, one exported class) is the application-service layer of the Orders module. It is the only place in the codebase where an order lifecycle is orchestrated end to end: it prices an order, reserves a human-readable order number, persists the order aggregate, enforces the status-transition table, applies stock side effects, and appends audit history. It exposes five public methods (`list`, `getById`, `create`, `changeStatus`, `delete`) and four private helpers (`aggregateItems`, `debitStock`, `replenishStock`, `reserveOrderNumber`).

The component is wired manually in `src/app.ts:42-44` and consumed by `OrderController` through constructor injection (`src/modules/orders/order.controller.ts:7`). It receives two collaborators: `OrderRepository` (`src/modules/orders/order.repository.ts:18`) and a raw `PrismaClient`. The second dependency is deliberately used to bypass the repository layer: `create()` and `changeStatus()` read and write `customers`, `products`, `orders`, `order_items`, `order_status_history` and `order_number_sequence` directly through an interactive `Prisma.TransactionClient` (`order.service.ts:58-123`, `131-178`, `204-254`). Three methods (`list`, `getById`, `delete`) go through the repository; two do not.

The service owns the following domain responsibilities, each analysed as a business rule in Section 3:

- Item normalisation and server-side pricing (`aggregateItems`, `75-84`).
- Existence and eligibility checks for customer and products (`59-73`).
- Money arithmetic in integer cents (`86-92`).
- Discount validation (`87-91`).
- Order-number generation through a single-row sequence (`245-254`).
- Initial state assignment (`PENDING`) and initial audit entry (`99-113`).
- Transition validation via the state-machine predicates (`147-149`).
- Stock debit and replenishment as lifecycle side effects (`151-156`, `204-243`).
- Append-only status history on every transition (`159-167`).
- Deletion eligibility (`181-192`).

Key findings:

- The service is the most business-critical and most coupled unit in the application. It has two afferent dependents (`OrderController`, composition root) and six efferent module dependencies (Prisma client, repository types, schema types, error taxonomy, state machine, pagination helper).
- Two commands, `create` and `changeStatus`, are transactional units of work. Any thrown `AppError` aborts the interactive transaction, so order rows, item rows, history rows, stock changes and sequence increments commit or roll back together (`58`, `131`).
- Stock is intentionally not validated or reserved at creation. It is checked only on the `PENDING -> PAID` transition (`151-153`, `204-231`). The integration test at `tests/orders.test.ts:109-132` confirms that an order can be created with a quantity greater than available stock and only fails later with `422 INSUFFICIENT_STOCK`.
- The stock check is a read-check-then-write sequence with no row lock, no conditional decrement and no version column (`208-230`). Under concurrent transitions the same order can be debited twice, and the same product can be decremented below zero.
- Order numbering is globally serialised on one physical row, `order_number_sequence.id = 1` (`245-251`). Every order creation holds an exclusive InnoDB row lock on that row until commit, which caps order-creation throughput at one writer per database and makes the sequence row the hottest write in the system.
- All lifecycle writes are audited inside the same transaction (`106-113`, `159-167`), but deletion removes the audit trail through `ON DELETE CASCADE` (`prisma/migrations/20260519182739_init/migration.sql:116,122`). A cancelled order that is deleted loses its full history.
- Failure handling relies on the shared `AppError` hierarchy and the centralized error middleware. Domain failures are typed (`NOT_FOUND`, `VALIDATION_ERROR`, `INACTIVE_PRODUCT`, `INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK`, `INVALID_ORDER_STATE_FOR_DELETE`), but Prisma failures other than `P2002`/`P2025` surface as `500 INTERNAL_SERVER_ERROR` (`src/middlewares/error.middleware.ts:37-64`).
- The component emits no domain events, no messages and no outbound calls. `changeStatus()` is a pure database command. This is the seam where the planned webhook feature would attach, but no hook, outbox or event publisher exists today.

Confidence note: every rule in Section 3 is derived from source and is marked with a confidence level. Rules marked as implicit are behaviours that follow from the absence of a check or from database constraints, not from a written requirement.

---

## 2. Data Flow Analysis

### 2.1 `create(input, userId)` - order creation

```
1.  POST /api/v1/orders reaches buildOrderRouter (src/modules/orders/order.routes.ts:12-26).
2.  authenticate sets req.user {id, email, role} from the JWT (src/middlewares/auth.middleware.ts:27-47).
3.  validate({ body: createOrderSchema }) parses and replaces req.body; defaults are applied here
    (discountCents = 0) and unknown fields are stripped (src/middlewares/validate.middleware.ts:14-16).
4.  OrderController.create checks req.user and calls orders.create(req.body, req.user.id)
    (src/modules/orders/order.controller.ts:28-36).
5.  OrderService.create guards items.length === 0 (order.service.ts:51-53).
6.  aggregateItems merges duplicate productId lines and sums quantities (order.service.ts:55, 194-202).
7.  prisma.$transaction opens an interactive transaction (order.service.ts:58).
8.  tx.customer.findUnique validates the customer exists, else NotFoundError('Customer') (59-60).
9.  tx.product.findMany loads the distinct products by id (62).
10. Cardinality check: products.length !== productIds.length -> NotFoundError('Product') (63-65).
11. Active check: any product with active = false -> 422 INACTIVE_PRODUCT with the list of SKUs (66-73).
12. Items are built with unitPriceCents copied from product.priceCents and totalCents = price * quantity (75-84).
13. subtotalCents is reduced from item totals (86).
14. discountCents > subtotalCents -> 400 VALIDATION_ERROR (87-91).
15. totalCents = subtotalCents - discountCents (92).
16. reserveOrderNumber(tx) upserts order_number_sequence id = 1 and formats ORD-%06d (93, 245-254).
17. tx.order.create writes the order, its items (nested create) and the initial history row
    (fromStatus null, toStatus PENDING, reason 'order created', changedById = userId) (95-120).
18. The same nested create returns items + product summary, history ordered ascending, customer summary (115-119).
19. Transaction commits; the controller answers 201 with the aggregate (order.controller.ts:32).
```

Failure semantics: any `AppError` thrown between steps 7 and 19 rolls back every write in the transaction, including the sequence increment. Because the sequence is a normal InnoDB row, a rollback restores the previous `nextValue`, so failed creations do not consume order numbers.

### 2.2 `changeStatus(id, input, userId)` - lifecycle transition

```
1.  PATCH /api/v1/orders/:id/status reaches the router; params and body are validated with
    orderIdParamSchema and updateOrderStatusSchema (order.routes.ts:19-23).
2.  OrderController.changeStatus checks req.user and calls orders.changeStatus(id, body, user.id)
    (order.controller.ts:38-46).
3.  OrderService.changeStatus opens an interactive transaction (order.service.ts:131).
4.  tx.order.findUnique loads the order and its items (132-136); no row lock is taken.
5.  Missing order -> NotFoundError('Order') (136).
6.  from === to -> 409 ConflictError INVALID_STATUS_TRANSITION (140-146).
7.  canTransition(from, to) from order.status.ts -> 409 InvalidStatusTransitionError (147-149).
8.  shouldDebitStock(from, to) -> debitStock(tx, items) (151-153, 204-231).
9.  shouldReplenishStock(from, to) -> replenishStock(tx, items) (154-156, 233-243).
10. tx.order.update writes the new status (158).
11. tx.orderStatusHistory.create appends fromStatus, toStatus, changedById, reason (159-167).
12. tx.order.findUnique re-reads the aggregate with items + product summary, history ascending, customer summary (169-176).
13. The refreshed order is returned and the controller answers 200 (order.controller.ts:42).
```

Failure semantics: a stock shortfall (step 8) throws `InsufficientStockError` and rolls back the whole transaction, so the status is not changed and no history row is written. The test at `tests/orders.test.ts:109-132` verifies both the 422 response and that stock remains unchanged.

### 2.3 `delete(id)`

```
1.  DELETE /api/v1/orders/:id -> authenticate -> validate(orderIdParamSchema) (order.routes.ts:24).
2.  OrderController.delete calls orders.delete(id) (order.controller.ts:48-55).
3.  OrderService.delete calls this.orders.findById(id) outside any explicit transaction (order.service.ts:182).
4.  Missing order -> NotFoundError('Order') (183).
5.  status not in {PENDING, CANCELLED} -> 409 INVALID_ORDER_STATE_FOR_DELETE (184-190).
6.  this.orders.deleteById(id) issues a single Prisma delete (191; order.repository.ts:66-68).
7.  MySQL cascades the delete to order_items and order_status_history (migration.sql:116, 122).
8.  Controller answers 204 with an empty body (order.controller.ts:51).
```

Failure semantics: there is no transaction and no re-check between step 5 and step 6, so a concurrent transition can move the order out of the deletable states after the guard has passed.

### 2.4 `list(query)` and `getById(id)`

```
list:
1.  GET /api/v1/orders -> authenticate -> validate({query: listOrdersQuerySchema}); page, pageSize and
    date coercion/defaults are applied by Zod (order.schemas.ts:23-30).
2.  OrderController.list forwards the validated query as ListOrdersQuery (order.controller.ts:9-17).
3.  OrderService.list computes skip = (page - 1) * pageSize and calls orders.list with status,
    customerId, from, to, skip, take (order.service.ts:32-40).
4.  OrderRepository.list builds the where clause and runs findMany + count in a batch transaction,
    ordered by createdAt desc (order.repository.ts:21-45).
5.  paginated(items, page, pageSize, total) adds totalPages (order.service.ts:41; response.ts:13-24).
6.  Controller answers 200 { data, pagination } (order.controller.ts:13).

getById:
1.  GET /api/v1/orders/:id -> authenticate -> validate(orderIdParamSchema) (order.routes.ts:17).
2.  OrderService.getById calls orders.findByIdWithRelations (order.service.ts:44-48).
3.  Missing order -> NotFoundError('Order') (46).
4.  Controller answers 200 with items + product summary, history ascending, customer summary
    (order.repository.ts:47-60; order.controller.ts:22).
```

### 2.5 Cross-cutting data observations

- The service never maps Prisma rows to domain objects. Responses are raw `Order` rows with selected relations, so persistence shape is the API contract.
- `create` and `changeStatus` return structurally identical aggregates, but `create` uses the rows returned by the nested write while `changeStatus` performs an extra read (`169-176`).
- The service passes `Prisma.TransactionClient` into private helpers; transaction scope is the only thing that keeps the multi-table lifecycle consistent.
- No data leaves the process. There is no event, log or metric emitted by the service itself; only thrown errors and returned rows.
---

## 3. Business Rules & Logic

### Overview of the business rules

| ID | Rule Type | Rule Description | Location |
|----|-----------|------------------|----------|
| BR-01 | Validation | An order must contain at least one item | `src/modules/orders/order.schemas.ts:13`; `src/modules/orders/order.service.ts:51-53` |
| BR-02 | Business Logic | Duplicate product lines are aggregated into one line per product with summed quantity | `src/modules/orders/order.service.ts:55,194-202` |
| BR-03 | Validation | The referenced customer must exist | `src/modules/orders/order.service.ts:59-60` |
| BR-04 | Validation | Every referenced product must exist | `src/modules/orders/order.service.ts:62-65` |
| BR-05 | Validation | Every referenced product must be active at creation time | `src/modules/orders/order.service.ts:66-73` |
| BR-06 | Computation | Unit price is taken from the product record on the server; clients cannot supply prices | `src/modules/orders/order.service.ts:75-84`; `src/modules/orders/order.schemas.ts:6-9` |
| BR-07 | Computation | Line totals, subtotal and total are computed in integer cents | `src/modules/orders/order.service.ts:82,86,92` |
| BR-08 | Validation | Discount must be a non-negative integer and must not exceed the subtotal | `src/modules/orders/order.schemas.ts:14`; `src/modules/orders/order.service.ts:87-91` |
| BR-09 | Business Logic | A new order always starts in `PENDING` | `src/modules/orders/order.service.ts:99,109`; `prisma/schema.prisma:78` |
| BR-10 | Business Logic | Order numbers follow `ORD-NNNNNN` and are issued from a single global sequence row | `src/modules/orders/order.service.ts:245-254` |
| BR-11 | Audit | Creation writes an initial history row from `null` to `PENDING` with reason `order created` | `src/modules/orders/order.service.ts:106-113` |
| BR-12 | Audit | Created-by and changed-by identities come from the authenticated principal | `src/modules/orders/order.service.ts:104,110,164` |
| BR-13 | Validation | Notes are optional, at most 1000 characters, and stored as `null` when absent | `src/modules/orders/order.schemas.ts:15`; `src/modules/orders/order.service.ts:103` |
| BR-14 | Data Integrity | Creation is one atomic unit of work across customer/product reads, sequence, order, items and history | `src/modules/orders/order.service.ts:58-123` |
| BR-15 | Business Logic | Stock is neither validated nor reserved when the order is created | `src/modules/orders/order.service.ts:62-73`; `tests/orders.test.ts:109-132` |
| BR-16 | Business Logic | Only transitions in the state-machine table are allowed | `src/modules/orders/order.status.ts:3-14`; `src/modules/orders/order.service.ts:147-149` |
| BR-17 | Validation | A transition to the current status is rejected with 409 | `src/modules/orders/order.service.ts:140-146` |
| BR-18 | Invariant | `DELIVERED` and `CANCELLED` are terminal states | `src/modules/orders/order.status.ts:7-10,20-22` |
| BR-19 | Business Logic | Stock is debited exactly on the `PENDING -> PAID` transition | `src/modules/orders/order.status.ts:24-31`; `src/modules/orders/order.service.ts:151-153` |
| BR-20 | Validation | A debit requires sufficient stock for every line and is all-or-nothing | `src/modules/orders/order.service.ts:208-230` |
| BR-21 | Business Logic | Stock is replenished on `PAID -> CANCELLED` and `PROCESSING -> CANCELLED` only | `src/modules/orders/order.status.ts:33-37`; `src/modules/orders/order.service.ts:154-156` |
| BR-22 | Data Integrity | Status change, stock movement and audit append are one atomic unit of work | `src/modules/orders/order.service.ts:131-178` |
| BR-23 | Audit | Every accepted transition appends an immutable-in-service history row | `src/modules/orders/order.service.ts:159-167` |
| BR-24 | Validation | Transition reason is optional, at most 500 characters, stored as `null` when absent | `src/modules/orders/order.schemas.ts:20`; `src/modules/orders/order.service.ts:165` |
| BR-25 | Read Model | Mutation responses always include items with product summary, history ascending and customer summary | `src/modules/orders/order.service.ts:115-119,169-176`; `order.repository.ts:47-60` |
| BR-26 | Business Logic | Only `PENDING` or `CANCELLED` orders may be deleted | `src/modules/orders/order.service.ts:184-190` |
| BR-27 | Data Integrity | Deleting an order cascades to its items and its entire status history | `prisma/migrations/20260519182739_init/migration.sql:116,122` |
| BR-28 | Concurrency | The delete eligibility check is performed outside any transaction | `src/modules/orders/order.service.ts:182-191` |
| BR-29 | Validation | List pagination defaults to page 1, page size 20, maximum page size 100 | `src/modules/orders/order.schemas.ts:24-25`; `src/modules/orders/order.service.ts:38` |
| BR-30 | Business Logic | List supports status, customer, and creation-date filters, newest first, without relations | `src/modules/orders/order.service.ts:33-40`; `order.repository.ts:21-45` |
| BR-31 | Business Logic | Reading a missing order by id returns 404 | `src/modules/orders/order.service.ts:44-48` |
| BR-32 | Authorization | The service applies no ownership, tenant or role scoping | `src/modules/orders/order.service.ts:32-48,126-130,181-192`; `src/modules/orders/order.routes.ts:14` |
| BR-33 | Business Logic | Product active status is enforced only at creation, never at later transitions | `src/modules/orders/order.service.ts:66-73,126-179` |

Confidence summary: BR-01 to BR-14 and BR-16 to BR-31 are directly evidenced by code and are high confidence. BR-15 is high confidence as behaviour (creation succeeds without stock) and medium confidence as intent (deferred stock control is consistent with the tests, but no written requirement in the repository states it). BR-32 is high confidence as behaviour and medium confidence as a deliberate decision, because no authorization policy document exists in the repository. BR-33 is high confidence as behaviour, medium confidence as intent.

---

### Detailed breakdown of the business rules

---

### Business Rule BR-01: Minimum item requirement

**Overview**:
An order cannot be persisted unless it carries at least one order line.

**Location**: `src/modules/orders/order.schemas.ts:13`; `src/modules/orders/order.service.ts:51-53`
**Confidence**: High

**Detailed description**:
The rule exists twice, at two different layers. The HTTP contract declares `items: z.array(orderItemInputSchema).min(1, 'order must contain at least one item')`, so a body with an empty array is rejected by the validation middleware before the controller executes, producing `400 VALIDATION_ERROR` with a Zod issue at path `items`. The service then repeats the check with `if (input.items.length === 0) throw new ValidationError('Order must contain at least one item')`. This is deliberate defence in depth: any programmatic caller that bypasses the HTTP layer still cannot create an empty order.

The two layers produce different error payloads for the same conceptual violation. The schema path produces a `details` array of Zod issues; the service path produces a `VALIDATION_ERROR` with no `details`. In normal HTTP operation only the schema path is reachable, so the service guard is effectively dead code from the API perspective. It remains relevant for direct service use, for future internal callers, and for tests that instantiate the service without the middleware.

The rule protects several downstream invariants at once. An empty order would have `subtotalCents = 0`, a zero or zero-clamped total, an initial history row with no economic meaning, and no product references from which stock could ever be debited. By rejecting at the boundary, the service guarantees that every persisted order has at least one `OrderItem` and therefore a well-defined price basis.

**Rule workflow**:
1. Request body reaches `validate({ body: createOrderSchema })`.
2. Zod parses `items`; an empty array raises `ZodError` and the middleware converts it to `ValidationError` (400).
3. If validation is bypassed, `OrderService.create` evaluates `input.items.length === 0`.
4. On failure it throws `ValidationError('Order must contain at least one item')` before opening a transaction.
5. On success it proceeds to aggregation and the transactional write.

---

### Business Rule BR-02: Duplicate line aggregation

**Overview**:
Multiple input lines that reference the same product are collapsed into a single order line whose quantity is the sum of the input quantities.

**Location**: `src/modules/orders/order.service.ts:55,194-202`
**Confidence**: High

**Detailed description**:
`aggregateItems` iterates the validated items and accumulates quantities in a `Map<string, number>` keyed by `productId`, then returns an array of `{ productId, quantity }` in first-appearance order. If a client submits `[{A, 2}, {B, 1}, {A, 3}]`, the persisted order contains two items, with `A` carrying quantity 5. The order of the resulting array follows insertion order of the map, so the first occurrence of a product determines its position in the response.

Aggregation has three observable consequences. First, the price lookup and total computation operate on the aggregated quantity, so the same price snapshot applies to all units of a product. Second, the stock check compares the aggregated demand against stock once per product instead of once per input line, which avoids a false negative where two individually acceptable lines would jointly exceed stock. Third, the product existence check compares `products.length` against the number of distinct product ids, not the number of input lines, so duplicates cannot cause a spurious `NotFoundError`.

The rule also defines what the client cannot do: it cannot create two separate lines for the same product within one order, and it cannot express per-line discounts or per-line notes because the input schema has no such fields and the aggregation discards line identity. If the business ever needs line-level pricing overrides, this helper is the point where that capability is lost.

**Rule workflow**:
1. `create` calls `aggregateItems(input.items)` before opening the transaction.
2. A map accumulates `quantity` per `productId`.
3. The map is converted into an array of `{ productId, quantity }`.
4. Distinct ids drive the product lookup and existence check.
5. Aggregated quantities drive pricing, subtotal, stock checks and debit/replenish operations.

---

### Business Rule BR-03: Customer existence

**Overview**:
An order must reference an existing customer; otherwise the request fails with 404.

**Location**: `src/modules/orders/order.service.ts:59-60`
**Confidence**: High

**Detailed description**:
Immediately after opening the transaction, the service reads the customer by primary key with `tx.customer.findUnique({ where: { id: input.customerId } })`. If the row is absent it throws `NotFoundError('Customer')`, which the shared taxonomy translates into HTTP 404 with `error.code = 'NOT_FOUND'` and message `Customer not found`. The customer id itself is validated as a UUID by Zod (`order.schemas.ts:12`), so a malformed identifier is rejected as 400 before the query runs.

The check occurs inside the transaction and before any write, which means a failed lookup consumes no sequence value and creates no partial rows. It also means the check is repeatable-read consistent with the rest of the transaction: if the customer is deleted concurrently after the read, the foreign key `orders_customerId_fkey` with `ON DELETE RESTRICT` (`migration.sql:110`) prevents commit from producing a dangling reference. The service thus relies on both an application check and a database constraint.

The rule does not verify customer eligibility beyond existence. There is no active flag, credit limit, blocked status or tenant check on `Customer`; the model only has identity and address fields (`prisma/schema.prisma:40-54`). Consequently, "customer exists" is the entire eligibility policy for the ordering party, and any future eligibility rule would have to be added here.

**Rule workflow**:
1. Transaction opens.
2. `tx.customer.findUnique` by `input.customerId`.
3. Missing row -> `NotFoundError('Customer')` -> rollback -> 404.
4. Existing row -> continue to product validation.
5. The order insert later satisfies the `orders_customerId_fkey` constraint.

---

### Business Rule BR-04: Product existence

**Overview**:
Every distinct product referenced by the order must exist; otherwise the request fails with 404.

**Location**: `src/modules/orders/order.service.ts:62-65`
**Confidence**: High

**Detailed description**:
After aggregation, the service loads all distinct products with a single `tx.product.findMany({ where: { id: { in: productIds } } })` query and compares `products.length` with `productIds.length`. A mismatch means at least one id did not resolve, and the service throws `NotFoundError('Product')`, producing 404 `NOT_FOUND` with message `Product not found`. The check is intentionally coarse: the error does not reveal which ids were missing, and the response `details` is empty.

Because the comparison uses distinct ids, the rule is not confused by duplicate input lines. Because the query is a single round trip, the rule is atomic with respect to the transaction snapshot. The subsequent item construction uses a non-null assertion (`products.find(...)!`) that is safe only because the cardinality check has already guaranteed every id is present. If the check were removed or altered, that assertion would become a runtime hazard.

The rule complements the database foreign key `order_items_productId_fkey` with `ON DELETE RESTRICT` (`migration.sql:119`). The application check provides a clean 404 before any write, while the constraint prevents a product from being deleted while referenced by an item. Neither mechanism validates that the product is sellable; that is BR-05.

**Rule workflow**:
1. Distinct product ids come from the aggregated items.
2. One `findMany` loads all matching products inside the transaction.
3. Count mismatch -> `NotFoundError('Product')` -> rollback -> 404.
4. All present -> continue to active-state validation.
5. Item construction maps each aggregated id to its loaded product.

---

### Business Rule BR-05: Active-product requirement

**Overview**:
An order cannot be created with a product whose `active` flag is false.

**Location**: `src/modules/orders/order.service.ts:66-73`
**Confidence**: High

**Detailed description**:
After existence is established, the service filters the loaded products for `active === false`. If the filtered list is non-empty it throws `UnprocessableEntityError('One or more products are inactive', 'INACTIVE_PRODUCT', { skus: inactive.map((p) => p.sku) })`, which becomes HTTP 422 with `error.code = 'INACTIVE_PRODUCT'` and `error.details.skus` containing the offending SKUs. The response is deliberately more diagnostic than the existence failure: the caller learns which products must be removed or reactivated.

The check is a creation-time gate only. `changeStatus` does not re-read or re-evaluate product active state, so an order created while a product was active can still be paid, processed, shipped and delivered after the product is deactivated. This is consistent with the idea that deactivation affects new demand, not committed demand, but it means the rule cannot be used as a kill switch for in-flight orders.

Because the active check happens inside the same transaction as the insert, a concurrent deactivation cannot interleave between validation and commit under repeatable read: the transaction sees a stable snapshot. The service does not distinguish between "inactive at the time of order" and "inactive because it was deleted"; deletion is blocked by the foreign key, so only the flag path is reachable.

**Rule workflow**:
1. Products are loaded by distinct id.
2. `products.filter(p => !p.active)` collects inactive rows.
3. Non-empty -> 422 `INACTIVE_PRODUCT` with the SKU list -> rollback.
4. Empty -> proceed to pricing.
5. Later transitions do not repeat this check (see BR-33).

---

### Business Rule BR-06: Server-side price snapshot

**Overview**:
Unit prices are read from the product record at creation time and copied onto the order item; the client cannot influence price.

**Location**: `src/modules/orders/order.service.ts:75-84`; `src/modules/orders/order.schemas.ts:6-9`
**Confidence**: High

**Detailed description**:
The input schema for an order line contains only `productId` and `quantity`. Zod object parsing strips unknown keys by default, so a client that sends `unitPriceCents` or `totalCents` has those fields discarded before the service sees them. The service then sets `unitPriceCents = product.priceCents` and computes `totalCents = unitPriceCents * quantity`. This makes the server the sole authority over pricing and removes the classic tampering vector in which a client submits a lower price.

The copy is a snapshot, not a reference. If the product price changes after the order is created, the existing `OrderItem.unitPriceCents` and `Order.totalCents` remain unchanged; only future orders use the new price. This is the standard commercial expectation for an order management system and is reinforced by the denormalised columns on `OrderItem` (`prisma/schema.prisma:99-114`). The trade-off is that historical orders cannot be recomputed from the current product catalogue; the item row is the authoritative record.

The snapshot also means that price changes are not transactional with order creation. A price update committed after the service read but before commit is not observed by the order, because the transaction uses a consistent snapshot. Two orders created at nearly the same time can therefore carry different prices without any error, which is acceptable for pricing but important to know when reconciling revenue.

**Rule workflow**:
1. Validated input contains only `productId` and `quantity`.
2. Products are loaded inside the transaction.
3. For each aggregated item, `unitPriceCents` is copied from the product row.
4. `totalCents` is computed as price times quantity.
5. The snapshot is persisted on `order_items` and reflected in `orders.subtotalCents` and `orders.totalCents`.

---

### Business Rule BR-07: Monetary computation in integer cents

**Overview**:
All monetary values are integers representing cents; line totals and subtotal are pure integer arithmetic.

**Location**: `src/modules/orders/order.service.ts:82,86,92`
**Confidence**: High

**Detailed description**:
The service never uses floating point. `OrderItem.totalCents` is `unitPriceCents * quantity`, and `subtotalCents` is the sum of those products. `totalCents` is `subtotalCents - discountCents`. The database columns are `INTEGER` (`migration.sql:55-57,77-78`), so the same representation is used from request to storage to response. This eliminates rounding drift and makes equality assertions in tests exact.

The computation is performed once, at creation, and stored. Read paths do not recompute totals, so an order's monetary values are stable for its lifetime. `changeStatus` never touches money; a cancellation does not zero the totals, and a deletion removes them. There is no tax, shipping, fee or currency field in the model, so the arithmetic surface is intentionally minimal: subtotal, discount, total, all in one implicit currency.

Integer arithmetic introduces an upper bound that the service does not validate. MySQL `INTEGER` is signed 32-bit, so a unit price times quantity above 2,147,483,647 would overflow at the database layer rather than in TypeScript, producing a Prisma error that the middleware maps to 500. The input schema bounds neither `quantity` nor the product price, so the guard exists only at the storage boundary.

**Rule workflow**:
1. For each aggregated item, `totalCents = unitPriceCents * quantity`.
2. `subtotalCents` is reduced over item totals.
3. `discountCents` is validated against the subtotal (BR-08).
4. `totalCents = subtotalCents - discountCents`.
5. All three values are persisted on the order; item totals are persisted on the items.

---

### Business Rule BR-08: Discount bounds

**Overview**:
A discount must be a non-negative integer and cannot exceed the order subtotal.

**Location**: `src/modules/orders/order.schemas.ts:14`; `src/modules/orders/order.service.ts:87-91`
**Confidence**: High

**Detailed description**:
The schema declares `discountCents: z.number().int().nonnegative().default(0)`. This rejects negative values, fractional values and non-numeric values at the HTTP boundary, and supplies `0` when the field is omitted. The service adds a relational check that the schema cannot express: `if (input.discountCents > subtotalCents)` throws `ValidationError('Discount cannot exceed subtotal', [{ path: 'discountCents', message: 'Discount exceeds subtotal' }])`, producing 400 with a Zod-shaped `details` array even though the failure is a service decision.

The bound is inclusive: a discount equal to the subtotal is allowed, producing a total of zero. There is no minimum total, no maximum discount percentage, no coupon or promotion entity and no approval workflow. The service therefore permits a 100 percent discount for any authenticated caller who can reach the endpoint, and it records the resulting zero-value order like any other. Whether that is acceptable is a policy question not answered in code.

The check occurs inside the transaction after the subtotal is known, which is necessary because the subtotal depends on aggregated quantities and snapshotted prices. A failure aborts the transaction before the order number is reserved, so an invalid discount does not consume sequence values. The error message and the detail message differ slightly ("cannot exceed" versus "exceeds"), but both identify the same field.

**Rule workflow**:
1. Zod enforces integer, non-negative, default 0.
2. Items are priced and subtotal is computed.
3. `discountCents > subtotalCents` -> 400 `VALIDATION_ERROR` with detail path `discountCents`.
4. Otherwise `totalCents = subtotalCents - discountCents`.
5. Discount and total are persisted on the order.

---

### Business Rule BR-09: Initial status `PENDING`

**Overview**:
Every new order is created in `PENDING`, independently of any client input.

**Location**: `src/modules/orders/order.service.ts:99,109`; `prisma/schema.prisma:78`
**Confidence**: High

**Detailed description**:
`createOrderSchema` has no `status` field, and the service hardcodes `status: OrderStatus.PENDING` in the order insert. The database also declares `status OrderStatus @default(PENDING)`. A client cannot create an order directly in `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` or `CANCELLED`; those states are reachable only through `changeStatus` and its transition guards. This is the anchor of the lifecycle: creation produces demand, payment produces commitment, and stock is not touched until commitment.

The initial history row is written with the same value: `fromStatus: null`, `toStatus: PENDING`. The null is meaningful because there is no previous state; it distinguishes creation from a transition. The service also writes `reason: 'order created'`, a hardcoded literal that is not localised and cannot be customised by the caller.

Because status is not part of the create contract, replaying an order from a client-side payload cannot reproduce a historical state. Import or migration scenarios must use direct database access or the status endpoint with a valid path of transitions. This keeps the normal API path linear but makes bulk historical loading more expensive.

**Rule workflow**:
1. `create` builds the order data object.
2. `status` is set to `OrderStatus.PENDING`.
3. The nested history create uses `fromStatus: null` and `toStatus: PENDING`.
4. The row is committed with that status.
5. Any later state is reachable only through `changeStatus`.

---

### Business Rule BR-10: Order-number generation

**Overview**:
Order numbers are formatted `ORD-` followed by a zero-padded six-digit sequence value, allocated from a single-row table inside the creation transaction.

**Location**: `src/modules/orders/order.service.ts:245-254`; `prisma/schema.prisma:133-138`; `prisma/migrations/20260519182739_init/migration.sql:102-107`
**Confidence**: High

**Detailed description**:
`reserveOrderNumber(tx)` performs an upsert on `order_number_sequence` with fixed primary key `id = 1`. On insert it creates `{ id: 1, nextValue: 2 }`; on update it increments `nextValue` by one. It then reads `seq.nextValue`, subtracts one and formats `ORD-${String(current).padStart(6, '0')}`. The first successful call therefore returns `ORD-000001` and leaves `nextValue = 2`; the second returns `ORD-000002`, and so on. The value is stored on `Order.orderNumber`, which has a unique index (`migration.sql:63`).

The algorithm has several deliberate properties. It is gap-free for rolled-back transactions, because the increment is part of the same InnoDB transaction as the order insert and a rollback restores the previous value. It is monotonic but not time-based: there is no daily, monthly or yearly reset, and no prefix derived from the date. It is global across all customers and users, so order numbers are a single namespace and can leak approximate business volume to anyone who can read orders.

The concurrency characteristics are the most significant part of this rule. Because every creation updates the same physical row, InnoDB holds an exclusive lock on it until commit, serialising all order creation in the database. This guarantees uniqueness and ordering but makes the sequence row a throughput bottleneck and a lock-contention point. If the row is missing and two transactions race to insert it, one insert can fail with a unique-constraint error (`P2002`), which the centralized middleware maps to 409 rather than retrying. The format also degrades beyond 999,999: `padStart` does not truncate, so the millionth order is `ORD-1000000`, seven digits, which still fits `VARCHAR(20)` but no longer matches the documented/tested six-digit shape.

**Rule workflow**:
1. Order total is computed and validated.
2. `reserveOrderNumber(tx)` upserts row `id = 1`.
3. New row case: `nextValue` becomes 2; existing row case: it is incremented by one.
4. `current = nextValue - 1`.
5. The value is formatted as `ORD-NNNNNN` with at least six digits.
6. The order insert persists it under a unique index; a duplicate would raise `P2002` and fail the transaction.

---

### Business Rule BR-11: Initial audit entry

**Overview**:
Creation writes the first `OrderStatusHistory` row describing the transition from no state to `PENDING`.

**Location**: `src/modules/orders/order.service.ts:106-113`
**Confidence**: High

**Detailed description**:
The order insert uses a nested `history.create` with `fromStatus: null`, `toStatus: OrderStatus.PENDING`, `changedById: userId` and `reason: 'order created'`. The row is created in the same statement as the order and the items, so an order can never exist without its initial history entry. The null `fromStatus` is the only representation of creation in the audit model, because the enum has no `CREATED` member.

The audit row is append-only from the service's perspective: there is no update or delete method for history. The only way history disappears is a cascade delete of the parent order (BR-27). `changedAt` is defaulted by the database (`prisma/schema.prisma:121`), not supplied by the service, so ordering relies on database time. When history is read back it is ordered ascending by `changedAt` (`order.repository.ts:56`, `order.service.ts:117,173`).

The reason literal is fixed and English. It is not derived from a request field, so a caller cannot annotate creation. This is consistent with the create schema having no `reason` field, but it means that distinctions such as "created by import" or "created by API" are not representable without a schema change.

**Rule workflow**:
1. The order data object includes `history.create`.
2. The nested write inserts one history row with `fromStatus = null`.
3. `toStatus` is `PENDING`, `changedById` is the authenticated user.
4. `reason` is the literal `order created`.
5. On read, the row appears first in ascending `changedAt` order.

---

### Business Rule BR-12: Actor attribution from the authenticated principal

**Overview**:
`createdById` and `changedById` are taken from the authenticated user id supplied by the controller, never from the request body.

**Location**: `src/modules/orders/order.service.ts:104,110,164`; `src/modules/orders/order.controller.ts:30-31,40-41`
**Confidence**: High

**Detailed description**:
The service signature requires a `userId: string` on both `create` and `changeStatus`. The controller obtains it from `req.user.id`, which the authentication middleware populates from the verified JWT `sub` claim (`src/middlewares/auth.middleware.ts:41-42`). No request schema includes a user or author field, so a caller cannot impersonate another user by adding one to the body. This is the strongest identity control in the component.

The value is written in three places: `Order.createdById`, the creation history `changedById`, and every transition history `changedById`. Both columns have foreign keys to `users` with `ON DELETE RESTRICT` (`migration.sql:113,125`), so a user referenced by an order or a history row cannot be deleted. This preserves attribution but also means user deletion must be handled by deactivation rather than removal.

Attribution is not validated beyond the foreign key. The service does not check that the user is active, that the role permits the operation, or that the user owns the order. `USER_ID` is trusted because it came from a signed token, but any authenticated user can change the status of any order (BR-32). The audit trail therefore answers "who did it" reliably but not "was that person allowed to do it".

**Rule workflow**:
1. `authenticate` verifies the JWT and sets `req.user`.
2. The controller rejects a missing user with `UnauthorizedError`.
3. The controller passes `req.user.id` as `userId`.
4. The service writes it to `createdById` and to every history row it creates.
5. Database foreign keys enforce that the referenced user exists.

---

### Business Rule BR-13: Optional notes

**Overview**:
Orders may carry free-text notes of up to 1000 characters; absence is stored as SQL `NULL`.

**Location**: `src/modules/orders/order.schemas.ts:15`; `src/modules/orders/order.service.ts:103`; `prisma/schema.prisma:82`
**Confidence**: High

**Detailed description**:
The schema declares `notes: z.string().max(1000).optional()`, so the field is not required and longer values are rejected as 400. The service applies `notes: input.notes ?? null`, which means `undefined` becomes `NULL` but an empty string remains an empty string. There is no trimming, no minimum length and no content validation, so a note consisting only of whitespace is accepted and stored.

The field is part of the order, not of individual items or history entries. It is written once at creation and is never updated by any service method; `changeStatus` accepts a `reason` but not a note. Consequently, notes are immutable through the API even though they are free text and may contain corrections or operational context that later needs revision. Any update would require direct database access or a new endpoint.

Notes are returned in every aggregate response because the service returns raw order rows. This means the field is visible to anyone who can read the order, with no redaction or role-based masking. If notes are used for internal operational comments, that visibility is a data-exposure consideration rather than a functional defect.

**Rule workflow**:
1. Zod enforces optional and maximum length 1000.
2. The service maps `undefined` to `null`.
3. The value is persisted in `orders.notes`.
4. It is returned verbatim on reads.
5. No service method updates it after creation.

---

### Business Rule BR-14: Creation atomicity

**Overview**:
Order creation is a single interactive transaction covering validation reads, sequence allocation, order, items and history.

**Location**: `src/modules/orders/order.service.ts:58-123`
**Confidence**: High

**Detailed description**:
`create` wraps every read and write after the initial input-length guard in `this.prisma.$transaction(async (tx) => { ... })`. The transaction reads the customer, reads the products, upserts the sequence, inserts the order with nested items and history, and returns the created aggregate. No write occurs outside the callback. A thrown `AppError` or a Prisma failure propagates out of the callback, Prisma issues a rollback, and the controller forwards the error to the centralized middleware.

Atomicity matters most for the combination of sequence allocation and order insert. Because both are in the same transaction, an order number cannot be consumed by an order that was never created, and an order cannot be created without a number. It also guarantees that the nested items and the initial history row are all-or-nothing with the order, so no partial aggregate can exist.

The transaction does not set an explicit isolation level, so it inherits the MySQL/InnoDB default (repeatable read) through Prisma. It also does not set a custom timeout, so Prisma's default interactive-transaction limits apply (framework behavior: a maximum wait to acquire a connection and a bounded execution time). Under sequence lock contention, a creation can therefore wait and eventually fail with a Prisma transaction error that the middleware does not map to a 4xx, surfacing as 500.

**Rule workflow**:
1. Input guard runs before the transaction.
2. Transaction opens with the default Prisma options.
3. All validation reads and all writes use `tx`.
4. Any error aborts the callback.
5. Commit publishes order, items, history and sequence together.
6. The created aggregate is returned to the controller.

---

### Business Rule BR-15: No stock validation or reservation at creation

**Overview**:
Creating an order does not check available stock and does not reserve inventory; stock is only examined when the order is paid.

**Location**: `src/modules/orders/order.service.ts:62-73`; `tests/orders.test.ts:109-132`
**Confidence**: High (behaviour); Medium (intent)

**Detailed description**:
The creation flow loads products to validate existence and active status, and reads `priceCents`, but never reads `stockQuantity`. There is no reservation table, no `reservedQuantity` column and no decrement at creation. An order for five units of a product with one unit in stock is accepted and persisted as `PENDING`; the integration test at `tests/orders.test.ts:109-132` asserts exactly this, expecting 201 on creation and only later a 422 on the `PENDING -> PAID` transition.

The rationale is visible in the lifecycle design: `shouldDebitStock` triggers only for `PENDING -> PAID`, and replenishment only for cancellation after payment. Creation is treated as an intent, not a commitment. This avoids holding inventory for abandoned carts, but it also means that accepted orders are not guaranteed to be fulfillable and that multiple pending orders can compete for the same stock. The first one paid wins; the others fail at payment.

The absence of reservation is an implicit rule, not an enforced one. Nothing prevents a future implementation from adding a reservation step; nothing documents the current policy outside the code and tests. The practical consequence for the API consumer is that a 201 response is not a stock promise, and for operations it is that oversell exposure is deferred to the payment moment rather than eliminated.

**Rule workflow**:
1. Creation reads products for existence, active flag and price only.
2. No `stockQuantity` comparison occurs.
3. The order is persisted as `PENDING` regardless of stock.
4. Stock is first evaluated on `PENDING -> PAID` (BR-19, BR-20).
5. If stock is insufficient at payment, the transition fails with 422 and the order remains `PENDING`.

---

### Business Rule BR-16: Transition table enforcement

**Overview**:
A status change is accepted only if the pair `(from, to)` appears in the static transition table.

**Location**: `src/modules/orders/order.status.ts:3-14`; `src/modules/orders/order.service.ts:147-149`
**Confidence**: High

**Detailed description**:
`order.status.ts` defines a frozen-in-practice record mapping each status to its allowed successors: `PENDING -> [PAID, CANCELLED]`, `PAID -> [PROCESSING, CANCELLED]`, `PROCESSING -> [SHIPPED, CANCELLED]`, `SHIPPED -> [DELIVERED]`, `DELIVERED -> []`, `CANCELLED -> []`. `canTransition(from, to)` performs an array inclusion check. `changeStatus` calls it after the same-status guard and throws `InvalidStatusTransitionError(from, to)` when it returns false. That error extends `ConflictError`, so the HTTP response is 409 with `error.code = 'INVALID_STATUS_TRANSITION'` and `error.details = { from, to }`.

The table is the single source of truth for the lifecycle graph. Because it is a plain in-memory constant, the rule is enforced only when writes go through `OrderService`. Direct SQL updates, seed scripts and any future bulk job bypass it. There is no database-level constraint, trigger or version column that would reject an out-of-band transition, and the seed script writes historical statuses and histories directly (`prisma/seed.ts:287-321`).

The table is also not exposed through the API. `allowedTransitions` and `isTerminal` are exported but unused by the service, so a client cannot discover the valid next states and must rely on documentation or trial and error. The ordering of the arrays is not semantically meaningful, and the record is not deeply frozen, so mutation at runtime is technically possible even though no code does it.

**Rule workflow**:
1. `changeStatus` loads the current order status.
2. Same-status guard runs first (BR-17).
3. `canTransition(from, to)` looks up `transitions[from]` and checks inclusion.
4. False -> 409 `INVALID_STATUS_TRANSITION` with `{ from, to }`.
5. True -> stock predicates and persistence continue.

---

### Business Rule BR-17: Same-status rejection

**Overview**:
Requesting the status the order already has is rejected with 409 rather than treated as a no-op.

**Location**: `src/modules/orders/order.service.ts:140-146`
**Confidence**: High

**Detailed description**:
Before consulting the transition table, the service compares `from === to` and throws `ConflictError('Order is already in ${to} status', 'INVALID_STATUS_TRANSITION', { from, to })`. The choice of `ConflictError` yields 409, and the explicit code reuses the same machine-readable value as an illegal transition. As a result, clients cannot distinguish "already there" from "not allowed from here" by code alone; they must inspect the message or the `details` pair.

The rule makes the command non-idempotent. A retried PATCH with the same target status returns an error instead of success, which complicates client retry logic after a timeout: if the first request succeeded but the response was lost, the retry fails with 409 rather than confirming the state. There is no idempotency key or request de-duplication anywhere in the flow.

Because the same-status case is handled before `canTransition`, it applies even to terminal states. A request to move `DELIVERED` to `DELIVERED` returns the same explicit conflict message rather than an empty allowed-transitions failure. The history row is not written and no stock movement occurs, since the throw happens before any side effect.

**Rule workflow**:
1. Load the order inside the transaction.
2. Compare `order.status` with `input.toStatus`.
3. Equal -> throw `ConflictError` with code `INVALID_STATUS_TRANSITION`.
4. Transaction rolls back (no writes have occurred).
5. Response is 409 with `details: { from, to }`.

---

### Business Rule BR-18: Terminal states

**Overview**:
`DELIVERED` and `CANCELLED` have no outgoing transitions and are therefore terminal.

**Location**: `src/modules/orders/order.status.ts:7-10,20-22`
**Confidence**: High

**Detailed description**:
The transition table maps both statuses to an empty array. `canTransition` therefore returns false for every attempted move out of them, producing 409. `isTerminal(status)` encodes the same fact by checking whether the allowed array is empty, but the service does not call it; the behaviour comes from `canTransition`. Operationally, a delivered order cannot be cancelled or returned through this API, and a cancelled order cannot be reinstated.

Terminality has stock implications. Cancellation is the only path that replenishes stock, and it must occur before delivery. Once an order is `DELIVERED`, there is no service-level mechanism to return inventory, issue a refund marker or reopen the order. A post-delivery return would require a new order or direct database intervention. Conversely, once an order is `CANCELLED`, any stock that was replenished cannot be debited again for the same order.

The terminal set is a domain decision embedded in code with no accompanying documentation of return/refund policy. The model has no `returned` or `refunded` status, and the history table has no reversal semantics. Any future lifecycle extension would require changing the enum, the table and the migration, which is a cross-cutting schema change rather than a configuration change.

**Rule workflow**:
1. A transition request names `DELIVERED` or `CANCELLED` as the source.
2. `canTransition` looks up an empty successor array.
3. Inclusion fails -> 409 `INVALID_STATUS_TRANSITION`.
4. No stock, order or history write occurs.
5. The order remains in its terminal state permanently through the API.

---

### Business Rule BR-19: Stock debit on `PENDING -> PAID`

**Overview**:
Inventory is decremented exactly once, on the transition from `PENDING` to `PAID`.

**Location**: `src/modules/orders/order.status.ts:24-31`; `src/modules/orders/order.service.ts:151-153`
**Confidence**: High

**Detailed description**:
`shouldDebitStock(from, to)` returns true only for the exact pair `PENDING, PAID`, encoded in the `STOCK_DEBIT_TRANSITION` constant. The service checks the predicate before writing the status and calls `debitStock(tx, order.items)` when it is true. Because the status update occurs after the debit in the same transaction, a debit failure prevents the status from becoming `PAID`.

The rule makes payment the inventory commitment point. Before payment, stock is informational; at payment, it becomes an obligation. This aligns with the creation rule BR-15 and explains why insufficient stock produces 422 at payment rather than at creation. It also means that a paid order that is later cancelled must replenish stock (BR-21), because payment already removed it.

The predicate is a pure function of the `(from, to)` pair, not of the order contents or product state. It does not check whether the order has items, whether the products are still active, or whether stock was already debited out-of-band. Idempotence depends entirely on the status guard: the only way to reach `PENDING -> PAID` twice is a concurrent execution that both read `PENDING` before either committed, which is the concurrency risk documented in Section 9.

**Rule workflow**:
1. `changeStatus` validates the transition table.
2. `shouldDebitStock(from, to)` evaluates true only for `PENDING -> PAID`.
3. `debitStock` validates and decrements stock.
4. If debit succeeds, the order status is updated to `PAID`.
5. Any failure rolls back both stock and status.

---

### Business Rule BR-20: All-or-nothing stock sufficiency

**Overview**:
A debit requires every aggregated line to have enough stock; if any line is short, no stock is changed and the transition fails with 422.

**Location**: `src/modules/orders/order.service.ts:208-230`
**Confidence**: High

**Detailed description**:
`debitStock` first loads all products referenced by the order items in one query, builds an `unavailable` array for any item whose product is missing or whose `stockQuantity` is below the requested quantity, and throws `InsufficientStockError(unavailable)` if the array is non-empty. The error includes `{ sku, requested, available }` per offending line, which becomes `error.details.unavailable` in the 422 response. Only after the whole list passes does the method iterate and decrement each product.

The check is all-or-nothing at the validation level: the method never performs a partial debit, because the decrement loop runs only after the full validation passes. At the transaction level it is also all-or-nothing: if a later decrement fails, the transaction rolls back earlier decrements. This gives the caller a clean semantic: either the entire order is paid and all stock is reserved, or nothing changes.

The check is not atomic against concurrent writers. It is a read (`findMany`) followed by unconditional `update` calls with `decrement`, without `SELECT ... FOR UPDATE`, a conditional `where: { stockQuantity: { gte: quantity } }`, or a version column. Two concurrent payments can both observe sufficient stock and both decrement, producing a negative balance. The test suite verifies the sequential shortfall path (`tests/orders.test.ts:109-132`) but contains no concurrency test. The missing-product branch is effectively unreachable through the API because the order-item foreign key is `ON DELETE RESTRICT`, but it is handled defensively by reporting the product id as the SKU with zero availability.

**Rule workflow**:
1. Load products for all order items inside the transaction.
2. For each item, compare requested quantity with `stockQuantity`.
3. Collect `{ sku, requested, available }` for every shortfall or missing product.
4. Non-empty list -> `InsufficientStockError` -> 422 -> rollback.
5. Empty list -> decrement each product by its quantity.
6. The caller then updates the order status to `PAID` in the same transaction.

---

### Business Rule BR-21: Stock replenishment on cancellation after payment

**Overview**:
Stock is incremented back only when a `PAID` or `PROCESSING` order is cancelled.

**Location**: `src/modules/orders/order.status.ts:33-37`; `src/modules/orders/order.service.ts:154-156,233-243`
**Confidence**: High

**Detailed description**:
`shouldReplenishStock(from, to)` returns true when the target is `CANCELLED` and the source is either `PAID` or `PROCESSING`. The service calls `replenishStock`, which unconditionally increments each product's `stockQuantity` by the item quantity. The symmetry with BR-19 is exact: stock is added back only for states in which it must already have been removed, and never for `PENDING -> CANCELLED`, because a pending order never debited stock.

The rule depends on the transition table for reachability. `PENDING -> CANCELLED` is allowed and does not replenish. `PAID -> CANCELLED` and `PROCESSING -> CANCELLED` are allowed and do replenish. `SHIPPED -> CANCELLED` is not allowed at all, so a shipped order cannot be cancelled through the API and its stock is never returned. `DELIVERED -> CANCELLED` is likewise impossible. This encodes a policy in which inventory reversal is possible only before shipment.

Replenishment is not idempotent by itself; it relies on the status guard and on the transaction. Under concurrent cancellation attempts, both transactions can read `PAID`, both pass the transition check, both increment stock and both write history, producing double replenishment. The service also does not verify that stock had actually been debited (for example, an order manually set to `PAID` by seed or SQL), so an out-of-band state can cause inventory inflation when cancelled.

**Rule workflow**:
1. Transition table allows the move to `CANCELLED`.
2. `shouldReplenishStock(from, to)` is true for `PAID` or `PROCESSING` sources.
3. `replenishStock` increments each product by the item quantity.
4. The status is updated to `CANCELLED`.
5. History is appended; all writes commit or roll back together.

---

### Business Rule BR-22: Status-change atomicity

**Overview**:
A status change, its stock side effects and its audit row are committed as one transaction.

**Location**: `src/modules/orders/order.service.ts:131-178`
**Confidence**: High

**Detailed description**:
`changeStatus` runs entirely inside `prisma.$transaction`. Within the callback it reads the order and items, validates the transition, optionally adjusts stock, updates the order status, inserts the history row and re-reads the aggregate. No write escapes the transaction. If any step throws, Prisma rolls back all prior writes, including any stock decrements or increments, and the controller forwards the error.

This is the strongest consistency guarantee in the component. It ensures, for example, that a stock shortfall cannot leave an order marked `PAID` without inventory, and that a history row cannot exist without the corresponding status change. It also ensures that a failed re-read after the update does not partially commit: the throw would roll back the update and the history insert.

Atomicity does not equal isolation. The transaction does not lock the order row when reading it, does not set a version, and does not use a conditional update. Two transactions can therefore both pass validation on the same `from` state and both commit sequentially, each writing a history row and each applying stock movement. The unit of work protects against partial failure, not against lost updates or double application.

**Rule workflow**:
1. Open interactive transaction.
2. Read order and items.
3. Validate same-status and transition table.
4. Apply stock side effects if predicates match.
5. Update order status.
6. Insert history row.
7. Re-read the aggregate.
8. Commit all writes together or roll back on any error.

---

### Business Rule BR-23: Append-only transition audit

**Overview**:
Every accepted transition writes one history row capturing source, target, actor and optional reason.

**Location**: `src/modules/orders/order.service.ts:159-167`; `prisma/schema.prisma:116-131`
**Confidence**: High

**Detailed description**:
After updating the order status, the service inserts into `order_status_history` with `orderId`, `fromStatus`, `toStatus`, `changedById` and `reason`. `changedAt` is a database default. The row is written for every valid transition, including the first one after creation, so the history forms a complete chain from `null -> PENDING` through each subsequent state. Reads order the chain ascending by `changedAt` (`order.repository.ts:56`), which is the closest thing to an ordered event log in the system.

The chain has no explicit sequence number. If two rows share the same millisecond timestamp for the same order, their relative order is not guaranteed by the `orderBy`. In normal operation transitions are serialised by the order status guard, so this is unlikely, but the concurrency window makes duplicate rows with near-identical timestamps possible. The table indexes `orderId` and `changedAt` separately, not as a composite, so the query can filter by order and sort by time with a filesort or index merge rather than a perfect covering order.

The service never updates or deletes history. The only deletion path is the cascade from the parent order (BR-27). This makes history append-only while the order exists, but not tamper-evident: there is no hash chain, no signature, no actor snapshot beyond a foreign key, and no retention policy. If a user's role or email changes later, the history still points to the user row and shows current data on a join; the service does not join it, so the API returns only `changedById`.

**Rule workflow**:
1. The transition passes all guards and stock handling succeeds.
2. `tx.order.update` writes the new status.
3. `tx.orderStatusHistory.create` writes `fromStatus`, `toStatus`, `changedById`, `reason`.
4. `changedAt` is assigned by the database.
5. The row is returned as part of the re-read aggregate, ordered ascending.
6. No service path updates or deletes the row.

---

### Business Rule BR-24: Optional transition reason

**Overview**:
A transition may carry a reason of up to 500 characters; absence is stored as `NULL`.

**Location**: `src/modules/orders/order.schemas.ts:20`; `src/modules/orders/order.service.ts:165`
**Confidence**: High

**Detailed description**:
`updateOrderStatusSchema` declares `reason: z.string().max(500).optional()`, and the service writes `input.reason ?? null`. The reason is attached only to the transition history row, not to the order. It is not required even for cancellation, so an operator can cancel a paid order without recording why, and the audit trail will show only that the cancellation happened.

The field is free text with no enumeration, minimum length or trimming. An empty string is stored as an empty string rather than `NULL`, because the nullish coalescing operator only replaces `undefined` and `null`. A whitespace-only string is likewise preserved. The database column is `VARCHAR(500)` (`prisma/schema.prisma:123`), consistent with the schema maximum, so a valid request cannot overflow it.

Reasons are returned as part of the history array. Because the service returns raw history rows, every reader of the order sees every reason. There is no separate privileged view for audit data. The field is also never validated against the transition type, so a `DELIVERED` transition can carry the reason "customer changed mind" without any semantic check.

**Rule workflow**:
1. Zod validates optional string, maximum 500.
2. The controller forwards the parsed body.
3. The service maps `undefined` to `null` and persists the value with the history row.
4. The value is returned in the aggregate history array.
5. No later method modifies it.

---

### Business Rule BR-25: Mutation response projection

**Overview**:
Both creation and status change return the order with items plus product summary, full history ascending and customer summary.

**Location**: `src/modules/orders/order.service.ts:115-119,169-176`; `src/modules/orders/order.repository.ts:47-60`
**Confidence**: High

**Detailed description**:
The nested create in `create` uses `include` for items (with product `id`, `sku`, `name`), history ordered by `changedAt` ascending, and customer (`id`, `name`, `email`). `changeStatus` performs an additional `findUnique` with the same include set after writing. `getById` delegates to `findByIdWithRelations`, which uses the identical projection. The three mutation and detail paths therefore return the same shape, while `list` returns plain order rows without relations.

The projection is a read model, not a domain mapping. It exposes all scalar columns of `Order` (including `createdById`, `customerId`, `orderNumber`, all monetary fields and timestamps) plus the three relations. Product details are limited to id, SKU and name: price and stock are deliberately not included in the nested product, even though the item carries the snapshotted price. Customer details are limited to id, name and email: phone, document and address are not exposed through the order aggregate.

The extra re-read in `changeStatus` costs a query but guarantees that the response reflects committed state, including database-generated timestamps and the newly inserted history row. It also means the method depends on a non-null assertion (`refreshed!`): if the re-read unexpectedly returned `null`, the method would return `undefined` and the controller would serialise an empty body with status 200 rather than raising an error. Under normal transactional visibility this cannot happen, because the order was just updated in the same transaction.

**Rule workflow**:
1. `create` returns the nested-write result directly.
2. `changeStatus` re-reads the order with the standard include set.
3. `getById` uses the repository equivalent.
4. The controller serialises the result with 201 or 200.
5. `list` intentionally omits relations.

---

### Business Rule BR-26: Delete eligibility

**Overview**:
Only orders in `PENDING` or `CANCELLED` may be deleted.

**Location**: `src/modules/orders/order.service.ts:184-190`
**Confidence**: High

**Detailed description**:
`delete` loads the order through the repository, then checks whether the status is `PENDING` or `CANCELLED`. Any other status produces `ConflictError('Order can only be deleted while in PENDING or CANCELLED status', 'INVALID_ORDER_STATE_FOR_DELETE', { status: order.status })`, which becomes 409 with the current status in `details`. The integration test at `tests/orders.test.ts:197-218` verifies the refusal for a `PAID` order.

The rule creates a deliberate asymmetry with the lifecycle. A pending order can be removed before it has any inventory effect; a cancelled order can be removed after inventory has been restored. Paid, processing, shipped and delivered orders cannot be removed, preserving the commercial record. Deletion is therefore a cleanup operation for abandoned or reversed orders, not a general-purpose administrative delete.

The check is performed on data loaded by a separate repository call, outside a transaction, and the delete is a second call. There is no conditional delete (`deleteMany` with a status filter), no row lock and no version check. A concurrent `PENDING -> PAID` transition can therefore commit between the check and the delete; the delete then removes a paid order and cascades its history, while the stock debit remains applied. This is a consistency gap in the rule's enforcement rather than in its stated intent.

**Rule workflow**:
1. Load the order by id.
2. Missing -> 404.
3. Status not `PENDING` and not `CANCELLED` -> 409 `INVALID_ORDER_STATE_FOR_DELETE`.
4. Otherwise delete by id.
5. Database cascades items and history; response is 204.

---

### Business Rule BR-27: Cascading deletion of items and history

**Overview**:
Deleting an order physically removes its items and its entire status history.

**Location**: `prisma/migrations/20260519182739_init/migration.sql:116,122`; `prisma/schema.prisma:108,125`
**Confidence**: High (behaviour); Medium (intent)

**Detailed description**:
Both `order_items.orderId` and `order_status_history.orderId` are foreign keys to `orders.id` with `ON DELETE CASCADE`. When `OrderRepository.deleteById` issues a single `prisma.order.delete`, MySQL removes the child rows automatically. The service does not archive, soft-delete or copy history before deletion, and the schema has no `deletedAt` column.

This means the audit trail is co-terminous with the order's lifetime. A cancelled order, which by definition has a multi-step history including any payment and cancellation reasons, can be deleted because `CANCELLED` is an allowed deletion state. After deletion, no record remains that the order ever existed, except for indirect effects such as stock changes and any external logs. From a compliance and forensics perspective, this is a material property of the component.

The cascade is also why the delete operation does not need multiple statements and cannot fail halfway at the application level: the database performs it atomically. However, because the deletion itself is not wrapped in an explicit transaction, the atomicity is that of the single statement, which is sufficient for the cascade but not for the check-then-delete sequence (BR-28).

**Rule workflow**:
1. Eligibility passes.
2. A single `DELETE FROM orders WHERE id = ?` is issued.
3. InnoDB deletes matching `order_items` and `order_status_history` rows via cascade.
4. The order row disappears.
5. No audit record of the deletion is retained by the application.

---

### Business Rule BR-28: Non-transactional delete guard

**Overview**:
The delete eligibility check and the delete itself are two separate, non-transactional operations.

**Location**: `src/modules/orders/order.service.ts:182-191`
**Confidence**: High

**Detailed description**:
`delete` calls `this.orders.findById(id)` and then, after the status check, `this.orders.deleteById(id)`. Each repository method issues its own query through the shared `PrismaClient`; there is no `$transaction` wrapper, no `SELECT ... FOR UPDATE`, and no conditional delete that includes the status in the `WHERE` clause. The time between the two calls is a window in which another transaction can change the order.

The practical failure mode is a lost lifecycle record. If a `PENDING` order is paid concurrently, the debit and status update commit, and then the delete removes the order and its history. Stock stays decremented because the cascade does not reverse stock and the delete path has no replenishment logic. The same window exists between the status check and the delete for a concurrent cancellation, although that case is less harmful because stock has already been replenished by the cancellation.

The rule is therefore correct in the single-threaded sense and racy under concurrency. Nothing in the service signals the gap: no optimistic condition, no retry, no lock. The only protection is the low probability of the interleaving and the fact that deletion of active orders is an administrative action. The test suite does not exercise concurrency for this path.

**Rule workflow**:
1. Read order (separate query, no lock).
2. Evaluate status in application memory.
3. Return 409 or continue.
4. Issue delete (separate statement).
5. A concurrent transition between steps 1 and 4 is not detected.

---

### Business Rule BR-29: Pagination contract

**Overview**:
List results are paginated with page 1 and page size 20 by default, a maximum page size of 100, and a computed total page count.

**Location**: `src/modules/orders/order.schemas.ts:24-25`; `src/modules/orders/order.service.ts:38,41`; `src/shared/http/response.ts:13-24`
**Confidence**: High

**Detailed description**:
The query schema coerces strings to numbers, requires an integer page of at least 1, and requires a page size between 1 and 100, defaulting to 20. The service computes `skip = (page - 1) * pageSize` and passes `take = pageSize` to the repository. The response helper computes `totalPages = Math.ceil(total / pageSize)` and returns `{ data, pagination: { page, pageSize, total, totalPages } }`. A page size of zero is guarded in the helper but cannot occur through the schema.

Pagination is offset-based, not cursor-based. Deep pages become progressively more expensive because MySQL must scan and discard `skip` rows, and the offset is not stable under concurrent inserts: a new order created between two page requests shifts subsequent rows, so a client paging through results can see duplicates or miss records. The repository's count and items query are executed in a batch transaction (`order.repository.ts:35-43`), which gives them a consistent snapshot relative to each other within that transaction.

The service does not enforce a maximum page number, so a request for page 1,000,000 simply returns an empty `data` array with the correct `total` and a very large `totalPages`. There is no link header, no cursor token and no `hasNext` flag; clients must derive navigation from `page`, `pageSize`, `total` and `totalPages`. The default page size of 20 is a component-level policy, not an environment setting.

**Rule workflow**:
1. Zod parses and defaults `page` and `pageSize`.
2. Service computes `skip`.
3. Repository runs `findMany` with `skip`/`take` and `count` in one batch transaction.
4. Service wraps results with `paginated`.
5. Response contains `data` and `pagination`.
6. Out-of-range pages yield an empty `data` array.

---

### Business Rule BR-30: List filters and ordering

**Overview**:
Orders can be filtered by status, customer and creation-date range, and are always returned newest first without relations.

**Location**: `src/modules/orders/order.service.ts:33-40`; `src/modules/orders/order.repository.ts:21-45`
**Confidence**: High

**Detailed description**:
`buildWhere` adds `status` when present, `customerId` when present, and a `createdAt` range when either bound is present: `gte` for `from` and `lte` for `to`. The range is inclusive on both ends. All filters are optional and combine with AND semantics. The query orders by `createdAt` descending, so the newest order is first; ties are not broken by a secondary key, which makes ordering non-deterministic for orders created in the same millisecond.

The filter set is deliberately shallow. There is no search by order number, customer name, product, total range or actor, and no full-text capability. `from` and `to` are coerced to `Date` by Zod, so an invalid date string is a validation error. A reversed range (`from` after `to`) is accepted and simply returns no rows, because the service does not compare the two bounds.

List results are plain `Order` rows. Unlike `getById` and the mutation responses, they contain no items, no history and no customer summary. A client that needs line details must issue a follow-up `GET /orders/:id` per order, which is an N+1 pattern at the client level. There is no projection parameter to request relations in the list.

**Rule workflow**:
1. Validated query supplies optional filters.
2. Service forwards them with pagination bounds.
3. Repository builds the `where` object.
4. `findMany` applies filters, ordering, skip and take; `count` uses the same `where`.
5. Service returns plain rows plus pagination metadata.

---

### Business Rule BR-31: Detail retrieval 404

**Overview**:
Reading an order that does not exist returns 404 `NOT_FOUND`.

**Location**: `src/modules/orders/order.service.ts:44-48`
**Confidence**: High

**Detailed description**:
`getById` calls `orders.findByIdWithRelations(id)` and throws `NotFoundError('Order')` when the result is `null`. The shared error class produces HTTP 404, code `NOT_FOUND` and message `Order not found`. The id is validated as a UUID by `orderIdParamSchema`, so a syntactically invalid id is rejected as 400 before reaching the service, and a syntactically valid but unknown id reaches the 404 path.

The same `NotFoundError('Order')` is used by `changeStatus` and `delete`, so the three single-order operations have a consistent missing-resource contract. None of them distinguishes "never existed" from "was deleted"; deletion is physical, so a deleted order is indistinguishable from an unknown id. This is a deliberate consequence of hard deletion rather than soft deletion.

The detail projection is the richest read in the module: order scalars, items with product summary, history ascending and customer summary. It does not include the creating user or the users referenced by history; only their ids are present through `createdById` and `changedById`. A client that needs actor names must resolve them through the users API, which is itself role-restricted for reads by id.

**Rule workflow**:
1. Validate the id as a UUID.
2. Query with relations.
3. `null` -> `NotFoundError('Order')` -> 404.
4. Otherwise return the aggregate.
5. Controller serialises with 200.

---

### Business Rule BR-32: No service-layer ownership or role scoping

**Overview**:
`OrderService` applies no tenant, customer-ownership or role restriction to any operation.

**Location**: `src/modules/orders/order.service.ts:32-48,126-130,181-192`; `src/modules/orders/order.routes.ts:14`
**Confidence**: High (behaviour); Medium (intent)

**Detailed description**:
Every public method accepts only an id or a query plus, for mutations, a user id used for attribution. None of them compares the authenticated user with `order.createdById`, a customer, a tenant or a role. The router applies only `authenticate` (`order.routes.ts:14`), not `requireRole`, so any authenticated `OPERATOR` or `ADMIN` can list every order, read any order, create an order for any customer, transition any order and delete any eligible order.

The list query filters only by values supplied by the caller. A caller can request `customerId` for any customer, or omit it to see all orders. There is no implicit scoping injected from the token. The only place in the application where role is checked is `GET /users/:id`, which requires `ADMIN`; that check does not extend to orders. This is the broadest authorization gap touching the component.

The behaviour is unambiguous in code; whether it is intended is not documented. The README describes a B2B order management system, and the architecture report flags the absence of tenant isolation as a high risk. Within this component's boundary, the rule can be stated simply: authentication is a precondition, authorization is not implemented. Any downstream consumer of `OrderService` must not assume that the service enforces access control.

**Rule workflow**:
1. Router authenticates the request.
2. No role guard runs for order routes.
3. Controller passes the body/params and the user id (for mutations only).
4. Service executes without comparing the actor to the resource.
5. Any authenticated caller can operate on any order.

---

### Business Rule BR-33: Active status is creation-time only

**Overview**:
The active-product check is applied when an order is created and never re-applied during the lifecycle.

**Location**: `src/modules/orders/order.service.ts:66-73,126-179`
**Confidence**: High (behaviour); Medium (intent)

**Detailed description**:
Creation filters products for `active === false` and rejects the order. `changeStatus` loads the order and its items but does not join or re-read products, so a product deactivated after order creation does not block payment, processing, shipping or delivery. `debitStock` reads products only to compare stock, not to check the active flag, and it does not reject inactive products.

This creates a distinction between new demand and committed demand: deactivation stops new orders but does not strand existing ones. That is usually the desired commercial behaviour, because cancelling in-flight orders automatically would be disruptive. The consequence is that the active flag cannot be used as an emergency stop for a product that is already in pending orders; those orders will still debit stock when paid.

The rule is implicit. There is no comment or documentation stating that active status is creation-scoped, and no test covers deactivation between creation and payment. The only evidence is the absence of an active check in the transition path. A reader could reasonably assume the opposite, which is why the rule is recorded here with medium intent confidence.

**Rule workflow**:
1. Creation validates active state (BR-05).
2. The order is persisted with items referencing the products.
3. Later transitions do not reload or evaluate `product.active`.
4. Stock checks use quantity only.
5. Deactivation affects only future creations.

---

## 4. Component Structure

### 4.1 File and folder layout (Orders module)

```
src/modules/orders/
├── order.routes.ts        # 27 lines - Express router factory, authenticate + Zod validation per route
├── order.controller.ts    # 56 lines - HTTP adapter, extracts req.user, maps service results to status codes
├── order.service.ts       # 255 lines - THIS COMPONENT: lifecycle orchestration and domain rules
├── order.repository.ts    # 69 lines - list/find/delete persistence for three of the five service methods
├── order.schemas.ts       # 34 lines - Zod input contracts and inferred TypeScript types
└── order.status.ts        # 37 lines - transition table and stock side-effect predicates
```

Supporting files outside the module:

```
src/app.ts                              # composition root: new OrderService(new OrderRepository(prisma), prisma) at 42-44
src/routes/index.ts                     # mounts buildOrderRouter under /orders at /api/v1
src/shared/errors/http-errors.ts        # NotFound, Conflict, Validation, UnprocessableEntity, InsufficientStock, InvalidStatusTransition
src/shared/errors/app-error.ts          # statusCode + errorCode + details carrier
src/shared/http/response.ts             # paginated() and buildPagination()
src/config/database.ts                  # shared PrismaClient instance
prisma/schema.prisma                    # Order, OrderItem, OrderStatusHistory, OrderNumberSequence, enums
prisma/migrations/20260519182739_init/migration.sql  # physical schema, FKs and cascade rules
tests/orders.test.ts                    # 8 HTTP integration tests covering the service indirectly
tests/setup.ts                          # truncates all order-related tables before each test
tests/helpers/factories.ts              # test users, customers, products and authenticated tokens
prisma/seed.ts                          # duplicates the order-number algorithm; writes orders/history directly
```

### 4.2 Public method inventory

| Method | Lines | Signature outline | Transaction | Collaborators | Purpose |
|--------|-------|-------------------|-------------|---------------|---------|
| `list` | 32-42 | `(query: ListOrdersQuery) => PaginatedResponse<Order>` | Delegated (batch transaction inside repository) | `OrderRepository.list`, `paginated` | Filtered, paginated order list without relations |
| `getById` | 44-48 | `(id: string) => OrderWithRelations` | None (single query) | `OrderRepository.findByIdWithRelations` | Detail read with items, history, customer |
| `create` | 50-124 | `(input: CreateOrderInput, userId: string) => OrderWithRelations` | Interactive transaction (58) | `prisma.$transaction`, `tx.customer`, `tx.product`, `tx.order`, `tx.orderNumberSequence` | Price, validate, number, persist aggregate and initial history |
| `changeStatus` | 126-179 | `(id, input: UpdateOrderStatusInput, userId) => OrderWithRelations` | Interactive transaction (131) | `tx.order`, `tx.orderStatusHistory`, `tx.product`, state-machine predicates | Enforce lifecycle, move stock, audit transition |
| `delete` | 181-192 | `(id: string) => void` | None (two separate queries) | `OrderRepository.findById`, `OrderRepository.deleteById` | Remove pending/cancelled orders and cascade children |

### 4.3 Private helper inventory

| Helper | Lines | Transaction client | Purpose |
|--------|-------|--------------------|---------|
| `aggregateItems` | 194-202 | No | Merge duplicate product ids and sum quantities |
| `debitStock` | 204-231 | Yes (`TxClient`) | Validate aggregate stock availability and decrement per product |
| `replenishStock` | 233-243 | Yes (`TxClient`) | Increment per product on cancellation after payment |
| `reserveOrderNumber` | 245-254 | Yes (`TxClient`) | Upsert global sequence row and format `ORD-NNNNNN` |

### 4.4 Transaction boundaries

| Operation | Boundary | Participating tables | Isolation / locking | Rollback trigger |
|-----------|----------|----------------------|---------------------|------------------|
| `create` | `prisma.$transaction` (58-123) | `customers` (read), `products` (read), `order_number_sequence` (upsert), `orders` (insert), `order_items` (nested insert), `order_status_history` (nested insert) | Prisma default; InnoDB repeatable read; exclusive lock on sequence row until commit | Any thrown error including validation, not-found, inactive product, discount overflow, sequence unique conflict |
| `changeStatus` | `prisma.$transaction` (131-178) | `orders` (read/update), `order_items` (read), `products` (read/update), `order_status_history` (insert) | Prisma default; no order-row lock on read; row locks taken by the later updates | Same-status conflict, invalid transition, insufficient stock, any Prisma failure |
| `delete` | None | `orders` (read/delete), `order_items` (cascade), `order_status_history` (cascade) | Single-statement atomicity for the delete; check is outside | Not-found or ineligible status before delete; DB error during delete |
| `list` | Repository batch transaction (order.repository.ts:35-43) | `orders` (findMany + count) | Consistent snapshot for the pair of statements | Prisma error |
| `getById` | None | `orders` + relations | Single query | Not-found handled by service |

### 4.5 Internal dependency structure of the class

```
OrderController
      |
      v
OrderService
  |-- OrderRepository --------------------> PrismaClient (orders table)
  |-- PrismaClient (injected) ------------> customers, products, orders, order_items,
  |                                          order_status_history, order_number_sequence
  |-- order.status.ts --------------------> @prisma/client OrderStatus enum
  |-- shared/errors ----------------------> AppError subclasses (HTTP status + code)
  |-- shared/http/response.ts ------------> PaginatedResponse<T>
  |-- order.schemas.ts (type only) -------> CreateOrderInput, UpdateOrderStatusInput, ListOrdersQuery
  \-- order.repository.ts (type only) ----> OrderWithRelations
```

There is one circular-looking but acyclic relationship at the type level: `OrderService` imports `OrderWithRelations` from `order.repository.ts`, while the repository is injected into the service. The import is type-only (`import type`), so it is erased at runtime and does not create a runtime cycle.

### 4.6 Code size and shape

- 255 lines, one exported class, no interfaces or abstract ports.
- Five public methods, four private helpers, two constructor dependencies.
- Longest method: `create` (75 lines including the nested write), followed by `changeStatus` (54 lines).
- `create` contains seven distinct rule groups: input guard, aggregation, customer lookup, product lookup, active check, pricing/discount, numbering/persistence.
- No comments or JSDoc in the file. All intent must be inferred from names and structure.

---

## 5. Dependency Analysis

### 5.1 Internal dependencies

| Dependency | Import site | Kind | Direction | Notes |
|------------|-------------|------|-----------|-------|
| `OrderRepository` | `order.service.ts:7` | Type-only import (`OrderWithRelations`) plus constructor injection (`28`) | Service -> Repository | Used by `list`, `getById`, `delete`; bypassed by `create` and `changeStatus` |
| `OrderWithRelations` | `order.service.ts:7` | Type | Repository -> Service contract | Composite read model: order + items + product summary + history + customer summary |
| `CreateOrderInput`, `ListOrdersQuery`, `UpdateOrderStatusInput` | `order.service.ts:8` | Type-only | Schemas -> Service | The service is statically bound to the Zod-inferred shapes; it does not parse or re-validate |
| `PrismaClient`, `Prisma.TransactionClient` | `order.service.ts:1-6,24` | Type + runtime (enum values) | Service -> ORM | Injected instance; `Prisma.TransactionClient` aliased as `TxClient` |
| `OrderStatus` | `order.service.ts:3` | Runtime value | Service -> ORM | Used for `PENDING` literals at `99`, `109`, `184` |
| `canTransition` | `order.service.ts:17-21` | Runtime function | Service -> State machine | Called at `147` |
| `shouldDebitStock` | `order.service.ts:17-21` | Runtime function | Service -> State machine | Called at `151` |
| `shouldReplenishStock` | `order.service.ts:17-21` | Runtime function | Service -> State machine | Called at `154` |
| `ValidationError`, `NotFoundError`, `ConflictError`, `UnprocessableEntityError`, `InvalidStatusTransitionError`, `InsufficientStockError` | `order.service.ts:9-16` | Runtime classes | Service -> Error taxonomy | Thrown at `52, 60, 64, 68, 88, 136, 141, 148, 185, 223` |
| `paginated`, `PaginatedResponse` | `order.service.ts:22` | Runtime function + type | Service -> HTTP helper | Called at `41` |
| `OrderController` | `order.controller.ts:2,7` | Runtime class | Controller -> Service | Only runtime afferent consumer |
| `buildControllers` | `src/app.ts:42-44` | Runtime factory | Composition -> Service | Instantiates service and injects collaborators |

### 5.2 Dependency chains

```
HTTP request
  -> order.routes.ts (buildOrderRouter)
    -> authenticate middleware (JWT verification)
    -> validate middleware (Zod schemas)
      -> OrderController (method handler)
        -> OrderService (public method)
          -> order.status.ts (pure predicates)
          -> OrderRepository (list / detail / delete)
            -> PrismaClient -> MySQL (orders, order_items, order_status_history)
          -> PrismaClient.$transaction
            -> MySQL (customers, products, orders, order_items,
                      order_status_history, order_number_sequence)
          -> shared/errors (typed exceptions)
            -> error.middleware.ts (HTTP envelope)
          -> shared/http/response.ts (pagination envelope)
```

Reverse dependency (afferent) chains:

```
src/app.ts:42-44  -> OrderService
OrderController   -> OrderService
```

No other production file imports `OrderService`. `prisma/seed.ts` reimplements the order-number algorithm locally (`seed.ts:165-176`) rather than importing the service, so the numbering rule is duplicated outside the component boundary.

### 5.3 External dependencies

| Dependency | Version | Type | Used by the component for | Coupling strength |
|------------|---------|------|---------------------------|-------------------|
| `@prisma/client` | 5.22.0 (exact) | ORM runtime | Transaction manager, typed queries, nested writes, enum values, error classes | High: the service is written directly against Prisma models and `Prisma.TransactionClient` |
| MySQL | 8.0 (`docker-compose.yml:3`) | Relational database | Persistence, unique index on `orderNumber`, foreign keys, cascade deletes, row locks on the sequence | High: correctness of numbering and atomicity depends on InnoDB semantics |
| `zod` | 3.23.8 (transitive) | Validation | Indirect: the service consumes shapes produced by schemas; does not import Zod | Low at runtime, high at contract level |
| `express` | 4.21.1 (transitive) | HTTP framework | Indirect via controller and router; service does not import Express | None at the service boundary |
| `vitest` + `supertest` | 2.1.4 / 7.0.0 (dev) | Test harness | Integration tests exercise the service through the HTTP app | Indirect |

There is no HTTP client, message broker client, cache client, scheduler or event emitter in the component. The service performs no network I/O other than the database protocol driven by Prisma.

### 5.4 Dependency quality observations

- The service depends on a concrete ORM client rather than a persistence port. `Prisma.TransactionClient` appears in private method signatures (`204`, `233`, `245`), so the transaction abstraction leaks into the domain layer.
- Two dependencies are used only for types (`order.repository.ts`, `order.schemas.ts`). They are erased at compile time and do not contribute runtime coupling, but they couple the service to the exact shape of the HTTP input contract.
- The state machine is a pure-function dependency with no I/O, which is the cleanest edge in the graph.
- The error taxonomy is a runtime dependency on six concrete classes. Adding a new domain failure requires the service to import a new class, and the mapping to HTTP lives outside the component.
- The numbering rule is duplicated in `prisma/seed.ts:165-176`. A change to the format or to the sequence semantics must be applied in two places, and the two implementations differ in transaction scope (the seed wraps only the upsert in a transaction, the service uses the creation transaction).

---

## 6. Afferent and Efferent Coupling

### 6.1 Class-level coupling

| Component | Afferent (Ca) | Efferent (Ce) | Instability (Ce / (Ca + Ce)) | Critical |
|-----------|---------------|---------------|------------------------------|----------|
| `OrderService` | 2 | 6 | 0.75 | High |
| `OrderController` | 2 (router, composition root) | 4 (`OrderService`, `ListOrdersQuery` type, Express types, `UnauthorizedError`) | 0.67 | Medium |
| `OrderRepository` | 2 (`OrderService`, composition root) | 2 (`PrismaClient`, `@prisma/client` types) | 0.50 | Medium |
| `order.status.ts` | 1 (`OrderService`) | 1 (`@prisma/client` `OrderStatus`) | 0.50 | High (domain rules, low mechanical coupling) |
| Error taxonomy (`shared/errors`) | 6 (auth, users, customers, products, orders, middlewares) | 1 (`AppError`) | 0.14 | High (shared, stable) |
| `shared/http/response.ts` | 3 (customer, product, order services) | 0 | 0.00 | Low |
| Prisma schema (`Order*` models) | 2 write paths (`create`, `changeStatus`) + 1 read path (repository) | 0 | 0.00 | High (data contract) |

The afferent count for `OrderService` is small but the efferent count is the highest among the five services because the service reaches across four persistence aggregates and three shared modules. Instability of 0.75 means the class is predominantly a consumer: changes in Prisma models, schema shapes or the state machine propagate into it, while only the controller and composition root depend on it.

### 6.2 Method-level coupling inside the class

| Member | Fan-in (internal callers) | Fan-out (internal + external calls) | Notes |
|--------|---------------------------|-------------------------------------|-------|
| `list` | 1 (controller) | 2 (`orders.list`, `paginated`) | Thin adapter over repository |
| `getById` | 1 (controller) | 1 (`orders.findByIdWithRelations`) | No branching beyond null check |
| `create` | 1 (controller) | 10 (`aggregateItems`, `prisma.$transaction`, `tx.customer.findUnique`, `tx.product.findMany`, `reserveOrderNumber`, `tx.order.create`, `find` per item, `reduce`) | Highest internal complexity; coordinates four tables and two helpers |
| `changeStatus` | 1 (controller) | 9 (`prisma.$transaction`, `tx.order.findUnique` twice, `canTransition`, `shouldDebitStock`, `shouldReplenishStock`, `debitStock`, `replenishStock`, `tx.order.update`, `tx.orderStatusHistory.create`) | Second-highest complexity; all lifecycle side effects |
| `delete` | 1 (controller) | 2 (`orders.findById`, `orders.deleteById`) | Simple but non-transactional |
| `aggregateItems` | 1 (`create`) | 1 (`Map` operations) | Pure, testable in isolation |
| `debitStock` | 1 (`changeStatus`) | 3 (`tx.product.findMany`, `tx.product.update`, error construction) | Read-check-write pattern |
| `replenishStock` | 1 (`changeStatus`) | 1 (`tx.product.update` per item) | N updates, no batching |
| `reserveOrderNumber` | 1 (`create`) | 1 (`tx.orderNumberSequence.upsert`) | Single hot row |

### 6.3 Data coupling (tables touched)

| Table | Read | Write | Methods |
|-------|------|-------|---------|
| `customers` | Yes | No | `create` |
| `products` | Yes | Yes (`stockQuantity`) | `create` (read), `changeStatus` via `debitStock`/`replenishStock` (read + write) |
| `orders` | Yes | Yes (insert, status update, delete) | `create`, `changeStatus`, `delete`, `list`, `getById` |
| `order_items` | Yes | Yes (insert, cascade delete) | `create`, `changeStatus` (read), `delete` (cascade) |
| `order_status_history` | Yes | Yes (insert, cascade delete) | `create`, `changeStatus`, `delete` (cascade), reads via repository |
| `order_number_sequence` | Yes | Yes (upsert/increment) | `create` |
| `users` | No | No (FK references only) | `create`, `changeStatus` set `createdById`/`changedById` |

The service touches six of the seven tables in the schema, either directly or through the repository. Only `users` is not read or written by the service, although it is referenced by foreign keys.

### 6.4 Cohesion assessment

The class is functionally cohesive around the order aggregate but not around a single concern. It mixes pricing, validation, numbering, persistence, inventory and auditing. The methods that share state (the two transactional commands) are the most complex, while the read methods are thin. There is no extracted pricing service, numbering service, inventory service or audit service; their logic lives in private helpers or inline blocks. This is a classic transaction-script shape: high functional cohesion at the use-case level, low separation of concerns at the implementation level.

---

## 7. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 via Prisma Client 5.22 | External datastore | Persist and query orders, items, history, sequence, and read/update customers and products | TCP 3306, MySQL wire protocol through Prisma | Relational rows mapped to Prisma models; integer cents; enums | Interactive transaction rollback on any thrown error; `P2002` -> 409 and `P2025` -> 404 by the centralized middleware; other Prisma errors -> 500 |
| `OrderRepository` | Internal component | List, detail read and physical delete of orders | In-process method calls | TypeScript objects (`Order`, `OrderWithRelations`, `PaginatedResponse`) | Propagates Prisma errors; no retry or wrapping |
| `order.status.ts` state machine | Internal pure module | Decide transition legality and stock side effects | In-process function calls | Enums and booleans | No I/O; cannot fail |
| `shared/errors` taxonomy | Internal shared module | Express domain failures as typed exceptions with HTTP status and machine-readable code | In-process exceptions | `AppError` subclasses with `statusCode`, `errorCode`, `details` | Translated by `errorMiddleware` into `{ error: { code, message, details? } }` |
| `shared/http/response.ts` | Internal shared module | Wrap list results with pagination metadata | In-process function calls | `{ data, pagination: { page, pageSize, total, totalPages } }` | No failure path |
| Authentication middleware (indirect) | Internal middleware | Supply the actor id used for attribution | In-process, after JWT verification | `req.user { id, email, role }` | Missing/invalid token -> 401 before the service is reached |
| Validation middleware (indirect) | Internal middleware | Supply parsed and defaulted input | In-process | Zod-parsed body/query/params | Zod failure -> 400 `VALIDATION_ERROR` before the service is reached |
| Outbound webhook delivery | Not implemented | Planned notification of status changes to B2B customers | None | None | None |

Notes on integration boundaries:

- The service has no inbound or outbound HTTP integration of its own. All external interaction is database I/O.
- `changeStatus` is the natural integration seam for the planned webhook feature described in `TRANSCRICAO.md` and the design documents, but no event emission, outbox table or queue exists in the current code. Any future integration would have to be added inside or after the transaction at `158-167`.
- The service does not log. Observability of failures depends on the centralized error middleware, which logs only unhandled (non-`AppError`, non-Zod, non-`P2002`/`P2025`) errors (`src/middlewares/error.middleware.ts:56-60`). Domain rejections such as insufficient stock are therefore not logged by the application.
- There is no cache. Every list, detail and lifecycle operation reads MySQL directly.
- There is no distributed transaction, saga or compensation logic. Consistency is limited to the local database transaction.

---

## 8. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Application Service / Transaction Script | `OrderService` public methods orchestrate use cases and call persistence directly | `order.service.ts:32-192` | Centralise order lifecycle rules in one place |
| Unit of Work | `prisma.$transaction(async (tx) => ...)` with all reads and writes on `tx` | `order.service.ts:58-123,131-178` | Atomic multi-table writes for create and transition |
| State | Static transition table plus `canTransition` predicate | `order.status.ts:3-14`; service call at `147` | Encode the lifecycle graph and reject illegal moves |
| Predicate-based side-effect selection | `shouldDebitStock` and `shouldReplenishStock` derive stock behaviour from the transition pair | `order.status.ts:24-37`; service calls at `151,154` | Keep inventory policy declarative and testable |
| Repository (partial) | `OrderRepository` injected into the service for list/detail/delete | `order.service.ts:27-30`; `order.repository.ts:18-68` | Abstract some persistence, but not applied to commands |
| Constructor Dependency Injection | Repository and Prisma client passed into the constructor; wired in the composition root | `order.service.ts:27-30`; `src/app.ts:42-44` | Explicit, testable wiring without a container |
| Domain Error Taxonomy / Exception Translation | Service throws `AppError` subclasses; middleware maps them to HTTP | `order.service.ts:9-16`; `src/shared/errors/`; `src/middlewares/error.middleware.ts:15-24` | Separate domain failures from transport concerns |
| Guard Clauses / Fail-Fast Validation | Early throws at each precondition | `order.service.ts:51,60,64,67,88,136,141,148,183` | Keep the happy path linear and avoid nested conditionals |
| Price Snapshot | `unitPriceCents` copied from the product at creation and stored on the item | `order.service.ts:75-84`; `prisma/schema.prisma:99-114` | Preserve historical pricing independent of catalogue changes |
| Audit Log (application-level) | History rows inserted on creation and every transition | `order.service.ts:106-113,159-167`; `prisma/schema.prisma:116-131` | Provide a status-change trail with actor and reason |
| Read-Model Projection | Fixed `include` sets on mutations and detail reads | `order.service.ts:115-119,169-176`; `order.repository.ts:47-60` | Return a consistent aggregate shape to clients |
| Batch Transaction for read consistency | `findMany` + `count` in one `$transaction([...])` | `order.repository.ts:35-43` | Align page rows with total count |
| Centralised Composition | Manual `new` graph in `buildControllers` | `src/app.ts:26-53` | Single place to see all wiring |

Patterns that are absent and shape the architecture:

- No domain events, outbox, saga or messaging. Lifecycle changes have no external notification path.
- No optimistic concurrency (version column) or pessimistic locking (`SELECT ... FOR UPDATE`).
- No idempotency key or request de-duplication on create or transition.
- No specification/query object for order filtering; the repository builds `where` inline.
- No separate pricing, inventory or numbering service. Those concerns are inlined in `OrderService`.
- No soft delete or archival. Deletion is physical.
- No interfaces or ports for the repository, so the service cannot be substituted without changing the concrete type.
- No caching layer, no read replica routing, no CQRS split between commands and queries.

---

## 9. Technical Debt & Risks

| Risk Level | Component Area | Issue | Evidence | Impact |
|------------|----------------|-------|----------|--------|
| Critical | `changeStatus` concurrency | The order row is read without a lock, version check or conditional update; two concurrent transitions can both observe the same source status | `order.service.ts:132-167` | Duplicate status transitions, double stock debit or replenishment, duplicate audit rows, lost lifecycle state |
| High | `debitStock` concurrency | Stock is validated by read-then-unconditional-decrement; no `gte` guard, no row lock, no conditional update | `order.service.ts:208-230` | Overselling and negative `stockQuantity` under concurrent payments |
| High | Order numbering | Every creation updates the single `order_number_sequence.id = 1` row inside the transaction | `order.service.ts:245-251`; `prisma/schema.prisma:133-138` | Global write serialisation, lock contention, throughput ceiling, transaction timeouts under load |
| High | Module boundary | `OrderService` reads and writes `customers`, `products` and `order_number_sequence` directly, bypassing repositories | `order.service.ts:58-123,204-254` | Schema changes leak into the domain service; duplicated persistence logic (for example, the numbering copy in `prisma/seed.ts:165-176`) |
| High | Test safety | `beforeEach` truncates every table using the configured `DATABASE_URL` with no environment guard | `tests/setup.ts:8-16` | Running the suite against a non-test database destroys data |
| Medium | Creation policy | No stock check or reservation at creation | `order.service.ts:62-73`; `tests/orders.test.ts:109-132` | Accepted orders can be unfulfillable; oversell exposure deferred to payment |
| Medium | Delete guard | Eligibility check and delete are separate, non-transactional operations | `order.service.ts:182-191` | A concurrent transition can be followed by deletion of a paid order, leaving stock debited with no order record |
| Medium | Audit retention | Deleting a `CANCELLED` order cascades its full history | `prisma/migrations/20260519182739_init/migration.sql:122` | Audit trail is destroyed by a supported API operation; no forensic record of the order |
| Medium | Idempotency | No idempotency key or request de-duplication on `create` or `changeStatus` | `order.service.ts:50-124,126-179` | Client retries after timeouts create duplicate orders; retried transitions return 409 after success |
| Medium | Error mapping | Prisma errors other than `P2002`/`P2025` are not mapped to 4xx | `src/middlewares/error.middleware.ts:37-64` | Foreign-key violations, transaction timeouts and overflow errors surface as 500 without domain context |
| Medium | Sequence bootstrap | Concurrent first-time upsert of the missing sequence row can produce a duplicate-key error | `order.service.ts:246-251` | One of the first concurrent creations fails with 409 instead of receiving a number |
| Low | Response assertion | `refreshed!` assumes the post-update re-read always returns a row | `order.service.ts:169-177` | If the assumption is ever violated, the method returns `undefined` and the controller emits 200 with an empty body |
| Low | Number format overflow | `padStart(6, '0')` does not truncate, so values above 999,999 produce more than six digits | `order.service.ts:253` | Format contract and the test regex `/^ORD-\d{6}$/` break at one million orders |
| Low | Free-text emptiness | Notes and reasons accept empty or whitespace-only strings | `order.service.ts:103,165` | Low-quality audit data; `''` and `NULL` are semantically indistinguishable to clients |
| Low | Dead exports | `allowedTransitions` and `isTerminal` are exported but unused by the service | `order.status.ts:16-22` | Domain capability is not exposed or tested; duplicated source of truth for terminality |
| Low | Missing resource detail | `NotFoundError('Product')` does not identify which product ids were missing | `order.service.ts:63-65` | Poor diagnostics for client-side correction |
| Low | No explicit isolation/timeout | The transactions do not set isolation level, timeout or max wait | `order.service.ts:58,131` | Behaviour depends on Prisma and MySQL defaults; lock waits can become 500s |
| Low | Seed divergence | Seed writes orders and history directly, with a `CANCELLED` flow that only originates from `PENDING` and no stock effects | `prisma/seed.ts:255-321` | Seeded data is not consistent with the lifecycle's stock policy; misleading for manual testing |
| Low | List ordering ties | Ordering by `createdAt` only | `order.repository.ts:38` | Non-deterministic order for rows created in the same millisecond |
| Low | Tests reset the sequence | `tests/setup.ts:12` deletes the sequence before each test | `tests/setup.ts:8-16` | Sequence monotonicity across orders is never verified; order-number tests always start at 1 |

Risk notes:

- The first two rows are structural, not cosmetic. They stem from the absence of a concurrency-control mechanism in a component that performs read-modify-write on shared inventory and shared lifecycle state.
- The numbering bottleneck is a direct consequence of choosing a single-row table as the sequence. It guarantees uniqueness and gap-free numbering but serialises all order creation.
- The audit-retention row is a policy consequence rather than an implementation bug: the cascade is declared correctly for referential integrity, but it conflicts with the audit purpose of the history table when combined with hard delete.
- Nothing in the component is asynchronous, retried or compensated. Every failure is immediate and, for the two commands, fully rolled back.

---

## 10. Test Coverage Analysis

### 10.1 Test inventory

| Test file | Type | Tests | How it reaches `OrderService` |
|-----------|------|-------|-------------------------------|
| `tests/orders.test.ts` | HTTP integration (Supertest against a real Express app and real MySQL) | 8 `it` blocks | Through `POST/GET/PATCH/DELETE /api/v1/orders`; no direct instantiation |
| `tests/auth.test.ts` | HTTP integration | 7 `it` blocks (per file content) | Not related to orders; only supplies the login helper used indirectly by `bootstrapAuthenticatedUser` |
| `tests/setup.ts` | Harness | 3 hooks | Connects, truncates order-related tables before each test, disconnects |
| `tests/helpers/factories.ts` | Harness | Helper functions | Builds the app, creates users/customers/products, logs in and returns a bearer token |

There are no unit tests for `OrderService`, no tests that instantiate the class with a mocked `PrismaClient`, and no tests for the private helpers. The project has no coverage instrumentation configured: `vitest.config.ts` contains no `coverage` block and `@vitest/coverage-v8` is not a dependency. Any coverage figure below is a qualitative estimate from mapping test cases to branches, not a measured percentage.

Note on documentation consistency: the architectural report states "10 integration tests" for `tests/orders.test.ts` at one point and "8 tests" at another. The file as analysed contains 8 `it` blocks. This report uses 8 as the measured value.

### 10.2 Test cases mapped to component behaviour

| Test (line) | Behaviour exercised | Service members | Assertions |
|-------------|---------------------|-----------------|------------|
| `creates an order in PENDING status and computes totals on the server` (`12-40`) | Happy-path creation with two products and a discount | `create`, `aggregateItems`, `reserveOrderNumber` | 201, `PENDING`, subtotal, discount, total, 2 items, 1 history row, `toStatus = PENDING`, `/^ORD-\d{6}$/` |
| `rejects an order with non-existent product` (`42-57`) | Product existence failure | `create` | 404 and `NOT_FOUND` |
| `transitions PENDING -> PAID and decrements stock` (`59-87`) | Debit path plus audit append | `changeStatus`, `debitStock` | 200, `PAID`, history `[PENDING, PAID]`, stock 5 -> 2 |
| `returns 409 on invalid status transition (PENDING -> SHIPPED)` (`89-107`) | Transition table enforcement | `changeStatus`, `canTransition` | 409 and `INVALID_STATUS_TRANSITION` |
| `returns 422 when transitioning PENDING -> PAID without enough stock` (`109-132`) | Stock shortfall, no partial debit, rollback | `changeStatus`, `debitStock` | 422, `INSUFFICIENT_STOCK`, `details.unavailable[0].sku`, stock unchanged at 1 |
| `replenishes stock when going PAID -> CANCELLED` (`134-161`) | Replenishment path | `changeStatus`, `replenishStock` | 200, stock 10 -> 6 after payment, 6 -> 10 after cancel |
| `lists orders filtered by status` (`163-195`) | List filter and pagination total | `list` (through repository) | 200, 1 paid row, 2 pending rows, `pagination.total` |
| `refuses to delete an order that is not PENDING or CANCELLED` (`197-218`) | Delete eligibility | `delete` | 409 and `INVALID_ORDER_STATE_FOR_DELETE` |

### 10.3 Per-member coverage assessment

| Member | Branches / rules | Covered branches | Uncovered branches | Qualitative coverage | Test quality |
|--------|------------------|------------------|--------------------|----------------------|--------------|
| `list` | Status filter, customer filter, date range, ordering, pagination math, empty page | Status filter, `total` computation, newest-first implicitly | `customerId`, `from`, `to`, reversed range, page bounds, page beyond end, tie ordering, absence of relations | Medium | Strong assertions on counts; no filter combination or date-bound test |
| `getById` | Found, not found, relation projection | None directly (only used implicitly by no test) | Both branches and all includes | None | Not exercised |
| `create` | Empty guard, aggregation, customer not found, product not found, inactive product, price snapshot, subtotal, discount bound, total, initial status, initial history, actor, notes, sequence format, transaction rollback | Happy path (2 items, discount), product not found, sequence regex, initial history, totals, actor implicitly | Empty guard (blocked by schema), duplicate aggregation, customer not found, inactive product, price tampering, discount equal to subtotal, discount above subtotal, notes null/empty/max, sequence monotonicity, rollback on late failure | Low to Medium | Good monetary and history assertions; no negative tests for most validations |
| `changeStatus` | Not found, same status, illegal pair, debit, insufficient stock, replenish, update, history, re-read, reason null/value | `PENDING -> PAID` debit, `PENDING -> SHIPPED` illegal, `PENDING -> PAID` shortfall, `PAID -> CANCELLED` replenish | Not found, same status, `PENDING -> CANCELLED`, `PAID -> PROCESSING`, `PROCESSING -> SHIPPED`, `PROCESSING -> CANCELLED`, `SHIPPED -> DELIVERED`, terminal-state attempts, reason null, re-read projection, concurrent double transition | Low to Medium | Strong side-effect assertions on stock and history for the covered paths |
| `delete` | Not found, PENDING success, CANCELLED success, other status refusal, cascade | Other-status refusal | Not found, successful delete, cascade verification, stock/audit effects, concurrency | Low | Only the negative path is tested |
| `aggregateItems` | Duplicate merge, insertion order, empty list | None directly | All branches | None | Only indirectly exercised with distinct products |
| `debitStock` | All-sufficient, one short, multiple short, missing product, no partial write | All-sufficient, one short, rollback of no writes | Multiple short lines, missing product, negative-result concurrency, multi-product partial ordering | Medium | Good rollback assertion (stock unchanged after 422) |
| `replenishStock` | Single item, multiple items | Single item (`PAID -> CANCELLED`) | Multiple items, `PROCESSING -> CANCELLED` | Low to Medium | Exact stock assertions |
| `reserveOrderNumber` | First creation, subsequent creation, rollback, concurrent creation, overflow | First creation and format | Monotonicity across multiple orders, rollback reuse, concurrency, overflow | Low | Regex only; sequence is reset before every test |

### 10.4 Coverage of business rules

| Rule | Covered by tests | Gap |
|------|------------------|-----|
| BR-01 minimum item | No (schema blocks it; no explicit assertion) | No test for empty `items` response shape |
| BR-02 duplicate aggregation | No | No test with repeated `productId` |
| BR-03 customer existence | No | No test for unknown customer |
| BR-04 product existence | Yes (`tests/orders.test.ts:42-57`) | Does not assert message or `details` |
| BR-05 inactive product | No | No test for `INACTIVE_PRODUCT` or the SKU list |
| BR-06 server-side pricing | Partially (totals derive from product prices) | No test that a client-supplied price is ignored |
| BR-07 integer arithmetic | Yes (exact subtotal and total) | No large-value or overflow test |
| BR-08 discount bounds | Partially (valid discount) | No test for negative, fractional (schema) or above-subtotal discount |
| BR-09 initial PENDING | Yes | None |
| BR-10 order numbers | Partially (format) | No monotonicity, uniqueness, rollback or concurrency test |
| BR-11 initial history | Yes (`history` length 1, `toStatus`) | Does not assert `fromStatus = null`, reason or actor |
| BR-12 actor attribution | Implicitly (token required) | No assertion that `createdById`/`changedById` equal the caller |
| BR-13 notes | No | No test for `null`, empty, 1000 or 1001 characters |
| BR-14 creation atomicity | Partially (product 404 leaves no order implicitly) | No assertion that a failed create leaves no sequence consumption or partial rows |
| BR-15 no stock check at creation | Yes (`109-119` creates with quantity 5 and stock 1) | None |
| BR-16 transition table | Yes (one illegal pair) | Only one of the many illegal pairs; no legal-path matrix |
| BR-17 same-status rejection | No | No test for `PAID -> PAID` |
| BR-18 terminal states | No | No test for `DELIVERED -> *` or `CANCELLED -> *` |
| BR-19 debit on payment | Yes | None |
| BR-20 all-or-nothing stock | Yes (single short line) | No multi-line partial scenario |
| BR-21 replenishment | Partially (`PAID -> CANCELLED`) | No `PROCESSING -> CANCELLED`; no `PENDING -> CANCELLED` no-replenish assertion |
| BR-22 transition atomicity | Yes implicitly (stock unchanged on failure) | No assertion that status and history are unchanged after a mid-transaction failure |
| BR-23 audit append | Yes for two transitions | No assertion on reason text, actor, or multiple rows ordering |
| BR-24 transition reason | Partially (sent but not asserted) | No assertion on persisted reason, null, empty or max length |
| BR-25 projection | Partially (items/history present) | No assertion on customer summary, product summary fields, or history ordering with multiple entries |
| BR-26 delete eligibility | Yes (refusal) | No successful deletion test |
| BR-27 cascade delete | No | No test that items and history disappear |
| BR-28 non-transactional delete | No | No concurrency test |
| BR-29 pagination | Partially (`pagination.total`) | No `page`, `pageSize`, `totalPages`, max/page-bound tests |
| BR-30 list filters | Partially (status only) | No customer or date-range filter test; no ordering assertion |
| BR-31 detail 404 | No | No `GET /orders/:id` tests at all |
| BR-32 no scoping | No | No cross-user access test (would require a second actor) |
| BR-33 active at creation only | No | No deactivate-then-pay test |

### 10.5 Test quality assessment

Strengths:

- Tests are true integration tests: real Express app, real Prisma, real MySQL, real JWT login. They validate the full path from HTTP to database, including middleware behaviour.
- Assertions focus on observable side effects that matter: persisted stock quantity after debit and replenishment, history sequence values, computed monetary totals, and machine-readable error codes.
- The insufficient-stock test asserts both the 422 response and the absence of a partial stock change, which is the most valuable rollback assertion in the suite.
- Test data is generated with unique emails and SKUs, avoiding cross-test collisions in the database.

Weaknesses:

- No unit tests, so private helpers and branch-level logic are only reached through a narrow set of HTTP scenarios.
- The suite covers roughly one legal and one illegal path per command. Most validation branches (inactive product, discount overflow, missing customer, same-status, terminal states) are untested.
- `getById` has zero tests, and successful deletion has zero tests.
- No concurrency tests, despite concurrency being the highest-severity risk in the component.
- No test for order-number monotonicity or uniqueness across multiple creations, because `tests/setup.ts:12` deletes the sequence before every test.
- Tests run serially in a single fork with `fileParallelism: false` (`vitest.config.ts:11-17`), which reduces interference but increases runtime and still does not make the truncation safe against a misconfigured `DATABASE_URL`.
- There is no coverage report, so regressions in uncovered branches are invisible in CI.

### 10.6 Coverage conclusion

Measured coverage cannot be stated because no instrumentation exists. Based on branch mapping, the component's public surface is approximately half-exercised: the happy path and the two most visible failure modes (missing product, insufficient stock) are covered, while most validation branches, both read endpoints for detail, successful deletion, audit details, numbering semantics and all concurrency behaviour are not. The private helpers are only covered through the integration paths that call them, and `aggregateItems` never receives duplicate input in any test.

---

## Analysis Notes and Ambiguities

1. **`OrderService` has no endpoints of its own.** As required by the analysis guidelines, the endpoint section is omitted. The HTTP surface is documented in the sibling analyses of `order.routes.ts` and `order.controller.ts`. For reference only, the service is reachable through `GET/POST /api/v1/orders`, `GET/DELETE /api/v1/orders/:id` and `PATCH /api/v1/orders/:id/status`.
2. **Afferent/efferent counts differ from the architectural report.** The report lists `OrderService` as Ca 2 / Ce 5. This analysis counts six efferent module dependencies because it separates the type-only imports (`order.repository.ts`, `order.schemas.ts`) from runtime imports. Both counts agree that the service has exactly two afferent dependents (`OrderController` and the composition root).
3. **Deferred stock control is inferred, not specified.** BR-15 is stated as behaviour with high confidence and as intent with medium confidence. No requirement document in the repository states that stock is checked only at payment; the evidence is the absence of a check and the explicit test that creation succeeds above stock.
4. **Authorization intent is ambiguous.** BR-32 describes the behaviour precisely. Whether the absence of role checks and tenant scoping is a deliberate simplification for a reference application or an oversight cannot be determined from the component. The architectural report treats it as a high risk.
5. **Transaction isolation and timeout behaviour depends on framework defaults.** The service sets no isolation level and no timeout. Statements about repeatable read and Prisma interactive-transaction limits are framework-level semantics (MySQL 8.0 InnoDB, Prisma 5.22), not explicit choices in this code. They are marked accordingly in the risk table.
6. **Sequence rollback behaviour assumes InnoDB transactional semantics.** The claim that failed creations do not consume order numbers follows from the increment being a normal row update inside the transaction. It is not tested and would not hold for a non-transactional sequence (for example, a Postgres `SEQUENCE`).
7. **Seed script divergence.** `prisma/seed.ts:255-321` reimplements order numbering and writes orders and history directly, bypassing the service and the state machine. Seeded `CANCELLED` orders originate only from `PENDING` and no stock is adjusted, so seeded data does not reflect the stock policy enforced by `changeStatus`. This is recorded as a low risk and as evidence that the lifecycle rules are not enforced at the database level.
8. **Two documentation inconsistencies were found.** The architectural report gives both "10 integration tests" and "8 tests" for `tests/orders.test.ts`; the file contains 8. The seed's `flowFromStatus` mapping for `CANCELLED` allows only a `PENDING` origin, while the service also allows cancellation from `PAID` and `PROCESSING`.
9. **No code was modified during this analysis.** All findings are read-only observations from the listed files.

---

## Report Metadata

| Field | Value |
|-------|-------|
| Component analysed | Order Service (`OrderService`) |
| Primary file | `src/modules/orders/order.service.ts` |
| Files read | `order.service.ts`, `order.controller.ts`, `order.routes.ts`, `order.repository.ts`, `order.schemas.ts`, `order.status.ts`, `src/app.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts`, `src/shared/http/response.ts`, `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql`, `prisma/seed.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `package.json`, `vitest.config.ts`, `docker-compose.yml`, `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md`, `docs/agents/MANIFEST.md` |
| Ignored folders | `node_modules`, `.git`, `dist`, `coverage` |
| Business rules extracted | 33 |
| Public methods analysed | 5 |
| Private helpers analysed | 4 |
| Test files located | 4 (2 test suites, 1 setup, 1 factory module) |
| Tests mapped to the component | 8 |
| Report status | Complete |
