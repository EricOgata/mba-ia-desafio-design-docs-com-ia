# ADR-002 — Retry com backoff e DLQ

- **Status:** Aceito
- **Data:** 2026-11-09
- **Decisores:** Diego, Bruno, Marcos
- **Rastreabilidade:** D-04, D-05, D-06; `[09:15]–[09:17] Diego`, `[09:17] Marcos`, `[09:18] Diego`, `[09:18] Bruno`, `[09:35]–[09:36] Larissa/Sofia`; `src/middlewares/auth.middleware.ts:49`, `prisma/schema.prisma`

## Contexto

Endpoints de clientes podem estar temporariamente indisponíveis. É necessário uma política de reentrega que dê tempo ao cliente para se recuperar, mas que não deixe eventos presos indefinidamente, além de um mecanismo de falha definitiva com reprocessamento controlado.

## Decisão

Adotar **5 tentativas** com backoff fixo `1m / 5m / 30m / 2h / 12h` (janela total ≈15h). Após esgotar as tentativas, o evento é movido para uma tabela separada `webhook_dead_letter` (DLQ). O reprocessamento é **manual**, via `POST /api/v1/admin/webhooks/dead-letter/:id/replay`, exigindo `requireRole('ADMIN')` e auditoria.

## Alternativas Consideradas

### Alternativa 1 — 3 tentativas
Considerada insuficiente pelo time; descartada em `[09:15]–[09:17] Diego`.

### Alternativa 2 — Retry indefinido
Descartada em `[09:15] Diego`: eventos ficariam presos para sempre, sem sinalização de falha.

### Alternativa 3 — Marcar como `failed` na própria outbox
Descartada em `[09:18] Diego`: manteria a outbox poluída com eventos terminais; tabela separada mantém a outbox focada em pendentes/em andamento.

## Consequências

### Positivas
- Janela de ~15h dá tempo ao cliente para se recuperar de indisponibilidade.
- Outbox permanece limpa; DLQ isola falhas definitivas.
- Replay manual dá controle ao operador, com autorização `ADMIN`.

### Negativas
- Falhas não são reportadas imediatamente (janela longa).
- Reprocessamento exige intervenção humana; notificação por e-mail foi adiada (D-23).
- Sequência de backoff é customizada, não uma fórmula exponencial padrão.

## Referências

- `TRANSCRICAO.md` — `[09:15]–[09:17] Diego`, `[09:17] Marcos`, `[09:18] Diego/Bruno`, `[09:35]–[09:36] Larissa/Sofia`
- `docs/DECISION-LOG.md` — D-04, D-05, D-06
- `docs/INTEGRATION-POINTS.md` — item 15, item 8
- `src/middlewares/auth.middleware.ts:49`