# PRD — Sistema de Webhooks de Notificação de Pedidos

- **Status:** Em revisão
- **Data:** 2026-11-09
- **Autor:** Equipe OMS

## 1. Resumo e contexto

Clientes B2B acompanham pedidos por polling em `GET /orders`. A feature entrega notificações HTTP assíncronas (webhooks outbound) quando o status de um pedido muda, reduzindo latência percebida e carga de polling.

## 2. Problema e motivação

- 3 clientes B2B (Atlas, MaxDistribuição, Nova Cargo) fazem polling em `GET /orders` (`[09:00] Marcos`).
- Risco de churn da Atlas se a experiência não melhorar (`[09:00] Marcos`).
- Latência percebida desejada: < 10s (`[09:02] Marcos`, `[09:10] Marcos`).
- Prazo: 3 sprints, incluindo 2 dias úteis de revisão de segurança (`[09:45]–[09:47] Marcos/Larissa/Sofia`).

## 3. Público-alvo e cenários

- Integradores B2B que consomem status de pedidos.
- Cenário: ao mudar status de um pedido, o cliente recebe POST com snapshot do evento e pode atualizar seu sistema sem polling.

## 4. Objetivos e métricas

- Latência percebida < 10s (`[09:02] Marcos`).
- Entrega at-least-once com dedup via `X-Event-Id`.
- Redução de polling (observar).
- Prazo de 3 sprints.

## 5. Escopo

### Incluso
- CRUD de endpoints de webhook por cliente.
- Filtro por status.
- Histórico das últimas 100 entregas.
- Replay manual de DLQ por ADMIN.
- Assinatura HMAC-SHA256.
- Rotação de secret com grace period 24h.

### Fora de escopo
- Envio de e-mail ao cliente quando o webhook falha (`[09:37] Larissa`).
- Rate limiting de envio por cliente (`[09:39] Diego/Larissa`).
- Dashboard visual/painel (`[09:39]–[09:40] Larissa`).
- Webhooks inbound (`[09:02] Sofia/Marcos`).
- Garantia de ordering global (`[09:12]–[09:13] Diego/Larissa`).
- Exactly-once (`[09:25] Diego`).

## 6. Requisitos funcionais

1. RF-01 Criar configuração de webhook (`POST /webhooks`) retornando secret gerada.
2. RF-02 Listar endpoints por cliente (`GET /webhooks?customer_id=...`).
3. RF-03 Editar URL/status/ativo (`PATCH /webhooks/:id`).
4. RF-04 Remover endpoint (`DELETE /webhooks/:id`).
5. RF-05 Histórico das últimas 100 entregas (`GET /webhooks/:id/deliveries`).
6. RF-06 Replay de DLQ por ADMIN (`POST /admin/webhooks/dead-letter/:id/replay`).
7. RF-07 Assinatura HMAC-SHA256 do corpo com secret por endpoint.
8. RF-08 Rotação de secret com grace period de 24h.
9. RF-09 Filtragem de status desejados na inserção da outbox.
10. RF-10 Timeout de 10s por entrega.

## 7. Requisitos não funcionais

- TLS obrigatório (URL HTTPS) (`[09:23] Sofia`).
- Payload ≤ 64KB (`[09:23]–[09:24]`).
- Timeout 10s (`[09:42] Diego`).
- Latência percebida < 10s (`[09:02] Marcos`).
- At-least-once com `X-Event-Id`.
- Reuso dos padrões existentes do projeto.

## 8. Decisões e trade-offs principais

- Outbox transacional no MySQL vs. envio síncrono/Redis Streams.
- Retry 5x com backoff vs. 3x/indefinido.
- DLQ em tabela separada vs. marcar failed na outbox.
- HMAC por endpoint vs. secret global.
- At-least-once vs. exactly-once.
- Worker separado vs. embutido na API.

## 9. Dependências

- `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126`).
- Prisma schema (novos modelos).
- Auth middleware (`authenticate`, `requireRole`).
- Pino logger.

## 10. Riscos

- Risco 1: acoplamento transacional reverte mudança de status se outbox falhar. Probabilidade média, impacto alto. Mitigação: documentar e testar rollback.
- Risco 2: crescimento da outbox sem arquivamento. Probabilidade alta, impacto médio. Mitigação: monitoramento; arquivamento adiado.
- Risco 3: duplicatas exigem dedup no cliente. Probabilidade média, impacto médio. Mitigação: documentar cenários e fornecer `X-Event-Id`.

## 11. Critérios de aceitação

- Mudança de status gera evento na outbox.
- Worker entrega com headers e HMAC válidos.
- Retry/backoff e DLQ conforme especificado.
- Replay por ADMIN funciona.
- URL não-HTTPS e payload > 64KB rejeitados.

## 12. Estratégia de testes e validação

- Testes unitários do service e outbox.
- Testes de integração do fluxo changeStatus → outbox.
- Testes do worker (retry, DLQ, replay).
- Revisão de segurança de 2 dias úteis.