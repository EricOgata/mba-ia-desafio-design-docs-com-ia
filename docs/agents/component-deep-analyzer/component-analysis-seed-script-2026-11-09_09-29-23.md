# Component Deep Analysis Report

**Component:** Seed Script
**Primary file:** `prisma/seed.ts`
**Project:** order-management-api (`mba-ia-desafio-design-docs-com-ia`)
**Analysis scope:** entire project root
**Excluded from analysis:** `node_modules`, `.git`, `dist`, `coverage`
**Analysis date:** 2026-11-09 09:29:23 (-03)
**Report file:** `docs/agents/component-deep-analyzer/component-analysis-seed-script-2026-11-09_09-29-23.md`

---

## 1. Executive Summary

The Seed Script is a standalone TypeScript CLI program at `prisma/seed.ts` that reconstructs the entire demonstration dataset for the Order Management System. It is development and operational tooling, not part of the HTTP request path: it creates its own `PrismaClient` (`prisma/seed.ts:4`), connects directly to MySQL through Prisma, and is invoked by `npm run db:seed` or by the Prisma seed hook declared in `package.json:22-24`. Its job is to produce a complete, deterministic-looking demo database containing two users, ten customers, twenty products, twenty-six orders, forty-one order items and seventy-eight status-history rows.

The script is destructive by design. Every run first deletes all rows from seven tables in foreign-key-safe order (`prisma/seed.ts:179-187`), then inserts fixed demo users with bcrypt-hashed passwords, fixed customers, fixed products and a hand-authored order plan. The order plan covers all six `OrderStatus` values and reconstructs a valid predecessor chain for each order through `flowFromStatus` (`prisma/seed.ts:255-262`). A local copy of the production order-number allocator reserves sequential numbers `ORD-000001` through `ORD-000026` (`prisma/seed.ts:165-176`, `prisma/seed.ts:285`).

Key findings:

- The seed inserts **26 orders**, not 27 as stated in the architectural report (`docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md:153`). This is a documentation drift finding, not a runtime defect.
- The script has **zero direct test coverage** and is not executed by `npm test`. No file in `tests/` references `prisma/seed.ts`.
- It bypasses every domain layer: no `OrderService`, no repositories, no Zod schemas, no order state machine, no error taxonomy, no Pino logger. It writes Prisma records directly and relies on database constraints plus TypeScript structural typing.
- It duplicates the order-number algorithm from `OrderService.reserveOrderNumber` (`src/modules/orders/order.service.ts:245-254`); the two copies must remain format-compatible.
- It hardcodes demo credentials `admin123` and `operator123` (`prisma/seed.ts:6-7`) and applies no environment or `DATABASE_URL` safety guard before deleting data.
- It does not adjust product stock for the seeded orders. Seeded `PAID` or `PROCESSING` orders, if later cancelled through the API, will inflate stock because the service replenishes stock that the seed never debited.
- The whole run is not transactional. Cleanup, customer inserts, product inserts, order inserts and history inserts are separate operations, so any failure can leave a partially seeded database.

The component exposes no REST, GraphQL, gRPC or other endpoints. Per the analysis guidelines, the Endpoints section is intentionally omitted.

---

## 2. Data Flow Analysis

### 2.1 Top-level invocation and control flow

```
1. Operator or CI runs `npm run db:seed`
   -> package.json:16 -> `tsx --env-file=.env prisma/seed.ts`
   Alternatively Prisma CLI runs the seed hook
   -> package.json:22-24 -> `tsx prisma/seed.ts`

2. Module initialization
   -> imports PrismaClient, OrderStatus, UserRole from @prisma/client (seed.ts:1)
   -> imports bcrypt (seed.ts:2)
   -> constructs `new PrismaClient()` without options (seed.ts:4)
   -> defines fixed credentials (seed.ts:6-7)
   -> defines customersSeed (seed.ts:9-140)
   -> defines productsSeed (seed.ts:142-163)
   -> defines reserveOrderNumber (seed.ts:165-176)

3. main() starts (seed.ts:178)
   -> logs "Cleaning existing data..."
   -> deletes orderStatusHistory, orderItem, order, product, customer, user,
      orderNumberSequence in that order (seed.ts:181-187)

4. User seeding (seed.ts:189-210)
   -> bcrypt.hash('admin123', 10)
   -> bcrypt.hash('operator123', 10)
   -> prisma.user.create admin@oms.local / ADMIN
   -> prisma.user.create operador@oms.local / OPERATOR

5. Customer seeding (seed.ts:212-216)
   -> Promise.all over 10 customersSeed entries
   -> 10 parallel prisma.customer.create calls

6. Product seeding (seed.ts:218-222)
   -> Promise.all over 20 productsSeed entries
   -> 20 parallel prisma.product.create calls

7. Order planning (seed.ts:226-253)
   -> 26 entries in ordersPlan, each with status, itemIdxs, quantities
   -> flowFromStatus defines predecessor chain per final status (seed.ts:255-262)

8. Order generation loop, i = 0..25 (seed.ts:264-321)
   -> customer = customers[i % customers.length]        (seed.ts:266)
   -> author = i % 3 === 0 ? admin : operator           (seed.ts:267)
   -> build items from products[itemIdxs[j]] and quantities[j]
      unitPriceCents = product.priceCents
      totalCents = unitPriceCents * quantity           (seed.ts:269-279)
   -> subtotalCents = sum(line totals)                  (seed.ts:281)
   -> discountCents = i % 7 === 0 ? floor(subtotal * 0.05) : 0
      totalCents = subtotalCents - discountCents       (seed.ts:282-283)
   -> orderNumber = await reserveOrderNumber()          (seed.ts:285)
      -> prisma.$transaction
         -> upsert orderNumberSequence id=1
            create { id:1, nextValue:2 }
            update { nextValue: { increment: 1 } }
         -> current = nextValue - 1
         -> `ORD-${String(current).padStart(6, '0')}`
   -> prisma.order.create with nested orderItems        (seed.ts:287-305)
   -> path = [...flowFromStatus[status], status]        (seed.ts:307)
   -> for each status in path:
        prisma.orderStatusHistory.create
        fromStatus = previous (null for first)
        toStatus = current status
        changedById = author.id
        reason = null for first, `seed transition X->Y` otherwise
                                                        (seed.ts:309-320)

9. Logs "Seed completed." (seed.ts:323-324)
10. main() settles
    -> .catch logs error and process.exit(1)             (seed.ts:327-332)
    -> .finally awaits prisma.$disconnect()              (seed.ts:333-335)
```

### 2.2 Write sequence and atomicity boundaries

```
Delete phase (no transaction)
  order_status_history -> order_items -> orders -> products -> customers -> users
  -> order_number_sequence

Insert phase
  users (2 create calls)
  customers (10 create calls in Promise.all)
  products (20 create calls in Promise.all)
  orders (26 iterations, serial)
    order_number_sequence allocation (own transaction per order)
    order + order_items (single nested Prisma create)
    order_status_history rows (1 to 5 separate create calls per order)

Only reserveOrderNumber uses prisma.$transaction.
There is no transaction spanning cleanup, users, customers, products, orders or history.
```

### 2.3 Data volume produced

| Entity | Count | Source |
|--------|-------|--------|
| Users | 2 | `prisma/seed.ts:194-210` |
| Customers | 10 | `prisma/seed.ts:9-140`, `prisma/seed.ts:213-216` |
| Products | 20 | `prisma/seed.ts:142-163`, `prisma/seed.ts:219-222` |
| Orders | 26 | `prisma/seed.ts:226-253`, loop at `prisma/seed.ts:264-321` |
| Order items | 41 | Sum of `itemIdxs` lengths across 26 plan entries |
| Status history rows | 78 | 4x1 + 5x2 + 4x3 + 4x4 + 6x5 + 3x2 |

Order status distribution:

| Status | Orders |
|--------|--------|
| PENDING | 4 |
| PAID | 5 |
| PROCESSING | 4 |
| SHIPPED | 4 |
| DELIVERED | 6 |
| CANCELLED | 3 |
| **Total** | **26** |

---

## 3. Business Rules & Logic

### 3.1 Overview of the business rules

| ID | Rule Type | Rule Description | Location | Confidence |
|----|-----------|------------------|----------|------------|
| BR-01 | Data lifecycle | Delete all rows from seven tables before seeding, in foreign-key-safe order | `prisma/seed.ts:179-187` | High |
| BR-02 | Identity | Create exactly two demo users with fixed emails, names and roles | `prisma/seed.ts:194-210` | High |
| BR-03 | Security | Hash fixed demo passwords with bcrypt cost 10 | `prisma/seed.ts:6-7`, `prisma/seed.ts:191-192` | High |
| BR-04 | Dataset | Create ten fixed customers with formatted CPF/CNPJ and JSON addresses | `prisma/seed.ts:9-140`, `prisma/seed.ts:213-216` | High |
| BR-05 | Dataset | Create twenty fixed products with unique SKUs, cent prices and stock | `prisma/seed.ts:142-163`, `prisma/seed.ts:219-222` | High |
| BR-06 | Dataset | Create twenty-six hand-authored orders covering all six statuses | `prisma/seed.ts:226-253` | High |
| BR-07 | Distribution | Assign order i to customer `i % 10` | `prisma/seed.ts:266` | High |
| BR-08 | Distribution | Assign order i to admin when `i % 3 === 0`, otherwise operator | `prisma/seed.ts:267` | High |
| BR-09 | Pricing | Snapshot product price per item and compute line and subtotal totals | `prisma/seed.ts:269-281` | High |
| BR-10 | Pricing | Apply a 5 percent discount to orders where `i % 7 === 0` | `prisma/seed.ts:282-283` | Medium |
| BR-11 | Numbering | Reserve sequential `ORD-nnnnnn` numbers through the sequence table | `prisma/seed.ts:165-176`, `prisma/seed.ts:285` | High |
| BR-12 | Persistence | Persist order, nested items, status and creator in one create call | `prisma/seed.ts:287-305` | High |
| BR-13 | Audit | Reconstruct a valid status-history chain for each final status | `prisma/seed.ts:255-262`, `prisma/seed.ts:307-320` | High |
| BR-14 | Validation | Rely on database constraints and TypeScript, not domain validation | `prisma/seed.ts:287-318` | High |
| BR-15 | Inventory | Do not debit or replenish stock for seeded orders | `prisma/seed.ts:142-163`, `prisma/seed.ts:287-321` | High |
| BR-16 | Reproducibility | Re-running resets the logical dataset but not UUIDs, timestamps or hashes | `prisma/seed.ts:179-187` | High |
| BR-17 | Failure handling | Log failures, exit with code 1, and disconnect in a finally block | `prisma/seed.ts:327-335` | High |
| BR-18 | Execution | Run through tsx with `.env` loading or through the Prisma seed hook | `package.json:16`, `package.json:22-24` | High |
| BR-19 | Concurrency | Use parallel inserts for reference data and serial creation for orders | `prisma/seed.ts:213-222`, `prisma/seed.ts:264-321` | High |
| BR-20 | Temporal data | Let the database assign all creation and change timestamps | `prisma/schema.prisma:31-32`, `prisma/schema.prisma:121` | High |

### 3.2 Detailed breakdown of the business rules

---

### Business Rule: BR-01 Destructive Reset Before Seeding

**Overview**:
Every execution begins by deleting all existing rows from the seven seeded tables. The script is a reset-and-load fixture, not an incremental loader.

**Detailed description**:
The cleanup block at `prisma/seed.ts:179-187` calls `deleteMany()` without any filter on `orderStatusHistory`, `orderItem`, `order`, `product`, `customer`, `user` and `orderNumberSequence`. The order is chosen so that child rows are removed before parent rows: history and items before orders, orders before customers and users, and all order-dependent data before products. This manual order is required because the schema uses `ON DELETE RESTRICT` from `orders` to `customers` and `users`, and from `order_items` to `products` (`prisma/migrations/20260519182739_init/migration.sql:110-125`). If a future schema change adds another restricting foreign key, the hardcoded sequence must be updated or the seed will abort part-way through cleanup.

The deletion has no `WHERE` clause and no environment check. Whatever database is reachable through `DATABASE_URL` will lose all user, customer, product, order, item, history and sequence data. This includes records created through the API after the previous seed, not only records created by the seed itself. There is no confirmation prompt, no dry-run mode and no backup step. The only protection is operational discipline outside the script.

Cleanup is not wrapped in a transaction. Each `deleteMany` is an independent statement. A crash or connection loss between statements leaves the database in a partially deleted state. A subsequent run can recover because it starts by deleting again, but there is no automatic rollback and no guarantee that the database is usable between the failure and the next successful run.

**Rule workflow**:
```
main() -> delete orderStatusHistory
       -> delete orderItem
       -> delete order
       -> delete product
       -> delete customer
       -> delete user
       -> delete orderNumberSequence
       -> proceed to insert phase
```

---

### Business Rule: BR-02 Exactly Two Demo Users with Fixed Roles

**Overview**:
The seed creates one administrator and one operator, with fixed emails and display names, and uses them as the actors for every seeded order and history row.

**Detailed description**:
The admin is created at `prisma/seed.ts:194-201` with email `admin@oms.local`, name `Administrador do Sistema` and role `UserRole.ADMIN`. The operator is created at `prisma/seed.ts:203-210` with email `operador@oms.local`, name `Operador Padrão` and role `UserRole.OPERATOR`. Both records are written through `prisma.user.create`, not through `UserService.createUser`, so the service-level duplicate-email check and the service's public projection are bypassed. Identity relies on the unique index `users_email_key` (`prisma/schema.prisma:27`) and on the cleanup step having removed previous users.

Exactly two users are produced. The rest of the seed depends on both references: every order has a `createdById`, and every history row has a `changedById`. The admin is used as author for nine orders and the operator for seventeen orders, following BR-08. The ordering of these two creates matters because the resulting `admin` and `operator` variables are later captured by the loop. If either create fails, `main()` rejects before any customers, products or orders are written.

The role values come from the Prisma-generated `UserRole` enum, which is one of `ADMIN` and `OPERATOR` (`prisma/schema.prisma:11-14`). The seed does not check whether an equivalent user already exists, because the preceding cleanup is expected to remove it. If cleanup is skipped or fails, the unique email constraint aborts the run. There is no support for additional users, tenant-specific users or environment-specific identities.

**Rule workflow**:
```
hash admin password -> create admin@oms.local (ADMIN)
hash operator password -> create operador@oms.local (OPERATOR)
store admin and operator references in local variables
```

---

### Business Rule: BR-03 Fixed Demo Passwords Hashed with bcrypt Cost 10

**Overview**:
The script uses two plaintext password constants and hashes each with bcrypt using ten rounds. The plaintext values are committed in source.

**Detailed description**:
`ADMIN_PASSWORD` is `admin123` and `OPERATOR_PASSWORD` is `operator123` (`prisma/seed.ts:6-7`). Both are hashed at `prisma/seed.ts:191-192` with `bcrypt.hash(password, 10)`. The cost factor matches `BCRYPT_ROUNDS = 10` used by `UserService.createUser` (`src/modules/users/user.service.ts:16,26`), so seeded accounts authenticate through the normal login flow at `AuthService.login`, which calls `bcrypt.compare` (`src/modules/auth/auth.service.ts:31-45`). The admin password has eight characters and the operator password has eleven, both satisfying the registration schema minimum of eight (`src/modules/auth/auth.schemas.ts:5`) and the login schema minimum of one (`src/modules/auth/auth.schemas.ts:12`).

bcrypt generates a random salt per hash, so a re-run produces different `passwordHash` values even though the plaintext and cost are identical. The hashes are stored in `users.passwordHash`, declared as `VARCHAR(255)` (`prisma/schema.prisma:28`), and are never logged by the script. The script does not print credentials, does not read them from environment variables and does not support rotation or per-environment passwords.

The security consequence is that the credentials are public to anyone with repository access. If the seed is pointed at a shared, staging or production database, those known credentials become valid login credentials for an `ADMIN` account and an `OPERATOR` account. The script has no environment guard that would prevent this, so the rule is safe only under the assumption that the target database is disposable and local.

**Rule workflow**:
```
read ADMIN_PASSWORD constant
-> bcrypt.hash(..., 10)
-> persist as admin.passwordHash

read OPERATOR_PASSWORD constant
-> bcrypt.hash(..., 10)
-> persist as operator.passwordHash
```

---

### Business Rule: BR-04 Ten Fixed Customers with Structured Address Data

**Overview**:
The seed inserts exactly ten customers, mixing individuals and companies, with fixed identity, contact and address values.

**Detailed description**:
`customersSeed` (`prisma/seed.ts:9-140`) contains ten object literals. Each has `name`, `email`, `phone`, `document` and an `address` object with `street`, `number`, `city`, `state` and `zipCode`. Eight entries use `@exemplo.com.br` addresses and represent individuals; two entries represent companies, `Logística Atlas Comercial Ltda` with `contato@atlascomercial.com.br` and `Distribuidora Nordeste S.A.` with `compras@nordestedist.com.br`. Documents mix formatted CPF values such as `123.456.789-00` and formatted CNPJ values such as `12.345.678/0001-90`. Phones use the Brazilian international prefix with a two-digit area code.

The records are inserted through `Promise.all(customersSeed.map((c) => prisma.customer.create({ data: c })))` at `prisma/seed.ts:213-216`. This bypasses `CustomerService` and the Zod schemas in `src/modules/customers/customer.schemas.ts`, so no application-level email normalization, phone format check or address shape validation occurs. The unique email index (`prisma/schema.prisma:43`) and the `VARCHAR` limits are the only enforced constraints. The `address` field is a MySQL `JSON` column (`prisma/schema.prisma:46`) with no database-level shape validation, so the object structure is guaranteed only by the literal data in the script.

`Promise.all` preserves input order in its result array, which is essential: later code indexes `customers[i % customers.length]` and expects index `i` to correspond to the `i`-th literal. The database insertion order is not guaranteed, but the result array order is. Every one of the ten customers is used by at least two seeded orders, so no customer is orphaned. If one create fails, `Promise.all` rejects and the remaining inserts may still complete or may be in flight, leaving a partial customer set.

**Rule workflow**:
```
customersSeed[0..9]
-> Promise.all map to prisma.customer.create
-> result array aligned with input index
-> customers referenced later as customers[i % 10]
```

---

### Business Rule: BR-05 Twenty Fixed Products with Unique SKUs and Cent Prices

**Overview**:
The seed inserts exactly twenty products with fixed SKU, name, description, integer cent price and integer stock quantity. All products are active by default.

**Detailed description**:
`productsSeed` (`prisma/seed.ts:142-163`) defines twenty entries. Each has `sku`, `name`, `description`, `priceCents` and `stockQuantity`. The `active` field is omitted, so the Prisma schema default `true` applies (`prisma/schema.prisma:63`), making every seeded product active. The SKUs follow category-like prefixes such as `NB-DELL-001`, `MON-LG-001` and `PAP-A4-001`, but no format rule is enforced. Prices are stored as integer cents, for example `459900` for a notebook and `3290` for a ream of paper. Stock quantities range from 12 to 300 units.

Products are inserted with `Promise.all(productsSeed.map((p) => prisma.product.create({ data: p })))` at `prisma/seed.ts:219-222`. As with customers, the result array preserves the literal index order. This order is a hard dependency of the order plan: `ordersPlan` stores `itemIdxs` as zero-based indexes into the `products` array, and the loop resolves `products[idx]` at `prisma/seed.ts:270`. Reordering `productsSeed`, filtering it, or changing the insertion path to one that does not preserve order would silently remap every order item to a different product.

The insert bypasses `ProductService` and product Zod schemas. The unique SKU index (`prisma/schema.prisma:58`) rejects duplicates, but there is no database `CHECK` for non-negative prices or stock, no category field, no SKU format constraint and no description length constraint at the database level. The script trusts the literal data. It also never updates product prices or stock later, so the values remain the catalog snapshot for the lifetime of the seeded database.

**Rule workflow**:
```
productsSeed[0..19]
-> Promise.all map to prisma.product.create
-> result array aligned with input index
-> orders resolve product by itemIdxs position
```

---

### Business Rule: BR-06 Twenty-Six Orders Covering All Six Statuses

**Overview**:
The order plan is a hand-authored array of twenty-six orders. Every `OrderStatus` value appears at least three times, and every order has one or more item references.

**Detailed description**:
`ordersPlan` is declared at `prisma/seed.ts:226-253` with twenty-six object literals at `prisma/seed.ts:227-252`. Each literal has a final `status`, an `itemIdxs` array of product positions and a `quantities` array. The status distribution is: four `PENDING`, five `PAID`, four `PROCESSING`, four `SHIPPED`, six `DELIVERED` and three `CANCELLED`. The plan produces forty-one order items in total. The plan is not generated algorithmically; it is a fixture authored to exercise list filters, status badges and order detail views.

The array type annotation constrains `status` to `OrderStatus` and both numeric arrays to `number[]`, but it does not enforce that `itemIdxs` and `quantities` have equal length, that indices are within the products array, or that quantities are positive. The loop uses non-null assertions `products[idx]!` and `plan.quantities[j]!` (`prisma/seed.ts:270-271`), so an out-of-range index produces `undefined` at runtime and likely a TypeError when `priceCents` is read. A length mismatch produces an undefined quantity and a `NaN` total. These failures are caught only by the top-level catch, after earlier rows may already be committed.

The plan is the single source of order count, status mix and item composition. Changing an entry changes not only that order but also the deterministic numbering, because order numbers are allocated in loop order. The plan does not include notes, explicit dates, customer identifiers or explicit product identifiers; those are derived by position and by the rules below. This makes the fixture compact but couples it to array ordering.

**Rule workflow**:
```
ordersPlan = 26 entries
for i in 0..25:
  read status, itemIdxs, quantities
  validate only through TypeScript and non-null assertions
  build one order
```

---

### Business Rule: BR-07 Round-Robin Customer Assignment

**Overview**:
Order `i` is assigned to customer `customers[i % 10]` (`prisma/seed.ts:266`). The rule distributes orders evenly across the ten seeded customers.

**Detailed description**:
The expression `const customer = customers[i % customers.length]!` maps loop index zero to the first customer, index one to the second, and so on, wrapping after index nine. Because there are twenty-six orders and ten customers, customers at positions 0 through 5 receive three orders each, while customers at positions 6 through 9 receive two orders each. This guarantees that every customer has at least two orders, which is useful for manually testing customer filters and order lists. No customer is left with zero orders.

The rule depends on the result array from `Promise.all` preserving the input order, not on database insertion order. It also depends on `customers.length` being non-zero; with an empty array, `i % 0` is `NaN` and the lookup yields `undefined`. The script does not check for that condition, but the fixed dataset guarantees ten entries.

The assignment has no business meaning beyond coverage. It does not model customer activity, geography, order value or any domain segmentation. The modulo constant is effectively `10` through `customers.length`; changing the customer dataset silently changes every assignment after the first ten orders.

**Rule workflow**:
```
for order index i:
  customer = customers[i % customers.length]
  use customer.id as order.customerId
```

---

### Business Rule: BR-08 Author Assignment and History Actor

**Overview**:
Orders whose zero-based index is divisible by three are authored by the admin; all others are authored by the operator. The same author is used for every history row of that order.

**Detailed description**:
`const author = i % 3 === 0 ? admin : operator` (`prisma/seed.ts:267`) assigns indices 0, 3, 6, 9, 12, 15, 18, 21 and 24 to the admin, which is nine orders. The remaining seventeen orders are authored by the operator. The selected user becomes both `order.createdById` (`prisma/seed.ts:295`) and `orderStatusHistory.changedById` for every transition of that order (`prisma/seed.ts:315`). The seed therefore presents a simplified audit story in which the creator performed every status change.

There is no authorization check, role check or service call. The assignment is purely a numeric pattern. It guarantees both roles appear as actors and that the admin account is not an orphan. It does not model a real workflow in which operators create orders and administrators later change statuses, and it does not vary the actor across the lifecycle of a single order.

Because the same reference is reused, if the admin or operator user were missing, the order create would fail on the `createdById` foreign key and the history create would fail on `changedById`. The fixed creation order in `main()` ensures both exist before the loop starts.

**Rule workflow**:
```
author = (i % 3 === 0) ? admin : operator
order.createdById = author.id
history.changedById = author.id for every history row
```

---

### Business Rule: BR-09 Item Price Snapshot and Total Computation

**Overview**:
Each order item takes its unit price from the current product record, computes a line total, and sums line totals into the order subtotal.

**Detailed description**:
Inside the loop, `items` is built by mapping each `itemIdxs` position to the corresponding product and quantity (`prisma/seed.ts:269-279`). The unit price is `product.priceCents`, the line total is `unitPriceCents * quantity`, and the subtotal is the sum of line totals (`prisma/seed.ts:281`). This mirrors the pricing behavior of `OrderService.create` (`src/modules/orders/order.service.ts:75-86`) without reusing its code. Prices are integers in cents, so all arithmetic stays in integer cents with no decimal rounding.

The price is a snapshot. It is written into `order_items.unitPriceCents` and `order_items.totalCents` (`prisma/schema.prisma:104-105`). If a product price changes later, existing seeded orders keep the old price, which is consistent with the production order-item model. There is no tax, shipping, surcharge or currency field in the schema or seed. The script does not use historical price tables because none exist.

The computation trusts the plan. It does not check that quantity is positive, that unit price is non-negative, or that `itemIdxs` contains unique products. Duplicate product positions in one order would produce multiple order items for the same product, which the schema permits; the API's `aggregateItems` behavior (`src/modules/orders/order.service.ts:55,194-202`) is not replicated. The current plan contains no duplicate position within a single order.

**Rule workflow**:
```
for each position j in plan:
  product = products[itemIdxs[j]]
  quantity = quantities[j]
  unitPriceCents = product.priceCents
  totalCents = unitPriceCents * quantity
subtotalCents = sum(totalCents)
```

---

### Business Rule: BR-10 Five Percent Discount on Every Seventh Order

**Overview**:
Orders at zero-based indices divisible by seven receive a discount equal to the floor of five percent of their subtotal. All other orders receive zero discount.

**Detailed description**:
`discountCents = i % 7 === 0 ? Math.floor(subtotalCents * 0.05) : 0` and `totalCents = subtotalCents - discountCents` (`prisma/seed.ts:282-283`). The rule applies to indices 0, 7, 14 and 21, which are the 1st, 8th, 15th and 22nd orders in one-based terms. Their computed subtotals and discounts are:

| Order position | Loop index | Status | Subtotal (cents) | Discount (cents) | Total (cents) |
|----------------|------------|--------|------------------|------------------|---------------|
| 1 | 0 | PENDING | 639,700 | 31,985 | 607,715 |
| 8 | 7 | PAID | 219,900 | 10,995 | 208,905 |
| 15 | 14 | SHIPPED | 459,700 | 22,985 | 436,715 |
| 22 | 21 | DELIVERED | 81,700 | 4,085 | 77,615 |

The 5 percent rate is a seed-only convention. There is no discount engine, coupon table or percentage rule in the application. `OrderService.create` only validates that a client-supplied `discountCents` does not exceed the subtotal (`src/modules/orders/order.service.ts:87-92`). The seed therefore demonstrates the presence of a discount field but not a reusable business policy. The use of `Math.floor` keeps the stored value an integer, with fractions of a cent discarded.

The rule never produces a discount greater than the subtotal at a 5 percent rate, so the service's discount guard is not exercised. If the rate were changed above 100 percent or the base changed, the seed could write a negative total without any database constraint stopping it. Nothing in the schema enforces `totalCents = subtotalCents - discountCents` or non-negative values.

**Rule workflow**:
```
if i % 7 === 0:
  discountCents = floor(subtotalCents * 0.05)
else:
  discountCents = 0
totalCents = subtotalCents - discountCents
```

---

### Business Rule: BR-11 Sequential Order Number Reservation

**Overview**:
Order numbers are allocated from a single-row sequence table in the format `ORD-` followed by a zero-padded six-digit number. The first seeded order is `ORD-000001`.

**Detailed description**:
`reserveOrderNumber` (`prisma/seed.ts:165-176`) runs an interactive transaction containing an upsert on `orderNumberSequence` row `id = 1`. On an empty table, the create branch writes `nextValue: 2` and returns `2`; the function then computes `current = seq.nextValue - 1 = 1` and formats `ORD-000001`. On subsequent calls, the update branch increments `nextValue` by one; the returned value minus one is the allocated number. Because the loop awaits each call sequentially, the seed produces `ORD-000001` through `ORD-000026` in plan order, leaving `nextValue = 27`. The sequence table is deleted during cleanup, so every full re-run restarts at `ORD-000001`.

The algorithm is a byte-for-byte duplicate of `OrderService.reserveOrderNumber` (`src/modules/orders/order.service.ts:245-254`). The difference is transactional scope: the service allocates inside the order-creation transaction, whereas the seed allocates in its own transaction and creates the order afterward. If order creation fails after allocation, the number is consumed and a gap remains. In the seed those gaps matter only if a failure is fixed and the script is resumed without a clean reset, which is not supported. The unique index `orders_orderNumber_key` (`prisma/schema.prisma:76`) protects against duplicate numbers.

`padStart(6, '0')` sets a minimum width, not a maximum. After 999,999 allocations the number grows to seven digits, which still fits `VARCHAR(20)`. There is no date prefix, customer prefix or retry logic. The sequence is global to the database, not per environment or per tenant.

**Rule workflow**:
```
BEGIN TRANSACTION
  UPSERT order_number_sequence id=1
    create: nextValue=2
    update: nextValue += 1
  current = nextValue - 1
COMMIT
return "ORD-" + current padded to 6 digits
```

---

### Business Rule: BR-12 Order Persistence with Nested Items

**Overview**:
Each order is written with its final status, customer, author, monetary totals and nested order items in one Prisma create call. Notes are null and history is written separately.

**Detailed description**:
The create at `prisma/seed.ts:287-305` supplies `orderNumber`, `customerId`, `status`, `subtotalCents`, `discountCents`, `totalCents`, `createdById` and a nested `items.create` array. Each item carries `productId`, `quantity`, `unitPriceCents` and `totalCents`. `notes` is omitted, so it remains null per the optional schema field (`prisma/schema.prisma:82`). The status is written directly from the plan; the order state machine in `src/modules/orders/order.status.ts` is not consulted. The order is created with its final status in one step rather than by applying a sequence of transitions.

The nested item write is atomic at the Prisma level for the order and its items. It is not atomic with the subsequent history writes. The order exists with its items before the first history row is inserted, and a failure during the history loop leaves a fully created order with a partial or empty history. This is visible to API consumers because the order detail query eagerly loads history (`src/modules/orders/order.repository.ts:47-59`).

The order response is not used after creation except for `order.id` when writing history. The script does not request included relations, so it does not verify the final shape. The database defaults supply `id`, `createdAt` and `updatedAt` (`prisma/schema.prisma:75-85`). Order items receive their own `createdAt` (`prisma/schema.prisma:106`).

**Rule workflow**:
```
prisma.order.create({
  orderNumber, customerId, status,
  subtotalCents, discountCents, totalCents,
  createdById,
  items.create[...]
})
-> order.id used for history rows
```

---

### Business Rule: BR-13 Status History Chain Reconstruction

**Overview**:
For every final status, the seed writes a complete predecessor chain ending in that status. The first row has a null source status; every later row records the transition and a generated reason string.

**Detailed description**:
`flowFromStatus` (`prisma/seed.ts:255-262`) is a record from final status to its ordered predecessors. `PENDING` has no predecessors; `PAID` has `[PENDING]`; `PROCESSING` has `[PENDING, PAID]`; `SHIPPED` has `[PENDING, PAID, PROCESSING]`; `DELIVERED` has `[PENDING, PAID, PROCESSING, SHIPPED]`; and `CANCELLED` has `[PENDING]`. The loop builds `path = [...flowFromStatus[plan.status], plan.status]` (`prisma/seed.ts:307`) and creates one history row per element. The first row has `fromStatus: null` and `reason: null`; subsequent rows have the previous status as `fromStatus` and the literal reason `seed transition ${prev}->${s}` (`prisma/seed.ts:311-317`). Every row uses the order's author as `changedById`.

The resulting chains are valid paths under the production state machine: `PENDING -> PAID -> PROCESSING -> SHIPPED -> DELIVERED` and `PENDING -> CANCELLED` are both allowed by `src/modules/orders/order.status.ts:3-10`. The seed does not demonstrate cancellations from `PAID` or `PROCESSING`, although the state machine allows them (`src/modules/orders/order.status.ts:4-6`). This is a representativeness limitation, not an invalid path. The total history volume is seventy-eight rows, calculated from the per-status chain lengths.

`changedAt` is not supplied, so MySQL/Primsa uses the current timestamp for each row (`prisma/schema.prisma:121`). Because all rows for one order are inserted in a tight loop, several transitions can share the same millisecond. The order repository loads history with `orderBy: { changedAt: 'asc' }` and no secondary key (`src/modules/orders/order.repository.ts:56`), so equal timestamps can produce an unstable display order. The reasons are generic synthetic strings, and the actor is always the creator, so the audit data is structurally valid but not behaviorally realistic.

**Rule workflow**:
```
path = flowFromStatus[finalStatus] + [finalStatus]
prev = null
for status in path:
  create history orderId, fromStatus=prev, toStatus=status,
    changedById=author.id, reason=(prev ? `seed transition ${prev}->${status}` : null)
  prev = status
```

---

### Business Rule: BR-14 Database Constraints and TypeScript Instead of Domain Validation

**Overview**:
The seed performs no application-level validation. Correctness depends on TypeScript compile-time checking, Prisma schema typing and MySQL constraints.

**Detailed description**:
There is no Zod parsing, no call to a service, no repository method and no explicit invariant check in `prisma/seed.ts`. The customer and product objects are passed directly to Prisma creates; the order plan is passed to Prisma nested creates; the history statuses are passed as enum values. TypeScript catches invalid enum members and unknown required fields at compile time, and Prisma's generated types catch mismatched field names for fresh object literals in some positions. Because values are held in variables and arrays, excess-property checks are weaker than for inline literals, so a schema change can require manual reconciliation.

The database enforces unique emails, unique SKUs, unique order numbers, enum membership and foreign keys (`prisma/migrations/20260519182739_init/migration.sql:11,26,43,63,110-125`). It does not enforce non-negative `priceCents`, `stockQuantity`, `quantity`, `subtotalCents`, `discountCents` or `totalCents`, nor an arithmetic relationship between subtotal, discount and total. It does not enforce a status transition history or a single active history chain. Any data that satisfies types and constraints is accepted.

This means the seed can create states that the API would reject. Examples include an order with a discount greater than its subtotal, an order in `PAID` without a preceding stock debit, a history row whose actor did not perform the transition, or a product with negative stock. The current fixture avoids most of these, but the seed is not a business-rule oracle. It is a data loader whose contract is the Prisma schema, not the domain service.

**Rule workflow**:
```
literal data
-> TypeScript structural check
-> Prisma create
-> MySQL unique/enum/foreign-key checks
-> stored without domain validation
```

---

### Business Rule: BR-15 No Stock Debit or Replenishment for Seeded Orders

**Overview**:
Product stock quantities are static catalog values. The seed creates orders in statuses that imply stock movements, but it never changes `products.stockQuantity`.

**Detailed description**:
`productsSeed` assigns fixed `stockQuantity` values (`prisma/seed.ts:142-163`), and the order loop never updates products. The production lifecycle debits stock on `PENDING -> PAID` and replenishes on `PAID|PROCESSING -> CANCELLED` (`src/modules/orders/order.status.ts:24-37`, `src/modules/orders/order.service.ts:151-156,204-243`). Seeded orders are placed directly in `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` and `CANCELLED` without those movements, so the stored stock is not the result of the seeded order history.

This creates observable inconsistencies during manual testing. A seeded `PENDING` order can be paid through the API and will debit stock for the first time, which is consistent with its own history but not with any prior reservation. A seeded `PAID` or `PROCESSING` order can be cancelled through the API and will increment stock even though no debit ever occurred, producing inventory inflation. A seeded `CANCELLED` order has no stock effect, which is correct for a cancellation that never passed through `PAID`. The seed also only creates cancellations from `PENDING`, so no seeded history demonstrates the replenishment path at all.

The script does not check whether the cumulative ordered quantities exceed the catalog stock. In the current fixture every individual line and every per-product cumulative demand is within the seeded stock, but that is a property of the hand-authored numbers, not an enforced rule. Because no stock is deducted, cumulative demand does not constrain future API operations either.

**Rule workflow**:
```
seed products with fixed stock
-> seed orders with final statuses directly
-> no product.update for stock
-> later API transitions act on stock that seed never changed
```

---

### Business Rule: BR-16 Re-run Resets Logical Data but Not Physical Identity

**Overview**:
A second run produces the same counts and logical composition, but different UUIDs, timestamps and bcrypt hashes. The script is reset-idempotent, not byte-idempotent.

**Detailed description**:
The cleanup at the start of every run removes all seven tables, so a re-run does not accumulate rows or fail on unique constraints. The same literal names, emails, SKUs, prices, statuses and order-number sequence are reconstructed. Counts remain two users, ten customers, twenty products, twenty-six orders, forty-one items and seventy-eight history rows. Order numbers restart at `ORD-000001` because the sequence row is deleted.

Physical identity changes. User, customer, product, order, item and history IDs are generated by Prisma `uuid()` defaults (`prisma/schema.prisma:26,41,57,75,100,117`), so every row receives a new UUID. All `createdAt`, `updatedAt` and `changedAt` values are new database timestamps. bcrypt produces a new random salt, so password hashes differ even for the same plaintext. Any external bookmark, log correlation or saved API response that references an old ID becomes stale.

Running the seed while the API is live is unsafe. The delete phase creates a window in which tables are empty, and concurrent API writes can race with the inserts. Two concurrent seed processes can interleave deletes and inserts, causing unique-constraint failures or a mixture of data from both runs. There is no advisory lock, no idempotency key and no coordination with the application.

**Rule workflow**:
```
run N: delete all -> insert logical dataset A with identity set N
run N+1: delete all -> insert logical dataset A with identity set N+1
logical equality: counts and literal values
physical equality: false (UUIDs, timestamps, hashes differ)
```

---

### Business Rule: BR-17 Failure Handling, Exit Code and Disconnect

**Overview**:
The script logs an unhandled error, exits with code 1, and attempts to disconnect Prisma in a finally block. There is no rollback or retry.

**Detailed description**:
`main()` is invoked at `prisma/seed.ts:327` without `await`. The promise chain attaches `.catch((err) => { console.error(err); process.exit(1); })` and `.finally(async () => { await prisma.$disconnect(); })` (`prisma/seed.ts:327-335`). A successful run resolves `main`, runs the finally callback, awaits disconnect, and then lets the Node event loop drain naturally. A failed run logs the raw error and calls `process.exit(1)`, which terminates the process with a non-zero status that npm and CI can detect.

On failure, the immediate `process.exit(1)` can prevent the asynchronous `finally` callback from completing. The process terminates anyway, so the practical effect is that connections are closed by process exit rather than by a graceful Prisma disconnect. More importantly, there is no transaction spanning the run, so whatever was written before the failure remains. The error is printed as a raw object; there is no categorization, no contextual message identifying which seed phase failed, and no retry or resume logic.

There is no explicit `$connect()` call. Prisma connects lazily on the first query, so a missing or unreachable database surfaces as a rejection from the first `deleteMany`, which then flows into the catch block. The top-level catch does not distinguish connection errors from constraint errors from programming errors.

**Rule workflow**:
```
start main()
  success -> finally -> $disconnect -> natural exit 0
  failure -> catch -> console.error -> process.exit(1)
             (finally may not complete; no rollback)
```

---

### Business Rule: BR-18 Execution Contract and Environment Loading

**Overview**:
The seed is run as a TypeScript file through `tsx`, either with an explicit `.env` flag from npm or through the Prisma CLI seed hook. It requires only `DATABASE_URL` and has no environment-specific behavior.

**Detailed description**:
`package.json:16` defines `db:seed` as `tsx --env-file=.env prisma/seed.ts`. `package.json:22-24` registers the Prisma seed hook as `tsx prisma/seed.ts`, so `prisma migrate reset` and `prisma db seed` can invoke it. The Prisma CLI loads `.env` for its own process and passes the environment to the spawned seed process, while the npm script loads `.env` explicitly through Node 20's `--env-file` support. The script itself does not import `src/config/env.ts`, so it bypasses the Zod validation and fail-fast behavior applied to the API runtime.

The only required variable is `DATABASE_URL`. `SHADOW_DATABASE_URL` is needed by `prisma migrate dev` but not by the seed. There is no `NODE_ENV` check, no target-environment allowlist, no `--confirm` flag and no `--dry-run` mode. The same code path runs against a local container, a shared development database or any production URL supplied in the environment.

The seed is excluded from the production build because `tsconfig.build.json:9-10` includes only `src/**/*.ts` and excludes `prisma`. It is type-checked by the root `tsconfig.json:28`, which includes `prisma/**/*.ts`, so compilation catches type errors but does not produce a runnable `dist/prisma/seed.js`. The script therefore depends on `tsx` and the TypeScript sources being present at execution time.

**Rule workflow**:
```
npm run db:seed
  -> tsx --env-file=.env prisma/seed.ts
or
prisma db seed / prisma migrate reset
  -> seed hook from package.json:22-24
  -> tsx prisma/seed.ts
-> Prisma reads DATABASE_URL
-> seed executes with no environment guard
```

---

### Business Rule: BR-19 Parallel Reference Inserts and Serial Order Inserts

**Overview**:
Customers and products are inserted in parallel with `Promise.all`; orders are inserted serially in a `for` loop. There is no transaction spanning either phase.

**Detailed description**:
The customer insert (`prisma/seed.ts:213-216`) and product insert (`prisma/seed.ts:219-222`) each launch multiple Prisma create promises concurrently. With ten and twenty records this is small, but it still consumes multiple connections from the pool and produces no deterministic database insertion order. The returned arrays preserve input order, which is why later positional lookups work. A failure in any one create rejects the aggregate promise, while the other creates may already have committed.

Orders are created in a sequential `for` loop (`prisma/seed.ts:264-321`) because each order needs a sequence allocation and because the order-number order is part of the fixture. Within one iteration, the order plus nested items is one atomic create, the number reservation is its own transaction, and the history rows are separate creates. There is no outer transaction, so a failure after order creation but before history completion leaves a partially audited order.

This mix is deliberate but undocumented. Parallelism improves speed for reference data; serialization preserves numbering. It also means the script is not safe for concurrent execution and not atomic for recovery. A production-grade loader would typically use chunked `createMany`, an outer transaction or idempotent upserts, but those are not present.

**Rule workflow**:
```
users: serial create x2
customers: Promise.all create x10
products: Promise.all create x20
orders: serial loop
  reserve number (transaction)
  create order + items
  create history rows serially
```

---

### Business Rule: BR-20 Database-Assigned Timestamps

**Overview**:
The seed never supplies business dates. All `createdAt`, `updatedAt` and `changedAt` values are assigned by database defaults at insertion time.

**Detailed description**:
The schema defines `createdAt` and `updatedAt` defaults for users, customers, products, orders and items, and `changedAt` for history (`prisma/schema.prisma:31-32,47-48,64-65,84-85,106,121`). The seed omits all of them. As a result, the entire dataset is stamped with the moment the seed runs, and all orders appear to have been created within the same few seconds.

This affects any feature that sorts or filters by time. Order lists are sorted by `createdAt` descending with no secondary key (`src/modules/orders/order.repository.ts:38`), and MySQL may return tied millisecond timestamps in an unstable order. History is sorted by `changedAt` ascending with no secondary key (`src/modules/orders/order.repository.ts:56`), so transitions within one order can display out of sequence when they share a millisecond. The seed cannot demonstrate daily order volume, aging, SLA or backdated history because no dates are backdated.

The database uses `DATETIME(3)`, millisecond precision, and no explicit timezone strategy is encoded in the schema or the seed. Timestamps are written in the database session's timezone. Manual comparisons across environments should account for that.

**Rule workflow**:
```
insert row without createdAt/updatedAt/changedAt
-> MySQL default CURRENT_TIMESTAMP(3)
-> all seed rows share the seed execution window
```

---

## 4. Component Structure

### 4.1 Component boundary

The Seed Script component consists of one executable TypeScript module, `prisma/seed.ts`, plus the configuration and schema contracts required to run it. It has no `src/` module boundary, no class, no exported API and no imports from application source.

```
prisma/
├── seed.ts                                  # The whole component (335 lines)
│   ├── imports and Prisma client            # lines 1-4
│   ├── fixed credentials                    # lines 6-7
│   ├── customersSeed dataset                # lines 9-140
│   ├── productsSeed dataset                 # lines 142-163
│   ├── reserveOrderNumber()                 # lines 165-176
│   ├── main()                               # lines 178-325
│   │   ├── destructive cleanup              # lines 179-187
│   │   ├── user seeding                     # lines 189-210
│   │   ├── customer seeding                 # lines 212-216
│   │   ├── product seeding                  # lines 218-222
│   │   ├── ordersPlan fixture               # lines 226-253
│   │   ├── flowFromStatus map               # lines 255-262
│   │   └── order generation loop            # lines 264-321
│   └── process bootstrap/error handling     # lines 327-335
├── schema.prisma                            # Data contract (7 models, 2 enums)
└── migrations/
    ├── migration_lock.toml                  # MySQL provider lock
    └── 20260519182739_init/migration.sql    # DDL, constraints, indexes
```

### 4.2 Supporting contracts

| File | Role relative to the seed | Evidence |
|------|--------------------------|----------|
| `prisma/schema.prisma` | Defines every model, field, enum, default, unique constraint, index and relation the seed writes against | `prisma/schema.prisma:11-138` |
| `prisma/migrations/20260519182739_init/migration.sql` | Physical DDL and foreign-key behavior that the cleanup order must respect | `prisma/migrations/20260519182739_init/migration.sql:1-125` |
| `package.json` | Declares `db:seed` and the Prisma seed hook; pins Prisma, bcrypt and tsx versions | `package.json:16,22-24,26-27,48,50` |
| `tsconfig.json` | Includes `prisma/**/*.ts` in type checking | `tsconfig.json:28` |
| `tsconfig.build.json` | Excludes `prisma` from the production build; seed is source-run only | `tsconfig.build.json:9-10` |
| `.env.example` | Documents `DATABASE_URL`, the only variable the seed needs | `.env.example:5` |
| `docker-compose.yml` | Provides the local MySQL 8.0 instance the seed targets in development | `docker-compose.yml:1-28` |
| `.eslintrc.json` | `no-console` warns except `warn`/`error`; explains inline disable comments for `console.log` | `.eslintrc.json:28`, `prisma/seed.ts:179,189,212,218,224,323` |

### 4.3 Internal organization notes

- The module is procedural. There is no exported function, class, interface or type.
- All data is declared as module-level constants (`customersSeed`, `productsSeed`, `ADMIN_PASSWORD`, `OPERATOR_PASSWORD`).
- `main()` is the orchestrator; `reserveOrderNumber()` is the only extracted helper.
- `ordersPlan` and `flowFromStatus` are declared inside `main()`, not at module scope.
- `productsSeed` is referenced by numeric position, creating a positional coupling between two constants.
- There is no separation between fixture data and persistence logic; both are in one file.
- Progress output uses `console.log`; errors use `console.error`, with ESLint disables.

---

## 5. Dependency Analysis

### 5.1 Internal dependencies

The Seed Script imports **zero internal application files**. It does not import `src/config/database.ts`, `src/config/env.ts`, `OrderService`, `OrderRepository`, `order.status.ts`, the error taxonomy, the logger or any Zod schema.

```
prisma/seed.ts -> (no src/ imports)
```

Inbound internal references are also zero at the TypeScript import level. No file in `src/` or `tests/` imports `prisma/seed.ts`. The only inbound references are configuration and tooling:

```
package.json:16        db:seed script -> prisma/seed.ts
package.json:22-24     prisma.seed hook -> prisma/seed.ts
tsconfig.json:28       type-checks prisma/**/*.ts
tsconfig.build.json    excludes prisma from the build
```

The seed does depend on application behavior by duplication rather than import:

```
prisma/seed.ts:165-176  duplicate of src/modules/orders/order.service.ts:245-254
prisma/seed.ts:255-262  partial reimplementation of src/modules/orders/order.status.ts:3-10
prisma/seed.ts:191-192  repeats bcrypt cost from src/modules/users/user.service.ts:16
prisma/seed.ts:287-318  writes the same tables as OrderService without its rules
```

### 5.2 External dependencies

| Dependency | Version | Type | Purpose in seed | Evidence |
|------------|---------|------|-----------------|----------|
| `@prisma/client` | 5.22.0 | Runtime npm package | ORM client, enums, transactions, all queries | `prisma/seed.ts:1,4`; `package.json:26` |
| `bcrypt` | 5.1.1 | Runtime npm package | Hash demo passwords with cost 10 | `prisma/seed.ts:2,191-192`; `package.json:27` |
| MySQL | 8.0 | External database | Persistence target for all seeded rows | `docker-compose.yml:3`; `prisma/schema.prisma:5-9` |
| Prisma CLI | 5.22.0 | Dev tooling | Generates client, can invoke the seed hook | `package.json:48`, `package.json:22-24` |
| `tsx` | 4.19.2 | Dev runner | Executes TypeScript seed source | `package.json:16,50` |
| Node.js | >= 20 | Runtime platform | `--env-file` support and ESM execution | `package.json:7,16` |
| TypeScript | 5.6.3 | Compile-time checker | Type-checks the script via root tsconfig | `package.json:51`, `tsconfig.json:28` |
| ESLint / Prettier | 8.57.1 / 3.3.3 | Dev tooling | Lint and format the script | `package.json:44-47`, `.eslintrc.json` |

### 5.3 Dependency graph

```
package.json db:seed                  package.json prisma.seed hook
        |                                      |
        v                                      v
   tsx --env-file=.env                  tsx (no explicit env flag)
        |                                      |
        +------------------+-------------------+
                           v
                     prisma/seed.ts
                      /            \
                     v              v
            @prisma/client       bcrypt
                     |
                     v
              MySQL 8.0 (DATABASE_URL)
                     ^
                     |
        prisma/schema.prisma + migration.sql
              (compile-time and DDL contract)

No edge to src/.
```

### 5.4 Runtime dependency assumptions

- `DATABASE_URL` must point to a reachable MySQL 8.0 database with the migration already applied.
- The Prisma Client must have been generated from the current schema.
- The process must be able to resolve `tsx` and the TypeScript sources.
- The target database must be disposable, because the script will delete all rows.
- The `bcrypt` native binding must be available for the installed platform.
- The database user must have `DELETE`, `INSERT` and `SELECT` privileges on all seven tables.

---

## 6. Afferent and Efferent Coupling

### 6.1 Methodology

For this component, the "units" are the TypeScript module and its top-level functions/constants. Afferent coupling (Ca) counts distinct internal units that depend on the unit; efferent coupling (Ce) counts distinct internal units the unit depends on. External npm packages and the database are reported separately because they are outside the internal source graph. The seed is a leaf in the internal graph: nothing imports it, and it imports nothing from `src/`.

### 6.2 Coupling table

| Unit | Afferent Coupling | Efferent Coupling | External Dependencies | Criticality |
|------|-------------------|-------------------|-----------------------|-------------|
| Seed Script module (`prisma/seed.ts`) | 0 internal source files; 2 tooling references (`package.json` script and seed hook) | 0 internal source files | `@prisma/client`, `bcrypt` | Medium |
| `main()` | 1 (module bootstrap at `prisma/seed.ts:327`) | 3 internal units (`reserveOrderNumber`, `customersSeed`, `productsSeed`) plus `prisma`, `bcrypt`, `ordersPlan`, `flowFromStatus` | `@prisma/client`, `bcrypt` | High |
| `reserveOrderNumber()` | 1 (`main`) | 1 (`prisma` client) | `@prisma/client` | Medium |
| `customersSeed` | 1 (`main`) | 0 | None | Low |
| `productsSeed` | 1 (`main`) | 0 | None | Medium (positional contract) |
| `ordersPlan` | 1 (`main`) | 0 | `OrderStatus` enum | High (drives all orders) |
| `flowFromStatus` | 1 (`main`) | 0 | `OrderStatus` enum | High (history validity) |
| `package.json` seed script | 0 | 1 (`prisma/seed.ts`) | `tsx` | High |

### 6.3 Interpretation

- **Afferent coupling is effectively zero in source.** The seed is an operational leaf; no runtime module or test depends on it. Its changes cannot break the API at compile time, and API changes do not automatically break it except through the Prisma schema.
- **Efferent coupling to the domain is zero by design, but logical coupling is high.** The script duplicates the order-number algorithm and reimplements the status chain, so behavior can diverge without any import edge. This is the main architectural risk of the component.
- **Positional coupling is significant.** `ordersPlan` depends on `productsSeed` array indices and `main` depends on the `Promise.all` result order. There is no type-level guarantee that index 18 still refers to the intended product.
- **Schema coupling is strong.** Every write is shaped by `prisma/schema.prisma`; a required field change breaks the script at compile time or runtime, and a new restricting foreign key can break the hardcoded cleanup order.
- **Operational coupling is high.** `package.json`, Prisma CLI, `tsx`, Node 20 and MySQL all participate in the execution contract.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 | External relational database | Persist users, customers, products, orders, items, history and sequence | MySQL wire protocol over TCP via Prisma | Relational rows; JSON `address`; enum columns; integer cents | No retry, no rollback; top-level catch logs and exits 1 |
| Prisma Client 5.22.0 | In-process ORM | Query building, nested writes, interactive transaction for sequence allocation | In-process library API | Generated TypeScript types mapped to SQL | Prisma errors propagate to the top-level catch; no classification |
| `order_number_sequence` table | Internal database integration | Serialize order-number allocation | SQL through Prisma transaction | Single row `id=1`, integer `nextValue` | Transaction rolls back on failure; allocated number may be lost if later order create fails |
| bcrypt 5.1.1 | In-process library | Hash the two demo passwords | In-process function call | String plaintext -> bcrypt hash string | Errors propagate to the top-level catch |
| Prisma CLI 5.22.0 | Development tooling | Invoke the seed through `prisma.seed` hook; apply schema before seeding | CLI process spawn | JSON configuration in `package.json`; exit codes | Prisma CLI reports non-zero exit from the seed |
| `tsx` 4.19.2 | TypeScript runner | Execute `prisma/seed.ts` without a build step | CLI process | TypeScript source | Process exits non-zero; raw error printed |
| Node.js `--env-file` | Environment loader | Load `DATABASE_URL` from `.env` for `npm run db:seed` | Process startup flag | `KEY=VALUE` lines | Missing file causes startup failure; missing variable fails at first Prisma query |
| `prisma/schema.prisma` and migrations | Schema contract | Define tables, enums, defaults, unique indexes and foreign keys | Compile-time types plus SQL DDL | Prisma schema and SQL | Schema drift causes compile or runtime errors |
| Docker Compose MySQL | Local infrastructure | Provide the development database | Container runtime | MySQL data volume | Healthcheck in Compose; seed itself has no dependency check |
| Test harness (indirect) | Development tooling | Truncates the same tables before integration tests, but does not run the seed | In-process Prisma | Relational rows | `tests/setup.ts:8-16`; no coupling to seed code |

There is no outbound HTTP call, message broker, queue, email, cache or third-party API integration. MySQL is the only stateful external dependency.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Script entry point with promise chain | `main()` invoked with `.catch().finally()` | `prisma/seed.ts:327-335` | Run a one-shot CLI task with exit code and disconnect |
| Declarative fixture datasets | `customersSeed`, `productsSeed` arrays | `prisma/seed.ts:9-163` | Keep demo data readable and centralized |
| Hand-authored scenario table | `ordersPlan` array | `prisma/seed.ts:226-253` | Describe order statuses, product positions and quantities |
| Table-driven history generation | `flowFromStatus` map plus path expansion | `prisma/seed.ts:255-262`, `prisma/seed.ts:307-320` | Derive valid predecessor chains from a final status |
| Interactive transaction | `prisma.$transaction` around sequence upsert | `prisma/seed.ts:166-175` | Allocate a unique order number atomically |
| Upsert-based sequence allocator | `orderNumberSequence.upsert` with increment | `prisma/seed.ts:167-172` | Bootstrap row `id=1` and increment on each call |
| Reset-and-load idempotency | `deleteMany` then create | `prisma/seed.ts:179-187` | Make repeated runs start from a known empty state |
| Deterministic modulo distribution | `i % customers.length`, `i % 3`, `i % 7` | `prisma/seed.ts:266-267,282` | Spread customers, authors and discounts predictably |
| Parallel bulk insert | `Promise.all` over customer/product arrays | `prisma/seed.ts:213-222` | Reduce wall-clock time for reference data |
| Direct data access / transaction script | Raw Prisma calls, no service or repository | `prisma/seed.ts:194-321` | Keep the standalone script self-contained |
| Graceful teardown | `finally` with `prisma.$disconnect()` | `prisma/seed.ts:333-335` | Release the Prisma connection |
| Console progress logging | `console.log` with inline ESLint disables | `prisma/seed.ts:179,189,212,218,224,323` | Provide simple CLI feedback without the Pino logger |
| Duplicated domain algorithm (anti-pattern) | Local copy of `reserveOrderNumber` | `prisma/seed.ts:165-176` vs `src/modules/orders/order.service.ts:245-254` | Avoid importing application code, at the cost of divergence risk |
| Bypassed domain layer (anti-pattern for fixtures) | No service, state machine or Zod validation | `prisma/seed.ts:287-318` | Write directly to the schema; data may not reflect domain invariants |

### 9.1 Architectural role

The seed is an **operational bootstrap** at the edge of the system. It is not a domain component and does not participate in the API runtime. In a clean-architecture sense it is an adapter that talks directly to the persistence layer. It deliberately avoids the application core so it can run as a standalone process with only Prisma and bcrypt as dependencies.

The trade-off is that the seed is not a reliable specification of the domain. The production behavior for order numbers, status transitions and stock movement lives in `OrderService` and `order.status.ts`; the seed reimplements a subset and omits stock effects. The two must be kept aligned manually.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Critical | Cleanup | Unconditional `deleteMany` on all seven tables with no environment guard or confirmation | Running against a shared or production database destroys data irreversibly |
| High | Credentials | Hardcoded `admin123` / `operator123` in committed source | Known admin credentials if seed touches a non-disposable database |
| High | Atomicity | No transaction spans cleanup and inserts | A failure leaves a partially deleted or partially seeded database |
| High | Inventory consistency | Seeded `PAID`/`PROCESSING` orders never debited stock | Cancelling them through the API increments stock that was never decremented, inflating inventory |
| High | Test coverage | Seed has no unit or integration tests and is not run by `npm test` | Regressions in counts, relationships, credentials or history are undetected |
| Medium | Domain duplication | `reserveOrderNumber` is copied verbatim from `OrderService` | Format or sequence changes must be applied in two places; divergence breaks seed/API consistency |
| Medium | Domain bypass | No service, state machine, Zod or repository use | Seed can create states the API would reject and does not validate fixture invariants |
| Medium | Positional coupling | `ordersPlan.itemIdxs` refer to `productsSeed` array positions | Reordering products silently changes every order composition |
| Medium | History realism | All history actors are the order creator; reasons are generic; `CANCELLED` only from `PENDING` | Demo audit data does not represent real workflows or all allowed cancellation paths |
| Medium | Timestamp ties | All rows share the seed execution window; history/order sorts have no secondary key | Unstable ordering in lists and history when millisecond timestamps tie |
| Medium | Concurrency | Two concurrent seed runs can interleave deletes and inserts | Unique-constraint failures or mixed data from both runs |
| Medium | Documentation drift | Architectural report states 27 orders; actual plan has 26 | Misleading operational documentation and metrics |
| Low | Order-number failure gap | Allocation commits before the order create | A failed order consumes a number and leaves a gap |
| Low | Error handling | `process.exit(1)` in `catch` can skip the asynchronous `finally` disconnect; raw error only | Less graceful cleanup and harder failure diagnosis |
| Low | Type safety of fixtures | `itemIdxs`/`quantities` length equality is unchecked; non-null assertions trust indices | Out-of-range data fails at runtime after earlier writes |
| Low | Data representativeness | No notes, no inactive products, no mixed history actors, no backdated dates | Demo data covers basic shapes but not realistic edge cases |
| Low | Content quality | Product name typo `Caneta Pilot G2 Pretra` | Cosmetic inconsistency in demo data |
| Low | Configuration | bcrypt cost and passwords are fixed; no per-environment overrides | Cannot adapt seeded credentials or cost without editing source |

### 10.1 Security assessment

- **Credential exposure:** The two demo passwords are plaintext constants in a committed file. Anyone who can run the seed can log in as an administrator if the database is reachable by the API.
- **Destructive capability:** The script requires only `DATABASE_URL` and executes deletes immediately. There is no allowlist, no `NODE_ENV` check and no backup. This is the highest-impact operational security risk of the component.
- **PII exposure:** The seed inserts realistic Brazilian names, emails, phone numbers, CPF/CNPJ-shaped documents and full addresses. They are synthetic, but they are committed and will appear in any database dump.
- **No injection surface:** All writes go through Prisma parameterized queries; the script does not build SQL strings.
- **No network egress:** The only network connection is the database. There is no outbound HTTP surface.
- **No secret manager:** `DATABASE_URL` is loaded from `.env` or the process environment; the seed does not read a vault or encrypted configuration.

### 10.2 Resilience assessment

The script has no retry, no backoff, no circuit breaker, no checkpoint and no resume capability. A transient database failure aborts the run after partial writes. The only recovery mechanism is to fix the cause and run the destructive seed again. The finally block is the only cleanup, and it is not guaranteed on the error path because of the immediate `process.exit(1)`.

---

## 11. Test Coverage Analysis

### 11.1 Test inventory

| Test file | Tests | Scope | Relation to seed |
|-----------|-------|-------|------------------|
| `tests/auth.test.ts` | 7 | Registration, duplicate email, validation, login, wrong password, `/auth/me`, unauthenticated access | Creates users through the API or factories, not through the seed; does not use seeded credentials |
| `tests/orders.test.ts` | 8 | Order creation and totals, missing product, `PENDING -> PAID` with stock debit, invalid transition, insufficient stock, cancellation replenishment, status filtering, delete guard | Exercises `OrderService` behavior that the seed reimplements or bypasses; does not execute the seed |
| `tests/setup.ts` | N/A | Connects Prisma and truncates seven tables before each test | Duplicates the table set but not the seed logic |
| `tests/helpers/factories.ts` | N/A | Creates users, customers, products and authenticated tokens for tests | Replaces the seed for test fixtures; does not import seed data |

No test file references `seed`, `admin@oms.local`, `operador@oms.local`, `admin123`, `operator123`, `customersSeed`, `productsSeed`, `ordersPlan` or `reserveOrderNumber` from the seed. `vitest.config.ts:7` includes only `tests/**/*.test.ts`, and no coverage provider, threshold or report is configured.

### 11.2 Coverage assessment

| Seed aspect | Unit Tests | Integration Tests | Direct Coverage | Test Quality |
|-------------|------------|-------------------|-----------------|--------------|
| `main()` end-to-end execution | 0 | 0 | 0% | Not exercised; manual only |
| Table cleanup and re-run | 0 | 0 | 0% | No test for destructive behavior or partial failure |
| Demo user creation | 0 | 0 | 0% | Auth tests use factories, not seeded users |
| bcrypt cost and demo credentials | 0 | 0 | 0% | No assertion that seeded accounts can log in |
| Customer dataset and address shape | 0 | 0 | 0% | No customer tests exist at all |
| Product dataset and SKU uniqueness | 0 | 0 | 0% | No product tests exist at all |
| Order plan counts and status distribution | 0 | 0 | 0% | No assertion on 26 orders or 4/5/4/4/6/3 split |
| Customer assignment (`i % 10`) | 0 | 0 | 0% | Not tested |
| Author assignment (`i % 3`) | 0 | 0 | 0% | Not tested |
| Item pricing and subtotal | 0 | 0 | 0% | API test covers service pricing (`tests/orders.test.ts:12-40`), not the seed copy |
| Discount rule (`i % 7`, 5%) | 0 | 0 | 0% | API test covers a client-supplied discount, not the seed rule |
| Order number reservation copy | 0 | 0 | 0% | API test asserts `^ORD-\d{6}$` (`tests/orders.test.ts:39`) for `OrderService`, not the seed implementation |
| History chain reconstruction | 0 | 0 | 0% | API tests cover service history (`tests/orders.test.ts:59-87,134-161`), not seed history |
| Stock non-adjustment | 0 | 0 | 0% | API tests cover service stock movements; seed bypass is untested |
| Failure path and exit code | 0 | 0 | 0% | Not tested |
| Environment guard | 0 | 0 | 0% | No guard exists and no test asserts one |
| Re-run identity behavior | 0 | 0 | 0% | Not tested |

### 11.3 Test strategy findings

The project uses integration-only testing against a real MySQL instance. `vitest.config.ts:11-16` disables file parallelism and uses a single fork, so tests run serially and share one database. `tests/setup.ts:8-16` truncates the same seven tables before each test, but it does not seed demo data. Factories in `tests/helpers/factories.ts` create the minimal users, customers and products each test needs.

This strategy is appropriate for the API but leaves the seed completely unverified. There is no test that runs `npm run db:seed`, that asserts the resulting counts, that logs in with the seeded credentials, or that checks the seeded status-history chains. Because the seed duplicates the numbering and history logic, the existing API tests cannot detect divergence in the seed copy.

The test suite also has no coverage instrumentation. The absence of a coverage provider means there is no measured percentage for any component, including the seed. The 0% direct coverage stated here is based on the absence of imports or executions of `prisma/seed.ts`, not on a generated coverage report.

---

## 12. Analysis Notes and Limitations

- The component was analyzed from `prisma/seed.ts`, `prisma/schema.prisma`, the initial migration, `package.json`, the TypeScript configuration, the test suite and the existing architectural analysis. No project files were modified.
- The seed was not executed during this analysis. Statements about runtime behavior are derived from the source and from the Prisma/MySQL semantics documented in the schema and migration.
- The count of 26 orders was verified directly from the 26 `ordersPlan` entries at `prisma/seed.ts:227-252`; the existing architectural report's figure of 27 is inaccurate and is recorded as documentation drift.
- The 41 order-item count and 78 history-row count are derived from the plan and the `flowFromStatus` chain lengths.
- The 5 percent discount rule and the modulo distributions are implementation facts. Their business intent is inferred and marked with Medium confidence because the repository contains no written rule explaining why every seventh order receives a discount or why customers and authors are distributed by modulo.
- The exact timezone of database-assigned timestamps depends on the MySQL session configuration; the schema does not declare a timezone policy.
- The report intentionally omits the Endpoints section because `prisma/seed.ts` exposes no REST, GraphQL, gRPC or other endpoint.
