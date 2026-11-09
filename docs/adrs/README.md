# Architectural Decision Records (ADRs)

Este diretório armazena os ADRs do projeto. Cada decisão arquitetural relevante deve ser registrada em um arquivo individual, com numeração estável e título em kebab-case.

> **Fonte de verdade:** `TRANSCRICAO.md`, `docs/DECISION-LOG.md` e o código-fonte do repositório.
>
> **Plano mestre:** `TODO.md`, Etapa 2.

---

## 1. Convenção de nomes

```text
ADR-NNN-titulo-em-kebab-case.md
```

| Regra | Detalhe |
| --- | --- |
| Prefixo | `ADR-` em maiúsculas |
| Número | `NNN` com três dígitos, começando em `001` |
| Título | minúsculas, sem acentos, palavras separadas por `-` |
| Extensão | `.md` |
| Renumeração | proibida; um ADR nunca muda de número |
| Remoção | proibida; ADR substituído mantém o arquivo e atualiza o status |
| Limite | no máximo 8 ADRs nesta entrega (`TODO.md`, item 2.3) |

Exemplos válidos:

```text
ADR-001-outbox-transacional-no-mysql.md
ADR-006-adocao-dos-padroes-existentes.md
```

---

## 2. Numeração planejada

A ordem abaixo segue a lista das 6 decisões principais do `TODO.md` (item 2.2). O número `006` já foi reservado para o ADR de reuso no item 1.2.

| Número | Arquivo planejado | Decisão coberta | Origem principal | Status |
| --- | --- | --- | --- | --- |
| `ADR-001` | `ADR-001-outbox-transacional-no-mysql.md` | Outbox transacional no MySQL | `D-01`, `[09:06] Diego` | Planejado |
| `ADR-002` | `ADR-002-retry-com-backoff-e-dlq.md` | Retry com backoff e DLQ | `D-04`, `D-05`, `D-06` | Planejado |
| `ADR-003` | `ADR-003-hmac-sha256-com-secret-por-endpoint.md` | Autenticação HMAC-SHA256 com secret por endpoint | `D-07`, `D-08`, `D-28` | Planejado |
| `ADR-004` | `ADR-004-garantia-at-least-once-com-x-event-id.md` | Garantia at-least-once com `X-Event-Id` | `D-09` | Planejado |
| `ADR-005` | `ADR-005-worker-separado-em-polling.md` | Worker separado em polling | `D-02`, `D-03` | Planejado |
| `ADR-006` | `ADR-006-adocao-dos-padroes-existentes.md` | Reuso dos padrões existentes do projeto | `D-10`, `D-11`, `C-01`–`C-09` | Planejado (reservado no item 1.2) |

### 2.1 ADRs secundários opcionais

Se o item 2.3 for executado, os números disponíveis são `ADR-007` e `ADR-008`, sem alterar os seis anteriores. Candidatos citados no `TODO.md`:

- snapshot do payload na inserção (`D-12`, `[09:51] Larissa`);
- IDs UUID (`D-13`, `[09:51] Larissa`);
- DLQ em tabela separada (`D-05`, `[09:18] Diego`);
- timeout de 10s (`D-14`, `[09:42] Diego`);
- limite de payload de 64KB (`D-15`, `[09:23]–[09:24]`).

> Atenção: os candidatos se sobrepõem parcialmente aos ADRs principais. A escolha final deve evitar ADRs duplicados e respeitar o limite de 8 arquivos.

---

## 3. Estrutura mínima de cada ADR

O `TODO.md` exige `Status`, `Contexto`, `Decisão`, `Alternativas Consideradas` (≥1 real) e `Consequências` (positivas e negativas). O template abaixo usa MADR e mantém a rastreabilidade.

```markdown
# ADR-NNN — Título da decisão

- **Status:** Proposto | Aceito | Rejeitado | Substituído | Obsoleto
- **Data:** AAAA-MM-DD
- **Decisores:** nomes dos participantes ou responsáveis
- **Rastreabilidade:** D-xx, [hh:mm] Nome; caminho de código quando aplicável

## Contexto

Situação e forças que motivaram a decisão.

## Decisão

O que foi decidido, em uma frase objetiva, seguida dos detalhes necessários.

## Alternativas Consideradas

### Alternativa 1 — nome

Descrição, motivo da rejeição e trade-off.

## Consequências

### Positivas

- ...

### Negativas

- ...

## Referências

- `TRANSCRICAO.md` — `[hh:mm] Nome`
- `docs/DECISION-LOG.md` — `D-xx`
- caminhos reais do código
```

### Regras de conteúdo

1. Toda alternativa citada precisa ter sido realmente discutida na transcrição ou existir no código.
2. Toda consequência deve ser coerente com a decisão registrada no `docs/DECISION-LOG.md`.
3. Timestamps devem usar o formato `[hh:mm] Nome`.
4. Caminhos de código devem existir no repositório.
5. O ADR não substitui o `docs/TRACKER.md` nem o `docs/DECISION-LOG.md`; ele detalha uma decisão específica.

---

## 4. Estado atual

| Item | Estado |
| --- | --- |
| Pasta `docs/adrs/` | Existe |
| Convenção de nomes | Definida neste README |
| Numeração | Planejada de `ADR-001` a `ADR-006`, com `007`–`008` reservados para itens opcionais |
| ADRs criados | 6 (ADR-001 a ADR-006), concluídos no item 2.2 |
| `ADR-006` | Reservado para o ADR de reuso, conforme decisão do item 1.2 |
