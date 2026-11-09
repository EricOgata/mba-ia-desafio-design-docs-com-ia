# Plano de Execução — Design Docs com IA

> **Objetivo:** transformar a transcrição de uma reunião técnica + o código de um OMS em um pacote de documentação acionável para a feature **Sistema de Webhooks de Notificação de Pedidos**.
>
> **Regra de ouro:** todo requisito, decisão ou restrição precisa ser rastreável a `TRANSCRICAO.md` (com timestamp e falante) ou a um arquivo real do código. Se não dá para apontar a origem, o item provavelmente foi inventado e deve ser removido.
>
> **Restrição absoluta:** não alterar `src/`, `prisma/`, `tests/` nem configurações da aplicação. A entrega é 100% documental.

---

## Como usar este arquivo

- Marque `[x]` ao concluir cada item.
- Cada etapa tem **Objetivo**, **Como resolver**, **Saída esperada** e **Pronto quando**.
- A ordem sugerida não é burocrática: os ADRs formam o esqueleto, o RFC consolida as decisões, o FDD detalha a implementação e o PRD fecha no nível de produto. O Tracker e o README vêm no fim.
- Considere usar IA em ciclos: gerar → revisar criticamente → corrigir prompt → gerar de novo. São esperados de 3 a 5 ciclos.

### Legenda

| Símbolo | Significado |
| --- | --- |
| `[ ]` | Pendente |
| `[~]` | Em andamento |
| `[x]` | Concluído |

---

## Diagnóstico inicial do repositório

| Item | Estado atual | Ação necessária |
| --- | --- | --- |
| `TRANSCRICAO.md` | Completa (323 linhas, 09:00–09:53) | Apenas ler; **não alterar** |
| `README.md` | Ainda é o enunciado original | Substituir pelo relato do processo |
| `docs/PRD.md` | Placeholder (3 linhas) | Produzir documento completo |
| `docs/RFC.md` | Placeholder (3 linhas) | Produzir documento completo |
| `docs/FDD.md` | Placeholder (3 linhas) | Produzir documento completo |
| `docs/TRACKER.md` | Placeholder (3 linhas) | Produzir tabela de rastreabilidade |
| `docs/adrs/` | Só contém `README.md` | Criar de 5 a 8 ADRs |
| `docs/agents/` | 27 relatórios de análise já gerados | Usar como apoio de leitura do código |
| `README_2.md` | Log auxiliar do que já foi analisado | Apoio opcional; não substitui o README |
| `src/`, `prisma/`, `tests/` | Intocados | Usar apenas como referência |

**Progresso estimado:** 45% (Etapas 0, 1.1, 2.1 e 2.2 concluídas; ADR-006 criado, concluindo o item 1.2; documentos de entrega ainda não produzidos).

---

## Mapa rápido das decisões da reunião (para não se perder)

| Tema | Decisão fechada | Origem |
| --- | --- | --- |
| Padrão de entrega | Outbox transacional no MySQL existente | `[09:06] Diego`, `[09:48] Larissa` |
| Processamento | Worker em processo separado, polling a cada 2s | `[09:09] Diego`, `[09:11] Diego` |
| Retry | 5 tentativas, backoff 1m/5m/30m/2h/12h | `[09:15]–[09:17] Diego` |
| Falha definitiva | DLQ em tabela separada + replay manual | `[09:18] Diego` |
| Segurança | HMAC-SHA256 do corpo, secret única por endpoint, rotação com grace period de 24h | `[09:20]–[09:22] Sofia` |
| Garantia de entrega | At-least-once com `X-Event-Id` para dedup no cliente | `[09:24]–[09:26] Diego` |
| Estrutura | Módulo `src/modules/webhooks` + `src/worker.ts` | `[09:27]–[09:28] Bruno` |
| Erros/logs | Prefixo `WEBHOOK_`, reuso de `AppError`, Pino e error middleware | `[09:28]–[09:30] Bruno/Larissa` |
| Payload | Snapshot JSON renderizado na inserção da outbox | `[09:51]–[09:52] Larissa/Diego` |
| Autorização | CRUD autenticado; replay de DLQ exige `ADMIN` via `requireRole` | `[09:35]–[09:36] Larissa/Sofia` |

### Fora de escopo / adiado (não transformar em requisito)

- Envio de e-mail ao cliente quando o webhook falha → próxima fase (`[09:37] Larissa`).
- Rate limiting de envio por cliente → observar e decidir depois (`[09:39] Diego/Larissa`).
- Dashboard visual / painel → projeto separado do frontend (`[09:39]–[09:40] Larissa`).
- Webhooks inbound (cliente → plataforma) → apenas outbound (`[09:02] Sofia/Marcos`).
- Garantia de ordering global → só por `order_id` e enquanto single-worker (`[09:12]–[09:13] Diego/Larissa`).
- Exactly-once → descartado em favor de at-least-once (`[09:25] Diego`).

---

# Etapas

## Etapa 0 — Setup e contextualização

- [x] **0.1 Fork e clone do repositório base**
  - Objetivo: garantir que a entrega será feita em repositório público próprio.
  - Como resolver: faça fork de `devfullcycle/mba-ia-desafio-design-docs-com-ia`, clone localmente e confirme que a branch está limpa.
  - Pronto quando: `git remote -v` aponta para o seu fork.
  - **Concluído:** `origin` = `git@github.com:EricOgata/mba-ia-desafio-design-docs-com-ia.git`; `upstream` = `devfullcycle/mba-ia-desafio-design-docs-com-ia`.

- [x] **0.2 Ler a transcrição inteira, sem pedir nada à IA ainda**
  - Objetivo: formar seu próprio entendimento antes de aceitar o resumo da IA.
  - Como resolver: leia `TRANSCRICAO.md` de ponta a ponta e anote, em rascunho, decisões, requisitos, descartes e dúvidas.
  - Saída: lista pessoal de fatos com timestamps.
  - Pronto quando: você consegue explicar em 1 minuto o que a feature faz e o que **não** faz.
  - **Concluído:** `TRANSCRICAO.md` lida integralmente (323 linhas, 09:00–09:53), incluindo os 5 participantes e os encerramentos pós-reunião (`[09:50]`–`[09:53]`). Resumo de apoio criado em `TRANSCRICAO_RESUMO.md`.

- [x] **0.3 Fornecer contexto à IA**
  - Objetivo: dar à IA acesso ao código e à transcrição, não apenas a um trecho.
  - Como resolver: use um agente que leia o repositório (ex.: OpenCode/Claude Code/Cursor) ou cole os arquivos-chave. Peça explicitamente para citar arquivo e linha.
  - Saída: análise inicial da estrutura e dos pontos de integração.
  - Pronto quando: a IA lista módulos, máquina de estados, tratamento de erros e logger com caminhos reais.
  - **Concluído:** `docs/agents/` contém 27 relatórios (análise arquitetônica, 25 componentes e auditoria de dependências), com caminhos e linhas reais. O relatório arquitetônico referencia explicitamente a transcrição e as decisões da feature.

- [x] **0.4 Montar o "decision log" mestre**
  - Objetivo: ter uma fonte única de verdade antes de escrever qualquer documento.
  - Como resolver: consolide uma tabela com `tema | decisão | status (fechada/adiada/descartada) | timestamp + falante`.
  - Saída: rascunho interno (`docs/` ou fora da entrega).
  - Pronto quando: cada decisão da reunião está classificada e referenciada.
  - **Concluído:** criado `docs/DECISION-LOG.md` com 29 decisões da reunião (`D-01`–`D-29`) e 9 restrições de código (`C-01`–`C-09`). Formato acordado: tabela mestre + detalhes, status `FECHADA`/`ADIADA`/`DESCARTADA`/`EM ABERTO`, origem com timestamp + falante ou caminho real, alternativas e trade-offs. ADRs ainda não existem, então os links aparecem como "ADR (a criar)".

---

## Etapa 1 — Mapear o código existente

- [x] **1.1 Identificar os pontos de integração reais**
  - Objetivo: o FDD precisa citar pelo menos 4 caminhos de arquivo existentes.
  - Como resolver: peça à IA (ou leia) os seguintes pontos e confirme que existem:
    - `src/modules/orders/order.service.ts` → método `changeStatus` (transação atual).
    - `src/modules/orders/order.status.ts` → máquina de estados.
    - `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` → taxonomia de erros.
    - `src/middlewares/error.middleware.ts` → tratamento centralizado.
    - `src/middlewares/auth.middleware.ts` → `authenticate` e `requireRole`.
    - `src/shared/logger/index.ts` → logger Pino.
    - `src/modules/orders/order.repository.ts` e `order.controller.ts` / `order.routes.ts`.
    - `prisma/schema.prisma` → modelos e padrão de IDs UUID.
    - `src/server.ts` → padrão de entry-point para criar `src/worker.ts`.
  - Saída: tabela `arquivo | responsabilidade | como o webhook se integra`.
  - Pronto quando: todos os caminhos foram verificados no disco (não inventados).
  - **Concluído:** criado `docs/INTEGRATION-POINTS.md` com 16 caminhos existentes verificados, classificação `Alterar`/`Reutilizar`/`Referenciar`/`Criar`, 9 arquivos novos previstos e decisões desta etapa. Erros em `src/modules/webhooks/webhook.errors.ts`; `publishWebhookEvent` em `src/modules/webhooks/webhook.outbox.ts`; admin em `/api/v1/admin/webhooks/...`. `package.json`, `env.ts` e `tests/` ficaram fora da tabela por decisão.

- [~] **1.2 Registrar o padrão arquitetural a ser seguido**
  - Objetivo: justificar o reuso em vez de criar estrutura paralela.
  - Como resolver: documente o padrão `controller + service + repository + routes + schemas` de `src/modules/*` e o padrão de códigos de erro (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`).
  - Saída: insumo para o ADR de reuso e para a seção "Integração com o sistema existente" do FDD.
  - Pronto quando: existe pelo menos 1 ADR que referencia explicitamente esses arquivos.
  - **Documentado em 2026-11-09:** criado `docs/ARCHITECTURE-PATTERNS.md` com anatomia das cinco camadas, direção de dependência, convenções de nome, exceções reais (`auth`, `orders`, `users`), padrão de erros (`AppError`, classes concretas, formato de resposta), validação Zod, auth, logger, composição, entry point, matriz de adaptação para webhooks e matriz de reuso. Todos os caminhos e linhas foram verificados.
  - **Pendente para concluir:** o critério "pronto quando" exige um ADR. Por decisão do usuário, o `docs/adrs/ADR-006-adocao-dos-padroes-existentes.md` será criado na Etapa 2 e deverá referenciar explicitamente `src/modules/orders/*`, `src/modules/customers/*`, `src/modules/products/*`, `src/shared/errors/*`, `src/middlewares/error.middleware.ts`, `src/app.ts` e `src/routes/index.ts`.

---

## Etapa 2 — ADRs (5 a 8)

- [x] **2.1 Criar a pasta e a numeração**
  - Objetivo: padronizar os arquivos.
  - Como resolver: criar `docs/adrs/` (já existe) e nomear `ADR-NNN-titulo-em-kebab-case.md`.
  - Pronto quando: os nomes seguem o padrão exigido.
  - **Concluído:** `docs/adrs/README.md` atualizado com a convenção `ADR-NNN-titulo-em-kebab-case.md`, regras de estabilidade (sem renumerar nem remover), limite de 8 ADRs, numeração planejada de `ADR-001` a `ADR-006` e template MADR. O `ADR-006` foi reservado para o ADR de reuso, conforme decisão do item 1.2. Nenhum ADR vazio foi criado; os arquivos serão produzidos no item 2.2.

- [x] **2.2 Produzir os ADRs das 6 decisões principais**
  - Objetivo: cobrir no mínimo 5 das 6 decisões obrigatórias.
  - Como resolver: um ADR por decisão, usando MADR:
    1. Outbox no MySQL.
    2. Retry com backoff e DLQ.
    3. Autenticação HMAC-SHA256 com secret por endpoint.
    4. Garantia at-least-once com `X-Event-Id`.
    5. Worker separado em polling.
    6. Reuso dos padrões existentes do projeto.
  - Estrutura mínima de cada ADR: `Status`, `Contexto`, `Decisão`, `Alternativas Consideradas` (≥1 real), `Consequências` (positivas e negativas).
  - Dica: use o `[09:48] Larissa` como resumo de conferência e cite timestamps no contexto.
  - Pronto quando: 5–8 ADRs existem e cobrem as decisões.

- [x] **2.3 (Opcional) ADRs secundários**
  - Objetivo: registrar decisões menores que não cabem no FDD.
  - Candidatos: snapshot do payload na inserção (`[09:51] Larissa`), IDs UUID (`[09:51] Larissa`), DLQ em tabela separada (`[09:18] Diego`), timeout de 10s (`[09:42] Diego`), limite de payload 64KB (`[09:23]–[09:24]`).
  - Cuidado: não ultrapassar 8 ADRs.
  - **Concluído:** avaliados e consolidados nos ADRs existentes — snapshot/UUID em ADR-001; DLQ em ADR-002; timeout 10s e payload 64KB ficam como detalhes de configuração no FDD. Nenhum ADR adicional criado, mantendo 6 ADRs (dentro do limite de 8) e evitando duplicação.

- [x] **2.4 Revisar contra alucinações**
  - Objetivo: garantir que cada ADR tem origem.
  - Como resolver: para cada alternativa citada, confirme que foi realmente discutida (ex.: Redis Streams em `[09:07]`, trigger MySQL em `[09:09]`, retry indefinido em `[09:15]`, exactly-once em `[09:25]`).
  - Pronto quando: nenhuma alternativa ou consequência contradiz a transcrição.
  - **Concluído:** revisados ADR-001 a ADR-006; todas as alternativas citadas (envio síncrono, Redis Streams, trigger MySQL, 3 tentativas, retry indefinido, failed na outbox, secret global, sem grace period, exactly-once, worker na API, múltiplos workers, estrutura paralela) têm origem na transcrição ou no código. Nenhuma contradição encontrada.

---

## Etapa 3 — RFC

- [x] **3.1 Escrever o RFC em nível de arquitetura**
  - Objetivo: propor a solução e abrir para revisão, sem descer ao detalhe do FDD.
  - Como resolver: produzir `docs/RFC.md` com:
    - Metadados: autor, status (ex.: `Em revisão`), data, revisores = Larissa, Marcos, Bruno, Diego e Sofia.
    - TL;DR da proposta.
    - Contexto e problema.
    - Proposta técnica em visão geral (outbox + worker + HMAC + DLQ).
    - Alternativas consideradas (≥2 reais): envio síncrono no `changeStatus` (`[09:03]–[09:04]`), Redis Streams (`[09:07]`), trigger MySQL (`[09:09]`), retry indefinido (`[09:15]`).
    - Questões em aberto (≥2): rate limiting (`[09:39]`), e-mail de falha (`[09:37]`), ordering global (`[09:12]`–`[09:13]`), arquivamento de linhas entregues após 30 dias (`[09:08]`).
    - Impacto e riscos.
    - Links para pelo menos 2 ADRs.
  - Restrição: 2 a 4 páginas; não repetir payloads, matriz de erros e fluxos detalhados do FDD.
  - Pronto quando: o RFC responde "o que propomos e por quê", não "como codar".

- [x] **3.2 Revisar concisão e rastreabilidade**
  - Objetivo: evitar duplicação com o FDD.
  - Como resolver: remover trechos que já pertencem ao FDD (contratos, status codes, fluxos passo a passo).
  - Pronto quando: cada parágrafo do RFC está no nível de decisão/arquitetura.

---

## Etapa 4 — FDD

- [x] **4.1 Especificar os fluxos técnicos**
  - Objetivo: permitir que um dev comece a implementar.
  - Como resolver: detalhar, com diagramas/texto:
    - Criação do evento na outbox dentro da transação do `changeStatus`.
    - Processamento pelo worker (polling 2s, batch pequeno).
    - Retry com backoff 1m/5m/30m/2h/12h.
    - Movimentação para DLQ após 5 falhas.
    - Replay manual via endpoint admin.
  - Pronto quando: existe um caminho claro do status mudando até o HTTP no cliente.

- [x] **4.2 Definir os contratos públicos**
  - Objetivo: atender o requisito de ≥4 endpoints com request/response e status codes.
  - Como resolver: especificar, no mínimo:
    - `POST /webhooks` (criar configuração; retorna secret gerada).
    - `GET /webhooks?customer_id=...` (listar).
    - `PATCH /webhooks/:id` (editar URL/status/ativo).
    - `DELETE /webhooks/:id` (remover).
    - `GET /webhooks/:id/deliveries` (histórico das últimas 100 entregas).
    - `POST /admin/webhooks/dead-letter/:id/replay` (replay; `ADMIN`).
    - Rotação de secret (`[09:21] Sofia`) se optar por documentar.
  - Incluir payloads JSON de exemplo, headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type`) e status codes.
  - Pronto quando: cada endpoint tem exemplo de request e response.

- [x] **4.3 Montar a matriz de erros `WEBHOOK_*`**
  - Objetivo: seguir o padrão existente de códigos.
  - Como resolver: listar códigos como `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_DELIVERY_TIMEOUT`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`, com HTTP status e causa.
  - Regra: URL não-HTTPS é erro de validação Zod (`[09:23] Sofia`); payload > 64KB é erro (`[09:24]`).
  - Pronto quando: todos os erros têm prefixo `WEBHOOK_` e origem.

- [x] **4.4 Documentar resiliência, observabilidade e integração**
  - Objetivo: cobrir os itens obrigatórios do FDD.
  - Como resolver:
    - Resiliência: timeout 10s, retry/backoff, DLQ, isolamento por processo.
    - Observabilidade: métricas (eventos pendentes, entregues, falhos, latência, tamanho da DLQ), logs estruturados Pino e tracing/correlação por `X-Event-Id`.
    - **Integração com o sistema existente** (seção obrigatória): citar ≥4 caminhos reais, por exemplo:
      - `src/modules/orders/order.service.ts` → inserir `publishWebhookEvent(tx, order, fromStatus, toStatus)` na mesma transação.
      - `src/shared/errors/app-error.ts` → estender `AppError` para erros `WEBHOOK_*`.
      - `src/middlewares/auth.middleware.ts` → reutilizar `authenticate` e `requireRole('ADMIN')`.
      - `src/middlewares/error.middleware.ts` → tratamento automático dos novos erros.
      - `src/shared/logger/index.ts` → logger do worker.
      - `src/server.ts` → referência para criar `src/worker.ts`.
    - Critérios de aceite técnicos e riscos com mitigação.
  - Pronto quando: a seção de integração nomeia arquivos que realmente existem.

- [x] **4.5 Revisar acionabilidade**
  - Objetivo: garantir que o FDD não é genérico.
  - Como resolver: perguntar "um dev conseguiria abrir o PR só com este documento?". Se a resposta for não, faltam contratos, estados ou regras.
  - Pronto quando: fluxos, contratos, erros e integração estão concretos.

---

## Etapa 5 — PRD

- [x] **5.1 Consolidar o PRD por último entre os grandes documentos**
  - Objetivo: traduzir o problema e o escopo para a linguagem de produto.
  - Como resolver: produzir `docs/PRD.md` com todas as seções:
    - Resumo e contexto.
    - Problema e motivação: 3 clientes B2B (Atlas, MaxDistribuição, Nova Cargo) fazem polling no `GET /orders`; risco de churn da Atlas (`[09:00] Marcos`).
    - Público-alvo e cenários de uso.
    - Objetivos e métricas: incluir meta quantitativa, por exemplo latência < 10s percebida (`[09:02] Marcos`) e prazo de 3 sprints (`[09:46]`–`[09:47]`). Não inventar números: use apenas os citados.
    - Escopo incluso e **fora de escopo** (≥2 itens descartados/adiados; ver lista acima).
    - Requisitos funcionais: ≥8, incluindo CRUD, filtro por status, deliveries, replay ADMIN, assinatura, rotação de secret.
    - Requisitos não funcionais: TLS obrigatório, payload ≤64KB, timeout 10s, latência <10s, at-least-once.
    - Decisões e trade-offs principais.
    - Dependências.
    - Riscos (≥2 com probabilidade, impacto e mitigação).
    - Critérios de aceitação.
    - Estratégia de testes e validação.
  - Cuidado: não promover itens fora de escopo a requisito.
  - Pronto quando: o PRD responde "por que e o quê", sem detalhes de implementação.

- [x] **5.2 Conferir contagem e metas**
  - Objetivo: atender os critérios de aceite.
  - Como resolver: contar requisitos funcionais (≥8), itens fora de escopo (≥2) e riscos completos (≥2).
  - Pronto quando: todos os números mínimos são atingidos com origem.

---

## Etapa 6 — Tracker de rastreabilidade

- [x] **6.1 Criar `docs/TRACKER.md` com a tabela obrigatória**
  - Objetivo: provar que cada item tem origem.
  - Como resolver: usar exatamente as colunas:
    `| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |`
  - Convenções de ID: `PRD-FR-01`, `PRD-RNF-01`, `RFC-ALT-01`, `FDD-CONTRATO-01`, `ADR-002`, etc.
  - Fonte: `TRANSCRICAO` ou `CODIGO`.
  - Localização: `[hh:mm] Nome` para transcrição; caminho de arquivo para código.

- [x] **6.2 Varrer os documentos prontos e registrar os itens**
  - Objetivo: cobertura ≥80% dos itens identificáveis.
  - Como resolver: percorra PRD, RFC, FDD e ADRs; para cada requisito/decisão/restrição/trade-off, crie uma linha. Priorize os itens estruturais e as decisões.
  - Metas: ≥70% das linhas com `TRANSCRICAO` e timestamp válido; ≥5 linhas com `CODIGO` e caminho real.
  - Pronto quando: nenhum item importante fica órfão.

- [x] **6.3 Validar os caminhos e timestamps**
  - Objetivo: eliminar linhas inventadas.
  - Como resolver: confira cada caminho de código com `ls`/editor e cada timestamp na transcrição. Se não encontrar a origem, remova ou reescreva o item no documento.
  - Pronto quando: 100% das linhas apontam para algo existente.

---

## Etapa 7 — README do processo

- [x] **7.1 Substituir o enunciado pelo relato do processo**
  - Objetivo: documentar a jornada de produção.
  - Como resolver: reescrever `README.md` com as seções obrigatórias:
    - **Sobre o desafio** (1–2 parágrafos com suas palavras).
    - **Ferramentas de IA utilizadas** (≥1, com o papel de cada uma).
    - **Workflow adotado** (ordem de produção e organização da interação).
    - **Prompts customizados** (≥2 em blocos de código).
    - **Iterações e ajustes** (≥2 correções concretas; descreva o que a IA errou e como você corrigiu).
    - **Como navegar a entrega** (caminhos e ordem de leitura).
  - Dica: manter um link para o enunciado original é permitido, mas o foco deve ser o processo.
  - Pronto quando: as 6 seções existem e há ≥2 prompts reais.

- [x] **7.2 Registrar evidências honestas das iterações**
  - Objetivo: evitar um relato genérico.
  - Como resolver: anote momentos como "a IA incluiu e-mail como requisito e removi porque `[09:37]` diz fora de escopo" ou "a IA inventou um endpoint de dashboard e corrigi". Use o `README_2.md` como insumo, se útil.
  - Pronto quando: as iterações citam erros e correções específicas.

---

## Etapa 8 — Revisão final e publicação

- [x] **8.1 Rodar checklist de consistência**
  - Objetivo: garantir que documentos, transcrição e código não se contradizem.
  - Como resolver: verificar:
    - Nenhum requisito/decisão contradiz a transcrição.
    - Nenhum arquivo citado é inexistente.
    - Itens fora de escopo não aparecem como requisitos.
    - RFC não duplica o FDD.
    - ADRs cobrem ≥5 das 6 decisões.
    - Tracker cobre ≥80% dos itens.
    - README tem todas as seções.
  - Pronto quando: todas as caixas abaixo estiverem marcadas.

- [x] **8.2 Conferir estrutura de arquivos**
  - Objetivo: atender a árvore obrigatória do entregável.
  - Como resolver: validar `README.md`, `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md` e `docs/adrs/ADR-NNN-*.md`.
  - Pronto quando: a estrutura bate com o exigido.

- [x] **8.3 Confirmar que o código não foi alterado**
  - Objetivo: respeitar a restrição absoluta.
  - Como resolver: rodar `git status` e verificar que `src/`, `prisma/`, `tests/` e configs estão sem mudanças.
  - Pronto quando: só arquivos de documentação aparecem no diff.

- [ ] **8.4 Commit, push e entrega**
  - Objetivo: publicar o pacote.
  - Como resolver: revisar o diff, commitar com mensagem clara e fazer push no fork público.
  - Pronto quando: o repositório público contém todos os arquivos e abre sem links quebrados.

---

## Checklist de aceite consolidado

### PRD
- [ ] `docs/PRD.md` existe em Markdown
- [ ] Todas as seções obrigatórias presentes
- [ ] ≥8 requisitos funcionais da reunião
- [ ] ≥1 objetivo com métrica e meta quantitativa
- [ ] Fora de escopo com ≥2 itens descartados/adiados
- [ ] ≥2 riscos com probabilidade, impacto e mitigação

### RFC
- [ ] `docs/RFC.md` existe em Markdown
- [ ] Todas as seções obrigatórias presentes
- [ ] ≥2 alternativas descartadas, cada uma com trade-off
- [ ] ≥2 questões em aberto
- [ ] Links para ≥2 ADRs

### FDD
- [ ] `docs/FDD.md` existe em Markdown
- [ ] Todas as seções obrigatórias presentes
- [ ] ≥4 endpoints com request/response de exemplo e status codes
- [ ] Matriz de erros com prefixo `WEBHOOK_`
- [ ] Seção "Integração com o sistema existente" com ≥4 caminhos reais
- [ ] Observabilidade cita métricas, logs e tracing

### ADRs
- [ ] 5 a 8 arquivos em `docs/adrs/` no padrão `ADR-NNN-*.md`
- [ ] Cada ADR tem Status, Contexto, Decisão, Alternativas e Consequências
- [ ] Conjunto cobre ≥5 das 6 decisões principais
- [ ] ≥1 ADR referencia arquivos/módulos/classes do código

### Tracker
- [ ] `docs/TRACKER.md` segue a tabela obrigatória
- [ ] ≥80% dos itens identificáveis cobertos
- [ ] ≥70% das linhas com `TRANSCRICAO` + `[hh:mm] Nome`
- [ ] ≥5 linhas com `CODIGO` + caminho real

### README
- [ ] Todas as seções obrigatórias presentes
- [ ] ≥1 ferramenta de IA listada
- [ ] ≥2 prompts customizados em blocos de código
- [ ] ≥2 iterações/ajustes concretos

### Consistência
- [ ] Nenhuma contradição com transcrição ou código
- [ ] Nenhum arquivo de código citado é inexistente
- [ ] `src/`, `prisma/`, `tests/` e configs intocados

---

## Ordem sugerida de leitura da entrega

1. `README.md` — entenda o processo e as ferramentas.
2. `docs/PRD.md` — por que e o quê.
3. `docs/RFC.md` — proposta e questões em aberto.
4. `docs/adrs/` — decisões pontuais.
5. `docs/FDD.md` — como construir em detalhe.
6. `docs/TRACKER.md` — de onde veio cada item.

---

## Prompts-base sugeridos

Use prompts dirigidos, com exigência de citação de origem. Exemplos:

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

---

## Observações finais

- O maior risco é a IA "completar" lacunas com padrões de mercado. Sempre confirme.
- Não transforme sugestões descartadas em requisitos: e-mail, rate limiting, dashboard e webhooks inbound são armadilhas clássicas.
- O Tracker é o melhor detector de alucinação: sem localização, o item não entra.
- A qualidade vem de iteração. Documentos gerados de primeira tendem a ser genéricos.
