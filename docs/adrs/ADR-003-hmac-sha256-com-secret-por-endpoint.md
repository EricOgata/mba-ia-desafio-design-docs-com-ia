# ADR-003 — HMAC-SHA256 com secret por endpoint

- **Status:** Aceito
- **Data:** 2026-11-09
- **Decisores:** Sofia, Diego
- **Rastreabilidade:** D-07, D-08, D-16, D-28, D-27; `[09:20] Sofia`, `[09:21]–[09:22] Sofia/Diego`, `[09:23] Sofia`, `[09:45]–[09:47] Marcos/Larissa/Sofia`; `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `prisma/schema.prisma`

## Contexto

Clientes precisam verificar que as notificações recebidas realmente originam da plataforma e não foram alteradas em trânsito. É necessário um mecanismo de autenticação de payload com gestão de segredo por endpoint, rotação segura e transporte obrigatório em TLS.

## Decisão

Usar **HMAC-SHA256** sobre o corpo bruto da requisição, entregue no header `X-Signature`. Cada endpoint possui **secret única**, rotacionável com **grace period de 24h** (ambas as secrets válidas durante a rotação). URLs de endpoint são **obrigatoriamente HTTPS**; `http` é recusado na validação Zod (`src/modules/webhooks/webhook.schemas.ts`). A rota de rotação de secret (D-28) permanece em aberto.

## Alternativas Consideradas

### Alternativa 1 — Secret global da plataforma
Descartada em `[09:21]–[09:22] Sofia`: um único vazamento comprometeria todos os endpoints. Per-endpoint limita o raio de explosão.

### Alternativa 2 — Sem grace period na rotação
Descartada: clientes precisariam atualizar a verificação instantaneamente, arriscando perda de eventos durante a transição.

## Consequências

### Positivas
- Padrão de mercado para webhooks; verificação simples para clientes.
- Blast radius limitado a um endpoint por secret.
- Transporte seguro garantido por validação Zod (C-09).

### Negativas
- Gestão de secrets por endpoint aumenta complexidade (geração, armazenamento, rotação).
- Grace period de 24h cria janela com duas secrets válidas.
- Logger atual (`src/shared/logger/index.ts`) não redige secrets de webhook — lacuna a tratar no FDD.

## Referências

- `TRANSCRICAO.md` — `[09:20] Sofia`, `[09:21]–[09:22] Sofia/Diego`, `[09:23] Sofia`, `[09:45]–[09:47] Marcos/Larissa/Sofia`
- `docs/DECISION-LOG.md` — D-07, D-08, D-16, D-28, D-27
- `docs/INTEGRATION-POINTS.md` — item 9, item 15
- `src/middlewares/validate.middleware.ts`
- `src/shared/logger/index.ts`