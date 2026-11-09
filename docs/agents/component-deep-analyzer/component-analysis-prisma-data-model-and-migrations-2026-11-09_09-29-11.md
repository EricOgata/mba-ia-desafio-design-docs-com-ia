# Component Deep Analysis Report - Prisma Data Model and Migrations

| Attribute | Value |
|-----------|-------|
| Component | Prisma Data Model and Migrations |
| Primary files | `prisma/schema.prisma`, `prisma/migrations/` |
| Project scope | Entire project root (`/home/eric-ogata/codes/full_cycle/mba_engenharia_ia/desafios/desafio4/mba-ia-desafio-design-docs-com-ia`) |
| Ignored folders | `node_modules`, `.git`, `dist`, `coverage` |
| Analysis date | 2026-11-09 |
| Report path | `docs/agents/component-deep-analyzer/component-analysis-prisma-data-model-and-migrations-2026-11-09_09-29-11.md` |
| Sibling exclusion | Read/write behavior of the audit-trail model is covered by the Order Status History analysis; this report covers its schema, constraints and referential rules only |

---

## 1. Executive Summary

The component is the declarative persistence foundation of the Order Management System. It consists of `prisma/schema.prisma` (138 lines) and a single initial migration, `prisma/migrations/20260519182739_init/migration.sql` (125 lines), plus `prisma/migrations/migration_lock.toml`. It defines seven MySQL tables (`users`, `customers`, `products`, `orders`, `order_items`, `order_status_history`, `order_number_sequence`), two native enums (`UserRole`, `OrderStatus`), six foreign keys, and the indexes that support the application's read paths. The schema is consumed indirectly: `@prisma/client` is generated from it and imported by 18 distinct files across `src`, `prisma` and `tests`. The component exposes no HTTP, GraphQL or gRPC endpoints; it is a data contract, not a service.

The model is conventional and compact. Every aggregate root and child row uses a surrogate UUID primary key stored as `CHAR(36)`; the only exception is `OrderNumberSequence`, an `INT` single-row counter. Monetary values are signed 32-bit integers in cents, with no decimal, currency or rounding columns. The order aggregate is composed of `orders`, `order_items` and `order_status_history`; products and customers are referenced by foreign key rather than copied, except for the unit price on each order item. The schema is the authority for uniqueness (email on users and customers, SKU on products, order number on orders), column capacity, enum membership and foreign-key actions. Business invariants such as positive quantities, non-negative money, total arithmetic, discount ceilings and legal status transitions are enforced by application code, not by the database.

Key findings:

- **Schema and migration are in parity.** The single migration materializes every model, enum, index and relation declared in `prisma/schema.prisma`. There is no drift between the two files at the time of analysis.
- **Integrity is split between database and application.** The database enforces four unique keys and six foreign keys; it does not enforce a single `CHECK` constraint. Negative prices, negative stock, mismatched order totals, duplicate order lines and illegal status jumps are all representable in the database.
- **Referential actions are asymmetric by design.** Deleting an order cascades to `order_items` and `order_status_history`, while deleting a customer, product or user is restricted by inbound foreign keys. This preserves master data but makes the hard-delete endpoints for customers and products fail with `P2003` once dependents exist; `P2003` is not translated by `src/middlewares/error.middleware.ts:37-54` and surfaces as HTTP 500.
- **There is no optimistic concurrency control.** `orders` has no `version` column, `order_status_history` has no uniqueness on transitions, and the status-change read does not use `SELECT ... FOR UPDATE`. Two overlapping `PENDING -> PAID` requests can both pass the application checks and produce two history rows and two stock decrements.
- **Index coverage is single-column and partially ineffective.** The schema has no composite indexes for the common access paths `(status, createdAt)`, `(customerId, createdAt)` or `(orderId, changedAt)`. The `name` and `document` indexes cannot serve the `contains` searches used by the repositories because those queries produce leading-wildcard `LIKE` predicates.
- **Collation changes uniqueness semantics.** Every table is `utf8mb4_unicode_ci`, which is case-insensitive and accent-insensitive. Unique email and SKU comparisons therefore ignore case and accents, a behavior not stated in the schema or the API contracts.
- **Historical fidelity is partial.** `order_items.unitPriceCents` is a price snapshot, but the order does not snapshot the customer name, the shipping address or the product name/SKU. Later master-data edits change how historical orders are displayed, and address edits change the only address associated with past orders.
- **The migration workflow is development-centric.** `package.json` provides `db:migrate` (`prisma migrate dev`), `db:reset` (`prisma migrate reset --force`) and `db:seed`, but no `migrate deploy` or schema-validation step. The shadow database is configured by an environment variable that is not validated by `src/config/env.ts` and is not provisioned by `docker-compose.yml`.
- **Tests exercise behavior, not the schema contract.** Fifteen integration tests cover the order lifecycle and auth flow, including order-number format, totals, stock movement, history and unique email. There is no test that applies the migration to an empty database, no test for `SKU` uniqueness, foreign-key restrictions, cascade deletion, duplicate documents, negative values, enum rejection or concurrency, and no coverage instrumentation is configured.

---

## 2. Data Flow Analysis

### 2.1 Schema-to-runtime pipeline

The schema has no executable data flow of its own. Its declarations are transformed into a generated client and a physical database, and all runtime movement happens through those two artifacts.

```
1. Developer edits prisma/schema.prisma
2. prisma generate -> node_modules/.prisma/client (typed delegates, enums, input types)
3. src/config/database.ts constructs the process-wide PrismaClient
4. src/app.ts injects the client into repositories and OrderService
5. Repositories/services issue typed queries; Prisma translates them to SQL
6. MySQL executes against the seven tables and returns rows
7. Prisma maps rows to generated TypeScript model types
8. Services project or combine models for HTTP responses
```

### 2.2 Order creation write flow

```
1. POST /api/v1/orders enters order.routes.ts
2. validate(createOrderSchema) checks customerId UUID, productId UUIDs, quantity > 0, discountCents >= 0
3. OrderController.create calls OrderService.create(input, userId)
4. OrderService aggregates duplicate product lines in memory (order.service.ts:194-202)
5. BEGIN interactive transaction (order.service.ts:58)
6. SELECT customers WHERE id = customerId            -> 404 if absent
7. SELECT products WHERE id IN (productIds)          -> 404 if any absent
8. Reject inactive products                           -> 422 INACTIVE_PRODUCT
9. Compute unitPriceCents from products.priceCents, line totalCents, subtotalCents
10. Reject discountCents > subtotalCents              -> 400 VALIDATION_ERROR
11. UPSERT order_number_sequence id=1, increment nextValue, derive ORD-nnnnnn
12. INSERT orders (status PENDING, totals, createdById)
13. INSERT order_items (nested create)
14. INSERT order_status_history (fromStatus null, toStatus PENDING, reason 'order created')
15. COMMIT
16. Re-read order with items/product, history ordered by changedAt asc, customer summary
17. Return 201
```

The sequence upsert at step 11 obtains an exclusive row lock on the single counter row and serializes all concurrent order creations until commit. The counter and the order insert are in the same transaction, so a rolled-back order does not consume a number.

### 2.3 Status-change write flow

```
1. PATCH /api/v1/orders/:id/status enters order.routes.ts
2. validate(orderIdParamSchema, updateOrderStatusSchema) checks UUID and enum value
3. OrderController.changeStatus calls OrderService.changeStatus(id, input, userId)
4. BEGIN interactive transaction (order.service.ts:131)
5. SELECT orders + order_items WHERE id = id          -> 404 if absent
6. Reject from === to                                   -> 409 INVALID_STATUS_TRANSITION
7. canTransition(from, to) from order.status.ts         -> 409 if illegal
8. If PENDING -> PAID: SELECT products, check stock, UPDATE products SET stockQuantity = stockQuantity - qty
9. If PAID|PROCESSING -> CANCELLED: UPDATE products SET stockQuantity = stockQuantity + qty
10. UPDATE orders SET status = to, updatedAt = now
11. INSERT order_status_history (from, to, changedById, reason)
12. Re-read order with items/product, history, customer
13. COMMIT
14. Return 200
```

The read at step 5 does not lock the order row. Under MySQL's default REPEATABLE READ, two concurrent transactions can both observe the same source status and both proceed through steps 7-11. The product updates at steps 8-9 are current reads and will apply both decrements. This is a data-model-level concurrency gap, not merely a service bug, because the schema provides no version column, no unique transition constraint and no lock hint.

### 2.4 Order deletion and referential cascade flow

```
1. DELETE /api/v1/orders/:id enters order.routes.ts
2. OrderController.delete calls OrderService.delete(id)
3. SELECT orders WHERE id = id                         -> 404 if absent
4. Reject unless status is PENDING or CANCELLED        -> 409 INVALID_ORDER_STATE_FOR_DELETE
5. DELETE orders WHERE id = id
6. InnoDB cascades DELETE to order_items (FK ON DELETE CASCADE)
7. InnoDB cascades DELETE to order_status_history (FK ON DELETE CASCADE)
8. Return 204
```

The guard at step 4 is a separate read outside any transaction. A concurrent transition can commit between steps 4 and 5, allowing a PAID order to be deleted. There is no tombstone: the order, its lines and its audit trail are physically removed.

### 2.5 Master-data write flows

```
User registration:
  auth.routes -> AuthService.register -> UserService.createUser
  -> SELECT users WHERE email (uniqueness pre-check)
  -> bcrypt hash -> INSERT users (role from request)

Customer create/update:
  customer.routes -> CustomerService -> CustomerRepository
  -> SELECT customers WHERE email (uniqueness pre-check)
  -> INSERT/UPDATE customers (address stored as JSON)

Product create/update:
  product.routes -> ProductService -> ProductRepository
  -> SELECT products WHERE sku (uniqueness pre-check)
  -> INSERT/UPDATE products (priceCents, stockQuantity, active)

Customer delete / Product delete:
  service existence check -> DELETE customers/products
  -> if any orders/order_items reference the row, MySQL raises P2003
  -> error middleware has no P2003 branch -> HTTP 500
```

### 2.6 Read flows and index use

```
Order list:
  SELECT orders WHERE [status] AND [customerId] AND createdAt BETWEEN from AND to
  ORDER BY createdAt DESC LIMIT take OFFSET skip
  SELECT COUNT(*) with the same WHERE
  -> uses one single-column index at a time; ORDER BY may require filesort

Order detail:
  SELECT orders WHERE id = ?
  -> SELECT order_items WHERE orderId = ? (index)
  -> SELECT products WHERE id IN (...) (PK)
  -> SELECT order_status_history WHERE orderId = ? ORDER BY changedAt ASC (orderId index + sort)
  -> SELECT customers WHERE id = ? (PK)

Customer list:
  SELECT customers WHERE name LIKE '%term%' OR email LIKE '%term%' OR document LIKE '%term%'
  ORDER BY createdAt DESC LIMIT/OFFSET
  -> leading wildcard prevents use of customers_document_idx; no email/name search index

Product list:
  SELECT products WHERE active = ? AND (name LIKE '%term%' OR sku LIKE '%term%')
  ORDER BY createdAt DESC LIMIT/OFFSET
  -> active index may apply; name index cannot serve the leading wildcard
```

### 2.7 Migration data flow

```
1. Developer edits prisma/schema.prisma
2. `prisma migrate dev` diffs the schema against the shadow database (SHADOW_DATABASE_URL)
3. Prisma writes a new timestamped migration SQL file
4. Prisma applies pending migrations to DATABASE_URL and records them in _prisma_migrations
5. Prisma regenerates the client
6. `prisma db seed` or `npm run db:seed` optionally repopulates development data
```

The repository currently has exactly one migration, `20260519182739_init`, so the history consists of a single baseline. There is no migration for a previous production state and no down migration; rollback is achieved by `prisma migrate reset --force`, which drops and recreates the schema.

### 2.8 Seed flow

`prisma/seed.ts` deletes all rows in FK-safe order, then inserts two users, ten customers, twenty products, twenty-six orders with items and status histories. It uses the same schema contracts as the application and demonstrates the intended value shapes (formatted CPF/CNPJ documents, JSON addresses, integer cents, 5 percent discount on every seventh order). The seed is a destructive development tool, not a production data path; its detailed behavior belongs to the Seed Script component analysis.

---

## 3. Business Rules & Logic

### 3.1 Overview of the business rules

| ID | Rule Type | Rule Description | Location |
|----|-----------|------------------|----------|
| BR-01 | Identity / Convention | Every table uses a surrogate UUID primary key stored as `CHAR(36)`, generated by Prisma; only `OrderNumberSequence` uses an `INT` key | `prisma/schema.prisma:26,41,57,75,100,117`; `prisma/migrations/20260519182739_init/migration.sql:3,17,33,51,73,88` |
| BR-02 | Business Identifier | `orders.orderNumber` is a unique, human-readable `ORD-nnnnnn` identifier generated inside the order transaction | `prisma/schema.prisma:76`; `migration.sql:52,63`; `src/modules/orders/order.service.ts:245-254` |
| BR-03 | Concurrency / Numbering | A single-row `order_number_sequence` table supplies monotonic order numbers and serializes concurrent order creation | `prisma/schema.prisma:133-138`; `migration.sql:102-107` |
| BR-04 | Value Representation | All monetary values are signed 32-bit integers in cents; there is no decimal, currency or rounding column | `prisma/schema.prisma:61,79-81,104-105`; `migration.sql:37,55-57,77-78` |
| BR-05 | Validation / Calculation | Order totals are computed server-side: `totalCents = subtotalCents - discountCents`, and discount cannot exceed subtotal | `prisma/schema.prisma:79-81`; `src/modules/orders/order.service.ts:86-92` |
| BR-06 | Historical Pricing | Each order item stores `unitPriceCents` and `totalCents` as a price snapshot taken at order creation | `prisma/schema.prisma:104-105`; `src/modules/orders/order.service.ts:75-84` |
| BR-07 | State Domain | `orders.status` is a six-value `OrderStatus` enum with database default `PENDING` | `prisma/schema.prisma:16-23,78`; `migration.sql:54` |
| BR-08 | Workflow / Lifecycle | Legal order transitions are defined by the application state machine; the database enum enforces membership only | `src/modules/orders/order.status.ts:3-14`; `src/modules/orders/order.service.ts:140-149` |
| BR-09 | Aggregate Ownership | Every order references exactly one existing customer and one existing creator user through non-null foreign keys | `prisma/schema.prisma:77,83,87-88`; `migration.sql:53,59,110-113` |
| BR-10 | Aggregate Composition | An order must contain at least one item; duplicate product lines are merged by the service, but the database has no composite unique key | `src/modules/orders/order.service.ts:51-55,194-202`; `prisma/schema.prisma:99-114` |
| BR-11 | Master Data Identity | Product `sku` is unique; products carry an `active` flag and a current price and stock quantity | `prisma/schema.prisma:58,61-63`; `migration.sql:37-45`; `src/modules/products/product.service.ts:31-34,47-51` |
| BR-12 | Identity / Access | User `email` is unique; `role` is restricted to `ADMIN` or `OPERATOR` with default `OPERATOR`; passwords are stored as bcrypt hashes | `prisma/schema.prisma:27-30`; `migration.sql:4-7,11`; `src/modules/users/user.service.ts:16,21-33` |
| BR-13 | Master Data Identity | Customer `email` is unique; `document` is indexed but not unique or normalized | `prisma/schema.prisma:43,45,52`; `migration.sql:19,26-27`; `src/modules/customers/customer.service.ts:30-33` |
| BR-14 | Inventory Integrity | Stock defaults to zero, is debited on `PENDING -> PAID`, and replenished on `PAID` or `PROCESSING -> CANCELLED`; the database has no non-negative check | `prisma/schema.prisma:62`; `src/modules/orders/order.status.ts:24-37`; `src/modules/orders/order.service.ts:151-156,204-243` |
| BR-15 | Referential Integrity | `orders.customerId -> customers.id` and `orders.createdById -> users.id` use `ON DELETE RESTRICT` and `ON UPDATE CASCADE` | `prisma/schema.prisma:87-88`; `migration.sql:110-113` |
| BR-16 | Referential Integrity | `order_items.orderId -> orders.id` cascades on delete; `order_items.productId -> products.id` restricts on delete | `prisma/schema.prisma:108-109`; `migration.sql:116,119` |
| BR-17 | Referential Integrity | `order_status_history.orderId -> orders.id` cascades on delete; `changedById -> users.id` restricts on delete | `prisma/schema.prisma:125-126`; `migration.sql:122,125` |
| BR-18 | Deletion Lifecycle | There is no soft-delete column anywhere; deletes are physical. Orders may be deleted only in `PENDING` or `CANCELLED`, and children are removed by cascade | `prisma/schema.prisma:108,125`; `migration.sql:116,122`; `src/modules/orders/order.service.ts:181-192` |
| BR-19 | Audit Constraints | Status history is append-only by convention: nullable `fromStatus`, required `toStatus` and actor, optional reason up to 500 characters, default `changedAt` | `prisma/schema.prisma:116-131`; `migration.sql:87-99`; `src/modules/orders/order.service.ts:106-113,159-167` |
| BR-20 | Temporal Convention | `createdAt` has a database default of `CURRENT_TIMESTAMP(3)`; `updatedAt` is required and maintained by Prisma; all timestamps are `DATETIME(3)` without timezone | `prisma/schema.prisma:31-32,47-48,64-65,84-85,106,121`; `migration.sql:8-9,23-24,40-41,60-61,79,92` |
| BR-21 | Data Shape | Each customer has exactly one required `address` JSON object; its shape is validated only by Zod, not by MySQL | `prisma/schema.prisma:46`; `migration.sql:22`; `src/modules/customers/customer.schemas.ts:5-11` |
| BR-22 | Comparison Semantics | All tables use `utf8mb4` with `utf8mb4_unicode_ci`, making comparisons and unique keys case-insensitive and accent-insensitive | `migration.sql:13,29,47,69,84,99,107`; `docker-compose.yml:28-29` |
| BR-23 | Governance | `schema.prisma` is the source of truth and the single init migration must match it; migration workflow is `migrate dev` plus shadow database and destructive reset | `prisma/schema.prisma:1-9`; `migration.sql:1-125`; `prisma/migrations/migration_lock.toml:3`; `package.json:11-13` |
| BR-24 | Naming Convention | Prisma models are singular PascalCase, database tables are snake_case plural via `@@map`, and columns retain camelCase names | `prisma/schema.prisma:37,53,71,96,113,130,137`; `migration.sql:2,16,32,50,72,87,102` |
| BR-25 | Scope / Uniqueness | Uniqueness is global within each table; there is no tenant, organization or workspace column | `prisma/schema.prisma:27,43,58,76`; `migration.sql:11,26,43,63` |
| BR-26 | Free-text Bounds | `Product.description`, `Order.notes` and `OrderStatusHistory.reason` are optional strings with different database and application limits | `prisma/schema.prisma:60,82,123`; `src/modules/products/product.schemas.ts:6`; `src/modules/orders/order.schemas.ts:15,20` |

### 3.2 Detailed breakdown of the business rules

---

### Business Rule BR-01: UUID Surrogate Primary Keys

**Overview**:
Every table uses a surrogate identifier instead of a natural or auto-increment key. The type is `String`, annotated with `@id`, `@default(uuid())` and `@db.Char(36)`. `OrderNumberSequence` is the deliberate exception, using `Int @id @default(1)`.

**Detailed description**:

The schema declares `id String @id @default(uuid()) @db.Char(36)` on `User`, `Customer`, `Product`, `Order`, `OrderItem` and `OrderStatusHistory` (`prisma/schema.prisma:26,41,57,75,100,117`). The generated DDL creates `CHAR(36) NOT NULL` primary keys without a database default (`migration.sql:3,17,33,51,73,88`). The UUID value is therefore generated by the Prisma client at insert time, not by MySQL. Every foreign-key column that points at one of these tables repeats the same `CHAR(36)` type, so the key width and collation are uniform across the schema. The separation between model and table name is handled by `@@map`, but the identifier itself is never remapped.

The business purpose is stable, opaque identity. Because IDs are random, they do not disclose row counts or creation order, and they remain valid if records are merged or migrated between environments. The API boundary reinforces this rule: Zod validates path and body identifiers with `z.string().uuid()` in `src/modules/orders/order.schemas.ts:4,7,12`, `src/modules/customers/customer.schemas.ts:4`, `src/modules/products/product.schemas.ts:3` and `src/modules/users/user.schemas.ts:4`. The repositories, however, accept any string and pass it to Prisma, so the format is guaranteed only for HTTP callers, not for internal or direct database access.

The trade-offs are storage and index behavior. A UUID stored as `CHAR(36)` occupies a fixed 36-character slot (up to 144 bytes under `utf8mb4`) per key and per foreign-key copy, versus 16 bytes for a binary UUID. Random version-4 values are not monotonically increasing, so inserts scatter across the B-tree and can cause page splits at high volume. The rule is nevertheless consistent and makes cross-table references portable. Any change to UUID version or storage format would require a schema-wide migration touching every primary and foreign key, which is a significant boundary constraint.

**Rule workflow**:

```
1. Application calls prisma.<model>.create without an id
2. Prisma client generates a UUID
3. INSERT writes CHAR(36) into the primary key
4. Child rows copy the parent id into their CHAR(36) foreign-key column
5. Reads and joins match on the 36-character value
```

---

### Business Rule BR-02: Human-Readable Order Number

**Overview**:
`orders.orderNumber` is a unique business identifier in the form `ORD-nnnnnn`, separate from the UUID primary key. It is the reference exposed to users and external systems.

**Detailed description**:

The column is `orderNumber String @unique @db.VarChar(20)` (`prisma/schema.prisma:76`), materialized as `VARCHAR(20) NOT NULL` with `UNIQUE INDEX orders_orderNumber_key` (`migration.sql:52,63`). The value is not generated by the database. `OrderService.reserveOrderNumber` reads the next counter value inside the order transaction and formats it as `ORD-` plus a six-digit zero-padded number (`src/modules/orders/order.service.ts:245-254`). The seed script contains a byte-for-byte duplicate of that formatting logic (`prisma/seed.ts:165-176`), so two independent producers must remain aligned.

The business meaning is a short, speakable, support-friendly identifier that can appear on invoices, emails and webhook payloads. The database guarantees global uniqueness, which prevents two orders from sharing a reference even if application logic fails. It does not guarantee the format: a direct insert can store any string up to 20 characters, and there is no `CHECK` constraint or trigger enforcing `ORD-` or six digits. The API test asserts the format for the normal creation path (`tests/orders.test.ts:39`), but that is evidence about the service, not about the schema.

The format has a soft capacity boundary. Six digits cover 999,999 orders; beyond that, `padStart(6, '0')` produces a seven-digit suffix, which still fits `VARCHAR(20)` and remains unique. Deletion never reuses a number because the sequence is monotonic and is not decremented when an order is removed. This makes `orderNumber` a durable external reference, but it also means deleted orders leave permanent gaps and dangling external references. The identifier is immutable in the service layer: there is no update path for `orderNumber`.

**Rule workflow**:

```
1. OrderService opens the order transaction
2. UPSERT order_number_sequence id=1 increments nextValue
3. current = nextValue - 1
4. format `ORD-${String(current).padStart(6, '0')}`
5. INSERT orders with the formatted value
6. Unique index rejects any collision at commit
```

---

### Business Rule BR-03: Transactional Single-Row Sequence

**Overview**:
`order_number_sequence` is a one-row counter table (`id INT @id @default(1)`, `nextValue INT @default(1)`) used to allocate order numbers without gaps inside the order transaction.

**Detailed description**:

The table is declared at `prisma/schema.prisma:133-138` and created at `migration.sql:102-107`. It has no foreign keys, no timestamps and no history. The application addresses it exclusively through `where: { id: 1 }` in `OrderService.reserveOrderNumber` (`src/modules/orders/order.service.ts:245-254`) and in the seed (`prisma/seed.ts:165-176`). The first call uses `create: { id: 1, nextValue: 2 }` and then derives `current = 1`, so the first order is `ORD-000001`; subsequent calls use `update: { nextValue: { increment: 1 } }`.

Because the increment and the order insert share one interactive transaction, a rollback also rolls back the counter, so numbers are not consumed by failed orders. The row-level lock held by the `UPDATE` serializes concurrent order creation: only one transaction can increment the counter at a time, and the others wait until commit. This provides gap-free numbering at the cost of a single hot row. The seed deletes the sequence row and rebuilds it, so a full reset restarts numbering at one.

The rule is enforced by convention, not by the database. The primary key allows additional rows with `id = 2`, `id = 3` and so on; nothing constrains the table to a single row, and nothing checks that `nextValue >= 1`. Any direct writer that inserts a second row or resets the value can break the numbering contract. The table is also a throughput ceiling: every order creation must acquire the same lock, so order-creation concurrency is serialized regardless of how many application instances run.

**Rule workflow**:

```
1. BEGIN transaction
2. UPSERT id=1: if absent create nextValue=2; if present increment by 1
3. Read returned nextValue
4. Derive current = nextValue - 1
5. Format and insert the order
6. COMMIT releases the row lock
7. ROLLBACK returns the counter to its previous value
```

---

### Business Rule BR-04: Integer-Cents Monetary Convention

**Overview**:
All money is represented as signed 32-bit integer cents. The schema has no decimal type, no currency code and no rounding policy column.

**Detailed description**:

The monetary columns are `Product.priceCents`, `Order.subtotalCents`, `Order.discountCents`, `Order.totalCents`, `OrderItem.unitPriceCents` and `OrderItem.totalCents` (`prisma/schema.prisma:61,79-81,104-105`). The migration maps all of them to MySQL `INTEGER NOT NULL` (`migration.sql:37,55-57,77-78`), with defaults of `0` only for `discountCents`; the other money columns are required. The naming convention makes the unit explicit and prevents the common error of mixing reais and cents.

The application reinforces the convention at the boundary: `priceCents` and `stockQuantity` are `z.number().int().nonnegative()` in `src/modules/products/product.schemas.ts:8-9`, and `discountCents` is `z.number().int().nonnegative().default(0)` in `src/modules/orders/order.schemas.ts:14`. Arithmetic is performed with native integers in the service and seed, so there is no floating-point drift. The database does not enforce non-negativity, however; a direct write can store a negative price, discount or total.

The model assumes a single currency. There is no `currency` column, no exchange rate and no tax or shipping amount: the totals cover merchandise and discount only. The signed `INT` range is 2,147,483,647 cents, approximately 21.47 million in the currency unit, per column. A line total is computed as `unitPriceCents * quantity` in JavaScript before insertion; if that product exceeds the MySQL `INT` range, the insert fails at the database rather than being clamped. Reporting and formatting are left to the API and clients.

**Rule workflow**:

```
1. Product price is received and validated as a non-negative integer
2. Order item total = product.priceCents * quantity
3. Subtotal = sum of item totals
4. Discount is validated as a non-negative integer <= subtotal
5. Total = subtotal - discount
6. All six values are persisted as INTEGER cents
```

---

### Business Rule BR-05: Server-Computed Totals and Discount Bound

**Overview**:
Order financial totals are derived by the server from product prices and quantities; clients cannot supply `subtotalCents` or `totalCents`, and a discount may not exceed the subtotal.

**Detailed description**:

`createOrderSchema` accepts only `customerId`, `items`, `discountCents` and `notes` (`src/modules/orders/order.schemas.ts:11-16`). It does not accept subtotal or total, so those values cannot be injected through the public API. `OrderService.create` looks up each product, copies `product.priceCents` into `unitPriceCents`, multiplies by the requested quantity to obtain `totalCents`, sums the lines into `subtotalCents`, rejects `discountCents > subtotalCents` with a `ValidationError`, and stores `totalCents = subtotalCents - discountCents` (`src/modules/orders/order.service.ts:75-92`). The seed follows the same formula (`prisma/seed.ts:269-283`).

The rule protects commercial correctness at the application boundary. A caller cannot underpay by claiming a lower subtotal, and a discount cannot create a negative total through the supported path. The service also merges duplicate product lines before pricing, so the same product is priced once but with the summed quantity. The order's initial status and its audit row are written in the same nested create, so the financial and lifecycle records are atomic.

The database is not a second line of defense. It accepts any `INTEGER` in the three columns and has no check that `totalCents = subtotalCents - discountCents`, that `subtotalCents` equals the sum of `order_items.totalCents`, or that all values are non-negative. A direct SQL write, a future repository method, or a bug in another service can persist an inconsistent order. That makes the rule a service-level invariant with database-level permissiveness, a mismatch that matters for reporting, reconciliation and any future integration that reads the tables directly.

**Rule workflow**:

```
1. Validate request body (discountCents integer >= 0)
2. Load products and reject missing or inactive products
3. Build line prices from current product prices
4. subtotalCents = sum(line.totalCents)
5. If discountCents > subtotalCents -> 400 VALIDATION_ERROR
6. totalCents = subtotalCents - discountCents
7. INSERT order and lines with the computed values
```

---

### Business Rule BR-06: Price Snapshot on Order Items

**Overview**:
`order_items.unitPriceCents` and `order_items.totalCents` capture the price at the moment the order is created, preserving the commercial terms even if the product price later changes.

**Detailed description**:

`OrderItem` stores both the unit price and the line total (`prisma/schema.prisma:104-105`), and the service populates them from the current `Product.priceCents` (`src/modules/orders/order.service.ts:75-84`). The seed does the same (`prisma/seed.ts:269-279`). The columns are required and have no default, so every insert must supply a value. `OrderItem` also has `createdAt` but no `updatedAt`, and no service method updates an existing line; the record is effectively immutable after creation.

The business purpose is historical accuracy for invoices, refunds and customer disputes. If a product's list price rises or falls after an order is placed, the order continues to show what the customer agreed to pay. This is the correct behavior for a commercial document and distinguishes `order_items` from a simple join table. The snapshot is per line, not per order, so different lines can carry different prices even for the same product if the product price changed between two orders.

The snapshot is incomplete. The order does not store a product name or SKU copy; `OrderRepository.findByIdWithRelations` includes `product: { select: { id, sku, name } }` (`src/modules/orders/order.repository.ts:50-55`) and therefore displays the current product name and SKU. Renaming a product or changing its SKU changes how historical orders appear, while the price remains frozen. There is also no customer name, email or shipping-address snapshot on the order, so master-data edits can alter the presentation of past orders. The rule therefore guarantees price fidelity but not full document fidelity.

**Rule workflow**:

```
1. Order creation loads the product row
2. unitPriceCents = product.priceCents at that moment
3. totalCents = unitPriceCents * quantity
4. INSERT order_items with both values
5. Later product price updates do not touch existing order_items
6. Historical reads return the stored price and the current product name/SKU
```

---

### Business Rule BR-07: Order Status Enum and Default PENDING

**Overview**:
`Order.status` is a native MySQL `ENUM` with six members (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`) and a database default of `PENDING`.

**Detailed description**:

The `OrderStatus` enum is declared at `prisma/schema.prisma:16-23`, and `Order.status` uses it with `@default(PENDING)` at line 78. The migration materializes the enum inline in the column definition and sets `NOT NULL DEFAULT 'PENDING'` (`migration.sql:54`). Because the value set is part of the column type, MySQL rejects any string outside the six members when strict mode is active. The application also sets `PENDING` explicitly in `OrderService.create` (`src/modules/orders/order.service.ts:99`), so normal inserts do not rely on the database default.

The default is a business statement: a newly created order is not yet paid. It allows the order row and its initial history row to be written before any payment interaction. The enum is also the contract shared with Zod: `updateOrderStatusSchema` uses `z.nativeEnum(OrderStatus)` (`src/modules/orders/order.schemas.ts:19`), so the same six values are accepted at the HTTP boundary. This keeps the enum definition in one generated place for both runtime and validation.

The enum describes the state domain, not the lifecycle. It cannot express which transitions are legal, and it has no ordering semantics: the declaration order does not mean `SHIPPED` is "greater than" `PAID`. Adding a new status requires an `ALTER TABLE` migration that changes the column definition, regeneration of the Prisma client, and updates to the state-machine table and any consumers. Removing or reordering values has a wider operational impact. The database also permits rows to remain in a status indefinitely; there is no timestamp per status and no expiry rule.

**Rule workflow**:

```
1. Order insert omits status (or explicitly sets PENDING)
2. MySQL applies DEFAULT 'PENDING'
3. HTTP status updates validate toStatus against the generated enum
4. MySQL rejects values outside the six members
5. Legal transition checks are performed separately in the service
```

---

### Business Rule BR-08: Order Lifecycle Transition Legality

**Overview**:
The order lifecycle is a directed state machine. Legal transitions are declared in application code; the database stores the current status but cannot enforce the transition graph.

**Detailed description**:

`src/modules/orders/order.status.ts:3-14` defines the transition map: `PENDING` may go to `PAID` or `CANCELLED`; `PAID` to `PROCESSING` or `CANCELLED`; `PROCESSING` to `SHIPPED` or `CANCELLED`; `SHIPPED` only to `DELIVERED`; `DELIVERED` and `CANCELLED` are terminal. `OrderService.changeStatus` reads the current order, rejects `from === to` with a conflict, then calls `canTransition(from, to)` and throws `InvalidStatusTransitionError` if the pair is not listed (`src/modules/orders/order.service.ts:138-149`). The same module exposes `shouldDebitStock` and `shouldReplenishStock`, which couple two specific edges to inventory movement.

The database contains no transition table, trigger or check constraint. It stores any of the six enum values in any order. A direct `UPDATE orders SET status = 'DELIVERED'` succeeds regardless of the current value. The rule is therefore only as strong as the service layer and the code paths that use it. The seed script deliberately writes historical statuses directly and builds a plausible history path for each one, which shows that privileged writers can bypass the state machine.

The most important consequence is concurrency. The status read and the status update are separated by application logic and are not protected by a row lock or version column. Two concurrent requests can both read `PENDING` and both transition to `PAID`, producing two audit rows and two stock decrements. The schema offers no `@@unique([orderId, toStatus])` or version field to prevent this. The state machine is correct in a single-writer model but not enforced for concurrent writers.

**Rule workflow**:

```
1. Read order and current status
2. If from === to -> 409 conflict
3. Look up transitions[from]
4. If to not in list -> 409 INVALID_STATUS_TRANSITION
5. Apply stock side effects for the specific edge
6. Update status and append history in the same transaction
7. Commit; no version or lock prevents a parallel identical transition
```

---

### Business Rule BR-09: Order Aggregate Ownership

**Overview**:
Every order belongs to exactly one customer and records exactly one creator user. Both relationships are non-null foreign keys, so an order cannot exist without a customer or without an accountable user.

**Detailed description**:

`Order.customerId` and `Order.createdById` are non-null `String @db.Char(36)` columns (`prisma/schema.prisma:77,83`). The relations are declared at lines 87-88, and the migration adds `orders_customerId_fkey` and `orders_createdById_fkey` with `ON DELETE RESTRICT ON UPDATE CASCADE` (`migration.sql:110-113`). The service supplies `customerId` from the request and `createdById` from the authenticated principal (`src/modules/orders/order.service.ts:98,104`), so the creator is always a real user id rather than a free-text name.

Ownership has two distinct meanings. The customer reference is the commercial counterparty and the basis for order history, filtering and future notifications. The creator reference is the audit actor for the initial history row and the accountability link for who booked the order. Separating them is important because an operator can create an order for a customer, and the same operator may later change its status. The schema does not store a customer snapshot, so the order always points at the current customer record.

The restrict rule protects referential meaning. Deleting a customer with orders is rejected by MySQL, and deleting a user referenced as a creator is rejected as well. That is correct for auditability but has an operational consequence: the customer delete endpoint exists (`src/modules/customers/customer.routes.ts`) and will fail with a foreign-key error once the customer has any order. The error middleware does not translate `P2003`, so the caller receives HTTP 500 rather than a conflict. The user model has no delete endpoint, so its restriction is latent.

**Rule workflow**:

```
1. Authenticated request supplies customerId; the token supplies userId
2. Service verifies the customer row exists
3. INSERT order with customerId and createdById
4. Foreign keys validate both parent rows
5. Deleting either parent is RESTRICTed while the order exists
6. ON UPDATE CASCADE would follow an id change, which the application never performs
```

---

### Business Rule BR-10: Order Composition and Line Uniqueness

**Overview**:
An order must contain at least one item. The service merges duplicate product lines into one line per product, but the database has no composite unique constraint to enforce that shape.

**Detailed description**:

`createOrderSchema` requires `items` to be an array with at least one element (`src/modules/orders/order.schemas.ts:13`), and `OrderService.create` repeats the check with a `ValidationError` (`src/modules/orders/order.service.ts:51-53`). Each item requires a UUID `productId` and a positive integer `quantity` (`src/modules/orders/order.schemas.ts:6-9`). The service then calls `aggregateItems`, which sums quantities per product id and returns one entry per distinct product (`src/modules/orders/order.service.ts:194-202`). The order is persisted with a nested `items.create`, so the parent and all lines commit together.

The business intent is a clean commercial document: one line per product, with a quantity that represents the total requested. Aggregating in the service also prevents a client from splitting the same product into multiple lines to obscure the total. The database does not share this intent. `order_items` has only a primary key on `id` and non-unique indexes on `orderId` and `productId` (`prisma/schema.prisma:111-112`); there is no `@@unique([orderId, productId])`. Direct writes, imports or future code paths can create duplicate lines for the same product in the same order.

The quantity itself is also only application-validated. The column is `quantity Int` with no check, so zero or negative quantities can be stored by direct SQL. There is no upper bound on the number of lines per order and no database rule tying `totalCents` to `quantity * unitPriceCents`. These gaps do not affect the supported HTTP path, but they define the difference between the service contract and the storage contract.

**Rule workflow**:

```
1. Validate items array length >= 1
2. Validate each productId as UUID and quantity as positive integer
3. Merge duplicate productId entries by summing quantities
4. Load and validate all referenced products
5. INSERT order and aggregated lines in one transaction
6. Database accepts the insert without a composite uniqueness check
```

---

### Business Rule BR-11: Product Catalog Identity and Availability

**Overview**:
Products are identified by a unique SKU and carry a current price, a stock quantity and an `active` flag. Inactive products cannot be ordered through the service, and SKU changes are checked for uniqueness.

**Detailed description**:

`Product.sku` is `String @unique @db.VarChar(64)` (`prisma/schema.prisma:58`), materialized as `UNIQUE INDEX products_sku_key` (`migration.sql:43`). `active` is `Boolean @default(true)` (line 63) with a non-unique index (line 69), `priceCents` is a required `Int` (line 61), and `stockQuantity` is an `Int` defaulting to zero (line 62). The service pre-checks SKU uniqueness on create and update (`src/modules/products/product.service.ts:31-34,47-51`), while the unique index remains the final authority and produces `P2002` if the pre-check races.

The `active` flag is a soft availability switch for the catalog. `OrderService.create` loads products and rejects the order with `422 INACTIVE_PRODUCT` if any selected product is inactive (`src/modules/orders/order.service.ts:66-73`). This allows a product to be retired from sale without deleting its row, which is necessary because historical `order_items` still reference it. The flag is not a full soft-delete mechanism: there is no `deletedAt`, no `deletedBy` and no rule preventing direct updates to inactive products.

Product master data is mutable. `ProductService.update` allows changes to SKU, name, description, price, stock and active status (`src/modules/products/product.service.ts:53-60`). Because `order_items` stores only the price snapshot and references the product by id, a SKU or name change alters how historical orders are displayed while the stored price remains unchanged. Hard deletion is blocked by the `RESTRICT` foreign key once any order item references the product, so the practical lifecycle for a used product is deactivation, not deletion. The database does not enforce that an inactive product cannot receive new stock or price changes; that is a process convention.

**Rule workflow**:

```
1. Create or update validates sku, priceCents >= 0, stockQuantity >= 0
2. Service pre-checks SKU by exact lookup
3. INSERT/UPDATE products
4. Unique index rejects a duplicate SKU at commit
5. Order creation filters selected products by active = true
6. Product deletion is restricted once order_items reference the row
```

---

### Business Rule BR-12: User Identity, Role and Credential Storage

**Overview**:
Users are identified by a unique email, carry one of two roles (`ADMIN`, `OPERATOR`, default `OPERATOR`), and store only a bcrypt password hash.

**Detailed description**:

`User.email` is `String @unique @db.VarChar(255)` (`prisma/schema.prisma:27`), `role` is `UserRole @default(OPERATOR)` (line 30), and `passwordHash` is `String @db.VarChar(255)` (line 28). The migration creates `UNIQUE INDEX users_email_key`, an inline `ENUM('ADMIN','OPERATOR') NOT NULL DEFAULT 'OPERATOR'`, and a required `VARCHAR(255)` password hash (`migration.sql:5-11`). The app hashes with bcrypt cost 10 in `UserService.createUser` (`src/modules/users/user.service.ts:16,26`), and the seed uses the same cost (`prisma/seed.ts:191-192`). A bcrypt hash is 60 characters, so the 255-character column has ample headroom.

Email uniqueness is the identity anchor for login and registration. `AuthService.login` looks up by email and compares the hash (`src/modules/auth/auth.service.ts:32-39`); `UserService.createUser` pre-checks the email and relies on the unique index for the race case. The role enum is the only authorization attribute in the data model; there is no permission table, no team membership and no per-resource grant. `PublicUser` deliberately omits `passwordHash` (`src/modules/users/user.service.ts:7-14,42-51`), and the auth test asserts that the hash is not present in the registration response (`tests/auth.test.ts:15-16`).

The database does not enforce hash format, email normalization, or role assignment policy. The registration schema accepts a client-supplied `role`, including `ADMIN` (`src/modules/auth/auth.schemas.ts:7`), and the service persists it unchanged. This is an application-level privilege-escalation risk, but the data model enables it by treating role as a writable request field rather than a server-controlled attribute. Users cannot be deleted through the API, and a user referenced by orders or history is protected by `RESTRICT`; password rotation and email change have no endpoints.

**Rule workflow**:

```
1. Registration validates email, password length and role
2. Service checks email uniqueness
3. bcrypt hashes the password with cost 10
4. INSERT users (email, passwordHash, name, role)
5. Unique index enforces email uniqueness
6. Login reads by email and compares the bcrypt hash
7. Public projections omit passwordHash
```

---

### Business Rule BR-13: Customer Identity and Document Indexing

**Overview**:
Customers are identified by a unique email. The tax document is indexed for lookup but is neither unique nor normalized, so duplicate customer records can share a document.

**Detailed description**:

`Customer.email` is `String @unique @db.VarChar(255)` and `Customer.document` is `String @db.VarChar(20)` with a non-unique index (`prisma/schema.prisma:43,45,52`). The migration creates `customers_email_key` and `customers_document_idx` (`migration.sql:26-27`). `CustomerService.create` checks email uniqueness and throws `409 EMAIL_ALREADY_USED` before insert (`src/modules/customers/customer.service.ts:29-33`); update performs the same check while excluding the current row (`43-50`). There is no service check for document uniqueness.

The business consequence is that email is the operational identity while the document is a searchable attribute. Two records with the same CPF or CNPJ but different emails can coexist, which is useful for data-entry recovery but dangerous for deduplication, credit checks and future invoicing. The seed demonstrates two document formats: formatted CPF (`123.456.789-00`) and formatted CNPJ (`12.345.678/0001-90`) (`prisma/seed.ts:14,79`). Because punctuation is stored, `123.456.789-00` and `12345678900` are different strings, and the index treats them as distinct. The application schema allows 11 to 20 characters (`src/modules/customers/customer.schemas.ts:17`), so both formats fit.

The document index is also not used effectively for the current search. `CustomerRepository.buildWhere` uses `contains` on name, email and document (`src/modules/customers/customer.repository.ts:15-21`), which Prisma translates to a leading-wildcard `LIKE`; MySQL cannot use a B-tree index for that pattern. The index therefore supports exact/prefix lookups that the current code does not issue. Email remains the only hard uniqueness rule, and customer records have a single required address rather than an address book.

**Rule workflow**:

```
1. Create/update validates email, phone, document length and address shape
2. Service pre-checks email uniqueness
3. INSERT/UPDATE customers
4. Unique index rejects duplicate email
5. Document is stored as provided and indexed non-uniquely
6. Search uses LIKE '%term%' and cannot exploit the document index
```

---

### Business Rule BR-14: Stock Quantity and Movement

**Overview**:
`products.stockQuantity` is the current on-hand quantity. It is decremented on `PENDING -> PAID` and incremented when a paid or processing order is cancelled; the database has no non-negative check.

**Detailed description**:

`Product.stockQuantity` is a required `Int` with `@default(0)` (`prisma/schema.prisma:62`), materialized as `INTEGER NOT NULL DEFAULT 0` (`migration.sql:38`). The application treats it as a non-negative quantity: create and update validate `z.number().int().nonnegative()` (`src/modules/products/product.schemas.ts:9`). Stock is not reserved when an order is created in `PENDING`; it is only checked and debited on the `PENDING -> PAID` edge (`src/modules/orders/order.status.ts:24-31`). `debitStock` reads all affected products, collects insufficient rows, throws `InsufficientStockError` with SKU/requested/available details, and only then decrements each product (`src/modules/orders/order.service.ts:204-231`). `replenishStock` increments on cancellation from `PAID` or `PROCESSING` (`233-243`).

The transactional design ensures that the stock check, the decrements, the status update and the history append commit or roll back together. The tests verify the normal paths: a `PENDING -> PAID` transition reduces stock by the ordered quantity (`tests/orders.test.ts:59-87`), insufficient stock returns 422 and leaves stock unchanged (`109-132`), and `PAID -> CANCELLED` restores the quantity (`134-161`). The stock field is a current-state projection, not a ledger: there is no `stock_movement` table, no reason code and no record of who changed stock outside an order transition.

The missing database check is significant. There is no `CHECK (stockQuantity >= 0)` and no trigger. The service's pre-check uses a read followed by a conditional update, so two concurrent `PENDING -> PAID` transactions can both pass the check and both decrement, potentially producing a negative quantity. Direct SQL can also set a negative value. Because the column is signed `INT`, the database accepts it. The same pattern applies to `priceCents` and the money columns, which are likewise unconstrained at the database level.

**Rule workflow**:

```
1. Order is created in PENDING without touching stock
2. PENDING -> PAID: read products and compare stockQuantity to ordered quantity
3. If any product is insufficient -> 422, no updates
4. Otherwise decrement each product in the same transaction
5. PAID|PROCESSING -> CANCELLED: increment each product
6. No database check prevents a resulting negative value under concurrency
```

---

### Business Rule BR-15: Order to Customer and Creator Referential Actions

**Overview**:
`orders.customerId` and `orders.createdById` are non-null foreign keys with `ON DELETE RESTRICT` and `ON UPDATE CASCADE`.

**Detailed description**:

The relations are declared at `prisma/schema.prisma:87-88` and materialized at `migration.sql:110-113` as `orders_customerId_fkey` and `orders_createdById_fkey`. Both columns are `CHAR(36) NOT NULL`, matching the parent `id` type and collation. `ON UPDATE CASCADE` means that if a parent id were changed, MySQL would propagate the new value to the order; the application never updates ids, so this is a safety net rather than an active behavior. `ON DELETE RESTRICT` means an order protects its customer and creator from deletion.

The rule preserves the meaning of the order as a commercial and audit record. An order without a customer would be orphaned and unqueryable by customer; an order without a creator would lose accountability. The restriction is also the reason the customer delete endpoint is unsafe in practice: once any order exists, MySQL rejects the delete with error `P2003`, and the centralized error middleware handles only `P2002` and `P2025` (`src/middlewares/error.middleware.ts:37-54`). The user model has no delete endpoint, but the same restriction applies to any future deletion path.

The update side of the rule has no practical trigger. Because IDs are generated and never edited by any service, `ON UPDATE CASCADE` is dormant. It does, however, mean that a manual id change would silently rewrite order references, which could be desirable for a merge operation but dangerous if done accidentally. There is no `ON DELETE SET NULL` option because both columns are required; making a customer optional would require a schema change and a corresponding business decision about anonymous or walk-in orders.

**Rule workflow**:

```
1. Order insert supplies customerId and createdById
2. MySQL verifies both parent rows exist
3. Customer or user delete while referenced -> P2003, delete rejected
4. Parent id update -> child foreign keys cascade the new value
5. Application error middleware does not map P2003 -> HTTP 500
```

---

### Business Rule BR-16: OrderItem Referential Actions

**Overview**:
`order_items.orderId` cascades on delete, while `order_items.productId` restricts on delete. The order line is owned by the order but must always point at a real product.

**Detailed description**:

`OrderItem.order` declares `onDelete: Cascade` on `orderId` (`prisma/schema.prisma:108`), and `OrderItem.product` leaves the default required-relation behavior, which Prisma renders as `ON DELETE RESTRICT ON UPDATE CASCADE` (`migration.sql:116,119`). The order side implements aggregate ownership: deleting an order removes its lines automatically, so no orphan line can survive. The product side implements catalog integrity: a product cannot disappear while any historical line references it.

The cascade is the mechanism behind the order delete endpoint. `OrderRepository.deleteById` issues a single `DELETE FROM orders` (`src/modules/orders/order.repository.ts:66-68`), and InnoDB removes the child rows. The repository does not explicitly delete lines, which keeps the method simple but hides the destructive scope at the call site. The restriction on products means that a product used in even one historical order can never be hard-deleted through the API; the service's `delete` method will hit `P2003` and surface as HTTP 500. `Product.active` is the only supported retirement mechanism.

The two actions express a deliberate ownership hierarchy. Lines are part of the order aggregate, so their lifecycle is subordinate. Products are independent master data, so their lifecycle cannot be controlled by an order. The schema has no `ON DELETE SET NULL` for `productId`, and the column is required, so there is no "deleted product" placeholder. Any future product archival design would need either a soft-delete flag or a nullable product reference with snapshot data.

**Rule workflow**:

```
1. DELETE orders row
2. InnoDB cascades to order_items by orderId
3. DELETE products row with existing order_items -> P2003, rejected
4. Product remains available for historical order display
5. Service returns HTTP 500 because P2003 is not mapped
```

---

### Business Rule BR-17: OrderStatusHistory Referential Actions

**Overview**:
`order_status_history.orderId` cascades on delete, and `changedById` restricts deletion of the acting user. The audit row is owned by the order and attributed to a real user.

**Detailed description**:

`OrderStatusHistory.order` declares `onDelete: Cascade` (`prisma/schema.prisma:125`), and `changedBy` uses the default required-relation restriction (`migration.sql:122,125`). `orderId` and `changedById` are both `CHAR(36) NOT NULL`. The schema declares indexes on `orderId` and `changedAt` (lines 128-129), but not on `changedById`; MySQL creates an implicit index for the foreign-key column because InnoDB requires one. That implicit index is visible in the database metadata but not in the Prisma schema, an asymmetry worth noting for index audits.

The cascade ties audit durability to the order aggregate. Deleting an order deletes its entire status history, including the initial `PENDING` entry. That is consistent with a hard-delete model but means the audit trail is not immutable at the storage layer. The restriction on `changedById` prevents a user from being deleted while any history row names them; since there is no user delete endpoint, this is a latent constraint. The history row remains readable as long as the order exists and the user row exists.

The rule also defines the only attribution path: every history row must name a user. There is no system actor, no nullable `changedById`, and no external actor column. If a future automated process changes status, it must run as a real user row or the schema must change. The detail-level read/write behavior of this model is intentionally left to the sibling Order Status History analysis; this rule covers only its referential contract.

**Rule workflow**:

```
1. Status change appends history with orderId and changedById
2. MySQL verifies both parent rows
3. DELETE order -> history rows cascade
4. DELETE user with history rows -> P2003, rejected
5. changedById index exists implicitly for the foreign key
```

---

### Business Rule BR-18: Hard Delete and Absence of Soft Delete

**Overview**:
No table has a `deletedAt`, `deletedBy` or `isDeleted` column. Deletion is physical. The order service allows deletion only for `PENDING` or `CANCELLED` orders, and the database removes children by cascade.

**Detailed description**:

The schema contains only `active` on `Product` as a lifecycle flag; there is no soft-delete column on `User`, `Customer`, `Order`, `OrderItem` or `OrderStatusHistory`. `OrderService.delete` reads the order, rejects any status other than `PENDING` or `CANCELLED` (`src/modules/orders/order.service.ts:181-192`), and calls the repository's hard delete. The database then cascades to `order_items` and `order_status_history` (`migration.sql:116,122`). Customer and product deletes are hard deletes at the repository level (`src/modules/customers/customer.repository.ts:54-56`, `src/modules/products/product.repository.ts:59-61`) but are usually blocked by `RESTRICT`.

The business intent is simplicity: an order that was never paid or was cancelled is not a financial record worth preserving, so it can be removed. That intent is only partially enforced. The status guard is a read outside a transaction, so a concurrent `PENDING -> PAID` can commit before the delete, allowing a paid order to be removed. Deleting a cancelled order also removes its cancellation history, even though cancellation is a meaningful business event. Once deleted, the order number is never reused, so external references to `ORD-nnnnnn` become permanently dangling.

The absence of soft delete is a schema-level decision with broad consequences. There is no query filter to exclude deleted rows, no restoration path and no deletion audit. Any future requirement to retain financial records, support GDPR-style erasure with exceptions, or replay order events would require a schema change. The `Product.active` flag shows that the model can express retirement, but it does not generalize to orders or customers.

**Rule workflow**:

```
1. Read order status
2. If status not PENDING or CANCELLED -> 409
3. DELETE orders row
4. Cascade removes order_items and order_status_history
5. No tombstone, no deletion audit, no number reuse
6. Concurrent status change between steps 1 and 3 is not prevented
```

---

### Business Rule BR-19: Append-Only Status History Constraints

**Overview**:
`order_status_history` records one row per status transition with a nullable source status, a required target status, a required actor, an optional reason of up to 500 characters and a default `changedAt`.

**Detailed description**:

The model is declared at `prisma/schema.prisma:116-131`. `fromStatus` is `OrderStatus?` to represent creation, `toStatus` is required, `changedById` is required, `reason` is `String? @db.VarChar(500)`, and `changedAt` defaults to `now()`. The migration creates a nullable enum column, a required enum column, a required actor `CHAR(36)`, a nullable `VARCHAR(500)` and `DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)` (`migration.sql:87-99`). The service writes the initial row with `fromStatus: null`, `toStatus: PENDING` and reason `'order created'` (`src/modules/orders/order.service.ts:106-113`), then one row per transition with the actual source, target, actor and optional reason (`159-167`).

The append-only property is a convention, not a database guarantee. There is no trigger preventing `UPDATE` or `DELETE`, no permission separation for the application user, and no unique constraint on `(orderId, toStatus)`. The database accepts a row where `fromStatus = toStatus`, a null `fromStatus` at any point, an empty reason and a `changedAt` supplied by the caller. The cascade from `orders` is the one deletion path that is structurally guaranteed. The audit value therefore depends on all writers using the service and on no one editing rows directly.

The nullable source status is the only representation of creation because the enum has no `CREATED` member. That works for a single initial row but makes it impossible to distinguish "created" from "unknown previous status" if a row is inserted manually. Ordering is by `changedAt` ascending, and the column has millisecond precision; two changes in the same millisecond, or a manually backdated row, can produce a non-deterministic order because there is no sequence or tie-breaker column. The sibling analysis covers the read/write behavior in depth.

**Rule workflow**:

```
1. Creation inserts fromStatus=null, toStatus=PENDING, reason 'order created'
2. Each transition inserts fromStatus=current, toStatus=next, changedById, reason
3. changedAt defaults to the database current timestamp
4. No update or delete endpoint exists
5. Database accepts direct updates/deletes and duplicate transitions
6. Deleting the order cascades the whole history
```

---

### Business Rule BR-20: Timestamp Convention

**Overview**:
All entities use `DATETIME(3)` timestamps. `createdAt` has a database default; `updatedAt` is required and maintained by Prisma; no timezone is stored.

**Detailed description**:

`createdAt` is declared as `DateTime @default(now())` on `User`, `Customer`, `Product`, `Order` and `OrderItem`; `OrderStatusHistory` uses `changedAt` with the same default (`prisma/schema.prisma:31,47,64,84,106,121`). The migration maps these columns to `DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)` (`migration.sql:8,23,40,60,79,92`). `updatedAt` is declared as `DateTime @updatedAt` on `User`, `Customer`, `Product` and `Order` (lines 32,48,65,85); the DDL is `DATETIME(3) NOT NULL` with no database default or `ON UPDATE` clause (`migration.sql:9,24,41,61`). The value is therefore supplied by Prisma on insert and on every update. `OrderItem` and `OrderStatusHistory` intentionally have no `updatedAt`.

The convention gives millisecond precision and consistent column types across the schema. `createdAt` is database-authoritative, so an insert through raw SQL or another client still gets a timestamp. `updatedAt` is client-authoritative, so a raw SQL update that omits it leaves a stale value, and a direct insert must supply it or fail the `NOT NULL` constraint. Prisma updates `updatedAt` automatically for any model update, including a status-only change, which means `orders.updatedAt` reflects the last status change as well as any other edit.

No timezone offset is stored. MySQL `DATETIME` has no zone, and the application/Prisma convention treats values as UTC. This is invisible in the schema and depends on every client using the same convention. The `orders.createdAt` index supports date-range filtering and descending order; there is no index on `updatedAt`, so "recently changed" queries would scan. There is no `deletedAt`, so temporal queries cannot exclude deleted rows.

**Rule workflow**:

```
1. INSERT: createdAt defaults to CURRENT_TIMESTAMP(3); Prisma supplies updatedAt
2. UPDATE: Prisma sets updatedAt to the current time
3. Raw SQL INSERT must supply updatedAt
4. Raw SQL UPDATE that omits updatedAt leaves it stale
5. Values are DATETIME(3) with no timezone metadata
```

---

### Business Rule BR-21: Customer Address JSON Shape

**Overview**:
Each customer has exactly one required `address` JSON object. Its internal shape is defined by Zod, not by the database.

**Detailed description**:

`Customer.address` is `Json` and required (`prisma/schema.prisma:46`), materialized as `JSON NOT NULL` (`migration.sql:22`). The application schema requires `street` (1-200), `number` (1-20), `city` (1-100), `state` (exactly 2) and `zipCode` (5-15) (`src/modules/customers/customer.schemas.ts:5-11`). The seed supplies the same shape with Brazilian state abbreviations and formatted postal codes (`prisma/seed.ts:15-21`). MySQL validates only that the value is valid JSON; it does not validate keys, types or lengths inside the document.

The model expresses a single-address customer. There is no `addresses` table, no address type (billing/shipping), no validity period and no address history. Updating a customer replaces the whole JSON object. That is sufficient for a simple order-management demo but limits real fulfillment scenarios that need separate billing and shipping addresses or address changes over time. The JSON column also cannot be searched with ordinary B-tree indexes; any query on city or postal code would need a generated column or functional index, neither of which exists.

The most important business consequence is that orders do not snapshot the address. `Order` has no shipping-address column and no relation to an address record; an order only points at `customerId`. If the customer moves, every historical order now displays the new address. For a system that treats orders as commercial documents, this is a fidelity gap: the price is snapshotted (BR-06) but the destination is not. The schema permits a future address snapshot, but it does not provide one today.

**Rule workflow**:

```
1. Create/update validates the address object with Zod
2. Prisma serializes it into the JSON column
3. MySQL stores the document without shape validation
4. Reads return the current address for the customer
5. Historical orders resolve the current customer address, not the order-time address
```

---

### Business Rule BR-22: Character Set and Collation Semantics

**Overview**:
Every table is created with `DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`, which makes string comparisons case-insensitive and accent-insensitive.

**Detailed description**:

Each `CREATE TABLE` in the migration ends with `DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci` (`migration.sql:13,29,47,69,84,99,107`). The Docker Compose service also starts MySQL with `--character-set-server=utf8mb4` and `--collation-server=utf8mb4_unicode_ci` (`docker-compose.yml:28-29`), so new databases and tables inherit the same defaults. `utf8mb4` supports the full Unicode range, including emoji and characters outside the Basic Multilingual Plane, and uses up to four bytes per character.

The collation is a behavioral rule, not just a storage setting. `utf8mb4_unicode_ci` compares strings without regard to case and, in general, without regard to accents. That means `User@Example.com` and `user@example.com` collide on the unique email index, and accented variants can collide as well. SKU uniqueness and email uniqueness therefore have semantics that the API documentation does not state. Search predicates using `contains` also inherit the case-insensitive behavior, so a search for `mariana` matches `Mariana`. This is usually desirable for user-facing search but must be understood for identifiers.

The choice also affects index size. A unique `VARCHAR(255)` email under `utf8mb4` can occupy up to 1,020 bytes in the index, and a `VARCHAR(200)` product name up to 800 bytes. Both are within InnoDB's 3,072-byte limit on DYNAMIC row format, so the indexes are valid. There is no per-column collation override for identifiers that should be case-sensitive or binary. If exact byte-wise uniqueness were required, the schema would need `utf8mb4_bin` or a normalized lowercase column, neither of which exists.

**Rule workflow**:

```
1. Database and tables default to utf8mb4_unicode_ci
2. Unique email and SKU comparisons ignore case and accents
3. LIKE '%term%' search inherits case-insensitive matching
4. Index key sizes are calculated in utf8mb4 bytes
5. No column overrides the collation for binary-exact identity
```

---

### Business Rule BR-23: Schema-Migration Parity and Migration Strategy

**Overview**:
`prisma/schema.prisma` is the source of truth. The repository contains one baseline migration that matches the schema, and the workflow is development-oriented: `migrate dev`, a shadow database and a destructive reset/seed.

**Detailed description**:

The datasource block declares `provider = "mysql"`, `url = env("DATABASE_URL")` and `shadowDatabaseUrl = env("SHADOW_DATABASE_URL")` (`prisma/schema.prisma:5-9`). The migration lock file records `provider = "mysql"` (`prisma/migrations/migration_lock.toml:3`), preventing an accidental provider switch without an explicit reset. The single migration `20260519182739_init` creates all seven tables, indexes and foreign keys in an order that satisfies dependencies: parent tables first, then orders, then children, then `ALTER TABLE ... ADD CONSTRAINT` for foreign keys (`migration.sql:1-125`). A line-by-line comparison shows parity with the schema at the time of analysis.

The migration workflow is defined in `package.json`: `db:migrate` runs `prisma migrate dev`, `db:reset` runs `prisma migrate reset --force`, and `db:seed` runs the TypeScript seed with `.env`. `prisma migrate dev` uses the shadow database to diff the schema, generates a new timestamped SQL file, applies it, and records it in Prisma's `_prisma_migrations` table. `prisma migrate reset` drops the database, replays all migrations and runs the seed. There is no `prisma migrate deploy` script, no `prisma validate` script and no CI configuration in the repository, so the production migration path is undefined.

The strategy has two consequences. First, there is no history of incremental changes: any future schema edit creates a second migration whose SQL must be reviewed against existing data. Second, there are no down migrations; rollback means reset, which is destructive and unsuitable for production. The shadow database URL is required for `migrate dev` but is not validated by `src/config/env.ts`, and `docker-compose.yml` provisions only the `oms` database, not `oms_shadow`. The application runtime and the migration tooling share the same `DATABASE_URL` credentials, so there is no separation between a DML runtime user and a DDL migration user.

**Rule workflow**:

```
1. Edit prisma/schema.prisma
2. `prisma migrate dev` connects to DATABASE_URL and SHADOW_DATABASE_URL
3. Prisma diffs the schema and writes a timestamped migration SQL file
4. Prisma applies it and records it in _prisma_migrations
5. Prisma regenerates the client
6. `prisma migrate reset --force` drops, replays and seeds for a clean development state
```

---

### Business Rule BR-24: Naming Conventions

**Overview**:
Prisma models use singular PascalCase, database tables use snake_case plural through `@@map`, and columns retain camelCase names. The result is a mixed convention between table and column identifiers.

**Detailed description**:

Every model maps its table explicitly: `User -> users` (`prisma/schema.prisma:37`), `Customer -> customers` (53), `Product -> products` (71), `Order -> orders` (96), `OrderItem -> order_items` (113), `OrderStatusHistory -> order_status_history` (130) and `OrderNumberSequence -> order_number_sequence` (137). The migration uses those snake_case names consistently (`migration.sql:2,16,32,50,72,87,102`). Columns, however, are not mapped; they keep the Prisma field names such as `passwordHash`, `priceCents`, `orderNumber`, `customerId` and `changedById`. MySQL on Linux is case-sensitive for table names by default, so the exact casing matters.

The convention gives readable SQL table names and keeps TypeScript field names natural. It also means that raw SQL must backtick camelCase column names, as the migration does (for example `` `passwordHash` ``, `` `orderNumber` ``, `` `changedById` ``). A developer writing unquoted SQL may accidentally rely on case-insensitive column resolution, which varies by platform and configuration. The mixed convention is visible in every query and in the generated DDL; it is consistent, but it requires discipline.

There are no `@@map` annotations on individual columns and no snake_case naming strategy at the Prisma level. Renaming a field is therefore a breaking change to the database column as well as to the generated client type, and requires a migration. The convention does not extend to enums, which are stored inline in each column rather than as named database types, and it does not include a schema/namespace prefix.

**Rule workflow**:

```
1. Prisma model field names define column names
2. @@map defines the physical table name
3. Migration DDL uses snake_case tables and camelCase columns
4. Raw SQL must quote camelCase columns
5. Field renames require both a migration and client regeneration
```

---

### Business Rule BR-25: Global Uniqueness and Single-Tenancy

**Overview**:
Uniqueness is global per table: user email, customer email, product SKU and order number are unique across the entire database. There is no tenant, organization or workspace column.

**Detailed description**:

The four unique indexes are `users_email_key`, `customers_email_key`, `products_sku_key` and `orders_orderNumber_key` (`migration.sql:11,26,43,63`). None is composite and none includes a tenant discriminator. The same email may exist once as a user and once as a customer because those are separate tables and separate uniqueness domains; that is intentional for staff versus buyers. Within each table, however, there is exactly one row per value. The service pre-checks mirror this global scope by calling `findByEmail` or `findBySku` without any tenant filter.

The model therefore assumes a single-tenant deployment. All customers, products, users and orders share one catalog and one numbering space. That is appropriate for the current order-management application but constrains any future SaaS or multi-brand scenario. Adding multi-tenancy would require a `tenantId` on every aggregate and composite unique keys such as `(tenantId, email)` and `(tenantId, sku)`, plus tenant-aware foreign keys and query filters. None of that exists, and a naive addition would break the current unique indexes.

The global scope also has a data-quality benefit: it prevents accidental duplicate SKUs and duplicate login emails across the whole installation. The cost is that two business units cannot reuse the same SKU or email even if they are logically separate. The order-number sequence is similarly global, so all orders share one numbering space regardless of customer or channel.

**Rule workflow**:

```
1. Insert checks a non-composite unique index
2. Service pre-checks use global findUnique by email or SKU
3. No tenant column participates in any key or query
4. Two tenants cannot share an email or SKU
5. Multi-tenancy would require new composite keys and migrations
```

---

### Business Rule BR-26: Free-text Bounds and Nullability

**Overview**:
Optional free-text fields have different limits at the application and database layers: product description and order notes are `TEXT`, while the history reason is `VARCHAR(500)`. Empty strings are allowed by the database.

**Detailed description**:

`Product.description` is `String? @db.Text`, `Order.notes` is `String? @db.Text`, and `OrderStatusHistory.reason` is `String? @db.VarChar(500)` (`prisma/schema.prisma:60,82,123`). The migration creates `TEXT NULL` for the first two and `VARCHAR(500) NULL` for the reason (`migration.sql:36,58,94`). The application limits are stricter for the `TEXT` fields: product description is capped at 2,000 characters (`src/modules/products/product.schemas.ts:6`) and order notes at 1,000 characters (`src/modules/orders/order.schemas.ts:15`). The reason limit is 500 in both layers (`src/modules/orders/order.schemas.ts:20`).

The database limits are byte-oriented for `TEXT` and character-oriented for `VARCHAR`. A `TEXT` column holds up to 65,535 bytes, which under `utf8mb4` is at most about 16,383 four-byte characters; the application's 1,000/2,000 character caps are far below that. The `reason` column enforces 500 characters in both layers, so it is the only free-text field with a matching boundary. There is no database check that a string is non-empty or trimmed; whitespace-only values are accepted if they pass Zod's minimum rules (and the optional fields have no minimum).

Nullability is meaningful. Missing description, notes or reason are stored as SQL `NULL`, not as empty strings, and the services pass `null` explicitly for absent description and notes (`src/modules/products/product.service.ts:38`, `src/modules/orders/order.service.ts:103`). There is no sanitization or HTML escaping at the data layer; that is an API/output concern. The schema also has no full-text index on any of these columns, so free-text search over them is not supported by an index.

**Rule workflow**:

```
1. Zod validates max lengths at the HTTP boundary
2. Optional fields may be omitted -> service stores null
3. TEXT columns accept up to 65,535 bytes; VARCHAR(500) enforces 500 characters
4. Database accepts empty or whitespace-only strings
5. No full-text index exists for content search
```

---

## 4. Component Structure

### 4.1 File structure

```
prisma/
|-- schema.prisma                              # Declarative source of truth: datasource, generator, 2 enums, 7 models
|-- seed.ts                                    # Destructive development seed (analyzed as a separate component)
`-- migrations/
    |-- migration_lock.toml                    # Pins provider = "mysql"
    `-- 20260519182739_init/
        `-- migration.sql                      # Single baseline DDL: 7 tables, 2 enums, 6 FKs, 15 secondary indexes
```

The analyzed boundary is `prisma/schema.prisma` plus `prisma/migrations/`. `prisma/seed.ts` is listed only as a consumer of the model; its behavior belongs to the Seed Script component analysis.

### 4.2 Entity inventory

| Prisma model | Table | Primary key | Unique keys | Foreign keys out | Declared indexes | Child collections |
|--------------|-------|-------------|-------------|------------------|------------------|-------------------|
| `User` | `users` | `id CHAR(36)` | `email` | none | PK, unique email | `createdOrders`, `statusHistoryChanges` |
| `Customer` | `customers` | `id CHAR(36)` | `email` | none | PK, unique email, `document` | `orders` |
| `Product` | `products` | `id CHAR(36)` | `sku` | none | PK, unique sku, `active`, `name` | `items` |
| `Order` | `orders` | `id CHAR(36)` | `orderNumber` | `customerId -> customers`, `createdById -> users` | PK, unique orderNumber, `customerId`, `status`, `createdAt`, `createdById` | `items`, `history` |
| `OrderItem` | `order_items` | `id CHAR(36)` | none | `orderId -> orders` (CASCADE), `productId -> products` (RESTRICT) | PK, `orderId`, `productId` | none |
| `OrderStatusHistory` | `order_status_history` | `id CHAR(36)` | none | `orderId -> orders` (CASCADE), `changedById -> users` (RESTRICT) | PK, `orderId`, `changedAt`; implicit `changedById` | none |
| `OrderNumberSequence` | `order_number_sequence` | `id INT DEFAULT 1` | none | none | PK | none |

### 4.3 Prisma-to-MySQL type mapping

| Prisma declaration | MySQL type | Notes |
|--------------------|------------|-------|
| `String @id @default(uuid()) @db.Char(36)` | `CHAR(36) NOT NULL` | UUID generated client-side; no DB default |
| `String @db.VarChar(n)` | `VARCHAR(n)` | utf8mb4 storage |
| `String? @db.Text` | `TEXT NULL` | up to 65,535 bytes |
| `Int` | `INTEGER` | signed 32-bit |
| `Boolean @default(true)` | `BOOLEAN NOT NULL DEFAULT true` | alias for `TINYINT(1)` |
| `DateTime @default(now())` | `DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3)` | millisecond precision, no timezone |
| `DateTime @updatedAt` | `DATETIME(3) NOT NULL` | maintained by Prisma, no DB default |
| `Json` | `JSON NOT NULL` | no database shape validation |
| enum field | inline `ENUM(...)` | native MySQL enum per column |
| `Int @id @default(1)` | `INTEGER NOT NULL DEFAULT 1` | single-row sequence table |

### 4.4 Relation matrix

| From | To | Cardinality | FK column | onDelete | onUpdate | Optional |
|------|----|-------------|-----------|----------|----------|----------|
| `Order` | `Customer` | N:1 | `customerId` | RESTRICT | CASCADE | no |
| `Order` | `User` (`createdBy`) | N:1 | `createdById` | RESTRICT | CASCADE | no |
| `OrderItem` | `Order` | N:1 | `orderId` | CASCADE | CASCADE | no |
| `OrderItem` | `Product` | N:1 | `productId` | RESTRICT | CASCADE | no |
| `OrderStatusHistory` | `Order` | N:1 | `orderId` | CASCADE | CASCADE | no |
| `OrderStatusHistory` | `User` (`changedBy`) | N:1 | `changedById` | RESTRICT | CASCADE | no |

### 4.5 Index inventory and effectiveness

| Table | Index | Columns | Type | Intended access path | Assessment |
|-------|-------|---------|------|---------------------|------------|
| `users` | `users_email_key` | `email` | unique | login, registration, user lookup | Effective; case/accent-insensitive |
| `customers` | `customers_email_key` | `email` | unique | create/update uniqueness | Effective |
| `customers` | `customers_document_idx` | `document` | non-unique | document search | Not usable for leading-wildcard `contains`; no exact lookup exists |
| `products` | `products_sku_key` | `sku` | unique | create/update, lookup | Effective |
| `products` | `products_active_idx` | `active` | non-unique | list filter | Low selectivity (two values) |
| `products` | `products_name_idx` | `name` | non-unique | name search | Not usable for leading-wildcard `contains` |
| `orders` | `orders_orderNumber_key` | `orderNumber` | unique | business identifier | Effective |
| `orders` | `orders_customerId_idx` | `customerId` | non-unique | list filter, FK support | Effective |
| `orders` | `orders_status_idx` | `status` | non-unique | list filter | Low selectivity alone |
| `orders` | `orders_createdAt_idx` | `createdAt` | non-unique | date range, `ORDER BY createdAt` | Effective |
| `orders` | `orders_createdById_idx` | `createdById` | non-unique | FK support | No query filters by creator |
| `order_items` | `order_items_orderId_idx` | `orderId` | non-unique | relation include, FK support | Effective |
| `order_items` | `order_items_productId_idx` | `productId` | non-unique | FK support | Effective |
| `order_status_history` | `order_status_history_orderId_idx` | `orderId` | non-unique | relation include, FK support | Effective |
| `order_status_history` | `order_status_history_changedAt_idx` | `changedAt` | non-unique | ordering | Partial; best path is `(orderId, changedAt)` |
| `order_status_history` | implicit FK index | `changedById` | implicit | FK support | Exists in MySQL but not declared in Prisma |
| `order_number_sequence` | PRIMARY | `id` | primary | upsert by id=1 | Effective |

---

## 5. Dependency Analysis

### 5.1 Internal dependency chains

The schema has no import statements and therefore no compile-time internal dependencies. Its influence is indirect through the generated Prisma client, which is imported across the application.

```
prisma/schema.prisma
  -> prisma generate
  -> @prisma/client (generated model types + enum values)
    -> src/config/database.ts (PrismaClient construction)
      -> src/app.ts (injected into every repository and OrderService)
        -> src/modules/users/user.repository.ts + user.service.ts
        -> src/modules/auth/auth.service.ts
        -> src/modules/customers/customer.repository.ts + customer.service.ts
        -> src/modules/products/product.repository.ts + product.service.ts
        -> src/modules/orders/order.repository.ts + order.service.ts + order.schemas.ts + order.status.ts
        -> src/middlewares/error.middleware.ts (Prisma error classes)
        -> src/server.ts (prisma.$disconnect)
    -> prisma/seed.ts (direct PrismaClient for all models)
    -> tests/setup.ts + tests/helpers/factories.ts + tests/orders.test.ts
```

### 5.2 Consumers and what they use

| Consumer | File | Prisma surface used | Coupling |
|----------|------|---------------------|----------|
| Prisma client provider | `src/config/database.ts:1-10` | `PrismaClient` constructor, log configuration | Hard runtime |
| Composition root | `src/app.ts:2,23,26-53` | `PrismaClient` type, injected into repositories/services | Hard compile-time |
| Server bootstrap | `src/server.ts:16` | `$disconnect` | Hard runtime |
| Error middleware | `src/middlewares/error.middleware.ts:2,37-54` | `Prisma.PrismaClientKnownRequestError`, codes `P2002` and `P2025` | Hard compile-time |
| User repository | `src/modules/users/user.repository.ts:1,6-16` | `User` model, `user.findUnique`, `user.create` | Hard runtime |
| User service | `src/modules/users/user.service.ts:2,42-51` | `User` type, role projection | Hard compile-time |
| Auth service | `src/modules/auth/auth.service.ts:32-49` | `User` returned by repository; role used in JWT | Indirect |
| Customer repository | `src/modules/customers/customer.repository.ts:1,12-56` | `Customer`, `Prisma.CustomerWhereInput`, CRUD | Hard runtime |
| Customer service | `src/modules/customers/customer.service.ts:1,23-63` | `Customer` type, email uniqueness flow | Hard compile-time |
| Product repository | `src/modules/products/product.repository.ts:1,13-61` | `Product`, `Prisma.ProductWhereInput`, CRUD | Hard runtime |
| Product service | `src/modules/products/product.service.ts:1,24-66` | `Product` type, SKU uniqueness flow | Hard compile-time |
| Order repository | `src/modules/orders/order.repository.ts:1,21-68` | `Order`, `OrderItem`, `OrderStatus`, `OrderStatusHistory`, query input types, includes | Hard runtime |
| Order service | `src/modules/orders/order.service.ts:1-6,58-254` | `Order`, `OrderStatus`, `Prisma.TransactionClient`, direct reads/writes of six tables | Hard runtime |
| Order schemas | `src/modules/orders/order.schemas.ts:2,19,26` | `OrderStatus` enum for Zod native enum | Hard compile-time |
| Order state machine | `src/modules/orders/order.status.ts:1-14` | `OrderStatus` enum keys and values | Hard compile-time |
| Seed | `prisma/seed.ts:1-335` | All models, enums, sequence, deleteMany, transactions | Hard runtime, destructive |
| Test setup | `tests/setup.ts:1-20` | `prisma` singleton, deleteMany on all tables | Hard runtime, destructive |
| Test factories | `tests/helpers/factories.ts:1-83` | `User`, `Customer`, `Product` creation | Hard runtime |
| Order tests | `tests/orders.test.ts:1-219` | `prisma.product.findUnique` for stock assertions | Hard runtime |

### 5.3 External dependencies

| Dependency | Version | Type | Purpose | Coupling |
|------------|---------|------|---------|----------|
| MySQL | 8.0 (`mysql:8.0` image) | Runtime data store | Tables, native enums, JSON, foreign keys, collation, transactions, row locks | Hard; network over TCP |
| `@prisma/client` | 5.22.0 (`package.json:26`) | Runtime ORM | Generated delegates, query builder, transaction API, error classes | Hard; compile-time and runtime |
| `prisma` CLI | 5.22.0 (`package.json:45`) | Development tool | `generate`, `migrate dev`, `migrate reset`, shadow-database diffing | Hard for schema workflow |
| Shadow MySQL database | `SHADOW_DATABASE_URL` | Development infrastructure | Schema diffing and migration generation | Required by `migrate dev` |
| `bcrypt` | 5.1.1 (`package.json:27`) | Seed and runtime | Password hashing for user rows | Runtime auth plus seed |
| Docker Compose MySQL | `docker-compose.yml:1-33` | Local infrastructure | Provisions the `oms` database and credentials | Development only |
| `DATABASE_URL` | `.env.example:5` | Configuration | Runtime and migration connection string | Hard runtime |
| `SHADOW_DATABASE_URL` | `.env.example:6` | Configuration | Shadow database for migration diffing | Hard for migration workflow |
| `uuid` | 11.0.3 (`package.json:32`) | Runtime support | Request-id generation; model ids are generated by Prisma itself | Soft for this component |

### 5.4 Configuration dependencies

The schema itself depends on two environment variables and no feature flags. `DATABASE_URL` is validated as a non-empty string by `src/config/env.ts:8`; `SHADOW_DATABASE_URL` is read directly by Prisma and is absent from that schema, so a missing value fails only when a migration command runs. The schema has no `binaryTargets`, no preview features and no `relationMode` override, so Prisma uses the default foreign-key mode and the native query engine.

---

## 6. Afferent and Efferent Coupling

### 6.1 Component-level coupling

Coupling is measured statically. Afferent coupling (Ca) counts distinct files outside the component that import the generated Prisma client or the component's artifacts. Efferent coupling (Ce) counts distinct internal files the component imports; the schema imports none. External packages are reported separately.

| Component | Afferent (Ca) | Efferent internal (Ce) | External dependencies | Instability I = Ce/(Ca+Ce) | Criticality |
|-----------|---------------|------------------------|-----------------------|----------------------------|-------------|
| `prisma/schema.prisma` | 18 files (14 in `src`, 1 in `prisma`, 3 in `tests`) | 0 | `@prisma/client`, `prisma` CLI, MySQL | 0.00 | Critical |
| `prisma/migrations/20260519182739_init/migration.sql` | 0 compile-time; applied by CLI | 0 | MySQL, `prisma` CLI | 0.00 | Critical (state) |
| `prisma/migrations/migration_lock.toml` | 0 | 0 | `prisma` CLI | 0.00 | Medium |

The 14 `src` files importing `@prisma/client` are: `config/database.ts`, `app.ts`, `server.ts`, `middlewares/error.middleware.ts`, `modules/users/user.repository.ts`, `modules/users/user.service.ts`, `modules/customers/customer.repository.ts`, `modules/customers/customer.service.ts`, `modules/products/product.repository.ts`, `modules/products/product.service.ts`, `modules/orders/order.repository.ts`, `modules/orders/order.service.ts`, `modules/orders/order.schemas.ts` and `modules/orders/order.status.ts`. The non-source consumers are `prisma/seed.ts`, `tests/setup.ts`, `tests/helpers/factories.ts` and `tests/orders.test.ts`.

The schema is maximally stable by the instability metric: many dependents, no internal dependencies. That stability is also its risk. Any schema change propagates to the generated client and therefore to all 18 consumers at once. There is no anti-corruption layer or domain-specific mapping that would contain the change; the generated types are used directly in repositories, services, schemas and tests.

### 6.2 Model-level relational coupling

At the data-model level, afferent coupling is the number of other models with a foreign key pointing at the model; efferent coupling is the number of models the model points at.

| Model | Afferent models | Efferent models | Incoming FKs | Outgoing FKs | Instability | Criticality |
|-------|-----------------|-----------------|--------------|--------------|-------------|-------------|
| `User` | 2 (`Order`, `OrderStatusHistory`) | 0 | `orders.createdById`, `order_status_history.changedById` | none | 0.00 | High |
| `Customer` | 1 (`Order`) | 0 | `orders.customerId` | none | 0.00 | High |
| `Product` | 1 (`OrderItem`) | 0 | `order_items.productId` | none | 0.00 | High |
| `Order` | 2 (`OrderItem`, `OrderStatusHistory`) | 2 (`Customer`, `User`) | `order_items.orderId`, `order_status_history.orderId` | `customerId`, `createdById` | 0.50 | Critical |
| `OrderItem` | 0 | 2 (`Order`, `Product`) | none | `orderId`, `productId` | 1.00 | Medium |
| `OrderStatusHistory` | 0 | 2 (`Order`, `User`) | none | `orderId`, `changedById` | 1.00 | Medium |
| `OrderNumberSequence` | 0 | 0 | none | none | 0.00 | High (contention) |

`Order` is the relational center: two incoming and two outgoing references, plus two child collections. `User` is the most referenced parent after `Order`, which is why its delete restrictions affect both orders and audit history. `OrderNumberSequence` has no relational coupling at all but is operationally critical because every order creation serializes on its single row. `OrderItem` and `OrderStatusHistory` are unstable leaves: they depend on two parents each and nothing depends on them.

### 6.3 Change propagation

| Change | Directly affected | Propagation |
|--------|-------------------|-------------|
| Add or rename a model field | Generated Prisma types | All 18 consumers; compile errors where the field is used |
| Change an enum value | Generated enum + MySQL column | `order.status.ts`, `order.schemas.ts`, services, seed, tests, any stored data |
| Change a foreign-key action | Migration DDL | Delete behavior in services and tests; no compile-time signal |
| Add a unique constraint | Migration DDL | Runtime `P2002` paths; service pre-checks become redundant but still needed for messaging |
| Change a column type or length | Migration DDL + generated types | Repositories, services, Zod schemas, seed, raw SQL |
| Add an index | Migration DDL only | No type change; query plans and write cost change silently |

Note: the standard Endpoints section is intentionally omitted because this component exposes no REST, GraphQL, gRPC or other endpoint surface. Its consumers expose endpoints; the data model itself is a contract and a migration artifact.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 | External stateful data store | Persists all seven tables; enforces unique keys, foreign keys, enums, JSON and collation | MySQL wire protocol over TCP via Prisma query engine (`DATABASE_URL`) | Relational rows; `CHAR(36)`, `VARCHAR`, `TEXT`, `INTEGER`, `DATETIME(3)`, `JSON`, `ENUM` | No repository-level catch; `P2002 -> 409`, `P2025 -> 404`; other Prisma errors including `P2003` fall through to 500 (`src/middlewares/error.middleware.ts:37-64`) |
| Prisma Client 5.22.0 | Runtime ORM | Generates typed delegates; translates queries; manages transactions and relation loading | In-process library plus native query engine | Generated TypeScript types mapped to SQL | Client throws `PrismaClientKnownRequestError`; only two codes are mapped |
| Prisma CLI 5.22.0 | Migration and code-generation tool | Creates/applies migrations, regenerates client, resets database | CLI process, reads `schema.prisma` and environment | SQL migration files and generated client artifacts | Command exits non-zero; no CI wrapper or automated validation in the repository |
| Shadow MySQL database | Migration infrastructure | Schema diffing for `prisma migrate dev` | MySQL wire protocol (`SHADOW_DATABASE_URL`) | Same schema as target database | Required but not validated by `src/config/env.ts`; not provisioned by `docker-compose.yml` |
| `_prisma_migrations` table | Prisma migration metadata | Records applied migration names, checksums and timestamps | Managed by Prisma CLI inside MySQL | Internal Prisma table | Prisma reports drift or failed migrations; no application access |
| Docker Compose MySQL service | Local infrastructure | Starts MySQL 8.0 with `oms` database and `utf8mb4` defaults | Container network and published port 3306 | MySQL data directory on named volume `oms_mysql_data` | Healthcheck uses `mysqladmin ping`; no backup, replication or PITR |
| Seed script | Development data bootstrap | Deletes all rows and inserts demo users, customers, products, orders, items and history | In-process `PrismaClient` | Same models and enums as runtime | Throws on any error; process exits 1; destructive by design |
| Test harness | Integration test data control | Connects, truncates all tables before each test, disconnects after suite | In-process `PrismaClient` against `DATABASE_URL` | Same models; factories create users, customers and products | No environment guard; a misconfigured URL truncates a real database |
| Environment configuration | Runtime and migration inputs | Supplies `DATABASE_URL` and `SHADOW_DATABASE_URL` | Process environment / `.env` | URL strings | `DATABASE_URL` fail-fast via Zod; shadow URL unchecked |
| Planned webhook outbox | Not implemented | Future at-least-once event publication tied to status changes | Would use MySQL transaction and worker polling | Would require new tables (outbox, DLQ) | Not present in schema; no integration point exists today |

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Declarative schema / migration-as-code | `schema.prisma` plus generated SQL migration | `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql` | Versioned, reviewable database definition |
| Data Mapper / generated gateway | Prisma Client maps rows to typed models; no hand-written SQL in repositories | `src/config/database.ts`, all `*.repository.ts` | Type-safe persistence without an active-record base class |
| Repository pattern (consumers) | `UserRepository`, `CustomerRepository`, `ProductRepository`, `OrderRepository` wrap Prisma queries | `src/modules/*/*.repository.ts` | Isolate query construction from services |
| Unit of Work | `prisma.$transaction(async (tx) => ...)` for order creation and status change; array transaction for list plus count | `src/modules/orders/order.service.ts:58-123,131-178`; `order.repository.ts:35-43` | Atomic multi-table writes and consistent pagination counts |
| Surrogate key | `uuid()` `CHAR(36)` primary keys on all aggregate and child rows | `prisma/schema.prisma` | Stable, opaque identity independent of mutable business keys |
| Natural/business key | Unique `email`, `sku`, `orderNumber` alongside surrogate PKs | `prisma/schema.prisma` | Human and integration-friendly lookup without using business data as the PK |
| Single-row sequence | `order_number_sequence` with `id = 1` and `nextValue` | `prisma/schema.prisma:133-138` | Gap-free order numbering serialized by a row lock |
| Snapshot pricing | `order_items.unitPriceCents` and `totalCents` copied at creation | `prisma/schema.prisma:104-105`; `order.service.ts:75-84` | Preserve agreed commercial terms |
| Append-only audit log (convention) | `order_status_history` rows written per transition, never updated by the service | `prisma/schema.prisma:116-131`; `order.service.ts:106-113,159-167` | Trace who changed status, when and why |
| Enum-backed state domain | Native MySQL enum plus generated TypeScript enum | `prisma/schema.prisma:16-23`; `order.status.ts` | Restrict status membership and share values across layers |
| State machine | Transition table with predicates | `src/modules/orders/order.status.ts:3-37` | Enforce legal lifecycle edges in application code |
| Referential actions | `ON DELETE CASCADE` for order children, `RESTRICT` for master data | `migration.sql:110-125` | Encode ownership and preservation policy in the database |
| Soft delete (absent) | No `deletedAt`; only `Product.active` | `prisma/schema.prisma` | Physical deletion model with product retirement flag |
| Naming map | `@@map` to snake_case plural tables; camelCase columns | `prisma/schema.prisma:37,53,71,96,113,130,137` | Readable SQL table names with natural TypeScript fields |
| Semi-structured column | `Customer.address` as JSON | `prisma/schema.prisma:46` | Flexible address storage without a separate table |
| Shadow database | `SHADOW_DATABASE_URL` for migration diffing | `prisma/schema.prisma:8`; `.env.example:6` | Safe schema comparison during development |
| Destructive seed | Delete-all then insert demo data | `prisma/seed.ts:178-325` | Reproducible local dataset |
| Shared database / single tenancy | One schema, global unique keys, no tenant column | `prisma/schema.prisma` | Simple deployment model for one organization |
| Anemic domain model | Prisma records are data bags; invariants live in services | `prisma/schema.prisma`; `src/modules/*/*.service.ts` | Keeps ORM models simple but leaves invariants outside the data layer |

Architectural decisions visible in the component:

- MySQL 8.0 is the only data store; there is no polyglot persistence, cache or search engine.
- The database is a shared integration point across all five feature modules, not a per-module schema.
- The model uses database-enforced uniqueness and referential actions, but application-enforced value ranges, arithmetic and workflow.
- The schema is optimized for normalized transactional writes, not for analytical queries, full-text search or event sourcing.
- There is no outbox, event, queue, dead-letter or webhook table; the planned notification feature has no persistence footprint in the current schema.

---

## 10. Technical Debt & Risks

| ID | Risk Level | Component Area | Issue | Impact | Evidence |
|----|------------|----------------|-------|--------|----------|
| R-01 | High | `orders` + `order_status_history` | No version column, row lock or unique transition constraint; status read and update are separated | Two concurrent `PENDING -> PAID` requests can both pass checks, double-decrement stock and write duplicate history | `prisma/schema.prisma:74-97,116-131`; `src/modules/orders/order.service.ts:131-167` |
| R-02 | High | Foreign keys + error middleware | `ON DELETE RESTRICT` on customer, product and user references, but `P2003` is not translated | Customer/product delete endpoints return HTTP 500 instead of a meaningful conflict; product deletion is effectively impossible after first order | `migration.sql:110-125`; `src/middlewares/error.middleware.ts:37-54`; `src/modules/customers/customer.repository.ts:54-56`; `src/modules/products/product.repository.ts:59-61` |
| R-03 | High | All numeric columns | No `CHECK` constraints for non-negative money, stock or quantity, or for total arithmetic | Direct writes and concurrent paths can persist negative stock, negative prices or inconsistent totals | `migration.sql:37-38,55-57,77-78`; no `CHECK` anywhere in the file |
| R-04 | High | `order_number_sequence` | All order creations contend on one row lock | Order-creation throughput is serialized per database; lock waits grow with concurrency | `prisma/schema.prisma:133-138`; `src/modules/orders/order.service.ts:245-254` |
| R-05 | High | Test harness | `beforeEach` truncates all tables using the configured `DATABASE_URL` with no environment guard | Running tests against a non-test database destroys data | `tests/setup.ts:8-16` |
| R-06 | High | Order deletion | Hard delete cascades `order_items` and `order_status_history`; the status guard is outside a transaction | A paid order can be deleted in a race; cancellation history is lost; no audit of deletion | `src/modules/orders/order.service.ts:181-192`; `migration.sql:116,122` |
| R-07 | Medium | Customer address | Orders do not snapshot the address; customer has only one JSON address | Editing a customer changes the shipping destination shown for all historical orders | `prisma/schema.prisma:46,74-97`; `src/modules/orders/order.repository.ts:50-58` |
| R-08 | Medium | Product master data | Order items snapshot price but not product name or SKU; reads include current product fields | Renaming a product or changing its SKU retroactively changes historical order presentation | `prisma/schema.prisma:104-105`; `src/modules/orders/order.repository.ts:50-55` |
| R-09 | Medium | `customers.document` | Document is indexed but not unique or normalized | Duplicate customer records can share a CPF/CNPJ; formatting variants defeat matching | `prisma/schema.prisma:45,52`; `migration.sql:21,27`; `prisma/seed.ts:14,79` |
| R-10 | Medium | Order indexes | No composite indexes for `(status, createdAt)` or `(customerId, createdAt)` | Filtered lists may sort large candidate sets; performance degrades with volume | `prisma/schema.prisma:92-95`; `src/modules/orders/order.repository.ts:21-43` |
| R-11 | Medium | Search indexes | `products_name_idx` and `customers_document_idx` cannot serve leading-wildcard `contains` predicates | Search scans the table despite the presence of indexes; index maintenance cost has no matching benefit | `src/modules/customers/customer.repository.ts:15-21`; `src/modules/products/product.repository.ts:16-21` |
| R-12 | Medium | Migration governance | Only `migrate dev` and destructive reset scripts exist; no `migrate deploy`, validation or CI check | Production migration path is undefined; schema drift can reach deployment unnoticed | `package.json:11-13`; no CI configuration in the repository |
| R-13 | Medium | Shadow database | `SHADOW_DATABASE_URL` is not validated and `oms_shadow` is not provisioned by Compose | `prisma migrate dev` fails on a fresh machine until the shadow database is created manually | `prisma/schema.prisma:8`; `.env.example:6`; `docker-compose.yml:1-33`; `src/config/env.ts:3-11` |
| R-14 | Medium | Native enums | Enum changes require `ALTER TABLE` and coordinated client regeneration | Adding/removing a status or role couples schema migration to application deployment and stored data | `migration.sql:7,54,90-91`; `prisma/schema.prisma:11-23` |
| R-15 | Medium | Order creation | No idempotency key or unique request reference | Duplicate client submissions create duplicate orders and consume two order numbers | `prisma/schema.prisma:74-97`; `src/modules/orders/order.service.ts:50-124` |
| R-16 | Medium | UUID keys | Random UUID v4 stored as `CHAR(36)` | Larger indexes and scattered inserts; page splits at scale; non-sequential B-tree locality | `prisma/schema.prisma:26,41,57,75,100,117`; `migration.sql:3,17,33,51,73,88` |
| R-17 | Medium | Tenancy | No tenant/organization column; all unique keys are global | Cannot isolate data per business unit without schema-wide change | `prisma/schema.prisma`; `migration.sql:11,26,43,63` |
| R-18 | Medium | `Customer.address` | JSON shape enforced only by Zod, not by MySQL | Direct writes can store arbitrary JSON; queries on address fields need functional indexes | `prisma/schema.prisma:46`; `migration.sql:22`; `src/modules/customers/customer.schemas.ts:5-11` |
| R-19 | Medium | Collation | `utf8mb4_unicode_ci` makes unique email/SKU comparisons case- and accent-insensitive | Unexpected collisions for values considered distinct by clients; behavior undocumented in API contracts | `migration.sql:13,29,47,69,84,99,107` |
| R-20 | Medium | Test coverage | No migration test, no constraint tests, no customer/product module tests, no coverage instrumentation | Schema regressions and foreign-key/unique failures can pass the suite | `vitest.config.ts:1-18`; `tests/` contains only `auth.test.ts` and `orders.test.ts` |
| R-21 | Medium | Credentials | Runtime and migration use the same `DATABASE_URL`; shadow uses root credentials in the example | Application process has DDL-capable credentials; a compromised app can alter or drop schema objects | `.env.example:5-6`; `prisma/schema.prisma:5-9` |
| R-22 | Medium | Registration and role | `role` is writable through public registration and persisted unchanged | A caller can self-register as `ADMIN`; the schema offers no server-only role column or guard | `src/modules/auth/auth.schemas.ts:7`; `src/modules/auth/auth.service.ts:27-29`; `src/modules/users/user.service.ts:27-32` |
| R-23 | Medium | Customer identity | Customer email can be updated with no history; email is the uniqueness anchor | Changing an email silently changes identity and communication routing; no audit row | `src/modules/customers/customer.service.ts:43-58`; `prisma/schema.prisma:43` |
| R-24 | Low | Numeric bounds | All money and stock columns are signed `INT` with no currency or overflow guard | Maximum value is about 2.1 billion cents per column; large line totals can fail at insert | `migration.sql:37-38,55-57,77-78` |
| R-25 | Low | Naming | Tables are snake_case while columns are camelCase | Raw SQL must quote camelCase identifiers; portability and readability cost | `migration.sql:3-9,52-61`; `prisma/schema.prisma:26-32,76-85` |
| R-26 | Low | History index | `changedById` has only the implicit MySQL FK index, not a declared Prisma index | Index inventory from the schema is incomplete; migrations may surprise reviewers | `prisma/schema.prisma:116-131`; `migration.sql:125` |
| R-27 | Low | History ordering | Ordering by `changedAt` alone with millisecond precision and no tie-breaker | Two changes in the same millisecond can be ordered non-deterministically | `prisma/schema.prisma:121`; `src/modules/orders/order.repository.ts:56` |
| R-28 | Low | Sequence constraints | Sequence table is not constrained to one row and `nextValue` has no lower bound | A direct writer can create additional counters or reset numbering | `prisma/schema.prisma:133-138`; `migration.sql:102-107` |
| R-29 | Low | `updatedAt` | Maintained by Prisma, not by a database trigger or default | Raw SQL updates leave stale timestamps; direct inserts must supply a value | `prisma/schema.prisma:32,48,65,85`; `migration.sql:9,24,41,61` |
| R-30 | Low | ID defaults | UUID primary keys have no database default | Raw SQL inserts must generate and supply the id; no database-side identity fallback | `migration.sql:3,17,33,51,73,88` |
| R-31 | Low | Search | No full-text indexes on product name/description or customer name | Contains search cannot scale; no ranked or tokenized search | `migration.sql:43-45,26-27`; repositories use `contains` |
| R-32 | Low | Pagination | Offset pagination with `skip`/`take` | Deep pages scan and discard rows; no keyset pagination despite indexed `createdAt` | `src/modules/orders/order.repository.ts:36-42`; `src/modules/customers/customer.repository.ts:27-33` |
| R-33 | Low | Retention | No archive or retention policy for `orders`, `order_items` or `order_status_history` | Tables grow without bound; deletion is the only removal path and destroys audit | `prisma/schema.prisma:74-131` |
| R-34 | Low | Connection pool | No explicit Prisma pool size or connection limit | Default pool may exhaust MySQL `max_connections` under many app instances | `src/config/database.ts:4-8`; no pool parameters |
| R-35 | Low | Unused indexes | `orders_createdById_idx`, `customers_document_idx` and `products_name_idx` do not match current queries | Write amplification without read benefit | `prisma/schema.prisma:52,70,95`; repository filter code |
| R-36 | Low | Free-text limits | Application limits (1,000/2,000 characters) are stricter than `TEXT` capacity | Direct writers can store content that the API would reject; display and validation diverge | `prisma/schema.prisma:60,82`; `src/modules/products/product.schemas.ts:6`; `src/modules/orders/order.schemas.ts:15` |

---

## 11. Test Coverage Analysis

### 11.1 Test inventory and locations

| Artifact | Type | Tests | What it exercises in the data model | Coverage assessment | Test quality |
|----------|------|-------|--------------------------------------|---------------------|--------------|
| `tests/orders.test.ts` | Integration via HTTP + direct Prisma reads | 8 | `orders`, `order_items`, `order_status_history`, `order_number_sequence`, `products.stockQuantity`, `customers`, `users` | Strong on order lifecycle; weak on database constraints | Good assertions on totals, order-number regex, history sequence, stock before/after and filtered lists; no cascade, restrict, concurrency or negative-value tests |
| `tests/auth.test.ts` | Integration via HTTP | 7 | `users` creation, unique email, role, bcrypt hash non-exposure, login lookup | Good for user identity path; no role-abuse test | Clear status and error-code assertions; verifies `passwordHash` is not returned; does not test case/accent email collisions |
| `tests/setup.ts` | Test harness | n/a | Connects Prisma and truncates all seven tables before each test | Proves the FK-safe deletion order; no safety guard | Deterministic cleanup but destructive against any configured database |
| `tests/helpers/factories.ts` | Fixtures | n/a | Creates `users`, `customers` and `products`; builds the app with the shared client | Supports the two suites; no order factory | Unique random emails/SKUs avoid collisions; address and price defaults are valid |
| `prisma/seed.ts` | Development script, not executed by tests | n/a | Exercises all models, the sequence, nested order creation and history writes | Not a test; no assertions | Useful as a data-shape reference; destructive and not run in CI |
| `prisma/migrations/20260519182739_init/migration.sql` | Migration artifact | 0 | Defines all tables, enums, indexes and foreign keys | No test applies it to an empty database | No schema-parity or migration-replay test |

### 11.2 Model-by-model coverage

| Model / table | Directly tested | Indirectly tested | Constraints or behaviors exercised | Notable gaps |
|---------------|-----------------|-------------------|------------------------------------|--------------|
| `User` / `users` | Yes, via auth tests and factories | Yes, as order creator and history actor | Unique email conflict, role persistence, hash non-exposure, login lookup | Case/accent collisions, role escalation, password hash length, no-delete restriction |
| `Customer` / `customers` | Creation only via factory | Yes, as order owner | Valid address and document insertion | Customer CRUD endpoints, email uniqueness, duplicate document, address shape rejection, delete restriction |
| `Product` / `products` | Creation only via factory | Yes, pricing, stock and active checks | Price and stock values, stock decrement/replenish, SKU generation | SKU uniqueness, SKU update conflict, inactive-product rejection, negative values, delete restriction |
| `Order` / `orders` | Yes | Yes | Status default, totals, discount, order-number format, list filters, delete guard | Total arithmetic at DB level, negative discount, delete race, idempotency |
| `OrderItem` / `order_items` | Yes, through order responses | Yes | Quantity aggregation, unit price, line total, item count | Duplicate line direct insert, zero/negative quantity, price snapshot after product update |
| `OrderStatusHistory` / `order_status_history` | Yes, through order responses | Yes | Initial `PENDING` row, transition rows, ordering, reason persistence | Illegal direct transitions, duplicate transitions, same-status row, cascade deletion, `changedById` restriction |
| `OrderNumberSequence` / `order_number_sequence` | Yes, indirectly | Yes | Format `ORD-nnnnnn`, multiple orders, sequential values | Concurrent allocation, single-row constraint, reset behavior, rollover past six digits |
| Migration and indexes | No | No | None | Migration replay, schema parity, index existence, collation behavior, foreign-key actions |

### 11.3 Coverage metrics and quality assessment

The repository has 15 integration tests and 0 unit tests for this component. There is no coverage provider in `vitest.config.ts` and no `coverage` script in `package.json`, so no line or branch percentage is produced. Any percentage would therefore be an estimate rather than a measurement.

Behavioral coverage of the order aggregate is good: creation, pricing, order-number format, status transitions, stock movement, history ordering, list filtering and delete protection are all asserted (`tests/orders.test.ts:12-218`). The auth suite covers the primary user-identity path (`tests/auth.test.ts:5-83`). The tests use a real MySQL database, which means foreign keys, enums, unique indexes and `DATETIME(3)` are exercised as infrastructure, even when the assertions do not target them.

Constraint coverage is low. Of the structural guarantees in the schema, only user email uniqueness is explicitly tested. Customer email uniqueness, SKU uniqueness and order-number uniqueness are not directly tested; `orderNumber` is checked for format, not for collision. None of the six foreign-key actions is tested: there is no assertion that deleting an order removes items and history, no assertion that deleting a referenced customer or product fails, and no test for `P2003`. No test attempts a negative price, negative stock, zero quantity, duplicate order line, mismatched total, invalid enum value or invalid JSON address. No test exercises concurrent status changes, concurrent order creation or sequence contention.

The harness has strengths and risks. It connects once, truncates all tables before each test, runs single-fork with file parallelism disabled (`vitest.config.ts:4-17`), and creates a cached Express app with the shared Prisma client (`tests/helpers/factories.ts:8-15`). That produces deterministic serial execution. The absence of an environment guard on the truncation (`tests/setup.ts:8-16`) is the main harness risk: the suite assumes `DATABASE_URL` points at a disposable database, and nothing enforces that assumption.

### 11.4 Test coverage summary

| Dimension | Assessment |
|-----------|------------|
| Order lifecycle behavior | High |
| Authentication and user identity | Medium-High |
| Customer and product CRUD | Low (no endpoint tests) |
| Database unique constraints | Low (one of four directly tested) |
| Foreign-key actions (cascade/restrict) | None |
| Value-range and arithmetic constraints | None |
| Enum and JSON validation at the database | None |
| Migration replay and schema parity | None |
| Concurrency and lock behavior | None |
| Coverage instrumentation | None configured |
| Overall schema-contract coverage | Low, estimated below one third of declared structural guarantees |

---

## Report Metadata

| Field | Value |
|-------|-------|
| Component analyzed | Prisma Data Model and Migrations |
| Files in scope | `prisma/schema.prisma`, `prisma/migrations/migration_lock.toml`, `prisma/migrations/20260519182739_init/migration.sql` |
| Files referenced for enforcement context | `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/modules/orders/order.repository.ts`, `src/modules/orders/order.schemas.ts`, `src/modules/customers/customer.repository.ts`, `src/modules/customers/customer.service.ts`, `src/modules/customers/customer.schemas.ts`, `src/modules/products/product.repository.ts`, `src/modules/products/product.service.ts`, `src/modules/products/product.schemas.ts`, `src/modules/users/user.repository.ts`, `src/modules/users/user.service.ts`, `src/modules/auth/auth.service.ts`, `src/middlewares/error.middleware.ts`, `src/config/database.ts`, `src/config/env.ts`, `prisma/seed.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `tests/auth.test.ts`, `tests/orders.test.ts` |
| Excluded folders | `node_modules`, `.git`, `dist`, `coverage` |
| Excluded scope | Detailed read/write behavior of the audit-trail model, covered by the Order Status History analysis |
| Endpoints | None; the component is a data contract, not a service |
| Report date | 2026-11-09 09:29:11 |
