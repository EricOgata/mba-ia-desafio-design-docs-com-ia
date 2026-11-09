# Dependency Audit Report

**Project:** `order-management-api` (Order Management System REST API)
**Repository:** `mba-ia-desafio-design-docs-com-ia`
**Audit date:** 2026-11-08 22:50:33
**Commit audited:** `e7f6311`
**Scope:** Entire project root, excluding `node_modules`, `.git`, `dist`, `coverage`
**Mode:** Read-only analysis. No project files were modified, no install was executed, and no upgrade or migration was performed.

---

## 1. Summary

This is a single-ecosystem Node.js/TypeScript application (ESM, `"type": "module"`) implementing an order management REST API. It uses Express 4 for HTTP, Prisma 5 against MySQL, Zod for validation, Pino for logging, JWT plus bcrypt for authentication, and Vitest/Supertest for tests. A separate infrastructure dependency exists: the `mysql:8.0` container image declared in `docker-compose.yml`.

**Manifests and lockfiles detected**

| File | Role | Status |
|---|---|---|
| `package.json` | npm manifest, 25 direct dependencies, all pinned to exact versions | Present |
| `package-lock.json` | npm lockfile, `lockfileVersion: 3`, 167 resolved production packages | Present |
| `docker-compose.yml` | MySQL 8.0 container image | Present |

No other package manager manifests were found (no `requirements.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`, `composer.json`, `pnpm-lock.yaml`, or `yarn.lock`). The audit therefore covers npm and the MySQL container image separately.

**Dependency counts**

- Direct npm dependencies: 25 (8 production, 17 development)
- Resolved production tree in lockfile: 167 packages (transitives scanned for risk context, not catalogued)
- Vulnerable direct packages: 3 (`uuid`, `zod`, `vitest`)
- Vulnerable transitive packages in the production resolution: 6 (`body-parser`, `qs`, `path-to-regexp`, `proxy-addr`, `tar`; `brace-expansion` is tooling-only)
- Deprecated packages or versions: 3 (`eslint@8.57.1`, `supertest@7.0.0`, `@types/uuid`)
- Declared but unused direct dependencies: 2 (`pino-http`, `@types/uuid`)

**Main findings**

1. Three direct dependencies carry published advisories: `uuid@11.0.3` (High), `zod@3.23.8` (High), and `vitest@2.1.4` (two Critical, one Medium).
2. The production tree contains six vulnerable transitive packages inherited through `express@4.21.1` and `bcrypt@5.1.1`. The most serious are `proxy-addr@2.0.7` (CVSS 9.1) and `body-parser@1.20.3` (CVSS 7.5), both reachable from `src/app.ts` depending on configuration.
3. The core runtime stack is one to three major versions behind: Prisma 5 vs 7, Express 4 vs 5, Zod 3 vs 4, TypeScript 5 vs 7, Vitest 2 vs 5.
4. `eslint@8.57.1` is end-of-life (2024-10-05), and its legacy `.eslintrc.json` configuration is incompatible with ESLint 10.
5. `@types/node@20.17.6` targets Node 20, which is end-of-life; the local runtime observed is Node 24.13.0 while `engines` still permits Node `>=20`.
6. `mysql:8.0` entered Oracle Sustaining Support on 2026-04-21 and is no longer in Premier Support; the image tag is not digest-pinned.
7. All npm licenses are permissive (MIT and Apache-2.0). No copyleft propagates into the application code. The MySQL container is GPLv2 but runs as a separate process.

**Limitations and assumptions**

- External validation was performed against the npm registry (dist-tags, deprecation metadata) and the OSV.dev vulnerability API (including advisory ranges and aliases to CVE identifiers).
- `node_modules` is not present in the repository. Installed-tree verification was not possible; versions were taken from `package.json` and `package-lock.json`.
- No container image scan was performed for `mysql:8.0`; see Section 5.
- Transitive versions are taken from `package-lock.json`. Direct dependencies are the only ones catalogued in Section 3, per audit scope.
- The lockfile is present, so production resolution is reproducible for the audited commit.

---

## 2. Critical Issues

### 2.1 Direct vulnerabilities

| Dependency | Installed | Advisory | CVE | CVSS 3.1 | Issue |
|---|---|---|---|---|---|
| `uuid` | 11.0.3 | GHSA-w5hq-g745-h8pq | CVE-2026-41907, CVE-2026-41988 | 7.5 High | `v3()`, `v5()`, `v6()` do not bounds-check caller-provided output buffers, allowing silent partial writes. Fixed in 11.1.1 (12.x fixed in 12.0.1, 13.x in 13.0.1). |
| `zod` | 3.23.8 | GHSA-mrhx-c7xc-m3w5 | CVE-2023-54404 | 7.5 High | Unbounded accumulation of validation issues for oversized arrays can exhaust memory. OSV lists all versions below 4.6.6 as affected; the advisory text describes the Zod 4 `$ZodArray` implementation, so applicability to the 3.x code path is not corroborated by the advisory details. |
| `vitest` | 2.1.4 | GHSA-9crc-q9x8-hgqq | CVE-2025-24964 | 9.6 Critical | Cross-site WebSocket hijacking against the Vitest API server when it is listening, leading to remote code execution. Fixed in 2.1.9, 3.0.5, 1.6.1. |
| `vitest` | 2.1.4 | GHSA-82fw-gwwq-j7x9 | CVE-2026-84373 | 5.9 Medium | Path traversal / arbitrary file read through `@vitest/mocker` redirect mocks when the dev server WebSocket is reachable. Fixed in 4.1.11 and 5.0.0-rc.2. |
| `vitest` | 2.1.4 | GHSA-5xrq-8626-4rwp | CVE-2026-47429 | 9.8 Critical | Arbitrary file read and execution when the Vitest UI server is exposed to the network, or when the UI/Browser Mode runs on Windows. Fixed in 3.2.6 and 4.1.0. |

Project-specific exposure:

- `uuid` is used only through `v4()` in `src/middlewares/request-logger.middleware.ts` without a caller-provided buffer. The vulnerable `v3`/`v5`/`v6` buffer APIs are not used, so the runtime exposure of the installed version is low despite the High advisory.
- `zod` is used on every validated request. `createOrderSchema` declares `items: z.array(orderItemInputSchema).min(1, ...)` with no maximum length (`src/modules/orders/order.schemas.ts:13`), and the global JSON body limit is `1mb` (`src/app.ts:59`). An oversized array of failing items can therefore produce an unbounded number of issue objects.
- `vitest` is a dev-only dependency. The Critical advisory requires the Vitest API/UI server to be listening; `vitest run` does not enable it. Exposure is confined to developer machines and any environment where the UI/API server is exposed.

### 2.2 Transitive vulnerabilities in the production resolution

These are not direct dependencies and are excluded from the Section 3 catalogue, but they are present in the resolved tree and reachable from application code.

| Package | Installed | Advisory | CVE | CVSS 3.1 | Reachability |
|---|---|---|---|---|---|
| `proxy-addr` | 2.0.7 | GHSA-jqcg-44mw-7w3h | CVE-2026-90711 | 9.1 Critical | Via `express`. IP spoofing through IPv4-mapped IPv6 trust subnets. Only effective when Express `trust proxy` is enabled; `src/app.ts` does not set it. Fixed in 2.0.8. |
| `body-parser` | 1.20.3 | GHSA-27r3-jxmh-p828 | CVE-2026-111181 | 7.5 High | Via `express.json()` in `src/app.ts:59`. Trailing bytes after a compressed request body cause denial of service. Compression inflation is enabled by default in body-parser. Fixed in 1.20.9. |
| `path-to-regexp` | 0.1.10 | GHSA-37ch-88jc-xwx2 | CVE-2024-45296, CVE-2026-4867 | 7.5 High | Via the Express router. ReDoS with multiple route parameters. Fixed in 0.1.13. |
| `path-to-regexp` | 0.1.10 | GHSA-rhx6-c78j-4q9w | CVE-2024-52798 | 7.5 High | Via the Express router. ReDoS. Fixed in 0.1.12. |
| `tar` | 6.2.1 | GHSA-34x7-hfp2-rc4v and 11 others | Multiple | 8.2 High | Via `bcrypt@5.1.1` -> `@mapbox/node-pre-gyp@1.0.11` -> `tar@6.2.1`. Install-time only, not runtime request handling. |
| `qs` | 6.13.0 | GHSA-4mjr-xmp4-gh2g and 4 others | CVE-2026-82417, CVE-2026-110816, CVE-2025-15284, CVE-2026-8723, CVE-2026-2391 | 5.3 Medium | Via Express query-string parsing (`extended` parser). Denial of service through array-limit and memory-exhaustion paths. Fixed in 6.14.1 through 6.16.1 depending on advisory. |
| `brace-expansion` | 1.1.14 | 6 advisories | Multiple | 7.5 High | Dev tooling only, through `minimatch` in the ESLint dependency chain. Not reachable at runtime. |

### 2.3 Deprecated and end-of-life core dependencies

- **`eslint@8.57.1`** — npm marks it "no longer supported"; ESLint 8 reached end-of-life on 2024-10-05. The repository uses the legacy `.eslintrc.json` format, which ESLint 10 no longer supports.
- **`@types/uuid@10.0.0`** — deprecated stub; `uuid` ships its own type definitions. The package is also unused because no source file imports types from it.
- **`supertest@7.0.0`** — npm deprecation notice: "Please upgrade to supertest v7.1.3+".
- **`express@4.21.1`** — Express 4 entered maintenance mode on 2025-04-01, with a targeted end no sooner than 2026-10-01 (a date now passed). The 4.x line remains on maintenance while 5.x is the current major.
- **`@prisma/client@5.22.0` / `prisma@5.22.0`** — two majors behind Prisma 7.10.1. Prisma has not published a support window for the 5.x line; Prisma 6 security patches end 2026-11-19.
- **`uuid@11.0.3`** — three majors behind 14.0.3, and affected by a High advisory fixed in 11.1.1.
- **`zod@3.23.8`** — one major behind 4.6.7. Zod 4 deprecates `z.nativeEnum()` and `z.string().uuid()`.
- **`typescript@5.6.3`** — two majors behind 7.0.2. The current `@typescript-eslint@8.72.1` peer range requires `typescript <6.1.0`, so the lint stack does not yet support TypeScript 7.

### 2.4 Infrastructure dependency

- **`mysql:8.0`** — MySQL 8.0 moved to Oracle Sustaining Support on 2026-04-21. The image tag is `8.0`, not a digest, so redeployments may resolve to different patch images. `docker-compose.yml:22` also passes `--default-authentication-plugin=mysql_native_password`, a flag removed in MySQL 8.4 and later.

### 2.5 Licence findings

- All 25 npm dependencies are permissive: MIT or Apache-2.0 (`@prisma/client`, `prisma`, `typescript`). No copyleft or network-copyleft licence appears in the direct or resolved npm tree.
- The MySQL container image is GPLv2. Because it runs as a separate process and is not linked into the application, it does not impose copyleft on the application source. Distribution of the image itself carries the GPL obligations of the image.
- No licence conflict or legal-risk item was identified in the npm dependency set.

---

## 3. Dependencies

All versions below are the exact pins from `package.json`. Latest versions were resolved from the npm registry on 2026-11-08. Status reflects the gap to the latest stable release, deprecation metadata, and whether the package is used.

### Production dependencies

| Dependency | Current Version | Latest Version | Status |
|---|---|---|---|
| `@prisma/client` | 5.22.0 | 7.10.1 | Legacy (2 majors) |
| `bcrypt` | 5.1.1 | 6.0.0 | Outdated (1 major) |
| `express` | 4.21.1 | 5.3.0 (4.x latest: 4.22.3) | Legacy (1 major; 4.x in maintenance) |
| `jsonwebtoken` | 9.0.2 | 9.0.3 | Outdated (patch) |
| `pino` | 9.5.0 | 10.4.0 | Outdated (1 major) |
| `pino-http` | 10.3.0 | 11.0.0 | Outdated (1 major); Unused |
| `uuid` | 11.0.3 | 14.0.3 | Legacy (3 majors) |
| `zod` | 3.23.8 | 4.6.7 | Legacy (1 major) |

### Development dependencies

| Dependency | Current Version | Latest Version | Status |
|---|---|---|---|
| `@types/bcrypt` | 5.0.2 | 6.0.0 | Outdated (1 major) |
| `@types/express` | 4.17.21 | 5.0.6 | Outdated (1 major; tracks Express 4) |
| `@types/jsonwebtoken` | 9.0.7 | 9.0.10 | Outdated (patch) |
| `@types/node` | 20.17.6 | 26.6.6 | Legacy (6 majors; Node 20 EOL) |
| `@types/supertest` | 6.0.2 | 7.2.1 | Outdated (1 major) |
| `@types/uuid` | 10.0.0 | 11.0.0 | Deprecated stub; Unused |
| `@typescript-eslint/eslint-plugin` | 8.13.0 | 8.72.1 | Outdated (minor) |
| `@typescript-eslint/parser` | 8.13.0 | 8.72.1 | Outdated (minor) |
| `eslint` | 8.57.1 | 10.14.0 | Deprecated; Legacy (2 majors; EOL 2024-10-05) |
| `eslint-config-prettier` | 9.1.0 | 10.1.8 | Outdated (1 major) |
| `pino-pretty` | 11.3.0 | 13.2.0 | Outdated (2 majors) |
| `prettier` | 3.3.3 | 3.9.10 | Outdated (minor) |
| `prisma` | 5.22.0 | 7.10.1 | Legacy (2 majors) |
| `supertest` | 7.0.0 | 7.3.2 | Deprecated version; Outdated (patch) |
| `tsx` | 4.19.2 | 4.23.15 | Outdated (minor) |
| `typescript` | 5.6.3 | 7.0.2 | Legacy (2 majors) |
| `vitest` | 2.1.4 | 5.1.0 | Legacy (3 majors) |

**Version notes**

- The `prisma` CLI package publishes `8.0.0-rc.33` on the `latest` dist-tag as a pre-release. The latest stable release is `7.10.1`, which matches `@prisma/client@7.10.1`. The table reports the stable release.
- `express@4.22.3` is the latest 4.x release and declares ranges that admit the patched transitive releases (`body-parser ~1.20.5`, `qs ~6.16.0`, `path-to-regexp ~0.1.13`, `proxy-addr ~2.0.7`). The table reports the latest stable major, `5.3.0`.
- `supertest@7.0.0` is explicitly deprecated by its maintainers in favour of 7.1.3 and later.
- `@types/express@5.0.6` corresponds to Express 5; the installed 4.17.21 matches the installed Express 4 line.

---

## 4. Risk Analysis

Severity is assigned from the advisory CVSS where available, adjusted for actual reachability in this repository where that changes exposure. The table lists each distinct risk item; repeated advisories against the same package are grouped.

| Severity | Dependency | Issue | Details |
|---|---|---|---|
| Critical | `vitest` 2.1.4 | CVE-2025-24964 / GHSA-9crc-q9x8-hgqq | Cross-site WebSocket hijacking against the Vitest API server leads to remote code execution (CVSS 9.6). Dev-only; requires the API/UI server to be listening. |
| Critical | `vitest` 2.1.4 | CVE-2026-47429 / GHSA-5xrq-8626-4rwp | Arbitrary file read and execution when the Vitest UI is exposed to the network, or when UI/Browser Mode runs on Windows (CVSS 9.8). Dev-only. |
| Critical | `proxy-addr` 2.0.7 (transitive) | CVE-2026-90711 / GHSA-jqcg-44mw-7w3h | IP spoofing via IPv4-mapped IPv6 trust subnet (CVSS 9.1). Only effective if Express `trust proxy` is enabled; not enabled in `src/app.ts`. |
| High | `body-parser` 1.20.3 (transitive) | CVE-2026-111181 / GHSA-27r3-jxmh-p828 | Denial of service via trailing bytes after a compressed request body (CVSS 7.5). Reachable through `express.json({ limit: '1mb' })`; inflation is enabled by default. |
| High | `path-to-regexp` 0.1.10 (transitive) | CVE-2024-45296, CVE-2026-4867, CVE-2024-52798 | ReDoS through the Express router (CVSS 7.5 for the primary advisories). Reachable on the route-matching path. |
| High | `uuid` 11.0.3 | CVE-2026-41907 / CVE-2026-41988 | Missing buffer bounds check in `v3`/`v5`/`v6` when a caller-provided buffer is used (CVSS 7.5). The application uses only `v4()` without a buffer, so project exposure is low. |
| High | `zod` 3.23.8 | CVE-2023-54404 / GHSA-mrhx-c7xc-m3w5 | Memory exhaustion from unbounded validation-issue accumulation on oversized arrays (CVSS 7.5). `createOrderSchema` has no maximum item count, and the request body limit is 1 MB. |
| High | `tar` 6.2.1 (transitive) | Multiple advisories, including GHSA-34x7-hfp2-rc4v | Arbitrary file creation/overwrite and hardlink traversal during archive extraction (up to CVSS 8.2). Install-time only, through bcrypt's `node-pre-gyp`. |
| High | `express` 4.21.1 | End-of-maintenance major | Express 4 is in maintenance mode with a targeted end no sooner than 2026-10-01, now passed. Its pinned transitives carry the body-parser, qs, and path-to-regexp advisories above. |
| High | `@prisma/client` 5.22.0 / `prisma` 5.22.0 | Unsupported major line | Two majors behind 7.10.1. No support window published for 5.x; Prisma 6 security patches end 2026-11-19. All persistence depends on this client. |
| High | `mysql:8.0` (infrastructure) | Sustaining Support | MySQL 8.0 entered Oracle Sustaining Support on 2026-04-21. The image tag is floating within 8.0.x and is not digest-pinned. |
| Medium | `vitest` 2.1.4 | CVE-2026-84373 / GHSA-82fw-gwwq-j7x9 | Path traversal and arbitrary file read through `@vitest/mocker` when the dev server WebSocket is reachable (CVSS 5.9). Dev-only. |
| Medium | `qs` 6.13.0 (transitive) | Five advisories, including CVE-2025-15284 | Denial of service through array-limit bypass and memory exhaustion in query parsing (up to CVSS 5.3). Reachable through Express query parsing. |
| Medium | `bcrypt` 5.1.1 | Outdated major plus vulnerable install chain | One major behind 6.0.0. The native build path pulls `@mapbox/node-pre-gyp@1.0.11` and `tar@6.2.1`. |
| Medium | `eslint` 8.57.1 | End-of-life and deprecated | EOL since 2024-10-05; npm marks the release unsupported. The legacy `.eslintrc.json` cannot be used with ESLint 10. |
| Medium | `typescript` 5.6.3 | Toolchain incompatibility | Two majors behind 7.0.2. TypeScript 7 removes `baseUrl`, which `tsconfig.json` uses together with `paths`. Current `@typescript-eslint@8.72.1` requires `typescript <6.1.0`. |
| Medium | `@types/node` 20.17.6 | Mismatch with runtime and EOL Node line | Types target Node 20, which is end-of-life; the observed local runtime is Node 24.13.0. `engines` still allows Node `>=20`. |
| Medium | `@typescript-eslint/eslint-plugin` 8.13.0 / `@typescript-eslint/parser` 8.13.0 | Peer-range constraint | Latest 8.72.1 still declares a peer range excluding TypeScript 7, coupling the lint and compiler upgrades. |
| Low | `brace-expansion` 1.1.14 (transitive) | Six DoS advisories | Dev-tooling only through the ESLint dependency chain; not reachable at runtime. |
| Low | `pino-http` 10.3.0 | Unused direct dependency | Declared in `package.json` but no source file imports it or references `pinoHttp`. Removes an entire maintenance surface if dropped. |
| Low | `@types/uuid` 10.0.0 | Deprecated stub, unused | `uuid` ships its own types; no source file imports types from this package. |
| Low | `supertest` 7.0.0 | Deprecated version | Maintainers direct users to 7.1.3 or later. Dev-only test client. |
| Low | `jsonwebtoken` 9.0.2 | Patch behind | 9.0.3 is a patch release; no advisory affects 9.0.2. |
| Low | `pino` 9.5.0 / `pino-pretty` 11.3.0 | Major/minor drift | One major behind Pino 10; `pino-pretty` is two majors behind. No advisory affects the installed versions. |
| Low | `prettier` 3.3.3 / `tsx` 4.19.2 | Minor drift | Formatting and TS execution tooling; no advisory affects the installed versions. |

**Severity rationale**

- `vitest` advisories are rated by CVSS as Critical but are dev-only. They affect developer environments and any exposed test UI/API, not the production request path.
- `proxy-addr` is Critical by CVSS but is not reachable in the current configuration because `trust proxy` is not set. It becomes reachable if proxy trust is enabled without restricting the trust subnet.
- `uuid` is High by CVSS but the vulnerable APIs are not used. The package-level risk remains because future use of `v3`/`v5`/`v6` with buffers would silently write out of bounds.
- `zod` is High and reachable: the order creation schema has no upper bound on the `items` array.

---

## 5. Unverified Dependencies

| Dependency | Current Version | Reason Not Verified |
|---|---|---|
| `mysql:8.0` (Docker image) | `8.0` tag | The registry tag was not resolved to an image digest, and no image-level CVE scan was performed. Version, support status, and the removed `mysql_native_password` flag were verified from Oracle's lifecycle documentation; package-level vulnerability coverage for the image is unknown. |

All 25 npm direct dependencies were verified against the npm registry and OSV.dev. No npm dependency is listed as unverified.

---

## 6. Critical File Analysis

The files below were selected because they concentrate the highest-risk dependencies or because a dependency incompatibility would break a core business path. Paths are relative to the repository root.

### 1. `src/app.ts`

- **Role:** Application composition root. Creates the Express app, mounts `express.json()`, the request logger, `/health`, the `/api/v1` router, the 404 handler, and the error middleware; wires every repository, service, and controller.
- **Risky dependencies:** `express@4.21.1`; transitive `body-parser@1.20.3`, `qs@6.13.0`, `path-to-regexp@0.1.10`, `proxy-addr@2.0.7`; `@prisma/client@5.22.0`.
- **Why critical:** Every HTTP request passes through this file. A compromise or outage here affects the entire API. The `express.json({ limit: '1mb' })` call is the reachable path for CVE-2026-111181. The absence of `app.set('trust proxy', ...)` is what currently keeps the Critical `proxy-addr` advisory unreachable.
- **Coupling:** Express 5 changes `req.query` to a non-writable getter, which conflicts with the validation middleware described below. The Express 4.21.1 pin also locks the transitive versions of `body-parser`, `qs`, and `path-to-regexp` to the vulnerable releases listed above.

### 2. `src/middlewares/validate.middleware.ts`

- **Role:** Generic request validation middleware for `body`, `query`, and `params` using Zod schemas.
- **Risky dependencies:** `zod@3.23.8`; `express@4.21.1` (types and request semantics).
- **Why critical:** It guards every validated route, including authentication and order creation. Line 19 uses `Object.assign(req.query, parsedQuery)`. Under Express 5, `req.query` is a non-writable getter that re-parses the query string on each access, so this assignment no longer replaces the parsed values. The middleware would appear to succeed while downstream handlers read the unvalidated original query. This is a correctness and input-validation defect at the HTTP boundary.
- **Coupling:** Tightly coupled to Express request mutability and to Zod's `ZodTypeAny` and `ZodError.issues` API.

### 3. `src/modules/orders/order.schemas.ts`

- **Role:** Zod schemas for order creation, status transitions, and order listing.
- **Risky dependencies:** `zod@3.23.8`; `@prisma/client@5.22.0` (`OrderStatus` enum import).
- **Why critical:** Defines the accepted shape of the highest-value business operation. Line 13 declares `items: z.array(orderItemInputSchema).min(1, ...)` without a maximum, which is the reachable surface for CVE-2023-54404. Lines 19 and 26 use `z.nativeEnum(OrderStatus)`, and lines 4, 7, 12, and 27 use `z.string().uuid()`; both patterns are deprecated in Zod 4. The module mixes validation and ORM enums, so a Prisma or Zod major change propagates into the API contract.
- **Coupling:** Direct import of a generated Prisma enum into runtime validation code. Any Prisma generator or client change can break schema construction.

### 4. `src/modules/orders/order.service.ts`

- **Role:** Core order domain logic: transactional order creation, stock checks and debits, status transitions, order-number reservation, and deletion rules.
- **Risky dependencies:** `@prisma/client@5.22.0` (`PrismaClient`, `Prisma.TransactionClient`, `OrderStatus`, `Prisma` types).
- **Why critical:** This is the highest-concentration business logic in the repository. It runs interactive transactions, increments and decrements stock, and creates order status history. A Prisma major change affects `$transaction` typing, enum values, nested writes, and `upsert` behavior. A failure here causes incorrect stock levels or inconsistent orders, not merely a failed request.
- **Coupling:** Depends on the generated Prisma client shape and on `OrderStatus` values shared with the database schema and the Zod schemas.

### 5. `src/config/env.ts`

- **Role:** Parses and validates environment variables at startup with Zod; calls `process.exit(1)` on invalid configuration.
- **Risky dependencies:** `zod@3.23.8`.
- **Why critical:** Every process starts here. The module is imported by `src/server.ts`, `src/config/database.ts`, `src/shared/logger/index.ts`, `src/modules/auth/auth.service.ts`, and `src/middlewares/auth.middleware.ts`. A Zod major change affects `z.coerce`, `z.enum`, `safeParse`, and `error.issues`. Because the module exits the process on failure, a schema incompatibility is a total startup failure.
- **Coupling:** Foundational singleton imported across the application; it is a single point of failure for configuration.

### 6. `src/modules/auth/auth.service.ts`

- **Role:** User login, password verification with bcrypt, and JWT access-token signing.
- **Risky dependencies:** `bcrypt@5.1.1`; `jsonwebtoken@9.0.2`.
- **Why critical:** Security-critical authentication path. `bcrypt.compare()` is the only password verification in the codebase, and `jwt.sign()` is the only token-issuing path. bcrypt is a native module built through `node-pre-gyp`, which is the install-time vector for the `tar@6.2.1` advisories. The `SignOptions['expiresIn']` cast on line 48 depends on the `jsonwebtoken` type surface.
- **Coupling:** Both dependencies are security boundaries; a breaking change or build failure disables login for all users.

### 7. `src/middlewares/auth.middleware.ts`

- **Role:** Bearer-token extraction, JWT verification, request-user attachment, and role enforcement.
- **Risky dependencies:** `jsonwebtoken@9.0.2`; `express@4.21.1` (module augmentation of `express-serve-static-core`).
- **Why critical:** Every protected route depends on it. `jwt.verify()` is the only token-validation path, and `requireRole()` is the only authorization gate. The module augments Express's `Request` type, so an Express 5 type change can break compilation across the API. A verification defect is an authentication bypass.
- **Coupling:** Depends on `env.JWT_SECRET`, Express request typing, and the JWT payload contract used by `auth.service.ts`.

### 8. `src/middlewares/request-logger.middleware.ts`

- **Role:** Assigns or propagates `X-Request-Id`, times each request, and emits a structured Pino log line on response finish.
- **Risky dependencies:** `uuid@11.0.3`; `pino@9.5.0`.
- **Why critical:** It is the only consumer of the vulnerable `uuid` package. The code calls `uuidv4()` without a buffer, so the CVE-2026-41907 buffer APIs are not used, but any future refactor to `v3`/`v5`/`v6` would activate the High advisory. It also supplies the correlation identifier used across all request logs, which is essential for incident tracing.
- **Coupling:** Relies on the Express request/response lifecycle and on the Pino logger singleton.

### 9. `src/shared/logger/index.ts`

- **Role:** Creates the Pino logger with redaction paths for authorization headers, cookies, passwords, password hashes, and tokens; configures the `pino-pretty` transport in development.
- **Risky dependencies:** `pino@9.5.0`; `pino-pretty@11.3.0`.
- **Why critical:** It is the observability and secret-redaction boundary. A misconfigured transport prevents the process from starting or silently drops redaction, leaking credentials into logs. The development branch loads `pino-pretty` as a transport target, so a major upgrade of either package must remain compatible.
- **Coupling:** Imported by the server bootstrap, error middleware, and request logger. `env.NODE_ENV` decides whether the transport is instantiated.

### 10. `prisma/schema.prisma`

- **Role:** Prisma data model: MySQL datasource, `prisma-client-js` generator, two enums and seven models, indexes, relations, and the `OrderNumberSequence` table.
- **Risky dependencies:** `prisma@5.22.0`; `@prisma/client@5.22.0`.
- **Why critical:** It is the single source of truth for the database schema and for the generated client types used throughout the application. The `OrderStatus` enum defined here flows into `order.schemas.ts`, `order.service.ts`, and `order.repository.ts`. The generator `prisma-client-js` is deprecated in Prisma 7 in favour of the `prisma-client` generator, which changes the generated client location and import path; Prisma 7 also requires a driver adapter or Accelerate URL when instantiating `PrismaClient`. The datasource uses `shadowDatabaseUrl`, which affects migration environments. A generator or datasource change can invalidate the generated client across the whole codebase.
- **Coupling:** All persistence code depends on the generated output; migrations and the `OrderNumberSequence` upsert are business-critical.

---

## 7. Integration Notes

### Production dependency usage

| Dependency | Where used | Integration summary | Single point of failure |
|---|---|---|---|
| `express` | `src/app.ts`, `src/middlewares/*`, all routes/controllers | HTTP framework, routing, JSON body parsing, query parsing, middleware chain. Express 4 request mutability is assumed by `validate.middleware.ts`. | Yes — all HTTP traffic |
| `@prisma/client` | `src/config/database.ts`, all repositories, `order.service.ts`, `order.schemas.ts` | ORM and generated types for every table; enums are imported into runtime validation. | Yes — all persistence |
| `prisma` | `package.json` scripts, `prisma/schema.prisma`, `prisma/seed.ts` | CLI for migrations, client generation, reset, and seeding. | Yes — schema and migration lifecycle |
| `zod` | `src/config/env.ts`, `src/middlewares/validate.middleware.ts`, all `*.schemas.ts` | Runtime validation at startup and at HTTP boundaries; `ZodError` is translated into `ValidationError`. | Yes — configuration startup and request validation |
| `bcrypt` | `src/modules/auth/auth.service.ts`, `src/modules/users/user.service.ts` | Password hashing and comparison; native module built through `node-pre-gyp`. | Yes — authentication |
| `jsonwebtoken` | `src/modules/auth/auth.service.ts`, `src/middlewares/auth.middleware.ts` | JWT signing and verification; token expiry driven by `env.JWT_EXPIRES_IN`. | Yes — authentication and authorization |
| `pino` | `src/shared/logger/index.ts`, `src/server.ts`, middleware | Structured JSON logging with redaction. | Yes — observability and secret redaction |
| `pino-http` | Not imported anywhere | Declared but unused. No request logging is delegated to it; custom `request-logger.middleware.ts` is used instead. | No |
| `uuid` | `src/middlewares/request-logger.middleware.ts` | Generates request IDs with `v4()`; propagates an incoming `X-Request-Id` when present. | No — request IDs degrade gracefully |

### Development dependency usage

| Dependency | Integration summary |
|---|---|
| `typescript`, `tsx` | `tsc` builds via `tsconfig.build.json`; `tsx` runs `src/server.ts` in watch mode and executes `prisma/seed.ts`. |
| `vitest`, `supertest`, `@types/supertest` | Test runner and HTTP assertion client for `tests/auth.test.ts` and `tests/orders.test.ts`. |
| `eslint`, `@typescript-eslint/*`, `eslint-config-prettier` | Linting through the legacy `.eslintrc.json`; syntax-level TypeScript rules. `eslint-config-prettier` disables formatting rules that conflict with Prettier. |
| `prettier` | Formatting; configuration in `.prettierrc`. |
| `pino-pretty` | Development-only Pino transport target, loaded by name from `src/shared/logger/index.ts`. |
| `@types/bcrypt`, `@types/jsonwebtoken`, `@types/express`, `@types/node` | Ambient and module type declarations for the runtime packages. `@types/express` tracks Express 4; `@types/node` tracks Node 20. |
| `@types/uuid` | Deprecated stub and unused; `uuid` provides its own declarations. |
| `prisma` | Development CLI, also used by `prisma generate` during install/build workflows. |

### Cross-cutting integration risks

- **Toolchain coupling:** `@typescript-eslint@8.72.1` declares a peer range of `typescript <6.1.0`. TypeScript 7.0.2 is outside that range, so the compiler and lint stack cannot both move to their latest majors without a coordinated change.
- **Express 5 semantic change:** `Object.assign(req.query, parsedQuery)` in `validate.middleware.ts:19` depends on Express 4's writable `req.query`. Express 5 exposes `req.query` as a getter that re-parses on each access.
- **ESLint 10 configuration change:** `.eslintrc.json` uses the legacy configuration system. ESLint 10 no longer supports it, so the lint configuration format is a hard compatibility boundary.
- **TypeScript 7 configuration change:** `tsconfig.json` uses `baseUrl` and `paths`. TypeScript 7 removes `baseUrl`, so path aliases depend on the compiler major.
- **Prisma generator change:** `prisma/schema.prisma` uses `prisma-client-js`. Prisma 7's `prisma-client` generator changes the generated client and its import path, which propagates to every `@prisma/client` import.
- **Zod 4 API changes:** `z.nativeEnum()` and `z.string().uuid()` are deprecated in Zod 4; both appear in `order.schemas.ts` and other schema modules.
- **No automated dependency updates:** No Dependabot, Renovate, or CI workflow is configured. All 25 direct dependencies are exact-pinned, so updates are manual and drift is invisible until an audit or a failure occurs.
- **No installed-tree verification:** `node_modules` is absent. The lockfile pins the resolution, but the actual installed tree could differ if an install is performed with a different npm version or if the lockfile is bypassed.

### Maintenance burden

- 25 exact-pinned direct dependencies across 8 production and 17 development packages, with no update automation.
- The toolchain is internally coupled: Express types to Express 4, `@types/node` to Node 20, `@typescript-eslint` to TypeScript below 6.1, ESLint 8 to the legacy config format, and Prisma 5 to the `prisma-client-js` generator. Updating any one of these in isolation is constrained by the others.
- Two declared dependencies (`pino-http`, `@types/uuid`) add maintenance surface without contributing code.
- The production tree carries six transitive advisory sources that are controlled by the direct `express` and `bcrypt` pins.

---

## 8. Save Confirmation

Report saved to:

`docs/agents/dependency-auditor/dependencies-report-2026-11-08_22-50-33.md`

Absolute path:

`/home/eric-ogata/codes/full_cycle/mba_engenharia_ia/desafios/desafio4/mba-ia-desafio-design-docs-com-ia/docs/agents/dependency-auditor/dependencies-report-2026-11-08_22-50-33.md`

No project files were modified. The only file created by this audit is the report above.
