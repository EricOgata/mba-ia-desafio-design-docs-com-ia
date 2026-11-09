# Padrões Arquiteturais — Sistema de Webhooks

> **Item do TODO:** 1.2 — Registrar o padrão arquitetural a ser seguido.
>
> **Objetivo:** justificar o reuso dos padrões existentes em vez de criar uma estrutura paralela.
>
> **Fonte:** código-fonte do repositório. Todos os caminhos e linhas foram verificados no disco. Nada aqui foi inferido de outro projeto.
>
> **Uso:** insumo direto do ADR de reuso (`ADR-006`, a criar na Etapa 2) e da seção "Integração com o sistema existente" do `docs/FDD.md`.
>
> **Relação com 1.1:** `docs/INTEGRATION-POINTS.md` lista **onde** a feature se integra; este documento descreve **como** o código existente está organizado e **o que deve ser replicado**.

---

## 1. Decisão desta etapa

| Item | Decisão |
| --- | --- |
| Onde registrar a saída | `docs/ARCHITECTURE-PATTERNS.md` |
| ADR de reuso | Adiado para a Etapa 2, com numeração reservada `ADR-006-adocao-dos-padroes-existentes.md` |
| Status do item 1.2 | **Documentado, porém não concluído**: o critério "pronto quando" exige pelo menos 1 ADR referenciando explicitamente estes arquivos |

> **Consequência:** o item 1.2 permanece `[~]` no `TODO.md` até o ADR-006 ser criado na Etapa 2. O conteúdo deste documento já atende ao restante do critério.

---

## 2. Base de evidência

Arquivos lidos integralmente para sustentar este documento:

| Área | Arquivos |
| --- | --- |
| Módulos | `src/modules/{auth,users,customers,products,orders}/*.{controller,service,repository,routes,schemas}.ts` e `src/modules/orders/order.status.ts` |
| Erros | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts` |
| Middlewares | `src/middlewares/error.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/request-logger.middleware.ts` |
| Composição | `src/app.ts`, `src/routes/index.ts`, `src/server.ts` |
| Infra | `src/config/database.ts`, `src/config/env.ts`, `src/shared/logger/index.ts`, `src/shared/http/response.ts` |
| Dados | `prisma/schema.prisma` |

---

## 3. Anatomia de um módulo

O projeto organiza cada domínio em `src/modules/<nome>/`, com cinco arquivos e uma responsabilidade por camada. O padrão aparece completo em `customers`, `products` e `orders`.

| Camada | Arquivo | Responsabilidade | Não faz | Evidência |
| --- | --- | --- | --- | --- |
| Controller | `<entidade>.controller.ts` | Extrai `req.params`, `req.body` e `req.query`, chama o service, escolhe o status HTTP e delega erros com `next(err)` | Não contém regra de negócio, não acessa Prisma, não valida schema | `src/modules/customers/customer.controller.ts:5-53`; `src/modules/orders/order.controller.ts:6-56` |
| Service | `<entidade>.service.ts` | Concentra regras de negócio, orquestra repository e transações, lança `AppError` e devolve dados de domínio | Não conhece `Request`/`Response`, não monta HTTP | `src/modules/customers/customer.service.ts:11-64`; `src/modules/orders/order.service.ts:26-255` |
| Repository | `<entidade>.repository.ts` | Executa consultas Prisma e encapsula `where`, paginação e transações de leitura | Não decide regra de negócio, não lança `AppError` de domínio | `src/modules/customers/customer.repository.ts:9-57`; `src/modules/products/product.repository.ts:10-62` |
| Routes | `<entidade>.routes.ts` | Cria o `Router`, aplica `authenticate`/`requireRole` e `validate`, e liga verbo + caminho ao handler | Não contém lógica de negócio | `src/modules/customers/customer.routes.ts:12-27`; `src/modules/orders/order.routes.ts:12-27` |
| Schemas | `<entidade>.schemas.ts` | Define schemas Zod de body, query e params e exporta os tipos com `z.infer` | Não executa consulta nem regra de negócio | `src/modules/customers/customer.schemas.ts:1-33`; `src/modules/orders/order.schemas.ts:1-34` |

### 3.1 Convenções de nome observadas

| Item | Convenção | Exemplos |
| --- | --- | --- |
| Pasta do módulo | plural | `users`, `customers`, `products`, `orders` |
| Prefixo dos arquivos | substantivo no singular | `user.*`, `customer.*`, `product.*`, `order.*` |
| Classes | `<Entidade>Controller`, `<Entidade>Service`, `<Entidade>Repository` | `OrderController`, `OrderService`, `OrderRepository` |
| Factory de router | `build<Entidade>Router(controller)` | `buildOrderRouter` (`src/modules/orders/order.routes.ts:12`) |
| Tipos de entrada | `<Ação><Entidade>Input` e `<Entidade>ListQuery` | `CreateCustomerInput`, `ListOrdersQuery` |
| Schemas | `<ação><Entidade>Schema`, `<entidade>IdParamSchema` | `createProductSchema`, `orderIdParamSchema` |

> Para webhooks, a convenção derivada é `src/modules/webhooks/webhook.{controller,service,repository,routes,schemas}.ts`, coerente com o prefixo singular já usado no projeto. O nome exato dos arquivos do worker continua em aberto (ver seção 11).

---

## 4. Direção de dependência

Fluxo real de uma requisição:

```text
HTTP
  -> routes (authenticate -> requireRole? -> validate -> controller)
  -> controller (extrai dados, chama service, responde)
  -> service (regra de negócio, transação, AppError)
  -> repository (Prisma)
  -> PrismaClient / MySQL
```

Regras observadas:

1. **Routes conhecem controller, não service.** As factories recebem o controller por injeção (`src/modules/orders/order.routes.ts:12`).
2. **Controller conhece service, não repository.** O construtor recebe apenas o service (`src/modules/orders/order.controller.ts:7`; `src/modules/customers/customer.controller.ts:6`).
3. **Service conhece repository e, quando precisa orquestrar transação, também o `PrismaClient`.** `OrderService` recebe os dois (`src/modules/orders/order.service.ts:27-30`).
4. **Repository conhece apenas `PrismaClient`.** Recebe a instância no construtor (`src/modules/customers/customer.repository.ts:10`; `src/modules/orders/order.repository.ts:19`).
5. **Schemas não dependem de nenhuma camada.** São importados por routes e service (`src/modules/orders/order.routes.ts:4-9`; `src/modules/orders/order.service.ts:8`).
6. **A composição fica fora do módulo.** `src/app.ts` instancia repository -> service -> controller (`src/app.ts:26-53`).

### 4.1 Regra de ouro para a feature

> O módulo de webhooks deve repetir essa direção. Nenhuma camada nova pode inverter a dependência (por exemplo: repository lançando regra de negócio ou controller acessando Prisma).

---

## 5. Exceções e variações reais

O padrão não é aplicado de forma dogmática. Estas variações foram observadas e devem ser tratadas como precedente, não como desvio a corrigir.

| Variação | O que acontece | Evidência | Leitura para webhooks |
| --- | --- | --- | --- |
| `auth` sem repository próprio | `AuthService` recebe `UserRepository` e `UserService`; `AuthController` recebe `AuthService` e `UserService` | `src/modules/auth/auth.service.ts:21-29`; `src/modules/auth/auth.controller.ts:6-10` | Um service pode compor outro quando o domínio é compartilhado |
| `orders` usa `PrismaClient` direto no service | `create` e `changeStatus` abrem `this.prisma.$transaction` e usam `tx` diretamente, em vez de esconder tudo no repository | `src/modules/orders/order.service.ts:58`, `:131` | A inserção da outbox dentro do `tx` de `changeStatus` é coerente com o padrão existente |
| Schema inline em rota | `user.routes.ts` define `idParamSchema` no próprio arquivo, além de existir `user.schemas.ts` | `src/modules/users/user.routes.ts:7` | Preferir `webhook.schemas.ts`; o inline é exceção, não regra |
| Rota aplica `authenticate` global | `customers`, `products` e `orders` usam `router.use(authenticate)`; `users` aplica por rota | `src/modules/orders/order.routes.ts:14`; `src/modules/users/user.routes.ts:12-18` | CRUD de webhook pode usar `router.use(authenticate)` e aplicar `requireRole('ADMIN')` só no replay |

---

## 6. Padrão de códigos de erro

### 6.1 Estrutura

Toda exceção de negócio deriva de `AppError`, que carrega três campos públicos:

| Campo | Papel | Evidência |
| --- | --- | --- |
| `statusCode` | status HTTP a devolver | `src/shared/errors/app-error.ts:4` |
| `errorCode` | código estável consumido pelo cliente | `src/shared/errors/app-error.ts:5` |
| `details` | contexto opcional (objeto ou lista) | `src/shared/errors/app-error.ts:6` |

As classes concretas ficam em `src/shared/errors/http-errors.ts` e são reexportadas por `src/shared/errors/index.ts:1-13`.

### 6.2 Classes base disponíveis

| Classe | HTTP | Código padrão | Evidência |
| --- | --- | --- | --- |
| `BadRequestError` | 400 | `BAD_REQUEST` | `src/shared/errors/http-errors.ts:3-7` |
| `ValidationError` | 400 | `VALIDATION_ERROR` | `src/shared/errors/http-errors.ts:9-13` |
| `UnauthorizedError` | 401 | `UNAUTHORIZED` | `src/shared/errors/http-errors.ts:15-19` |
| `ForbiddenError` | 403 | `FORBIDDEN` | `src/shared/errors/http-errors.ts:21-25` |
| `NotFoundError` | 404 | `NOT_FOUND` | `src/shared/errors/http-errors.ts:27-31` |
| `ConflictError` | 409 | `CONFLICT` (sobrescrevível) | `src/shared/errors/http-errors.ts:33-37` |
| `UnprocessableEntityError` | 422 | `UNPROCESSABLE_ENTITY` (sobrescrevível) | `src/shared/errors/http-errors.ts:39-43` |

### 6.3 Códigos de domínio já usados

O padrão real é **estender a classe base e passar um código específico no construtor**, não criar um enum central.

| Código | Classe | HTTP | Onde é lançado | Evidência |
| --- | --- | --- | --- | --- |
| `INVALID_STATUS_TRANSITION` | `InvalidStatusTransitionError extends ConflictError` | 409 | `changeStatus` quando a transição não é permitida; e diretamente em `ConflictError` quando o pedido já está no status | `src/shared/errors/http-errors.ts:45-53`; `src/modules/orders/order.service.ts:141-149` |
| `INSUFFICIENT_STOCK` | `InsufficientStockError extends UnprocessableEntityError` | 422 | `debitStock` quando falta estoque | `src/shared/errors/http-errors.ts:55-63`; `src/modules/orders/order.service.ts:222-224` |
| `INACTIVE_PRODUCT` | `UnprocessableEntityError` direto | 422 | `create` quando há produto inativo | `src/modules/orders/order.service.ts:66-73` |
| `INVALID_ORDER_STATE_FOR_DELETE` | `ConflictError` direto | 409 | `delete` fora de `PENDING`/`CANCELLED` | `src/modules/orders/order.service.ts:183-190` |
| `EMAIL_ALREADY_USED` | `ConflictError` direto | 409 | `create` de customer e de user | `src/modules/customers/customer.service.ts:29-33`; `src/modules/users/user.service.ts:21-25` |
| `SKU_ALREADY_USED` | `ConflictError` direto | 409 | `create` e `update` de product | `src/modules/products/product.service.ts:33`, `:50` |

### 6.4 Formato da resposta de erro

O middleware central serializa qualquer `AppError` no mesmo envelope:

```json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "One or more products do not have enough stock",
    "details": { "unavailable": [] }
  }
}
```

Evidência: `src/middlewares/error.middleware.ts:15-24`. O campo `details` só aparece quando definido (`:20`).

O middleware também cobre:

| Situação | Resposta | Evidência |
| --- | --- | --- |
| `ZodError` não tratado pelo `validate` | 400 `VALIDATION_ERROR` com lista de `path`/`message` | `src/middlewares/error.middleware.ts:26-35` |
| Prisma `P2002` (unique) | 409 `CONFLICT` | `src/middlewares/error.middleware.ts:38-47` |
| Prisma `P2025` (registro ausente) | 404 `NOT_FOUND` | `src/middlewares/error.middleware.ts:48-54` |
| Erro não previsto | 500 `INTERNAL_SERVER_ERROR`, com log estruturado e `requestId` | `src/middlewares/error.middleware.ts:56-64` |

### 6.5 Regra derivada para `WEBHOOK_*`

O padrão a seguir, já decidido em `D-11` (`[09:28]–[09:30] Bruno/Larissa`):

1. Criar classes em `src/modules/webhooks/webhook.errors.ts`, estendendo `AppError` ou a subclasse cujo HTTP já é o correto.
2. Passar o código `WEBHOOK_*` pelo construtor, como `InvalidStatusTransitionError` e `InsufficientStockError` já fazem.
3. Não alterar `error.middleware.ts`: ele já trata qualquer descendente de `AppError`.
4. Não criar um segundo mecanismo de erro ou um enum paralelo.

> O catálogo fechado de códigos `WEBHOOK_*` pertence ao FDD (item 4.3). Aqui fica registrado apenas o padrão estrutural.

---

## 7. Validação com Zod

| Regra | Evidência |
| --- | --- |
| Validação entra por middleware, não pelo controller | `src/middlewares/validate.middleware.ts:11-37` |
| O middleware aceita `body`, `query` e `params` | `src/middlewares/validate.middleware.ts:5-9`, `:14-23` |
| `body` é substituído pelo valor parseado | `src/middlewares/validate.middleware.ts:15` |
| `query` é mesclado ao original | `src/middlewares/validate.middleware.ts:18-19` |
| `params` é substituído pelo valor parseado | `src/middlewares/validate.middleware.ts:22` |
| `ZodError` vira `ValidationError` com `path`/`message` | `src/middlewares/validate.middleware.ts:26-33` |
| Listagens padronizam `page=1`, `pageSize=20`, máximo `100` | `src/modules/orders/order.schemas.ts:23-26`; `src/modules/customers/customer.schemas.ts:25-28`; `src/modules/products/product.schemas.ts:16-19` |
| IDs são validados como UUID | `src/modules/orders/order.schemas.ts:4`; `src/modules/customers/customer.schemas.ts:3-5` |
| Schemas exportam tipos com `z.infer` | `src/modules/customers/customer.schemas.ts:31-33`; `src/modules/products/product.schemas.ts:26-28` |

> Para webhooks, a validação de URL HTTPS (`D-16`, `[09:23] Sofia`) deve entrar no schema Zod, conforme `C-09`, e não no service.

---

## 8. Autenticação e autorização

| Peça | Comportamento | Evidência |
| --- | --- | --- |
| `authenticate` | Lê `Authorization: Bearer`, valida JWT com `JWT_SECRET` e preenche `req.user` com `id`, `email` e `role` | `src/middlewares/auth.middleware.ts:27-47` |
| `requireRole(...roles)` | Exige `req.user` e pelo menos uma das roles; caso contrário, 401 ou 403 | `src/middlewares/auth.middleware.ts:49-61` |
| Uso de `ADMIN` | Rota de usuário protegida com `authenticate` + `requireRole('ADMIN')` | `src/modules/users/user.routes.ts:12-18` |
| Uso por módulo | `customers`, `products` e `orders` aplicam `router.use(authenticate)` | `src/modules/customers/customer.routes.ts:14`; `src/modules/products/product.routes.ts:14`; `src/modules/orders/order.routes.ts:14` |
| Guarda no controller | Controllers que usam `req.user` falham com `UnauthorizedError` se ele não existir | `src/modules/orders/order.controller.ts:30`, `:40`; `src/modules/auth/auth.controller.ts:32` |

> Aplicação em webhooks: CRUD com `authenticate`; replay de DLQ com `authenticate` + `requireRole('ADMIN')` (`D-06`, `D-18`, `C-02`).

---

## 9. Logger e observabilidade

| Item | Comportamento | Evidência |
| --- | --- | --- |
| Biblioteca | Pino | `src/shared/logger/index.ts:1` |
| Nível | `env.LOG_LEVEL` | `src/shared/logger/index.ts:15` |
| Redaction | `authorization`, `cookie`, `password`, `passwordHash`, `token`, `accessToken` | `src/shared/logger/index.ts:4-11` |
| Base | `service: 'order-management-api'` e `env` | `src/shared/logger/index.ts:20` |
| Export | Instância única `logger` | `src/shared/logger/index.ts:32` |
| Request log | Gera/propaga `X-Request-Id` e registra método, path, status, duração e `userId` | `src/middlewares/request-logger.middleware.ts:5-27` |

**Lacuna identificada:** o logger atual não redige secrets de webhook. Se o módulo logar o payload de configuração, a secret pode vazar. O FDD deve decidir se adiciona um caminho de redaction específico. Isso não muda o padrão de reuso do logger.

---

## 10. Composição e registro de rotas

### 10.1 Composition root

`src/app.ts` é o único lugar que monta a árvore de dependências:

```text
buildControllers(prisma)
  -> new <Entidade>Repository(prisma)
  -> new <Entidade>Service(repository[, prisma])
  -> new <Entidade>Controller(service)
  -> retorna Controllers
buildApp(deps)
  -> express.json({ limit: '1mb' })
  -> requestLogger
  -> GET /health
  -> /api/v1 -> buildApiRouter(controllers)
  -> 404 -> errorMiddleware
```

Evidência: `src/app.ts:26-53` (controllers), `:55-75` (app).

### 10.2 Registro de rota

| Item | Comportamento | Evidência |
| --- | --- | --- |
| Tipo `Controllers` | Um campo por módulo | `src/routes/index.ts:13-19` |
| `buildApiRouter` | Monta um router por prefixo | `src/routes/index.ts:21-31` |
| Prefixo global | `/api/v1` | `src/app.ts:67` |

> Adicionar webhooks exige alterar `src/app.ts` e `src/routes/index.ts` (já registrado em `docs/INTEGRATION-POINTS.md`, itens 13 e 14).

### 10.3 Entry point

| Item | Comportamento | Evidência |
| --- | --- | --- |
| Bootstrap | `buildApp({ prisma })`, `app.listen(env.PORT)` | `src/server.ts:6-11` |
| Shutdown | `SIGINT`/`SIGTERM`, fecha servidor e `prisma.$disconnect()` | `src/server.ts:13-21` |
| Falha de bootstrap | `logger.fatal` e `exit(1)` | `src/server.ts:24-27` |
| Prisma | `createPrismaClient()` + singleton `prisma` | `src/config/database.ts:4-10` |

> O worker deve seguir o mesmo formato de bootstrap/shutdown em um entry point próprio (`src/worker.ts`), com instância Prisma do próprio processo (`C-04`, `C-07`).

---

## 11. Adaptação para o módulo de webhooks

A tabela abaixo traduz o padrão existente para os arquivos planejados. **Nada em `src/modules/webhooks/` existe hoje**; os nomes vêm de `docs/INTEGRATION-POINTS.md` e do padrão observado.

| Camada | Arquivo planejado | Padrão de referência | O que replicar | Status |
| --- | --- | --- | --- | --- |
| Schemas | `src/modules/webhooks/webhook.schemas.ts` | `order.schemas.ts` | Zod por operação, `idParamSchema`, paginação e validação HTTPS | **Criar** |
| Repository | `src/modules/webhooks/webhook.repository.ts` | `customer.repository.ts` | Só Prisma; `buildWhere` privado; sem regra de negócio | **Criar** |
| Service | `src/modules/webhooks/webhook.service.ts` | `customer.service.ts` + `order.service.ts` | Regras, transações quando necessário, `AppError` | **Criar** |
| Controller | `src/modules/webhooks/webhook.controller.ts` | `order.controller.ts` | Handlers finos, `next(err)`, status HTTP | **Criar** |
| Routes | `src/modules/webhooks/webhook.routes.ts` | `order.routes.ts` + `user.routes.ts` | `authenticate`, `requireRole('ADMIN')` no replay, `validate` | **Criar** |
| Erros | `src/modules/webhooks/webhook.errors.ts` | `http-errors.ts` | Classes `WEBHOOK_*` estendendo `AppError` | **Criar** |
| Outbox | `src/modules/webhooks/webhook.outbox.ts` | `order.service.ts:131` | Função pura recebendo `tx`, usada dentro da transação | **Criar** |
| Processamento | `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts` | — | Nome ainda em aberto | **Criar** |
| Entry point do worker | `src/worker.ts` | `src/server.ts` | Bootstrap, shutdown e Prisma próprio | **Criar** |
| Composição | `src/app.ts` | `buildControllers` | Instanciar e registrar o controller | **Alterar** |
| Rotas da API | `src/routes/index.ts` | `Controllers` + `buildApiRouter` | Adicionar `/webhooks` e `/admin/webhooks` | **Alterar** |
| Gatilho | `src/modules/orders/order.service.ts:126-179` | `$transaction` existente | Chamar `publishWebhookEvent(tx, order, from, to)` dentro do `tx` | **Alterar** |

### 11.1 O que é reuso puro

| Recurso existente | Uso em webhooks | Evidência |
| --- | --- | --- |
| `AppError` e subclasses | Base dos erros `WEBHOOK_*` | `src/shared/errors/app-error.ts`; `src/shared/errors/http-errors.ts` |
| `errorMiddleware` | Serializa os novos erros sem alteração | `src/middlewares/error.middleware.ts:15-24` |
| `validate` | Valida body/query/params | `src/middlewares/validate.middleware.ts` |
| `authenticate` / `requireRole` | CRUD e replay | `src/middlewares/auth.middleware.ts` |
| `logger` | API e worker | `src/shared/logger/index.ts` |
| `createPrismaClient` | Instância do worker | `src/config/database.ts:4-8` |
| `paginated` / `PaginatedResponse` | Listagens e histórico | `src/shared/http/response.ts:1-24` |
| Padrão UUID | IDs da outbox e das configurações | `prisma/schema.prisma:26,41,57,75,100,117` |

### 11.2 O que **não** deve ser replicado

| Anti-padrão | Por quê |
| --- | --- |
| Lógica de negócio no controller | Contraria todos os módulos existentes |
| `AppError` lançado no repository | Repository atual só faz acesso a dados |
| Validação de URL/HTTPS no service | `D-16` e `C-09` colocam a validação no schema Zod |
| Segundo logger ou segundo middleware de erro | `D-11` exige reuso |
| Chamada HTTP síncrona dentro de `changeStatus` | `D-01` exige outbox transacional |
| Worker reutilizando a instância Prisma da API | `C-04` exige instância própria por processo |

---

## 12. Matriz de decisão para o ADR-006

O ADR de reuso deve conseguir responder, com base neste documento:

| Pergunta | Resposta sustentada |
| --- | --- |
| Por que não criar estrutura paralela? | O padrão modular já existe e é repetido em 4 módulos completos (`customers`, `products`, `orders` e parcialmente `users`/`auth`) |
| Quais arquivos provam o padrão? | `src/modules/orders/*`, `src/modules/customers/*`, `src/modules/products/*` |
| Quais arquivos provam o padrão de erro? | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts` |
| Quais arquivos provam a composição? | `src/app.ts:26-75`, `src/routes/index.ts:13-31` |
| Quais peças são reutilizadas sem alteração? | `errorMiddleware`, `validate`, `authenticate`, `requireRole`, `logger`, `createPrismaClient`, `paginated` |
| Quais peças são alteradas? | `order.service.ts`, `app.ts`, `routes/index.ts`, `prisma/schema.prisma` |
| Quais são novas? | `src/modules/webhooks/*` e `src/worker.ts` |
| Qual exceção do projeto foi considerada? | `OrderService` já usa `PrismaClient.$transaction` diretamente; a outbox segue o mesmo estilo |

---

## 13. Pontos em aberto

| # | Ponto | Onde fechar |
| --- | --- | --- |
| 1 | Nome do arquivo do processador: `webhook.worker.ts` ou `webhook.processor.ts` | FDD (4.1) |
| 2 | Nomes dos modelos Prisma de configuração, outbox, delivery e DLQ | FDD (4.1) |
| 3 | Catálogo completo de códigos `WEBHOOK_*` e respectivos HTTP | FDD (4.3) |
| 4 | Posição exata de `publishWebhookEvent` dentro do `changeStatus` | FDD (4.1) |
| 5 | Redaction de secrets de webhook no logger | FDD (4.4) |
| 6 | ~~Divergência de nomenclatura: `docs/adrs/README.md` cita `0001-titulo.md`, enquanto o TODO 2.1 e a decisão desta etapa usam `ADR-NNN-...`~~ | **Resolvido no item 2.1:** `docs/adrs/README.md` agora define `ADR-NNN-titulo-em-kebab-case.md` |
| 7 | Criação efetiva do `ADR-006-adocao-dos-padroes-existentes.md` | Etapa 2.2 |

---

## 14. Checklist do item 1.2

- [x] Padrão `controller + service + repository + routes + schemas` documentado com caminhos reais
- [x] Direção de dependência descrita
- [x] Convenções de nome registradas
- [x] Exceções reais registradas (`auth`, `orders`, schema inline em `users`)
- [x] Padrão de erros documentado (`AppError`, classes concretas, formato de resposta)
- [x] Códigos `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` rastreados até a linha
- [x] Validação Zod, auth, logger, composição e entry point documentados
- [x] Matriz de adaptação e matriz de reuso para webhooks
- [x] Nenhum arquivo de código da aplicação alterado
- [ ] **ADR-006 referenciando explicitamente estes arquivos** — adiado para a Etapa 2, conforme decisão registrada na seção 1
