# Tracker de rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | PRD | Requisito funcional | Criar configuração de webhook | TRANSCRICAO | `[09:27] Bruno` |
| PRD-FR-05 | PRD | Requisito funcional | Histórico das últimas 100 entregas | TRANSCRICAO | `[09:40] Diego` |
| PRD-FR-06 | PRD | Requisito funcional | Replay de DLQ por ADMIN | TRANSCRICAO | `[09:18] Diego`, `[09:35] Larissa/Sofia` |
| PRD-FR-07 | PRD | Requisito funcional | Assinatura HMAC-SHA256 | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-08 | PRD | Requisito funcional | Rotação de secret com grace 24h | TRANSCRICAO | `[09:21]–[09:22] Sofia/Diego` |
| PRD-RNF-01 | PRD | Requisito não funcional | URL obrigatoriamente HTTPS | TRANSCRICAO | `[09:23] Sofia` |
| PRD-RNF-02 | PRD | Requisito não funcional | Payload ≤ 64KB | TRANSCRICAO | `[09:23]–[09:24]` |
| PRD-RNF-03 | PRD | Requisito não funcional | Timeout 10s | TRANSCRICAO | `[09:42] Diego` |
| PRD-RNF-04 | PRD | Requisito não funcional | Latência percebida < 10s | TRANSCRICAO | `[09:02] Marcos`, `[09:10] Marcos` |
| RFC-ALT-01 | RFC | Alternativa descartada | Envio síncrono no changeStatus | TRANSCRICAO | `[09:03]–[09:04] Bruno` |
| RFC-ALT-02 | RFC | Alternativa descartada | Redis Streams | TRANSCRICAO | `[09:07] Larissa/Diego` |
| RFC-ALT-03 | RFC | Alternativa descartada | Trigger MySQL | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-04 | RFC | Alternativa descartada | Retry indefinido | TRANSCRICAO | `[09:15] Diego` |
| RFC-ALT-05 | RFC | Alternativa descartada | Exactly-once | TRANSCRICAO | `[09:25] Diego` |
| RFC-OQ-01 | RFC | Questão em aberto | Rate limiting por cliente | TRANSCRICAO | `[09:39] Diego/Larissa` |
| RFC-OQ-02 | RFC | Questão em aberto | E-mail de falha | TRANSCRICAO | `[09:37] Larissa` |
| RFC-OQ-03 | RFC | Questão em aberto | Ordering global | TRANSCRICAO | `[09:12]–[09:13] Diego/Larissa` |
| RFC-OQ-04 | RFC | Questão em aberto | Arquivamento após 30 dias | TRANSCRICAO | `[09:08]` |
| ADR-001 | ADR | Decisão | Outbox transacional no MySQL | TRANSCRICAO/CODIGO | `[09:06] Diego`; `src/modules/orders/order.service.ts:126` |
| ADR-002 | ADR | Decisão | Retry 5x backoff + DLQ + replay | TRANSCRICAO/CODIGO | `[09:15]–[09:18] Diego`; `src/middlewares/auth.middleware.ts:49` |
| ADR-003 | ADR | Decisão | HMAC-SHA256 secret por endpoint | TRANSCRICAO/CODIGO | `[09:20]–[09:23] Sofia`; `src/middlewares/validate.middleware.ts` |
| ADR-004 | ADR | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | `[09:24]–[09:26] Diego/Sofia` |
| ADR-005 | ADR | Decisão | Worker separado polling 2s | TRANSCRICAO/CODIGO | `[09:09]–[09:11] Diego/Larissa`; `src/server.ts`, `src/config/database.ts` |
| ADR-006 | ADR | Decisão | Reuso dos padrões existentes | TRANSCRICAO/CODIGO | `[09:27]–[09:30] Bruno/Larissa`; `src/modules/orders/*`, `src/app.ts`, `src/routes/index.ts` |
| FDD-CONTRATO-01 | FDD | Contrato | POST /webhooks | TRANSCRICAO | `[09:27] Bruno` |
| FDD-CONTRATO-06 | FDD | Contrato | Replay endpoint ADMIN | TRANSCRICAO | `[09:18] Diego`, `[09:35] Larissa/Sofia` |
| FDD-ERRO-01 | FDD | Erro | WEBHOOK_INVALID_URL (HTTPS) | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-02 | FDD | Erro | WEBHOOK_PAYLOAD_TOO_LARGE | TRANSCRICAO | `[09:23]–[09:24]` |
| FDD-INT-01 | FDD | Integração | publishWebhookEvent em changeStatus | CODIGO | `src/modules/orders/order.service.ts:126` |
| FDD-INT-02 | FDD | Integração | AppError/WEBHOOK_* | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-03 | FDD | Integração | authenticate/requireRole | CODIGO | `src/middlewares/auth.middleware.ts:49` |
| FDD-INT-04 | FDD | Integração | error middleware | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-05 | FDD | Integração | logger Pino | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-06 | FDD | Integração | server.ts → worker.ts | CODIGO | `src/server.ts` |
| FDD-INT-07 | FDD | Integração | app.ts composition root | CODIGO | `src/app.ts` |
| FDD-INT-08 | FDD | Integração | routes/index.ts | CODIGO | `src/routes/index.ts` |
| FDD-INT-09 | FDD | Integração | schema.prisma UUID | CODIGO | `prisma/schema.prisma` |