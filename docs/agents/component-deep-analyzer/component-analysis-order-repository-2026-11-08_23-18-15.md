# Component Deep Analysis Report — Order Repository

- Component: `OrderRepository`
- Primary file: `src/modules/orders/order.repository.ts` (69 lines)
- Scope analyzed: order repository, its query construction, eager-loading, pagination/search filters, sequence-allocation boundary and Prisma transaction participation, plus consumers, schema, migrations, seed and tests
- Project root: `.` (entire project)
- Ignored folders: `node_modules`, `.git`, `dist`, `coverage`
- Analysis date: 2026-11-08 23:18:15
- Analysis mode: read-only static inspection; no code executed, no tests run

---

## 1. Executive Summary

`OrderRepository` is the persistence adapter for the Orders aggregate. It is a 69-line class (`src/modules/orders/order.repository.ts:18-69`) that wraps a `PrismaClient` and exposes exactly four methods: `list`, `findByIdWithRelations`, `findById` and `deleteById`. It also exports two structural contracts, `OrderListFilters` (`order.repository.ts:3-10`) and `OrderWithRelations` (`order.repository.ts:12-16`).

Its role in the system is narrow and asymmetrical. It owns the **read** side of the Orders module (filtered, paginated list plus one aggregate detail projection) and the **delete** command. It does **not** own the write path: order creation and status transitions are executed by `OrderService` directly against an interactive transaction client (`src/modules/orders/order.service.ts:58-123`, `131-178`), and order-number allocation lives in a private service method (`order.service.ts:245-254`). The repository therefore participates in Prisma transactions in exactly one place — the array-form `$transaction([findMany, count])` inside `list` (`order.repository.ts:35-43`) — and it structurally cannot participate in any interactive transaction, because its constructor requires the full `PrismaClient` and not `Prisma.TransactionClient` (`order.repository.ts:19`).

Key findings:

1. **Query shape is small, explicit and parameterized.** All filters are composed as a conjunctive Prisma `OrderWhereInput` in `buildWhere` (`order.repository.ts:21-31`); there is no raw SQL, no string interpolation and therefore no injection surface.
2. **Three filter dimensions only:** `status`, `customerId`, `from`/`to` over `createdAt`. There is no `search` term, no `orderNumber` lookup, no `createdById` filter and no sorting parameter.
3. **Pagination is offset-based and delegated.** The repository performs no validation, clamping or defaulting of `skip`/`take`; the `page`/`pageSize` contract is enforced upstream by Zod (`order.schemas.ts:24-25`) and converted by `OrderService.list` (`order.service.ts:38-39`).
4. **Ordering is non-deterministic under ties.** `orderBy: { createdAt: 'desc' }` has no secondary key (`order.repository.ts:38`), and `createdAt` is `DATETIME(3)` millisecond precision. Concurrent or batch-inserted orders can share a timestamp, making page boundaries unstable.
5. **Eager-loading is a 1 + N-relation read shaped by `include`/`select`.** `findByIdWithRelations` loads items with a nested product projection, the full status history in ascending chronological order and a reduced customer projection (`order.repository.ts:47-60`). No `relationLoadStrategy` is specified, so Prisma's default `query` strategy applies.
6. **Sequence allocation is outside this component.** `ORD-nnnnnn` numbers are produced by `OrderService.reserveOrderNumber` via an upsert on the single `order_number_sequence` row (`order.service.ts:245-254`), inside the creation transaction. The repository exposes no create/update method and no sequence method, so it cannot allocate numbers and is not on that code path.
7. **Delete is a hard delete whose children are removed by database cascades** (`prisma/migrations/20260519182739_init/migration.sql:116,122`), with no soft-delete marker and no audit record of the deletion itself.
8. **No test targets this file directly.** Coverage is incidental through the HTTP integration suite (`tests/orders.test.ts`); two of the four public methods (`findByIdWithRelations`, `deleteById`) have no observed test path at all.

Architectural weight is low in code volume but medium in boundary significance: it is the only place in the Orders module where listing semantics, pagination consistency and the aggregate read model are defined, and it is where the repository abstraction stops being applied (writes bypass it entirely).

---

## 2. Data Flow Analysis

### 2.1 List path (the only method that uses a Prisma transaction)

```
1. HTTP client: GET /api/v1/orders?status=&customerId=&from=&to=&page=&pageSize=
2. buildOrderRouter -> router.use(authenticate)                  (order.routes.ts:14)
3. validate({ query: listOrdersQuerySchema })                    (order.routes.ts:16)
   - page coerced, integer, min 1, default 1                     (order.schemas.ts:24)
   - pageSize coerced, integer, min 1, max 100, default 20       (order.schemas.ts:25)
   - status nativeEnum(OrderStatus), optional                    (order.schemas.ts:26)
   - customerId uuid, optional                                    (order.schemas.ts:27)
   - from / to coerced Date, optional                             (order.schemas.ts:28-29)
4. OrderController.list casts req.query and calls service        (order.controller.ts:9-17)
5. OrderService.list computes skip = (page-1)*pageSize, take = pageSize, calls repository (order.service.ts:32-40)
6. OrderRepository.list -> buildWhere composes AND predicate     (order.repository.ts:21-31, 33-34)
7. prisma.$transaction([ order.findMany(...), order.count(...) ]) (order.repository.ts:35-43)
   a. SELECT all order columns FROM orders WHERE <and-predicate>
      ORDER BY createdAt DESC LIMIT take OFFSET skip
   b. SELECT COUNT(*) FROM orders WHERE <same and-predicate>
8. Repository returns { items, total }                            (order.repository.ts:44)
9. OrderService.list wraps with paginated(items, page, pageSize, total) (order.service.ts:41)
10. Controller responds 200 with { data, pagination: {page,pageSize,total,totalPages} }
```

Notes on this path:

- `buildWhere` returns the **same object instance** that is later handed to both `findMany` and `count`; there is no second construction and no risk of predicate drift between page and total.
- The `where.createdAt` container is created only when at least one bound is supplied (`order.repository.ts:25-29`), so an unfiltered list does not send an empty `createdAt: {}` predicate.
- No `include`/`select` is used in `list`, so only `Order` scalar columns are read; relations are never loaded for list rows.

### 2.2 Detail path

```
1. GET /api/v1/orders/:id (authenticate, orderIdParamSchema uuid param) (order.routes.ts:17)
2. OrderController.getById -> OrderService.getById                (order.controller.ts:19-26, order.service.ts:44-48)
3. OrderRepository.findByIdWithRelations(id)                      (order.repository.ts:47-60)
4. prisma.order.findUnique({ where: { id }, include: {...} })
   - order row
   - items (full OrderItem) -> product (id, sku, name only)
   - history ordered by changedAt ASC
   - customer (id, name, email only)
5. null -> NotFoundError('Order') at service layer                (order.service.ts:46)
6. 200 with the composite object (no envelope, no serialization layer)
```

Prisma's default relation load strategy is `query` (no `relationLoadStrategy` is set and the `relationJoins` preview feature is not enabled in `prisma/schema.prisma:1-3`), so this aggregate read resolves into multiple SQL statements issued by the query engine rather than one joined statement: the order, the items, the products referenced by those items, the history and the customer. The exact batching is engine-internal and therefore stated here as a multi-statement read rather than a guaranteed statement count.

### 2.3 Delete path

```
1. DELETE /api/v1/orders/:id (authenticate, uuid param)           (order.routes.ts:24)
2. OrderController.delete -> OrderService.delete                  (order.controller.ts:48-55, order.service.ts:181-192)
3. orders.findById(id) -> scalar Order row                        (order.repository.ts:62-64)
4. Guard in service: status must be PENDING or CANCELLED          (order.service.ts:184-190)
5. orders.deleteById(id) -> prisma.order.delete({ where: { id } }) (order.repository.ts:66-68)
6. InnoDB cascades: order_items and order_status_history rows     (migration.sql:116,122)
7. void return; controller responds 204
```

Two service-layer facts bound this path: the existence/state check at step 4 happens **outside** any transaction, and the delete at step 5 is a standalone statement with no accompanying `$transaction`, so the check and the delete are not atomic with respect to a concurrent status change. This is documented as a risk in section 10.

### 2.4 Sequence allocation path (owned by the service, not the repository)

```
OrderService.create -> prisma.$transaction(async (tx) => {
  validate customer (tx.customer.findUnique)                       (order.service.ts:59-60)
  validate products, active flag                                   (order.service.ts:62-73)
  compute item prices, subtotal, discount, total                   (order.service.ts:75-92)
  reserveOrderNumber(tx):
    tx.orderNumberSequence.upsert({ where:{id:1},
      create:{id:1,nextValue:2}, update:{nextValue:{increment:1}},
      select:{nextValue:true} })                                   (order.service.ts:246-251)
    current = nextValue - 1
    return `ORD-${String(current).padStart(6,'0')}`                (order.service.ts:252-253)
  tx.order.create({ ...items, history })                           (order.service.ts:95-120)
})
```

The repository is absent from this flow. It holds no reference to the transaction client, defines no `create` method and no sequence method. Its only relationship to sequence allocation is negative: because it cannot accept `Prisma.TransactionClient`, it cannot be reused inside this transaction, which is the mechanical reason the service writes `tx.order.create` inline.

### 2.5 Interactive surface

`OrderRepository` exposes **no network endpoints** (see section 7). All inbound calls are in-process method calls from `OrderService`, and all outbound I/O is Prisma/MySQL.

---

## 3. Business Rules & Logic

Rules are stated as implemented, not as intended. Where a rule is implicit (no comment, no ADR, no test), the evidence and a confidence level are given in the detailed breakdown.

### 3.1 Overview of the business rules

| Rule ID | Rule Type | Rule Description | Location |
|---------|-----------|------------------|----------|
| BR-01 | Query semantics | List filters are conjunctive (AND); only `status`, `customerId`, `createdAt` range are supported | `order.repository.ts:21-31` |
| BR-02 | Query semantics | Date range is inclusive on both ends (`gte` / `lte`) and uses the `createdAt` column only | `order.repository.ts:25-29` |
| BR-03 | Query semantics | An unsatisfiable range (from > to) silently yields an empty result, not an error | `order.repository.ts:25-29` |
| BR-04 | Pagination | Offset pagination is delegated: repository applies `skip`/`take` verbatim and performs no defaulting, clamping or validation | `order.repository.ts:39-40` |
| BR-05 | Pagination | `total` is the cardinality of the filtered set, independent of the page window | `order.repository.ts:42` |
| BR-06 | Ordering | List order is `createdAt` descending only; no deterministic tiebreaker exists | `order.repository.ts:38` |
| BR-07 | Consistency | Page rows and total count are read inside one batch transaction | `order.repository.ts:35-43` |
| BR-08 | Projection | List returns bare `Order` scalar rows; no relations, no field pruning | `order.repository.ts:36-41` |
| BR-09 | Eager loading | Detail read is a fixed aggregate: items + product mini-projection + full history + customer mini-projection | `order.repository.ts:47-60` |
| BR-10 | Ordering | Status history is returned in ascending `changedAt` order | `order.repository.ts:56` |
| BR-11 | Data minimization | Nested customer/product reads expose only whitelisted attributes | `order.repository.ts:53,57` |
| BR-12 | Lookup | `findById` returns scalar order or `null`; never throws for a missing row | `order.repository.ts:62-64` |
| BR-13 | Deletion | Delete is physical; child rows are removed by FK cascade | `order.repository.ts:66-68`, `migration.sql:116,122` |
| BR-14 | Deletion | Repository enforces no state precondition; PENDING/CANCELLED guard is a service rule | `order.service.ts:184-190` |
| BR-15 | Access scope | No ownership or tenant scoping: any authenticated caller can read any order | `order.repository.ts:21-31` |
| BR-16 | Filter values | Filter identifiers are trusted; an unknown `customerId` produces an empty page, not a 404 | `order.repository.ts:24,34` |
| BR-17 | Transaction boundary | Repository accepts only root `PrismaClient`; it cannot join an interactive transaction | `order.repository.ts:19` |
| BR-18 | Write ownership | Repository exposes no create/update for orders; order-number allocation is service-owned | `order.repository.ts:18-69`, `order.service.ts:95-120,245-254` |

---

### 3.2 Detailed breakdown of the business rules

---

### Business Rule: BR-01 — Conjunctive filter semantics with three dimensions

**Overview**:
Every optional list filter narrows the result set and all supplied filters must hold simultaneously. Unsupplied filters are omitted from the predicate rather than neutralized with `undefined`.

**Detailed description**:

`buildWhere` (`order.repository.ts:21-31`) starts from an empty `Prisma.OrderWhereInput` and conditionally assigns `status` (line 23), `customerId` (line 24) and a `createdAt` range object (lines 25-29). Because each assignment is a distinct top-level key on the same object, Prisma composes them as `AND`. The resulting filter space is exactly three-dimensional; there is no free-text `search` (unlike customers and products, which do have `search` at `customer.repository.ts:12-22` and `product.repository.ts:13-23`), no `orderNumber` filter, no `createdById` filter and no multi-status selection.

The guard style is truthiness (`if (filters.status)`, `if (filters.customerId)`), which is safe here because valid values are non-empty enum strings and UUIDs enforced upstream by Zod (`order.schemas.ts:26-27`). The practical consequence for API consumers is that `GET /api/v1/orders?status=PENDING&customerId=<uuid>&from=...&to=...` returns the intersection, and that an unsupported parameter such as `?orderNumber=ORD-000001` is silently ignored (it survives Zod as an unknown key and never reaches the repository).

This rule is the primary extensibility seam of the component. Adding a filter requires a new optional field on `OrderListFilters` (`order.repository.ts:3-10`), a new branch in `buildWhere`, and a new optional field in `listOrdersQuerySchema` (`order.schemas.ts:23-30`). Because the repository receives an already-parsed object, it has no knowledge of pagination defaults and cannot distinguish "filter absent" from "filter explicitly undefined".

**Rule workflow**:
```
filters -> {} 
  if status      -> where.status = status
  if customerId  -> where.customerId = customerId
  if from or to  -> where.createdAt = { gte?, lte? }
-> Prisma AND of all assigned keys -> applied identically to findMany and count
```

---

### Business Rule: BR-02 — Inclusive, `createdAt`-only date range

**Overview**:
`from` translates to `createdAt >= from` and `to` translates to `createdAt <= to`; both bounds are inclusive and both apply to the order creation timestamp only.

**Detailed description**:

The implementation builds `where.createdAt = {}` and then sets `gte` and/or `lte` (`order.repository.ts:25-29`). Either bound may appear alone: `from`-only yields an open-ended lower bound, `to`-only an open-ended upper bound, and both yields a closed interval. There is no `updatedAt`, `paidAt` or `shippedAt` alternative, because those timestamps do not exist on the `Order` model (`prisma/schema.prisma:74-97` records only `createdAt` and `updatedAt`).

Inclusivity has a concrete reporting consequence. A caller asking for `to=2026-11-08` after Zod coercion (`z.coerce.date()` at `order.schemas.ts:29`) produces a `Date` at `2026-11-08T00:00:00.000Z`; `lte` then includes only orders created exactly at or before midnight UTC, excluding the whole business day. Callers must therefore send an explicit end-of-day timestamp or use `to` as "start of next day" semantics conceptually, which the schema does not document. Additionally, Prisma writes `createdAt` as UTC into a MySQL `DATETIME(3)` column, so comparisons are UTC-based while MySQL session time zone is not part of this component's contract.

The rule is implemented with `gte`/`lte` rather than Prisma's `in`-range or raw SQL, so index usage is left entirely to MySQL's optimizer. The `orders_createdAt_idx` index (`migration.sql:66`) can serve a pure date range, but when `status` or `customerId` co-exist there is no composite index, and the planner must choose between filtering on the selective equality column and sorting, or scanning the date index and filtering. No `EXPLAIN` output exists in the repository to confirm which plan is used (confidence: medium, based on schema only).

**Rule workflow**:
```
from only -> createdAt >= from
to only   -> createdAt <= to
both      -> from <= createdAt <= to
neither   -> no createdAt predicate at all (key not added)
```

---

### Business Rule: BR-03 — Unsatisfiable ranges degrade to an empty result

**Overview**:
If `from` is later than `to`, the query is still executed and returns zero rows with `total = 0`; no validation error is raised at any layer.

**Detailed description**:

`listOrdersQuerySchema` validates each bound independently (`order.schemas.ts:28-29`) but never compares them. `buildWhere` likewise assigns both bounds without comparison (`order.repository.ts:27-28`). Prisma emits `createdAt >= from AND createdAt <= to`; when `from > to` this predicate is unsatisfiable and MySQL returns an empty set. The API surface then reports HTTP 200 with `data: []` and `pagination.total: 0`.

The business interpretation is ambiguous: an inverted range could mean "no data exists", "the client made a mistake", or "the client intended the reverse interval". The system chooses the first interpretation implicitly. This is consistent with the API router analysis of the same schema (`order.schemas.ts:23-30` referenced from the router report) and is not contradicted anywhere in the codebase. No test asserts either behavior (confidence: high that behavior is as described; the intent is undocumented).

From a client-contract standpoint the consequence is that inverted ranges are indistinguishable from genuinely empty result sets, which can mask front-end bugs. Because the same predicate is reused for `count` (BR-05), the reported total is coherent with the empty page, so no internal inconsistency is produced.

**Rule workflow**:
```
from > to -> predicate impossible -> findMany: 0 rows, count: 0 -> 200 { data: [], pagination.total: 0 }
```

---

### Business Rule: BR-04 — Pagination is offset-based and fully delegated to the caller

**Overview**:
The repository applies `skip` and `take` exactly as received. It defines no defaults, no maximum, no minimum and no conversion from page numbers.

**Detailed description**:

`OrderListFilters` requires `skip` and `take` as mandatory numbers (`order.repository.ts:8-9`), and `list` passes them directly into Prisma (`order.repository.ts:39-40`). The page contract lives one layer up: Zod coerces `page` to an integer `>= 1` with default `1` and `pageSize` to an integer in `[1,100]` with default `20` (`order.schemas.ts:24-25`), and `OrderService.list` computes `skip = (page - 1) * pageSize`, `take = pageSize` (`order.service.ts:38-39`). Consequently the repository sees only two non-negative integers and has no concept of `page`.

This split is deliberate layering but creates a latent boundary weakness: any internal caller other than `OrderService.list` could pass `skip = -1` or `take = 0` and receive a Prisma validation error surfaced as HTTP 500 by the error middleware (`error.middleware.ts:56-64` only special-cases `P2002`/`P2025`). In the current codebase there is exactly one caller, so the weakness is theoretical. There is also no cursor/keyset pagination and no upper bound on `skip`, so a deep page issues a large `OFFSET`, which MySQL must scan and discard, producing latency proportional to the offset.

Because `pageSize <= 100` is enforced by Zod, the maximum rows per response is bounded, but the number of pages is not. A caller can walk arbitrarily deep into the result set at increasing cost. This rule therefore couples the repository's performance profile to an upstream schema constraint, not to anything the repository itself enforces.

**Rule workflow**:
```
page, pageSize (Zod: 1..∞, 1..100, defaults 1/20)
  -> skip=(page-1)*pageSize, take=pageSize (service)
  -> ORDER BY createdAt DESC LIMIT take OFFSET skip (repository, verbatim)
```

---

### Business Rule: BR-05 — `total` is filtered cardinality, not page size

**Overview**:
The returned `total` counts every row matching the filters, regardless of `skip`/`take`, and is produced by an independent `count` statement with the identical predicate.

**Detailed description**:

`list` destructures a two-element batch transaction `[findMany, count]` and returns `{ items, total }` (`order.repository.ts:35-44`). The `where` variable is the same object passed to both operations, so the count cannot drift from the page predicate. The service then feeds `total` into `paginated()` (`order.service.ts:41`), where `totalPages = Math.ceil(total / pageSize)` and `pageSize === 0` is special-cased to `0` (`src/shared/http/response.ts:13-20`).

The business meaning is that `total` answers "how many orders match my filters", not "how many are on this page". Two behavioural edge cases follow. First, when `skip` exceeds `total`, `data` is empty but `total` is non-zero and `totalPages` still reflects the full set — a client must not infer emptiness of the domain from an empty page. Second, `count` is executed for every list request, so each list costs a second statement whose cost grows with the number of matching rows, and there is no approximate-count path or caching.

The count is not guarded by any authorization scope either (BR-15), so it is an oracle for the size of the entire orders table when no filters are supplied. Query-parameter combination is the only thing that narrows it; `pageSize` does not.

**Rule workflow**:
```
where -> count(*) => total
items(page) + total -> paginated() => { data, pagination:{ page, pageSize, total, totalPages } }
```

---

### Business Rule: BR-06 — `createdAt` descending with no deterministic tiebreaker

**Overview**:
List ordering is `createdAt DESC` only. Orders sharing the same millisecond timestamp are returned in an order the database does not guarantee to be stable across queries.

**Detailed description**:

`orderBy: { createdAt: 'desc' }` (`order.repository.ts:38`) refers to a `DATETIME(3)` column (`migration.sql:60`), i.e. millisecond resolution. The sequence seed creates 26 orders in a tight loop (`prisma/seed.ts:264-321`), and the integration test creates three orders back-to-back (`tests/orders.test.ts:169-176`), so ties are realistic rather than theoretical. MySQL is free to return tied rows in any order, and it may choose differently depending on the access path (status index scan versus createdAt index scan), which means the same request can return the same row twice or omit a row across two pages.

No secondary sort on `id` or `orderNumber` exists in the repository, the service, or the schema (sort parameters are not accepted anywhere; the router analysis confirms no endpoint takes a sort argument). The consequence is that offset pagination — already fragile under concurrent inserts — is additionally fragile under timestamp ties, independently of concurrency.

Business-wise, "newest first" is the only expressed intent. There is no "oldest first", no sort by `orderNumber`, no sort by `totalCents` and no sort by status. Clients needing any other order must fetch and sort locally, subject to `pageSize <= 100`.

**Rule workflow**:
```
SELECT ... WHERE <filters> ORDER BY createdAt DESC LIMIT take OFFSET skip
  ties -> undefined relative order -> unstable pagination boundaries
```

---

### Business Rule: BR-07 — Page and total are read in one batch transaction

**Overview**:
`list` is the component's single use of `prisma.$transaction`, in array form, wrapping exactly two read statements so that the page and its total observe the same database state.

**Detailed description**:

`this.prisma.$transaction([findMany, count])` (`order.repository.ts:35-43`) sends both operations through one transaction using Prisma's sequential batch form. The intent is read consistency: without it, a concurrent insert between the two statements could produce a page whose `total` does not include the newly visible row, or vice versa. Prisma does not set an explicit isolation level here, so consistency rests on the MySQL default (`REPEATABLE READ` for InnoDB), which is sufficient for a consistent snapshot within the transaction (confidence: high for the mechanism, medium for the exact server configuration since the datasource URL carries no isolation override).

The transaction is read-only and short, containing no row locks beyond what the storage engine uses for consistent reads. It is not an interactive transaction, so it does not hold a Prisma pool connection across user code and is not subject to the 5-second interactive-transaction timeout. It does, however, consume a connection for the duration of the `count`, which on an unfiltered list scans the whole `orders` table.

A subtle consequence is that `total` and `items` are simultaneous by construction but not by business requirement — nothing in the API contract documents snapshot semantics. If a future change moves `count` out of the transaction (for performance), the contract would silently weaken without any test failing, because no test exercises concurrent writes during a list.

**Rule workflow**:
```
$transaction([ findMany(where, orderBy, skip, take), count(where) ])
  -> single BEGIN/COMMIT -> consistent snapshot -> { items, total }
```

---

### Business Rule: BR-08 — List projection is the bare order row

**Overview**:
List responses carry every scalar column of `orders` and no related entities. No `select` is used, so the shape is exactly the Prisma `Order` type.

**Detailed description**:

`findMany` in `list` (`order.repository.ts:36-41`) specifies `where`, `orderBy`, `skip`, `take` and nothing else. Prisma therefore returns all scalar fields: `id`, `orderNumber`, `customerId`, `status`, `subtotalCents`, `discountCents`, `totalCents`, `notes`, `createdById`, `createdAt`, `updatedAt` (`prisma/schema.prisma:74-97`). The declared return type `Order[]` matches that shape. Clients that need items or customer data must issue a second request per order against `GET /:id`, which is the classic N+1 at the API level.

This is a deliberate thin-read choice: list queries stay cheap (no joins, no relation batching), at the cost of client-side round trips. It also means list rows expose `notes` (free text up to `TEXT`) and `createdById`, which are not needed to render a table. Since `pageSize` is capped at 100, the payload is bounded but can still be large if notes are long.

The type `Order` is imported as a type-only import from `@prisma/client` (`order.repository.ts:1`), so the shape is not duplicated locally and cannot drift from the schema. That is the main structural strength of this rule.

**Rule workflow**:
```
findMany(where, orderBy, skip, take) -> Order[] (all scalar columns) -> total separately
```

---

### Business Rule: BR-09 — Fixed aggregate contract for the detail read

**Overview**:
`findByIdWithRelations` always loads exactly three relations — `items` (with a nested product view), `history` and `customer` — and always in the same shape. There is no projection parameter or include toggle.

**Detailed description**:

The `include` block (`order.repository.ts:50-58`) loads items with a nested `product` selection of `id`, `sku`, `name` (lines 51-55), the history ordered ascending (line 56) and the customer with `id`, `name`, `email` (line 57). The return type `OrderWithRelations` (`order.repository.ts:12-16`) encodes exactly this shape, asserting that `items` is an array of `OrderItem` augmented with a non-null `product` object and that `customer` is a non-null reduced object. That assertion is valid because `order_items.productId` is a required FK to `products` (`migration.sql:119`) and `orders.customerId` is a required FK (`migration.sql:110`).

Because the aggregate is invariant, the consumer (`OrderService.getById`, `order.service.ts:44-48`) and the controller pass it straight to the client with no mapping step. The same literal `include` block is duplicated inside `OrderService.create` (`order.service.ts:115-119`) and `OrderService.changeStatus` (`order.service.ts:169-176`), meaning three separate definitions of the same read shape exist in the module. `OrderWithRelations` is imported by the service (`order.service.ts:7`), so the type is shared, but the runtime `include` literals are not.

The read is a multi-statement operation under Prisma's default `query` relation-load strategy (no `relationLoadStrategy: 'join'` and no `relationJoins` preview flag in `prisma/schema.prisma:1-3`). For a single order this is normally four to five statements: order, items, products for those items, history, customer. That is acceptable for a detail endpoint and would be unacceptable if reused inside a list loop — which is precisely why `list` does not include relations.

**Rule workflow**:
```
findUnique(id)
  -> order
  -> items (all OrderItem columns) + product(id, sku, name)
  -> history (all columns, changedAt ASC)
  -> customer (id, name, email)
  -> null if not found (service maps to 404)
```

---

### Business Rule: BR-10 — History is chronological ascending

**Overview**:
Status history is always returned oldest-first by `changedAt`, which reflects the order of state transitions as an audit trail.

**Detailed description**:

`history: { orderBy: { changedAt: 'asc' } }` (`order.repository.ts:56`) orders the audit rows from the initial `PENDING` entry (created with `fromStatus: null` at `order.service.ts:106-113`) through each subsequent transition. The test suite asserts the resulting sequence `['PENDING','PAID']` indirectly through the create/status response (`tests/orders.test.ts:80-83`), though those responses are produced by the service's own `include`, not by this method.

`changedAt` is `DATETIME(3)` with a default of `CURRENT_TIMESTAMP(3)` (`migration.sql:92`), and there is no secondary ordering key. Two transitions recorded within the same millisecond could therefore be returned in either order. In practice transitions require separate HTTP requests and stock updates, so ties are rare; the ordering is deterministic enough for the domain but not guaranteed by contract.

There is no pagination or limiting of history. An order that cycles through many transitions returns all rows. Because the state machine has six statuses and forbids backward moves (`order.status.ts:3-10`), the history length is bounded in practice by a handful of rows, so the absence of a limit is not a current performance risk.

**Rule workflow**:
```
history WHERE orderId = id ORDER BY changedAt ASC -> array (all rows)
```

---

### Business Rule: BR-11 — Attribute minimization on nested relations

**Overview**:
Nested customer and product reads expose a whitelist of attributes; no other columns of those tables are loaded.

**Detailed description**:

The product projection is `{ id, sku, name }` (`order.repository.ts:53`) and the customer projection is `{ id, name, email }` (`order.repository.ts:57`). Notably absent are `products.priceCents`/`stockQuantity`/`active` and `customers.phone`/`document`/`address`. Historical line-item pricing is preserved on `order_items.unitPriceCents`/`totalCents`, so the detail view is self-sufficient for monetary display without joining current product prices — a correct denormalization for order history.

The data-minimization effect is also a privacy control: `customers.document` (tax identifier) and `customers.address` (JSON) do not leak through the order detail endpoint, even though the order belongs to that customer. The repository returns `null` customer projection only if the FK were violated, which the database prevents.

One asymmetry is worth noting: the `createdBy` relation (the user who created the order) is never loaded, so the detail response exposes `createdById` as a raw UUID but no author name or email. This avoids exposing `users.passwordHash`, deliberately or incidentally; the safest reading is that the author relation was simply not needed by the UI (confidence: medium).

**Rule workflow**:
```
items -> product.select(id, sku, name)
customer -> select(id, name, email)
(no user/createdBy include)
```

---

### Business Rule: BR-12 — Scalar identity lookup returns `null`, never throws

**Overview**:
`findById` is a plain unique lookup that returns the `Order` row or `null`; absence is not an exception at this layer.

**Detailed description**:

`findById` calls `prisma.order.findUnique({ where: { id } })` and returns its result directly (`order.repository.ts:62-64`). Prisma's `findUnique` returns `null` for a missing row rather than raising `P2025`, so the repository's `Order | null` return type is honest. The consumer decides semantics: `OrderService.delete` converts `null` into `NotFoundError('Order')` (`order.service.ts:182-183`), while `OrderService.create`/`changeStatus` use their own `tx.order.findUnique` calls and never call this method.

No relations are loaded, so this is the cheapest read in the component — appropriate for a precondition check whose only fields of interest are `id` and `status`. Because it reads the row outside any transaction in the delete path, the status it observes can be stale by the time `deleteById` runs (see BR-14 and section 10).

The method is also the only place where a missing row is a normal outcome rather than an error, which makes it the correct primitive for optimistic checks. No test calls it directly; it is exercised only through the refused-delete test (`tests/orders.test.ts:197-218`).

**Rule workflow**:
```
findUnique({ id }) -> Order | null
  null -> caller decides (service: 404)
```

---

### Business Rule: BR-13 — Physical delete with cascading children

**Overview**:
`deleteById` issues a hard `DELETE` on the order row. `order_items` and `order_status_history` children disappear through `ON DELETE CASCADE`; nothing is tombstoned.

**Detailed description**:

`prisma.order.delete({ where: { id } })` (`order.repository.ts:66-68`) removes the row physically. The schema declares `onDelete: Cascade` on `OrderItem.order` (`prisma/schema.prisma:108`) and `OrderStatusHistory.order` (`prisma/schema.prisma:125`), materialized in the migration as `ON DELETE CASCADE` (`migration.sql:116,122`). The repository therefore relies on database referential actions rather than issuing child deletes itself — a correct choice that keeps the method one statement, but one that makes the destructive scope invisible at the call site.

No audit of the deletion is written. `OrderStatusHistory` records status changes only; there is no `deletedAt`, no `deletedBy`, no `OrderDeletionLog`. Once deleted, the order's number, totals, items and history are unrecoverable except from backups. Because `orderNumber` is unique but the sequence never decrements, a deleted number is never reissued, so external references to `ORD-nnnnnn` become permanently dangling.

Foreign keys from other tables into `orders` are limited to children with cascade, so the delete cannot fail due to dependents. It can fail only if the row is already absent, in which case Prisma raises `P2025`, which the error middleware maps to 404 `NOT_FOUND` (`error.middleware.ts:48-53`).

**Rule workflow**:
```
delete({ id })
  -> row absent: Prisma P2025 -> middleware 404
  -> row present: DELETE orders; InnoDB cascades order_items, order_status_history
  -> void (no row count, no audit)
```

---

### Business Rule: BR-14 — Delete preconditions belong to the service, not the repository

**Overview**:
The repository deletes anything it is given. The rule "only PENDING or CANCELLED orders may be deleted" is implemented in `OrderService.delete` and is not enforceable at the persistence boundary.

**Detailed description**:

`OrderService.delete` reads the order with `orders.findById`, rejects `PAID`, `PROCESSING`, `SHIPPED` and `DELIVERED` with a `ConflictError` carrying code `INVALID_ORDER_STATE_FOR_DELETE` (`order.service.ts:181-190`), and only then calls `orders.deleteById`. The repository has no status predicate in its `delete` call (Prisma only permits unique `where` on `delete`), so it cannot express "delete if status in (...)" as a single atomic statement.

This split creates a concurrency window: between the read at `order.service.ts:182` and the delete at line 191, another request could move the order from `PENDING` to `PAID`. The delete would then remove a paid order, bypassing the guard. The window is not theoretical under concurrent traffic, but no locking, `updateMany` with status predicate or `deleteMany`-with-guard pattern exists to close it.

The test suite verifies the refusal path only (`tests/orders.test.ts:197-218`); the permitted deletion path is never exercised, so neither the cascade behavior nor a successful 204 is asserted anywhere.

**Rule workflow**:
```
findById -> status in {PENDING, CANCELLED}? 
  no  -> 409 INVALID_ORDER_STATE_FOR_DELETE (no delete issued)
  yes -> deleteById -> 204
(no atomicity between check and delete)
```

---

### Business Rule: BR-15 — Global read scope, no ownership filter

**Overview**:
No query in the repository is scoped by the authenticated user, role, tenant or customer relationship. Any caller who passes the route's `authenticate` middleware can read any order.

**Detailed description**:

`buildWhere` supports `customerId` only as an optional client-supplied filter; it is never injected from `req.user`. `findByIdWithRelations` and `findById` look up by primary key with no additional predicate. Consequently an `OPERATOR` can read any customer's orders, and an `ADMIN` has no broader or narrower view — the only access control is "has a valid JWT", applied once at `router.use(authenticate)` (`order.routes.ts:14`).

`createdById` is written on creation (`order.service.ts:104`) and indexed (`migration.sql:67`) but never used as a filter, so the index currently serves no query in this component. This suggests an intended "my orders" capability that does not exist.

The business meaning is a shared operational back office rather than a multi-tenant or customer-facing storefront. No code, comment or ADR states this explicitly, so the rule is inferred from uniform absence of scoping (confidence: high on behavior, medium on intent). The security consequence — horizontal access to all order data by any authenticated user — is recorded in section 10.

**Rule workflow**:
```
authenticate (JWT) -> no role check, no ownership predicate -> all orders visible
```

---

### Business Rule: BR-16 — Filter identifiers are trusted; unknown values yield empty sets

**Overview**:
The repository does not verify that a `customerId` (or any filter value) exists. A well-formed but unknown identifier returns an empty page, not an error.

**Detailed description**:

`where.customerId = filters.customerId` (`order.repository.ts:24`) is a plain equality predicate against a `CHAR(36)` FK column. If the UUID is syntactically valid (guaranteed by `order.schemas.ts:27`) but refers to no customer, the query simply matches nothing. There is no `customer.findUnique` pre-check, and none is expected at the persistence layer — `OrderService.list` does not add one either (`order.service.ts:32-42`).

This differs sharply from the write path, where `OrderService.create` explicitly checks the customer and raises `NotFoundError('Customer')` (`order.service.ts:59-60`). The asymmetry is defensible for collection endpoints (an empty result is a valid answer to "orders for this customer") but means a typo'd `customerId` is indistinguishable from a customer with no orders. Clients cannot tell which case they hit from the response body.

Because the value is bound as a parameter by Prisma, there is no injection risk, and because the column is `CHAR(36)`, collation/trailing-space quirks are minimal. The rule is entirely about semantics, not safety.

**Rule workflow**:
```
customerId valid UUID + unknown -> WHERE customerId = ? -> 0 rows, total 0, HTTP 200
```

---

### Business Rule: BR-17 — The repository cannot join an interactive transaction

**Overview**:
The constructor requires the full `PrismaClient` (`order.repository.ts:19`). Prisma's `Prisma.TransactionClient` is structurally a reduced client, so a repository instance can never be constructed from a `tx` handle.

**Detailed description**:

`constructor(private readonly prisma: PrismaClient) {}` (`order.repository.ts:19`) fixes the dependency type at the root client. `Prisma.TransactionClient` omits `$connect`, `$disconnect`, `$on`, `$transaction`, `$use` and `$extends`, so it is not assignable to `PrismaClient`; TypeScript (project uses `strict: true`, `tsconfig.json`) rejects passing `tx` into this constructor. There is no overload, no optional second constructor accepting `Prisma.TransactionClient`, and no generic client parameter.

The visible consequence is that `OrderService` holds **two** dependencies: the repository and the root client (`order.service.ts:26-30`). It uses the repository for list/detail/delete and reaches into `tx.customer`, `tx.product`, `tx.order`, `tx.orderStatusHistory` and `tx.orderNumberSequence` for the transactional commands (`order.service.ts:58-123`, `131-178`, `204-254`). The repository abstraction is therefore only half-applied: reads and deletes go through it; creates, status updates and stock mutations do not.

This is the single most consequential structural fact about the component. It is not a bug in isolation — a repository typed against `PrismaClient` is a common pattern — but combined with the service's need for atomic multi-table writes, it forces the service to duplicate the aggregate read shape (`order.service.ts:115-119,169-176`) and to own sequence allocation, which is where the write-path risks concentrate (section 10).

**Rule workflow**:
```
new OrderRepository(prisma)      -> allowed (app.ts:42)
new OrderRepository(tx)          -> type error (TransactionClient lacks root members)
=> all transactional writes bypass the repository
```

---

### Business Rule: BR-18 — The repository is not the order write path and does not allocate sequence numbers

**Overview**:
`OrderRepository` exposes no `create` and no `update`. Order creation, number allocation, status transition, stock movement and history append are all performed by `OrderService` through the transaction client. The repository's only relation to order numbers is that it never touches them.

**Detailed description**:

The class surface is exactly `list`, `findByIdWithRelations`, `findById`, `deleteById` (`order.repository.ts:33-68`). There is no `save`, `create`, `update`, `updateStatus` or `reserveOrderNumber`. The `ORD-nnnnnn` generator is `OrderService.reserveOrderNumber` (`order.service.ts:245-254`): an upsert on `order_number_sequence` row `id = 1` with `create: { id: 1, nextValue: 2 }` and `update: { nextValue: { increment: 1 } }`, returning `nextValue`; the allocated number is `nextValue - 1` formatted with `padStart(6, '0')`. On an empty table the first allocation returns `2 - 1 = 1` → `ORD-000001`; subsequent allocations increment before read, yielding 2, 3, and so on. The seed script contains a byte-identical copy of this logic (`prisma/seed.ts:165-176`), so the numbering rule exists in two places.

Transaction participation is therefore highly asymmetric. The repository uses Prisma's **array** transaction once (`list`, read-only). The **interactive** transactions that perform writes never route through it; they hold a single connection for their full duration and serialize all order creation on the single sequence row's exclusive lock, which InnoDB holds until commit. Numbering is gap-free across rolled-back attempts (the increment rolls back with the transaction) but monotonic into the future (deleted orders' numbers are never reused).

The business rules that follow from this arrangement are: `orderNumber` is unique (`migration.sql:63`), immutable at the repository level (no update method can change it), allocated only inside a transaction, and formatted as `ORD-` plus at least six digits. Numbers above 999999 naturally expand to seven digits through `padStart` (it only pads, never truncates), and the `VARCHAR(20)` column accommodates them. Nothing prevents a future direct `prisma.order.create` from supplying an arbitrary number, because uniqueness is the only database constraint — the formatting rule has no `CHECK` constraint and lives solely in the service method (confidence: high; verified absence of check constraints in `migration.sql`).

**Rule workflow**:
```
OrderService.create
  -> $transaction(interactive)
      -> reserveOrderNumber(tx): upsert sequence id=1, increment, read
      -> format ORD-<nextValue-1 padded to 6>
      -> tx.order.create({ orderNumber, ... })   [repository not involved]
OrderRepository
  -> list / findById / findByIdWithRelations / deleteById only
```

---

## 4. Component Structure

```
src/
├── modules/
│   └── orders/
│       ├── order.repository.ts        # ANALYZED COMPONENT (69 lines)
│       │   ├── import type { Order, OrderItem, OrderStatus, OrderStatusHistory, Prisma, PrismaClient }  (line 1)
│       │   ├── export type OrderListFilters        (lines 3-10)  status?, customerId?, from?, to?, skip, take
│       │   ├── export type OrderWithRelations      (lines 12-16) Order + items(product) + history + customer
│       │   └── export class OrderRepository        (lines 18-69)
│       │       ├── constructor(prisma: PrismaClient)                  (line 19)
│       │       ├── private buildWhere(...)                            (lines 21-31)
│       │       ├── list(filters)                                      (lines 33-45)
│       │       ├── findByIdWithRelations(id)                          (lines 47-60)
│       │       ├── findById(id)                                       (lines 62-64)
│       │       └── deleteById(id)                                     (lines 66-68)
│       ├── order.service.ts           # consumer; owns writes + sequence allocation
│       ├── order.controller.ts        # HTTP adapter; calls service only
│       ├── order.routes.ts            # route table + authenticate + Zod validation
│       ├── order.schemas.ts           # list query contract (page/pageSize/filters)
│       └── order.status.ts            # state machine (not used by repository)
├── config/
│   └── database.ts                    # Prisma singleton passed into the repository
├── shared/
│   └── http/response.ts               # { data, pagination } envelope used by the service
└── app.ts                             # composition: new OrderRepository(prisma) (line 42)

prisma/
├── schema.prisma                      # Order / OrderItem / OrderStatusHistory / OrderNumberSequence
├── migrations/20260519182739_init/migration.sql   # indexes + FK cascade actions
└── seed.ts                            # duplicate order-number allocator (lines 165-176)

tests/
├── orders.test.ts                     # 8 integration tests; only incidental repository coverage
├── setup.ts                           # truncates orders + sequence before each test
└── helpers/factories.ts               # builds the app with the shared Prisma singleton
```

Boundary observation: the component is a single file with no internal sub-structure, no helpers and no configuration. Everything it needs is injected (`prisma`) or imported as a type. All of its operational semantics are therefore visible in 69 lines, which makes the component easy to audit and equally easy to orphan — as happened for the write path.

---

## 5. Dependency Analysis

### 5.1 Internal dependencies (compile-time)

```
src/app.ts:16 (import) ──┐
                         ├──> OrderRepository ──(type-only)──> @prisma/client
src/modules/orders/      │
  order.service.ts:7 ────┘   (type-only import of OrderRepository, OrderWithRelations)
  order.service.ts:28        (constructor injection)
  order.service.ts:33,45,182,191  (runtime calls)

src/app.ts:42 ──> new OrderRepository(prisma)   ← only instantiation site
src/config/database.ts ──> prisma singleton ──┘
```

Consumers by method:

| Method | Consumers | Call site |
|--------|-----------|-----------|
| `list` | `OrderService.list` | `order.service.ts:33-40` |
| `findByIdWithRelations` | `OrderService.getById` | `order.service.ts:45` |
| `findById` | `OrderService.delete` | `order.service.ts:182` |
| `deleteById` | `OrderService.delete` | `order.service.ts:191` |

No other module imports the repository. `OrderController` depends on `OrderService`, never on the repository (`order.controller.ts:2`). There are no circular dependencies at the module level.

### 5.2 Internal dependencies of the order-number path (for boundary clarity)

```
OrderService ──> tx.customer          (create validation)
             ──> tx.product           (create validation, stock debit/replenish)
             ──> tx.order             (create, status update, re-read)
             ──> tx.orderStatusHistory (append audit)
             ──> tx.orderNumberSequence (allocate number)
```

None of these pass through `OrderRepository`, which is the boundary gap described in BR-17/BR-18.

### 5.3 External dependencies

| Dependency | Version | Usage | Coupling |
|------------|---------|-------|----------|
| `@prisma/client` | 5.22.0 (`package.json:26`) | `PrismaClient`, `Prisma.OrderWhereInput`, `Order`, `OrderItem`, `OrderStatus`, `OrderStatusHistory` types; runtime query engine | Hard, compile-time and runtime |
| `prisma` (CLI) | 5.22.0 (dev) | Schema/migration generation, client generation | Build-time only |
| MySQL 8.0 | — (docker-compose, `DATABASE_URL`) | Storage for `orders`, `order_items`, `order_status_history`, `customers`, `products` | Hard, runtime, over TCP |
| Node.js >= 20 | `package.json:8` | Runtime (`ES2022` target, ESM) | Platform |

The component itself imports nothing from `src/shared`, nothing from `src/config` and nothing from other feature modules. It is a leaf in the internal dependency graph and depends outward only on generated Prisma artifacts.

### 5.4 Database dependencies exercised by the component

| Table | Operation | Columns involved |
|-------|-----------|------------------|
| `orders` | `findMany`, `count`, `findUnique`, `delete` | all scalars in list/detail; `id` for lookups/delete |
| `order_items` | read (via `include`), cascade delete | `orderId`, all item columns |
| `products` | read (nested select), cascade-restrict on delete | `id`, `sku`, `name` |
| `order_status_history` | read (via `include`), cascade delete | `orderId`, `changedAt`, all history columns |
| `customers` | read (nested select) | `id`, `name`, `email` |

---

## 6. Afferent and Efferent Coupling

Method and type granularity (TypeScript classes + exported type aliases, which are the "components" under the OO/structural paradigm used here):

| Component | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Critical | Notes |
|-----------|------------------------|------------------------|----------|-------|
| `OrderRepository` (class) | 2 (`app.ts`, `order.service.ts`) | 1 (`@prisma/client`) | Medium | Leaf in the internal graph; only instantiation site is composition root |
| `OrderRepository.list` | 1 (`OrderService.list`) | 4 (`buildWhere`, `prisma.order.findMany`, `prisma.order.count`, `$transaction`) | Medium | Only transaction participant |
| `OrderRepository.findByIdWithRelations` | 1 (`OrderService.getById`) | 1 (`prisma.order.findUnique`) | Low | Untested path |
| `OrderRepository.findById` | 1 (`OrderService.delete`) | 1 (`prisma.order.findUnique`) | Low | Precondition read |
| `OrderRepository.deleteById` | 1 (`OrderService.delete`) | 1 (`prisma.order.delete`) | High | Destructive; cascades; untested |
| `buildWhere` (private) | 1 (`list`) | 0 | Medium | Sole location of filter semantics |
| `OrderListFilters` (type) | 1 (`order.service.ts`) | 0 | Medium | Contract for pagination/filtering |
| `OrderWithRelations` (type) | 1 (`order.service.ts`) | 3 (`Order`, `OrderItem`, `OrderStatusHistory`) | Medium | Duplicated at runtime by service `include` literals |

Metrics for the class as a whole: Ca = 2, Ce = 1, instability `I = Ce / (Ca + Ce) = 1/3 ≈ 0.33`. The component is more depended upon than depending, which is the expected position for a persistence adapter: schema changes propagate inward to it and its consumers, while changes to the service do not propagate outward.

Cohesion: high. All four methods operate on the `Order` aggregate, all use the same injected client, and `buildWhere` exists solely to serve `list`. There is no unrelated behavior, no logging, no mapping and no validation in the class. The single cohesion weakness is that `findById` (a precondition primitive) and `findByIdWithRelations` (a presentation-shaped aggregate) serve different concerns under the same class name, which is acceptable at this size.

Coupling direction is clean (composition root → repository → Prisma). The architectural defect is not in the repository's own coupling but in the **missing** afferent edge from the transactional write path: `OrderService` should depend on the repository for order persistence but instead depends directly on `tx.order` and `tx.orderNumberSequence`. That makes the repository's Ca artificially low (2 instead of ~4) and moves schema coupling into the service.

---

## 7. Endpoints

`OrderRepository` exposes **no endpoints**. It is an in-process persistence adapter with no router, no controller, no HTTP/gRPC/GraphQL surface and no serialization of its own. Per the analysis guidelines, an endpoint inventory is therefore not applicable to this component.

For traceability only, the consuming HTTP endpoints and the methods they reach are listed below. These endpoints belong to the Orders module surface, not to the repository.

| Consuming endpoint | Method | Repository method reached | Tested |
|--------------------|--------|---------------------------|--------|
| `/api/v1/orders` | GET | `list` | Yes (`tests/orders.test.ts:163-195`, status filter only) |
| `/api/v1/orders/{id}` | GET | `findByIdWithRelations` | No |
| `/api/v1/orders/{id}` | DELETE | `findById` + `deleteById` | Partially (refusal path only, `tests/orders.test.ts:197-218`) |

`POST /api/v1/orders` and `PATCH /api/v1/orders/{id}/status` do not reach the repository at all.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| MySQL 8.0 (`orders`, `order_items`, `order_status_history`, `customers`, `products`) | External stateful service | Persistent read/write of order data | MySQL wire protocol over TCP via Prisma query engine (`DATABASE_URL`) | Relational rows mapped to generated TypeScript types; `DATETIME(3)`, `CHAR(36)`, `INT`, `TEXT`, `ENUM` | No catch in repository; `P2025` → 404, `P2002` → 409, all other Prisma errors → 500 with generic body (`error.middleware.ts:37-64`) |
| Prisma Client 5.22.0 | In-process library | Query construction, relation loading, batch transaction | Direct function calls | `Prisma.OrderWhereInput`, `Order`, `OrderWithRelations` | Exceptions propagate to the Express error middleware |
| `OrderService` | Internal caller (upstream) | Business orchestration; converts `null` to 404, enforces delete guard, computes pagination | Direct method calls | Plain TypeScript objects | Service maps `null` → `NotFoundError`; repository never returns domain errors |
| `OrderRepository` → `Prisma.TransactionClient` | **Absent integration** | Interactive transactions for create/status change | N/A — type barrier | N/A | N/A (writes bypass the repository) |

Observed properties of these integration points:

- **No retry, no timeout, no circuit breaker.** A slow or unavailable MySQL produces a Prisma error that surfaces as HTTP 500. There is no fallback, no read replica routing and no cached read path.
- **No connection management in the repository.** The client is a shared singleton created in `src/config/database.ts:4-10`; the repository neither connects nor disconnects.
- **No environment branching.** The repository reads no configuration. Logging levels for Prisma are decided in the provider (`database.ts:6`), not here.
- **Transaction isolation is implicit.** The batch transaction sets no `isolationLevel`; consistency depends on the MySQL server default.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Repository | `OrderRepository` class wrapping Prisma delegates | `order.repository.ts:18-69` | Isolate query construction from domain logic; one seam for order persistence |
| Query Object / Predicate Builder | `buildWhere` returning `Prisma.OrderWhereInput` | `order.repository.ts:21-31` | Compose optional filters without conditional query strings |
| Dependency Injection (constructor) | `constructor(private readonly prisma: PrismaClient)` | `order.repository.ts:19`; wired at `app.ts:42` | Explicit, testable dependency; single composition root |
| Data Transfer / Read Model | `OrderWithRelations` type + `include`/`select` projections | `order.repository.ts:12-16, 47-60` | Fixed aggregate shape for the detail endpoint |
| Unit of Work (batch/read-only variant) | `prisma.$transaction([findMany, count])` | `order.repository.ts:35-43` | Snapshot-consistent page + total |
| Offset Pagination / Page envelope | `skip`/`take` + `{ data, pagination }` | `order.repository.ts:39-40`; `src/shared/http/response.ts:13-24` | Uniform list contract across modules |
| Command separation (partial CQRS-lite) | Reads and delete in repository; commands in service | `order.service.ts:50-192` | Writes need multi-table atomicity; reads do not |
| Type-only imports | `import type { ... } from '@prisma/client'` | `order.repository.ts:1` | Zero runtime dependency on generated types; erased at compile time |

Architectural decisions evidenced by the component:

1. **Persistence technology is Prisma, not an abstraction over Prisma.** The repository is a façade, not a technology-neutral port: it exposes `Prisma.OrderWhereInput` internally and Prisma-generated entity types externally. Replacing the ORM would change the type surface of the service.
2. **Layering is layered-by-feature, not hexagonal.** There is no domain entity class; `Order` is the Prisma model. Business rules live in the service and state machine, and the repository holds only query semantics.
3. **Reads are shaped for the UI, not normalized.** The detail read returns customer and product fragments, while the list read returns raw rows. There is no separate assembler layer.
4. **Transactions are owned by the consumer, not the adapter.** This is why the repository has a client dependency rather than a transaction-aware interface.
5. **No caching layer, no read/write split, no second-level cache.** Every request hits MySQL.

Patterns considered and **not** present: Specification (filters are inline conditionals), Soft Delete / Temporal Table, Optimistic Concurrency (no version column), Cursor Pagination, Outbox, Event Emission, Audit Interceptor, Result/Either error type.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Transaction boundary (BR-17/BR-18) | Repository accepts only `PrismaClient`, so the service bypasses it for every write; aggregate read shape is duplicated in three places (`order.repository.ts:50-58`, `order.service.ts:115-119`, `169-176`) | Repository abstraction is half-applied; schema changes must be propagated manually to service `include` literals; the `OrderWithRelations` type can pass review while runtime shapes drift |
| High | `deleteById` + service guard (BR-14) | Status check (`order.service.ts:182-190`) and delete (`order.repository.ts:66-68`) are not atomic; no transaction, no `SELECT ... FOR UPDATE`, no conditional delete | A concurrent PENDING→PAID transition can be followed by a successful delete of a paid order; stock and audit rows are destroyed with it |
| High | `deleteById` (BR-13) | Hard delete with `ON DELETE CASCADE`, no soft-delete flag, no deletion audit record | Irreversible loss of order, items and history; dangling `ORD-nnnnnn` references; no forensic trail |
| Medium | Ordering (BR-06) | `orderBy createdAt DESC` with no tiebreaker on a `DATETIME(3)` column | Duplicate or skipped rows across pages when timestamps tie; query-plan-dependent ordering |
| Medium | Pagination (BR-04) | Offset pagination with unbounded `skip`; `pageSize` capped only by upstream Zod | Deep pages cost an O(offset) scan; `pageSize`/`page` contract is not enforced at the repository boundary, so internal misuse yields 500 |
| Medium | `list` performance (BR-05) | `count(*)` with the same predicate on every request, no caching, no approximate count | Doubles statements per list; unfiltered count scans the whole `orders` table; scales linearly with table growth |
| Medium | Index strategy | Separate single-column indexes on `status`, `customerId`, `createdAt` (`migration.sql:64-66`) but no composite index for the common `(status, createdAt)` or `(customerId, createdAt)` combinations | MySQL must choose one index and filter/sort the rest; `ORDER BY createdAt DESC` combined with an equality filter can force a filesort (confidence: medium, no `EXPLAIN` evidence in repo) |
| Medium | Access scope (BR-15) | No ownership/tenant predicate; `createdById` is stored and indexed but never queried | Any authenticated user can enumerate all orders; no "my orders" capability despite the available index |
| Medium | Test coverage | No unit tests for the repository; `findByIdWithRelations` and `deleteById` unreached; `customerId`, `from`, `to`, inverted ranges, deep pages untested | Regressions in filters, eager-loading or delete cascade would not be caught by the suite |
| Low | Date semantics (BR-02) | `to` is inclusive at the supplied instant; a bare date (`YYYY-MM-DD`) means UTC midnight | Same-day orders silently excluded; timezone ambiguity for callers |
| Low | Empty-range semantics (BR-03) | `from > to` returns 200 with an empty page | Client mistakes are indistinguishable from genuinely empty data |
| Low | Detail read cost (BR-09) | No `relationLoadStrategy: 'join'`; default `query` strategy emits multiple statements for one order | Extra round trips per detail request; bounded, not a scaling risk at current sizes |
| Low | List payload (BR-08) | No `select` on list; `notes` (TEXT) and `createdById` always returned | Unnecessary payload weight up to 100 rows per page |
| Low | Filter trust (BR-16) | Unknown `customerId` yields empty page instead of 404 | Ambiguous client feedback; no data corruption |
| Low | Duplicated sequence logic | `reserveOrderNumber` exists verbatim in both `order.service.ts:245-254` and `prisma/seed.ts:165-176` | Two implementations of a numbering rule that must stay byte-compatible; a formatting change in one place only breaks seed/production consistency |
| Low | Error surface | `delete` on a missing row relies on `P2025` mapping; the repository does not normalize it | If middleware mapping changes, an expected 404 becomes a 500 |

Code smells observed (no remediation implied): duplicated `include` literal (shotgun surgery), primitive obsession in `skip`/`take` (no value object), and a partial abstraction boundary (inappropriate intimacy between `OrderService` and the `order_number_sequence` table). Complexity is otherwise trivial: `buildWhere` has three independent branches, `list` is a single expression, and there is no nesting deeper than two levels.

Security review of the component:

- **SQL injection: not applicable.** All values are bound parameters through Prisma; no raw SQL (`$queryRaw`) exists in the file.
- **Sensitive data exposure: limited but non-zero.** Nested reads whitelist `customers`/`products` fields (BR-11) and never load `users`, so `passwordHash` cannot leak. However list rows expose `notes` and `createdById`, and the detail exposes the customer's email to any authenticated caller.
- **Authorization: absent at this layer** (BR-15), by design of the surrounding routes.
- **Resource exhaustion: partially mitigated.** `pageSize <= 100` at the schema bounds page size, but `skip`, `count` cost and detail fan-out are unbounded.
- **Resilience: none.** No retry, timeout, circuit breaker or fallback; database failures surface as generic 500s.

---

## 11. Test Coverage Analysis

### 11.1 Test inventory

| Test file | Tests | Relevant to repository |
|-----------|-------|------------------------|
| `tests/orders.test.ts` | 8 `it` blocks | 3 exercise repository methods indirectly (list, findById, and creation flows that do not use it) |
| `tests/auth.test.ts` | — | None |
| `tests/setup.ts` | Setup/teardown | Truncates `orderStatusHistory`, `orderItem`, `order`, `orderNumberSequence`, `product`, `customer`, `user` before each test (`tests/setup.ts:8-16`) |
| `tests/helpers/factories.ts` | Factories | Builds the app with the shared Prisma singleton (`factories.ts:10-15`), creating users, customers and products directly via Prisma |
| `vitest.config.ts` | Config | `include: ['tests/**/*.test.ts']`, `singleFork: true`, `fileParallelism: false`, 30 s timeouts, `setupFiles: ['tests/setup.ts']` |

No test file targets `order.repository.ts` directly, there is no mocked `PrismaClient`, and no coverage provider is installed (`@vitest/coverage-v8` is absent from `package.json:35-53`), so percentages below are estimates derived from code paths, not instrumented measurements.

### 11.2 Coverage by repository member

| Component / method | Unit Tests | Integration Tests | Estimated Coverage | Test Quality |
|--------------------|------------|-------------------|--------------------|--------------|
| `OrderRepository` (module load, constructor) | 0 | 8 (every test builds the app, `factories.ts:12`) | High (construction only) | Instantiation is implicitly exercised; no assertion on wiring |
| `buildWhere` | 0 | 1 (`?status=PAID`, `tests/orders.test.ts:183-188`) | ~25% branches (status branch only; `customerId`, `from`, `to`, combined, inverted all unexercised) | No negative/edge cases; no direct assertions on predicate |
| `list` | 0 | 1 (`tests/orders.test.ts:163-195`) | ~60% statements, ~30% branches | Asserts `data.length` and `pagination.total`; does not assert ordering, page 2+, `pageSize`, or totalPages |
| `findByIdWithRelations` | 0 | 0 | 0% | `GET /api/v1/orders/:id` has no test at all; eager-load shape, history ordering and projections are unverified |
| `findById` | 0 | 1 (via refused delete, `tests/orders.test.ts:212-217`) | Coverage achieved, but only the "exists with PAID status" path | No test of the `null` branch (404 delete of unknown id) |
| `deleteById` | 0 | 0 | 0% | Successful delete, cascade of items/history and 204 response are never asserted; only the 409 refusal (which never reaches the method) is covered |
| Types `OrderListFilters`, `OrderWithRelations` | 0 | Compile-time only | N/A | Enforced by `tsc` (`strict: true`), not by runtime tests |

Aggregate estimate for the file: roughly **50-55% of statements** and **25-30% of branches** exercised, with **0%** of the component's behavior directly asserted. Confidence: medium — no instrumentation was run, and the estimate is based on path tracing of the eight integration tests.

### 11.3 Behavioral gaps that matter

1. **No date-range test.** `from`/`to` are the only non-enum filters with non-trivial semantics (inclusivity, UTC midnight, inverted ranges). None is exercised.
2. **No `customerId` filter test.** The predicate branch at `order.repository.ts:24` is dead in the suite.
3. **No pagination-depth test.** `page=2`, `pageSize=1`, `pageSize=101`, `page=0` are untested; `skip` arithmetic at `order.service.ts:38` is therefore unverified beyond `page=1`.
4. **No ordering assertion.** Ties and DESC semantics are unverified; the three-order setup at `tests/orders.test.ts:169-176` could create identical `createdAt` values without the test noticing.
5. **No successful delete.** The cascade contract (`migration.sql:116,122`) and the 204 response are unverified; a regression that changed `onDelete` to `Restrict` would not be caught.
6. **No detail-endpoint test.** Item/product/customer projections, `history` ordering and the 404 path are unverified.
7. **Sequence behavior is only asserted indirectly** by `expect(res.body.orderNumber).toMatch(/^ORD-\d{6}$/)` (`tests/orders.test.ts:39`); monotonicity across creations, rollback behavior on failed creation and the second-allocation value are untested. `tests/setup.ts:12` resets the sequence between tests, so each test starts at `ORD-000001` and no test observes increment behavior across a suite boundary.

### 11.4 Test-harness properties affecting this component

- Tests are true integration tests against a real MySQL database using the production Prisma singleton (`tests/setup.ts:2`), so they exercise the real SQL and real cascades — a strength for anything they touch.
- `beforeEach` truncation is global and ordered child-before-parent (`tests/setup.ts:8-16`), which is compatible with the cascade rules but means no test can rely on data from another test, including the sequence counter.
- `fileParallelism: false` and `singleFork: true` (`vitest.config.ts`) serialize test files, avoiding cross-file interference on the shared tables.
- The factories bypass the repository for user/customer/product creation (`factories.ts:22-73`), so repository code is not involved in fixture setup.

---

## 12. Assumptions, Ambiguities and Confidence Levels

| Topic | Statement | Confidence |
|-------|-----------|------------|
| Sequence ownership | Allocation is service-owned; repository has no sequence method | High (code-verified) |
| Interactive-transaction exclusion | `Prisma.TransactionClient` is not assignable to `PrismaClient`, so the constructor blocks transactional reuse | High (type structure; project uses `strict: true`) |
| Relation load strategy | Prisma default `query` strategy applies because no `relationLoadStrategy` and no `relationJoins` preview flag exist | Medium (based on Prisma 5.22 defaults; statement count is engine-internal) |
| Index plans | No composite index; possible filesort for `status` + `createdAt` ordering | Medium (no `EXPLAIN` output available; not executed) |
| Transaction isolation | Batch transaction relies on MySQL default isolation for snapshot consistency | Medium (no explicit `isolationLevel` in code; server config not inspected) |
| Date interpretation | `createdAt` comparisons are UTC-based; `to=<date>` means midnight UTC | High for the JS/Zod side; medium for stored-value timezone assumptions |
| Access scope intent | Global visibility is the implemented behavior; whether it is intended is not documented (PRD/FDD are templates) | High on behavior, low on intent |
| Test coverage percentages | Estimates from path tracing; no coverage tooling configured | Medium |

No fabricated business rules were introduced. Where the code is silent (for example, whether `from > to` should be a 400), the report states the implemented behavior and marks the intent as undocumented.

---

## 13. Report Metadata

- Files inspected: `src/modules/orders/order.repository.ts`, `order.service.ts`, `order.controller.ts`, `order.routes.ts`, `order.schemas.ts`, `order.status.ts`, `src/app.ts`, `src/config/database.ts`, `src/shared/http/response.ts`, `src/middlewares/error.middleware.ts`, `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql`, `prisma/seed.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `vitest.config.ts`, `tsconfig.json`, `package.json`
- No files were modified
- No code was executed and no tests were run
