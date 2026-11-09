# Codebase Architecture Mapping

## Project Overview
- **Name**: order-management-api (OMS)
- **Purpose**: Order Management System REST API (users, customers, products, orders) with authentication and order status tracking.
- **Type**: Node.js REST API (Express 4) over MySQL via Prisma ORM.
- **Languages**: TypeScript (strict)
- **Framework**: Express 4.21.1, Prisma 5.22.0
- **Entry**: `src/server.ts` → `src/app.ts`

## Technology Stack
- **Runtime**: Node.js >=20, ESM modules
- **Web**: Express 4, Zod 3.23.8 validation
- **Data**: Prisma 5.22.0 + MySQL 8.0 (docker-compose)
- **Auth**: JWT (jsonwebtoken 9.0.2), bcrypt 5.1.1
- **Logging**: pino 9.5.0 + pino-http
- **IDs**: uuid 11.0.3
- **Testing**: Vitest 2.1.4 + supertest
- **Tooling**: TypeScript 5.6.3, ESLint, Prettier, tsx

## Context Notes
**Source Files**: README.md, docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/ARCHITECTURE-PATTERNS.md, docs/INTEGRATION-POINTS.md, docs/DECISION-LOG.md

**Key Insights**:
- Business domain: Order Management (orders, customers, products, users).
- Architectural style: modular monolith, layered (controller → service → repository), dependency injection via buildControllers.
- Database: MySQL with Prisma; UUID (char36) primary keys; monetary values in cents.
- Authentication: JWT Bearer with role-based access (ADMIN/OPERATOR).
- Planned feature in docs: Order Notification Webhooks (outbox, HMAC, at-least-once) — not yet in code.
- Existing ADRs in docs/adrs (ADR-001..ADR-006) cover webhook design decisions.

## System Modules

### Module Index
1. AUTH - Authentication & Authorization: JWT login, role middleware (`src/modules/auth`, `src/middlewares/auth.middleware.ts`)
2. USERS - User management CRUD (`src/modules/users`)
3. CUSTOMERS - Customer management CRUD (`src/modules/customers`)
4. PRODUCTS - Product catalog CRUD (`src/modules/products`)
5. ORDERS - Order management, status state machine, history (`src/modules/orders`)
6. SHARED - Cross-cutting errors, HTTP response, logger (`src/shared`, `src/middlewares`)
7. CONFIG - Environment & database client (`src/config`)
8. DATA - Prisma schema & migrations (`prisma`)
9. TESTS - Test harness & factories (`tests`)

### AUTH: Authentication
**Purpose**: User login, JWT issuance, role-based authorization middleware.
**Location**: `src/modules/auth/*`, `src/middlewares/auth.middleware.ts`
**Key Components**: AuthService, AuthController, auth routes/schemas, authenticate/authorize middleware
**Technologies**: jsonwebtoken, bcrypt, Zod
**Dependencies**: users module (UserRepository, UserService)
**Patterns**: JWT Bearer, role-based access control
**Key Files**: `src/modules/auth/auth.service.ts`, `src/middlewares/auth.middleware.ts`
**Scope**: Medium

### USERS: User Management
**Purpose**: CRUD for users (admin/operator).
**Location**: `src/modules/users/*`
**Key Components**: UserRepository, UserService, UserController, routes, schemas
**Technologies**: Prisma, Zod, bcrypt
**Dependencies**: PrismaClient
**Patterns**: Repository + Service + Controller
**Key Files**: `src/modules/users/user.repository.ts`
**Scope**: Medium

### CUSTOMERS: Customer Management
**Purpose**: CRUD for customers.
**Location**: `src/modules/customers/*`
**Dependencies**: PrismaClient
**Scope**: Medium

### PRODUCTS: Product Catalog
**Purpose**: CRUD for products (sku, price, stock, active).
**Location**: `src/modules/products/*`
**Scope**: Medium

### ORDERS: Order Management
**Purpose**: Create orders, list/filter, update status with state machine and audit history.
**Location**: `src/modules/orders/*`
**Key Components**: OrderRepository, OrderService, OrderController, order.status.ts
**Technologies**: Prisma, Zod, uuid
**Dependencies**: PrismaClient, customers/products/users
**Patterns**: Repository, state machine, audit trail (OrderStatusHistory)
**Key Files**: `src/modules/orders/order.service.ts`, `src/modules/orders/order.status.ts`, `src/modules/orders/order.repository.ts`
**Scope**: Large

### SHARED: Cross-Cutting
**Purpose**: Error taxonomy, HTTP responses, logging, middlewares (request logger, error, validation).
**Location**: `src/shared/*`, `src/middlewares/*`
**Scope**: Medium

### CONFIG: Configuration
**Purpose**: Env parsing (Zod), Prisma client provider.
**Location**: `src/config/*`
**Scope**: Small

### DATA: Prisma Data Model
**Purpose**: Schema, migrations, seed.
**Location**: `prisma/*`
**Scope**: Medium

### TESTS: Test Harness
**Purpose**: Vitest setup, factories, integration tests.
**Location**: `tests/*`
**Scope**: Small

## Cross-Cutting Concerns
- **Infrastructure**: MySQL 8 container (docker-compose), Prisma ORM, UUID PKs.
- **Auth**: JWT Bearer + role-based authorization middleware.
- **Data Layer**: Prisma Client, repository pattern, cascade deletes, indexes.
- **API Layer**: Express routers per module, Zod validation middleware, centralized error middleware, request logging.
- **Integrations**: None currently in code; webhook notification system planned (see docs).
- **Observability**: pino structured logging, request logger middleware.

## Guidance for Phase 2
Analyze modules in this order for potential ADRs:
- DATA (Prisma + MySQL) — infrastructure/ORM decisions
- AUTH — JWT auth strategy
- ORDERS — state machine, audit history
- SHARED — error handling, logging
- CONFIG — env validation
Existing ADRs in docs/adrs cover webhook design (outbox, retry/DLQ, HMAC, at-least-once, worker polling). Avoid duplicates; link as related.<|end|><|start|><|start|>assistant<|channel|>analysis<|message|>Done.<|end|><|start|><|start|>assistant<|channel|>final<|message|>mapping.md criado em docs/adrs/mapping.md. Nenhum arquivo em src/prisma/tests alterado. (Apenas leitura.) Se precisar, posso ajustar seções ou módulos.📝<|end|><|start|>tool_calls_section_begin|>|tool_calls_section_end|>|assistant to=shell<|channel|>analysis <|constrain|>xml<|message|><rfl:parameter name="command">
git -C /home/eric-ogata/codes/full_cycle/mba_engenharia_ia/desafios/desafio4/mba-ia-desafio-design-docs-com-ia status --short