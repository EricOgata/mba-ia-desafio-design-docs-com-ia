# ADR-004 — Garantia at-least-once com `X-Event-Id`

- **Status:** Aceito
- **Data:** 2026-11-09
- **Decisores:** Diego, Sofia
- **Rastreabilidade:** D-09, D-20, D-25; `[09:24]–[09:26] Diego`, `[09:25] Sofia`, `[09:44] Diego/Sofia`; `src/modules/webhooks/webhook.outbox.ts`, `prisma/schema.prisma`

## Contexto

A semântica de entrega precisa ser clara para clientes integradores. Exactly-once exigiria coordenação bidirecional (idempotência, ACKs, estado em ambos os lados), aumentando a complexidade para a plataforma e para o cliente. O time optou por uma garantia mais simples e padrão de mercado.

## Decisão

Garantir **at-least-once delivery**: todo evento inserido na outbox será entregue ao menos uma vez, mas duplicatas são possíveis. A deduplicação é **responsabilidade do cliente**, usando o header `X-Event-Id` (UUID gerado na inserção da outbox). Exactly-once foi explicitamente descartado.

Duplicatas são mais prováveis em retries, crash do worker após entrega mas antes da atualização de status, e replay da DLQ.

## Alternativas Consideradas

### Alternativa 1 — Exactly-once
Descartada em `[09:25] Diego`: exigiria coordenação (idempotency keys, acknowledgments, gerenciamento de estado em ambos os lados) com complexidade significativa para plataforma e clientes.

## Consequências

### Positivas
- Implementação mais simples, sem necessidade de ACK do cliente.
- Padrão de mercado (Stripe, GitHub, Shopify usam at-least-once).
- `X-Event-Id` permite dedup eficiente no lado do cliente.

### Negativas
- Clientes devem implementar handlers idempotentes ou rastrear IDs vistos.
- Duplicatas possíveis em cenários de falha parcial.
- Sem garantia de ordering global (D-29 descartada).

## Referências

- `TRANSCRICAO.md` — `[09:24]–[09:26] Diego`, `[09:25] Sofia`, `[09:44] Diego/Sofia`
- `docs/DECISION-LOG.md` — D-09, D-20, D-25
- `docs/INTEGRATION-POINTS.md` — item 3, item 15
- `src/modules/webhooks/webhook.outbox.ts`