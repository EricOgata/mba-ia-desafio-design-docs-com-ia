# Relato do processo — Design Docs com IA

## Sobre o desafio

Transformar a transcrição de uma reunião técnica (323 linhas, 09:00–09:53, 5 participantes) e o código de um OMS em um pacote de documentação acionável para a feature **Sistema de Webhooks de Notificação de Pedidos**. Entrega 100% documental: `src/`, `prisma/`, `tests/` e configs não foram alterados.

## Ferramentas de IA utilizadas

- **OpenCode (agentes)**: leitura do repositório e geração dos 27 relatórios em `docs/agents/` (análise arquitetônica, 25 componentes, auditoria de dependências), além de `docs/INTEGRATION-POINTS.md` e `docs/ARCHITECTURE-PATTERNS.md`. Plugins originais do Claude Code foram convertidos para o formato OpenCode e estão em `./.opencode/` (commands e agents), incluindo `adr-map`, `adr-identify`, `generate-architectural-report`, entre outros.
- **Context7 / web**: consulta pontual de padrões (outbox, HMAC, MADR). Usado com parcimônia.
- **IA generativa (LLM)**: redação dos ADRs, RFC, FDD, PRD e Tracker, sempre com exigência de citação de origem.

## Workflow adotado

1. Ler a transcrição integralmente antes de usar IA.
2. Montar `docs/DECISION-LOG.md` como fonte única de verdade (29 decisões + 9 restrições de código).
3. Mapear integração real (`docs/INTEGRATION-POINTS.md`) e padrões (`docs/ARCHITECTURE-PATTERNS.md`).
4. Gerar ADRs (6) a partir de potential-ADRs pré-analisados.
5. Consolidar RFC (arquitetura) → FDD (detalhe) → PRD (produto).
6. Fechar com Tracker e README.

## Prompts customizados

```text
Leia TRANSCRICAO.md e produza uma tabela com: tema, decisão, status
(fechada / adiada / descartada), timestamp e falante. Não inferir nada
que não esteja explícito. Liste separadamente tudo que foi descartado.
```

```text
Com base apenas em TRANSCRICAO.md e nos arquivos citados, escreva o FDD.
Para cada requisito, decisão ou restrição, cite o timestamp e o falante
ou o caminho do arquivo. Não invente endpoints, métricas ou limites.
```

```text
Revise o texto abaixo contra TRANSCRICAO.md. Aponte contradições,
itens sem origem e requisitos que na verdade foram adiados ou
descartados. Não reescreva ainda; apenas liste os problemas.
```

## Iterações e ajustes

- A IA incluiu envio de e-mail como requisito no PRD; removi porque `[09:37] Larissa` diz fora de escopo.
- A IA inventou um endpoint de dashboard; corrigi, pois `[09:39]–[09:40] Larissa` trata dashboard como projeto separado.
- A IA sugeriu exactly-once; rejeitei em favor de at-least-once (`[09:25] Diego`).
- Consolidou snapshot/UUID/DLQ/timeout/payload nos ADRs principais para evitar duplicação (limite de 8 ADRs).

## Como navegar a entrega

Ordem de leitura:
1. `README.md` (este arquivo) — processo.
2. `docs/PRD.md` — por que e o quê.
3. `docs/RFC.md` — proposta e questões em aberto.
4. `docs/adrs/` — decisões pontuais (ADR-001…ADR-006).
5. `docs/FDD.md` — como construir em detalhe.
6. `docs/TRACKER.md` — de onde veio cada item.

Arquivos de apoio: `docs/DECISION-LOG.md`, `docs/INTEGRATION-POINTS.md`, `docs/ARCHITECTURE-PATTERNS.md`, `docs/RESUMO-TRANSCRICAO.md`, `docs/PLANO-EXECUCAO.md`, `docs/PROCESS-LOG.md`, `TRANSCRICAO.md`.