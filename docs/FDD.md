# FDD — Sistema de Webhooks de Notificação de Pedidos

- **Status:** Em revisão
- **Data:** 2026-11-09
- **Rastreabilidade:** `docs/DECISION-LOG.md`, `docs/INTEGRATION-POINTS.md`, `docs/ARCHITECTURE-PATTERNS.md`, `docs/adrs/ADR-001`…`ADR-006`, `TRANSCRICAO.md`

## 1. Visão geral

Feature outbound de notificação: quando o status de um pedido muda, um evento é inserido na outbox transacional e entregue via HTTP pelo worker, com retry/backoff e DLQ, autenticado por HMAC-SHA256.

## 2. Fluxos técnicos

### 2.1 Criação do evento na outbox

1. `OrderService.changeStatus` executa dentro de `$transaction` (`src/modules/orders/order.service.ts:126`).
2. Atualiza `orders`, insere `order_status_history`, gerencia estoque.
3. Chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` (`src/modules/webhooks/webhook.outbox.ts`, a criar) na mesma transação.
4. Função verifica se algum endpoint ativo assina o status; se não, skip silencioso.
5. Gera UUID (`X-Event-Id`), renderiza snapshot JSON (payload enxuto, sem itens), insere em `webhook_outbox`.
6. Se qualquer passo falhar, a transação inteira é revertida (C-06).

### 2.2 Processamento pelo worker

1. Worker (`src/worker.ts`) faz polling a cada 2s, oldest-first.
2. Para cada evento pendente com `next_retry_at <= now`: marca `in_progress`, monta headers, envia HTTP POST com timeout 10s.
3. Em sucesso (2xx): marca `delivered`, registra entrega.
4. Em falha: incrementa tentativa, calcula próximo backoff; se tentativas esgotadas, move para `webhook_dead_letter` e remove da outbox.

### 2.3 Retry com backoff

Tentativas: 5. Intervalos: `1m, 5m, 30m, 2h, 12h`. Janela total ≈15h.

### 2.4 DLQ e replay

Evento esgotado → `webhook_dead_letter`. Replay via `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (role `ADMIN`), movendo atomicamente de volta para a outbox como `pending`.

## 3. Contratos públicos

Base: `/api/v1`. Auth: `authenticate` (exceto webhook delivery, que é outbound). CRUD autenticado.

### 3.1 `POST /webhooks`

Request:
```json
{ "customer_id": "uuid", "url": "https://example.com/hook", "events": ["order.shipped","order.delivered"], "active": true }
```
Response `201`:
```json
{ "id": "uuid", "customer_id": "uuid", "url": "https://...", "events": ["..."], "active": true, "secret": "whsec_...", "created_at": "..." }
```
A secret é retornada apenas na criação.

### 3.2 `GET /webhooks?customer_id=...`

Response `200`: lista de endpoints (sem secret).

### 3.3 `PATCH /webhooks/:id`

Request: `{ "url": "...", "active": false, "events": [...] }`. Response `200` com recurso atualizado.

### 3.4 `DELETE /webhooks/:id`

Response `204`.

### 3.5 `GET /webhooks/:id/deliveries`

Response `200`: últimas 100 entregas (`event_id`, `status`, `attempt`, `http_status`, `latency_ms`, `created_at`).

### 3.6 `POST /admin/webhooks/dead-letter/:id/replay`

Auth: `ADMIN`. Response `202` com id do evento re-enfileirado. Erros: `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404).

### 3.7 Rotação de secret (D-28, em aberto)

Contrato a definir; requisito: gerar nova secret, manter a antiga válida por 24h, retornar a nova.

### 3.8 Headers de entrega

`Content-Type: application/json`, `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`.

Payload de exemplo:
```json
{ "event_id": "uuid", "event_type": "order.shipped", "order_id": "uuid", "from_status": "processing", "to_status": "shipped", "occurred_at": "2026-11-09T09:00:00Z" }
```

## 4. Matriz de erros `WEBHOOK_*`

| Código | HTTP | Causa |
| --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint inexistente |
| `WEBHOOK_INVALID_URL` | 400 | URL inválida ou não-HTTPS (validação Zod) |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Secret ausente |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 413 | Payload > 64KB |
| `WEBHOOK_DELIVERY_TIMEOUT` | 504 | Timeout de 10s na entrega |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Item de DLQ inexistente |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | Filtro de status inválido |

Todos estendem `AppError` (`src/shared/errors/app-error.ts`) e são tratados pelo error middleware (`src/middlewares/error.middleware.ts`).

## 5. Resiliência

- Timeout de 10s por entrega (`[09:42] Diego`).
- Retry/backoff conforme seção 2.3.
- DLQ após 5 falhas.
- Isolamento por processo: worker crash não derruba a API.
- At-least-once: duplicatas possíveis; dedup é do cliente.

## 6. Observabilidade

- Métricas: eventos pendentes, entregues, falhos, latência de entrega, tamanho da DLQ.
- Logs estruturados via Pino (`src/shared/logger/index.ts`), com correlação por `X-Event-Id`.
- Tracing: `X-Event-Id` propagado como correlação end-to-end.
- Lacuna: logger não redige secrets de webhook — adicionar paths de redação.

## 7. Integração com o sistema existente

- `src/modules/orders/order.service.ts:126` → inserir `publishWebhookEvent(tx, order, fromStatus, toStatus)` na mesma transação.
- `src/shared/errors/app-error.ts` e `http-errors.ts` → estender `AppError` para erros `WEBHOOK_*`.
- `src/middlewares/error.middleware.ts` → tratamento automático dos novos erros.
- `src/middlewares/auth.middleware.ts:49` → reutilizar `authenticate` e `requireRole('ADMIN')` para replay.
- `src/middlewares/validate.middleware.ts` → validação Zod, incluindo HTTPS.
- `src/shared/logger/index.ts` → logger do worker e redação de secrets.
- `src/server.ts` → referência para criar `src/worker.ts`.
- `src/app.ts` → registrar `WebhookController` em `buildControllers`.
- `src/routes/index.ts` → montar `/webhooks` e `/admin/webhooks`.
- `prisma/schema.prisma` → novos modelos `WebhookEndpoint`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter` (UUID).
- `src/config/database.ts` → worker cria própria instância de `PrismaClient`.

Arquivos novos: `src/modules/webhooks/webhook.{controller,service,repository,routes,schemas,errors,outbox,worker}.ts`.

## 8. Critérios de aceite técnicos

- Mudança de status gera evento na outbox na mesma transação.
- Worker entrega com headers corretos e HMAC válido.
- Retry segue backoff; DLQ após 5 falhas.
- Replay por ADMIN re-enfileira.
- URL não-HTTPS rejeitada na validação.
- Payload > 64KB rejeitado.

## 9. Riscos e mitigações

- Acoplamento transacional → documentado; testes de rollback.
- Crescimento da outbox → monitoramento; arquivamento adiado.
- Duplicatas → dedup no cliente via `X-Event-Id`.
- Logger expõe secrets → adicionar redação.

## 10. Fora de escopo

E-mail de falha, rate limiting, dashboard, webhooks inbound, ordering global, exactly-once.