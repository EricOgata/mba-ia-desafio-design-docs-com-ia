# Decision Log — Sistema de Webhooks de Notificação de Pedidos

> **Fonte de verdade:** `TRANSCRICAO.md` e código-fonte do repositório. Este documento é um registro de trabalho, não substitui a transcrição nem o `docs/TRACKER.md`.
>
> **Papel:** servir de fonte para os ADRs, o RFC, o FDD e o PRD. O Tracker continua sendo a referência cruzada dos documentos finais; o decision log não duplica essa função.
>
> **Estado:** versão inicial criada em 2026-11-09. Os ADRs ainda serão produzidos na Etapa 2; por isso os links aparecem como "ADR (a criar)" em vez de arquivos inexistentes.

---

## 1. Convenções

| Item | Regra |
| --- | --- |
| IDs de decisão | `D-01`, `D-02`... para itens discutidos na reunião |
| IDs de código | `C-01`, `C-02`... para decisões/restrições já impostas pelo código existente |
| Status | `FECHADA`, `ADIADA`, `DESCARTADA`, `EM ABERTO` |
| Origem na transcrição | `[hh:mm] Nome` |
| Origem no código | caminho do arquivo, com linha quando relevante |
| Alternativas | registradas apenas quando foram realmente discutidas; `—` quando não houve discussão |
| Links | apontam para documentos planejados (`RFC`, `FDD`, `PRD`, `TRACKER`) ou "ADR (a criar)" |
| Alternativas dentro de uma decisão | não recebem ID próprio; ficam na coluna "Alternativas consideradas" da decisão que as avaliou |
| Itens de escopo descartados | recebem linha própria quando afetam o escopo do produto |
| Fonte `CODIGO` | usada na seção 5, separada das decisões da reunião |

### Resumo por status

| Status | Quantidade | Itens |
| --- | --- | --- |
| `FECHADA` | 21 | D-01 a D-20, D-27 |
| `ADIADA` | 2 | D-22, D-23 |
| `DESCARTADA` | 3 | D-25, D-26, D-29 |
| `EM ABERTO` | 3 | D-21, D-24, D-28 |
| **Total** | **29** | — |

---

## 2. Tabela mestre

| ID | Tema | Decisão | Status | Origem | Alternativas consideradas | Trade-off / consequência | Docs relacionados |
| --- | --- | --- | --- | --- | --- | --- | --- |
| D-01 | Padrão de entrega | Outbox transacional em `webhook_outbox` no MySQL existente | `FECHADA` | `[09:06] Diego`; `[09:07] Larissa/Diego` | Envio síncrono; Redis Streams | Sem infraestrutura nova; a latência passa a depender do worker | RFC; FDD; ADR (a criar) |
| D-02 | Processamento | Worker em processo separado, polling a cada 2s | `FECHADA` | `[09:09] Diego`; `[09:11] Diego/Larissa` | Trigger do MySQL; worker dentro da API | Isola o worker da API; polling adiciona latência de até 2s | RFC; FDD; ADR (a criar) |
| D-03 | Ordering | Single-worker com ordering implícita por `order_id` | `FECHADA` | `[09:12]–[09:13] Diego/Larissa` | Múltiplos workers com partição ou lock pessimista | Não há garantia de ordering global; limitação conhecida | RFC; FDD; ADR (a criar) |
| D-04 | Retry | 5 tentativas com backoff 1m/5m/30m/2h/12h | `FECHADA` | `[09:15]–[09:17] Diego` | 3 tentativas; retry indefinido | Janela de ~15h; depois disso o evento vai para DLQ | RFC; FDD; ADR (a criar) |
| D-05 | Falha definitiva | DLQ em tabela separada `webhook_dead_letter` | `FECHADA` | `[09:18] Diego` | Marcar como falha na própria outbox | Outbox principal mais limpa; exige tabela e fluxo adicionais | RFC; FDD; ADR (a criar) |
| D-06 | Replay | `POST /admin/webhooks/dead-letter/:id/replay`, com `ADMIN` e auditoria | `FECHADA` | `[09:18] Diego`; `[09:35]–[09:36] Larissa/Sofia` | — | Reprocessamento manual; exige controle de acesso e registro de quem executou | RFC; FDD; ADR (a criar) |
| D-07 | Autenticidade | HMAC-SHA256 sobre o corpo, header `X-Signature` | `FECHADA` | `[09:20] Sofia` | — | Exige secret compartilhada; padrão de mercado | RFC; FDD; ADR (a criar) |
| D-08 | Secret | Secret única por endpoint, rotacionável com grace period de 24h | `FECHADA` | `[09:21]–[09:22] Sofia` | Secret global da plataforma | Mais gestão de secrets; o vazamento de uma não compromete todas | RFC; FDD; ADR (a criar) |
| D-09 | Garantia de entrega | At-least-once com `X-Event-Id` para dedup no cliente | `FECHADA` | `[09:24]–[09:26] Diego` | Exactly-once | Duplicatas são possíveis; a dedup fica do lado do cliente | RFC; FDD; ADR (a criar) |
| D-10 | Estrutura | Módulo `src/modules/webhooks`; `src/worker.ts`; lógica no módulo | `FECHADA` | `[09:27]–[09:28] Bruno/Diego` | — | Segue o padrão existente; exige novo entry point e script | FDD; ADR (a criar) |
| D-11 | Erros e logs | Prefixo `WEBHOOK_`; reuso de `AppError`, Pino e error middleware | `FECHADA` | `[09:28]–[09:30] Bruno/Larissa` | — | Consistência com o projeto; acoplamento aos padrões atuais | FDD; ADR (a criar) |
| D-12 | Payload | Snapshot JSON renderizado na inserção da outbox | `FECHADA` | `[09:51]–[09:52] Larissa/Diego` | Guardar só `order_id` e renderizar no envio | O evento reflete o estado da mudança; pode divergir do pedido atual | FDD |
| D-13 | Identificador | UUID como ID da outbox | `FECHADA` | `[09:51] Larissa` | Auto-incremento | Segue o padrão do projeto | FDD |
| D-14 | Timeout HTTP | 10 segundos por chamada | `FECHADA` | `[09:42] Diego` | — | Cliente lento é tratado como falha e entra em retry | FDD |
| D-15 | Limite de payload | 64KB; erro se ultrapassar | `FECHADA` | `[09:23]–[09:24] Sofia/Diego/Larissa` | 500KB; truncar o payload | Eventos grandes falham; mantém o payload enxuto | FDD; PRD |
| D-16 | TLS | URL obrigatoriamente `https`; `http` recusado na validação Zod | `FECHADA` | `[09:23] Sofia` | — | Segurança no transporte; clientes precisam suportar TLS | FDD; PRD |
| D-17 | Filtro de eventos | Filtrar status desejados na inserção da outbox | `FECHADA` | `[09:33]–[09:34] Marcos/Bruno/Diego` | Filtrar no momento do envio | Menos linhas na outbox; se nenhum webhook quer o status, nada é inserido | FDD; PRD |
| D-18 | Autorização do CRUD | CRUD de configuração aceita qualquer role autenticada por enquanto | `FECHADA` | `[09:36]–[09:37] Sofia/Marcos` | Endurecer por role agora | Simplicidade inicial; endurecimento fica para depois | FDD; PRD |
| D-19 | Conteúdo do payload | Payload enxuto, sem itens do pedido | `FECHADA` | `[09:43] Diego` | Incluir itens do pedido | Evita inflar o payload; cliente busca detalhes em `GET /orders/:id` | FDD |
| D-20 | Headers | `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type` | `FECHADA` | `[09:44] Diego/Sofia` | — | Mais metadados; permite dedup, validação e identificação do cadastro | FDD |
| D-21 | `customer_id` | Não vem do JWT; deve ser informado no body ou no path | `EM ABERTO` | `[09:32] Larissa` | Usar o JWT (rejeitado: JWT é do operador) | Exige informar o cliente explicitamente; evita confundir operador com cliente | FDD; PRD |
| D-22 | Arquivamento da outbox | Arquivar linhas entregues após ~30 dias | `ADIADA` | `[09:08] Diego` | Implementar agora; deixar para depois | A outbox cresce; o arquivamento fica fora desta feature | PRD; FDD |
| D-23 | E-mail de fallback | Avisar o cliente por e-mail após falhas | `ADIADA` | `[09:37] Larissa` | Implementar nesta fase | Cliente não é avisado por e-mail nesta fase | PRD; RFC |
| D-24 | Rate limiting | Limitar o volume de envios por cliente | `EM ABERTO` | `[09:39] Diego/Larissa` | Implementar agora; observar e decidir depois | Risco de bombardear o cliente; fica em observação | PRD; RFC |
| D-25 | Dashboard | Painel visual para o cliente acompanhar webhooks | `DESCARTADA` | `[09:39]–[09:40] Larissa` | Construir painel nesta fase | Entrega fica só em endpoints; painel é projeto do frontend | PRD |
| D-26 | Webhooks inbound | Receber eventos enviados pelo cliente | `DESCARTADA` | `[09:02] Sofia/Marcos` | Suportar inbound e outbound | A feature é somente outbound | PRD |
| D-27 | Prazo e revisão | 3 sprints, incluindo 2 dias úteis de revisão de segurança | `FECHADA` | `[09:45]–[09:47] Marcos/Larissa/Sofia` | — | Prazo inclui a revisão da Sofia antes do deploy | PRD |
| D-28 | Rota da rotação | Endpoint de rotação de secret ainda sem rota definida | `EM ABERTO` | `[09:21] Sofia` | — | O requisito existe, mas o contrato HTTP precisa ser fechado | FDD |
| D-29 | Ordering global | Não garantir ordering global | `DESCARTADA` | `[09:12]–[09:13] Diego/Larissa` | Múltiplos workers com partição ou lock | Ordem só por `order_id` e enquanto single-worker | RFC; FDD; PRD |

---

## 3. Detalhes das decisões fechadas

### D-01 — Outbox transacional no MySQL

- **Contexto:** a transação de `changeStatus` já atualiza `orders`, insere em `order_status_history` e mexe em estoque. Uma chamada HTTP síncrona travaria a mudança de status e não permitiria rollback por cliente offline.
- **Decisão:** inserir o evento em `webhook_outbox` dentro da mesma transação SQL; um worker separado lê e dispara as chamadas.
- **Alternativas:** envio síncrono no service (descartado); Redis Streams (descartado por exigir mais infraestrutura).
- **Trade-off:** não exige infraestrutura nova, mas a entrega passa a depender do worker e do polling.
- **Origem:** `[09:04] Bruno`, `[09:06] Diego`, `[09:07] Larissa/Diego`.
- **Relacionados:** RFC, FDD, ADR de outbox (a criar), C-06.

### D-02 — Worker em processo separado com polling de 2s

- **Contexto:** se o worker rodar dentro da API, reinícios da API derrubam o processamento. MySQL não tem listener nativo e trigger não notifica processo externo.
- **Decisão:** worker em processo Node separado, lendo a outbox por polling a cada 2 segundos.
- **Alternativas:** trigger do MySQL (descartado); worker dentro da API (descartado).
- **Trade-off:** isolamento de processo; a reunião aceitou 2s de polling como suficiente para o requisito de menos de 10s.
- **Origem:** `[09:09] Diego`, `[09:11] Diego/Larissa`, `[09:10] Marcos`.
- **Relacionados:** RFC, FDD, ADR de worker (a criar), C-04, C-07.

### D-03 — Single-worker e ordering por `order_id`

- **Contexto:** com múltiplos workers em paralelo, a ordem de processamento deixaria de ser garantida.
- **Decisão:** operar com single-worker, processando os eventos mais antigos por `created_at`; a ordering é implícita por `order_id`.
- **Alternativas:** múltiplos workers com particionamento por `order_id` ou lock pessimista (deixado para o futuro).
- **Trade-off:** não há garantia de ordering global; a limitação deve ser documentada.
- **Origem:** `[09:12]–[09:13] Diego/Larissa`, `[09:14] Marcos`.
- **Relacionados:** RFC, FDD, ADR de worker/ordering (a criar), D-29.

### D-04 — Retry com backoff exponencial

- **Contexto:** clientes podem ficar indisponíveis por períodos prolongados; retry indefinido deixaria eventos pendurados para sempre.
- **Decisão:** 5 tentativas com intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas.
- **Alternativas:** 3 tentativas (considerada pouca); retry indefinido (descartado).
- **Trade-off:** janela total de quase 15 horas; depois disso, o evento é considerado falha permanente.
- **Origem:** `[09:15]–[09:17] Diego`, `[09:17] Marcos`.
- **Relacionados:** RFC, FDD, ADR de retry/DLQ (a criar).

### D-05 — DLQ em tabela separada

- **Contexto:** eventos que esgotam as tentativas precisam de um destino para debug e reprocessamento.
- **Decisão:** criar `webhook_dead_letter`, com payload, motivo da falha e timestamp.
- **Alternativas:** marcar o evento como `failed` na própria outbox.
- **Trade-off:** mantém a outbox principal limpa e serve como evidência; exige tabela e fluxo adicionais.
- **Origem:** `[09:18] Diego`, `[09:18] Bruno`.
- **Relacionados:** RFC, FDD, ADR de retry/DLQ (a criar).

### D-06 — Replay manual de DLQ por ADMIN

- **Contexto:** é preciso permitir que um operador recolha um evento da DLQ e tente novamente.
- **Decisão:** `POST /admin/webhooks/dead-letter/:id/replay`, recolocando o evento na outbox como pendente. Exige role `ADMIN` e registro de quem executou o replay.
- **Alternativas:** — (não foi discutido um mecanismo automático).
- **Trade-off:** reprocessamento manual; maior controle de acesso e auditoria.
- **Origem:** `[09:18] Diego`, `[09:35]–[09:36] Larissa/Sofia`.
- **Relacionados:** RFC, FDD, ADR de retry/DLQ (a criar), C-02.

### D-07 — HMAC-SHA256 sobre o corpo

- **Contexto:** o cliente precisa validar que a requisição veio da plataforma e que o payload não foi adulterado.
- **Decisão:** assinar o corpo com HMAC-SHA256 e enviar a assinatura no header `X-Signature`.
- **Alternativas:** — (o algoritmo foi perguntado, mas SHA-256 foi escolhido como padrão de mercado).
- **Trade-off:** exige compartilhamento e gestão de secret entre as partes.
- **Origem:** `[09:20] Sofia`.
- **Relacionados:** RFC, FDD, ADR de HMAC (a criar).

### D-08 — Secret única por endpoint com rotação de 24h

- **Contexto:** uma secret global vazaria todos os endpoints se fosse comprometida.
- **Decisão:** cada endpoint de webhook tem secret própria, rotacionável pela API. A secret antiga permanece válida por 24 horas após a rotação.
- **Alternativas:** secret global da plataforma (descartada).
- **Trade-off:** mais gestão de secrets; o vazamento de uma não compromete as demais.
- **Origem:** `[09:21]–[09:22] Sofia`, `[09:22] Diego`.
- **Relacionados:** RFC, FDD, ADR de HMAC (a criar), D-28.

### D-09 — At-least-once com `X-Event-Id`

- **Contexto:** garantir exactly-once exigiria coordenação dos dois lados e aumentaria muito a complexidade.
- **Decisão:** garantir at-least-once; o cliente pode receber o mesmo evento mais de uma vez e deduplica pelo `X-Event-Id`, um UUID gerado quando o evento entra na outbox.
- **Alternativas:** exactly-once (descartado).
- **Trade-off:** duplicatas são possíveis; a responsabilidade de dedup fica com o cliente.
- **Origem:** `[09:24]–[09:26] Diego`, `[09:25] Sofia`.
- **Relacionados:** RFC, FDD, ADR de at-least-once (a criar).

### D-10 — Módulo `src/modules/webhooks` e worker separado

- **Contexto:** o projeto segue um padrão modular claro, com controller, service, repository, routes e schemas.
- **Decisão:** criar `src/modules/webhooks` seguindo o mesmo padrão; o entry point do worker fica em `src/worker.ts`, com a lógica no módulo (`webhook.worker.ts` ou `webhook.processor.ts`).
- **Alternativas:** —.
- **Trade-off:** consistência com o projeto; exige novo entry point e script `npm run worker`.
- **Origem:** `[09:27]–[09:28] Bruno/Diego`, `[09:11] Larissa`.
- **Relacionados:** FDD, `docs/adrs/ADR-006-adocao-dos-padroes-existentes.md` (a criar na Etapa 2), C-07, C-08.

### D-11 — Prefixo `WEBHOOK_` e reuso dos padrões existentes

- **Contexto:** o projeto já tem `AppError`, classes específicas, códigos como `INSUFFICIENT_STOCK`, logger Pino e error middleware centralizado.
- **Decisão:** usar códigos com prefixo `WEBHOOK_` e reutilizar `AppError`, Pino e o middleware de erro; não criar estrutura paralela.
- **Alternativas:** —.
- **Trade-off:** consistência e menos código novo; acoplamento aos padrões atuais.
- **Origem:** `[09:28]–[09:30] Bruno/Larissa`.
- **Relacionados:** FDD, `docs/adrs/ADR-006-adocao-dos-padroes-existentes.md` (a criar na Etapa 2), C-01, C-03; ver `docs/ARCHITECTURE-PATTERNS.md`.

### D-12 — Snapshot do payload na inserção

- **Contexto:** se o evento guardasse apenas o `order_id`, o payload poderia refletir um estado posterior do pedido, e não o estado do momento da mudança.
- **Decisão:** renderizar o payload no momento da inserção da outbox e armazená-lo como snapshot.
- **Alternativas:** guardar apenas `order_id` e renderizar na hora do envio.
- **Trade-off:** o evento reflete fielmente a mudança; o payload pode ficar desatualizado em relação ao pedido atual.
- **Origem:** `[09:51]–[09:52] Larissa/Diego`, `[09:52] Bruno`.
- **Relacionados:** FDD, C-06.

### D-13 — UUID como ID da outbox

- **Contexto:** o restante do projeto usa UUID como identificador.
- **Decisão:** usar UUID na outbox.
- **Alternativas:** auto-incremento (considerado e descartado).
- **Trade-off:** segue o padrão do projeto.
- **Origem:** `[09:51] Larissa`.
- **Relacionados:** FDD, C-05.

### D-14 — Timeout HTTP de 10 segundos

- **Contexto:** o worker não pode ficar preso indefinidamente em um cliente lento.
- **Decisão:** timeout de 10 segundos por chamada; estouro é tratado como falha e entra em retry.
- **Alternativas:** —.
- **Trade-off:** clientes lentos falham e são reprocessados.
- **Origem:** `[09:42] Diego`, `[09:42] Sofia`.
- **Relacionados:** FDD.

### D-15 — Limite de payload de 64KB

- **Contexto:** um evento muito grande indica problema e não deve ser enviado.
- **Decisão:** limite de 64KB; se ultrapassar, retornar erro (não truncar).
- **Alternativas:** 500KB (considerado); truncar o payload (Sofia foi contra).
- **Trade-off:** eventos acima do limite falham; o payload permanece enxuto.
- **Origem:** `[09:23]–[09:24] Sofia/Diego/Larissa`.
- **Relacionados:** FDD, PRD.

### D-16 — HTTPS obrigatório

- **Contexto:** os eventos saem da infraestrutura da plataforma para endpoints externos.
- **Decisão:** exigir `https` na URL; cadastro com `http` é recusado na validação Zod.
- **Alternativas:** —.
- **Trade-off:** segurança no transporte; clientes precisam oferecer TLS.
- **Origem:** `[09:23] Sofia`.
- **Relacionados:** FDD, PRD, C-09.

### D-17 — Filtro de eventos na inserção da outbox

- **Contexto:** cada webhook pode escolher quais status deseja receber.
- **Decisão:** aplicar o filtro no momento de inserir na outbox; se nenhum webhook do customer quer aquele status, o evento nem é inserido.
- **Alternativas:** filtrar no momento do envio.
- **Trade-off:** economiza linhas na outbox; exige consultar as preferências na transação.
- **Origem:** `[09:33]–[09:34] Marcos/Bruno/Diego`.
- **Relacionados:** FDD, PRD.

### D-18 — CRUD com autenticação normal

- **Contexto:** o painel do cliente usa usuários que representam o cliente, autenticados no sistema.
- **Decisão:** os endpoints de CRUD de configuração aceitam qualquer role autenticada por enquanto; o endurecimento por role fica para depois.
- **Alternativas:** endurecer por role imediatamente.
- **Trade-off:** simplicidade inicial; possível endurecimento futuro.
- **Origem:** `[09:36]–[09:37] Sofia/Marcos`.
- **Relacionados:** FDD, PRD, C-02.

### D-19 — Payload enxuto, sem itens do pedido

- **Contexto:** incluir os itens do pedido inflaria o payload.
- **Decisão:** enviar apenas os campos básicos do evento e do pedido; o cliente consulta `GET /orders/:id` se precisar de detalhes.
- **Alternativas:** incluir os itens do pedido no payload.
- **Trade-off:** payload menor; o cliente pode precisar de uma segunda chamada.
- **Origem:** `[09:43] Diego`, `[09:44] Bruno`.
- **Relacionados:** FDD.

### D-20 — Headers do envio

- **Contexto:** o cliente precisa identificar o evento, validar a assinatura e reconhecer o cadastro que gerou o envio.
- **Decisão:** enviar `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`.
- **Alternativas:** —.
- **Trade-off:** mais metadados; permite dedup, validação e identificação.
- **Origem:** `[09:44] Diego/Sofia`.
- **Relacionados:** FDD.

### D-27 — Prazo de 3 sprints e revisão de segurança

- **Contexto:** a Atlas quer a entrega até o fim de novembro.
- **Decisão:** estimar 3 sprints, incluindo pelo menos 2 dias úteis de revisão de segurança da Sofia antes do deploy.
- **Alternativas:** —.
- **Trade-off:** o prazo inclui a revisão de segurança; HMAC e geração de secret serão revisados com calma.
- **Origem:** `[09:45]–[09:47] Marcos/Larissa/Sofia`.
- **Relacionados:** PRD.

---

## 4. Detalhes dos itens adiados, descartados e em aberto

### D-21 — `customer_id` no body ou no path (EM ABERTO)

- **Contexto:** o JWT atual é do usuário operador, não do cliente. Marcos inicialmente disse que o `customer_id` seria implícito no JWT, mas Bruno apontou a diferença.
- **Decisão parcial:** o `customer_id` não vem do JWT e deve ser informado explicitamente; falta escolher entre body e path.
- **Alternativas:** usar o JWT (rejeitado).
- **Trade-off:** exige informar o cliente explicitamente; evita confundir operador com cliente.
- **Origem:** `[09:31] Marcos`, `[09:32] Bruno/Larissa`.
- **Relacionados:** FDD, PRD.

### D-22 — Arquivamento da outbox após ~30 dias (ADIADA)

- **Contexto:** linhas entregues podem acumular na outbox.
- **Decisão:** arquivar linhas entregues depois de aproximadamente 30 dias, fora do escopo desta feature.
- **Alternativas:** implementar agora; deixar para depois.
- **Trade-off:** a outbox cresce; o arquivamento fica para uma fase posterior.
- **Origem:** `[09:08] Diego`.
- **Relacionados:** PRD, FDD.

### D-23 — E-mail de fallback (ADIADA)

- **Contexto:** Marcos perguntou se o cliente poderia ser avisado por e-mail após falhas repetidas.
- **Decisão:** e-mail fica fora de escopo nesta fase; talvez na próxima, depois de medir o impacto.
- **Alternativas:** implementar nesta fase.
- **Trade-off:** o cliente não recebe aviso por e-mail nesta entrega.
- **Origem:** `[09:37] Larissa`, `[09:38] Marcos`.
- **Relacionados:** PRD, RFC.

### D-24 — Rate limiting de envio (EM ABERTO)

- **Contexto:** se um cliente tiver muitas mudanças de status em pouco tempo, pode receber muitas chamadas.
- **Decisão:** não entra no escopo agora; observar e implementar se virar problema.
- **Alternativas:** implementar agora; observar e decidir depois.
- **Trade-off:** risco de bombardear o cliente; fica em observação.
- **Origem:** `[09:39] Diego/Larissa`.
- **Relacionados:** PRD, RFC.

### D-25 — Dashboard visual (DESCARTADA)

- **Contexto:** Marcos perguntou sobre um painel para o cliente ver os webhooks.
- **Decisão:** não será feito agora; a entrega fica só em endpoints, e painel é projeto separado do time de frontend.
- **Alternativas:** construir painel nesta fase.
- **Trade-off:** o cliente opera via API; não há interface visual nesta entrega.
- **Origem:** `[09:39]–[09:40] Larissa/Marcos`.
- **Relacionados:** PRD.

### D-26 — Webhooks inbound (DESCARTADA)

- **Contexto:** Sofia perguntou se os clientes também enviariam eventos para a plataforma.
- **Decisão:** o fluxo é somente outbound; a plataforma envia, o cliente recebe.
- **Alternativas:** suportar inbound e outbound.
- **Trade-off:** a feature cobre apenas notificações de saída.
- **Origem:** `[09:02] Sofia/Marcos`.
- **Relacionados:** PRD.

### D-28 — Rota da rotação de secret (EM ABERTO)

- **Contexto:** Sofia definiu que a secret deve ser rotacionável e que a antiga permanece válida por 24h.
- **Decisão parcial:** o requisito existe, mas a rota HTTP do endpoint de rotação não foi definida na reunião.
- **Alternativas:** —.
- **Trade-off:** o contrato precisa ser fechado no FDD.
- **Origem:** `[09:21] Sofia`.
- **Relacionados:** FDD, D-08.

### D-29 — Ordering global (DESCARTADA)

- **Contexto:** os clientes não pediram garantia de ordering global; querem saber se cada pedido mudou.
- **Decisão:** não garantir ordering global; a ordem só é implícita por `order_id` e enquanto houver single-worker.
- **Alternativas:** múltiplos workers com particionamento por `order_id` ou lock pessimista (futuro).
- **Trade-off:** sem ordering global; a limitação deve ser documentada.
- **Origem:** `[09:12]–[09:14] Diego/Larissa/Marcos`.
- **Relacionados:** RFC, FDD, PRD, D-03.

---

## 5. Decisões e restrições originadas do código (`CODIGO`)

> Itens desta seção não são decisões da reunião. São restrições ou padrões já presentes no código que a implementação da feature deve respeitar. Todos os caminhos foram verificados no repositório.

| ID | Tema | Restrição vigente | Evidência no código | Relacionados |
| --- | --- | --- | --- | --- |
| C-01 | Taxonomia de erros | Novos erros devem estender `AppError` (ou subclasses) e usar códigos `WEBHOOK_*`; o middleware central já trata `AppError`, Zod e Prisma | `src/shared/errors/app-error.ts`; `src/shared/errors/http-errors.ts`; `src/shared/errors/index.ts`; `src/middlewares/error.middleware.ts` | D-11 |
| C-02 | Autenticação e autorização | Reutilizar `authenticate` e `requireRole`; o replay de DLQ pode exigir `requireRole('ADMIN')` | `src/middlewares/auth.middleware.ts:49`; uso em `src/modules/users/user.routes.ts:15` | D-06, D-18 |
| C-03 | Logging | Reutilizar o logger Pino existente, sem criar um novo mecanismo de log | `src/shared/logger/index.ts`; `src/middlewares/request-logger.middleware.ts` | D-11 |
| C-04 | Prisma por processo | O worker deve criar sua própria instância de `PrismaClient`, no próprio processo, usando o mesmo `DATABASE_URL` | `src/config/database.ts` (`createPrismaClient`, `prisma`); `prisma/schema.prisma` | D-02, D-10 |
| C-05 | Identificadores UUID | O padrão do schema é `String @id @default(uuid()) @db.Char(36)`; a outbox deve seguir esse padrão | `prisma/schema.prisma:26,41,57,75,100,117` | D-13 |
| C-06 | Transação e máquina de estados | A inserção na outbox deve ocorrer dentro do `$transaction` de `changeStatus`; se falhar, a transação inteira sofre rollback; as transições devem respeitar `canTransition` | `src/modules/orders/order.service.ts:126`; `src/modules/orders/order.status.ts` | D-01, D-12 |
| C-07 | Entry point | O projeto tem `src/server.ts` como entry point; o worker deve ser um novo entry point, como `src/worker.ts` | `src/server.ts` | D-02, D-10 |
| C-08 | Padrão modular e rotas | O módulo de webhooks deve seguir controller/service/repository/routes/schemas e ser registrado no router da API | `src/modules/orders/*`; `src/routes/index.ts`; `src/app.ts` | D-10 |
| C-09 | Validação Zod | Reutilizar o middleware de validação e os schemas por módulo; a exigência de HTTPS pode ser validada no schema | `src/middlewares/validate.middleware.ts`; `src/modules/orders/order.schemas.ts` | D-16 |

### Notas de integração

- **C-06** é o ponto mais crítico: o método `changeStatus` já executa update do pedido, inserção no histórico e débito/reposição de estoque dentro de `this.prisma.$transaction`. A outbox precisa entrar nessa mesma transação, conforme `[09:40]–[09:41] Bruno/Diego`.
- **C-02** já é usado em `src/modules/users/user.routes.ts` com `requireRole('ADMIN')`, o que serve de referência direta para o endpoint de replay.
- **C-04** está alinhado com `[09:30] Bruno`: o worker é outro processo Node, portanto não compartilha a instância de `PrismaClient` da API.

---

## 6. Regras de manutenção

1. **Não criar item sem origem.** Todo `D-xx` precisa de timestamp + falante; todo `C-xx` precisa de caminho de arquivo real.
2. **Manter IDs estáveis.** Quando um ADR for criado, atualizar a coluna "Docs relacionados" com o link do ADR, sem renumerar o `D-xx`.
3. **Atualizar status, não apagar histórico.** Se um item `EM ABERTO` for decidido, mudar para `FECHADA` e registrar a nova origem.
4. **Não duplicar o Tracker.** O Tracker rastreia os documentos finais (PRD, RFC, FDD, ADRs); este log é a fonte primária das decisões.
5. **Alternativas não viram itens.** Redis Streams, envio síncrono, trigger MySQL, exactly-once e secret global permanecem como alternativas nas decisões que as avaliaram.
6. **Itens fora de escopo continuam visíveis.** Eles devem aparecer no PRD como "fora de escopo" e, quando aplicável, no RFC como questão em aberto.

---

## 7. Histórico de alterações

| Data | Alteração |
| --- | --- |
| 2026-11-09 | Criação inicial com 29 decisões da reunião e 9 restrições de código, conforme consenso de formato com o usuário |
