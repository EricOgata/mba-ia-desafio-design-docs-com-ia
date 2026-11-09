# RFC — Sistema de Webhooks de Notificação de Pedidos

- **Status:** Em revisão
- **Data:** 2026-11-09
- **Autor:** Equipe OMS
- **Revisores:** Larissa, Marcos, Bruno, Diego, Sofia
- **Rastreabilidade:** D-01–D-29, `docs/DECISION-LOG.md`, `docs/adrs/ADR-001` … `ADR-006`

## TL;DR

Substituir o polling feito por clientes B2B (`GET /orders`) por notificações HTTP assíncronas via **outbox transacional no MySQL**, entregues por um **worker separado** com retry/backoff e DLQ, autenticadas com **HMAC-SHA256 por endpoint** e garantia **at-least-once** com `X-Event-Id`. A solução reutiliza os padrões existentes do projeto e não introduz nova infraestrutura.

## Contexto e problema

Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) fazem polling em `GET /orders` para acompanhar mudanças de status. A Atlas corre risco de churn se a experiência não melhorar (`[09:00] Marcos`). A latência percebida desejada é inferior a 10s (`[09:02] Marcos`, `[09:10] Marcos`). O escopo é estritamente **outbound** (plataforma → cliente); webhooks inbound ficam fora de escopo (`[09:02] Sofia/Marcos`).

## Proposta técnica (visão geral)

1. **Outbox transacional**: ao mudar o status de um pedido, `OrderService.changeStatus` insere um evento em `webhook_outbox` na mesma transação (ADR-001). O payload é um snapshot JSON renderizado na inserção, enxuto (sem itens), com IDs UUID.
2. **Worker separado**: processo `src/worker.ts` faz polling a cada 2s, oldest-first, single-worker, com ordering implícita por `order_id` (ADR-005).
3. **Retry e DLQ**: 5 tentativas com backoff `1m/5m/30m/2h/12h`; após esgotar, evento vai para `webhook_dead_letter`, com replay manual por `ADMIN` (ADR-002).
4. **Autenticação**: HMAC-SHA256 sobre o corpo, secret única por endpoint, rotação com grace period de 24h, HTTPS obrigatório (ADR-003).
5. **Garantia**: at-least-once com `X-Event-Id` para dedup no cliente (ADR-004).
6. **Reuso**: módulo `src/modules/webhooks` segue os padrões existentes (`controller/service/repository/routes/schemas`, `AppError`, Pino, error middleware, `authenticate`/`requireRole`) (ADR-006).

## Alternativas consideradas

- **Envio síncrono no `changeStatus`** (`[09:03]–[09:04] Bruno`): bloquearia a transação e arriscaria rollback se o cliente estivesse offline. Rejeitada.
- **Redis Streams** (`[09:07] Larissa/Diego`): exigiria nova infraestrutura. Rejeitada em favor do MySQL existente.
- **Trigger MySQL** (`[09:09] Diego`): não consegue notificar processo externo. Rejeitada.
- **Retry indefinido** (`[09:15] Diego`): eventos ficariam presos para sempre. Rejeitada em favor de 5 tentativas + DLQ.
- **Exactly-once** (`[09:25] Diego`): exigiria coordenação bidirecional. Rejeitada em favor de at-least-once.

## Questões em aberto

- Rate limiting de envio por cliente — observar e decidir depois (`[09:39] Diego/Larissa`).
- Notificação por e-mail ao cliente quando o webhook falha — próxima fase (`[09:37] Larissa`).
- Ordering global — descartado; apenas por `order_id` enquanto single-worker (`[09:12]–[09:13] Diego/Larissa`).
- Arquivamento de linhas entregues após 30 dias — adiado (`[09:08]`).
- Rota de rotação de secret ainda sem contrato definido (D-28, `[09:21] Sofia`).
- Dashboard visual/painel — projeto separado do frontend (`[09:39]–[09:40] Larissa`).

## Impacto e riscos

- **Impacto operacional**: primeira implantação multi-processo (API + worker); requer monitoramento e restart de dois processos.
- **Risco — acoplamento transacional**: se `publishWebhookEvent` falhar, a mudança de status é revertida (intencional, mas não óbvio). Mitigação: documentar no FDD.
- **Risco — crescimento da outbox**: sem arquivamento, a tabela cresce. Mitigação: adiar com monitoramento.
- **Risco — duplicatas**: clientes devem implementar dedup. Mitigação: documentar cenários no FDD e fornecer `X-Event-Id`.
- **Risco — segurança**: secrets não redigidos pelo logger atual. Mitigação: revisão de segurança de 2 dias úteis dentro das 3 sprints (`[09:45]–[09:47] Marcos/Larissa/Sofia`).

## Links

- ADR-001 — Outbox transacional no MySQL
- ADR-002 — Retry com backoff e DLQ
- ADR-003 — HMAC-SHA256 com secret por endpoint
- ADR-004 — Garantia at-least-once com `X-Event-Id`
- ADR-005 — Worker separado em polling
- ADR-006 — Adoção dos padrões existentes

`docs/DECISION-LOG.md`, `docs/INTEGRATION-POINTS.md`, `docs/ARCHITECTURE-PATTERNS.md`.