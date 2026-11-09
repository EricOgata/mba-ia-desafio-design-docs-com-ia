# ADR-001 — Outbox transacional no MySQL

- **Status:** Aceito
- **Data:** 2026-11-09
- **Decisores:** Diego, Larissa, Bruno
- **Rastreabilidade:** D-01, D-12, D-13, D-17, D-19, D-20; `[09:06] Diego`, `[09:07] Larissa/Diego`, `[09:51]–[09:52] Larissa/Diego`, `[09:33]–[09:34] Marcos/Bruno/Diego`, `[09:51] Larissa`, `[09:43] Diego`, `[09:44] Diego/Sofia`; `src/modules/orders/order.service.ts:126`, `prisma/schema.prisma`

## Contexto

A feature de webhooks de notificação de pedidos exige que um evento seja gerado quando o status de um pedido muda, sem bloquear a transação de mudança de status nem arriscar rollback caso o endpoint do cliente esteja indisponível. A transação atual de `OrderService.changeStatus` já atualiza `orders`, insere em `order_status_history` e gerencia estoque. Incluir uma chamada HTTP síncrona nessa transação acoplaria a mudança de status à disponibilidade externa.

## Decisão

Adotar o padrão de **outbox transacional** no MySQL existente: inserir eventos em uma tabela `webhook_outbox` **dentro da mesma transação** de `changeStatus`, via função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` em `src/modules/webhooks/webhook.outbox.ts`. Um worker separado (ADR-005) lê e entrega os eventos assincronamente.

Detalhes consolidados:
- Snapshot JSON renderizado no momento da inserção (D-12); eventos refletem o estado naquele instante.
- Filtragem de status desejados na inserção (D-17), evitando linhas desnecessárias.
- IDs UUID para linhas da outbox, seguindo `@default(uuid()) @db.Char(36)` (D-13, C-05).
- Payload enxuto, sem itens do pedido (D-19).
- Headers padrão: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type` (D-20).

## Alternativas Consideradas

### Alternativa 1 — Envio síncrono dentro de `changeStatus`
Descartada em `[09:03]–[09:04] Bruno`: bloquearia a transação e arriscaria rollback se o cliente estivesse offline, acoplando a mudança de status à disponibilidade externa.

### Alternativa 2 — Redis Streams
Descartada em `[09:07] Larissa/Diego`: exigiria nova infraestrutura, contrariando a decisão de reuso do MySQL existente.

### Alternativa 3 — Trigger MySQL
Descartada em `[09:09] Diego`: triggers não conseguem notificar um processo externo.

## Consequências

### Positivas
- Nenhuma nova infraestrutura: reuso do MySQL e do Prisma existentes.
- Entrega desacoplada da disponibilidade do cliente.
- Eventos refletem o estado no momento da mudança, garantindo consistência histórica.

### Negativas
- Latência de entrega limitada pelo intervalo de polling do worker (até 2s).
- A tabela de outbox cresce ao longo do tempo; arquivamento adiado (D-22).
- Se `publishWebhookEvent` falhar, toda a transação de `changeStatus` é revertida (C-06), incluindo a mudança de status — acoplamento intencional, mas não óbvio.

## Referências

- `TRANSCRICAO.md` — `[09:06] Diego`, `[09:07] Larissa/Diego`, `[09:51]–[09:52] Larissa/Diego`
- `docs/DECISION-LOG.md` — D-01, D-12, D-13, D-17, D-19, D-20
- `docs/INTEGRATION-POINTS.md` — itens 1, 15
- `src/modules/orders/order.service.ts:126`
- `prisma/schema.prisma`