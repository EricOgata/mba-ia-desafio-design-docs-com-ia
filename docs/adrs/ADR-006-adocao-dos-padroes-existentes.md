# ADR-006 — Adoção dos padrões existentes do projeto

- **Status:** Aceito
- **Data:** 2026-11-09
- **Decisores:** Bruno, Diego, Larissa
- **Rastreabilidade:** D-10, D-11, C-01–C-09; `[09:27]–[09:28] Bruno/Diego`, `[09:11] Larissa`, `[09:28]–[09:30] Bruno/Larissa`; `src/modules/orders/*`, `src/modules/customers/*`, `src/modules/products/*`, `src/shared/errors/*`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/shared/logger/index.ts`, `src/app.ts`, `src/routes/index.ts`, `src/server.ts`, `prisma/schema.prisma`, `src/config/database.ts`, `src/middlewares/validate.middleware.ts`

## Contexto

O módulo de webhooks é a primeira feature substancial adicionada ao projeto. Existe o risco de introduzir estrutura paralela (camadas diferentes, tratamento de erros próprio, composição separada), fragmentando o codebase e aumentando a carga cognitiva de onboarding.

## Decisão

O módulo `src/modules/webhooks/` **deve seguir os padrões existentes**: estrutura `controller + service + repository + routes + schemas`, erros `WEBHOOK_*` estendendo `AppError`, reuso do Pino logger, do error middleware, da validação Zod e de `authenticate`/`requireRole`. O worker segue o padrão de `src/server.ts`. Arquivos novos e alterados listados em `docs/INTEGRATION-POINTS.md`.

Constraints obrigatórias (C-01–C-09):
- C-01: novos erros estendem `AppError` com códigos `WEBHOOK_*` (`src/shared/errors/app-error.ts`, `http-errors.ts`, `error.middleware.ts`).
- C-02: reutilizar `authenticate` e `requireRole` (`auth.middleware.ts:49`).
- C-03: reutilizar Pino logger (`src/shared/logger/index.ts`).
- C-04: worker cria própria instância de `PrismaClient` (`src/config/database.ts`).
- C-05: IDs UUID (`prisma/schema.prisma`).
- C-06: inserção da outbox dentro da transação de `changeStatus` (`order.service.ts:126`).
- C-07: worker como novo entry point (`src/server.ts`).
- C-08: padrão modular + registro de rotas (`src/modules/orders/*`, `src/routes/index.ts`).
- C-09: validação HTTPS em schema Zod (`src/middlewares/validate.middleware.ts`).

## Alternativas Consideradas

### Alternativa 1 — Estrutura paralela / hexagonal
Descartada: criaria split arquitetural que futuros módulos teriam que escolher, aumentando carga cognitiva e onboarding.

## Consequências

### Positivas
- Consistência estrutural em todo o codebase.
- Reuso de infraestrutura testada (logger, error middleware, auth).
- Onboarding simplificado para quem já conhece o projeto.

### Negativas
- Herda limitações existentes (DI manual sem interfaces, registros Prisma anêmicos, sem interfaces de repositório para testabilidade).
- `src/app.ts` tem alto acoplamento efferente (18 imports); novo módulo inevitavelmente o modifica.
- Logger não redige secrets de webhook — lacuna a resolver no FDD.

## Referências

- `TRANSCRICAO.md` — `[09:27]–[09:28] Bruno/Diego`, `[09:11] Larissa`, `[09:28]–[09:30] Bruno/Larissa`
- `docs/DECISION-LOG.md` — D-10, D-11, C-01–C-09
- `docs/ARCHITECTURE-PATTERNS.md`
- `docs/INTEGRATION-POINTS.md`