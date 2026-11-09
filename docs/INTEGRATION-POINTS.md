# Pontos de Integração — Sistema de Webhooks

> **Item do TODO:** 1.1 — Identificar os pontos de integração reais.
>
> **Fonte:** código-fonte do repositório e decisões registradas em `docs/DECISION-LOG.md`. Todos os caminhos abaixo foram verificados no disco; nenhum é suposição.
>
> **Uso:** este documento é insumo direto da seção "Integração com o sistema existente" do `docs/FDD.md`.
>
> **Complemento:** os padrões que devem ser replicados estão em `docs/ARCHITECTURE-PATTERNS.md` (item 1.2).

---

## 1. Decisões desta etapa

| Decisão | Escolha |
| --- | --- |
| Onde registrar a saída | `docs/INTEGRATION-POINTS.md` |
| Abrangência | 16 caminhos existentes, separados por tipo de integração |
| Classificação | `Alterar`, `Reutilizar`, `Referenciar`, `Criar` |
| Local dos erros `WEBHOOK_*` | `src/modules/webhooks/webhook.errors.ts` |
| Local de `publishWebhookEvent` | `src/modules/webhooks/webhook.outbox.ts` |
| Caminho não-`src` na tabela | Apenas `prisma/schema.prisma` |
| Rotas admin | `/api/v1/admin/webhooks/...` |
| `package.json` | Fora da tabela; o script do worker será tratado como configuração no FDD |

---

## 2. Tabela de integração — caminhos existentes

| # | Caminho | Responsabilidade atual | Integração com webhooks | Tipo | Evidência |
| --- | --- | --- | --- | --- | --- |
| 1 | `src/modules/orders/order.service.ts` | `OrderService.changeStatus` (linha 126) executa `$transaction`: valida transição, debita/recompõe estoque, atualiza `orders` e insere em `order_status_history` | Inserir o evento na outbox dentro do mesmo `tx`, chamando `publishWebhookEvent(tx, order, from, to)`. Se a inserção falhar, lançar erro para rollback da transação inteira | **Alterar** | D-01, D-12, C-06; `[09:40]–[09:41] Bruno/Diego` |
| 2 | `src/modules/orders/order.status.ts` | Máquina de estados: `canTransition`, `shouldDebitStock`, `shouldReplenishStock` | Usar `fromStatus`/`toStatus` para montar o evento. Não alterar a máquina de estados; o evento só é gerado em transição válida | **Referenciar** | C-06; `[09:12]–[09:13] Diego/Larissa` |
| 3 | `src/modules/orders/order.controller.ts` | Handler `changeStatus` (linha 38) chama `orders.changeStatus(id, body, user.id)` | Nenhuma mudança: o gatilho continua entrando pelo endpoint existente | **Referenciar** | Código |
| 4 | `src/modules/orders/order.routes.ts` | Rota `PATCH /:id/status` com `authenticate` e `validate` | Nenhuma mudança: a rota existente dispara o service que passa a gravar a outbox | **Referenciar** | Código |
| 5 | `src/shared/errors/app-error.ts` | Classe base `AppError` com `statusCode`, `errorCode` e `details` | As novas classes de erro de webhook devem estender `AppError` (ou subclasses) | **Referenciar** | C-01; `[09:28]–[09:29] Bruno` |
| 6 | `src/shared/errors/http-errors.ts` | Classes concretas como `NotFoundError`, `ConflictError`, `InvalidStatusTransitionError` e `InsufficientStockError` | Não será alterado. Serve de padrão para as novas classes, que ficarão em `src/modules/webhooks/webhook.errors.ts` | **Referenciar** | C-01; `[09:28]–[09:29] Bruno` |
| 7 | `src/middlewares/error.middleware.ts` | Trata `AppError`, `ZodError`, erros conhecidos do Prisma e erro genérico 500 | Reutilizar sem mudança: erros que estendem `AppError` já são mapeados para o status e o código corretos | **Reutilizar** | C-01; `[09:29] Bruno` |
| 8 | `src/middlewares/auth.middleware.ts` | `authenticate` e `requireRole` (linha 49) | Usar `authenticate` nas rotas de CRUD e `requireRole('ADMIN')` no replay de DLQ | **Reutilizar** | C-02; `[09:35]–[09:36] Larissa/Sofia` |
| 9 | `src/middlewares/validate.middleware.ts` | Valida `body`, `query` e `params` com Zod | Usar nos schemas de webhook, incluindo a validação de URL HTTPS | **Reutilizar** | C-09; `[09:23] Sofia` |
| 10 | `src/shared/logger/index.ts` | Logger Pino com redaction de authorization, cookie, password, token etc. | Worker e módulo de webhooks usam o mesmo logger | **Reutilizar** | C-03; `[09:29] Bruno` |
| 11 | `src/config/database.ts` | `createPrismaClient()` e singleton `prisma` | O worker cria sua própria instância de `PrismaClient` no processo, usando o mesmo `DATABASE_URL`; a API mantém a instância atual | **Reutilizar** | C-04; `[09:30] Bruno` |
| 12 | `src/server.ts` | Entry point da API: `buildApp`, `listen`, shutdown e disconnect do Prisma | Serve de referência para criar `src/worker.ts` como entry point separado | **Referenciar** | C-07; `[09:11] Larissa` |
| 13 | `src/app.ts` | Composition root: `buildControllers` instancia repositories/services/controllers; `buildApp` registra JSON, request logger, health, API router, 404 e error middleware | Instanciar `WebhookRepository`, `WebhookService` e `WebhookController`; adicionar o controller ao tipo `Controllers` | **Alterar** | D-10; código |
| 14 | `src/routes/index.ts` | Define o tipo `Controllers` e `buildApiRouter`, que monta `/auth`, `/users`, `/customers`, `/products` e `/orders` | Adicionar o controller de webhooks e montar `/webhooks` e `/admin/webhooks` (o prefixo `/api/v1` vem de `app.ts`) | **Alterar** | D-10; `[09:18] Diego` |
| 15 | `prisma/schema.prisma` | Modelos `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory` e `OrderNumberSequence`; IDs UUID | Propor modelos para configuração de webhook, outbox, histórico de entregas e DLQ. Nomes exatos serão definidos no FDD | **Alterar (proposta)** | D-01, D-05, D-08, D-13; linhas 26, 41, 57, 75, 100, 117 |
| 16 | `src/modules/users/user.routes.ts` | Usa `authenticate` e `requireRole('ADMIN')` (linha 15) nas rotas de usuários | Serve de referência direta para proteger o endpoint de replay de DLQ com `requireRole('ADMIN')` | **Referenciar** | C-02; código |

### Resumo por tipo

| Tipo | Quantidade | Caminhos |
| --- | --- | --- |
| **Alterar** | 4 | `order.service.ts`, `app.ts`, `routes/index.ts`, `prisma/schema.prisma` |
| **Reutilizar** | 5 | `error.middleware.ts`, `auth.middleware.ts`, `validate.middleware.ts`, `logger/index.ts`, `config/database.ts` |
| **Referenciar** | 7 | `order.status.ts`, `order.controller.ts`, `order.routes.ts`, `app-error.ts`, `http-errors.ts`, `server.ts`, `user.routes.ts` |
| **Criar** | — | Ver seção 3 |

---

## 3. Arquivos novos previstos (`Criar`)

Estes caminhos ainda não existem. A tabela separa o que foi decidido na reunião do que segue apenas o padrão do projeto e precisa ser confirmado no FDD.

| Caminho | Papel | Status |
| --- | --- | --- |
| `src/worker.ts` | Entry point do worker, separado da API | **Decidido** em `[09:11] Larissa`; segue o padrão de `src/server.ts` |
| `src/modules/webhooks/webhook.outbox.ts` | Função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` | **Decidido nesta etapa** (ver seção 1) |
| `src/modules/webhooks/webhook.errors.ts` | Classes de erro com códigos `WEBHOOK_*`, estendendo `AppError` | **Decidido nesta etapa** (ver seção 1) |
| `src/modules/webhooks/webhook.worker.ts` **ou** `webhook.processor.ts` | Lógica de processamento do worker | **Alternativa da reunião** em `[09:28] Bruno/Diego`; nome exato a definir no FDD |
| `src/modules/webhooks/webhook.controller.ts` | Controller HTTP do módulo | **Previsto pelo padrão** de `src/modules/orders/*`; confirmar no FDD |
| `src/modules/webhooks/webhook.service.ts` | Regras de negócio do módulo | **Previsto pelo padrão**; confirmar no FDD |
| `src/modules/webhooks/webhook.repository.ts` | Acesso a dados do módulo | **Previsto pelo padrão**; confirmar no FDD |
| `src/modules/webhooks/webhook.routes.ts` | Rotas do módulo | **Previsto pelo padrão**; confirmar no FDD |
| `src/modules/webhooks/webhook.schemas.ts` | Schemas Zod, incluindo validação de HTTPS | **Previsto pelo padrão**; confirmar no FDD |

> A reunião decidiu que o módulo segue o padrão `controller + service + repository + routes + schemas` (`[09:27] Bruno`), mas não definiu todos os nomes de arquivo. Por isso, os nomes acima são derivados do padrão existente, não uma decisão fechada.

---

## 4. O que fica fora desta tabela (por decisão)

| Caminho | Motivo |
| --- | --- |
| `package.json` | O script do worker (`npm run worker`) é uma alteração de configuração, não um ponto de integração de código. Deve ser documentado no FDD. |
| `src/config/env.ts` | Nenhuma variável de ambiente específica de webhook foi decidida na reunião. Se o FDD precisar, isso será registrado lá. |
| `tests/` | A estratégia de testes pertence ao FDD/PRD; não é ponto de integração da feature. |

---

## 5. Observações para o FDD

1. **Posição da chamada no `changeStatus`:** a restrição é que `publishWebhookEvent` rode dentro do mesmo `tx`. A posição exata — depois do update do pedido e da inserção no histórico, antes ou depois da query `refreshed` — deve ser definida no FDD.
2. **Modelos do schema:** os nomes `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery` e `WebhookDeadLetter` são apenas candidatos; não foram decididos na reunião. O FDD deve fechar a modelagem.
3. **Nome do arquivo do worker:** a reunião citou `webhook.worker.ts` ou `webhook.processor.ts`; o FDD deve escolher um.
4. **Redaction de secrets:** o logger atual redige authorization, cookie, password e tokens, mas não secrets de webhook. O FDD deve avaliar se o módulo precisa adicionar essa proteção.
5. **Resposta do `changeStatus`:** o método retorna `OrderWithRelations`; a inserção na outbox não deve alterar o contrato de resposta existente.
6. **`package.json`:** o script `worker` precisa entrar no plano de implementação, mesmo fora da tabela de integração.

---

## 6. Checklist do item 1.1

- [x] 16 caminhos existentes verificados no disco
- [x] Cada caminho tem responsabilidade atual e forma de integração descritas
- [x] Pelo menos 4 caminhos reais referenciados (16 no total)
- [x] Decisões de local de erros e da função de outbox registradas
- [x] Rotas admin definidas como `/api/v1/admin/webhooks/...`
- [x] `prisma/schema.prisma` incluído como proposta
- [x] `package.json`, `env.ts` e `tests/` explicitamente fora da tabela
- [x] Nenhum arquivo de código da aplicação alterado
- [ ] Detalhes de modelagem, posição exata da chamada e nome do processor ficam para o FDD
