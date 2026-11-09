# ADR-005 — Worker separado em polling

- **Status:** Aceito
- **Data:** 2026-11-09
- **Decisores:** Diego, Larissa
- **Rastreabilidade:** D-02, D-03, D-29; `[09:09] Diego`, `[09:11] Diego/Larissa`, `[09:12]–[09:13] Diego/Larissa`, `[09:10] Marcos`; `src/config/database.ts`, `src/server.ts`

## Contexto

A entrega de webhooks é assíncrona e não deve ser bloqueada nem interrompida por reinícios da API. É a primeira vez que o sistema rodará mais de um processo Node.js. Precisa-se de um mecanismo de detecção de novos eventos sem infraestrutura adicional.

## Decisão

Executar um **worker em processo separado** (`src/worker.ts`), que cria sua própria instância de `PrismaClient` (C-04) usando o mesmo `DATABASE_URL`, segue o padrão de bootstrap/shutdown de `src/server.ts` (C-07) e faz **polling a cada 2s** na tabela `webhook_outbox`, processando eventos oldest-first. Modelo **single-worker**, com ordering implícita por `order_id`. Ordering global não é garantido (D-29 descartada).

## Alternativas Consideradas

### Alternativa 1 — Worker dentro da API
Descartada em `[09:11] Diego/Larissa`: reinícios da API interromperiam o processamento de entrega.

### Alternativa 2 — MySQL trigger
Descartada em `[09:09] Diego`: triggers não conseguem notificar um processo externo.

### Alternativa 3 — Múltiplos workers com partitioning/locking
Adiada em `[09:12]–[09:13] Diego/Larissa`: adicionaria complexidade de ordering e locking; single-worker é o ponto de partida consciente.

## Consequências

### Positivas
- Isolamento de processos: falha do worker não derruba a API e vice-versa.
- Sem nova infraestrutura: polling sobre MySQL existente.
- Simplicidade operacional inicial (sem distributed locking).

### Negativas
- Overhead operacional: dois processos para deploy, monitorar e reiniciar.
- Polling adiciona até 2s de latência sobre o tempo de entrega.
- Single-worker impede escalonamento horizontal de throughput de entrega.
- Exige instância própria de `PrismaClient` e graceful shutdown.

## Referências

- `TRANSCRICAO.md` — `[09:09] Diego`, `[09:10] Marcos`, `[09:11] Diego/Larissa`, `[09:12]–[09:13] Diego/Larissa`
- `docs/DECISION-LOG.md` — D-02, D-03, D-29
- `docs/INTEGRATION-POINTS.md` — itens 11, 12
- `src/config/database.ts`
- `src/server.ts`