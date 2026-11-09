# Component Deep Analysis Report - MySQL Container

## Document Metadata

| Item | Value |
|------|-------|
| Component | MySQL Container (`mysql` service) |
| Primary files | `docker-compose.yml` (lines 1-28), `.env.example` (lines 5-6, 11-14) |
| Related contracts | `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql`, `package.json`, `src/config/env.ts`, `src/config/database.ts`, `tests/setup.ts`, `vitest.config.ts` |
| Project root | `/home/eric-ogata/codes/full_cycle/mba_engenharia_ia/desafios/desafio4/mba-ia-desafio-design-docs-com-ia` |
| Analysis date | 2026-11-09 09:26:44 |
| Excluded folders | `node_modules`, `.git`, `dist`, `coverage` |
| Analysis mode | Static read-only inspection. No code, container, schema, or test was executed or modified. |
| Component type | Local infrastructure definition (Docker Compose service), not application source code |

---

## 1. Executive Summary

The "MySQL Container" component is a single Docker Compose service named `mysql`, declared in `docker-compose.yml:1-28`. It runs the official `mysql:8.0` image as container `oms-mysql`, publishes TCP port `3306` to the host, persists `/var/lib/mysql` in the named volume `oms_mysql_data`, exposes a `mysqladmin ping` healthcheck, and starts `mysqld` with three explicit flags: `mysql_native_password` as the default authentication plugin, `utf8mb4` as the server character set, and `utf8mb4_unicode_ci` as the server collation. Four environment variables with Compose fallback defaults provision the root password, database name, application user, and application password.

The component is the only stateful external dependency of the Order Management System. It is the physical persistence layer for every Prisma model in `prisma/schema.prisma`: `users`, `customers`, `products`, `orders`, `order_items`, `order_status_history`, and `order_number_sequence`. The application process, the seed script, the test harness, and the Prisma migration tooling all consume it indirectly through connection strings rather than through direct code imports. The Compose file does not define an application container, a network, an initialization SQL directory, a backup job, or a production deployment artifact.

Key findings:

- The component is sufficient for local development and for the current integration test suite, but it is not hardened for shared or production use.
- The image tag `mysql:8.0` is mutable and not digest-pinned. The dependency audit already records that MySQL 8.0 entered Oracle Sustaining Support on 2026-04-21.
- The default credentials are committed in `.env.example` and duplicated as Compose fallbacks: root password `root_password`, database `oms`, user `oms_user`, password `oms_password` (`docker-compose.yml:7-10`, `.env.example:11-14`).
- Port `3306` is published on all host interfaces (`docker-compose.yml:12`), and there is no TLS configuration for client connections (`DATABASE_URL` and `SHADOW_DATABASE_URL` in `.env.example:5-6` contain no TLS parameters).
- `docker compose up` alone creates an empty database. It does not create the seven application tables. Schema creation is out-of-band through `npm run db:migrate` (`package.json:14`), and there is no `prisma migrate deploy` script for non-interactive environments.
- The `SHADOW_DATABASE_URL` points to `oms_shadow` under root credentials (`SHADOW_DATABASE_URL`, `.env.example:6`). The Compose file does not create that database, and `src/config/env.ts` does not validate the variable.
- The healthcheck is a liveness probe, not a readiness or credential probe. `mysqladmin ping` reports server liveness; access-denied responses are not a reliable failure signal for that command.
- The named volume is the only copy of the data. There is no replication, failover, backup, point-in-time recovery, volume snapshot, or size limit.
- The test suite has 15 integration tests that depend on a live MySQL instance at whatever `DATABASE_URL` points to. `tests/setup.ts:8-16` deletes every application table before each test, with no environment guard.
- No test, script, or CI job validates the Compose file, the healthcheck, the volume lifecycle, the credentials contract, or a fresh-volume migration path.

Role in the system: the component is a local, single-instance MySQL 8.0 database for development, migration, seeding, and integration testing. It is not a managed database service, and it has no production topology definition. Section 7 (Endpoints) is intentionally omitted because this component exposes no REST, GraphQL, or gRPC API surface. Its only listener is the MySQL wire-protocol port, which is documented under Integration Points.

---

## 2. Data Flow Analysis

### 2.1 First-boot provisioning flow

1. An operator optionally creates a `.env` file in the project root. Docker Compose automatically reads `.env` for variable interpolation, but the file is not committed because `.gitignore:4` excludes it. The repository ships only `.env.example`.
2. `docker compose up -d` reads `docker-compose.yml` and resolves every `${VAR:-default}` expression from the shell environment or the Compose `.env` file. If neither provides a value, the fallback after `:-` is used (`docker-compose.yml:7-10`).
3. Docker resolves `mysql:8.0` (`docker-compose.yml:3`) against the local image cache or Docker Hub. The tag is not pinned to a digest, so the resolved patch version depends on pull time and platform.
4. Docker creates or reuses the default Compose bridge network for the project and the named volume `oms_mysql_data` (`docker-compose.yml:26-28`). The actual Docker volume name is project-prefixed unless an explicit `name` is set.
5. Docker starts the container as `oms-mysql` (`docker-compose.yml:4`) with the restart policy `unless-stopped` (`docker-compose.yml:5`).
6. If `/var/lib/mysql` is empty, the official MySQL entrypoint performs first-time initialization: it sets the root password from `MYSQL_ROOT_PASSWORD`, creates the database from `MYSQL_DATABASE`, creates the user from `MYSQL_USER` with `MYSQL_PASSWORD`, and grants that user privileges on the created database. These behaviors belong to the official image entrypoint, not to repository code; confidence is medium because the entrypoint is external to the repository.
7. `mysqld` starts with `--default-authentication-plugin=mysql_native_password`, `--character-set-server=utf8mb4`, and `--collation-server=utf8mb4_unicode_ci` (`docker-compose.yml:21-24`).
8. The healthcheck runs `mysqladmin ping -h localhost -uroot -p<interpolated root password>` every 5 seconds, with a 5-second timeout, 20 retries, and a 30-second start period (`docker-compose.yml:15-20`). Docker marks the container `healthy` when the probe exits successfully.
9. If the data directory is not empty, initialization is skipped. Existing databases, users, passwords, and grants are retained even if the environment variables have changed. This is the first-initialization-only behavior of the official image.

### 2.2 Schema and seed flow

1. An operator runs `npm run db:migrate` (`package.json:14`), which invokes `prisma migrate dev`.
2. Prisma CLI reads `DATABASE_URL` and `SHADOW_DATABASE_URL` from the process environment or the `.env` file. The application environment schema in `src/config/env.ts:3-10` validates `DATABASE_URL` but does not validate `SHADOW_DATABASE_URL`.
3. Prisma connects over TCP to `localhost:3306` using the credentials embedded in `DATABASE_URL` (`mysql://oms_user:oms_password@localhost:3306/oms`, `.env.example:5`).
4. For `migrate dev`, Prisma uses the root credentials in `SHADOW_DATABASE_URL` to create, use, and drop a shadow database named `oms_shadow` on the same server. The Compose file does not create `oms_shadow`; the root credentials are expected to have the global privileges needed to create it.
5. Prisma applies the migration in `prisma/migrations/20260519182739_init/migration.sql`, creating seven tables, two enum types, indexes, and foreign keys. Prisma records applied migrations in its own `_prisma_migrations` table.
6. An operator optionally runs `npm run db:seed` (`package.json:16`). The seed script uses `DATABASE_URL` through a standalone `PrismaClient` (`prisma/seed.ts:1-4`) and deletes existing rows before inserting the demo data (`prisma/seed.ts:181-187`).

### 2.3 Application runtime flow

1. The host process starts with `npm run dev` or `npm start` (`package.json:11-13`). Node loads `.env` through `--env-file=.env`.
2. `src/config/env.ts:3-27` validates `DATABASE_URL`, `JWT_SECRET`, and four other variables. A missing `DATABASE_URL` terminates the process with exit code 1 before the HTTP server starts.
3. `src/config/database.ts:4-10` constructs a single `PrismaClient` with logging configured by `NODE_ENV`. Prisma reads the datasource URL from the environment at connection time; the repository does not construct an explicit connection pool or TLS configuration.
4. Prisma connects lazily over TCP to the published host port `3306`. The MySQL server sees the connection as coming through the Docker port mapping, not necessarily as `localhost` at the server's own socket layer.
5. Each request handler, repository, and transaction issues SQL through Prisma. Reads and writes target the tables created by the migration.
6. InnoDB writes data and indexes to files under `/var/lib/mysql`, which is backed by the `oms_mysql_data` volume. The container filesystem outside that mount is ephemeral.

### 2.4 Test flow

1. `npm test` runs `vitest run` (`package.json:17`) without an explicit `--env-file` flag. Vitest's automatic `.env` loading covers only `VITE_`-prefixed variables, so `DATABASE_URL` and `JWT_SECRET` must already be present in the shell environment or injected by the caller. `vitest.config.ts` does not use `loadEnv` to import all variables.
2. `tests/setup.ts:1-6` imports `src/config/database.ts` and calls `prisma.$connect()` in `beforeAll`. Importing the database module triggers environment validation in `src/config/env.ts`.
3. `tests/setup.ts:8-16` deletes rows from `orderStatusHistory`, `orderItem`, `order`, `orderNumberSequence`, `product`, `customer`, and `user` before every test, in foreign-key-safe order.
4. `tests/auth.test.ts` and `tests/orders.test.ts` drive the Express application through Supertest and persist data through the same MySQL instance. `tests/helpers/factories.ts` creates users, customers, and products directly through Prisma.
5. `vitest.config.ts:4-18` disables file parallelism, uses a single forked worker, and sets 30-second test and hook timeouts. These settings avoid cross-test interference on the shared database but also mean the suite is serial.
6. `afterAll` calls `prisma.$disconnect()` (`tests/setup.ts:18-20`). The container is not stopped by the test suite; it must be running before the tests start and is left running afterwards.

### 2.5 Stop, restart, and teardown flow

1. `docker compose stop` or `docker stop oms-mysql` stops the server. The named volume remains.
2. `docker compose restart` or a Docker daemon restart starts the container again. Because the data directory is populated, provisioning is skipped and the existing data and credentials are used.
3. `docker compose down` removes the container and the default network but keeps the named volume. The next `up` reuses the data.
4. `docker compose down -v` or `docker volume rm <project>_oms_mysql_data` destroys the volume. The next `up` performs first-time initialization again, using the current environment values. All prior data is lost.
5. Replacing the image with a newer `8.0.x` patch or a different tag reuses the existing data directory. Schema compatibility is not checked by Compose; it is checked only when the application or migration tooling connects.

### 2.6 Failure paths observed in the flow

- **Port conflict:** if another process or container already listens on host port `3306`, the container fails to start because the mapping is fixed and there is no override file.
- **Credential drift:** if `MYSQL_PASSWORD` is changed without changing `DATABASE_URL`, or vice versa, the application receives an access-denied error at query time.
- **Wrong database name:** if `MYSQL_DATABASE` and the path component of `DATABASE_URL` differ, the application connects to a database that may not exist or may not contain the migrated tables.
- **Missing schema:** if `db:migrate` is skipped, the database exists but the seven tables do not. Prisma queries fail and the centralized error middleware maps most Prisma errors to HTTP 500.
- **Shadow database failure:** if root cannot authenticate from the host through the published port, or lacks `CREATE DATABASE`/`DROP DATABASE`, `prisma migrate dev` fails before applying any migration.
- **Healthcheck false positive:** because `mysqladmin ping` reports the server as alive even when authentication fails, a credential mismatch can still produce a healthy container status.
- **Destructive tests:** if `DATABASE_URL` points at a populated development database, `tests/setup.ts` deletes all application rows before each test.

---

## 3. Business Rules & Logic

The rules below are infrastructure and operational rules expressed by the Compose service, the environment contract, and the boundaries those artifacts create for the application, migrations, seeds, and tests. They are not application domain rules. Where a behavior comes from the official MySQL image entrypoint or from MySQL server semantics rather than from a repository file, the confidence column and the evidence paragraph say so explicitly.

## Overview of the business rules:

| Rule ID | Rule Type | Rule Description | Location | Confidence |
|---------|-----------|------------------|----------|------------|
| BR-01 | Runtime constraint | The database service runs the official `mysql:8.0` image tag; no digest or platform is pinned | `docker-compose.yml:3` | High |
| BR-02 | Operational | The container is named `oms-mysql` and uses `restart: unless-stopped`; it is a singleton per Docker host | `docker-compose.yml:4-5` | High |
| BR-03 | Provisioning | The database name defaults to `oms` and is created only during first initialization | `docker-compose.yml:8`; `.env.example:12` | High / Medium for entrypoint behavior |
| BR-04 | Provisioning / security | The application user defaults to `oms_user` with password `oms_password` and receives all privileges only on the `oms` database | `docker-compose.yml:9-10`; `.env.example:13-14` | High / Medium for grant scope |
| BR-05 | Security | The root password defaults to `root_password` and is used by the healthcheck and the shadow database URL | `docker-compose.yml:7,16`; `.env.example:6,11` | High / Medium for host scope |
| BR-06 | Configuration | Every Compose variable uses `${VAR:-default}`; unset or empty values fall back to committed defaults | `docker-compose.yml:7-10` | High |
| BR-07 | Network | Host port `3306` maps to container port `3306` on all host interfaces | `docker-compose.yml:11-12` | High |
| BR-08 | Persistence | `/var/lib/mysql` is stored in the named local volume `oms_mysql_data`; `down -v` destroys it | `docker-compose.yml:13-14,26-28` | High |
| BR-09 | Lifecycle | Provisioning variables take effect only on an empty data directory; later edits do not rotate existing credentials or create missing databases | Official image entrypoint behavior; `docker-compose.yml:6-10` | Medium |
| BR-10 | Availability | A `mysqladmin ping` healthcheck runs every 5 seconds with 5-second timeout, 20 retries, and 30-second start period | `docker-compose.yml:15-20` | High / Medium for probe semantics |
| BR-11 | Data encoding | The server default character set is `utf8mb4` | `docker-compose.yml:23` | High |
| BR-12 | Data semantics | The server default collation is `utf8mb4_unicode_ci`, which makes comparisons and unique indexes case-insensitive | `docker-compose.yml:24`; `prisma/migrations/20260519182739_init/migration.sql` | High / Medium for detailed collation behavior |
| BR-13 | Security / compatibility | The server default authentication plugin is `mysql_native_password` | `docker-compose.yml:22` | High / High for external support status |
| BR-14 | Operations | The Compose service does not create application tables; schema management is out-of-band through Prisma migrations | `docker-compose.yml:1-28`; `package.json:14-15` | High |
| BR-15 | Migration | `SHADOW_DATABASE_URL` points to `oms_shadow` under root credentials; Compose does not create that database and the application schema does not validate it | `.env.example:6`; `prisma/schema.prisma:5-9`; `src/config/env.ts:3-10` | High / Medium for root host scope |
| BR-16 | Schema | The migration lock fixes the provider to MySQL and the initial DDL uses MySQL 8.0 features and `utf8mb4_unicode_ci` | `prisma/migrations/migration_lock.toml:3`; `prisma/migrations/20260519182739_init/migration.sql:1-125` | High |
| BR-17 | Testing | The test suite requires a live database at `DATABASE_URL` and truncates all application tables before each test, with no environment guard | `tests/setup.ts:1-20`; `vitest.config.ts:4-18`; `package.json:17` | High |
| BR-18 | Network topology | The service joins the Compose default bridge network and is reachable as `mysql:3306` or `oms-mysql:3306` from peer containers | `docker-compose.yml:1-28` | High |
| BR-19 | Configuration consistency | Compose provisioning variables, `DATABASE_URL`, and `SHADOW_DATABASE_URL` must agree, but no code validates that agreement | `docker-compose.yml:7-10`; `.env.example:5-6,11-14` | High |
| BR-20 | Durability / recovery | There is one MySQL instance, one local volume, and no replication, backup, or point-in-time recovery | `docker-compose.yml:2-28` | High |
| BR-21 | Security | Client connections and data at rest are unencrypted; credentials are visible through container metadata and the healthcheck command | `docker-compose.yml:7-16`; `.env.example:5-6` | High |
| BR-22 | Resource governance | No CPU, memory, PID, ulimit, logging-driver, or MySQL tuning limits are declared | `docker-compose.yml:1-28` | High / Medium for image defaults |
| BR-23 | Time semantics | No server time zone is configured; `DATETIME(3)` columns and `CURRENT_TIMESTAMP(3)` defaults depend on the server session time zone | `docker-compose.yml:21-24`; `prisma/migrations/20260519182739_init/migration.sql` | Medium |

## Detailed breakdown of the business rules:

---

### Business Rule: BR-01 - MySQL 8.0 image contract

**Overview**:
The service declares `image: mysql:8.0` (`docker-compose.yml:3`). That tag is the only executable runtime identity of the component. It determines the server version, the entrypoint behavior, the supported SQL features, the available authentication plugins, the default server variables, and the base operating system of the container.

**Detailed description**:
The tag is a floating minor-version tag, not a digest. Docker resolves it to the current `8.0.x` patch image at pull time. Two developers, two CI runs, or two hosts can therefore run different patch releases while claiming the same Compose configuration. The tag also does not pin a platform, so an ARM host and an AMD64 host can receive different image manifests. The repository contains no `image@sha256:` reference, no `platform:` key, and no local image build definition. The MySQL 8.0 provider matches the Prisma datasource provider `mysql` (`prisma/schema.prisma:5-9`) and supports every feature used by the migration: `ENUM`, `JSON`, `CHAR(36)`, `DATETIME(3)`, `TEXT`, foreign keys, and `utf8mb4` collations.

The local use cases are development, schema migration, seed data, and integration testing. The image is not used by an application container because no `Dockerfile` or application service exists in the repository. The Node process, Prisma CLI, seed script, and test runner all run on the host and connect through the published port. This means the image contract is also a host-tooling contract: the host must be able to resolve `localhost:3306` and speak the MySQL protocol through Prisma's query engine.

The ambiguity is version drift. The dependency audit records that MySQL 8.0 entered Oracle Sustaining Support on 2026-04-21, and the image tag is not digest-pinned (`docs/agents/dependency-auditor/dependencies-report-2026-11-08_22-50-33.md`). The container can therefore receive patch updates without any repository change, and it can stop receiving security fixes while still resolving successfully. The exact patch version and image digest cannot be determined from the repository alone because no Docker daemon or registry lookup was performed for this analysis.

**Rule workflow**:
1. Compose reads `image: mysql:8.0`.
2. Docker checks the local image cache for the tag.
3. If absent, Docker pulls the current `8.0` manifest from the registry.
4. Docker starts the resolved image with the declared command flags and environment.
5. The resolved patch version remains fixed only for the lifetime of that local image until it is pulled again.

**Evidence**: `docker-compose.yml:3`; confidence High for the declaration, Medium for the resolved patch and digest.

---

### Business Rule: BR-02 - Container identity and restart policy

**Overview**:
The service sets `container_name: oms-mysql` (`docker-compose.yml:4`) and `restart: unless-stopped` (`docker-compose.yml:5`). The container has a fixed, human-readable name and a supervision policy that restarts it after crashes and daemon restarts unless an operator explicitly stopped it.

**Detailed description**:
The fixed container name creates a singleton identity on the Docker host. Operators can use `docker exec -it oms-mysql ...`, `docker logs oms-mysql`, and `docker inspect oms-mysql` without discovering a generated name. The name is global to the Docker daemon, not scoped to the Compose project. Two checkouts of the repository, two Compose projects, or a second stack that also declares `oms-mysql` cannot run simultaneously on the same host. Compose would normally generate a project-prefixed name and allow parallel projects; the explicit name removes that isolation.

The restart policy means Docker starts the container when the daemon starts, restarts it when the main process exits with a non-zero status, and leaves it stopped after a manual stop. It does not restart based on health status. A container that is running but unhealthy is not restarted by this policy, and there is no maximum retry count or delay configuration. A permanent configuration error, such as a bad command flag or an incompatible data directory, can therefore produce a restart loop rather than a terminal failed state. There is no explicit `stop_grace_period`; the Docker default applies.

The operational use case is a long-lived local database that survives host reboots and terminal sessions. The limitation is that the policy supervises the `mysqld` process, not the data or the application. It does not make the database highly available, does not preserve data if the volume is removed, and does not coordinate with application startup because the application is not a Compose service.

**Rule workflow**:
1. Docker creates the container with the literal name `oms-mysql`.
2. On daemon start, Docker starts the container unless it was manually stopped.
3. If `mysqld` exits non-zero, Docker applies the `unless-stopped` policy and restarts it.
4. A manual `docker stop` marks the container as intentionally stopped; daemon restarts do not override that state.
5. Health status does not influence the restart decision.

**Evidence**: `docker-compose.yml:4-5`; confidence High.

---

### Business Rule: BR-03 - Database provisioning

**Overview**:
`MYSQL_DATABASE` defaults to `oms` (`docker-compose.yml:8`, `.env.example:12`). The database name is the schema namespace that the application connection string also targets (`DATABASE_URL=mysql://.../oms`, `.env.example:5`). The official image entrypoint is responsible for creating the database, and it does so only during first initialization.

**Detailed description**:
On an empty data directory, the entrypoint creates the database named by `MYSQL_DATABASE` if it does not exist. It then uses that database as the grant scope for the application user. The repository does not mount any SQL file into `/docker-entrypoint-initdb.d`, so the database starts empty: no application tables, no indexes, no foreign keys, and no `_prisma_migrations` table. Schema creation is a separate step performed by Prisma migrations. The database name is not validated against the path component of `DATABASE_URL`; the two values are independent strings that happen to share the same default.

The primary use case is a zero-configuration first boot for local development. A developer can run `docker compose up -d`, then `npm run db:migrate`, and obtain the expected schema. The database name also appears in the application's Prisma datasource resolution, so a mismatch between `MYSQL_DATABASE` and `DATABASE_URL` produces either an unknown-database error or a connection to a different, unmigrated database. The Compose fallback and the example URL both use `oms`, which is why the default path works.

The limitation is that the provisioning rule is not idempotent with respect to configuration changes. If the volume already contains a data directory, the entrypoint skips initialization. Changing `MYSQL_DATABASE` in `.env` and restarting the container does not create the new database and does not move existing data. The only ways to apply the new value are manual SQL, a new volume, or a full reset. The exact entrypoint behavior is external to the repository, so confidence in the first-initialization-only semantics is medium.

**Rule workflow**:
1. Compose resolves `MYSQL_DATABASE`, defaulting to `oms`.
2. Docker starts the container with an empty or populated `/var/lib/mysql`.
3. If empty, the entrypoint creates the database and applies grants.
4. If populated, the entrypoint skips creation and uses the existing data directory.
5. Prisma migrations later create the application tables inside the selected database.

**Evidence**: `docker-compose.yml:8`; `.env.example:5,12`; `prisma/schema.prisma:5-9`; confidence High for the declared value, Medium for entrypoint lifecycle behavior.

---

### Business Rule: BR-04 - Application user provisioning and grant scope

**Overview**:
`MYSQL_USER` defaults to `oms_user` and `MYSQL_PASSWORD` defaults to `oms_password` (`docker-compose.yml:9-10`, `.env.example:13-14`). The official image entrypoint creates that account and grants it privileges on the `MYSQL_DATABASE` schema only. The application connection string uses the same user and password (`DATABASE_URL=mysql://oms_user:oms_password@localhost:3306/oms`).

**Detailed description**:
The application account is the identity used by the Node process, the Prisma Client, the seed script, and the integration tests. The entrypoint creates the account with access from any host (`'oms_user'@'%'`) and grants all privileges on the `oms` database. "All privileges" on a schema includes data manipulation, DDL, index creation, and locking within that schema. It does not include global administrative privileges such as creating or dropping other databases. That distinction matters because the shadow database is a separate schema and uses root credentials instead of this account.

The use case is a single account that can both run application queries and support schema migrations inside the application schema. Prisma migrations executed with this account can create and alter tables in `oms`. The account cannot create `oms_shadow`, which is why `SHADOW_DATABASE_URL` uses root. The account is also used from the host through the published port, so its host pattern must permit the Docker port-mapping source address. The official image's `'%'` host pattern supports that, subject to the server's name-resolution behavior.

The limitation is least privilege. The runtime application account has DDL rights on the entire application schema, even though the runtime workload only needs DML. The password is stored in plaintext in `.env.example` and in Compose fallbacks, and it is visible through `docker inspect` and `docker compose config`. If the volume already exists, changing `MYSQL_PASSWORD` in the environment does not change the existing account password. The grant scope is external entrypoint behavior, so confidence in the exact SQL is medium; the declared variables are high confidence.

**Rule workflow**:
1. Compose resolves `MYSQL_USER` and `MYSQL_PASSWORD`, defaulting to `oms_user` and `oms_password`.
2. On an empty data directory, the entrypoint creates `'oms_user'@'%'` with the supplied password.
3. The entrypoint grants all privileges on the `oms` database to that account.
4. The application connects with the same credentials from `DATABASE_URL`.
5. On a populated volume, no account creation or password change occurs.

**Evidence**: `docker-compose.yml:9-10`; `.env.example:5,13-14`; confidence High for declarations, Medium for entrypoint SQL and host-pattern details.

---

### Business Rule: BR-05 - Root credential contract

**Overview**:
`MYSQL_ROOT_PASSWORD` defaults to `root_password` (`docker-compose.yml:7`, `.env.example:11`). The root credential is used by the container healthcheck (`docker-compose.yml:16`) and by the Prisma shadow database URL (`SHADOW_DATABASE_URL=mysql://root:root_password@localhost:3306/oms_shadow`, `.env.example:6`). Root is the administrative identity for the instance.

**Detailed description**:
The root password is the most privileged credential in the component. It is set during first initialization and is intended to remain stable for the life of the data directory. The healthcheck uses it to authenticate locally inside the container with `mysqladmin ping -h localhost -uroot -p...`. The shadow database URL uses it from the host through the published port so that Prisma can create and drop `oms_shadow`. The application runtime does not use root; it uses the less-privileged application account.

The use cases are migration diffing and container health probing. Both depend on root having broad privileges. The shadow database workflow requires global `CREATE DATABASE` and `DROP DATABASE` rights, which the application account does not have. The healthcheck only needs a local connection, but the command still carries the root password as an argument. Because the password is interpolated by Compose at parse time, it appears in the container's healthcheck configuration and can be read through `docker inspect` or `docker compose config`.

The ambiguity is host scope. The Compose file does not set `MYSQL_ROOT_HOST`. The official image's default root account is scoped to `localhost` inside the container unless that variable is set. A host-side client connecting through the published port is not necessarily seen by `mysqld` as `localhost`; it may be seen as the Docker bridge gateway address. In that case, `SHADOW_DATABASE_URL` using root from the host can be rejected even though the password is correct. The repository does not document or test this path. Confidence is high for the declared password and its two consumers, medium for the effective root host scope.

**Rule workflow**:
1. Compose resolves `MYSQL_ROOT_PASSWORD`, defaulting to `root_password`.
2. On first initialization, the entrypoint sets the root password.
3. The healthcheck authenticates as root over localhost inside the container.
4. Prisma CLI authenticates as root from the host for the shadow database.
5. On a populated volume, changing the environment variable does not rotate the existing root password.

**Evidence**: `docker-compose.yml:7,16`; `.env.example:6,11`; confidence High for declared values, Medium for root host scope and entrypoint behavior.

---

### Business Rule: BR-06 - Environment interpolation and fallback defaults

**Overview**:
Every variable in the Compose `environment` block uses the `${VAR:-default}` form (`docker-compose.yml:7-10`). Docker Compose substitutes the shell environment or the project `.env` file when a value is present and non-empty; otherwise it uses the literal fallback. The repository ships `.env.example`, not `.env`, so the fallbacks are the only values that exist in a clean checkout.

**Detailed description**:
The fallback syntax makes the service startable without any local configuration. It also means that the committed values in `.env.example` and the fallbacks in `docker-compose.yml` are duplicated. The four fallbacks are `root_password`, `oms`, `oms_user`, and `oms_password`. The application connection strings in `.env.example:5-6` are separate strings that embed the same credentials. Docker Compose automatically loads `.env` from the project directory for interpolation; it does not load `.env.example`. The Node process loads `.env` separately through `--env-file=.env` (`package.json:11-13`).

The use case is developer convenience: a new checkout can start the database without editing files, and an operator can override one or more values through the shell or `.env`. The same mechanism creates a configuration contract with three independent sources: the shell environment, the Compose `.env`, and the application `.env` consumed by Node. Compose precedence generally favors the shell over `.env`, while the application's `--env-file` behavior and the shell can also interact. The repository does not document the precedence or provide a single source of truth.

The limitation is silent divergence. If an operator changes only `MYSQL_PASSWORD`, the application still uses the old password from `DATABASE_URL`. If an operator changes only `DATABASE_URL`, the server still has the old account password. If `MYSQL_DATABASE` and the URL path differ, the application targets a different schema. No validation compares these values, and the healthcheck may still report healthy because liveness is independent of credential correctness.

**Rule workflow**:
1. Compose reads the shell environment for each `${VAR}`.
2. If absent or empty, Compose reads the project `.env` file.
3. If still absent or empty, Compose applies the literal fallback after `:-`.
4. Compose writes the resolved values into the container environment and healthcheck command.
5. The Node process independently reads `.env` through `--env-file`, so its values can differ from the values Compose resolved.

**Evidence**: `docker-compose.yml:7-10`; `.env.example:5-6,11-14`; `.gitignore:4`; confidence High.

---

### Business Rule: BR-07 - Host port publication

**Overview**:
The service maps `'3306:3306'` (`docker-compose.yml:11-12`). The left side is the host port, the right side is the container port. No host IP address is specified, so Docker publishes the port on all host interfaces, typically IPv4 `0.0.0.0` and IPv6 `::`.

**Detailed description**:
The port mapping is the only path from host processes to the database. The Node application, the Prisma CLI, the seed script, and the test runner all connect to `localhost:3306` using `DATABASE_URL` (`mysql://oms_user:oms_password@localhost:3306/oms`). The database is therefore reachable from any process on the host, not only from the repository. If the host has a public network interface and the firewall permits inbound traffic, the database is also reachable from the network. The Compose file does not bind the mapping to `127.0.0.1`, does not define a reverse proxy, and does not enable MySQL TLS.

The use case is straightforward: host-run Node code needs a stable port. A fixed host port also makes the connection strings in `.env.example` deterministic. The cost is port exclusivity and exposure. Only one process can bind host port `3306`; a locally installed MySQL server, another container, or a second checkout with the same mapping conflicts. The exposure is not limited to the Docker network because published ports are handled by the host's network stack.

The rule has no environment-specific variation in the repository. There is no `docker-compose.override.yml`, no profile, and no parameterized host port. Changing the host port requires editing the Compose file or supplying a separate override file, and it also requires changing every connection string that assumes `3306`. The container port itself cannot be changed without changing the MySQL server's `port` setting, which is not configured.

**Rule workflow**:
1. Compose requests a host port binding for container port `3306`.
2. Docker binds the host's wildcard address on port `3306`.
3. Host clients resolve `localhost` and connect to the published port.
4. Docker forwards the TCP stream to the container's `mysqld` listener.
5. If host port `3306` is already in use, container startup fails.

**Evidence**: `docker-compose.yml:11-12`; `.env.example:5-6`; confidence High.

---

### Business Rule: BR-08 - Named volume persistence and destructive lifecycle

**Overview**:
`/var/lib/mysql` is mounted from the named volume `oms_mysql_data` (`docker-compose.yml:13-14`). The volume uses the `local` driver (`docker-compose.yml:26-28`). Data survives container removal and recreation, but is destroyed by `docker compose down -v` or by removing the volume.

**Detailed description**:
The named volume is the only persistent state of the component. Everything else in the container filesystem is disposable. Because the volume is declared without an explicit `name`, Compose prefixes it with the project name, producing a Docker volume such as `<project>_oms_mysql_data`. The exact prefix depends on the directory name or the `-p` flag. The volume is not a bind mount, so the host path is managed by Docker and is not visible in the repository. The local driver stores data on the host filesystem, but the repository does not declare a specific path, filesystem, size limit, or encryption option.

The use case is continuity across development sessions, container upgrades, and `docker compose down` without `-v`. Recreating the container with the same volume preserves users, tables, rows, grants, and the migration history table. This is also what makes the first-initialization rule consequential: the entrypoint sees a populated directory and skips provisioning. The volume is not shared between projects because of the project prefix, although an operator can explicitly reference the same volume from another stack.

The destructive boundary is important. `docker compose down -v` removes the volume and all data. `docker volume rm` does the same. `docker compose down` without `-v` does not. There is no backup job, snapshot, export, or replication, so the volume is a single copy. A host filesystem failure, accidental volume deletion, or destructive test against the same volume loses the data permanently. The volume has no explicit size limit, so it can grow until the host filesystem is exhausted.

**Rule workflow**:
1. Compose creates or reuses the project-scoped volume for `oms_mysql_data`.
2. Docker mounts it at `/var/lib/mysql` in the container.
3. `mysqld` writes data files, redo logs, undo logs, and the migration history there.
4. Container removal without `-v` leaves the volume intact.
5. `down -v` or volume removal deletes the data and forces first-time initialization on the next start.

**Evidence**: `docker-compose.yml:13-14,26-28`; confidence High, Medium for the exact project-prefixed volume name.

---

### Business Rule: BR-09 - First-initialization-only provisioning

**Overview**:
The `MYSQL_*` environment variables are consumed by the official image entrypoint only when the data directory is empty. On a populated volume, changing `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, or `MYSQL_PASSWORD` does not alter the existing database, account, or password.

**Detailed description**:
The official MySQL image entrypoint checks whether the data directory already contains an initialized database. If it does, the entrypoint skips the initialization SQL and starts the server with the existing data. This behavior is not declared in `docker-compose.yml`; it comes from the image. The repository's Compose file has no init scripts, no pre-start hook, and no credential-rotation mechanism. The environment variables describe the desired first-boot state, not the ongoing state.

The use case is a stable local database across restarts. A developer can stop and start the container, change unrelated `.env` values, or upgrade the image patch, and the database remains intact. The limitation is that credential and database-name changes become silent no-ops. An operator who changes `MYSQL_PASSWORD` and restarts expects the application to use the new password, but the server still has the old one. The application's `DATABASE_URL` may also have been updated, producing an authentication failure. The same applies to `MYSQL_DATABASE`: a new database name is not created on an existing volume.

The healthcheck can mask this condition. `mysqladmin ping` is a liveness probe; it reports that the server is responding, not that the supplied credentials are valid. A container can therefore appear healthy while the application cannot authenticate. The only reliable ways to apply provisioning changes are manual SQL with an administrative account, removal of the volume, or creation of a new volume. Confidence is medium because the entrypoint is external to the repository.

**Rule workflow**:
1. Docker starts the container with the configured `MYSQL_*` variables.
2. The entrypoint checks `/var/lib/mysql` for an existing initialized database.
3. If empty, it applies root password, database creation, user creation, and grants.
4. If populated, it skips all provisioning SQL.
5. `mysqld` starts and serves the existing state regardless of environment changes.

**Evidence**: `docker-compose.yml:6-10`; official image entrypoint behavior; confidence Medium.

---

### Business Rule: BR-10 - Healthcheck contract

**Overview**:
The healthcheck runs `mysqladmin ping -h localhost -uroot -p${MYSQL_ROOT_PASSWORD:-root_password}` (`docker-compose.yml:16`) every 5 seconds with a 5-second timeout, 20 retries, and a 30-second start period (`docker-compose.yml:17-20`). Docker marks the container healthy or unhealthy based on the command exit status.

**Detailed description**:
The probe budget is approximately 30 seconds of start period plus 20 retries at 5-second intervals, for roughly 130 seconds before the container is declared unhealthy. Each probe has a 5-second timeout. The check runs inside the container and connects to `localhost`, so it uses the container's loopback path rather than the published host port. The root password is interpolated by Compose into the command, which means it is part of the container configuration and is visible through Docker inspection.

The intended use case is operator visibility and, if the service is later placed behind an orchestrator, a basic availability signal. The health status is not consumed by the application because the application is not a Compose service and has no `depends_on` condition. It is also not consumed by the restart policy, which restarts on process exit, not on health failure. The probe does not check that the application database exists, that the application user can authenticate, that the schema has been migrated, or that the server can execute a query.

The limitation is the semantics of `mysqladmin ping`. It is a liveness check: the command reports the server as alive when the server responds, and an access-denied response can still produce a successful exit status. It is therefore not a credential test and not a readiness test for the application. A container with a stale root password, a missing application database, or an empty schema can still be reported healthy. The exact behavior is documented by MySQL, not by this repository, so confidence in the declared timing is high and confidence in the probe's diagnostic strength is medium.

**Rule workflow**:
1. Docker starts the container and begins the 30-second start period.
2. Every 5 seconds, Docker executes `mysqladmin ping` inside the container.
3. The command attempts a root connection to localhost and exits.
4. A zero exit status marks the probe successful; a non-zero status counts as a failure.
5. After 20 consecutive failures, Docker marks the container unhealthy; the container keeps running.
6. The application and restart policy do not observe the health status.

**Evidence**: `docker-compose.yml:15-20`; confidence High for declared values, Medium for `mysqladmin ping` semantics.

---

### Business Rule: BR-11 - Server character set

**Overview**:
The server starts with `--character-set-server=utf8mb4` (`docker-compose.yml:23`). This sets the default character set for new databases, tables, and columns that do not specify one. The migration DDL also declares `utf8mb4` explicitly (`prisma/migrations/20260519182739_init/migration.sql`).

**Detailed description**:
`utf8mb4` is the four-byte UTF-8 character set. It can store the full Unicode range, including supplementary-plane characters such as emoji. The alternative `utf8mb3` stores only up to three bytes and cannot represent those characters. The application stores names, addresses, product descriptions, order notes, and status reasons, so full Unicode support is relevant for Portuguese accents, international customer names, and free-text fields. The setting applies to new objects; existing objects created before the setting change would keep their own character set.

The use case is consistent text storage across the schema. The migration declares the same character set on every table, so the server default and the DDL agree for the initial schema. The database connection also negotiates a client character set through the Prisma/MySQL driver. The Compose file does not set `--character-set-client-handshake` or `--character-set-filesystem`, so those remain at image defaults. If a client negotiates a narrower character set, MySQL may transcode or reject characters depending on the operation.

The limitation is that the server default is not the only source of truth. A table or column can override it, and the migration does specify it explicitly. The component does not enforce a particular client character set, so application behavior depends on the driver. The setting is visible and deterministic in the Compose file, so confidence is high.

**Rule workflow**:
1. `mysqld` starts with `--character-set-server=utf8mb4`.
2. New databases inherit the server character set unless they specify one.
3. The migration DDL explicitly declares `utf8mb4` on each table.
4. Clients negotiate their own connection character set.
5. Text values are stored, compared, and returned according to the column and connection character sets.

**Evidence**: `docker-compose.yml:23`; `prisma/migrations/20260519182739_init/migration.sql:1-125`; confidence High.

---

### Business Rule: BR-12 - Server collation and comparison semantics

**Overview**:
The server starts with `--collation-server=utf8mb4_unicode_ci` (`docker-compose.yml:24`). The migration DDL declares the same collation on every table. The `_ci` suffix means case-insensitive comparisons and unique indexes. The `unicode` part identifies an older Unicode collation algorithm rather than MySQL 8.0's default `utf8mb4_0900_ai_ci`.

**Detailed description**:
Collation controls equality, ordering, `LIKE`, and unique-index behavior. Under `utf8mb4_unicode_ci`, `User@Example.com` and `user@example.com` compare as equal, so the unique index on `users.email` treats them as the same key. The same applies to `customers.email` and to any application query that compares strings. The application does not normalize e-mail addresses or names before writing them, so the effective identity semantics depend on this database setting. The auth and customers modules rely on the unique index and on case-insensitive lookups without documenting the dependency.

The use case is user-friendly search and identity handling: a login by e-mail can succeed regardless of case, and a customer search by name can match different capitalizations. The migration's explicit `DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci` makes the initial tables independent of the server default, although both currently agree. The collation is accent-insensitive for many characters, but the exact behavior follows the older Unicode 4.0 rules, not the Unicode 9.0 rules used by `utf8mb4_0900_ai_ci`. Characters added after Unicode 4.0 can therefore sort or compare differently than they would under the MySQL 8.0 default.

The limitation is environment dependence and hidden semantics. If the database is moved to a server with MySQL 8.0's default collation, or if a table is created without the explicit DDL, e-mail uniqueness and search behavior can change. The repository does not have a test that proves case-insensitive uniqueness; the duplicate-email test uses the same exact string (`tests/auth.test.ts:19-28`). The component also does not use a binary collation for any field, so there is no case-sensitive string comparison for tokens or secrets. Confidence is high for the declared setting and medium for the detailed linguistic behavior.

**Rule workflow**:
1. `mysqld` starts with `--collation-server=utf8mb4_unicode_ci`.
2. New databases inherit the server collation unless they specify one.
3. The migration DDL declares `utf8mb4_unicode_ci` on each table.
4. String equality, unique indexes, `LIKE`, and `ORDER BY` use the column collation.
5. Application queries observe case-insensitive comparisons for the affected columns.

**Evidence**: `docker-compose.yml:24`; `prisma/migrations/20260519182739_init/migration.sql:13,29,47,69,84,99,107`; confidence High for the setting, Medium for detailed collation behavior.

---

### Business Rule: BR-13 - Default authentication plugin

**Overview**:
The server starts with `--default-authentication-plugin=mysql_native_password` (`docker-compose.yml:22`). This makes `mysql_native_password` the default plugin for accounts created without an explicit plugin. It is a compatibility choice with security and lifecycle consequences.

**Detailed description**:
`mysql_native_password` uses a SHA-1-based challenge-response protocol. It predates MySQL 8.0's default `caching_sha2_password`, which uses SHA-256 and supports faster authentication and stronger password handling. The flag is deprecated in later MySQL 8.0 patch releases and was removed in MySQL 8.4. The Compose file pins the image to the `8.0` tag, so the flag currently works, but it couples the service to the 8.0 line. If the image tag is changed to a newer major version, the server can fail to start because the option no longer exists.

The use case is broad client compatibility. Some older MySQL clients and drivers do not support `caching_sha2_password` without additional configuration. Prisma 5.22 supports both plugins, so the flag is not required by the application stack. The flag affects accounts created without an explicit plugin, including the application user created by the entrypoint. It does not, by itself, enable TLS or protect the authentication exchange from network observation; the published port carries the protocol in plaintext.

The limitation is security posture. Native password authentication is weaker than `caching_sha2_password`, and the server does not set `--require_secure_transport`. The component therefore combines a legacy authentication plugin with plaintext transport on a port published to all host interfaces. The exact deprecation and removal status is external MySQL lifecycle information, already recorded by the dependency audit, so confidence is high for that status and high for the declared flag.

**Rule workflow**:
1. `mysqld` starts with the native-password default.
2. Accounts created without an explicit plugin use `mysql_native_password`.
3. The entrypoint-created application account authenticates with the native plugin.
4. Clients negotiate the plugin during the connection handshake.
5. If the image is moved to a version where the flag is removed, server startup fails.

**Evidence**: `docker-compose.yml:22`; `docs/agents/dependency-auditor/dependencies-report-2026-11-08_22-50-33.md`; confidence High for declaration and lifecycle status.

---

### Business Rule: BR-14 - Out-of-band schema management

**Overview**:
The Compose service provisions a database but not a schema. There is no `/docker-entrypoint-initdb.d` mount, no init SQL file, and no application service that runs migrations. The seven application tables are created only when `npm run db:migrate` (`package.json:14`) or an equivalent Prisma command is executed.

**Detailed description**:
`docker compose up -d` produces a running MySQL server with an empty `oms` database. The migration in `prisma/migrations/20260519182739_init/migration.sql` is not applied automatically. The application starts successfully even if the tables are missing because Prisma connects lazily; the first query fails instead. The Express error middleware maps most Prisma errors to HTTP 500, so a missing schema appears as a generic server error rather than a startup failure.

The use case is controlled schema evolution. Prisma owns the schema history, records applied migrations in `_prisma_migrations`, and can detect drift. This is preferable to hand-written init SQL for a project that already uses Prisma. The seed script is also separate (`npm run db:seed`, `package.json:16`), so data population is a deliberate step. The repository exposes `db:migrate`, `db:reset`, and `db:seed`, but it does not expose `prisma migrate deploy`, which is the non-interactive command typically used in CI or production.

The limitation is operational fragility. A new developer who starts the container but forgets the migration step sees a running but unusable database. A CI environment must run migrations explicitly before tests. A production-like environment would need a migration step outside this Compose file. The first-initialization rule compounds the issue: recreating the container with a populated volume does not reapply migrations, so schema drift is possible if the volume predates a migration. Confidence is high because the absence of init scripts and the presence of the Prisma scripts are directly observable.

**Rule workflow**:
1. `docker compose up -d` starts MySQL and creates the empty database.
2. The operator runs `npm run db:migrate`.
3. Prisma CLI connects, creates the shadow database if needed, and applies pending migrations to `oms`.
4. Prisma records the applied migration in `_prisma_migrations`.
5. The operator optionally runs `npm run db:seed`.
6. The application starts and queries the migrated schema.

**Evidence**: `docker-compose.yml:1-28`; `package.json:14-16`; `prisma/migrations/20260519182739_init/migration.sql`; confidence High.

---

### Business Rule: BR-15 - Shadow database contract

**Overview**:
`SHADOW_DATABASE_URL` is `mysql://root:root_password@localhost:3306/oms_shadow` (`.env.example:6`). Prisma uses it for schema diffing during `prisma migrate dev`. The Compose service does not create `oms_shadow`, and `src/config/env.ts:3-10` does not validate the variable.

**Detailed description**:
The shadow database is a temporary schema that Prisma creates, uses, and drops to detect schema drift during development migrations. It requires an account with global `CREATE DATABASE` and `DROP DATABASE` privileges, which is why the URL uses root rather than `oms_user`. The URL points to the same server and port as the application database but a different schema name. Because Compose only provisions `MYSQL_DATABASE=oms`, the `oms_shadow` schema does not exist at first boot. Prisma is expected to create it automatically when the root account has the necessary privileges.

The use case is developer-driven migration authoring. `npm run db:migrate` runs `prisma migrate dev`, which needs the shadow database. `npm run db:reset` also runs Prisma reset logic and can depend on the same tooling. The application runtime never reads `SHADOW_DATABASE_URL`; `src/config/env.ts` omits it, so a valid application environment can still fail at migration time. This separation is a deliberate boundary between runtime configuration and tooling configuration, but the repository does not document it.

The limitation is privilege and host scope. If root cannot authenticate from the host through the published port, or if the root account lacks global database-creation rights, the migration fails. If `oms_shadow` already exists and contains unrelated data, Prisma may drop or reset it. The shadow URL uses the same weak default password as the root account, and it is committed in `.env.example`. The host scope of root is not configured through `MYSQL_ROOT_HOST` in the Compose file, so the exact ability of a host client to connect as root is environment-dependent. Confidence is high for the declared URL and its consumer, medium for the effective root privileges and host matching.

**Rule workflow**:
1. The operator runs `npm run db:migrate` or `npm run db:reset`.
2. Prisma CLI reads `SHADOW_DATABASE_URL` from the environment.
3. Prisma connects to `localhost:3306` as root.
4. Prisma creates `oms_shadow` if it does not exist and has permission.
5. Prisma applies the schema diff and drops the shadow database.
6. If any step fails, the migration does not proceed.

**Evidence**: `.env.example:6`; `prisma/schema.prisma:5-9`; `src/config/env.ts:3-10`; confidence High for the contract, Medium for root host scope.

---

### Business Rule: BR-16 - Migration lock and DDL compatibility

**Overview**:
`prisma/migrations/migration_lock.toml:3` fixes the provider to `mysql`. The initial migration (`prisma/migrations/20260519182739_init/migration.sql:1-125`) creates seven tables, two enum-based status columns, JSON storage, millisecond `DATETIME` columns, `CHAR(36)` identifiers, indexes, and foreign keys with `RESTRICT` and `CASCADE` actions.

**Detailed description**:
The migration lock prevents Prisma from generating migrations for a different database provider. It is a repository-level contract that the schema is MySQL, not PostgreSQL or SQLite. The DDL uses MySQL-specific syntax: backtick-quoted identifiers, inline `ENUM(...)`, `JSON`, `DATETIME(3)`, `BOOLEAN` as a tiny integer alias, and `DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci`. The tables are `users`, `customers`, `products`, `orders`, `order_items`, `order_status_history`, and `order_number_sequence`. The enum values are `UserRole` (`ADMIN`, `OPERATOR`) and `OrderStatus` (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`).

The use case is a deterministic schema baseline for the application. Foreign keys enforce referential integrity: orders reference customers and users with `ON DELETE RESTRICT`, order items reference orders with `ON DELETE CASCADE` and products with `ON DELETE RESTRICT`, and status history references orders with `ON DELETE CASCADE` and users with `ON DELETE RESTRICT`. The single-row `order_number_sequence` table is the persistence backing for order-number allocation. Indexes support customer, status, date, and creator lookups. The JSON `address` column stores a structured object that MySQL validates as JSON.

The limitation is that the DDL is not idempotent. It uses plain `CREATE TABLE` and `ALTER TABLE ... ADD CONSTRAINT`, not `IF NOT EXISTS`. Re-running the file directly against an initialized database fails. Prisma's `_prisma_migrations` table is the mechanism that prevents re-application, so any manual schema work outside Prisma can desynchronize the history. The DDL's explicit collation also means the migration is not portable to a different collation without editing it. Confidence is high because both the lock and the DDL are explicit repository files.

**Rule workflow**:
1. Prisma reads the migration lock and confirms the MySQL provider.
2. Prisma compares the migration history with `_prisma_migrations`.
3. Pending migrations are applied in order inside a transaction where supported.
4. The initial migration creates the seven tables, indexes, and foreign keys.
5. The migration is recorded as applied.

**Evidence**: `prisma/migrations/migration_lock.toml:3`; `prisma/migrations/20260519182739_init/migration.sql:1-125`; confidence High.

---

### Business Rule: BR-17 - Test harness database contract

**Overview**:
The test suite requires a live MySQL server reachable through `DATABASE_URL`. `tests/setup.ts:1-20` connects Prisma before all tests, deletes every application table before each test, and disconnects after all tests. `vitest.config.ts:4-18` runs the suite serially in a single fork with 30-second timeouts. `package.json:17` runs `vitest run`; Vitest autoloads only `VITE_`-prefixed variables from `.env`, so `DATABASE_URL` and `JWT_SECRET` are not autoloaded and must come from the shell or another injector.

**Detailed description**:
The harness is destructive by design. It deletes `orderStatusHistory`, `orderItem`, `order`, `orderNumberSequence`, `product`, `customer`, and `user` in that order before every test. The order respects foreign keys. There is no check of `NODE_ENV`, no check that the database name contains "test", no separate test database, and no transaction rollback strategy. Whatever database `DATABASE_URL` points to is truncated. The same default URL in `.env.example` points to `oms`, which is also the development database. Running `npm test` with the example configuration therefore deletes development data before every test.

The use case is deterministic integration testing against the real database engine. The tests exercise Prisma, MySQL constraints, transactions, and the Express pipeline together. The serial configuration avoids concurrent truncation and cross-test interference. The 30-second timeouts accommodate container startup and query latency. The harness does not start or stop the container; the database must already be running and migrated. There is no `testcontainers` dependency and no per-suite schema isolation.

The limitation is environmental coupling. `npm test` does not pass `--env-file=.env`, so the operator must export `DATABASE_URL` and `JWT_SECRET` or use a shell wrapper. Importing `src/config/database.ts` triggers `src/config/env.ts` validation, which calls `process.exit(1)` if `DATABASE_URL` is missing or `JWT_SECRET` is shorter than 16 characters. A missing database produces a connection failure in `beforeAll`, and a missing schema produces Prisma errors in individual tests. The suite has no test for the Compose file, the healthcheck, the volume, or credential rotation. Confidence is high because the harness and configuration are explicit.

**Rule workflow**:
1. The operator ensures MySQL is running and migrated.
2. The operator exports `DATABASE_URL` and `JWT_SECRET`, or runs the tests in an environment that already has them.
3. Vitest loads `tests/setup.ts` and connects Prisma in `beforeAll`.
4. Before each test, the harness deletes all application rows.
5. Tests create data through factories or HTTP requests and assert against the live database.
6. `afterAll` disconnects Prisma; the container is left running.

**Evidence**: `tests/setup.ts:1-20`; `vitest.config.ts:4-18`; `package.json:17`; `.env.example:5,8`; confidence High.

---

### Business Rule: BR-18 - Network topology and service discovery

**Overview**:
The Compose file declares no `networks:` section. Docker Compose therefore attaches the `mysql` service to the project's default bridge network. The service is addressable by its service name `mysql` and by its container name `oms-mysql` from other containers on the same network. Host processes reach it through the published `localhost:3306` mapping.

**Detailed description**:
The default network is created per Compose project and named with the project prefix, for example `<project>_default`. Containers in the same project can resolve `mysql` through Docker's embedded DNS. Containers in other projects cannot resolve that alias unless they are explicitly attached to the same network. The container name `oms-mysql` is also resolvable on that network, but it is global to the Docker host, which is why the singleton rule matters. There is no custom subnet, static IP, network alias, or network policy. The service does not declare `expose`; the port is reachable on the network because `mysqld` listens on `0.0.0.0:3306` inside the container.

The use case is future container-to-container access. If a webhook worker is later added to the same Compose project, it can use a connection string with host `mysql` or `oms-mysql` and port `3306` without going through the host. The current application is not a Compose service, so it uses the published port instead. The network does not isolate the database from the host because the port mapping bypasses that boundary.

The limitation is that no network isolation policy exists. The default bridge network permits all containers in the project to reach the database port. There is no TLS, no network encryption, and no firewall rule inside the Compose file. The published port is reachable from any host interface. The repository does not define a separate network for the application tier or a network policy for the database. Confidence is high because the absence of a `networks:` key and the presence of the port mapping are explicit.

**Rule workflow**:
1. Compose creates the project default bridge network.
2. The `mysql` container is attached to it with aliases `mysql` and `oms-mysql`.
3. Peer containers resolve those aliases through Docker DNS.
4. Host processes connect through the published `3306` mapping.
5. No additional network policy or encryption applies.

**Evidence**: `docker-compose.yml:1-28`; confidence High.

---

### Business Rule: BR-19 - Credential consistency across configuration sources

**Overview**:
Three configuration surfaces must agree for the stack to work: the Compose `MYSQL_*` variables used for provisioning (`docker-compose.yml:7-10`), the `DATABASE_URL` used by the application, seed, and tests (`.env.example:5`), and the `SHADOW_DATABASE_URL` used by Prisma migrations (`.env.example:6`). No code validates that they agree.

**Detailed description**:
The Compose variables define what the server creates on first boot. `DATABASE_URL` defines how the application authenticates and which schema it selects. `SHADOW_DATABASE_URL` defines how Prisma authenticates for schema diffing. The defaults are internally consistent: `oms_user` / `oms_password` / `oms` in both Compose and `DATABASE_URL`, and root / `root_password` / `oms_shadow` in the shadow URL. That consistency is convention, not enforcement. The application's environment schema validates only that `DATABASE_URL` is a non-empty string; it does not parse the URL, compare the user or database with the Compose variables, or check reachability.

The use case is a single `.env` file that configures both Docker Compose and the Node process. Because Compose reads `.env` automatically and Node reads it through `--env-file=.env`, a single file can serve both. The limitation is that the two readers have different precedence rules. Compose also reads the shell environment, which can override `.env`; the Node process may also receive shell variables. A variable set in the shell but not in `.env` can therefore affect Compose without affecting a Node process that only reads `.env`, or vice versa. The repository does not document a canonical precedence order.

The limitation extends to password encoding. `MYSQL_PASSWORD` is a raw value, while the password in `DATABASE_URL` is part of a URL and must be percent-encoded if it contains reserved characters such as `@`, `:`, `/`, or `#`. The example uses alphanumeric passwords, so the issue is latent. Changing to a password with special characters without encoding it in both URLs produces an authentication or parse failure. Confidence is high for the absence of validation and the three sources; the exact precedence is environment-dependent.

**Rule workflow**:
1. Compose resolves `MYSQL_*` from the shell and `.env`.
2. The entrypoint provisions the server with those values on first boot.
3. Node reads `.env` through `--env-file` and validates `DATABASE_URL`.
4. Prisma connects with the URL credentials and database path.
5. Prisma CLI connects with `SHADOW_DATABASE_URL` for migrations.
6. Any mismatch surfaces as an authentication, unknown-database, or migration error.

**Evidence**: `docker-compose.yml:7-10`; `.env.example:5-6,11-14`; `src/config/env.ts:3-10`; confidence High.

---

### Business Rule: BR-20 - Data durability and recovery boundary

**Overview**:
The component defines one MySQL instance, one local volume, and no replication, failover, backup, point-in-time recovery, binary-log retention policy, or snapshot job (`docker-compose.yml:2-28`). The restart policy supervises the process, not the data.

**Detailed description**:
InnoDB provides crash consistency for committed transactions under its default durability settings, but it does not provide redundancy. The data directory lives on a single host filesystem behind the local volume driver. If the host filesystem is lost, corrupted, or the volume is deleted, the data is lost. There is no second copy, no replica, no automated dump, and no restore procedure in the repository. MySQL 8.0 enables binary logging by default, but the Compose file does not configure `binlog_expire_logs_seconds`, provide a base backup, or export binary logs off the volume. Binary logs stored only in the same volume do not provide a recoverable point-in-time backup if that volume is lost.

The use case is local development where reproducibility comes from migrations and seed data, not from backup. A developer can destroy the volume and rebuild the schema and demo data. The limitation is that the same topology is the only topology defined in the repository. There is no separate production Compose file, Kubernetes manifest, managed-database configuration, or backup script. The dependency audit and the architectural report both classify the single-instance MySQL as a high risk for availability and data loss.

The recovery boundary is explicit in the teardown behavior: `docker compose down` preserves data, `docker compose down -v` destroys it. There is no documented recovery objective, backup schedule, retention period, or restore test. The component cannot recover from a lost volume without re-running migrations and seed, and it cannot recover user-generated data at all. Confidence is high because the absence of backup and replication configuration is directly observable.

**Rule workflow**:
1. `mysqld` writes committed transactions to InnoDB files in the volume.
2. The volume driver stores those files on the host filesystem.
3. Container or daemon restart reuses the same files.
4. Volume deletion or host filesystem loss removes the only copy.
5. No automated backup or restore process intervenes.

**Evidence**: `docker-compose.yml:2-28`; confidence High.

---

### Business Rule: BR-21 - Transport and data-at-rest security boundary

**Overview**:
The component has no TLS configuration for client connections, no encryption-at-rest configuration, and no secret manager. The database port is published to all host interfaces. Credentials are present in environment variables, in the healthcheck command, and in the committed `.env.example`.

**Detailed description**:
`DATABASE_URL` and `SHADOW_DATABASE_URL` contain no `ssl`, `sslmode`, or certificate parameters (`.env.example:5-6`). The Compose command does not set `--require_secure_transport=ON` or any TLS certificate options. MySQL therefore accepts plaintext connections on the published port by default. An observer with access to the host network can read queries, results, and authentication exchanges. Data files in the volume are not encrypted by any declared option; the local volume driver does not add encryption. The only access control is the MySQL account system and the host firewall, which is outside the repository.

The use case is local development, where plaintext connections on a loopback or developer workstation are common. The limitation is that the same configuration is the only one defined. The port mapping uses the wildcard host address, so the exposure is not limited to loopback. The root password is visible in the healthcheck command inside the container configuration and through Docker inspection. The application and root passwords are visible in `.env.example` and in Compose fallbacks. The `.env` file is gitignored (`.gitignore:4`), but the example values are committed and are also the fallbacks.

The component has no secret rotation mechanism, no vault integration, and no file-based secret mount. The first-initialization rule means that rotating a password in the environment does not rotate it in the database. The combination of weak defaults, plaintext transport, wildcard port publication, and no at-rest encryption defines a development-only security boundary. Confidence is high because all of these absences are directly observable in the files.

**Rule workflow**:
1. The client opens a plaintext TCP connection to the published port.
2. The server negotiates the authentication plugin and exchanges credentials.
3. Queries and results travel in plaintext.
4. InnoDB writes unencrypted data files to the volume.
5. Access depends on MySQL grants and host firewall rules outside the repository.

**Evidence**: `docker-compose.yml:7-16`; `.env.example:5-6`; `.gitignore:4`; confidence High.

---

### Business Rule: BR-22 - Resource governance and server tuning

**Overview**:
The Compose service declares no CPU limit, memory limit, PID limit, ulimit, shared-memory size, logging driver, log rotation, or MySQL performance tuning. It relies entirely on image defaults for `max_connections`, `innodb_buffer_pool_size`, `sql_mode`, and other server variables.

**Detailed description**:
The only `mysqld` flags are the authentication plugin, character set, and collation (`docker-compose.yml:21-24`). There is no `deploy.resources`, `mem_limit`, `cpus`, `pids_limit`, `ulimits`, `shm_size`, `read_only`, `security_opt`, `user`, or `logging` block. The container can consume host CPU and memory without a Compose-level ceiling. MySQL's own defaults apply, including a finite connection limit and a default InnoDB buffer pool size that may be small relative to a developer workstation or large relative to a CI runner. The Compose file also does not pin `sql_mode`, so the effective SQL mode follows the image's compiled default, which for MySQL 8.0 includes strict transaction and date handling.

The use case is a simple local database that starts without tuning. The limitation is unpredictable performance and resource contention. A heavy query, a large import, or a connection storm can consume host resources without a container limit. Docker's default `json-file` logging driver has no `max-size` or `max-file` in this file, so container logs can grow until the host disk is full. The absence of an explicit `stop_grace_period` means Docker uses its default grace period when stopping the container; InnoDB crash recovery handles an unclean stop, but the shutdown is not tuned for large write workloads.

The rule is not inherently wrong for local development, but it is undocumented. The repository does not state which image defaults are assumed, and patch updates can change defaults. Confidence is high for the absence of limits and tuning, medium for the exact image default values.

**Rule workflow**:
1. Docker starts the container without resource limits.
2. `mysqld` starts with the three declared flags and image defaults for everything else.
3. The server uses host CPU, memory, and disk according to its defaults and workload.
4. Docker logs use the default driver and default rotation settings.
5. No Compose-level signal adjusts behavior for CI, production, or constrained hosts.

**Evidence**: `docker-compose.yml:1-28`; confidence High for absence, Medium for default values.

---

### Business Rule: BR-23 - Time semantics

**Overview**:
The Compose command does not set `--default-time-zone`. The migration uses `DATETIME(3)` columns and `CURRENT_TIMESTAMP(3)` defaults. MySQL `DATETIME` stores a date and time without a time zone, and the session time zone defaults to `SYSTEM`, which follows the container operating system time zone.

**Detailed description**:
The schema's `createdAt`, `updatedAt`, `changedAt`, and other temporal columns are `DATETIME(3)`. They store millisecond precision but no offset. The defaults `CURRENT_TIMESTAMP(3)` are evaluated in the session time zone. Prisma maps JavaScript `Date` values to `DATETIME` and typically serializes them in UTC, but the connection's session time zone and the server's `SYSTEM` time zone determine how defaults and raw SQL comparisons behave. The repository does not set a time zone in `DATABASE_URL` and does not pass `--default-time-zone`; the official image may load time zone tables for `CONVERT_TZ`, but that does not set the session default.

The use case is audit timestamps and ordering. The application mostly compares times relatively and does not perform time-zone conversions. The limitation is portability and reproducibility. The official image's container time zone is typically UTC, but that is an image default, not a repository declaration. If the container time zone changes, or if the database is moved to a server with a different `SYSTEM` time zone, `CURRENT_TIMESTAMP(3)` defaults and any server-side date functions shift. Existing `DATETIME` values do not shift because they have no zone information, which can create mixed semantics between old and new rows.

The component has no test that asserts time zone behavior, and no documentation states the intended zone. The analysis therefore records this as an implicit rule with medium confidence. The `DATETIME(3)` choice and the absence of `TIMESTAMP` or `DATETIME` with explicit zone conversion are observable; the intended operational time zone is not.

**Rule workflow**:
1. `mysqld` starts without a default time zone override.
2. The session time zone resolves to `SYSTEM`.
3. `CURRENT_TIMESTAMP(3)` defaults use the container system clock and zone.
4. Prisma writes `DATETIME(3)` values without an offset.
5. Readers interpret the stored values according to application and driver conventions, not a database-enforced zone.

**Evidence**: `docker-compose.yml:21-24`; `prisma/migrations/20260519182739_init/migration.sql:8-9,23-24,40-41,60-61,79,92`; confidence Medium.

---

## 4. Component Structure

### 4.1 Declared Compose structure

```text
docker-compose.yml
├── services
│   └── mysql
│       ├── image: mysql:8.0                         # line 3
│       ├── container_name: oms-mysql                # line 4
│       ├── restart: unless-stopped                  # line 5
│       ├── environment                              # lines 6-10
│       │   ├── MYSQL_ROOT_PASSWORD: ${...:-root_password}
│       │   ├── MYSQL_DATABASE: ${...:-oms}
│       │   ├── MYSQL_USER: ${...:-oms_user}
│       │   └── MYSQL_PASSWORD: ${...:-oms_password}
│       ├── ports                                    # lines 11-12
│       │   └── '3306:3306'
│       ├── volumes                                  # lines 13-14
│       │   └── oms_mysql_data:/var/lib/mysql
│       ├── healthcheck                              # lines 15-20
│       │   ├── test: mysqladmin ping -h localhost -uroot -p...
│       │   ├── interval: 5s
│       │   ├── timeout: 5s
│       │   ├── retries: 20
│       │   └── start_period: 30s
│       └── command                                  # lines 21-24
│           ├── --default-authentication-plugin=mysql_native_password
│           ├── --character-set-server=utf8mb4
│           └── --collation-server=utf8mb4_unicode_ci
└── volumes
    └── oms_mysql_data                               # lines 26-28
        └── driver: local
```

### 4.2 Related repository files

```text
project-root/
├── docker-compose.yml                              # The component definition
├── .env.example                                    # Database credentials and connection URLs, lines 5-6, 11-14
├── .gitignore                                      # Excludes .env, line 4
├── package.json                                    # db:migrate, db:reset, db:seed, test scripts, lines 14-17
├── prisma/
│   ├── schema.prisma                               # MySQL datasource, DATABASE_URL, SHADOW_DATABASE_URL, lines 5-9
│   ├── seed.ts                                     # Direct Prisma writes, lines 1-4, 181-187
│   └── migrations/
│       ├── migration_lock.toml                     # Provider fixed to mysql, line 3
│       └── 20260519182739_init/
│           └── migration.sql                       # Seven tables, enums, JSON, indexes, FKs, lines 1-125
├── src/
│   └── config/
│       ├── env.ts                                  # DATABASE_URL validation, lines 3-10
│       └── database.ts                             # PrismaClient construction, lines 4-10
└── tests/
    ├── setup.ts                                    # Connect, truncate all tables, disconnect, lines 1-20
    ├── auth.test.ts                                # 7 integration tests
    ├── orders.test.ts                              # 8 integration tests
    └── helpers/
        └── factories.ts                            # Direct Prisma row creation
```

### 4.3 Configuration matrix

| Setting | Declared value | Effective default | Consumer | Evidence |
|---------|----------------|-------------------|----------|----------|
| Image | `mysql:8.0` | Current 8.0.x patch resolved at pull | Docker | `docker-compose.yml:3` |
| Container name | `oms-mysql` | None; literal name | Docker, operators | `docker-compose.yml:4` |
| Restart policy | `unless-stopped` | None | Docker | `docker-compose.yml:5` |
| Root password | `${MYSQL_ROOT_PASSWORD:-root_password}` | `root_password` | Entrypoint, healthcheck, shadow URL | `docker-compose.yml:7,16`; `.env.example:6,11` |
| Database | `${MYSQL_DATABASE:-oms}` | `oms` | Entrypoint, `DATABASE_URL` path | `docker-compose.yml:8`; `.env.example:5,12` |
| Application user | `${MYSQL_USER:-oms_user}` | `oms_user` | Entrypoint, `DATABASE_URL` user | `docker-compose.yml:9`; `.env.example:5,13` |
| Application password | `${MYSQL_PASSWORD:-oms_password}` | `oms_password` | Entrypoint, `DATABASE_URL` password | `docker-compose.yml:10`; `.env.example:5,14` |
| Host port | `3306:3306` | All host interfaces | Host clients | `docker-compose.yml:11-12` |
| Data volume | `oms_mysql_data` | Project-prefixed name | `mysqld` data directory | `docker-compose.yml:13-14,26-28` |
| Health command | `mysqladmin ping` | Root over localhost | Docker | `docker-compose.yml:16` |
| Health timing | 5s / 5s / 20 / 30s | None | Docker | `docker-compose.yml:17-20` |
| Auth plugin | `mysql_native_password` | Image default changed by flag | Server, clients | `docker-compose.yml:22` |
| Character set | `utf8mb4` | Image default changed by flag | Server, tables | `docker-compose.yml:23` |
| Collation | `utf8mb4_unicode_ci` | Image default changed by flag | Server, tables | `docker-compose.yml:24` |
| Time zone | Not declared | `SYSTEM` | Server session | Absence from `docker-compose.yml:21-24` |
| Schema bootstrap | Not declared | No tables created | Operator/Prisma | Absence of init mount; `package.json:14-16` |

---

## 5. Dependency Analysis

### 5.1 Internal dependency chains

```text
docker-compose.yml
├── reads shell environment
├── reads project .env for interpolation (optional)
├── pulls mysql:8.0
├── creates/reuses oms_mysql_data volume
├── binds host port 3306
└── starts mysqld with command flags
    ├── consumed by host Node process through DATABASE_URL
    │   ├── src/config/env.ts validates DATABASE_URL
    │   ├── src/config/database.ts constructs PrismaClient
    │   └── src/modules/** repositories execute queries
    ├── consumed by Prisma CLI through DATABASE_URL and SHADOW_DATABASE_URL
    │   ├── prisma/schema.prisma declares provider and datasource
    │   ├── prisma/migrations/migration_lock.toml fixes provider
    │   └── prisma/migrations/20260519182739_init/migration.sql applies DDL
    ├── consumed by prisma/seed.ts through DATABASE_URL
    └── consumed by tests through DATABASE_URL
        ├── tests/setup.ts connects and truncates
        ├── tests/helpers/factories.ts writes rows
        ├── tests/auth.test.ts exercises user/auth tables
        └── tests/orders.test.ts exercises order/product/customer tables
```

Dependency directions:

- `docker-compose.yml` -> host environment and `.env` for interpolation.
- `docker-compose.yml` -> Docker Hub registry for `mysql:8.0`.
- `docker-compose.yml` -> `oms_mysql_data` volume for persistence.
- `docker-compose.yml` -> host TCP port `3306` for publication.
- `src/config/env.ts` -> `DATABASE_URL` string contract.
- `src/config/database.ts` -> Prisma Client -> MySQL wire protocol -> `mysql` service.
- `prisma/schema.prisma` -> MySQL provider and connection URLs.
- `package.json` scripts -> Prisma CLI -> MySQL.
- `tests/setup.ts` -> Prisma Client -> MySQL.
- `prisma/seed.ts` -> Prisma Client -> MySQL.

### 5.2 Internal dependencies

| Dependency | Type | Direction | Purpose | Coupling |
|------------|------|-----------|---------|----------|
| `docker-compose.yml` | Infrastructure definition | Container -> host | Declares image, env, ports, volume, health, command | Hard, operational |
| `.env.example` | Configuration template | Operator -> Compose and Node | Documents `MYSQL_*`, `DATABASE_URL`, `SHADOW_DATABASE_URL` | Soft, documentary |
| `.env` (not committed) | Runtime configuration | Host -> Compose and Node | Supplies real values; gitignored | Hard, runtime |
| `oms_mysql_data` volume | Stateful storage | Container -> host filesystem | Stores `/var/lib/mysql` | Hard, persistent |
| `prisma/schema.prisma` | Schema contract | Application -> MySQL | Declares provider, datasource, and models | Hard, compile-time and runtime |
| `prisma/migrations/20260519182739_init/migration.sql` | DDL contract | Prisma CLI -> MySQL | Creates tables, indexes, foreign keys | Hard, one-time |
| `prisma/migrations/migration_lock.toml` | Provider lock | Prisma CLI -> migration engine | Confirms MySQL provider | Hard, tooling |
| `src/config/env.ts` | Runtime configuration validation | Node -> environment | Requires non-empty `DATABASE_URL` | Hard, startup |
| `src/config/database.ts` | Client construction | Node -> Prisma -> MySQL | Creates shared `PrismaClient` | Hard, runtime |
| `tests/setup.ts` | Test harness | Tests -> MySQL | Connects and truncates all tables | Hard, test-time |
| `prisma/seed.ts` | Seed script | Operator -> MySQL | Deletes and inserts demo data | Hard, operational |
| `package.json` scripts | Operational interface | Operator -> Prisma CLI | Exposes migrate, reset, seed, test | Hard, operational |

### 5.3 External dependencies

| Dependency | Version / reference | Type | Purpose | Direction | Error behavior |
|------------|---------------------|------|---------|-----------|----------------|
| Docker Engine | Not declared | Container runtime | Runs the MySQL container, volume, port mapping | Host -> component | Container fails to start on port conflict or image error |
| Docker Compose | Compose Spec; no `version` key | Orchestration | Reads YAML, interpolates variables, creates network and volume | Host -> component | Parse error or unresolved variable prevents startup |
| Docker Hub / registry | `mysql:8.0` tag | Image source | Supplies the MySQL server binary and entrypoint | Component -> registry | Pull failure prevents startup; mutable tag can change patch |
| Official MySQL image entrypoint | Part of `mysql:8.0` | Initialization | Sets root password, creates database and user, grants privileges | Image -> server | First-boot errors can leave an unusable data directory |
| MySQL Server | 8.0.x resolved from tag | Database engine | SQL execution, storage, authentication, transactions | Component internal | Startup failure on invalid flags or incompatible data |
| `mysqladmin` | Part of image | Health probe | Sends ping to localhost | Docker -> server | Exit status controls health state |
| Host filesystem | Local driver | Storage | Stores volume data | Volume -> host | Host disk failure or full disk affects availability |
| Host TCP stack | Port `3306` | Network | Publishes the database to host clients | Host -> component | Port conflict or firewall blocks access |
| Prisma Client | 5.22.0 (`package.json:26`) | ORM client | Connects and executes queries | Application -> MySQL | Lazy connect; query failures surface per request |
| Prisma CLI | 5.22.0 (`package.json:48`) | Migration tooling | Applies migrations and manages shadow database | Operator -> MySQL | Fails on connection, permission, or shadow database errors |
| Node.js | `>=20` (`package.json:8`) | Host runtime | Runs application, Prisma CLI, seed, tests | Host process | `--env-file` requires Node 20.6+ in practice |
| `bcrypt` | 5.1.1 (`package.json:27`) | Application dependency | Hashes seed and test passwords before storage | In-process | Not a database dependency, but seed/tests write hashes |
| `vitest` / `supertest` | 2.1.4 / 7.0.0 | Test tooling | Drives the application against the live database | Tests -> MySQL | Connection or schema failure fails the suite |

### 5.4 Dependency observations

- The component has exactly one external stateful dependency at runtime: MySQL. All other runtime dependencies are in-process libraries.
- The Compose service depends on network access to a container registry at first pull; after the image is cached, startup is local.
- The application does not depend on Compose at build time. It depends on the published host port and the connection string at runtime.
- The migration tooling has a distinct dependency on `SHADOW_DATABASE_URL`, which is not used by the application process.
- The seed script and tests use the same `DATABASE_URL` and the same application account as the runtime, so they share the same failure modes.
- The repository does not declare a Docker Compose version. Modern Compose treats the file as a Compose Spec document; older `docker-compose` v1 may interpret it differently.
- The image tag is the only version constraint for the database engine. There is no lock file, checksum, or digest in the repository.

---

## 6. Afferent and Efferent Coupling

For an infrastructure component, coupling is counted as the number of declared relationships rather than the number of source-level symbols. The units below are the Compose service, the volume, the healthcheck probe, and the configuration contracts that connect the infrastructure to the application. The counts are relationship counts, not compiled class metrics.

| Component | Afferent Coupling | Efferent Coupling | Critical | Explanation |
|-----------|-------------------|-------------------|----------|-------------|
| `mysql` service | 7 | 8 | High | Inbound: host operator, Node runtime, Prisma CLI, seed script, test harness, Docker health engine, future worker. Outbound: image registry, env vars, volume, host port, health command, three server flags, default network. |
| `oms_mysql_data` volume | 1 | 1 | High | Inbound: `mysql` service mount. Outbound: host filesystem through the local driver. |
| Healthcheck probe | 1 | 1 | Medium | Inbound: Docker engine. Outbound: `mysqld` through `mysqladmin`. |
| `.env.example` credential contract | 6 | 0 | High | Inbound readers: Compose, Node `--env-file`, Prisma CLI, seed script, tests, operator. It depends on nothing at runtime. |
| `DATABASE_URL` contract | 5 | 1 | High | Inbound: `env.ts`, `database.ts`, Prisma CLI, seed, tests. Outbound: MySQL endpoint. |
| `SHADOW_DATABASE_URL` contract | 2 | 1 | Medium | Inbound: Prisma CLI, operator. Outbound: MySQL endpoint with root credentials. |
| `oms` schema | 7 | 0 | High | Inbound tables: `users`, `customers`, `products`, `orders`, `order_items`, `order_status_history`, `order_number_sequence`. Outbound: none declared. |
| Migration DDL | 2 | 0 | Medium | Inbound: Prisma CLI, `_prisma_migrations`. Outbound: none; it is a one-way schema definition. |
| Container name `oms-mysql` | 4 | 0 | Medium | Inbound references: Compose, operator commands, documentation, potential peer containers. Outbound: none. |
| Host port `3306` | 5 | 1 | High | Inbound: Node runtime, Prisma CLI, seed, tests, external clients. Outbound: container port mapping. |

Interpretation:

- The `mysql` service has high afferent coupling because every data path in the project terminates there. It is the central stateful node.
- The service also has high efferent coupling because it depends on an external image, host resources, environment variables, and a volume. Failures in any of those dependencies affect the whole system.
- The volume has low symbolic coupling but high criticality: one inbound mount and one outbound host filesystem relationship are enough to make it the single point of data loss.
- The configuration contracts have high afferent coupling because they are read by multiple tools with different precedence rules. This is a configuration-coupling risk, not a code-coupling risk.
- The healthcheck has low coupling and medium criticality. It observes the server but is not observed by the application or the restart policy.
- The `oms` schema is coupled to seven tables and to the Prisma models. The schema is the true interface between the application and the container.

---

## 8. Integration Points

Section 7 (Endpoints) is omitted because the MySQL Container exposes no REST, GraphQL, gRPC, or other application API endpoint. Its only listener is the MySQL wire-protocol port. The integration points below describe that listener and the surrounding operational contracts.

| Integration | Type | Purpose | Protocol | Data Format | Error Handling | Direction | Evidence |
|-------------|------|---------|----------|-------------|----------------|-----------|----------|
| Host Node application -> MySQL | Runtime data access | Reads and writes users, customers, products, orders, items, history, sequence | MySQL wire protocol over TCP | Relational rows mapped by Prisma; `CHAR(36)`, `INT`, `TEXT`, `JSON`, `ENUM`, `DATETIME(3)` | Lazy connect; failures surface at query time; Prisma `P2002` -> 409, `P2025` -> 404, others -> 500 | Host -> container | `src/config/database.ts:4-10`; `.env.example:5` |
| Prisma CLI migrate -> MySQL | Schema migration | Applies pending migrations to `oms` | MySQL wire protocol over TCP | DDL from `migration.sql` | Fail-fast on connection, permission, or SQL error; migration transaction where supported | Host -> container | `package.json:14`; `prisma/migrations/20260519182739_init/migration.sql` |
| Prisma CLI shadow -> MySQL | Migration diffing | Creates, uses, and drops `oms_shadow` | MySQL wire protocol over TCP | DDL and schema diff metadata | Fail-fast if root cannot connect or lacks `CREATE DATABASE`/`DROP DATABASE` | Host -> container | `.env.example:6`; `prisma/schema.prisma:5-9` |
| Seed script -> MySQL | Data population | Deletes and inserts demo users, customers, products, orders, history | MySQL wire protocol over TCP | Prisma create/delete operations with bcrypt hashes and JSON addresses | Unhandled rejection; process exits with code 1 | Host -> container | `prisma/seed.ts:1-4,178-335`; `package.json:16` |
| Test harness -> MySQL | Integration testing | Connects, truncates all application tables before each test, creates fixtures, disconnects | MySQL wire protocol over TCP | DML via Prisma and HTTP-driven application operations | Connection failure in `beforeAll`; test failures; no retry or isolation guard | Host -> container | `tests/setup.ts:1-20`; `tests/helpers/factories.ts`; `vitest.config.ts:4-18` |
| Docker health engine -> `mysqld` | Health probe | Determines container health status | Local TCP or socket; `mysqladmin` CLI | Command exit status | Marks healthy after a zero exit; marks unhealthy after 20 failures; does not stop or restart container | Docker -> container | `docker-compose.yml:15-20` |
| Host operator / MySQL client -> MySQL | Manual administration | Inspect data, run ad-hoc SQL, reset passwords, debug migrations | MySQL wire protocol over TCP | SQL statements | Client-dependent; access denied, unknown database, connection refused | Host -> container | `docker-compose.yml:11-12`; `.env.example:5-6` |
| Docker registry -> Docker Engine | Image supply | Pulls `mysql:8.0` | HTTPS / OCI registry protocol | OCI image manifest and layers | Pull failure aborts startup; mutable tag can resolve to a new patch | Registry -> host | `docker-compose.yml:3` |
| `mysql` service -> `oms_mysql_data` | Persistence | Stores the InnoDB data directory | Local filesystem mount | InnoDB files, redo/undo logs, binary logs (enabled by default in MySQL 8.0) | I/O error, full disk, or permission error can crash or corrupt the server | Container -> host filesystem | `docker-compose.yml:13-14,26-28` |
| Future webhook worker -> MySQL (planned) | Future runtime data access | Poll an outbox table and update delivery state | MySQL wire protocol over TCP | Relational rows; outbox and dead-letter tables not yet present | Not implemented; no error contract exists | Future container -> container or host -> container | `TRANSCRICAO.md`; architectural report integration table |

Integration observations:

- Every data path is point-to-point over TCP. There is no connection pooler, proxy, service mesh, or load balancer between the host and MySQL.
- The application and the migration tooling use different credentials. The application uses `oms_user`; the migration shadow path uses root.
- The healthcheck is the only automated consumer of the server's operational state, and it is not connected to application readiness.
- The Compose service does not participate in application-level tracing. There is no MySQL exporter, metrics endpoint, slow-query log configuration, or log shipping.
- The only declared inbound host surface is TCP `3306`. There is no declared outbound network path from the container; image pulls happen at the Docker layer, not inside the container.
- The future webhook worker is the only planned integration that would add a second database client. It would share the same server and schema but run in a separate process.

---

## 9. Design Patterns & Architecture

### 9.1 Identified patterns

| Pattern | Implementation | Location | Purpose | Assessment |
|---------|----------------|----------|---------|------------|
| Infrastructure as Code | Declarative Docker Compose service | `docker-compose.yml:1-28` | Reproducible local database | Present and minimal |
| Named Volume Persistence | `oms_mysql_data:/var/lib/mysql` | `docker-compose.yml:13-14,26-28` | Preserve data across container lifecycle | Present |
| Container Healthcheck | `mysqladmin ping` with interval, timeout, retries, start period | `docker-compose.yml:15-20` | Liveness signal for operators | Present but not readiness |
| Environment Variable Configuration | `MYSQL_*` and URLs supplied through environment | `docker-compose.yml:6-10`; `.env.example:5-6,11-14` | Separate configuration from image | Present |
| Fallback Defaults | `${VAR:-default}` syntax | `docker-compose.yml:7-10` | Zero-config startup | Present, but duplicates committed secrets |
| Singleton Container | Explicit `container_name: oms-mysql` | `docker-compose.yml:4` | Predictable name for operators | Present, limits parallel stacks |
| Supervisor / Restart Policy | `restart: unless-stopped` | `docker-compose.yml:5` | Restart after crash or daemon reboot | Present, no health-based restart |
| First-Boot Initialization | Official image entrypoint | External to repository; driven by `docker-compose.yml:6-10` | Create database, user, grants on empty volume | Present, one-time only |
| Out-of-Band Schema Migration | Prisma migrations, no init SQL mount | `package.json:14-16`; `prisma/migrations/` | Controlled schema evolution | Present, requires manual step |
| Shadow Database | `SHADOW_DATABASE_URL` with root | `.env.example:6`; `prisma/schema.prisma:8` | Schema diffing for `migrate dev` | Present, not provisioned by Compose |
| Default Bridge Network | No `networks:` key | `docker-compose.yml:1-28` | Container-to-container connectivity | Present, no custom isolation |
| Mutable Tag | `image: mysql:8.0` | `docker-compose.yml:3` | Convenient patch updates | Anti-pattern for reproducibility |
| Secret in Command Line | Root password in healthcheck `-p` argument | `docker-compose.yml:16` | Authenticate the probe | Anti-pattern; visible in container metadata |
| Password in URL | Credentials embedded in `DATABASE_URL` | `.env.example:5-6` | Single-string connection configuration | Common but exposes secrets in environment |
| No Application Containerization | No `Dockerfile` or app service | Repository root | Host-run application | Deliberate or incomplete; deployment gap |

### 9.2 Architectural decisions observed

- **Local-only topology.** The Compose file defines exactly one service and one volume. There is no application service, reverse proxy, cache, queue, or replica. The application is expected to run on the host.
- **Database as the only stateful tier.** All persistence is delegated to MySQL. The container filesystem outside `/var/lib/mysql` is disposable.
- **Manual schema lifecycle.** The database starts empty and is shaped by Prisma migrations. The container does not know about the application schema.
- **Development-first credentials.** Defaults are designed for convenience, not secrecy. The same values appear in the example file and in the Compose fallbacks.
- **Liveness-only observability.** The healthcheck observes process responsiveness. There is no readiness probe, metrics endpoint, or log aggregation contract.
- **Single point of failure.** One container, one volume, one host port, one database. There is no redundancy at any layer.
- **Version flexibility over reproducibility.** The image tag floats within 8.0.x. The repository does not pin a digest or a patch version.
- **Compatibility over modern authentication.** The native-password flag favors older clients over the stronger MySQL 8.0 default.

### 9.3 Quality attribute assessment

| Quality attribute | Rating | Evidence | Consequence |
|-------------------|--------|----------|-------------|
| Availability | Low | One container, one host, `restart: unless-stopped` only | Host or container failure stops all data access; no failover |
| Reliability | Medium | InnoDB crash consistency, healthcheck, named volume | Survives process restarts, but not volume or host loss |
| Recoverability | Low | No backup, snapshot, replication, or PITR | Lost volume means permanent data loss |
| Security | Low | Default credentials, wildcard port, no TLS, no at-rest encryption, legacy auth plugin | Credential reuse, plaintext traffic, broad exposure |
| Maintainability | Medium | 28-line Compose file, single service | Easy to read, but relies on undocumented image behavior |
| Portability | Medium | Docker-based, but local volume and fixed host port | Works across Docker hosts, but data does not move with the repository |
| Observability | Low | Healthcheck only; no metrics, logs config, or tracing | Difficult to diagnose performance or failure causes |
| Performance | Unknown | No tuning flags or resource limits | Depends entirely on image defaults and host resources |
| Testability | Low | No infrastructure tests; tests depend on an external live database | Container behavior is unverified; test suite is not hermetic |
| Operability | Medium | Simple start/stop commands; manual migration and seed steps | Easy for a single developer, error-prone for teams and CI |
| Reproducibility | Low | Mutable image tag, no digest, no platform pin, image defaults | Different hosts can run different server versions and defaults |
| Isolation | Low | Default bridge network, wildcard host port | Database is reachable from any host interface and any project container on the network |

---

## 10. Technical Debt & Risks

| ID | Risk Level | Component Area | Issue | Impact | Evidence |
|----|------------|----------------|-------|--------|----------|
| TD-01 | High | Image lifecycle | `mysql:8.0` is a mutable tag, not a digest; MySQL 8.0 is in Oracle Sustaining Support as of 2026-04-21 | Different hosts can run different patches; security fixes may stop while the tag still resolves | `docker-compose.yml:3`; dependency audit report |
| TD-02 | High | Credentials | Root password `root_password` and application password `oms_password` are committed in `.env.example` and duplicated as Compose fallbacks | Credential reuse and unauthorized access in any exposed environment | `docker-compose.yml:7,10`; `.env.example:11,14` |
| TD-03 | High | Network exposure | `3306:3306` publishes the database on all host interfaces | Database is reachable from the LAN or internet if the host firewall permits | `docker-compose.yml:11-12` |
| TD-04 | High | Transport security | No TLS parameters in `DATABASE_URL`/`SHADOW_DATABASE_URL`; no `--require_secure_transport` | Queries, results, and authentication travel in plaintext | `.env.example:5-6`; `docker-compose.yml:21-24` |
| TD-05 | High | Durability | No replication, failover, backup, snapshot, or point-in-time recovery | Volume or host loss destroys all data; no recovery objective is defined | `docker-compose.yml:2-28` |
| TD-06 | High | Authentication | `mysql_native_password` is forced and is deprecated in later 8.0 patches and removed in 8.4 | Weaker password protocol; future image upgrade can fail at startup | `docker-compose.yml:22`; dependency audit report |
| TD-07 | High | Schema bootstrap | Compose creates an empty database but no tables; no init SQL mount; no `migrate deploy` script | A running container can be unusable until a manual migration is run | `docker-compose.yml:1-28`; `package.json:14-16` |
| TD-08 | High | Test safety | `tests/setup.ts:8-16` deletes all application rows from whatever `DATABASE_URL` points to, with no environment guard | Running tests against a development or shared database destroys data | `tests/setup.ts:8-16`; `.env.example:5` |
| TD-09 | High | Shadow database | `SHADOW_DATABASE_URL` uses root, points to an unprovisioned `oms_shadow`, and is not validated by the application schema; root host scope is not configured | `prisma migrate dev` can fail on permissions or host matching; root credentials are used for a development-only tool | `.env.example:6`; `src/config/env.ts:3-10`; `docker-compose.yml` absence of `MYSQL_ROOT_HOST` |
| TD-10 | Medium | Healthcheck | `mysqladmin ping` is a liveness probe; access-denied responses can still report success | Credential or readiness failures can appear healthy | `docker-compose.yml:15-20` |
| TD-11 | Medium | Secret exposure | Root password is embedded in the healthcheck command and visible through `docker inspect`/`docker compose config` | Anyone with Docker API access can read the root credential | `docker-compose.yml:16` |
| TD-12 | Medium | Credential rotation | First-initialization-only provisioning means environment changes do not alter existing users or passwords | Password rotation appears to succeed but the server keeps the old credential | `docker-compose.yml:6-10`; official image behavior |
| TD-13 | Medium | Configuration consistency | Compose variables, `DATABASE_URL`, and `SHADOW_DATABASE_URL` are independent strings with no cross-validation | Changing one value without the others produces authentication or unknown-database errors | `docker-compose.yml:7-10`; `.env.example:5-6,11-14` |
| TD-14 | Medium | Container identity | Fixed `container_name: oms-mysql` is global to the Docker host | Two checkouts or two stacks cannot run simultaneously on one host | `docker-compose.yml:4` |
| TD-15 | Medium | Host port | Fixed `3306:3306` mapping with no override file or parameterization | Conflicts with a local MySQL server or another container; no easy parallel environment | `docker-compose.yml:11-12` |
| TD-16 | Medium | Resource governance | No CPU, memory, PID, ulimit, shared-memory, or logging limits; no MySQL tuning flags | A heavy workload can exhaust host resources; container logs can grow without bound | `docker-compose.yml:1-28` |
| TD-17 | Medium | Time semantics | No `--default-time-zone`; `DATETIME(3)` columns and `CURRENT_TIMESTAMP(3)` defaults depend on the container system zone | Timestamps can shift if the container or host zone changes; mixed semantics across migrations | `docker-compose.yml:21-24`; `prisma/migrations/20260519182739_init/migration.sql` |
| TD-18 | Medium | Collation semantics | `utf8mb4_unicode_ci` is an older Unicode collation and is case-insensitive for all string comparisons and unique indexes | E-mail and name identity semantics differ from MySQL 8.0 defaults; behavior is environment-dependent if the collation changes | `docker-compose.yml:24`; migration DDL |
| TD-19 | Medium | SQL mode | `sql_mode` is not pinned; the effective mode follows the image default | Patch updates can change strictness and date-handling behavior | Absence from `docker-compose.yml:21-24` |
| TD-20 | Medium | Logging | No `logging` driver or rotation configuration; no slow-query log, error-log destination, or metrics exporter | Disk exhaustion from logs; limited diagnosis of slow queries and failures | `docker-compose.yml:1-28` |
| TD-21 | Medium | Shutdown | No explicit `stop_grace_period`; Docker default applies | Large write workloads may be interrupted; InnoDB recovery handles crash consistency but shutdown is not tuned | Absence from `docker-compose.yml:1-28` |
| TD-22 | Medium | Reproducibility | No `platform` pin; ARM and AMD64 hosts can resolve different image manifests | Different CPU architectures may receive different binaries and behavior | `docker-compose.yml:3` |
| TD-23 | Medium | Migration operations | No `prisma migrate deploy` script; only interactive `migrate dev` and destructive `migrate reset` are exposed | CI or non-interactive environments have no documented migration command | `package.json:14-15` |
| TD-24 | Medium | Deployment topology | No application `Dockerfile` or Compose service; the Node process runs on the host | The database is containerized but the application is not, so the stack is not reproducible as a unit | Repository root; no `Dockerfile` |
| TD-25 | Medium | Root host scope | `MYSQL_ROOT_HOST` is not set, while `SHADOW_DATABASE_URL` expects root from the host | Host-side root connection for migrations can be rejected depending on Docker networking | `docker-compose.yml:6-10`; `.env.example:6` |
| TD-26 | Medium | Secret encoding | Passwords are raw in `MYSQL_*` and URL-embedded in `DATABASE_URL`/`SHADOW_DATABASE_URL` | Special characters can break URL parsing or authentication | `.env.example:5-6,11-14` |
| TD-27 | Low | Network isolation | No custom network, network alias, or policy | All project containers can reach the database port; no tier isolation | `docker-compose.yml:1-28` |
| TD-28 | Low | Healthcheck timing | No `start_interval`; probe every 5 seconds during the start period | Minor probe overhead; slower detection of early readiness | `docker-compose.yml:15-20` |
| TD-29 | Low | Volume naming | Volume has no explicit `name`, so the actual Docker name depends on the Compose project name | Different directory names or `-p` flags produce different volumes; scripts that assume a literal name fail | `docker-compose.yml:26-28` |
| TD-30 | Low | Image vulnerability coverage | No image scan or CVE report is present in the repository | Package-level vulnerabilities in the image are unknown | Dependency audit report, section on image scanning |

Risk concentration:

- The high-risk items cluster in four areas: secrets and network exposure (TD-02, TD-03, TD-04, TD-11), data durability (TD-05), schema and environment reproducibility (TD-01, TD-07, TD-09), and destructive testing (TD-08).
- The medium-risk items cluster in operational predictability: credential rotation, configuration consistency, resource limits, time zone, collation, logging, and migration commands.
- The low-risk items are mostly naming and observability details that matter when the stack is used by more than one developer or when scripts assume literal resource names.
- None of the risks are mitigated by automated checks in the repository. There is no CI configuration, no Compose validation job, no image scan, and no infrastructure test.

---

## 11. Test Coverage Analysis

### 11.1 Test inventory

The MySQL Container has no dedicated test file. The test suite that exercises it lives in `tests/` and is integration-oriented. The suite contains 15 tests: 7 in `tests/auth.test.ts` and 8 in `tests/orders.test.ts`.

| Test file | Lines | Type | Database dependency | What it proves | What it does not prove |
|-----------|-------|------|---------------------|----------------|------------------------|
| `tests/setup.ts` | 20 | Integration harness | High: connects Prisma in `beforeAll`, truncates all tables in `beforeEach`, disconnects in `afterAll` | The application can connect to the configured MySQL server; tables exist; rows can be deleted in foreign-key-safe order | No environment guard, no test database isolation, no container lifecycle, no schema creation |
| `tests/auth.test.ts` | 84 | Integration (HTTP + DB) | High: user rows and unique e-mail index | Registration, duplicate e-mail rejection, invalid payload rejection, login, authenticated `/auth/me`, missing-token rejection | No case-insensitive e-mail test, no collation test, no MySQL auth-plugin test, no rate limiting, no concurrency |
| `tests/orders.test.ts` | 219 | Integration (HTTP + DB) | High: orders, items, products, customers, history, sequence | Order creation and server-side totals, non-existent product handling, status transitions, stock decrement and replenishment, filtering, deletion guard | No concurrent transition test, no row-lock test, no volume or restart test, no time-zone test |
| `tests/helpers/factories.ts` | 83 | Test helper | High: direct Prisma writes | Unique users, customers with JSON address, products with stock, authenticated bootstrap | No schema edge cases, no invalid data cases, no database-level constraint tests |
| `vitest.config.ts` | 19 | Test configuration | Indirect | Serial execution, single fork, 30-second timeouts, setup file loading | No coverage provider, no test database guard, no container startup |
| `package.json` | 54 | Script definitions | Indirect | `test` runs `vitest run`; `db:migrate`, `db:reset`, `db:seed` exist | `test` does not autoload non-`VITE_` `.env` variables; no `migrate deploy` script; no test-container script |
| `prisma/seed.ts` | 335 | Seed script, not a test | High | Not executed by the test suite; demonstrates bulk delete and insert patterns | No seed test or idempotency test |

### 11.2 Coverage assessment

The coverage percentages below are qualitative assessments of infrastructure behavior, not line-coverage numbers. The repository has no coverage provider configured, no `coverage` script in `package.json`, and `coverage/` is gitignored (`.gitignore:3`). YAML and shell configuration cannot be measured by the TypeScript coverage tooling even if it were enabled.

| Component area | Unit tests | Integration tests | Direct coverage | Indirect coverage | Test quality |
|----------------|------------|-------------------|-----------------|-------------------|--------------|
| Compose service definition | 0 | 0 | 0% | None; tests assume the service is already running | No validation of image, restart policy, command flags, or env fallbacks |
| Database provisioning | 0 | 0 | 0% | Implicit only: tests fail if the database or user is missing | No test for first-boot creation or credential drift |
| Application user and grants | 0 | 15 (implicit) | 0% | All tests authenticate as `oms_user` through `DATABASE_URL` | Positive path only; no negative privilege test |
| Root and shadow database | 0 | 0 | 0% | None | No test for `prisma migrate dev` or `oms_shadow` creation |
| Schema and migrations | 0 | 15 (indirect) | 0% | High: tests exercise all seven tables through Prisma and HTTP | Good CRUD coverage; no fresh-volume migration test, no drift test |
| Data types and constraints | 0 | 15 (indirect) | 0% | Moderate: `CHAR(36)`, `INT`, `TEXT`, `JSON`, `ENUM`, `DATETIME(3)`, unique indexes, foreign keys | Good positive coverage; no edge cases for JSON shape, max lengths, or enum rejection |
| Collation semantics | 0 | 7 auth tests (indirect) | 0% | Weak: duplicate-email test uses the same string, not a case variant | No proof of case-insensitive uniqueness or search |
| Transactions and concurrency | 0 | 8 orders tests (indirect) | 0% | Moderate: status transition and stock decrement paths | No concurrent requests; single-fork execution hides row-lock behavior |
| Persistence and volume lifecycle | 0 | 0 | 0% | None | No restart, recreate, `down`, or `down -v` test |
| Healthcheck | 0 | 0 | 0% | None | No probe test, no false-positive test, no timing test |
| Port publication and network | 0 | 15 (implicit) | 0% | Implicit: tests connect through the published port | No port-conflict, bind-address, or container-network test |
| Credential consistency | 0 | 0 | 0% | None | No test that Compose variables and URLs agree |
| Time semantics | 0 | 0 | 0% | None | No time-zone or timestamp assertion |
| Security posture | 0 | 0 | 0% | None | No TLS, auth-plugin, secret-exposure, or exposure test |
| Resource limits and logging | 0 | 0 | 0% | None | No resource or log-rotation test |

Overall assessment:

- Direct coverage of the component is 0%. No test reads, parses, validates, starts, stops, or inspects the Compose service.
- Indirect coverage of the schema is meaningful: 15 integration tests exercise the real MySQL engine, all seven tables, foreign keys, unique indexes, enums, JSON, and transaction paths. This is stronger than a mocked unit test for schema compatibility.
- Indirect coverage of operational behavior is weak. The tests do not prove that a fresh volume can be initialized, migrated, and seeded from scratch. They assume an already-prepared database.
- The test suite is not hermetic. It depends on an external MySQL instance, an external `.env` or shell environment, and a pre-applied schema. It does not use `testcontainers` or a disposable database.
- The test suite is destructive. `tests/setup.ts:8-16` deletes all application rows before every test. There is no assertion that the database is a test database.

### 11.3 Test quality details

Strengths:

- The suite uses the real database engine rather than mocking Prisma, so it catches schema mismatches, foreign-key violations, enum handling, JSON storage, and transaction behavior.
- Assertions are specific: HTTP status codes, application error codes, computed totals, item counts, history length, and order-number format.
- `tests/orders.test.ts:59-88` verifies that a status transition decrements stock; `tests/orders.test.ts:134-162` verifies replenishment on cancellation. These paths depend on real transactions and real rows.
- `tests/auth.test.ts:19-28` verifies duplicate-email rejection through the unique index, although it does not test case variation.
- Serial execution (`fileParallelism: false`, `singleFork: true`) prevents concurrent truncation and cross-test interference.
- Factories generate unique e-mail addresses and SKUs using timestamps and random suffixes, which avoids collisions across runs.

Weaknesses:

- No test validates the Compose file syntax, the image tag, the healthcheck, the restart policy, the port mapping, or the volume declaration.
- No test starts from an empty volume and runs the full sequence: `docker compose up`, `db:migrate`, `db:seed`, application start, query.
- No test verifies that `oms_shadow` can be created with the configured root credentials.
- No test verifies that changing `MYSQL_PASSWORD` or `MYSQL_DATABASE` after first boot has no effect. The first-initialization rule is undocumented in tests.
- No test verifies that `mysqladmin ping` reports unhealthy when credentials are wrong. The probe's false-positive behavior is untested.
- No test verifies case-insensitive collation with a different-case e-mail or name.
- No test verifies time-zone behavior for `CURRENT_TIMESTAMP(3)` or `DATETIME(3)`.
- No concurrency test for status transitions, stock decrement, or order-number allocation. The single-fork configuration cannot exercise parallel database contention.
- No test verifies volume persistence across container recreation.
- No test verifies the seed script's idempotency or its full delete-and-insert behavior.
- `npm test` does not autoload `DATABASE_URL` or `JWT_SECRET` from `.env`, so a developer who relies on `.env` alone must export the variables or use a wrapper.
- No coverage provider is configured, so even the TypeScript code has no line or branch coverage report.

### 11.4 Test-related risks

- The most severe test risk is data destruction: the harness truncates the configured database without checking its identity (`tests/setup.ts:8-16`).
- The second most severe risk is false confidence: the suite passes against a manually prepared database and does not prove that a new environment can be provisioned.
- The third risk is hidden environment coupling: importing `src/config/database.ts` triggers `src/config/env.ts` validation, which exits the process if `DATABASE_URL` or `JWT_SECRET` is missing or invalid. A test run can terminate before any test executes.
- The fourth risk is concurrency blindness: serial execution avoids flakiness but cannot detect the row-lock and double-transition issues described in the architectural report.

---

## Appendix A - Files Inspected

| File | Purpose in this analysis |
|------|--------------------------|
| `docker-compose.yml` | Primary component definition: image, container name, restart policy, environment, ports, volume, healthcheck, command |
| `.env.example` | Database credentials and connection URL contract |
| `.gitignore` | Confirms `.env` is excluded and `coverage/` is excluded |
| `package.json` | Database scripts, test script, Prisma and Node versions |
| `prisma/schema.prisma` | MySQL provider, datasource URLs, seven models, two enums |
| `prisma/migrations/migration_lock.toml` | Provider lock |
| `prisma/migrations/20260519182739_init/migration.sql` | DDL, data types, indexes, foreign keys, charset and collation |
| `prisma/seed.ts` | Database write patterns and delete-before-insert behavior |
| `src/config/env.ts` | `DATABASE_URL` validation and process-exit behavior |
| `src/config/database.ts` | PrismaClient construction and connection behavior |
| `tests/setup.ts` | Live database connection, truncation order, disconnect |
| `tests/auth.test.ts` | Auth integration tests that depend on MySQL |
| `tests/orders.test.ts` | Order integration tests that depend on MySQL |
| `tests/helpers/factories.ts` | Direct Prisma fixture creation |
| `vitest.config.ts` | Serial execution, timeouts, setup file |
| `docs/agents/MANIFEST.md` | Component inventory and report conventions |
| `docs/agents/architectural-analyzer/architectural-report-2026-11-08_22-46-20.md` | System context and prior risk classification |
| `docs/agents/dependency-auditor/dependencies-report-2026-11-08_22-50-33.md` | MySQL 8.0 lifecycle and image tag findings |
| `README.md` | Project context and constraint that application code must not be modified |

## Appendix B - Assumptions and Ambiguities

1. **Official image entrypoint behavior.** The repository does not contain the `mysql:8.0` entrypoint script. Statements about first-initialization-only provisioning, `'oms_user'@'%'`, grant scope, and root host scope are based on the official image's documented behavior and are marked with medium confidence where they are not directly observable. The entrypoint's actual test is whether the data directory is already initialized (for example, whether the `mysql` system schema exists), not merely whether the directory is empty; the report uses "empty" and "populated" as shorthand for uninitialized and initialized.
2. **`mysqladmin ping` semantics.** The statement that a ping can succeed despite an access-denied response is based on MySQL documentation, not on a repository test. It is marked medium confidence.
3. **Docker Compose variable precedence.** The repository does not document whether the shell, `.env`, or `--env-file` wins in every combination. The analysis states the general behavior and treats exact precedence as environment-dependent.
4. **Volume name prefix.** The actual Docker volume name depends on the Compose project name. The analysis refers to `<project>_oms_mysql_data` and treats the literal name `oms_mysql_data` as the Compose key, not necessarily the Docker name.
5. **Root host scope.** `MYSQL_ROOT_HOST` is not set. Whether a host-side root connection through the published port is accepted depends on how Docker presents the source address to MySQL and on the image's default account host pattern. This is flagged as an ambiguity, not a confirmed failure.
6. **Image digest and patch version.** No Docker daemon or registry lookup was performed. The exact resolved `8.0.x` version, digest, base OS, and platform manifest are unknown.
7. **Image default values.** Values such as `max_connections`, `innodb_buffer_pool_size`, `sql_mode`, and the container time zone are described as image defaults, not as repository declarations. They can change across patches.
8. **No runtime measurements.** No container was started, no query was timed, no disk usage was measured, and no failure was reproduced. All findings are static.
9. **No CI or deployment files.** The repository contains no CI workflow, Dockerfile, Kubernetes manifest, Helm chart, or production Compose file. The analysis treats the local Compose service as the entire component.
10. **Application code scope.** Application source was inspected only for contracts that consume the database (`src/config/env.ts`, `src/config/database.ts`) and for the test harness. Business logic inside modules was not analyzed.

## Appendix C - Analysis Boundary

Included in scope:

- The `mysql` service definition and its named volume.
- The database-related environment contract in `.env.example`.
- The schema, migration, seed, test-harness, and connection-string contracts that directly determine how the container is used.
- The absence of production topology, backup, TLS, and observability configuration.

Excluded from scope:

- Application domain logic in `src/modules/`.
- HTTP route behavior, except where the test harness demonstrates database dependency.
- Any file under `node_modules`, `.git`, `dist`, or `coverage`.
- Any change, refactor, or remediation. This report is analysis-only.

---

End of report.

