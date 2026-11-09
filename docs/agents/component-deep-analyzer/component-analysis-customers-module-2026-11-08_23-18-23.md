# Component Deep Analysis Report — Customers Module

**Component:** Customers Module
**Primary location:** `src/modules/customers/`
**Analysis scope:** `src/modules/customers/` plus directly coupled contracts in `src/app.ts`, `src/routes/index.ts`, `src/middlewares/`, `src/shared/`, `src/modules/orders/`, `prisma/schema.prisma`, `prisma/migrations/`, `prisma/seed.ts` and the project test suite
**Excluded paths:** `node_modules`, `.git`, `dist`, `coverage`
**Analyzed revision:** `e7f6311` (working tree)
**Analysis date:** 2026-11-08 23:18:23
**Report confidence:** High for implemented behavior (all five module files read in full). Low for intended behavior because `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` and `docs/TRACKER.md` are empty placeholders (3 lines each). No written requirements for the Customers domain exist in the repository.

---

## 1. Executive Summary

The Customers Module is one of the five feature modules of the Order Management System (OMS) REST API. It owns the customer master data lifecycle: creation, retrieval, paginated search, partial update and hard deletion of customers. It is a conventional layered module of 5 TypeScript files and 234 lines total, mounted at `/api/v1/customers` and implemented as `routes -> controller -> service -> repository -> Prisma -> MySQL`.

The module is structurally simple and internally cohesive. It has a single persistence aggregate (`Customer`), a single external dependency for data access (Prisma/MySQL) and a small set of cross-cutting dependencies (JWT authentication middleware, Zod validation middleware, shared error taxonomy, pagination helper). Its business logic consists of one real invariant — unique customer e-mail — plus existence checks, input length/format constraints and pagination/search semantics. There are no domain objects, no value objects, no domain events and no business workflows beyond CRUD.

The module is nevertheless business-critical because customers are the reference entity for orders: every order must point to an existing customer (`src/modules/orders/order.service.ts:59-60`), the database enforces an `ON DELETE RESTRICT` foreign key (`prisma/migrations/20260519182739_init/migration.sql:110`), and order read models embed a customer summary (`src/modules/orders/order.repository.ts:57`). A defect in customer deletion/update semantics therefore propagates into the order domain.

### Key findings

| Severity | Finding | Evidence |
|----------|---------|----------|
| High | All customer endpoints require only authentication; there is no role check, ownership check or tenant scoping. Any authenticated `OPERATOR` or `ADMIN` can list, read, modify and delete every customer. | `src/modules/customers/customer.routes.ts:14-24`, `src/middlewares/auth.middleware.ts:49-61` |
| High | Customer PII (name, e-mail, phone, document, full address) is stored in plaintext, returned in full on every read/list/update, and no masking or field projection exists. | `prisma/schema.prisma:40-54`, `src/modules/customers/customer.repository.ts:27-34`, `src/modules/customers/customer.controller.ts:8-43` |
| High | The module has zero direct automated tests. No `tests/customers.test.ts` exists; the only customer usage in tests is a Prisma-level factory that bypasses routes, controller, service, schemas and repository. | `tests/helpers/factories.ts:42-60`, `tests/orders.test.ts`, `tests/setup.ts:14` |
| Medium | Deleting a customer that has orders violates the `ON DELETE RESTRICT` foreign key. Prisma `P2003` is not mapped by the central error middleware, so the API returns `500 INTERNAL_SERVER_ERROR` instead of a 409. | `src/modules/customers/customer.service.ts:60-63`, `prisma/migrations/20260519182739_init/migration.sql:110`, `src/middlewares/error.middleware.ts:37-54` |
| Medium | E-mail uniqueness uses a check-then-write sequence with no transaction or lock. Concurrent requests race; the loser surfaces as generic `CONFLICT` (Prisma `P2002`) instead of the module-specific `EMAIL_ALREADY_USED` code. | `src/modules/customers/customer.service.ts:29-41`, `src/middlewares/error.middleware.ts:37-47` |
| Medium | `document` (CPF/CNPJ) is length-validated only, is not unique, has no checksum validation and has no normalization. Duplicate customers with the same document are permitted. | `src/modules/customers/customer.schemas.ts:19`, `prisma/schema.prisma:45,52` |
| Medium | `PATCH` is a shallow partial: `address` cannot be partially updated. Supplying `address` replaces the whole object and must satisfy all five mandatory sub-fields. | `src/modules/customers/customer.schemas.ts:7-13,23` |
| Medium | Search uses unescaped `contains` predicates and a leading-wildcard pattern, which defeats the indexes and lets `%` / `_` act as SQL `LIKE` wildcards. | `src/modules/customers/customer.repository.ts:12-22`, `prisma/schema.prisma:52` |
| Low | No soft delete, no audit trail, no `createdBy`/`updatedBy` on customers, and no optimistic concurrency. Deletions are irreversible and updates are last-write-wins. | `src/modules/customers/customer.service.ts:43-63`, `prisma/schema.prisma:40-54` |
| Low | E-mail identity is case-insensitive only because of the MySQL `utf8mb4_unicode_ci` collation. The application performs no normalization, so stored casing differs from the effective identity key. | `docker-compose.yml`, `prisma/migrations/20260519182739_init/migration.sql:16-29`, `src/modules/customers/customer.repository.ts:42-44` |

### Role in the system

- Inbound: mounted at `/api/v1/customers` by `src/routes/index.ts:26`, using `buildCustomerRouter` from `src/modules/customers/customer.routes.ts:12-27`.
- Composition: instantiated in `buildControllers` at `src/app.ts:34-36` (`CustomerRepository` -> `CustomerService` -> `CustomerController`).
- Outbound: Prisma client (`@prisma/client` 5.22.0) to MySQL 8.0; shared `NotFoundError`/`ConflictError`; shared `paginated` helper.
- Cross-module: the Orders module reads the `customers` table directly through `Prisma.TransactionClient` rather than through `CustomerRepository` (`src/modules/orders/order.service.ts:59-60`), and order projections embed `{ id, name, email }` from `customers` (`src/modules/orders/order.repository.ts:57`).

---

## 2. Data Flow Analysis

### 2.1 Global request pipeline for every customer request

```
1. Node/Express receives the HTTP request; express.json({ limit: '1mb' }) parses JSON
   (src/app.ts:59)
2. requestLogger assigns or propagates X-Request-Id, starts a timer and registers a
   response-finish log (src/middlewares/request-logger.middleware.ts:5-27)
3. buildApiRouter dispatches /api/v1/customers to buildCustomerRouter
   (src/routes/index.ts:26)
4. Customer router applies authenticate to every route
   (src/modules/customers/customer.routes.ts:14)
   - Missing / malformed Authorization -> 401 UNAUTHORIZED
   - Invalid or expired JWT -> 401 UNAUTHORIZED
   - Valid JWT -> req.user = { id, email, role }
5. Route-level validate() parses the declared Zod source(s) and REPLACES the parsed
   value on the request (body/params/query)
   (src/middlewares/validate.middleware.ts:11-36)
   - Failure -> 400 VALIDATION_ERROR with [{ path, message }]
6. CustomerController handler forwards to CustomerService
   (src/modules/customers/customer.controller.ts:8-52)
7. CustomerService applies business rules and calls CustomerRepository
   (src/modules/customers/customer.service.ts:14-63)
8. CustomerRepository issues Prisma queries against MySQL "customers"
   (src/modules/customers/customer.repository.ts:12-56)
9. Result is serialized as JSON by the controller with the endpoint-specific status
10. Any thrown error goes to next(err) -> errorMiddleware
    (src/middlewares/error.middleware.ts:14-65)
    - AppError -> its statusCode + { error: { code, message, details? } }
    - ZodError -> 400 VALIDATION_ERROR
    - Prisma P2002 -> 409 CONFLICT
    - Prisma P2025 -> 404 NOT_FOUND
    - Anything else -> logged and masked as 500 INTERNAL_SERVER_ERROR
```

### 2.2 List flow (`GET /api/v1/customers`)

```
1. authenticate validates the bearer token (customer.routes.ts:14)
2. validate({ query: listCustomersQuerySchema }) coerces and defaults:
   page=1, pageSize=20, search trimmed (customer.routes.ts:16; schemas:25-29)
   - page < 1 or non-integer -> 400
   - pageSize < 1 or > 100 -> 400
   - search present but empty after trim -> 400
3. CustomerController.list casts req.query to ListCustomersQuery and calls
   CustomerService.list (controller:8-16)
4. CustomerService computes skip = (page - 1) * pageSize and take = pageSize,
   forwards { search, skip, take } (service:14-19)
5. CustomerRepository.buildWhere builds an OR filter over name/email/document
   when search is set (repository:12-22)
6. CustomerRepository runs findMany + count inside a Prisma $transaction
   (repository:24-36)
   - filter applied to both queries
   - orderBy createdAt DESC
   - skip/take applied only to findMany
7. CustomerService wraps items in paginated(): { data, pagination }
   with totalPages = ceil(total / pageSize) (service:20; response.ts:13-24)
8. Controller responds 200 with the envelope, including full customer rows
```

### 2.3 Read-by-id flow (`GET /api/v1/customers/:id`)

```
1. authenticate (routes:14)
2. validate({ params: customerIdParamSchema }) enforces a UUID string
   (routes:17; schemas:3-5)
3. Controller.getById calls CustomerService.getById(req.params.id)
   (controller:18-25)
4. Service calls CustomerRepository.findById -> prisma.customer.findUnique
   (service:23-27; repository:38-40)
5a. null -> throw NotFoundError('Customer') -> 404 NOT_FOUND "Customer not found"
5b. found -> return full Customer record -> 200
```

### 2.4 Create flow (`POST /api/v1/customers`)

```
1. authenticate (routes:14)
2. validate({ body: createCustomerSchema }) (routes:18)
   - validates name/email/phone/document/address lengths and formats
   - strips unknown keys (Zod default "strip")
3. Controller.create calls CustomerService.create(req.body) (controller:27-34)
4. Service calls CustomerRepository.findByEmail(input.email) (service:29-30)
5a. Existing row -> ConflictError('Email already registered', 'EMAIL_ALREADY_USED')
    -> 409 with that machine code (service:31-33)
5b. No row -> CustomerRepository.create with an explicit whitelist of five fields
    (service:34-40; repository:46-48)
6. Prisma INSERT into MySQL "customers"
   - id generated by Prisma @default(uuid()) (CHAR(36))
   - createdAt/updatedAt generated by the database
   - unique index customers_email_key is the final authority
7. 201 Created with the full created Customer JSON
8. Race loser on the unique index -> Prisma P2002 -> errorMiddleware -> 409 CONFLICT
   with message "Unique constraint violation on: email"
```

### 2.5 Update flow (`PATCH /api/v1/customers/:id`)

```
1. authenticate (routes:14)
2. validate({ params: customerIdParamSchema, body: updateCustomerSchema })
   (routes:19-23)
   - body is createCustomerSchema.partial() (shallow)
   - {} is valid; any supplied field must satisfy its full constraint
   - a supplied address must contain all five mandatory sub-fields
3. Controller.update calls CustomerService.update(id, req.body) (controller:36-43)
4. Service calls getById(id) -> existence check -> 404 if missing (service:44)
5. If input.email is provided:
   - findByEmail(input.email) (service:46)
   - if another row owns that e-mail (sameEmail.id !== id) -> 409 EMAIL_ALREADY_USED
   (service:47-50)
   - re-submitting the same e-mail for the same customer is allowed
6. Service builds an explicit update object containing only defined fields
   (service:51-57)
7. Repository.update -> prisma.customer.update({ where: { id }, data })
   (repository:50-52)
8. 200 with the full updated Customer JSON
9. Concurrent deletion between steps 4 and 7 -> Prisma P2025 -> 404 NOT_FOUND
```

### 2.6 Delete flow (`DELETE /api/v1/customers/:id`)

```
1. authenticate (routes:14)
2. validate({ params: customerIdParamSchema }) (routes:24)
3. Controller.delete calls CustomerService.delete(id) (controller:45-52)
4. Service calls getById(id) -> 404 if missing (service:61)
5. Repository.delete -> prisma.customer.delete({ where: { id } }) (repository:54-56)
6a. No referencing orders -> row removed; controller returns 204 with no body
6b. Referencing orders -> MySQL FK orders_customerId_fkey ON DELETE RESTRICT
    -> Prisma P2003 -> NOT mapped by errorMiddleware -> 500 INTERNAL_SERVER_ERROR
6c. Concurrent double delete -> P2025 -> 404 NOT_FOUND
```

### 2.7 Cross-module data flow (customer as an order dependency)

```
1. POST /api/v1/orders with customerId (src/modules/orders/order.schemas.ts)
2. OrderService.create opens prisma.$transaction (order.service.ts:58)
3. Direct tx.customer.findUnique({ where: { id } }) (order.service.ts:59)
   - missing -> NotFoundError('Customer') -> 404 NOT_FOUND
   - this bypasses CustomerRepository and CustomerService
4. Order is persisted with customerId and an embedded customer summary is returned
   (order.service.ts:95-120)
5. Later customer changes are visible in order projections because the join is live
   (order.repository.ts:47-59)
```

### 2.8 Trust boundaries

| Boundary | Crossing point | Controls applied |
|----------|----------------|------------------|
| Network -> application | Express JSON parser, router | 1 MB body limit; JWT authentication; Zod schemas on body/query/params |
| Application -> identity provider | `authenticate` middleware | JWT signature and expiry only; no algorithm pin, issuer or audience |
| Application -> database | Prisma Client | Parameterized SQL; UUID primary key; unique e-mail index; JSON address column |
| Customers Module -> Orders Module | Shared `customers` table and FK | No API call; direct table access by Orders; FK `ON DELETE RESTRICT` |
| Application -> logs | Pino via request logger and error middleware | Structured logs; customer body and PII are not logged |

---

## 3. Business Rules & Logic

## Overview of the business rules

| ID | Rule Type | Rule Description | Location |
|----|-----------|------------------|----------|
| BR-01 | Security / Validation | Every customer endpoint requires a valid bearer JWT | `src/modules/customers/customer.routes.ts:14` |
| BR-02 | Validation | Customer identifiers must be UUID strings | `src/modules/customers/customer.schemas.ts:3-5` |
| BR-03 | Validation | Name is mandatory, 2 to 150 characters | `src/modules/customers/customer.schemas.ts:16` |
| BR-04 | Validation | E-mail is mandatory, RFC-style e-mail format, at most 255 characters | `src/modules/customers/customer.schemas.ts:17` |
| BR-05 | Validation | Phone is mandatory, 8 to 32 characters, no format enforcement | `src/modules/customers/customer.schemas.ts:18` |
| BR-06 | Validation | Document is mandatory, 11 to 20 characters, not unique, no checksum | `src/modules/customers/customer.schemas.ts:19` |
| BR-07 | Validation | Address is mandatory on create and must satisfy five sub-field constraints | `src/modules/customers/customer.schemas.ts:7-13,20` |
| BR-08 | Validation / API | PATCH accepts a shallow partial; an empty body is valid; address is all-or-nothing | `src/modules/customers/customer.schemas.ts:23` |
| BR-09 | Validation | List defaults are page=1, pageSize=20; pageSize max 100; search trimmed and non-empty | `src/modules/customers/customer.schemas.ts:25-29` |
| BR-10 | Business Logic | Search matches name OR e-mail OR document using substring matching | `src/modules/customers/customer.repository.ts:12-22` |
| BR-11 | Business Logic | Lists are ordered by newest first (`createdAt DESC`) | `src/modules/customers/customer.repository.ts:29` |
| BR-12 | Business Logic | E-mail must be unique on create; violation returns 409 `EMAIL_ALREADY_USED` | `src/modules/customers/customer.service.ts:29-33` |
| BR-13 | Business Logic | E-mail must be unique on update, excluding the customer being updated | `src/modules/customers/customer.service.ts:43-50` |
| BR-14 | Business Logic | Read, update and delete require an existing customer; otherwise 404 | `src/modules/customers/customer.service.ts:23-27,44,61` |
| BR-15 | Business Logic | Deletion is hard and is blocked when orders reference the customer | `src/modules/customers/customer.service.ts:60-63`, migration:110 |
| BR-16 | Data Integrity | The database unique index is the final e-mail uniqueness authority | `prisma/schema.prisma:43`, `src/middlewares/error.middleware.ts:37-47` |
| BR-17 | Validation / Security | Unknown input fields are stripped; server-owned fields cannot be set by clients | `src/modules/customers/customer.schemas.ts:15-29` |
| BR-18 | API Contract | Create returns 201, reads/updates 200, delete 204; list uses a pagination envelope | `src/modules/customers/customer.controller.ts:8-52` |
| BR-19 | API Contract | All module errors use the central error envelope and shared error taxonomy | `src/modules/customers/customer.service.ts:3`, `src/middlewares/error.middleware.ts:14-65` |
| BR-20 | Cross-module | An order can only reference an existing customer | `src/modules/orders/order.service.ts:59-60` |
| BR-21 | Persistence | Address is stored as an opaque JSON column with no database-level shape enforcement | `prisma/schema.prisma:46` |
| BR-22 | Authorization | There is no role restriction and no customer/tenant scoping on any endpoint | `src/modules/customers/customer.routes.ts:14-24` |
| BR-23 | Consistency | The paginated total is computed with the same filter and in the same transaction as the page | `src/modules/customers/customer.repository.ts:24-36` |
| BR-24 | Identity | E-mail comparison is case-insensitive because of the MySQL collation, not application logic | `prisma/migrations/20260519182739_init/migration.sql:16-29`, `docker-compose.yml` |
| BR-25 | Search Semantics | Search terms are not escaped for SQL `LIKE`, so `%` and `_` remain wildcards | `src/modules/customers/customer.repository.ts:17-19` |

## Detailed breakdown of the business rules

---

### Business Rule: BR-01 — Authentication is mandatory for all customer endpoints

**Overview**:
Every route in the Customers Module is protected by the shared JWT authentication middleware. No customer data is reachable anonymously.

**Detailed description**:
`buildCustomerRouter` calls `router.use(authenticate)` before registering any route (`src/modules/customers/customer.routes.ts:14`). The middleware requires an `Authorization` header starting with `Bearer `, extracts the token, verifies it with `env.JWT_SECRET`, and hydrates `req.user` with `{ id, email, role }` (`src/middlewares/auth.middleware.ts:27-47`). A missing header, an empty bearer token, an invalid signature or an expired token all produce `UnauthorizedError`, translated to HTTP 401 with code `UNAUTHORIZED`.

Authentication is the only gate in front of customer operations. The module does not check whether the authenticated principal is allowed to act on a specific customer, nor does it compare `req.user.id` with any customer ownership field. Consequently, "authenticated" is equivalent to "fully authorized" for this component. The rule therefore satisfies confidentiality against anonymous access but not authorization within the authenticated population.

The middleware is applied at router level rather than per route, so it is impossible to add a customer endpoint in this router without authentication unless the router itself is refactored. This is a strong, centralized default. The router is mounted under `/api/v1/customers` by `src/routes/index.ts:26`, after `express.json()` and `requestLogger` have run (`src/app.ts:59-67`), meaning the request body is already parsed when authentication executes.

**Rule workflow**:
```
Request -> authenticate -> header present and bearer? -> token present? ->
jwt.verify(token, JWT_SECRET) -> set req.user -> continue to validation
Any failure at any step -> UnauthorizedError -> 401 UNAUTHORIZED
```

---

### Business Rule: BR-02 — Customer identifiers must be UUID strings

**Overview**:
All `/:id` routes accept only strings that Zod recognizes as UUIDs. This is the first line of defense against malformed identifiers and against raw query injection attempts.

**Detailed description**:
`customerIdParamSchema` is `z.string().uuid()` (`src/modules/customers/customer.schemas.ts:3-5`). It is attached to `GET /:id`, `PATCH /:id` and `DELETE /:id` (`src/modules/customers/customer.routes.ts:17,19-23,24`). A value that is not a UUID causes the validation middleware to throw `ValidationError('Validation failed', details)`, which the central error middleware renders as HTTP 400 with code `VALIDATION_ERROR` and a `details` array containing `{ path: "id", message: ... }`.

Because the schema asserts UUID syntax, the service and repository can assume a well-formed identifier. Prisma still parameterizes the `where: { id }` clause, so the rule is not the primary SQL-injection defense; it is a contract and data-quality rule. The actual database column is `CHAR(36)` (`prisma/schema.prisma:41`), and Prisma generates UUIDv4 identifiers by default, so normal client traffic always satisfies the rule.

The UUID requirement has two practical consequences. First, non-UUID identifiers such as numeric IDs or slugs cannot be introduced without changing the schema. Second, malformed identifiers are rejected before any database round trip, which avoids unnecessary queries and avoids leaking whether a syntactically invalid row exists. The validator accepts canonical UUID versions 1 through 5 as implemented by Zod 3.23.8; the application itself only generates version 4.

**Rule workflow**:
```
Route matched -> validate({ params: customerIdParamSchema }) ->
z.string().uuid().parse(req.params) ->
valid -> controller receives id -> service -> repository
invalid -> ZodError -> ValidationError -> 400 VALIDATION_ERROR (details[].path = "id")
```

---

### Business Rule: BR-03 — Customer name must be 2 to 150 characters

**Overview**:
`name` is mandatory on create and, if supplied on update, must contain between 2 and 150 characters.

**Detailed description**:
The create schema declares `name: z.string().min(2).max(150)` (`src/modules/customers/customer.schemas.ts:16`). This mirrors the database column `VARCHAR(150)` (`prisma/schema.prisma:42`), so application validation and persistence capacity agree. One-character names, empty strings and names longer than 150 characters are rejected with HTTP 400 before any query. Names are not trimmed, so a value of two spaces passes the length rule.

On update, `createCustomerSchema.partial()` makes the field optional but preserves both bounds (`src/modules/customers/customer.schemas.ts:23`). The service then forwards the value only when it is not `undefined` (`src/modules/customers/customer.service.ts:51-57`). Because the code checks `!== undefined` rather than truthiness, an empty string would be forwarded, but it can never reach the service because the schema rejects it first.

The rule performs no Unicode normalization, no whitespace collapsing and no character-set restriction. It accepts digits, punctuation and emoji as long as the JavaScript string length is within bounds. Consistency with the database is therefore length-based, matching MySQL's character semantics for `utf8mb4`.

**Rule workflow**:
```
Create: body.name required -> length 2..150 -> accepted, else 400
Update: body.name absent -> unchanged
        body.name present -> length 2..150 -> accepted, else 400
        null -> 400 (z.string rejects null)
```

---

### Business Rule: BR-04 — E-mail must be a valid address of at most 255 characters

**Overview**:
`email` is mandatory on create, validated as an e-mail address and capped at 255 characters. On update it is optional but, when supplied, must satisfy the same constraints.

**Detailed description**:
The schema uses `z.string().email().max(255)` (`src/modules/customers/customer.schemas.ts:17`). This is a syntactic validation: it rejects strings without an `@` and domain shape that Zod's e-mail regular expression does not accept, but it does not perform DNS verification, mailbox verification or domain-existence checks. The 255-character cap matches `VARCHAR(255)` in `prisma/schema.prisma:43`.

The e-mail is also the customer's business key. BR-12 and BR-13 enforce uniqueness on create and update, and the database adds a unique index (`prisma/schema.prisma:43`). Validation happens before the uniqueness lookup, so malformed addresses never consume a database query. The service passes the e-mail through unchanged: no lowercasing, no trimming, no Unicode normalization.

Zod's parser strips unknown keys but does not transform string values, so `"  user@example.com  "` fails the e-mail format check rather than being trimmed to a valid address. A syntactically valid but non-existent domain is accepted and stored. The rule therefore protects data shape and column capacity, not deliverability.

**Rule workflow**:
```
Create: body.email required -> zod.email() -> length <= 255 -> uniqueness check (BR-12)
Update: body.email absent -> unchanged
        body.email present -> zod.email() -> length <= 255 -> uniqueness check (BR-13)
Any format/length failure -> 400 VALIDATION_ERROR with path "email"
```

---

### Business Rule: BR-05 — Phone must be 8 to 32 characters with no format enforcement

**Overview**:
`phone` is mandatory on create and, if supplied on update, must be 8 to 32 characters. The module does not validate country codes, digits-only content or E.164 shape.

**Detailed description**:
The constraint is `z.string().min(8).max(32)` (`src/modules/customers/customer.schemas.ts:18`) and the column is `VARCHAR(32)` (`prisma/schema.prisma:44`). It accepts letters, spaces, parentheses, hyphens and plus signs; `"abcdefgh"` is a valid phone under this rule. The seed data uses the Brazilian international format (`+5511987654321`) and test fixtures use `+5511999999999`, but neither pattern is enforced (`prisma/seed.ts:13`, `tests/helpers/factories.ts:49`).

Because no normalization is applied, the same real phone number can be stored in multiple textual forms (`+55 11 98765-4321`, `5511987654321`, `(11) 98765-4321`). There is no uniqueness constraint, so duplicate phone numbers are allowed. This is consistent with the domain assumption that e-mail, not phone, is the identity key.

The rule is purely a storage-capacity and minimum-quality guard. Any future requirement for SMS delivery or phone-based lookup would need additional validation and formatting that this component does not currently provide. As implemented, the only failure mode is length outside 8..32 or a non-string type.

**Rule workflow**:
```
Create: body.phone required -> string length 8..32 -> stored verbatim
Update: body.phone absent -> unchanged
        body.phone present -> string length 8..32 -> stored verbatim
Failure -> 400 VALIDATION_ERROR with path "phone"
```

---

### Business Rule: BR-06 — Document must be 11 to 20 characters, is not unique and has no checksum validation

**Overview**:
`document` is mandatory, must be between 11 and 20 characters, is stored as a plain string, is indexed but not unique, and is not validated as CPF or CNPJ.

**Detailed description**:
The schema declares `document: z.string().min(11).max(20)` (`src/modules/customers/customer.schemas.ts:19`), while the database stores it in `VARCHAR(20)` with a non-unique index (`prisma/schema.prisma:45,52`). The minimum of 11 accommodates an unformatted CPF; the maximum of 20 accommodates a formatted CNPJ plus punctuation. The seed mixes formatted CPF (`123.456.789-00`) and formatted CNPJ (`12.345.678/0001-90`), both of which fit (`prisma/seed.ts:14,79`).

There is no checksum verification, no digit-only requirement and no normalization of punctuation. `"abcdefghijk"` passes. More importantly, there is no uniqueness rule: two customers can share the same document. The index exists to support substring search (`contains` on document in `customer.repository.ts:19`), not to enforce identity. The service's uniqueness logic covers only `email` (`src/modules/customers/customer.service.ts:30-33`).

This means the component cannot guarantee one customer record per legal entity. Duplicate onboarding is possible, and orders may be attached to different duplicates of the same company or person. Any deduplication or fiscal validation must happen outside this module, and currently nothing in the codebase performs it.

**Rule workflow**:
```
Create: body.document required -> string length 11..20 -> stored verbatim (no uniqueness query)
Update: body.document absent -> unchanged
        body.document present -> string length 11..20 -> stored verbatim
Failure -> 400 VALIDATION_ERROR with path "document"
```

---

### Business Rule: BR-07 — Address is mandatory on create and must be complete and bounded

**Overview**:
`address` is a required nested object with `street`, `number`, `city`, `state` and `zipCode`. Each sub-field has its own length constraint, but there is no semantic validation of state codes or postal-code formats.

**Detailed description**:
`addressSchema` is a Zod object (`src/modules/customers/customer.schemas.ts:7-13`) with `street` 1..200, `number` 1..20, `city` 1..100, `state` exactly 2 characters and `zipCode` 5..15 characters. The create schema requires it as a non-optional property (`:20`). Note that `number` is a string, not a number, which permits values such as `"100A"` and `"s/n"`, but rejects a numeric JSON value because Zod's `z.string()` does not coerce.

The state field is length-only: `"ZZ"` or `"12"` are accepted, and there is no whitelist of Brazilian UFs. The postal code is also length-only: `"abcde"` passes. Street and city are length-bounded but not trimmed. Zod object parsing strips unknown sub-fields by default, so an `address` containing `country` or `complement` loses those fields silently rather than rejecting the request. This does not preserve an "extra address information" channel.

Because `address` is stored in a MySQL `JSON` column (`prisma/schema.prisma:46`), the database validates only JSON syntax, not shape. The five-field contract exists exclusively in the application schema. Any writer bypassing the module, such as a migration script or direct SQL, can store arbitrary JSON, and readers will return it as-is. Within the module, the rule guarantees a complete, bounded address at creation time.

**Rule workflow**:
```
Create: body.address present? -> object with street, number, city, state, zipCode
        each sub-field validated independently
        unknown sub-fields stripped
        all valid -> stored as JSON
        any missing/invalid -> 400 VALIDATION_ERROR with path "address.<field>"
Update: body.address absent -> unchanged
        body.address present -> full object required, same sub-field rules
```

---

### Business Rule: BR-08 — PATCH applies a shallow partial update and permits an empty body

**Overview**:
`updateCustomerSchema` is `createCustomerSchema.partial()`: every top-level field becomes optional, but nested address structure remains all-or-nothing, and `{}` is a valid request.

**Detailed description**:
The schema at `src/modules/customers/customer.schemas.ts:23` reuses the create contract with `.partial()`, which makes `name`, `email`, `phone`, `document` and `address` optional. It does not recursively partially apply to `addressSchema`, so a caller cannot send `{ "address": { "city": "Rio" } }`; the nested schema still requires all five properties. This is a significant API ergonomics constraint: changing one address field requires resending the entire address.

An empty body passes validation and is forwarded to the service. The service spreads only defined fields into the update payload (`src/modules/customers/customer.service.ts:51-57`), so an empty body results in `prisma.customer.update({ where: { id }, data: {} })`, which touches no columns except the Prisma-managed `updatedAt` behavior. The endpoint still returns 200 with the current row, and it still performs the existence lookup. The same behavior occurs if the client sends only unknown keys, since Zod strips them and leaves `{}`.

The rule also controls replace-vs-merge semantics. Sending `name` alone leaves email, phone, document and address untouched. Sending `address` replaces the entire JSON value; there is no deep merge. Because the shallow partial does not allow `null`, fields cannot be cleared through this endpoint. This is consistent with the database schema, where all five fields are non-nullable.

**Rule workflow**:
```
PATCH /:id with body B
  B parsed by updateCustomerSchema (create.partial)
  {} -> allowed
  unknown keys only -> stripped -> {}
  address supplied -> must include all five sub-fields
  name/email/phone/document supplied -> individual constraints apply
  -> service existence check -> optional email uniqueness -> update defined fields only
```

---

### Business Rule: BR-09 — List pagination and search parameters have defaults and hard bounds

**Overview**:
`GET /customers` defaults to page 1 and 20 items per page, caps page size at 100, coerces numeric query strings, and trims the search term when present.

**Detailed description**:
`listCustomersQuerySchema` uses `z.coerce.number().int().min(1).default(1)` for `page`, `z.coerce.number().int().min(1).max(100).default(20)` for `pageSize`, and `z.string().trim().min(1).optional()` for `search` (`src/modules/customers/customer.schemas.ts:25-29`). Coercion means `?page=2` (a string in Express) becomes the integer 2. Missing parameters receive defaults. `page=0`, negative values, decimals, `pageSize=101` and empty-after-trim search all produce 400 `VALIDATION_ERROR`.

The service converts page/pageSize into `skip` and `take` (`src/modules/customers/customer.service.ts:15-19`). There is no explicit maximum for `page`; a very large page produces a very large `skip`, which MySQL will process as an offset scan. There is also no cap on total result count, only on page size, so exhaustive extraction is possible through repeated paginated calls.

The use of `.default()` in Zod means the parsed object always contains `page` and `pageSize`. However, the validation middleware applies the parsed query with `Object.assign(req.query, parsedQuery)` rather than replacing it (`src/middlewares/validate.middleware.ts:17-20`). Any unmatched query keys remain on `req.query`, but the controller casts the object to `ListCustomersQuery`, so extra keys are ignored by the service. The resulting pagination metadata is generated by the shared helper with `totalPages = ceil(total / pageSize)` (`src/shared/http/response.ts:13-20`).

**Rule workflow**:
```
GET /customers?page=&pageSize=&search=
  coerce page -> int >= 1, default 1
  coerce pageSize -> int 1..100, default 20
  trim search -> if present and empty -> 400; if absent -> no filter
  service: skip = (page - 1) * pageSize, take = pageSize
  repository: findMany + count -> service wraps into { data, pagination }
```

---

### Business Rule: BR-10 — Search matches name, e-mail or document by substring

**Overview**:
When `search` is present, the list endpoint returns customers whose name, e-mail or document contains the term. Matching is a disjunction (OR), not a conjunction.

**Detailed description**:
`buildWhere` returns `{ OR: [{ name: { contains: term } }, { email: { contains: term } }, { document: { contains: term } }] }` (`src/modules/customers/customer.repository.ts:12-22`). With MySQL, Prisma renders `contains` as a `LIKE '%term%'` predicate. A customer matches if any one of the three fields contains the substring. There is no ranking, no relevance ordering and no tokenization: the result is then sorted by `createdAt DESC` like any other list.

Case sensitivity is determined by the database collation. The migration declares `utf8mb4_unicode_ci` (`prisma/migrations/20260519182739_init/migration.sql:29`), and the Docker MySQL server sets the same collation (`docker-compose.yml`, `--collation-server=utf8mb4_unicode_ci`). Under `_ci`, `LIKE` is case-insensitive, so `mariana` matches `Mariana`. Accent sensitivity follows the collation as well. The application does not set Prisma's `mode: 'insensitive'`, which is not supported for MySQL anyway.

The implementation does not escape SQL `LIKE` metacharacters in `term`. Prisma parameterizes the value, so this is not SQL injection, but `%` and `_` inside the search string retain wildcard meaning. A search for `%` matches every row, and `_` matches any single character. There is also no index that can serve a leading-wildcard `LIKE`, so the query scans the customers table even though `document` is indexed. This is acceptable at seed scale but is a scalability constraint.

**Rule workflow**:
```
search absent -> where = {}
search present -> where = OR[
  name contains term,
  email contains term,
  document contains term
]
-> applied to both findMany and count
-> results ordered by createdAt DESC
-> pagination envelope returned
```

---

### Business Rule: BR-11 — Customer lists are ordered newest first

**Overview**:
Every list response is sorted by `createdAt` descending, with no client-selectable sort order.

**Detailed description**:
`CustomerRepository.list` applies `orderBy: { createdAt: 'desc' }` (`src/modules/customers/customer.repository.ts:29`). Newly created customers therefore appear first. The endpoint does not accept a `sort` or `order` query parameter; the schema would strip such keys, and the repository ignores anything not in the explicit filter object. This makes the ordering a fixed domain presentation rule rather than a client choice.

There is no secondary sort key. MySQL does not guarantee a stable order among rows with identical `createdAt` values, which can happen when multiple rows are inserted in the same millisecond, including bulk seed inserts. Under pagination, ties can cause a row to appear on two pages or be skipped between requests. The probability is low in interactive use but non-zero for automated imports.

Because there is no `updatedAt` ordering option, clients cannot discover recently changed customers. The rule is simple and predictable for the common "recent customers" use case but limits reporting flexibility. Any additional ordering would require a schema change and repository change, since no dynamic sort is accepted.

**Rule workflow**:
```
list(filters) -> findMany({ where, orderBy: { createdAt: 'desc' }, skip, take })
-> rows returned newest first
-> count computed independently over the same where
```

---

### Business Rule: BR-12 — E-mail addresses must be unique at creation

**Overview**:
`POST /customers` rejects a duplicate e-mail with 409 and the module-specific code `EMAIL_ALREADY_USED`.

**Detailed description**:
`CustomerService.create` performs `findByEmail(input.email)` before inserting (`src/modules/customers/customer.service.ts:29-33`). If a row is found, it throws `new ConflictError('Email already registered', 'EMAIL_ALREADY_USED')`. `ConflictError` maps to HTTP 409 (`src/shared/errors/http-errors.ts:33-37`), and the error middleware renders `{ error: { code: 'EMAIL_ALREADY_USED', message: 'Email already registered' } }` (`src/middlewares/error.middleware.ts:15-24`).

The check is not scoped by anything other than e-mail. Since the module has no tenant concept, a duplicate e-mail is a global conflict across the whole customer base. There is no normalization before the lookup: the exact string is passed to Prisma. Case-insensitive equality is provided by the MySQL collation, so `User@Example.com` and `user@example.com` collide even though the service passes them as different strings. The stored value keeps the first writer's casing.

The pre-check is a usability rule, not the authoritative invariant. Two concurrent creates can both pass `findByEmail` and then race at the unique index. The loser receives Prisma `P2002`, which the central middleware maps to a generic 409 with code `CONFLICT` (`src/middlewares/error.middleware.ts:37-47`), not `EMAIL_ALREADY_USED`. The database constraint is analyzed separately as BR-16.

**Rule workflow**:
```
create(input)
  -> findByEmail(input.email)
     existing -> ConflictError('Email already registered', 'EMAIL_ALREADY_USED') -> 409
     none -> create({ name, email, phone, document, address })
          -> P2002 race -> 409 CONFLICT (generic)
          -> success -> 201
```

---

### Business Rule: BR-13 — E-mail must remain unique on update, excluding the same customer

**Overview**:
When an update changes or resends the e-mail, the service checks that no other customer owns it. Reusing the customer's own current e-mail is allowed.

**Detailed description**:
`CustomerService.update` first verifies that the customer exists (`src/modules/customers/customer.service.ts:44`). If `input.email` is defined, it performs `findByEmail(input.email)` and throws `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` only when a row is found and its `id` differs from the target id (`:45-50`). This explicit self-exclusion avoids a false conflict when the client echoes the unchanged e-mail in a full-representation PATCH.

Note that `if (input.email)` is a truthiness check, not an `undefined` check. Because the schema guarantees a non-empty valid e-mail whenever the field is present, the two are equivalent in practice. The service then forwards every defined field, including email, to the repository (`:51-57`). The repository executes `prisma.customer.update`, which applies the change atomically at the row level but does not re-check uniqueness in application code.

The same race as in creation applies: another customer can take the e-mail between the check and the update, producing a `P2002` and a generic `CONFLICT`. Also, because there is no transaction spanning the existence check, e-mail check and update, a concurrent delete of the target row produces `P2025`, which the middleware maps to 404. The rule enforces logical uniqueness but not a serializable update protocol.

**Rule workflow**:
```
update(id, input)
  -> getById(id): missing -> 404
  -> if input.email defined:
       findByEmail(input.email)
       owner exists && owner.id != id -> 409 EMAIL_ALREADY_USED
       owner is self or none -> continue
  -> repository.update(id, only defined fields)
     -> P2002 -> 409 CONFLICT
     -> P2025 -> 404
     -> success -> 200
```

---

### Business Rule: BR-14 — Read, update and delete require an existing customer

**Overview**:
Any operation targeting a specific customer first resolves the row; if it does not exist, the API returns 404 `NOT_FOUND` with message `Customer not found`.

**Detailed description**:
`CustomerService.getById` is the single existence gate (`src/modules/customers/customer.service.ts:23-27`). It calls `findById`, and if the result is `null`, throws `NotFoundError('Customer')`. `NotFoundError` builds the message `${resource} not found` and the code `NOT_FOUND` (`src/shared/errors/http-errors.ts:27-31`). `update` calls `getById` before any mutation (`:44`), and `delete` does the same (`:61`). `GET /:id` calls `getById` directly (`controller:20`).

This rule gives predictable idempotent semantics for missing resources: reads, updates and deletes all return the same 404 rather than silently succeeding. It also means update and delete issue an extra `SELECT` before the write. There is no `P2025` pre-check shortcut; if the row disappears between the existence check and the write, the database error is translated to 404 by the central middleware.

The rule does not distinguish "never existed" from "deleted a moment ago" and does not expose a soft-deleted state, because there is no soft delete. It also does not reveal the customer's details in the error, which is appropriate. All 404 responses for this module use the same `Customer not found` message regardless of the caller's role.

**Rule workflow**:
```
GET /:id -> getById -> findById -> null -> 404 NOT_FOUND
PATCH /:id -> getById -> null -> 404; exists -> proceed
DELETE /:id -> getById -> null -> 404; exists -> delete
Race after check -> P2025 -> 404 NOT_FOUND
```

---

### Business Rule: BR-15 — Deletion is hard and is blocked when orders reference the customer

**Overview**:
`DELETE /customers/:id` permanently removes the row. If any order references the customer, the foreign key prevents deletion; the module currently surfaces that as a 500.

**Detailed description**:
`CustomerService.delete` checks existence and then calls `prisma.customer.delete` (`src/modules/customers/customer.service.ts:60-63`, `customer.repository.ts:54-56`). There is no `deletedAt`, status flag or archive table. Once the row is removed, the customer is gone; the controller responds `204 No Content` (`customer.controller.ts:45-52`). No cascade is declared on the relation from `Order` to `Customer`; the migration declares `ON DELETE RESTRICT` (`prisma/migrations/20260519182739_init/migration.sql:110`).

That RESTRICT is an important cross-aggregate business rule. A customer with order history cannot be deleted, preserving referential integrity and financial/audit records. The intended HTTP representation would be a 409 conflict explaining that orders reference the customer. However, the error middleware only maps Prisma `P2002` and `P2025` (`src/middlewares/error.middleware.ts:37-54`); a foreign-key violation is `P2003`, which falls through to the generic branch and becomes `500 INTERNAL_SERVER_ERROR` with a logged stack trace. The caller cannot distinguish a legitimate business block from a server fault.

There is also no transactional wrapper around the existence check and delete. Two concurrent deletes both pass `getById`; one succeeds with 204, the other receives `P2025` and therefore 404. That behavior is acceptable idempotency-wise, though the status differs between callers. Deletion is not audited, so after a successful 204 there is no record of who deleted the customer or when.

**Rule workflow**:
```
DELETE /:id
  -> getById: missing -> 404
  -> prisma.customer.delete
     no orders -> row deleted -> 204 No Content
     orders exist -> P2003 -> 500 INTERNAL_SERVER_ERROR (unmapped)
     concurrent delete -> P2025 -> 404
```

---

### Business Rule: BR-16 — The database unique index is the authoritative e-mail invariant

**Overview**:
`customers.email` has a unique database index. Even if the application pre-check is bypassed or raced, MySQL refuses a duplicate e-mail.

**Detailed description**:
`prisma/schema.prisma:43` declares `email String @unique`, and the initial migration creates `UNIQUE INDEX customers_email_key(email)` (`prisma/migrations/20260519182739_init/migration.sql:26`). This is the only hard uniqueness guarantee in the module. The service-level checks in BR-12 and BR-13 improve error quality but are advisory because they are not atomic with the write.

When the index rejects an insert or update, Prisma raises `PrismaClientKnownRequestError` with code `P2002`. The central error middleware inspects `err.meta.target`, joins the field list and returns 409 with code `CONFLICT` and a message such as `Unique constraint violation on: email` (`src/middlewares/error.middleware.ts:37-47`). This differs from the module's own `EMAIL_ALREADY_USED` code. Clients must therefore handle two different 409 codes for the same business condition.

The index also defines identity semantics at the storage layer. Because the column collation is `utf8mb4_unicode_ci`, uniqueness is case-insensitive and accent-insensitive according to MySQL collation rules. The application does not normalize before insert, so `"A@X.com"` and `"a@x.com"` are duplicates; the first stored representation wins. The guarantee applies to all writers, including the seed script and direct Prisma use in tests.

**Rule workflow**:
```
Application pre-check passes -> INSERT/UPDATE -> unique index check ->
  duplicate -> P2002 -> errorMiddleware -> 409 CONFLICT
  unique -> commit
Bypass/race -> same database check still applies
```

---

### Business Rule: BR-17 — Unknown input fields are stripped and server-owned fields are not writable

**Overview**:
Zod object parsing removes undeclared keys from body, query and params. Clients cannot set `id`, `createdAt` or `updatedAt`, and cannot inject arbitrary customer fields.

**Detailed description**:
`createCustomerSchema`, `updateCustomerSchema`, `customerIdParamSchema` and `listCustomersQuerySchema` are all plain `z.object` schemas (`src/modules/customers/customer.schemas.ts:3-29`). Zod's default behavior for objects is to strip unknown keys rather than reject or pass them through. The validation middleware assigns the parsed result back to `req.body` or `req.params` and merges parsed query keys into `req.query` (`src/middlewares/validate.middleware.ts:14-23`). Consequently, a body containing `id`, `createdAt`, `updatedAt` or any unrelated field reaches the service with those keys removed.

The service adds a second layer of whitelisting. `create` explicitly maps only `name`, `email`, `phone`, `document` and `address` (`src/modules/customers/customer.service.ts:34-40`), and `update` builds a new object with only defined known fields (`:51-57`). Even if a future schema change allowed extra keys, the service would not forward them. This is an effective defense against mass assignment.

The rule also removes unexpected nested address keys. `addressSchema` is a nested `z.object`, so fields such as `country`, `complement` or `latitude` are dropped. There is no `.strict()` anywhere, so clients receive no error for typos or unsupported fields; the request succeeds with the extra data silently ignored. This is forgiving but can hide integration mistakes.

**Rule workflow**:
```
raw body/query/params -> z.object.parse -> known keys retained, unknown keys removed
-> body replaced / params replaced / query merged
-> service whitelists create fields and defined update fields
-> repository receives only declared columns
```

---

### Business Rule: BR-18 — Response status codes and payload shapes are fixed per operation

**Overview**:
Create returns 201 with the full customer, list returns 200 with `{ data, pagination }`, read and update return 200 with the full customer, and delete returns 204 with no body.

**Detailed description**:
The controller sets status codes explicitly: `list` 200, `getById` 200, `create` 201, `update` 200, `delete` 204 (`src/modules/customers/customer.controller.ts:8-52`). List results are wrapped by `paginated(items, page, pageSize, total)` into `{ data: Customer[], pagination: { page, pageSize, total, totalPages } }` (`src/shared/http/response.ts:8-24`). All other operations return the raw Prisma `Customer` row, which includes `id`, `name`, `email`, `phone`, `document`, `address`, `createdAt` and `updatedAt`.

There is no response DTO or serializer. The database record is the API representation, so any column added to the `Customer` model later will automatically appear in responses. Conversely, the module cannot hide fields without changing the data access or adding a mapper. This exposes all PII and internal timestamps. It also means the API contract is coupled to the Prisma model.

The delete response is `res.status(204).send()`, which produces no body and no content type. Error responses do not use these success shapes; they use the central error envelope `{ error: { code, message, details? } }`. There is no `Location` header on creation and no `ETag` or `Last-Modified` on reads, so clients cannot discover the new resource URL from headers or perform conditional requests.

**Rule workflow**:
```
GET /          -> 200 { data, pagination }
GET /:id       -> 200 Customer
POST /         -> 201 Customer
PATCH /:id     -> 200 Customer
DELETE /:id    -> 204 (empty)
error path     -> status from AppError / Prisma mapping, body { error: {...} }
```

---

### Business Rule: BR-19 — All errors use the shared error taxonomy and central envelope

**Overview**:
The module throws `NotFoundError` and `ConflictError`, relies on the validation middleware for `ValidationError`, and lets unexpected errors bubble to the central error middleware, which produces a single JSON error shape.

**Detailed description**:
`CustomerService` imports `ConflictError` and `NotFoundError` from `src/shared/errors/index.js` (`src/modules/customers/customer.service.ts:3`). The validation middleware converts `ZodError` into `ValidationError` with an array of `{ path, message }` details (`src/middlewares/validate.middleware.ts:26-33`). The error middleware handles `AppError` instances first, returning `statusCode` and `{ code, message, details? }` (`src/middlewares/error.middleware.ts:15-24`). Unknown errors are logged with the request id and masked as 500 with no internal detail.

The design gives clients a stable machine-readable `error.code`. This module can emit `UNAUTHORIZED`, `VALIDATION_ERROR`, `NOT_FOUND`, `EMAIL_ALREADY_USED`, `CONFLICT` and `INTERNAL_SERVER_ERROR`. Note the inconsistency between `EMAIL_ALREADY_USED` (service pre-check) and `CONFLICT` (database race), described in BR-16. Also, `NotFoundError` uses the generic message `Customer not found` without additional details.

The shared taxonomy is a cross-cutting architectural contract: eleven source files import it, and the error middleware is the only place that translates errors to HTTP. The Customers Module does not write its own error responses and does not catch errors beyond delegating to `next(err)` in each controller handler. This keeps the module consistent with the rest of the API, but it also means any unmapped database error becomes an opaque 500.

**Rule workflow**:
```
service throws AppError subclass
  -> controller catch -> next(err)
  -> errorMiddleware -> status = err.statusCode, body = { error: { code, message, details? } }
ZodError from validate -> ValidationError -> same envelope, 400
Unknown error -> logger.error(...) -> 500 { code: INTERNAL_SERVER_ERROR }
```

---

### Business Rule: BR-20 — An order can only reference an existing customer

**Overview**:
The Orders module validates `customerId` against the `customers` table before creating an order. A missing customer produces 404 `NOT_FOUND`.

**Detailed description**:
`OrderService.create` opens a Prisma transaction and executes `tx.customer.findUnique({ where: { id: input.customerId } })` (`src/modules/orders/order.service.ts:58-60`). If the customer is absent, it throws `NotFoundError('Customer')`, which aborts the transaction and returns 404. The check occurs inside the same transaction as the order, items, history and order-number reservation, so an order cannot be committed without a valid customer at that moment.

The dependency is implemented through direct table access, not through `CustomerRepository` or `CustomerService`. This bypasses the repository boundary and couples the Orders module to the `customers` schema. The customer identifier is validated as a UUID by the order schema before reaching the service, so the lookup is well-formed. The order then persists `customerId`, enforced by the foreign key with `ON DELETE RESTRICT`.

Read paths in Orders embed a customer summary. `OrderRepository.findByIdWithRelations` includes `customer: { select: { id, name, email } }` (`src/modules/orders/order.repository.ts:47-59`), and `OrderService.create`/`changeStatus` return the same projection (`src/modules/orders/order.service.ts:115-119,169-176`). Customer renames and e-mail changes therefore become visible in future order reads, while historical responses are not snapshotted. This is a live-reference model, not an event-sourced or immutable snapshot model.

**Rule workflow**:
```
POST /orders { customerId, items, ... }
  -> transaction begins
  -> tx.customer.findUnique({ id: customerId })
     missing -> NotFoundError('Customer') -> rollback -> 404 NOT_FOUND
     exists -> products, totals, order, items, history, sequence
  -> commit -> 201 with embedded customer summary
```

---

### Business Rule: BR-21 — Address is persisted as opaque JSON

**Overview**:
The `address` column is MySQL `JSON`, so the database guarantees valid JSON syntax but not the five-field structure or value constraints.

**Detailed description**:
`prisma/schema.prisma:46` declares `address Json`, and the migration creates `address JSON NOT NULL` (`prisma/migrations/20260519182739_init/migration.sql:22`). The application schema enforces the object shape at the HTTP boundary, but no database constraint, check or generated column enforces the presence of `street`, `number`, `city`, `state` or `zipCode`. Any non-null JSON value, including an array, string or number, would satisfy the column type if written outside the module.

This is a deliberate flexibility choice for a Brazilian address model, but it has consequences. The module cannot query individual address fields efficiently without JSON path expressions, and there is no index on city or postal code. The API simply returns the parsed JSON object. Schema evolution, such as adding a `country` field, requires no migration but also has no database-level guard against inconsistent historical records.

Within the module boundary, the rule is safe: create and update always pass the validated five-field object. The risk arises from other writers, including `prisma/seed.ts` and test factories, which write address objects directly. The seed uses complete objects (`prisma/seed.ts:15-21`), and the test factory does the same (`tests/helpers/factories.ts:51-57`), so current data is consistent.

**Rule workflow**:
```
HTTP address object -> Zod validation (BR-07) -> Prisma Json column -> MySQL JSON
Read -> Prisma parses JSON -> full object returned to client
Database enforces only NOT NULL and valid JSON syntax
```

---

### Business Rule: BR-22 — Any authenticated actor has full CRUD access to all customers

**Overview**:
There is no role check, ownership check or tenant filter. `requireRole` exists in the codebase but is not applied to any customer route.

**Detailed description**:
The customer router applies only `authenticate` (`src/modules/customers/customer.routes.ts:14`). It never imports or invokes `requireRole`, which is available at `src/middlewares/auth.middleware.ts:49-61`. A valid token for an `OPERATOR` or `ADMIN` grants access to list, read, create, update and delete every customer. The list query has no implicit scoping by `req.user`; the repository returns all rows matching the optional search.

This is the module's most consequential authorization behavior. In a multi-tenant or B2B context, any authenticated operator could enumerate all customers, modify their addresses, change their e-mails and attempt deletion. The customer UUID is not guessable, but the list endpoint makes enumeration unnecessary. There is no audit of who performed mutations, so misuse cannot be traced through the customer record itself.

The architecture report flags the same gap for customers, products and orders. The only role-protected endpoint in the system is `GET /api/v1/users/:id` with `requireRole('ADMIN')`. Within the Customers Module, authorization is entirely binary: unauthenticated (401) or authenticated (allowed). There is no 403 path in practice.

**Rule workflow**:
```
Request -> authenticate -> valid token?
  no -> 401
  yes -> role ignored -> all customer operations allowed
(no requireRole, no ownership predicate, no tenant predicate)
```

---

### Business Rule: BR-23 — Pagination total is computed with the same filter and in one transaction

**Overview**:
The list endpoint retrieves the page and the total count in a single Prisma `$transaction`, using the same `where` clause, so the metadata describes the same logical dataset as the returned page.

**Detailed description**:
`CustomerRepository.list` builds `where` once and then calls `this.prisma.$transaction([findMany({ where, orderBy, skip, take }), count({ where })])` (`src/modules/customers/customer.repository.ts:24-36`). The array form of `$transaction` executes both statements in one database transaction, giving a consistent snapshot under the configured isolation level for the duration of the two queries. This avoids the common bug where the total is computed from a different filter than the page.

The service then passes `total` to `paginated`, which computes `totalPages = pageSize === 0 ? 0 : Math.ceil(total / pageSize)` (`src/shared/http/response.ts:13-20`). A page beyond the last page returns an empty `data` array with a non-zero `total` and valid `totalPages`, which is conventional. When `total` is zero, `totalPages` is zero while `page` remains 1; consumers must tolerate "page 1 of 0".

The transaction protects against filter mismatch, not against concurrent inserts between requests. A customer created after the request completes will change subsequent totals. Also, the repository does not enforce a maximum `take`; the service passes the schema-validated page size, but a direct caller of the repository could request an arbitrarily large page. Within the module's HTTP path, BR-09 bounds it.

**Rule workflow**:
```
buildWhere(search) -> where
$transaction([
  findMany({ where, orderBy: createdAt desc, skip, take }),
  count({ where })
]) -> { items, total }
paginated(items, page, pageSize, total) ->
  { data: items, pagination: { page, pageSize, total, totalPages } }
```

---

### Business Rule: BR-24 — E-mail identity is case-insensitive through database collation

**Overview**:
The application does not normalize e-mail casing. Case-insensitive uniqueness and lookup come from the `utf8mb4_unicode_ci` collation on the `email` column.

**Detailed description**:
The migration creates the `customers` table with `DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci` (`prisma/migrations/20260519182739_init/migration.sql:16-29`), and the Docker MySQL server sets `--collation-server=utf8mb4_unicode_ci` (`docker-compose.yml`). Under this collation, equality and `LIKE` comparisons ignore case for the relevant character repertoire. The schema does not specify a binary collation, so the default applies.

`findByEmail` passes the raw input to `prisma.customer.findUnique({ where: { email } })` (`src/modules/customers/customer.repository.ts:42-44`). Prisma generates an equality predicate, and MySQL resolves it using the column collation. Therefore `"USER@EXAMPLE.COM"` finds a stored `"user@example.com"`. The unique index likewise treats them as duplicates. The stored value is exactly what the client sent on the first successful create; later reads return that original casing, not the casing used in the lookup.

This creates a hidden dependency on infrastructure configuration. If the database were recreated with a case-sensitive collation, application behavior would silently change: uniqueness would become case-sensitive and lookups would become exact-match. Likewise, accent-insensitivity (`_ci` collations) may merge values that differ only by accents. The application has no explicit normalization layer to guarantee stable identity independent of the database, and the API does not document the behavior.

**Rule workflow**:
```
create/update email value -> stored verbatim in utf8mb4_unicode_ci column
findByEmail(value) -> MySQL equality under _ci collation ->
  matches rows differing only by case/accent
unique index -> conflicts across case/accent variants
```

---

### Business Rule: BR-25 — Search terms retain SQL LIKE wildcard semantics

**Overview**:
The search term is embedded in Prisma `contains` predicates without escaping `%` or `_`, so those characters act as wildcards rather than literals.

**Detailed description**:
`buildWhere` assigns `term = filters.search` and uses `{ name: { contains: term } }`, `{ email: { contains: term } }` and `{ document: { contains: term } }` (`src/modules/customers/customer.repository.ts:14-20`). Prisma sends the value as a bound parameter and wraps it for `LIKE '%...%'`. Bound parameters prevent SQL injection, but MySQL still interprets `%` and `_` inside the parameter value as pattern metacharacters. There is no `ESCAPE` clause or application-level escaping of `\`, `%` or `_`.

A search for `%` therefore matches every customer, and a search for `_` matches any customer with at least one character in the searched fields. A user searching a literal document containing `%` or `_` cannot target it precisely. The impact is limited to result breadth, not data exposure beyond what the list endpoint already returns, but it can surprise clients and enables cheap full-result scans.

The leading wildcard also prevents efficient index use. The `document` index (`prisma/schema.prisma:52`) cannot serve `LIKE '%term%'`, and `name`/`email` have no dedicated index for this pattern. At current scale this is invisible; at large scale it becomes a table scan per list request. The rule is implicit and should be treated as medium confidence for exact Prisma-to-MySQL SQL generation, but the absence of escaping in the source is unambiguous.

**Rule workflow**:
```
search = "50%"
-> where OR[
  name contains "50%",
  email contains "50%",
  document contains "50%"
]
-> SQL LIKE '%50%%' -> wildcard interpretation -> broad matches
```

---

## 4. Component Structure

```
src/modules/customers/
├── customer.routes.ts        # 27 lines - Express router factory, auth + per-route Zod validation
├── customer.controller.ts    # 53 lines - HTTP adapters for 5 operations, status codes, next(err)
├── customer.service.ts       # 64 lines - business rules: existence, e-mail uniqueness, pagination mapping
├── customer.repository.ts    # 57 lines - Prisma queries: search, transaction, CRUD
└── customer.schemas.ts       # 33 lines - Zod contracts: id, address, create, update, list query
```

### File responsibilities and key symbols

| File | Key symbols | Responsibility | Notes |
|------|-------------|----------------|-------|
| `src/modules/customers/customer.routes.ts` | `buildCustomerRouter(controller)` | Binds HTTP verbs/paths, applies `authenticate` and `validate`, returns `Router` | Factory pattern; router-wide auth at line 14 |
| `src/modules/customers/customer.controller.ts` | `CustomerController` with `list`, `getById`, `create`, `update`, `delete` | Request in / response out; no business logic | Class-field arrow handlers preserve `this`; all errors call `next(err)` |
| `src/modules/customers/customer.service.ts` | `CustomerService` with `list`, `getById`, `create`, `update`, `delete` | Business rules and orchestration | Depends on concrete repository; imports shared errors and pagination |
| `src/modules/customers/customer.repository.ts` | `CustomerRepository`, `CustomerListFilters` | All SQL/Prisma access; builds `where`; executes transaction | Only file in module aware of Prisma query shapes |
| `src/modules/customers/customer.schemas.ts` | `customerIdParamSchema`, `createCustomerSchema`, `updateCustomerSchema`, `listCustomersQuerySchema`, input types | Input contracts and TypeScript inference | Update is shallow partial of create |

### Internal layering and call direction

```
customer.routes.ts
  -> authenticate (src/middlewares/auth.middleware.ts)
  -> validate (src/middlewares/validate.middleware.ts)
  -> customer.schemas.ts
  -> customer.controller.ts
       -> customer.service.ts
            -> customer.repository.ts
                 -> @prisma/client -> MySQL customers
            -> src/shared/errors/index.ts
            -> src/shared/http/response.ts
```

The layering is strictly downward. The repository does not import the service or controller; the service does not import Express; the controller does not import Prisma. This yields high cohesion and clean separation of transport, business logic and persistence. The only boundary leak is that the controller casts `req.query` to the schema type using `as unknown as ListCustomersQuery` (`src/modules/customers/customer.controller.ts:10`), relying on the middleware to have parsed it correctly.

### Code metrics (static)

| Metric | Value |
|--------|-------|
| Files | 5 |
| Total lines | 234 |
| Classes | 3 (`CustomerController`, `CustomerService`, `CustomerRepository`) |
| Exported schemas | 4 |
| Public operations | 5 (one per HTTP verb/path combination, except get/create share naming) |
| Longest method | `CustomerService.update` (~16 lines) |
| Cyclomatic complexity | Low; no nested loops or deep branching |
| `any` usage | None |
| TypeScript strictness | `strict: true`, `noUncheckedIndexedAccess: true` (`tsconfig.json:9-10`) |
| Test files for this module | 0 direct |

---

## 5. Dependency Analysis

### Internal dependencies

```
src/app.ts
  -> CustomerRepository -> @prisma/client (PrismaClient type)
  -> CustomerService    -> CustomerRepository, shared/errors, shared/http, customer.schemas
  -> CustomerController -> CustomerService, customer.schemas

src/routes/index.ts
  -> CustomerController (type)
  -> buildCustomerRouter

customer.routes.ts
  -> express Router
  -> middlewares/auth.middleware (authenticate)
  -> middlewares/validate.middleware (validate)
  -> customer.schemas (4 schemas)
  -> customer.controller (type)

customer.controller.ts
  -> express (RequestHandler type)
  -> customer.service (type)
  -> customer.schemas (ListCustomersQuery type)

customer.service.ts
  -> @prisma/client (Customer type)
  -> customer.repository (type)
  -> shared/errors (ConflictError, NotFoundError)
  -> customer.schemas (CreateCustomerInput, UpdateCustomerInput, ListCustomersQuery)
  -> shared/http/response (paginated, PaginatedResponse)

customer.repository.ts
  -> @prisma/client (Customer, Prisma, PrismaClient types)
```

### Internal dependents (who imports the module)

| Dependent | What it imports | Purpose |
|-----------|-----------------|---------|
| `src/app.ts:10-12` | `CustomerRepository`, `CustomerService`, `CustomerController` | Manual composition of the module |
| `src/routes/index.ts:4,9` | `CustomerController` type, `buildCustomerRouter` | Mounts `/api/v1/customers` |

No other module imports Customers code. The Orders module depends on the `customers` table, not on the module's classes (`src/modules/orders/order.service.ts:59`).

### External dependencies

| Dependency | Version | Purpose in this component | Coupling |
|------------|---------|---------------------------|----------|
| `express` | 4.21.1 | `Router`, `RequestHandler` | Router/controller layer only |
| `zod` | 3.23.8 | Request contracts and coercion/defaults | Schemas file only |
| `@prisma/client` | 5.22.0 | Typed data access and query building | Repository, service types |
| `prisma` CLI | 5.22.0 | Schema/migrations (development) | Not runtime for module |
| MySQL | 8.0 (`docker-compose.yml`) | Persistence, unique index, FK RESTRICT, JSON column, collation | Via Prisma only |
| `jsonwebtoken` | 9.0.2 | Not imported by module; used by auth middleware | Transitive runtime dependency |
| `uuid` | 11.0.3 | Not imported by module; request IDs and Prisma UUID generation | Transitive |

### Configuration dependencies

- `DATABASE_URL` is required for Prisma to reach MySQL (`src/config/env.ts:7`).
- `JWT_SECRET` and `JWT_EXPIRES_IN` govern the authentication middleware used by the router (`src/config/env.ts:8-9`).
- `NODE_ENV` controls Prisma logging (`src/config/database.ts:6`).
- MySQL collation `utf8mb4_unicode_ci` is provisioned externally and affects e-mail/search semantics (BR-24).
- No feature flags, environment-specific module configuration or runtime tunables exist in the module.

---

## 6. Afferent and Efferent Coupling

Method: counts are static and derived from `import` / `import type` statements. External npm packages are excluded from Ca/Ce and reported separately. "Internal" counts refer to files inside `src/modules/customers/`; "external" counts refer to files elsewhere in `src/`. The module-level figures match the architectural report (Ca = 2, Ce = 4).

| Component | Afferent Coupling (external / internal) | Efferent Coupling (internal / external npm) | Criticality |
|-----------|------------------------------------------|----------------------------------------------|-------------|
| `CustomerService` | 1 / 1 (`app.ts`, `customer.controller.ts`) | 2 internal (`repository`, `schemas`) + 2 shared (`errors`, `http`) / 1 npm (`@prisma/client` type) | High |
| `CustomerRepository` | 1 / 1 (`app.ts`, `customer.service.ts`) | 0 internal / 1 npm (`@prisma/client`) | Medium |
| `CustomerController` | 2 / 1 (`app.ts`, `routes/index.ts`, `customer.routes.ts`) | 2 internal (`service`, `schemas`) / 1 npm (`express` type) | Low |
| `customer.schemas.ts` | 0 / 3 (`routes`, `controller`, `service`) | 0 / 1 npm (`zod`) | High (contract) |
| `buildCustomerRouter` | 1 / 0 (`routes/index.ts`) | 2 middlewares + 2 module files / 1 npm (`express`) | Low |
| Customers Module (boundary) | 2 (`app.ts`, `routes/index.ts`) | 4 (`auth.middleware`, `validate.middleware`, `shared/errors`, `shared/http`) | High (customer master data) |

### Interpretation

- The module is **stable and loosely coupled** at the boundary: only the composition root and the route registry know it exists. It does not import any other feature module, so changes in Auth, Users, Products or Orders do not compile-break it.
- The **service is the highest-risk internal unit**: it carries all business rules and is imported by the controller and composition root. It is still small (64 lines) and easy to reason about.
- The **schemas file has high internal fan-in** (imported by routes, controller and service). It is the de facto contract of the module; changing a schema can affect all three layers even though only validation and types are involved.
- Coupling to **shared infrastructure is appropriate**: errors and pagination are stable cross-cutting contracts used by other modules as well.
- The module's efferent coupling is **lower than its true architectural dependency**, because the Orders module bypasses it to read/write the `customers` table directly. That coupling is hidden from static import analysis and is documented separately as a boundary leak.
- Cohesion is **functional/high**: all five files exist solely for the customer aggregate, and there is no unrelated logic.

---

## 7. Endpoints

Base path: `/api/v1/customers`. All endpoints require `Authorization: Bearer <JWT>`.

| Endpoint | Method | Description | Success | Main Error Codes |
|----------|--------|-------------|---------|------------------|
| `/api/v1/customers` | GET | List customers with optional `search`, paginated via `page` and `pageSize` | 200 `{ data, pagination }` | 400 `VALIDATION_ERROR`, 401 `UNAUTHORIZED` |
| `/api/v1/customers/{id}` | GET | Get one customer by UUID | 200 `Customer` | 400 `VALIDATION_ERROR`, 401 `UNAUTHORIZED`, 404 `NOT_FOUND` |
| `/api/v1/customers` | POST | Create a customer | 201 `Customer` | 400 `VALIDATION_ERROR`, 401 `UNAUTHORIZED`, 409 `EMAIL_ALREADY_USED` / `CONFLICT` |
| `/api/v1/customers/{id}` | PATCH | Partially update a customer; body fields optional | 200 `Customer` | 400 `VALIDATION_ERROR`, 401 `UNAUTHORIZED`, 404 `NOT_FOUND`, 409 `EMAIL_ALREADY_USED` / `CONFLICT` |
| `/api/v1/customers/{id}` | DELETE | Hard-delete a customer | 204 no body | 401 `UNAUTHORIZED`, 404 `NOT_FOUND`, 500 `INTERNAL_SERVER_ERROR` (FK `P2003`), 500 for other unmapped DB errors |

### Request contract summary

| Operation | Source | Schema | Notes |
|-----------|--------|--------|-------|
| List | query | `listCustomersQuerySchema` | `page` int >= 1 default 1; `pageSize` int 1..100 default 20; `search` trimmed non-empty optional |
| Get | params | `customerIdParamSchema` | `id` UUID |
| Create | body | `createCustomerSchema` | All fields required; address object with 5 required fields |
| Update | params + body | `customerIdParamSchema` + `updateCustomerSchema` | Shallow partial; `{}` accepted; address all-or-nothing |
| Delete | params | `customerIdParamSchema` | `id` UUID |

### Example response envelope (list)

```json
{
  "data": [
    {
      "id": "3f1c...",
      "name": "Mariana Silva Santos",
      "email": "mariana.santos@exemplo.com.br",
      "phone": "+5511987654321",
      "document": "123.456.789-00",
      "address": {
        "street": "Rua das Acácias",
        "number": "125",
        "city": "São Paulo",
        "state": "SP",
        "zipCode": "01310-100"
      },
      "createdAt": "2026-11-08T23:00:00.000Z",
      "updatedAt": "2026-11-08T23:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol / Driver | Data Format | Error Handling |
|-------------|------|---------|-------------------|-------------|----------------|
| MySQL 8.0 `customers` table | Database | Persist and query customer master data | TCP / Prisma Client 5.22 (SQL) | Relational rows; `address` as JSON; `id` CHAR(36) | Prisma exceptions bubble to central middleware; only `P2002`/`P2025` mapped; `P2003` becomes 500 |
| JWT authentication middleware | Internal middleware | Verify bearer token and hydrate `req.user` | In-process call | JWT payload `{ sub, email, role, iat, exp }` | `UnauthorizedError` -> 401; no algorithm pin or audience check |
| Zod validation middleware | Internal middleware | Validate/coerce body, query and params | In-process call | Parsed JS objects with unknown keys stripped | `ValidationError` -> 400 with `details[]` |
| Central error middleware | Internal middleware | Translate `AppError`, `ZodError`, Prisma codes to HTTP | In-process call | `{ error: { code, message, details? } }` | Logs unhandled errors with request id; masks as 500 |
| Pagination response helper | Shared utility | Build `{ data, pagination }` envelope | In-process call | JSON object | No failure mode; guards `pageSize === 0` |
| Shared error taxonomy | Shared kernel | Standard status/code contract | In-process import | TypeScript classes | `AppError` subclasses carry status and code |
| Orders Module (indirect) | Internal module / shared table | Validate customer existence for orders; embed customer summary in order reads | Direct Prisma transaction client, no HTTP call | Relational rows; projection `{ id, name, email }` | `NotFoundError('Customer')` -> 404; FK `RESTRICT` blocks customer deletion |
| Prisma migration / seed | Development tooling | Create schema and demo customers | Prisma CLI / `tsx` | SQL DDL, TypeScript data | Seed is destructive; no environment guard |
| Test DB harness | Test infrastructure | Truncate customers between tests, create fixtures directly | Prisma | TypeScript fixtures | No `NODE_ENV`/URL guard; direct writes bypass module rules |

### Notable integration characteristics

- There are **no outbound HTTP calls**, no message broker, no event bus, no cache and no webhook integration in this module.
- All integration is **synchronous and in-process** except the MySQL connection.
- The **Orders Module bypasses the Customers Module API and repository**, creating an implicit schema-level integration. Any change to the `customers` table or the meaning of `customerId` must be coordinated with `src/modules/orders/order.service.ts` and `order.repository.ts`.
- The **error contract is the main cross-module API contract** for failures; success payloads are raw Prisma records with no versioning.
- **No idempotency keys**, retries, circuit breakers or timeouts are configured at the module level. Prisma/MySQL connection behavior is governed by the client and `DATABASE_URL` options, which are not customized in `src/config/database.ts`.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered architecture (3-tier + schema) | routes -> controller -> service -> repository | `src/modules/customers/*` | Separation of transport, business logic and persistence |
| Repository pattern | `CustomerRepository` encapsulates all Prisma access | `src/modules/customers/customer.repository.ts` | Keep ORM details out of service; centralize query building |
| Dependency injection (constructor) | `CustomerService(repository)`, `CustomerController(service)` | `customer.service.ts:12`, `customer.controller.ts:6` | Testability and explicit dependencies; wiring done in `src/app.ts:34-36` |
| Composition root | `buildControllers(prisma)` news up repository/service/controller | `src/app.ts:26-53` | Single place for object graph construction; no DI container |
| Factory function for router | `buildCustomerRouter(controller)` | `src/modules/customers/customer.routes.ts:12-27` | Decouple router creation from controller instance |
| Schema-first validation / DTO | Zod schemas define request contracts and inferred types | `src/modules/customers/customer.schemas.ts` | Validate at boundary; derive TypeScript types; strip unknown keys |
| Middleware chain | `authenticate` then `validate` then handler | `customer.routes.ts:14-24` | Reusable cross-cutting concerns |
| Shared kernel / error taxonomy | `AppError` subclasses + central translator | `src/shared/errors/*`, `src/middlewares/error.middleware.ts` | Uniform machine-readable errors across modules |
| Pagination envelope | `paginated()` and `buildPagination()` | `src/shared/http/response.ts` | Consistent list response shape |
| Transactional read consistency | Array-form `prisma.$transaction([findMany, count])` | `customer.repository.ts:26-34` | Page and total computed from the same filter/snapshot |
| Error boundary / fail-fast at middleware | Controllers only call `next(err)` | `customer.controller.ts:8-52` | Centralized error mapping and logging |
| Anemic domain model | Prisma `Customer` is a data record; no domain entity or value objects | `prisma/schema.prisma:40-54` | Simplicity; business rules live in service functions |

### Architectural decisions observed

1. **Feature-module organization with identical internal shape across all five modules.** Customers, Products, Users and Orders share the same file naming and layer responsibilities, which makes navigation predictable and allows cross-module consistency review. The cost is duplicated boilerplate (the update-field spread in `customer.service.ts:51-57` is nearly identical to `product.service.ts:53-60`).

2. **Concrete dependencies, no interfaces/ports.** The service depends on the concrete `CustomerRepository` class, not an abstraction. Constructor injection still permits substituting a test double structurally in TypeScript, but there is no formal contract file. This is consistent with the rest of the codebase and keeps the module small.

3. **Validation at the HTTP boundary only.** There is no domain-level validation layer. Service methods assume validated input; direct callers (tests, scripts, future workers) must satisfy the same invariants themselves. The repository accepts `Prisma.CustomerUncheckedCreateInput`, which is broader than the service's input type.

4. **Database-enforced integrity for identity and references.** Unique e-mail and FK `RESTRICT` are delegated to MySQL, which is appropriate for concurrency, but the error translation layer only covers two Prisma codes, so some integrity outcomes are poorly represented in HTTP.

5. **No soft delete or audit model.** The component follows a straightforward CRUD lifecycle. There is no temporal modeling, no `deletedAt`, no change history and no actor attribution. This aligns with the codebase's only audit model, which is specific to order status history.

6. **No CQRS, caching or read model.** List and read both hit MySQL directly with the same Prisma model. The module is entirely synchronous request/response.

### Quality attributes assessment

| Attribute | Assessment | Evidence |
|-----------|------------|----------|
| Cohesion | High | All files serve the customer aggregate; no unrelated logic |
| Coupling | Low at module boundary; medium to database semantics | Only 2 external dependents; hidden coupling via Orders direct table access |
| Maintainability | Good | 234 lines, consistent layering, typed schemas, small methods |
| Testability | Structurally good, operationally untested | Constructor injection; no interfaces; zero tests for the module |
| Security | Weak | Auth only; no roles/ownership; full PII exposure; plaintext storage |
| Performance | Adequate at small scale | `LIKE '%term%'` scans; no index for name/email search; pagination bounded |
| Observability | Minimal | Access logs and error logs only; no customer-specific metrics/traces |
| Consistency | Good for reads, weak for write races | $transaction for list; check-then-write for create/update/delete |
| API evolvability | Limited | Raw Prisma model as response; no versioning/DTO; shallow PATCH |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| High | Authorization | No `requireRole`, ownership check or tenant scoping on any customer endpoint | Any authenticated actor can enumerate, read, modify or delete all customers; no B2B isolation |
| High | Data protection | Full PII returned on list/read/update; plaintext name, e-mail, phone, document and address in MySQL | Privacy/compliance exposure; no masking or field-level access control |
| High | Test coverage | No `tests/customers.test.ts`; factory writes directly via Prisma (BR-12 to BR-25 untested) | Regressions in validation, uniqueness, update and delete semantics would go undetected |
| High | Delete semantics | FK `P2003` is not mapped in the central error middleware | Deleting a customer with orders returns 500 instead of a business-meaningful 409; logs a false server fault |
| Medium | Concurrency | Create/update e-mail uniqueness uses check-then-write with no transaction or lock | Race produces generic `CONFLICT` instead of `EMAIL_ALREADY_USED`; inconsistent client handling |
| Medium | Identity data | `document` is not unique, not normalized and not checksum-validated | Duplicate customer records for the same legal entity; unreliable fiscal identity |
| Medium | Update contract | Shallow partial update prevents partial address changes | Clients must resend the entire address; higher risk of overwriting concurrent changes |
| Medium | Search | `contains` does not escape `%`/`_` and uses leading wildcards | Unexpected broad matches; full table scans; index on document is ineffective for search |
| Medium | Error handling | Body-parser malformed-JSON errors and Prisma `P2003`/other known errors fall through to 500 | Client mistakes and legitimate conflicts are misreported as server failures |
| Medium | Auditability | No `createdBy`, `updatedBy`, soft delete or change history for customers | Cannot trace who changed or deleted master data; deletion is irreversible |
| Low | Concurrency | Existence check and subsequent update/delete are separate statements | Concurrent delete causes a 404 after a successful check; two deletes see different statuses |
| Low | Pagination stability | Only `createdAt DESC` ordering, no tiebreaker | Rows with identical timestamps may shift across pages |
| Low | Pagination semantics | `totalPages = 0` when `total = 0`, while `page = 1` | Consumers must special-case empty results |
| Low | Query cost | No maximum `page`; large offsets possible | Slow deep pagination at scale |
| Low | Duplication | Update field-spread logic duplicates the Products module | Two copies of the same pattern can drift |
| Low | API contract | Raw Prisma model as response; no DTO/serializer | New columns leak automatically; no response versioning or field hiding |
| Low | Validation | State and postal code are length-only; phone has no format rule; names are not trimmed | Inconsistent data quality; no UF/CEP correctness |
| Low | E-mail normalization | Casing identity depends on MySQL collation | Changing collation or database engine changes behavior silently |
| Low | Observability | No metrics, traces or audit logs specific to customer operations | Limited ability to detect enumeration or bulk mutation |
| Low | Documentation | No OpenAPI/schema documentation; `docs/*.md` are placeholders | Consumers must read code to learn contracts and error codes |
| Info | Dependencies | `zod` pinned at 3.23.8; UUID acceptance version depends on Zod internals | A future Zod upgrade could change accepted UUID versions/error messages |

### Code smells observed

- **Duplicated update-construction logic** between `customer.service.ts:51-57` and `product.service.ts:53-60` (copy-paste specialization).
- **Primitive obsession**: e-mail, phone, document, postal code and state are plain strings with no value objects; validation is scattered between schema and database semantics.
- **Anemic model**: `Customer` has no behavior; all invariants live in `CustomerService`.
- **Hidden temporal coupling**: the controller assumes validation middleware has already populated `req.query` defaults and transformed types (`customer.controller.ts:10` cast).
- **Leaky persistence model**: API responses are Prisma records, so persistence changes become API changes.
- **Magic behavior**: Zod `.strip()` silently drops unknown fields, including nested address fields, with no client feedback.
- **Inconsistent error vocabulary**: `EMAIL_ALREADY_USED` vs `CONFLICT` for the same condition.

---

## 11. Test Coverage Analysis

### Test inventory relevant to the component

| Test file / helper | Relevance to Customers Module | Type |
|--------------------|-------------------------------|------|
| `tests/customers.test.ts` | Does not exist | N/A |
| `tests/orders.test.ts` | 8 integration tests; all create customers via factory and pass `customer.id` to orders | Indirect data dependency only |
| `tests/auth.test.ts` | 7 integration tests; no customer usage | Unrelated |
| `tests/helpers/factories.ts:42-60` | `createTestCustomer()` inserts directly with `prisma.customer.create` | Test fixture, bypasses the module |
| `tests/setup.ts:8-16` | Deletes `customer` rows before each test (after orders/items/history) | Teardown/fixture isolation |
| `prisma/seed.ts:9-140,213-216` | Seeds 10 customers; exercises database constraints only | Development data, not a test |
| `vitest.config.ts` | Includes only `tests/**/*.test.ts`; no coverage provider configured | Test infrastructure |

### Coverage assessment

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|-------------------|----------|--------------|
| `customer.schemas.ts` | 0 | 0 | Not measured; statically 0% of rules asserted | No tests for any constraint (lengths, UUID, partial, query coercion) |
| `buildCustomerRouter` | 0 | 0 | 0% of routes exercised over HTTP | Auth wiring and validation composition unverified |
| `CustomerController` | 0 | 0 | 0% | Status codes and response shapes unverified |
| `CustomerService` | 0 | 0 | 0% direct | Every business rule (BR-12 to BR-15) untested; no test double exists |
| `CustomerRepository` | 0 | 0 | 0% direct | Search `OR`, ordering, transaction count, CRUD queries untested |
| `Customer` data model / MySQL constraints | 0 | 8 order tests use rows indirectly | Not measured | Exercise insert and FK from orders; never unique/FK-RESTRICT/JSON edge cases |
| Cross-module order-customer behavior | 0 | 1 test asserts 404 for non-existent product; customer existence path not separately asserted | Low | The 404 in `tests/orders.test.ts:42-57` is for a missing product, not a missing customer |

**Metrics caveat:** no coverage provider (for example `@vitest/coverage-v8`) is installed and no `coverage` script exists in `package.json:10-21`. No numeric line/branch percentages can be produced without adding tooling, which is outside this analysis. The percentages above are static assessments, not measured reports. Tests were not executed by this analysis.

### Untested behaviors (explicit list)

1. Authentication failure and success on customer routes (401 paths).
2. UUID param rejection for get/update/delete.
3. All create validation branches: name bounds, e-mail format/length, phone bounds, document bounds, address required and each sub-field.
4. Duplicate e-mail on create returning 409 `EMAIL_ALREADY_USED`.
5. Duplicate e-mail on update and self-e-mail allowance.
6. E-mail uniqueness race returning `CONFLICT`.
7. Pagination defaults, coercion and bounds; invalid `page`/`pageSize`.
8. Search trimming, empty search rejection, OR matching across name/email/document, wildcard behavior, case-insensitive matching.
9. Ordering by `createdAt DESC` and pagination envelope math, including `total = 0`.
10. 404 paths for get/update/delete and update/delete race with `P2025`.
11. Successful hard delete and 204 response.
12. Delete blocked by existing order (`P2003`) and the current 500 mapping.
13. Unknown fields being stripped; attempts to set `id`/timestamps.
14. Shallow-partial address behavior and full address replacement.
15. Empty PATCH body behavior and `updatedAt` change.
16. Full PII exposure in responses (a security characteristic that should at least be documented by a test).

### Test strategy observations

- The existing suite is **integration-only** against a real MySQL instance, with serial execution (`fileParallelism: false`, single fork) and a global truncate in `tests/setup.ts`. There are no unit tests, mocks or in-memory repositories anywhere in the project.
- The customer factory is intentionally low-level and creates valid rows quickly for order tests. It is appropriate as a fixture but cannot validate the Customers Module because it bypasses every layer under test.
- `tests/setup.ts:14` deletes customers after dependent tables have been cleared, which correctly respects the FK. Any new customer test must follow the same ordering or use the factories.
- There is **no test database guard**: the harness truncates whatever `DATABASE_URL` points to (`tests/setup.ts:8-16`). This is a project-level risk that also affects any future customer tests.

---

## 12. Analysis Notes and Confidence

### Evidence basis

- All five module files were read in full (234 lines).
- Coupled infrastructure was read: `src/app.ts`, `src/routes/index.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/request-logger.middleware.ts`, `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/http/response.ts`, `src/config/env.ts`, `src/config/database.ts`.
- Cross-module usage was verified by searching all `src`, `tests` and `prisma` files for `modules/customers` and `customer.` references.
- Persistence contracts were read from `prisma/schema.prisma`, the initial migration and `prisma/seed.ts`.
- Tests and test infrastructure were read: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/helpers/factories.ts`, `tests/setup.ts`, `vitest.config.ts`.
- The architectural report at `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md` was used for system context and cross-checked against the code.

### Confidence by topic

| Topic | Confidence | Reason |
|-------|------------|--------|
| File structure, routes, controller/service/repository behavior | High | Direct source inspection |
| Validation constraints and defaults | High | Zod schemas read in full |
| E-mail uniqueness and existence rules | High | Explicit service branches |
| Delete blocked by FK and 500 mapping | High | Schema/migration and error middleware inspected |
| Case-insensitive e-mail identity | Medium-High | Depends on MySQL collation configured in migration and Docker; not executed |
| `LIKE` wildcard behavior for `%`/`_` | Medium | Prisma-to-SQL translation inferred; no escaping in source is certain |
| Exact test coverage percentages | Unavailable | No coverage provider configured; no test run performed |
| Intended product requirements | Low | PRD/RFC/FDD/TRACKER are empty placeholders |

### Explicit ambiguities

- Whether duplicate `document` values are intentionally allowed cannot be determined from documentation; the code clearly allows them.
- Whether customer endpoints were intended to be role-restricted is unknown; the code applies authentication only.
- Whether `PATCH` was intended to support deep address merging is unknown; the schema implements shallow partial only.
- Whether the 500 on customer deletion with orders is a known limitation or an oversight is unknown; nothing in the code or docs addresses it.
- Whether e-mail casing should be normalized is unknown; implementation delegates identity to the database collation.

---

*End of report.*
