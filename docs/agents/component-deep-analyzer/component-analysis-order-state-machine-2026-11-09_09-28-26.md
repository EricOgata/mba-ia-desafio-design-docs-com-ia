# Component Deep Analysis Report - Order State Machine

| Attribute | Value |
|-----------|-------|
| Component | Order State Machine |
| Primary file | `src/modules/orders/order.status.ts` (37 lines) |
| Language / runtime | TypeScript 5.6.3, Node.js >= 20, ESM |
| Component type | Pure domain module (transition policy), no I/O, no state |
| Primary consumer | `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`) |
| Analysis date | 2026-11-09 09:28:26 |
| Scope | Entire project root, excluding `node_modules`, `.git`, `dist`, `coverage` |
| Files inspected | `src/modules/orders/order.status.ts`, `order.service.ts`, `order.schemas.ts`, `order.routes.ts`, `order.controller.ts`, `order.repository.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts`, `src/middlewares/error.middleware.ts`, `src/modules/auth/auth.service.ts` (context only), `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql`, `prisma/seed.ts`, `tests/orders.test.ts`, `tests/auth.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`, `vitest.config.ts`, `package.json`, `tsconfig.json`, `docker-compose.yml`, `README.md`, `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md` |
| Method | Static reading only. No code was executed, no tests were run, no files were modified. |

---

## 1. Executive Summary

The Order State Machine is a 37-line, side-effect-free TypeScript module that defines the legal lifecycle of an order. It exports two lookup functions over a compile-time transition table (`canTransition`, `allowedTransitions`), one derived predicate (`isTerminal`), one pair constant (`STOCK_DEBIT_TRANSITION`) and two stock side-effect predicates (`shouldDebitStock`, `shouldReplenishStock`). The module owns policy only. Enforcement, persistence, audit writing and inventory movement all live in `OrderService.changeStatus`.

The lifecycle has six states taken from the Prisma `OrderStatus` enum: `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`. The table declares seven legal transitions out of the 30 possible non-self ordered pairs and 36 including self-pairs: `PENDING -> PAID`, `PENDING -> CANCELLED`, `PAID -> PROCESSING`, `PAID -> CANCELLED`, `PROCESSING -> SHIPPED`, `PROCESSING -> CANCELLED`, `SHIPPED -> DELIVERED`. `DELIVERED` and `CANCELLED` have empty successor arrays and are therefore terminal.

Key findings:

1. The table is typed as `Record<OrderStatus, ReadonlyArray<OrderStatus>>` (`order.status.ts:3`), so the TypeScript compiler requires one entry per enum member. Adding a status to `prisma/schema.prisma` without updating this file produces a compile error rather than a silent runtime gap.
2. The module has the lowest mechanical coupling in the orders module: one internal afferent dependency (`order.service.ts:17-21`) and one external dependency (`@prisma/client` for the enum). Instability is 0.5 and abstractness 0, giving a distance from the main sequence of 0.5 for the file as a unit.
3. Stock policy is expressed as a second, independent rule set (`order.status.ts:24-37`) rather than as a property of the transition table. Consistency between "which edges are legal" and "which edges move stock" is maintained by convention only; no type or test ties the two together.
4. Two of the six exports, `allowedTransitions` and `isTerminal`, have zero consumers in the project. Terminality is enforced in practice by `canTransition`, not by `isTerminal`.
5. The consumer evaluates the table against an unlocked read of `order.status` and then issues an unconditional `UPDATE` (`order.service.ts:131-158`). Two concurrent requests can both observe the same source state, both pass `canTransition`, and both apply stock effects and history rows. This is a state-machine-level concurrency gap, not merely a persistence detail.
6. The matrix exists only in application code. MySQL constrains the vocabulary through an `ENUM` column (`migration.sql:54`) but has no `CHECK` constraint on transitions, and the seed script writes orders directly in any status (`prisma/seed.ts:226-252`) using a fabricated history graph that contradicts the matrix (`prisma/seed.ts:255-261`).
7. No transition event is published. The state machine and its consumer have no outbox or event emission, so the only externally observable effect of a transition is the returned order aggregate and the `OrderStatusHistory` row.
8. Rejected transitions leave no trace in logs: `errorMiddleware` returns `AppError` responses before reaching its `logger.error` call (`error.middleware.ts:15-24`), and the state machine itself has no telemetry.

---

## 2. Data Flow Analysis

### 2.1 Entry path

The component has no entry point of its own. It is reached synchronously, in-process, from one service method. The complete path from the network boundary to the module is:

```
1. PATCH /api/v1/orders/{id}/status arrives at the Express app
2. authenticate middleware attaches req.user                    (order.routes.ts:14)
3. validate({ params: orderIdParamSchema, body: updateOrderStatusSchema })
      - id must be a UUID                                       (order.schemas.ts:4)
      - toStatus must be z.nativeEnum(OrderStatus)               (order.schemas.ts:18-21)
      - reason optional, max 500 chars                           (order.schemas.ts:20)
4. OrderController.changeStatus requires req.user                (order.controller.ts:38-46)
5. OrderService.changeStatus(id, input, userId)                  (order.service.ts:126)
6. prisma.$transaction begins                                    (order.service.ts:131)
7. tx.order.findUnique({ where: { id }, include: { items: true } })
      -> from = order.status                                     (order.service.ts:132-138)
8. same-status guard: from === to -> 409 ConflictError           (order.service.ts:140-146)
9. canTransition(from, to)                                       (order.service.ts:147)
      -> transitions[from].includes(to)                          (order.status.ts:12-14)
      -> false -> InvalidStatusTransitionError -> 409             (order.service.ts:148)
10. shouldDebitStock(from, to)                                   (order.service.ts:151)
      -> true only for (PENDING, PAID)                           (order.status.ts:29-31)
      -> debitStock(tx, items) reads products, checks availability,
         throws InsufficientStockError (422) or decrements       (order.service.ts:204-231)
11. shouldReplenishStock(from, to)                               (order.service.ts:154)
      -> true only for (PAID|PROCESSING, CANCELLED)              (order.status.ts:33-37)
      -> replenishStock increments each product                  (order.service.ts:233-243)
12. tx.order.update({ data: { status: to } })                    (order.service.ts:158)
13. tx.orderStatusHistory.create({ fromStatus, toStatus,
      changedById: userId, reason })                             (order.service.ts:159-167)
14. tx.order.findUnique with items, history, customer            (order.service.ts:169-176)
15. transaction commits; 200 with refreshed aggregate            (order.controller.ts:42)
```

### 2.2 What the state machine contributes to the flow

Within that sequence, `order.status.ts` is consulted at exactly three points:

| Call site | Function | Input | Output | Effect on flow |
|-----------|----------|-------|--------|----------------|
| `order.service.ts:147` | `canTransition` | `from`, `to` | `boolean` | Stops the transaction with HTTP 409 if false |
| `order.service.ts:151` | `shouldDebitStock` | `from`, `to` | `boolean` | Selects the inventory decrement branch |
| `order.service.ts:154` | `shouldReplenishStock` | `from`, `to` | `boolean` | Selects the inventory increment branch |

All three are total functions over two enum values. They read module-level constants only, allocate no objects, perform no I/O, and cannot throw for in-vocabulary inputs. Their output depends on nothing else: not on order totals, item count, customer, actor, role, timestamp, feature flag or configuration. This makes the transition decision reproducible for a given `(from, to)` pair for the whole process lifetime.

### 2.3 Branch outcome map

```

                      +---------------------------+
                      | from === to ?             |
                      +---------------------------+
                          | yes                 | no
                          v                     v
                409 INVALID_STATUS_TRANSITION   canTransition(from, to)
                "Order is already in X status"  |
                                                |
                            +-------------------+-------------------+
                            | false                                 | true
                            v                                       v
            409 INVALID_STATUS_TRANSITION                stock predicates
            "Invalid status transition                     |
             from X to Y"                                  |
                                                 +---------+---------+
                                                 |                   |
                                        shouldDebitStock      shouldReplenishStock
                                        (PENDING -> PAID)     (PAID|PROCESSING -> CANCELLED)
                                                 |                   |
                                        check stock then       increment stock
                                        decrement stock              |
                                                 |                   |
                                                 +---------+---------+
                                                           v
                                                 UPDATE status to `to`
                                                           v
                                                 INSERT history row
                                                           v
                                                 re-read + 200
```

### 2.4 Ordering and rollback semantics

The legality check runs before any side effect, and all side effects run inside one interactive Prisma transaction. If `debitStock` finds a shortage, it collects every unavailable SKU, throws `InsufficientStockError` (`order.service.ts:211-224`) and the transaction rolls back, so neither the status nor any product is modified. `tests/orders.test.ts:109-132` asserts this by requiring HTTP 422, `INSUFFICIENT_STOCK`, the offending SKU in `details.unavailable`, and an unchanged stock quantity of 1.

Because the status `UPDATE` happens after the stock movement, a successful debit is always followed by a status write in the same transaction. Conversely, an illegal transition is rejected before the stock predicates are even evaluated, so a request such as `PENDING -> SHIPPED` cannot touch inventory. The final re-read (`order.service.ts:169-176`) occurs inside the transaction, so the response reflects the just-written status and history.

### 2.5 Exit paths

| Exit | Condition | HTTP | Body |
|------|-----------|------|------|
| Success | Legal transition, sufficient stock | 200 | Refreshed order with items, history, customer |
| Order missing | `findUnique` returns null | 404 | `NOT_FOUND` |
| Same status | `from === to` | 409 | `INVALID_STATUS_TRANSITION`, details `{ from, to }` |
| Illegal edge | `canTransition` false | 409 | `INVALID_STATUS_TRANSITION`, details `{ from, to }` |
| Stock shortage | Debit branch finds insufficient stock | 422 | `INSUFFICIENT_STOCK`, details `unavailable[]` |
| Unexpected failure | Any other thrown error | 500 | `INTERNAL_SERVER_ERROR` (logged) |

---

## 3. Business Rules & Logic

### 3.1 Transition matrix

Rows are source states (`from`), columns are target states (`to`). `YES` means the edge is present in `transitions` (`src/modules/orders/order.status.ts:3-10`).

| from \ to | PENDING | PAID | PROCESSING | SHIPPED | DELIVERED | CANCELLED |
|-----------|---------|------|------------|---------|-----------|-----------|
| PENDING | NO | YES | NO | NO | NO | YES |
| PAID | NO | NO | YES | NO | NO | YES |
| PROCESSING | NO | NO | NO | YES | NO | YES |
| SHIPPED | NO | NO | NO | NO | YES | NO |
| DELIVERED | NO | NO | NO | NO | NO | NO |
| CANCELLED | NO | NO | NO | NO | NO | NO |

Counts: 7 allowed edges, 29 forbidden edges, of which 6 are self-edges and 23 are forbidden non-self edges.

### 3.2 Overview of the business rules

| ID | Rule Type | Rule Description | Location |
|----|-----------|------------------|----------|
| BR-01 | Definition | The lifecycle vocabulary is exactly the six `OrderStatus` values, and the table must cover every one of them | `prisma/schema.prisma:16-23`; `src/modules/orders/order.status.ts:1-10` |
| BR-02 | Initial state | Every new order starts in `PENDING`; creation is recorded as `null -> PENDING` | `src/modules/orders/order.service.ts:99,109-112`; `prisma/schema.prisma:78` |
| BR-03 | Transition | From `PENDING`, only `PAID` and `CANCELLED` are reachable | `src/modules/orders/order.status.ts:4` |
| BR-04 | Transition | From `PAID`, only `PROCESSING` and `CANCELLED` are reachable | `src/modules/orders/order.status.ts:5` |
| BR-05 | Transition | From `PROCESSING`, only `SHIPPED` and `CANCELLED` are reachable | `src/modules/orders/order.status.ts:6` |
| BR-06 | Transition | From `SHIPPED`, only `DELIVERED` is reachable | `src/modules/orders/order.status.ts:7` |
| BR-07 | Termination | `DELIVERED` is terminal and admits no successor | `src/modules/orders/order.status.ts:8,20-22` |
| BR-08 | Termination | `CANCELLED` is terminal and admits no successor | `src/modules/orders/order.status.ts:9,20-22` |
| BR-09 | Invariant | A status may not transition to itself | `src/modules/orders/order.status.ts:3-10`; `src/modules/orders/order.service.ts:140-146` |
| BR-10 | Invariant | The lifecycle is forward-only; no reverse edge exists | `src/modules/orders/order.status.ts:3-10` |
| BR-11 | Validation | Illegal edges are rejected with 409 and leave no side effects | `src/modules/orders/order.status.ts:12-14`; `src/modules/orders/order.service.ts:147-149`; `src/shared/errors/http-errors.ts:45-53` |
| BR-12 | Business Logic | Stock is debited exactly and only on `PENDING -> PAID` | `src/modules/orders/order.status.ts:24-31`; `src/modules/orders/order.service.ts:151-153,204-231` |
| BR-13 | Business Logic | Stock is replenished exactly and only on `PAID -> CANCELLED` and `PROCESSING -> CANCELLED` | `src/modules/orders/order.status.ts:33-37`; `src/modules/orders/order.service.ts:154-156,233-243` |
| BR-14 | Business Logic | `PENDING -> CANCELLED` has no stock effect | `src/modules/orders/order.status.ts:33-37` |
| BR-15 | Business Logic | Cancellation is impossible after shipment, so stock is never reversed post-shipment | `src/modules/orders/order.status.ts:7,33-37` |
| BR-16 | Invariant | Transition decisions are pure, deterministic and actor-agnostic | `src/modules/orders/order.status.ts:12-37` |
| BR-17 | Audit | Every applied transition appends an `OrderStatusHistory` row in the same transaction | `src/modules/orders/order.service.ts:159-167`; `prisma/schema.prisma:116-131` |
| BR-18 | Validation | A transition reason is optional, at most 500 characters, and not semantically validated | `src/modules/orders/order.schemas.ts:18-21`; `prisma/schema.prisma:123` |
| BR-19 | Business Logic | Only `PENDING` or `CANCELLED` orders may be deleted | `src/modules/orders/order.service.ts:181-192` |
| BR-20 | Contract | Allowed-successor and terminal-state queries exist but are not part of the public contract | `src/modules/orders/order.status.ts:16-22` |
| BR-21 | Business Logic | Inventory commitment is deferred to payment; no stock is reserved at creation | `src/modules/orders/order.status.ts:24-31`; `src/modules/orders/order.service.ts:50-124` |

### 3.3 Detailed breakdown of the business rules

---

### Business Rule: BR-01 Status vocabulary

**Overview**: The state machine recognizes exactly six statuses, inherited from the Prisma `OrderStatus` enum, and its table is structurally required to define an entry for every one of them.

**Detailed description**:

The vocabulary is declared once in `prisma/schema.prisma:16-23` as `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`, and is materialized by Prisma as a MySQL `ENUM` on `orders.status`, `order_status_history.fromStatus` and `order_status_history.toStatus` (`prisma/migrations/20260519182739_init/migration.sql:54,90-91`). `order.status.ts:1` imports the generated TypeScript enum from `@prisma/client`, and `order.status.ts:3` types the table as `Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>>`. Because `Record` over a union of literal keys requires every key, the compiler enforces exhaustiveness of the table in both directions: a new enum member must be added here or the build fails, and a removed or renamed member breaks the existing entry.

The same vocabulary is enforced at three other points, each independent of this module. Zod validates `toStatus` with `z.nativeEnum(OrderStatus)` in `updateOrderStatusSchema` (`order.schemas.ts:18-21`), so an unknown string is a 400 before the service is entered; the same schema is used for the optional `status` filter in `listOrdersQuerySchema` (`order.schemas.ts:26`). The database rejects an out-of-vocabulary value at the storage layer. The seed script imports the enum directly (`prisma/seed.ts:1`). The state machine is therefore one of four places where the six-value vocabulary is asserted, and it is the only one that binds the vocabulary to a transition policy.

The machine is flat: there are no sub-states, no compound status values, no orthogonal regions and no version or generation counter on the status field. `Order` carries a single scalar `status` column with a default of `PENDING` (`prisma/schema.prisma:78`). Concepts such as "paid but on hold", "partially shipped", "returned" or "refunded" have no representation. Any extension of the lifecycle requires a schema migration plus an edit to this table; there is no configuration or database-driven mechanism that can add a state or edge at runtime.

**Rule workflow**:

```
Prisma schema enum OrderStatus (6 members)
  -> prisma generate
  -> generated OrderStatus type/enum imported at order.status.ts:1
  -> Record<OrderStatus, ...> requires an entry for each member at compile time
  -> runtime calls canTransition(from, to) only with generated enum values
```

**Confidence**: High (explicit in schema and types).

---

### Business Rule: BR-02 Initial state PENDING

**Overview**: Order creation always produces a `PENDING` order, and the audit trail represents the creation event as a transition from `null` to `PENDING`.

**Detailed description**:

The initial state is not declared inside `order.status.ts`; it is enforced in two other places. `OrderService.create` hardcodes `status: OrderStatus.PENDING` in the insert (`order.service.ts:99`) and `createOrderSchema` has no `status` field at all (`order.schemas.ts:11-16`), so a client cannot request any other starting state. The database adds a second enforcement point with `status OrderStatus @default(PENDING)` (`prisma/schema.prisma:78`, rendered as `DEFAULT 'PENDING'` in `migration.sql:54`). The state machine's table starts from `PENDING` as the only state with two outgoing edges that is reachable without any prior transition, although the table itself does not encode reachability.

Creation is recorded as `fromStatus: null, toStatus: PENDING` in the nested history write (`order.service.ts:106-113`), with `reason: 'order created'`. The enum has no `CREATED` member, so `null` is the only representation of "no previous state". This has a direct consequence for the state machine: `canTransition` is never called during creation, and the module has no concept of a creation pseudo-state. The first history row and the first status value come from two different code paths (nested create versus the state machine), which is why `PENDING` must be treated as both an initial value and a regular transition source.

The rule is not enforced for writers that bypass the service. The seed script inserts orders directly with `status: plan.status` (`prisma/seed.ts:287-305`), which can be `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED` or `CANCELLED` (`prisma/seed.ts:226-252`), and it fabricates a history path for each (`prisma/seed.ts:255-321`). Consequently, the invariant "all orders begin in PENDING" holds for API-created data only.

**Rule workflow**:

```
POST /api/v1/orders
  -> schema rejects any client-supplied status (field does not exist)
  -> service inserts order with status = PENDING
  -> nested history row: fromStatus = null, toStatus = PENDING, reason = 'order created'
  -> response contains status PENDING and one history entry
```

**Confidence**: High for API path; High that the DB default and seed bypass exist; the seed bypass is explicit code, not an inference.

---

### Business Rule: BR-03 PENDING successors

**Overview**: A `PENDING` order may move only to `PAID` or `CANCELLED`.

**Detailed description**:

`transitions[OrderStatus.PENDING]` is `[OrderStatus.PAID, OrderStatus.CANCELLED]` (`order.status.ts:4`). This encodes the first decision point of the order lifecycle: an order that has been created but not paid can either become a commercial commitment (`PAID`) or be abandoned (`CANCELLED`). Because `canTransition` is consulted before any side effect, a request for `PENDING -> PROCESSING`, `PENDING -> SHIPPED`, `PENDING -> DELIVERED` or `PENDING -> PENDING` fails with 409 and leaves stock untouched; `tests/orders.test.ts:89-107` asserts the `PENDING -> SHIPPED` case and its `INVALID_STATUS_TRANSITION` code.

The rule couples status to fulfilment sequencing. An order cannot enter processing or shipping without passing through `PAID`, so payment is a mandatory gateway for warehouse work. There is no way to skip payment for a free, complimentary or already-settled order through the API, because the only edge out of `PENDING` other than cancellation is `PAID` and the `PAID` transition triggers the stock debit. This is a business decision encoded in the graph rather than in a condition: the machine has no notion of payment method, amount, or zero-value orders.

The two edges have different side effects. `PENDING -> PAID` triggers the only stock debit in the system (`order.status.ts:24-31`); `PENDING -> CANCELLED` triggers no stock movement at all (`order.status.ts:33-37`) because no inventory was ever taken. Both edges write a history row. Both are legal regardless of who requests them, what the order totals are, or how much time has passed since creation; the table has no temporal or actor conditions.

**Rule workflow**:

```
from = PENDING
  -> to = PAID      : legal, debit stock, update status, append history
  -> to = CANCELLED : legal, no stock effect, update status, append history
  -> any other to   : canTransition false -> 409, no writes
```

**Confidence**: High (explicit table entry; side effects explicit in service).

---

### Business Rule: BR-04 PAID successors

**Overview**: A `PAID` order may move only to `PROCESSING` or `CANCELLED`.

**Detailed description**:

`transitions[OrderStatus.PAID]` is `[OrderStatus.PROCESSING, OrderStatus.CANCELLED]` (`order.status.ts:5`). `PAID` represents the point at which inventory has already been debited, so both successors must be interpreted against that fact. `PROCESSING` begins fulfilment; `CANCELLED` closes the order and, because stock was debited at payment, triggers replenishment through `shouldReplenishStock` (`order.status.ts:33-37`).

The rule forbids `PAID -> SHIPPED` and `PAID -> DELIVERED`. Fulfilment must be explicitly acknowledged by entering `PROCESSING` before shipping, so the system records an operational handover between payment and dispatch. The rule also forbids `PAID -> PENDING`, so a payment cannot be "un-recorded" through a status change, and it forbids `PAID -> PAID`, which is intercepted by the same-status guard with a distinct message (`order.service.ts:140-146`).

There is no distinction between full and partial payment, no `PARTIALLY_PAID` state, and no payment reference on the order. `PAID` is a terminal commitment in the narrow sense that inventory has moved, but it is not a terminal state of the machine because two outgoing edges exist. Cancellation from `PAID` is the only way to reverse the debit, and it is a hard cancellation, not a refund: the API records the status and replenishes stock but has no refund, ledger or payment-provider interaction.

**Rule workflow**:

```
from = PAID
  -> to = PROCESSING : legal, no stock effect, update, history
  -> to = CANCELLED  : legal, replenish stock, update, history
  -> to = SHIPPED    : 409 INVALID_STATUS_TRANSITION
  -> to = PENDING    : 409 INVALID_STATUS_TRANSITION
```

**Confidence**: High.

---

### Business Rule: BR-05 PROCESSING successors

**Overview**: A `PROCESSING` order may move only to `SHIPPED` or `CANCELLED`.

**Detailed description**:

`transitions[OrderStatus.PROCESSING]` is `[OrderStatus.SHIPPED, OrderStatus.CANCELLED]` (`order.status.ts:6`). `PROCESSING` is the fulfilment-in-progress state; the two edges represent dispatch or cancellation before dispatch. Cancellation from this state still replenishes stock, because the debit occurred at `PENDING -> PAID` and the goods have not left the warehouse from the model's point of view. `tests/orders.test.ts:134-161` validates the replenishment logic only for `PAID -> CANCELLED`; the `PROCESSING -> CANCELLED` path is implemented but not covered by any test.

The rule forbids `PROCESSING -> DELIVERED`, so delivery cannot be recorded without an explicit shipment step. It also forbids `PROCESSING -> PAID`, so an order cannot be moved back to the paid state once fulfilment starts. The only "exit" from `PROCESSING` other than shipping is cancellation, which means an order that is stuck in processing cannot be corrected, reopened or moved to another fulfilment attempt; the alternatives are to ship it or cancel it.

`PROCESSING` has no associated data beyond the status itself. There is no picking list, carrier, tracking number, warehouse or operator assignment, and no timestamp field other than the history row's `changedAt`. The state machine therefore cannot distinguish a processing order that has just entered the state from one that has been sitting there indefinitely; there is no age or timeout rule, and no edge is time-dependent.

**Rule workflow**:

```
from = PROCESSING
  -> to = SHIPPED   : legal, no stock effect, update, history
  -> to = CANCELLED : legal, replenish stock, update, history
  -> to = DELIVERED : 409 INVALID_STATUS_TRANSITION
  -> to = PAID      : 409 INVALID_STATUS_TRANSITION
```

**Confidence**: High.

---

### Business Rule: BR-06 SHIPPED successor

**Overview**: A `SHIPPED` order may move only to `DELIVERED`.

**Detailed description**:

`transitions[OrderStatus.SHIPPED]` is `[OrderStatus.DELIVERED]` (`order.status.ts:7`). This is the single one-element successor array in the table and the only state with exactly one exit. It makes `DELIVERED` a mandatory next step and removes cancellation as an option once goods are considered dispatched.

Because the edge has no stock predicate, `SHIPPED -> DELIVERED` performs only a status update and a history insert. Stock remains at its post-payment level. The absence of a `SHIPPED -> CANCELLED` edge is the load-bearing detail for inventory: `shouldReplenishStock` only fires for `PAID` and `PROCESSING` sources, and those are the only sources for which a cancellation edge exists. The two rule sets are aligned in effect, but neither references the other, so the alignment is not enforced structurally.

The rule has no dependency on delivery data. There is no delivery confirmation, signature, carrier event or received-at timestamp in the order model; `DELIVERED` is a manual operator assertion. The machine cannot express a failed delivery, a return to sender or a partial delivery. Once `SHIPPED` is recorded, the only reachable future state is `DELIVERED`, and the machine provides no path back.

**Rule workflow**:

```
from = SHIPPED
  -> to = DELIVERED : legal, no stock effect, update, history
  -> to = CANCELLED : 409 INVALID_STATUS_TRANSITION (no edge)
  -> to = anything else : 409 INVALID_STATUS_TRANSITION
```

**Confidence**: High.

---

### Business Rule: BR-07 DELIVERED is terminal

**Overview**: `DELIVERED` has no outgoing transitions and therefore admits no further status change through the state machine.

**Detailed description**:

`transitions[OrderStatus.DELIVERED]` is the empty array (`order.status.ts:8`), and `canTransition` returns false for every pair whose source is `DELIVERED`, because `includes` on an empty array is always false. `isTerminal` encodes the same property by testing whether the successor array is empty (`order.status.ts:20-22`), but no production code calls it, so the actual rejection comes from `canTransition` alone.

Terminality here is exhaustive with respect to the vocabulary: a delivered order cannot be cancelled, returned, refunded, reopened or moved back to any earlier state. Because cancellation is the only stock-replenishing path, a delivered order can never return inventory through the API. A post-delivery return would require either a new order or direct database manipulation, and neither is modeled. The audit history of a delivered order is immutable at the API level except for one important exception: the order itself can be hard-deleted only when it is `PENDING` or `CANCELLED` (BR-19), so a `DELIVERED` order cannot be deleted through the API and its history is retained.

The business consequence is that `DELIVERED` is a final commercial record for reporting, but the model has no after-sales lifecycle. There is no `RETURNED`, `REFUNDED`, `EXCHANGED` or `DISPUTED` state. Any operational process that needs those concepts must live outside this state machine, which also means it cannot be represented consistently in `OrderStatusHistory`.

**Rule workflow**:

```
from = DELIVERED
  -> canTransition(DELIVERED, X) = transitions[DELIVERED].includes(X)
  -> transitions[DELIVERED] = []
  -> result false for every X
  -> 409 INVALID_STATUS_TRANSITION, no writes
```

**Confidence**: High for behavior; the intent to exclude returns is not documented in the repository (only the absence of edges is observable), so the policy rationale carries Medium confidence.

---

### Business Rule: BR-08 CANCELLED is terminal

**Overview**: `CANCELLED` has no outgoing transitions and therefore admits no reinstatement or further movement.

**Detailed description**:

`transitions[OrderStatus.CANCELLED]` is the empty array (`order.status.ts:9`). Once an order reaches `CANCELLED`, every attempted transition fails with 409, including `CANCELLED -> PENDING`. This is the state-machine expression of "a cancelled order is dead": there is no resume, no re-payment, no re-open. A customer who wants the same goods must place a new order, which receives a new order number and a new history.

Cancellation is reachable from three states: `PENDING`, `PAID` and `PROCESSING`. The stock effect differs by source (none, increment, increment respectively), but the destination and terminality are identical. Consequently, the `CANCELLED` state does not record where it came from in the status column itself; that information exists only in the history rows and in the stock effect that was applied at the time.

Unlike `DELIVERED`, `CANCELLED` orders are deletable through the API (`order.service.ts:184`). Deleting one removes the order and cascades to items and status history (`migration.sql:116,122`), so the terminal state can be the end of the record's existence as well as the end of its lifecycle. This is a deliberate-looking property of the delete rule, but it means terminality does not imply retention.

**Rule workflow**:

```
from = CANCELLED
  -> canTransition(CANCELLED, X) = false for every X
  -> 409 INVALID_STATUS_TRANSITION, no writes
  -> DELETE /orders/{id} remains allowed and removes the aggregate
```

**Confidence**: High for behavior; conflict between terminality and deletability is observable and noted, not inferred.

---

### Business Rule: BR-09 Self-transition rejection

**Overview**: No status may transition to itself, and the service also rejects self-transitions explicitly before consulting the table.

**Detailed description**:

The table never lists a state as its own successor, so `canTransition(X, X)` is false for all six states by construction. In addition, `changeStatus` performs an explicit equality check (`order.service.ts:140-146`) and throws `ConflictError('Order is already in ${to} status', 'INVALID_STATUS_TRANSITION', { from, to })` before calling `canTransition`. The explicit guard therefore does not change legality; it changes the error message and short-circuits the evaluation.

Both paths produce HTTP 409 with `error.code = 'INVALID_STATUS_TRANSITION'` and `error.details = { from, to }`, because `InvalidStatusTransitionError` extends `ConflictError` and sets the same code and detail shape (`http-errors.ts:45-53`). The only observable difference is the human-readable message: "Order is already in PAID status" versus "Invalid status transition from PENDING to PAID". A client that switches on the code cannot distinguish a repeated request from an illegal edge; a client that parses the message can.

The guard applies to terminal states as well. `DELIVERED -> DELIVERED` and `CANCELLED -> CANCELLED` return the "already in" conflict instead of an empty-successor failure, and the guard fires before any stock predicate or history insert, so no side effect occurs. The rule is not covered by any test; the integration suite contains no repeated-status request.

**Rule workflow**:

```
request toStatus
  -> from === to ? yes -> 409 ConflictError "Order is already in X status"
  -> no -> canTransition(from, to) -> includes check (self is never included)
```

**Confidence**: High.

---

### Business Rule: BR-10 Forward-only lifecycle

**Overview**: No transition moves an order to an earlier lifecycle position; the only branching is the cancellation branch, which is terminal.

**Detailed description**:

The graph contains no reverse edges: `PAID -> PENDING`, `PROCESSING -> PAID`, `SHIPPED -> PROCESSING` and `DELIVERED -> SHIPPED` are all absent, as are shorter regressions such as `PROCESSING -> PENDING`. The forward chain is total for non-cancelled orders: `PENDING -> PAID -> PROCESSING -> SHIPPED -> DELIVERED`. Each step is mandatory; no forward edge skips a stage.

This design means a mistaken transition cannot be corrected by moving backward. If an operator wrongly marks an order `PAID`, the only in-model correction is `PAID -> CANCELLED`, which replenishes stock and ends the order; there is no way to restore `PENDING`. If an operator wrongly marks an order `PROCESSING`, the only alternatives are `SHIPPED` or `CANCELLED`, both of which have business consequences. The model accepts that mistakes are handled by cancellation and re-creation rather than by state rollback.

Forward-only progression also bounds history size in practice: a non-cancelled order produces at most five history rows (creation plus four transitions), and a cancelled order at most four. No code relies on this bound for pagination or storage, but it is a direct consequence of the acyclic graph. The graph is acyclic, so no cycle-detection or transition-count limit is needed.

**Rule workflow**:

```
rank: PENDING < PAID < PROCESSING < SHIPPED < DELIVERED
every legal edge increases rank, except -> CANCELLED which exits the chain
reverse or skip edges are absent -> canTransition false -> 409
```

**Confidence**: High for the graph; the "rank" description is an analytical restatement, not a code construct.

---

### Business Rule: BR-11 Illegal transitions are rejected without side effects

**Overview**: Any pair not present in the table produces HTTP 409 with code `INVALID_STATUS_TRANSITION`, details `{ from, to }`, and no status, stock or history mutation.

**Detailed description**:

`canTransition` is a single inclusion test over the successor array of the source state (`order.status.ts:12-14`). Its boolean result is consumed exactly once, in `changeStatus` (`order.service.ts:147-149`), where false causes `InvalidStatusTransitionError` to be thrown. That error is a subclass of `ConflictError` (`http-errors.ts:33-53`), so the central error middleware serializes it as 409 with `{ error: { code, message, details } }` (`error.middleware.ts:15-24`). No other layer re-checks the transition; the HTTP boundary checks only that the target is a member of the enum.

The rejection occurs inside the transaction but before any write. `findUnique` has already read the order and its items, and the transaction is open, so a thrown error rolls back nothing that was written because nothing was written yet. Stock predicates are not evaluated, the status is not updated, and no history row is inserted. The test suite verifies the HTTP effect for `PENDING -> SHIPPED` (`tests/orders.test.ts:89-107`) but does not assert the absence of a history row or stock change for the rejected request, so the no-side-effect property is established by code order rather than by test.

The rejection is not logged. `errorMiddleware` handles `AppError` instances in its first branch and returns immediately (`error.middleware.ts:15-24`), before the `logger.error` call at line 57, so a rejected transition produces no application log entry. There is no counter, metric or audit record of attempted illegal transitions. The requesting client receives the code and the pair, but the system retains no evidence after the response.

**Rule workflow**:

```
from, to -> canTransition
  -> false -> throw InvalidStatusTransitionError(from, to)
     -> ConflictError statusCode 409, code INVALID_STATUS_TRANSITION
     -> error middleware returns { error: { code, message, details } }
     -> transaction rollback (no writes performed)
     -> no log entry (AppError branch returns before logger.error)
```

**Confidence**: High.

---

### Business Rule: BR-12 Stock debit on PENDING to PAID only

**Overview**: Inventory is decremented exactly once, only when an order moves from `PENDING` to `PAID`.

**Detailed description**:

`STOCK_DEBIT_TRANSITION` pins the pair `{ from: PENDING, to: PAID }` (`order.status.ts:24-27`), and `shouldDebitStock` returns true only for that exact pair (`order.status.ts:29-31`). The service calls it after legality is confirmed and before the status update (`order.service.ts:151-153`). When true, `debitStock` loads all referenced products, collects every item whose product is missing or whose `stockQuantity` is below the requested quantity, and if the list is non-empty throws `InsufficientStockError` with `{ unavailable: [{ sku, requested, available }] }` (`order.service.ts:204-224`). Only if every item is satisfiable does it decrement each product (`order.service.ts:225-230`).

The placement inside the transaction is the mechanism that preserves the invariant "a paid order has had stock debited". A shortage aborts the transaction, so the order remains `PENDING` and no history row is written. The integration test at `tests/orders.test.ts:109-132` creates a `PENDING` order for 5 units of a product with stock 1, requests `PAID`, expects 422 with the product SKU in `details.unavailable`, and then asserts that stock is still 1. The successful path is tested at `tests/orders.test.ts:59-87`, which debits 3 from 5 and asserts 2 remaining.

The rule is expressed independently of `canTransition`. The table and the debit predicate agree today because both are literal pairs, but nothing enforces that a future legal edge into `PAID` would or would not debit. If another edge into `PAID` were added to the table, `shouldDebitStock` would not fire for it, and the order would become `PAID` without inventory movement. The reverse is also unguarded: `shouldDebitStock` could be changed without touching the table, although the service's legality check runs first, so a debit for an illegal pair would be unreachable. The two sources of truth are therefore consistent by construction of the current literals, not by type or test.

**Rule workflow**:

```
from = PENDING and to = PAID
  -> canTransition true
  -> shouldDebitStock true
  -> load products for all items
  -> any product missing or stock < quantity ?
       yes -> InsufficientStockError 422 -> rollback
       no  -> decrement each product by item.quantity
  -> update order.status = PAID
  -> insert history row
```

**Confidence**: High.

---

### Business Rule: BR-13 Stock replenishment on cancellation after payment

**Overview**: Inventory is incremented when an order is cancelled from `PAID` or `PROCESSING`, and only then.

**Detailed description**:

`shouldReplenishStock` returns true when `to === CANCELLED` and `from` is either `PAID` or `PROCESSING` (`order.status.ts:33-37`). The service calls it after the debit branch and before the status update (`order.service.ts:154-156`), and `replenishStock` unconditionally increments every product by the item quantity (`order.service.ts:233-243`). There is no availability check, no cap and no comparison against the original debit; the increment is blind.

The rule is intended to mirror BR-12: stock is added back only for states in which it must already have been removed. `PAID` and `PROCESSING` are both post-debit states, because the only entry into `PAID` is the debiting edge. `PENDING` is excluded because it never debited. `SHIPPED` and `DELIVERED` are excluded both by the predicate and by the absence of a `CANCELLED` edge from those states. The integration test at `tests/orders.test.ts:134-161` verifies the `PAID -> CANCELLED` case by driving stock from 10 to 6 on payment and back to 10 on cancellation.

The blind increment assumes a consistent stock history. If a `PAID` or `PROCESSING` order was created without a corresponding debit (for example by the seed script, which inserts orders directly in those statuses and never adjusts stock, `prisma/seed.ts:226-252`), then cancelling it inflates inventory above its original level. The state machine has no way to detect this because it receives only the status pair, not the stock ledger. The rule is a policy on a transition pair, not a reconciliation mechanism.

**Rule workflow**:

```
from in {PAID, PROCESSING} and to = CANCELLED
  -> canTransition true
  -> shouldReplenishStock true
  -> for each item: increment product.stockQuantity by item.quantity
  -> update order.status = CANCELLED
  -> insert history row
```

**Confidence**: High for behavior; the assumption of a prior debit is explicit in the predicate structure but not checked.

---

### Business Rule: BR-14 PENDING to CANCELLED has no stock effect

**Overview**: Cancelling an unpaid order changes only its status and history; inventory is untouched.

**Detailed description**:

`shouldDebitStock(PENDING, CANCELLED)` is false because the pair is not `{PENDING, PAID}`, and `shouldReplenishStock(PENDING, CANCELLED)` is false because the source is not `PAID` or `PROCESSING`. The cancellation therefore executes the status update and the history insert with no product writes. This is the correct counterpart to deferred commitment: a `PENDING` order never took stock, so cancelling it must not add stock.

The rule is only implicit in the two predicates; there is no test that drives `PENDING -> CANCELLED` and asserts unchanged stock. The test suite contains no cancellation from `PENDING` at all. The property follows from the predicate truth table, which can be evaluated statically, but no regression net would catch a change that made `shouldReplenishStock` return true for `PENDING`.

Operationally, this rule is what makes abandoned carts harmless to inventory. Since creation does not reserve stock (BR-21), a `PENDING` order is a claim without inventory, and cancellation is a no-op for the warehouse. The cost of that choice is that stock is not protected between order creation and payment, which is analyzed under BR-21.

**Rule workflow**:

```
from = PENDING, to = CANCELLED
  -> canTransition true
  -> shouldDebitStock false
  -> shouldReplenishStock false
  -> update status + insert history, no product writes
```

**Confidence**: High.

---

### Business Rule: BR-15 No cancellation after shipment

**Overview**: Once an order is `SHIPPED`, it cannot be cancelled, so inventory is never reversed after dispatch.

**Detailed description**:

The table gives `SHIPPED` exactly one successor, `DELIVERED` (`order.status.ts:7`), and gives `DELIVERED` none. Neither state has a `CANCELLED` edge, so `canTransition(SHIPPED, CANCELLED)` and `canTransition(DELIVERED, CANCELLED)` both return false. Combined with the replenishment predicate, which only accepts `PAID` and `PROCESSING` sources, this means stock is returned only before dispatch.

The business meaning is a hard cut-off for inventory reversal. A shipped order can still be delivered, but it cannot be cancelled; a delivered order is final. There is no `RETURNED` state, no restocking workflow, no partial return and no refund record. Any post-shipment return process must be handled outside the state machine and therefore outside `OrderStatusHistory`, which makes the order's audit trail incomplete with respect to after-sales events.

The rule appears as an absence rather than a stated constraint. Neither `order.status.ts` nor any project document in the repository states that post-shipment cancellation is forbidden by policy; the only evidence is the missing edge. The analysis therefore records the behavior with high confidence and the rationale with medium confidence.

**Rule workflow**:

```
from in {SHIPPED, DELIVERED}
  -> no CANCELLED successor
  -> canTransition false -> 409 INVALID_STATUS_TRANSITION
  -> no replenishment branch reachable
```

**Confidence**: High for behavior; Medium for intent.

---

### Business Rule: BR-16 Pure, deterministic, actor-agnostic decisions

**Overview**: Transition legality and stock selection depend only on the `(from, to)` pair; no actor, role, amount, time or external state influences them.

**Detailed description**:

Every exported decision function has the same signature shape: two enum arguments and a boolean or array result (`order.status.ts:12-37`). None reads a database, clock, configuration, feature flag, request context or environment variable. The functions are referentially transparent: the same pair always yields the same answer for the life of the process, and the module holds no mutable state that a caller can change through its API (the table is module-private). This makes the module trivially unit-testable and safe to call inside a transaction, and it means the decision cannot fail with an I/O error.

Actor identity is not an input. `OrderService.changeStatus` receives `userId` and uses it only for the history row (`order.service.ts:159-167`); the state machine never sees it. There is no role check on the route either: `order.routes.ts:14` applies `authenticate` to the whole router and no `requireRole` or equivalent is used. As a result, any authenticated user, `ADMIN` or `OPERATOR`, can perform any legal transition, including `PENDING -> PAID` (which moves stock) and any cancellation. The matrix is uniform across roles.

The same purity means the machine cannot enforce rules that require context: it cannot require a payment reference for `PAID`, a carrier for `SHIPPED`, a reason for `CANCELLED`, a minimum age before `PROCESSING`, or a four-eyes approval for high-value cancellations. Those constraints do not exist anywhere else in the order write path either, so the lifecycle is governed solely by the two-value pair. This is a deliberate structural simplification visible in the signatures, not an omission inside a larger rules engine.

**Rule workflow**:

```
canTransition(from, to) = table[from].includes(to)
inputs: two enum values only
no side effects, no I/O, no randomness, no clock, no actor
=> identical result for identical pair
```

**Confidence**: High.

---

### Business Rule: BR-17 Every applied transition appends an audit row

**Overview**: Each accepted status change inserts one `OrderStatusHistory` row with source, target, actor, timestamp and optional reason in the same transaction as the status update.

**Detailed description**:

After the status update, `changeStatus` inserts `{ orderId, fromStatus: from, toStatus: to, changedById: userId, reason: input.reason ?? null }` (`order.service.ts:159-167`). `changedAt` defaults to `now()` in the database (`prisma/schema.prisma:121`). Because the insert shares the interactive transaction with the update, a failed history insert aborts the status change, and a rolled-back transaction leaves neither. The order's creation row is written separately by the nested create (`order.service.ts:106-113`) with `fromStatus: null` and `reason: 'order created'`.

The history is append-only from the application's perspective: there is no update or delete method for `OrderStatusHistory` in the repository or service. It is not append-only at the database level, however, because `order_status_history.orderId` has `ON DELETE CASCADE` (`migration.sql:122`), so deleting an order removes its entire history. Since `CANCELLED` orders are deletable (BR-19), an order that was paid and then cancelled can lose its audit trail through a supported API call.

The table has no uniqueness constraint on `(orderId, fromStatus, toStatus, changedAt)`, so duplicate rows are possible. Under sequential use this cannot happen because a successful transition changes the source status, and a repeat is rejected by the same-status guard. Under concurrent use, two transactions can both read the same source status and both insert a history row with the same `fromStatus` and `toStatus` (see the concurrency risk in section 10). The audit trail would then contain a duplicate transition that did not occur twice in reality.

**Rule workflow**:

```
legal transition + successful stock effects
  -> UPDATE orders.status = to
  -> INSERT order_status_history(orderId, fromStatus, toStatus, changedById, reason)
  -> COMMIT both together
order deletion -> CASCADE removes history rows
```

**Confidence**: High.

---

### Business Rule: BR-18 Transition reason is optional and unconstrained in meaning

**Overview**: A caller may attach a reason of up to 500 characters to any transition; it is not required, not validated against the transition type, and appears only in the history row.

**Detailed description**:

`updateOrderStatusSchema` declares `reason: z.string().max(500).optional()` (`order.schemas.ts:20`), the service writes `input.reason ?? null` (`order.service.ts:165`), and the database column is `VARCHAR(500) NULL` (`prisma/schema.prisma:123`; `migration.sql:94`). The reason is therefore doubly bounded at 500 characters and may be absent. Omitting it produces `null`; there is no default text for a normal transition, unlike creation, which stores `'order created'`.

The rule applies uniformly to every edge. A cancellation can be recorded without a reason, a `PAID` transition can carry a cancellation-style explanation, and a `DELIVERED` transition can carry any string. There is no enumeration of reason codes, no required reason for destructive transitions and no free-text sanitation beyond length. Because the reason is not returned on the `Order` itself, only readers of the `history` array see it; the service always includes history in the status-change response and in `getById` (`order.repository.ts:47-60`).

The reason is the only free-text field in the transition record and the only place where operator intent can be captured. It is not linked to the stock effect, so a cancellation that replenishes stock does not require an explanation, and the resulting history row cannot be used to distinguish a customer-requested cancellation from an operational correction except by reading the text. The field is also not indexed, so it is not searchable at the database level within the current schema.

**Rule workflow**:

```
PATCH body { toStatus, reason? }
  -> zod max(500) optional
  -> service: reason ?? null
  -> history row stores the string or null
  -> order row does not store the reason
```

**Confidence**: High.

---

### Business Rule: BR-19 Only PENDING or CANCELLED orders may be deleted

**Overview**: Deletion is allowed for orders in `PENDING` or `CANCELLED` and rejected with 409 for every other status.

**Detailed description**:

This rule lives outside `order.status.ts` but consumes the same status vocabulary directly. `OrderService.delete` loads the order, checks `status !== PENDING && status !== CANCELLED`, and throws `ConflictError('Order can only be deleted while in PENDING or CANCELLED status', 'INVALID_ORDER_STATE_FOR_DELETE', { status })` for anything else (`order.service.ts:181-192`). The integration test at `tests/orders.test.ts:197-218` creates an order, pays it, and asserts that deletion returns 409 with that code.

The rule treats a terminal state as disposable. `CANCELLED` is terminal for transitions but eligible for deletion, which means the final state of a cancelled order can be the removal of the entire aggregate: order, items and status history, by cascade (`migration.sql:116,122`). `PENDING` is also deletable because nothing irreversible has happened. `PAID`, `PROCESSING`, `SHIPPED` and `DELIVERED` are protected, presumably because they represent committed inventory or delivery records, but the code does not state the rationale.

The check is a read-then-delete sequence outside a single conditional statement; the repository's `deleteById` is a plain `delete` by unique id (`order.repository.ts:66-68`), and the service does not wrap the read and delete in a transaction. A concurrent status change between the read and the delete could therefore delete an order that has just become `PAID`. No test or documentation addresses that race. The rule also does not consult the state machine's terminal predicate, so it is a separate policy that must be maintained in parallel with the matrix.

**Rule workflow**:

```
DELETE /api/v1/orders/{id}
  -> findById
  -> not found -> 404
  -> status in {PENDING, CANCELLED} ? no -> 409 INVALID_ORDER_STATE_FOR_DELETE
  -> yes -> deleteById -> cascade items + history -> 204
```

**Confidence**: High for behavior; Medium for the intent behind allowing deletion of `CANCELLED`.

---

### Business Rule: BR-20 No discovery of allowed transitions through the API

**Overview**: `allowedTransitions` and `isTerminal` are exported capabilities that no production code consumes and no endpoint exposes.

**Detailed description**:

The module exports `allowedTransitions(from)` returning the successor array (`order.status.ts:16-18`) and `isTerminal(status)` returning whether that array is empty (`order.status.ts:20-22`). A repository-wide search finds no import of either function; the only consumer of the module is `order.service.ts`, which imports exactly `canTransition`, `shouldDebitStock` and `shouldReplenishStock` (`order.service.ts:17-21`). Neither function is used by the controller, the routes, the seed or the tests.

As a result, a client cannot ask the API which transitions are available for an order. The only way to discover legality is to attempt the transition and read the 409, or to consult documentation that does not exist in the repository. The order response includes the current status but not the successor list. This is a contract-level gap rather than a runtime defect: the state machine knows the answer, but the interface does not publish it.

The two functions are also untested, so their correctness is verified only by inspection. `isTerminal` derives terminality from the same table that `canTransition` uses, which means the two cannot disagree at runtime for in-vocabulary input; nevertheless the service relies on `canTransition` rather than `isTerminal`, so a future change to the notion of terminality would have to be applied in the table and would automatically affect both. The unused exports therefore constitute dormant surface area rather than a second source of truth.

**Rule workflow**:

```
client -> GET /orders/{id} -> status only
client -> PATCH /orders/{id}/status -> attempt -> 200 or 409
module -> allowedTransitions / isTerminal -> no caller
```

**Confidence**: High.

---

### Business Rule: BR-21 Inventory commitment is deferred to payment

**Overview**: Order creation validates products and prices but does not reserve or check stock; inventory is committed only when the order becomes `PAID`.

**Detailed description**:

`OrderService.create` verifies that the customer exists, that all products exist, that none are inactive, and that the discount does not exceed the subtotal; it computes unit prices and totals on the server and inserts the order with `PENDING` (`order.service.ts:50-124`). It never reads `stockQuantity` and never decrements it. The stock check appears only in `debitStock`, which is called from the `PENDING -> PAID` branch (`order.service.ts:151-153,204-224`). The state machine encodes this policy through the location of `STOCK_DEBIT_TRANSITION`: the debiting edge is the payment edge, not the creation action.

The business effect is that `PENDING` orders are non-binding claims. Any number of `PENDING` orders may reference the same units, and the first one to be paid wins; the others fail at payment with 422 `INSUFFICIENT_STOCK`. This avoids holding inventory for abandoned carts, but it also means an accepted order is not a guarantee of fulfilment. There is no reservation record, no expiry on `PENDING` orders and no automatic cancellation of stale orders, so an unlimited number of unpaid orders can coexist against the same stock.

The rule is a direct consequence of the graph plus the stock predicate, but it is not stated in a comment or document. It matters for the state machine because it explains why `PENDING -> CANCELLED` has no replenishment: nothing was taken. It also means the lifecycle has no inventory-aware transition except `PENDING -> PAID`, and the failure mode of that edge is the only place where stock availability affects the state graph. The API exposes the shortage in `details.unavailable` with SKU, requested and available quantities (`order.service.ts:211-224`).

**Rule workflow**:

```
POST /orders -> validate customer/products/active/discount -> INSERT PENDING (no stock read)
PATCH status PENDING -> PAID -> debitStock -> availability check
  -> sufficient: decrement, become PAID
  -> insufficient: 422 INSUFFICIENT_STOCK, remain PENDING
```

**Confidence**: High for behavior; the deferral is explicit in code structure.

---

## 4. Component Structure

### 4.1 File structure

```
src/modules/orders/
├── order.status.ts        # This component: transition table, transition query, terminality,
│                          # stock side-effect predicates (37 lines, no imports beyond the enum)
├── order.service.ts       # Sole consumer: enforces canTransition at line 147, applies stock
│                          # effects at 151-156, writes status and history at 158-167
├── order.schemas.ts       # Independent vocabulary assertion via z.nativeEnum(OrderStatus) (18-21)
├── order.controller.ts    # HTTP adapter, no state logic
├── order.routes.ts        # PATCH /:id/status registration (19-23), authenticate only
└── order.repository.ts    # No status transition logic; no method updates status

prisma/
├── schema.prisma          # OrderStatus enum (16-23), Order.status default PENDING (78),
│                          # OrderStatusHistory model (116-131)
├── migrations/20260519182739_init/migration.sql
│                          # MySQL ENUM for status/fromStatus/toStatus (54, 90-91); no CHECK
└── seed.ts                # Bypasses the state machine, writes arbitrary statuses and a
                           # divergent history graph (255-261)

tests/
├── orders.test.ts         # Only tests that exercise the state machine (indirectly, over HTTP)
├── setup.ts               # Truncates order tables before each test
└── helpers/factories.ts   # Builds app, users, customers, products
```

### 4.2 Exported surface

| Symbol | Kind | Lines | Signature | Production consumers |
|--------|------|-------|-----------|----------------------|
| `canTransition` | function | 12-14 | `(from: OrderStatus, to: OrderStatus) => boolean` | 1 (`order.service.ts:147`) |
| `allowedTransitions` | function | 16-18 | `(from: OrderStatus) => ReadonlyArray<OrderStatus>` | 0 |
| `isTerminal` | function | 20-22 | `(status: OrderStatus) => boolean` | 0 |
| `STOCK_DEBIT_TRANSITION` | const | 24-27 | `{ readonly from: PENDING; readonly to: PAID }` | 1 internal (`shouldDebitStock`) |
| `shouldDebitStock` | function | 29-31 | `(from: OrderStatus, to: OrderStatus) => boolean` | 1 (`order.service.ts:151`) |
| `shouldReplenishStock` | function | 33-37 | `(from: OrderStatus, to: OrderStatus) => boolean` | 1 (`order.service.ts:154`) |
| `transitions` | module-private const | 3-10 | `Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>>` | 3 internal readers |

### 4.3 Internal organization

The file is a single declaration block with no helpers, no classes, no state and no tests co-located. Its internal dependencies form a shallow fan-out from the table:

```
transitions (3-10)
   ├── read by canTransition (13)
   ├── read by allowedTransitions (17)
   └── read by isTerminal (21)

STOCK_DEBIT_TRANSITION (24-27)
   └── read by shouldDebitStock (30)

shouldReplenishStock (33-37) -> literal status comparisons only
```

The table is module-private and no function mutates it. `Readonly` is a compile-time modifier only: the object is not passed through `Object.freeze`, and `allowedTransitions` returns the underlying array reference (`order.status.ts:17`). A caller that casts away `ReadonlyArray` could therefore mutate the shared successor list for the lifetime of the process. No current caller does this, including the unused exports.

---

## 5. Dependency Analysis

### 5.1 Internal dependencies

| Direction | From | To | Symbols | Binding | Notes |
|-----------|------|----|---------|---------|-------|
| Afferent | `src/modules/orders/order.service.ts:17-21` | `order.status.ts` | `canTransition`, `shouldDebitStock`, `shouldReplenishStock` | Compile-time import, runtime function calls | The only production importer in the repository |
| Internal (same file) | `canTransition`, `allowedTransitions`, `isTerminal` | `transitions` | table constant | Module scope | Not an exported dependency |
| Internal (same file) | `shouldDebitStock` | `STOCK_DEBIT_TRANSITION` | pair constant | Module scope | Pins the debit edge |
| Parallel vocabulary | `src/modules/orders/order.schemas.ts:2,19` | `@prisma/client` | `OrderStatus` | Compile-time + runtime enum | Independent of the state machine; both depend on the enum |

No other module imports `order.status.ts`. Auth, users, customers, products, the router registry, the composition root, the error middleware and the seed script do not reference it.

### 5.2 External dependencies

| Dependency | Version | Declaration | Usage in this component | Required at runtime |
|------------|---------|-------------|-------------------------|---------------------|
| `@prisma/client` | 5.22.0 | `package.json:26` | `OrderStatus` enum imported at `order.status.ts:1`; used as runtime object for computed keys and comparisons | Yes (enum object) |
| Generated Prisma Client artifact | Generated from `prisma/schema.prisma` | No direct declaration | Provides the enum member set and therefore the key set of the table | Yes |
| TypeScript compiler | 5.6.3 | `package.json:51` (dev) | Enforces the `Record<OrderStatus, ...>` exhaustiveness at build time | Build only |
| MySQL 8.0 | `mysql:8.0` in `docker-compose.yml` | Not imported by this file | Stores the enum values that `from` will hold at runtime | Indirect |

The module has no dependency on Express, Zod, Pino, the error taxonomy, the repository, the logger or configuration. That makes it independently testable in isolation, although no such test exists.

### 5.3 Dependency chains

```
HTTP PATCH /api/v1/orders/{id}/status
  -> OrderController.changeStatus                 (order.controller.ts:38-46)
  -> OrderService.changeStatus                    (order.service.ts:126-179)
  -> order.status.ts
       -> @prisma/client OrderStatus               (order.status.ts:1)
            -> generated Prisma Client
                 -> prisma/schema.prisma enum OrderStatus
  -> Prisma transaction writes to MySQL
       -> orders.status ENUM column
       -> order_status_history.fromStatus/toStatus ENUM columns
```

Reverse dependency (who depends on the state machine): exactly one class, `OrderService`. The state machine depends on nothing internal, so it is a leaf in the module graph.

### 5.4 Dependency asymmetry worth noting

The vocabulary has two independent imports of the same enum: `order.status.ts:1` and `order.schemas.ts:2`. Validation and policy therefore agree today because they resolve the same generated artifact, but neither imports the other. A change to one path (for example, replacing `z.nativeEnum(OrderStatus)` with a hand-written union) would not be caught by the compiler at the other path. The database `ENUM` and the seed's direct enum use are two further independent assertions of the same six values.

---

## 6. Afferent and Efferent Coupling

### 6.1 Symbol-level coupling

Afferent coupling (Ca) counts distinct importers/callers in the repository; efferent coupling (Ce) counts distinct external or internal modules this symbol depends on.

| Component | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Criticality | Notes |
|-----------|------------------------|------------------------|-------------|-------|
| `canTransition` | 1 (`OrderService.changeStatus`) | 1 (`@prisma/client` enum type) | High | Single enforcement gate for all status changes |
| `shouldDebitStock` | 1 (`OrderService.changeStatus`) | 1 (`@prisma/client`) plus internal constant | High | Controls the only inventory decrement in the system |
| `shouldReplenishStock` | 1 (`OrderService.changeStatus`) | 1 (`@prisma/client`) | High | Controls the only inventory increment path |
| `STOCK_DEBIT_TRANSITION` | 1 (internal: `shouldDebitStock`) | 1 (`@prisma/client`) | Medium | Policy constant, not exported through the HTTP contract |
| `allowedTransitions` | 0 | 1 (`@prisma/client`) | Low | Dead export; no consumer, no test |
| `isTerminal` | 0 | 1 (`@prisma/client`) | Low | Dead export; terminality enforced via `canTransition` |
| `transitions` (private) | 3 internal readers | 1 (`@prisma/client`) | High | Single source of the graph |
| Module `order.status.ts` (file) | 1 (`order.service.ts`) | 1 (`@prisma/client`) | High | Leaf module with minimal fan-in and fan-out |

### 6.2 Module-level metrics (indicative)

For the file treated as a unit:

- `Ca = 1`, `Ce = 1`
- Instability `I = Ce / (Ca + Ce) = 0.5`
- Abstractness `A = 0` (no interfaces or abstract types; all exports are concrete functions and one constant)
- Distance from the main sequence `D = |A + I - 1| = 0.5`

Interpretation without prescription: the module is exactly mid-way between stable and unstable and is fully concrete. Half of its dependency weight is a third-party enum, and its single afferent edge is a service that concentrates all lifecycle orchestration. Stability in the Martin sense is low, but the actual change surface is small because the only external artifact it depends on is a generated enum.

### 6.3 Character of the coupling

- Data coupling: all three live functions exchange only enum values and booleans. No shared mutable state crosses the boundary.
- Control coupling: `canTransition`, `shouldDebitStock` and `shouldReplenishStock` return booleans that steer service branches, so the service's control flow is coupled to the predicates. This is the dominant coupling style of the component.
- No content coupling: the successor arrays returned by `allowedTransitions` would expose internal storage, but that function has no callers.
- No common or stamp coupling: no shared records, DTOs or global objects.
- Temporal coupling: the service must call legality before stock selection and stock before the update; the module itself imposes no order, but the service's correctness depends on it.
- Cohesion: high. All six exports operate on the same transition table and express one responsibility, lifecycle policy. There is no unrelated helper or utility in the file.

---

## 7. Endpoints

Not applicable. The Order State Machine exposes no HTTP, GraphQL, gRPC or message-broker surface. It is a pure in-process module. Its only invocation path is the existing endpoint `PATCH /api/v1/orders/{id}/status`, which is owned by the orders module and analyzed in section 8 as an integration point. No route, controller or handler imports this component directly.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol / Mechanism | Data Format | Error Handling |
|-------------|------|---------|----------------------|-------------|----------------|
| `OrderService.changeStatus` | Internal consumer | Evaluate legality and stock policy | In-process function calls (`order.service.ts:17-21,147-156`) | TypeScript enum values, booleans | Returns boolean; the service throws `InvalidStatusTransitionError` (409) |
| `PATCH /api/v1/orders/{id}/status` | HTTP API (indirect, not owned) | Sole external way to drive a transition | HTTPS/JSON via Express (`order.routes.ts:19-23`) | Body `{ toStatus: OrderStatus, reason?: string }`; path UUID | 400 validation, 401 unauthenticated, 404 missing order, 409 illegal/same status, 422 stock shortage |
| `@prisma/client` `OrderStatus` | External library enum | Supply the vocabulary and key set | Static import (`order.status.ts:1`) | Generated TypeScript enum / string literal union at runtime | Compile-time failure if the enum changes shape |
| MySQL `orders.status` | Persistence | Store the current state | Prisma Client `UPDATE` inside a transaction (`order.service.ts:158`) | `ENUM('PENDING','PAID','PROCESSING','SHIPPED','DELIVERED','CANCELLED')` (`migration.sql:54`) | Database rejects out-of-vocabulary values; no transition constraint |
| `order_status_history` | Audit persistence | Record each accepted transition | Prisma `create` in the same transaction (`order.service.ts:159-167`) | `fromStatus` nullable enum, `toStatus` enum, `changedById`, `changedAt`, `reason` | Any failure aborts the transaction; cascade-deleted with the order |
| `updateOrderStatusSchema` | Boundary validation | Constrain `toStatus` to the vocabulary and `reason` to 500 chars | Zod middleware before the controller (`order.routes.ts:21`) | JSON body | `ZodError` -> 400 `VALIDATION_ERROR` |
| `InvalidStatusTransitionError` / `ConflictError` | Error taxonomy | Translate an illegal edge into HTTP 409 | Class inheritance (`http-errors.ts:33-53`) | `{ code: 'INVALID_STATUS_TRANSITION', details: { from, to } }` | Central error middleware serializes `AppError` (`error.middleware.ts:15-24`) |
| `prisma/seed.ts` | Data writer (bypass) | Populate demo orders in arbitrary statuses | Direct Prisma `create` (`prisma/seed.ts:287-321`) | Historical flow `flowFromStatus` (`prisma/seed.ts:255-261`) | No use of the state machine; no stock effects |
| Webhook / event publication | Absent integration | Notify external systems of `from -> to` | None implemented; the architectural report records a planned outbox seam (`architectural-report:290`) | N/A | N/A |

Notes on the integration boundary:

- The HTTP layer guarantees that `toStatus` is a member of the enum before the state machine is reached, so the module never receives an out-of-vocabulary target from the API.
- The state machine does not validate `from`. It trusts the value loaded from the database. The MySQL `ENUM` and the generated client normally guarantee membership, but if a database is migrated to a newer enum while the running application still holds an older generated client, `transitions[from]` could be `undefined` and `transitions[from].includes(to)` would throw a `TypeError`, surfacing as HTTP 500 rather than 409.
- No event, message, webhook or outbox row is produced by a transition. Any future event consumer would have to be attached to the service, because the state machine has no publication mechanism.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Table-driven finite state machine | `Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>>` mapping each state to its successors | `order.status.ts:3-10` | Declarative lifecycle graph in one data structure |
| Exhaustive mapped type | `Record<OrderStatus, ...>` forces an entry per enum member | `order.status.ts:3` | Compile-time guarantee that no state is omitted |
| Pure predicate / policy functions | `canTransition`, `shouldDebitStock`, `shouldReplenishStock` with two enum inputs | `order.status.ts:12-37` | Side-effect-free decisions, deterministic and testable |
| Policy-mechanism separation | Module decides legality and stock selection; `OrderService` performs writes | `order.status.ts` vs `order.service.ts:126-179` | Keeps domain rules independent of persistence |
| Derived predicate (computed terminality) | `isTerminal` derives from the successor array length instead of a separate list | `order.status.ts:20-22` | Prevents a second terminal-state list from drifting |
| Named transition constant | `STOCK_DEBIT_TRANSITION` gives the debiting edge a name | `order.status.ts:24-27` | Expresses the edge as a referenced rule rather than repeated literals |
| Guard clauses | Same-status check, legality check, stock availability check, delete-state check | `order.service.ts:140-156,184-190` | Fail before side effects; each guard has a distinct error |
| Fail-fast command | Throws on every invalid condition inside the transaction rather than returning partial results | `order.service.ts:131-178` | Atomic status changes |
| Transaction script | Status change orchestration in one method inside `prisma.$transaction` | `order.service.ts:126-179` | Atomicity of status, stock and audit |
| Audit log / event-sourcing-lite | Append-only `OrderStatusHistory` rows with actor and reason | `order.service.ts:159-167`; `prisma/schema.prisma:116-131` | Reconstruct lifecycle and attribute changes |
| Shallow immutability | `Readonly<...>` types only; no deep freeze | `order.status.ts:3` | Compile-time intent, runtime mutability remains possible |
| Anemic state enum | State is a single column; no state objects or polymorphic handlers | `prisma/schema.prisma:78` | Minimal representation; behavior centralized in the service |

Architectural observations:

- The module is a classic finite state machine with an explicit transition relation, but the "machine" does not encapsulate its own state. State lives in the `Order` row, and the transition function is external. This is a stateless-polymorphism-free FSM: policy as data plus predicates.
- Stock policy is not part of the state graph. It is a separate predicate layer that must be kept consistent with the graph by convention. The current implementation is consistent, but the architecture permits divergence: a new edge can be added without any stock consequence, and a stock predicate can be changed without the graph.
- The module has no notion of guards that depend on context (actor, amount, time). All guards are unconditional edges. Context-dependent rules would have to be added as service-level preconditions, which is where BR-19 (delete eligibility) already lives.
- Terminality is encoded twice in intent (empty arrays and `isTerminal`) but only once in effect (`canTransition`). There is no separate "final state" annotation.
- The API contract does not expose the graph. There is no discoverability endpoint, no `allowedTransitions` in the response, and no OpenAPI or documentation artifact listing edges. The seven legal edges are discoverable only from source or by trial.

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact | Evidence |
|------------|----------------|-------|--------|----------|
| Critical | Consumer of the state machine | The transition is evaluated on an unlocked read and applied with an unconditional `UPDATE`; there is no row lock, version column or conditional update on `status` | Two concurrent requests can both observe `PENDING`, both pass `canTransition`, both debit stock (double decrement) and both insert history. For `PAID -> CANCELLED`, double replenishment is possible | `order.service.ts:131-158`; architectural report `architectural-report-2026-11-08_22-46-20.md:354` |
| High | Persistence boundary | The matrix is not enforced in the database: MySQL constrains only the enum vocabulary, and there is no `CHECK` constraint or trigger on transitions | Any writer that bypasses `OrderService` can create an illegal state; the state machine is advisory outside the service | `migration.sql:54,90-91` (no `CHECK`); `order.repository.ts` has no status update; seed writes arbitrary states |
| High | Seed data path | `prisma/seed.ts` writes orders directly in any status and fabricates history from a graph that contradicts the matrix, without stock effects | Seeded `PAID`/`PROCESSING` orders have no prior debit, so cancelling them inflates stock; `DELIVERED` histories show direct `PENDING -> DELIVERED` paths that the API forbids; manual testing and demos can misrepresent policy | `prisma/seed.ts:226-252,255-261`; no `stockQuantity` mutation in seed |
| High | Stock policy consistency | `shouldDebitStock`/`shouldReplenishStock` are independent of `transitions`; no type, test or assertion ties legal edges to stock effects | A future legal edge into `PAID` would not debit stock; a change to the replenish predicate would not be constrained by the graph; the invariant "post-debit states replenish" is conventional | `order.status.ts:3-10` vs `24-37` |
| Medium | Authorization | The state machine and the route are actor-agnostic; the router applies only `authenticate`, with no role gate on transitions | Any authenticated `OPERATOR` can record payment (`PENDING -> PAID`, moving inventory), cancel paid or processing orders, and mark deliveries, with no approval path | `order.routes.ts:14`; no `requireRole` in the orders router; `order.status.ts` receives no actor |
| Medium | Audit of rejected attempts | Rejected transitions are not logged; the central error middleware returns `AppError` before its logging call | Illegal-transition probes, repeated requests and misuse leave no trace in logs or metrics; forensic reconstruction of attempted behavior is impossible | `error.middleware.ts:15-24` vs `57`; no telemetry in `order.status.ts` |
| Medium | Post-shipment lifecycle | `SHIPPED` and `DELIVERED` have no cancellation or return edge, and no return/refund state exists | Returns, refunds and restocking cannot be represented; any such process bypasses `OrderStatusHistory` and the state machine; inventory is never reversed after dispatch | `order.status.ts:7-9,33-37` |
| Medium | Deferred inventory commitment | Orders are accepted without stock reservation or check; the shortage is discovered only at `PENDING -> PAID` | Multiple `PENDING` orders can compete for the same units; accepted orders can fail at payment with 422; no reservation expiry or auto-cancellation exists | `order.service.ts:50-124` (no stock read); `151-153,204-224`; `order.status.ts:24-31` |
| Medium | Delete rule outside the machine | Deletion eligibility (`PENDING` or `CANCELLED`) is duplicated policy that does not consult the state machine, and the read-then-delete sequence is not transactional | Terminal `CANCELLED` records can be destroyed with their audit history; a concurrent status change between read and delete can remove an order that has just become `PAID` | `order.service.ts:181-192`; cascade at `migration.sql:116,122` |
| Low | Dead exports | `allowedTransitions` and `isTerminal` have no consumers and no tests | Dormant surface can drift or be misunderstood; terminality is represented twice in intent; no API can expose allowed next states | `order.status.ts:16-22`; repository-wide search finds no import |
| Low | Runtime vocabulary guard | `transitions[from]` is used without a default; an unknown `from` value causes `TypeError` on `.includes` | If a database value is not in the running client's enum (schema/code version skew), the failure is a 500 instead of a 409, with no domain-specific code | `order.status.ts:12-14` |
| Low | Mutability leak | `Readonly` is shallow and the table is not frozen; `allowedTransitions` returns the internal array reference | A consumer could cast away readonly and mutate a successor array, changing policy for the process lifetime; no current consumer does so | `order.status.ts:3,17` |
| Low | Error-code ambiguity | Same-status and illegal-edge rejections share code `INVALID_STATUS_TRANSITION` and identical details | A client retrying an already-applied transition cannot distinguish it from an illegal move by code; only the message differs | `order.service.ts:140-149`; `http-errors.ts:45-53` |
| Low | Reason quality | Reason is optional on every edge, including cancellation, and has no semantic constraint | Audit rows may lack any explanation; free text cannot be aggregated or validated; cancellation rationale is not guaranteed | `order.schemas.ts:18-21`; `order.service.ts:165` |
| Info | Event publication | No domain event is emitted on transition; the planned outbox seam is explicitly unimplemented | External systems cannot observe changes except by polling; the state machine cannot feed future integrations without a service change | architectural report `:290`; no event code in `order.service.ts` |

---

## 11. Test Coverage Analysis

### 11.1 Test inventory

| File | Type | Count | Relationship to the state machine |
|------|------|-------|-----------------------------------|
| `tests/orders.test.ts` | Integration (HTTP via supertest, real MySQL) | 8 tests | Only suite that exercises the state machine, and only indirectly through `PATCH /orders/{id}/status` |
| `tests/auth.test.ts` | Integration | 7 tests | No order or status coverage |
| `tests/setup.ts` | Fixture | 3 hooks | Truncates `orderStatusHistory`, `orderItem`, `order`, `orderNumberSequence`, `product`, `customer`, `user` before each test (`tests/setup.ts:8-16`) |
| `tests/helpers/factories.ts` | Fixture | 5 helpers | Creates the app, users, customers and products; no order or status helper |

There is no unit test file for `order.status.ts`, no direct import of the module in any test, no mock or stub of the state machine, and no property-based or table-driven matrix test. `vitest.config.ts:3-17` configures no `coverage` provider, reporter or thresholds, so no instrumented line/branch percentage is produced by the project tooling.

### 11.2 Coverage by exported function

| Function | Direct unit tests | Indirect integration tests | Executed branches | Assessment |
|----------|-------------------|----------------------------|-------------------|------------|
| `canTransition` | 0 | 2 (`PENDING -> PAID` allowed at `tests/orders.test.ts:59-87`; `PENDING -> SHIPPED` rejected at `89-107`) | true and false | Both boolean outcomes exercised, but only for 2 of 36 ordered pairs |
| `shouldDebitStock` | 0 | 1 positive (`PENDING -> PAID`, `59-87`) | true; false only incidentally on the replenish path (`134-161`) | True branch proven by stock assertion; false branch not asserted directly |
| `shouldReplenishStock` | 0 | 1 positive (`PAID -> CANCELLED`, `134-161`) | true for `PAID`; false incidentally | `PROCESSING -> CANCELLED` never executed; `PENDING -> CANCELLED` no-effect never asserted |
| `allowedTransitions` | 0 | 0 | none | Function body never executes in the suite |
| `isTerminal` | 0 | 0 | none | Function body never executes in the suite |
| `STOCK_DEBIT_TRANSITION` | 0 | 1 indirect | read on every `shouldDebitStock` call | Exercised, not asserted by identity |
| `transitions` table | 0 | via the above | 3 of 6 status keys used as source (`PENDING`, `PAID`, and `PAID` again) | `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED` never used as sources |

Because coverage is not instrumented, the following is a static estimate, not a measurement: `order.status.ts` has eight executable statements (import/table initialization, three `return` expressions in live functions, two `return` expressions in dead functions, one constant declaration, one predicate expression in `shouldReplenishStock`). The integration suite executes at least six of them, so statement coverage is approximately 75 percent. Branch-level truth-table coverage of the domain, however, is far lower, as shown below.

### 11.3 Transition-pair coverage

| Coverage dimension | Covered | Total | Ratio |
|--------------------|---------|-------|-------|
| Legal edges exercised end-to-end | 2 (`PENDING -> PAID`, `PAID -> CANCELLED`) | 7 | 28.6% |
| Forbidden non-self pairs exercised (rejection) | 1 (`PENDING -> SHIPPED`) | 23 | 4.3% |
| Self-pairs exercised (same-status guard) | 0 | 6 | 0% |
| Terminal states exercised as a source | 0 (`DELIVERED`, `CANCELLED`) | 2 | 0% |
| Source states exercised at all | 2 (`PENDING`, `PAID`) | 6 | 33.3% |
| Target states exercised as a legal target | 2 (`PAID`, `CANCELLED`) | 6 | 33.3% |
| Stock predicate combinations asserted | 2 (debit true, replenish true for `PAID`) | 5 distinct effective pairs (`PENDING->PAID`, `PAID->CANCELLED`, `PROCESSING->CANCELLED`, `PENDING->CANCELLED`, none) | 40% |

Untested legal paths: `PAID -> PROCESSING`, `PROCESSING -> SHIPPED`, `PROCESSING -> CANCELLED`, `SHIPPED -> DELIVERED`, `PENDING -> CANCELLED`.

Untested rules: BR-09 (same status), BR-07 and BR-08 (terminal rejection), BR-14 (no stock effect on `PENDING -> CANCELLED`), BR-15 (no cancellation after shipment), BR-18 (reason persistence, null and non-null), BR-20 (`allowedTransitions`, `isTerminal`), and the concurrent-evaluation behavior behind the Critical risk.

Test-to-rule mapping:

| Test | Lines | Rules evidenced |
|------|-------|-----------------|
| creates an order in PENDING and computes totals | `tests/orders.test.ts:12-40` | BR-02 (initial state and initial history row), BR-01 (vocabulary) |
| rejects an order with non-existent product | `42-57` | None (creation validation) |
| transitions PENDING -> PAID and decrements stock | `59-87` | BR-03, BR-12, BR-17 (history sequence `PENDING`, `PAID`) |
| returns 409 on invalid transition PENDING -> SHIPPED | `89-107` | BR-03, BR-10, BR-11 |
| returns 422 on PENDING -> PAID without enough stock | `109-132` | BR-12, BR-21 (no reservation), rollback behavior |
| replenishes stock on PAID -> CANCELLED | `134-161` | BR-04, BR-13, BR-17 |
| lists orders filtered by status | `163-195` | BR-01 (enum filter only) |
| refuses to delete a non-PENDING/CANCELLED order | `197-218` | BR-19 |

### 11.4 Test quality assessment

The integration suite is strong where it covers a path: the `PENDING -> PAID` test asserts the returned status, the exact history sequence and the product stock value read back from the database (`tests/orders.test.ts:78-86`); the stock-shortage test asserts not only 422 and the error code but the offending SKU and the fact that stock was not modified (`109-132`); the replenishment test asserts both the intermediate and final stock values (`150-160`). These are meaningful behavioral assertions rather than status-code-only checks.

The suite is weak as a regression net for the state machine itself. Changing any untested edge in `order.status.ts` would not fail a test. The two dead exports are never executed, so they could be broken without detection. There is no exhaustive traversal of the 7 legal edges or the 23 forbidden pairs, no test that every status is either terminal or has successors, and no test of the same-status guard. All tests require a live MySQL instance, run sequentially (`fileParallelism: false`, `singleFork: true`), and are therefore slower and more environment-dependent than unit tests would be; there is no in-memory or mocked persistence path.

The absence of coverage tooling means the project cannot state a measured percentage, cannot enforce thresholds, and cannot detect coverage regressions. Any coverage figure in this report is a static estimate based on executed call sites, not an instrumented result.

---

## Appendix A - Scope, Evidence and Ambiguities

### A.1 Boundary decisions

- The component boundary is the module `src/modules/orders/order.status.ts` plus the minimal consumer code required to explain how it is used (`order.service.ts:126-179` for transitions, `181-192` for the adjacent delete rule).
- `OrderService`, `OrderRepository`, the validation middleware, the error taxonomy and the API router registry are analyzed in sibling reports; this report references them only to explain consumption, ordering, error translation and enforcement gaps.
- Persistence models were inspected to establish where state is stored and what constraints exist, not to analyze the repository.

### A.2 Documented ambiguities

1. **Intent behind terminality of `CANCELLED` versus deletability.** The matrix makes `CANCELLED` terminal, but `OrderService.delete` allows deleting it. Both are explicit in code; no comment or repository document states which property takes precedence for retention. The report records the behavior and marks the rationale as Medium confidence.
2. **Absence of post-shipment cancellation.** The graph has no such edge, but no document states that returns are out of scope. Behavior confidence is High; policy intent is Medium.
3. **Consistency of seed history with the matrix.** `flowFromStatus` in `prisma/seed.ts:255-261` allows `PENDING -> DELIVERED` and other skipped paths that the state machine forbids. It is unclear whether this is a deliberate shortcut for demo data or an oversight. The report treats it as a factual divergence, not as a second specification.
4. **Planned event seam.** The architectural report (`architectural-report-2026-11-08_22-46-20.md:290`) records a planned outbox insert inside the status transaction. No such code exists in `order.service.ts`. The report records the absence; it does not assume the planned design is authoritative.
5. **Coverage figures.** No coverage provider is configured, so all percentages in section 11 are static estimates from call-site and branch analysis, not instrumented measurements.

### A.3 Verification commands used (read-only)

| Purpose | Command |
|---------|---------|
| Enumerate source, test and Prisma files | `find src tests prisma -type f` |
| Find all references to the component | `grep -rn "order.status" src tests prisma` |
| Find all vocabulary and transition usages | `grep -rn -E "canTransition|allowedTransitions|isTerminal|shouldDebitStock|shouldReplenishStock|STOCK_DEBIT_TRANSITION|OrderStatus"` |
| Confirm absence of database transition constraints | `grep -n -i "check" prisma/migrations/20260519182739_init/migration.sql` |
| Confirm absence of coverage configuration | `grep -rn "coverage" vitest.config.ts package.json` |
| Confirm seed stock behavior | `grep -n "stockQuantity\|decrement\|increment" prisma/seed.ts` |

### A.4 Files not modified

No project file was created, edited or deleted during this analysis. The only write performed was the creation of this report in `docs/agents/component-deep-analyzer/`.

---

*End of report.*
