# Resumo da Transcrição — Sistema de Webhooks de Notificação de Pedidos

> **Documento de apoio.** Resume `TRANSCRICAO.md` removendo amenidades, saudações e conversa paralela. A fonte oficial continua sendo a transcrição; os timestamps abaixo permitem conferir cada item.

---

## Visão geral em 30 segundos

- Três clientes B2B precisam ser avisados quando o status de seus pedidos muda, em vez de ficar consultando `GET /orders`.
- A solução é **outbound webhook** com **padrão outbox no MySQL**, gravado na mesma transação da mudança de status.
- Um **worker separado** lê a outbox por **polling a cada 2 segundos** e dispara as chamadas HTTP.
- Entrega **at-least-once**, com `X-Event-Id` para o cliente deduplicar.
- Segurança com **HMAC-SHA256**, secret única por endpoint e rotação com grace period de 24h.
- Falhas passam por **5 tentativas com backoff** e, depois, vão para uma **DLQ** com replay manual por ADMIN.
- **Fora de escopo:** e-mail de falha, rate limiting, dashboard, webhooks inbound e ordering global.

---

## Dados da reunião

| Item | Informação |
| --- | --- |
| Data | Quinta-feira, 09:00 |
| Duração | ~55 minutos (09:00–09:53) |
| Formato | Call remota (Meet) |
| Participantes | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno/Pedidos), Diego (Eng. Sênior/Plataforma), Sofia (Eng. Segurança) |

---

## 1. Problema e contexto

- Atlas Comercial, MaxDistribuição e Nova Cargo pediram notificação em tempo real de mudanças de status (`[09:00] Marcos`).
- Hoje eles fazem polling no `GET /orders`, o que torna a integração lenta e cara (`[09:00] Marcos`).
- "Tempo real" para os clientes significa **qualquer latência abaixo de 10 segundos** (`[09:02] Marcos`).
- A Atlas sinalizou risco de migrar para o concorrente se a entrega não sair até o fim do trimestre (`[09:00] Marcos`).
- O fluxo é **somente outbound**: a plataforma envia para o cliente, o cliente não envia para a plataforma (`[09:02] Sofia/Marcos`).

---

## 2. Decisões fechadas

| # | Tema | Decisão | Justificativa / trade-off | Origem |
| --- | --- | --- | --- | --- |
| D1 | Padrão de entrega | **Outbox transacional no MySQL existente** | A transação principal já é pesada; chamada HTTP síncrona travaria mudanças de status e não permitiria rollback por cliente offline. Redis Streams exigiria mais infraestrutura | `[09:04] Bruno`, `[09:06] Diego`, `[09:07] Larissa/Diego` |
| D2 | Processamento | **Worker em processo separado, polling a cada 2s** | Se o worker rodar dentro da API, reinícios da API derrubam o worker. MySQL não tem listener nativo; trigger não notifica processo externo. 2s atende aos 10s | `[09:09] Diego`, `[09:11] Diego/Larissa` |
| D3 | Ordenação | **Single-worker, ordering implícita por `order_id`** | Ordem global não é garantida. Escalar para múltiplos workers exigiria particionamento ou lock pessimista, deixado para o futuro | `[09:12]–[09:13] Diego/Larissa` |
| D4 | Retry | **5 tentativas: 1m, 5m, 30m, 2h, 12h** | 3 tentativas mataria casos de indisponibilidade de 2h; retry indefinido deixaria evento pendurado para sempre. Janela total de ~15h | `[09:15]–[09:17] Diego` |
| D5 | Falha definitiva | **DLQ em tabela separada `webhook_dead_letter`** | Mantém a outbox principal limpa e serve como evidência para debug e reprocessamento. Guarda payload, motivo da falha e timestamp | `[09:18] Diego` |
| D6 | Replay | **Endpoint admin manual** `POST /admin/webhooks/dead-letter/:id/replay` | Recoloca o evento na outbox como pendente. Exige role `ADMIN` e deve registrar quem fez o replay | `[09:18] Diego`, `[09:35]–[09:36] Larissa/Sofia` |
| D7 | Autenticidade | **HMAC-SHA256 sobre o corpo do request**, header `X-Signature` | Padrão de mercado; cliente valida origem e integridade | `[09:20] Sofia` |
| D8 | Secret | **Secret única por endpoint**, rotacionável com grace period de 24h | Secret global vazaria todos os endpoints. A antiga continua válida por 24h durante a migração | `[09:21]–[09:22] Sofia` |
| D9 | Garantia de entrega | **At-least-once com `X-Event-Id`** | Cliente pode receber duplicado e deduplica pelo UUID. Exactly-once exigiria coordenação dos dois lados | `[09:24]–[09:26] Diego` |
| D10 | Estrutura | **Módulo `src/modules/webhooks`**, seguindo controller/service/repository/routes/schemas; worker em `src/worker.ts` + lógica no módulo | Reuso do padrão existente da codebase | `[09:27]–[09:28] Bruno/Diego` |
| D11 | Erros e logs | **Prefixo `WEBHOOK_`**, reuso de `AppError`, Pino e error middleware centralizado | O middleware já trata AppError, Zod e Prisma sem alteração | `[09:28]–[09:30] Bruno/Larissa` |
| D12 | Payload | **Snapshot JSON renderizado na inserção da outbox** | O evento reflete o estado do pedido no momento da mudança, mesmo que o pedido mude depois | `[09:51]–[09:52] Larissa/Diego` |
| D13 | IDs | **UUID** para a outbox | Segue o padrão do projeto, em que tudo é UUID | `[09:51] Larissa` |
| D14 | Timeout HTTP | **10 segundos** | Cliente lento é tratado como falha e entra em retry | `[09:42] Diego` |
| D15 | Limite de payload | **64KB; erro se ultrapassar** | Não truncar; evento acima disso indica problema | `[09:23]–[09:24] Sofia/Diego/Larissa` |

---

## 3. Requisitos funcionais identificados

| ID | Requisito | Detalhe | Origem |
| --- | --- | --- | --- |
| RF-01 | Criar configuração de webhook | `POST`; campos: URL, lista de status desejados; secret gerada pela plataforma e devolvida na criação; `customer_id` informado no body ou path | `[09:31] Marcos`, `[09:32] Larissa` |
| RF-02 | Listar webhooks de um customer | `GET` autenticado | `[09:33] Bruno` |
| RF-03 | Editar webhook | `PATCH` para URL, status e estado ativo | `[09:33] Bruno` |
| RF-04 | Remover webhook | `DELETE` | `[09:33] Bruno` |
| RF-05 | Filtrar eventos por status | Cada endpoint escolhe quais status quer receber; filtro aplicado **na inserção da outbox**, não no envio | `[09:33]–[09:34] Marcos/Bruno/Diego` |
| RF-06 | Histórico de entregas | `GET /webhooks/:id/deliveries` com as últimas 100 entregas: sucesso/falha, payload, response e tempo de resposta | `[09:34] Marcos` |
| RF-07 | Replay manual de DLQ | `POST /admin/webhooks/dead-letter/:id/replay`; exige `ADMIN` e auditoria de quem executou | `[09:18] Diego`, `[09:35]–[09:36]` |
| RF-08 | Rotacionar secret | Cliente solicita nova secret pela API; antiga permanece válida por 24h | `[09:21]–[09:22] Sofia` |
| RF-09 | CRUD com autenticação normal | Endpoints de configuração aceitam qualquer role autenticada por enquanto; endurecimento fica para depois | `[09:36]–[09:37] Sofia/Marcos` |

---

## 4. Requisitos não funcionais e restrições

| ID | Requisito | Detalhe | Origem |
| --- | --- | --- | --- |
| RNF-01 | Latência | Percepção de tempo real: **< 10 segundos**; polling de 2s atende | `[09:02] Marcos`, `[09:09] Diego` |
| RNF-02 | TLS obrigatório | URL precisa ser `https`; `http` é recusado com erro de validação no schema Zod | `[09:23] Sofia` |
| RNF-03 | Limite de payload | **64KB**, com erro acima disso | `[09:23]–[09:24]` |
| RNF-04 | Timeout de entrega | **10 segundos** por chamada | `[09:42] Diego` |
| RNF-05 | Garantia | **At-least-once**; duplicatas possíveis | `[09:24]–[09:26] Diego` |
| RNF-06 | Assinatura | **HMAC-SHA256** do corpo, enviado em `X-Signature` | `[09:20] Sofia` |
| RNF-07 | Isolamento do worker | Worker em processo Node separado, mesmo banco e mesma `DATABASE_URL`, mas `PrismaClient` próprio | `[09:11] Diego`, `[09:30] Bruno` |
| RNF-08 | Observabilidade | Reuso do logger Pino; erros tratados pelo middleware central | `[09:29] Bruno` |
| RNF-09 | Segurança de release | Sofia exige pelo menos **2 dias úteis** de revisão de segurança antes do deploy | `[09:46] Sofia` |

---

## 5. Arquitetura e fluxo resumido

1. O cliente altera o status de um pedido.
2. `changeStatus` executa a transação atual: atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity`.
3. **Dentro da mesma transação**, insere o evento em `webhook_outbox` com payload já renderizado (snapshot), `event_id` UUID e status pendente (`[09:40]–[09:41] Bruno/Diego`, `[09:52] Larissa`).
4. Se a inserção na outbox falhar, **toda a transação sofre rollback**. Não pode existir status alterado sem evento (`[09:40] Bruno`).
5. O worker faz polling a cada 2s, busca os eventos pendentes mais antigos em lote pequeno, processa e marca o resultado (`[09:09] Diego`).
6. Em sucesso, marca como entregue. Em falha, agenda retry conforme o backoff.
7. Após 5 falhas, move para `webhook_dead_letter`.
8. Um ADMIN pode recolocar o evento na outbox via endpoint de replay.

### Pontos de integração citados

- O método `changeStatus` será estendido com uma função do tipo `publishWebhookEvent(tx, order, fromStatus, toStatus)`, recebendo o client da transação (`[09:41] Bruno/Diego`).
- O worker reutiliza o mesmo banco e a mesma stack, mas em processo separado (`[09:11] Diego`, `[09:30] Bruno`).

---

## 6. Contrato do evento

### Payload

```json
{
  "event_id": "uuid",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-09T12:00:00.000Z",
  "order_id": "uuid",
  "order_number": "string",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "uuid",
  "total_cents": 12345
}
```

- Campos definidos em `[09:43] Diego`.
- **Não envia itens do pedido** para manter o payload enxuto; o cliente consulta `GET /orders/:id` se precisar de detalhes (`[09:43] Diego`).

### Headers

| Header | Conteúdo | Origem |
| --- | --- | --- |
| `X-Event-Id` | UUID único do evento, gerado na entrada da outbox | `[09:25] Diego` |
| `X-Signature` | HMAC-SHA256 do corpo | `[09:20] Sofia` |
| `X-Timestamp` | Timestamp do envio, para o cliente detectar replay attack se quiser | `[09:44] Diego` |
| `X-Webhook-Id` | ID do cadastro de webhook que gerou o envio | `[09:44] Sofia/Diego` |
| `Content-Type` | `application/json` | `[09:44] Diego` |

---

## 7. Fora de escopo, adiado ou descartado

| Item | Situação | Motivo | Origem |
| --- | --- | --- | --- |
| E-mail ao cliente quando o webhook falha | **Adiado** para próxima fase | Medir impacto antes | `[09:37] Larissa` |
| Rate limiting de envio por cliente | **Em aberto / observar** | Pode virar problema se houver muitas mudanças em pouco tempo; decidir depois | `[09:39] Diego/Larissa` |
| Dashboard visual para o cliente | **Fora de escopo** | Projeto separado do time de frontend; fase atual é só endpoints | `[09:39]–[09:40] Larissa` |
| Webhooks inbound (cliente → plataforma) | **Descartado** | O pedido é apenas receber notificações | `[09:02] Sofia/Marcos` |
| Ordering global | **Não garantido** | Só ordering por `order_id` e enquanto single-worker; escala futura exigiria partição/lock | `[09:12]–[09:13] Diego/Larissa` |
| Exactly-once | **Descartado** | Complexidade de coordenação dos dois lados; at-least-once resolve a maioria dos casos | `[09:25] Diego` |
| Retry indefinido | **Descartado** | Evento ficaria pendurado para sempre se o cliente sumisse | `[09:15] Diego` |
| Redis Streams | **Descartado** | Exigiria subir mais infraestrutura; outbox no MySQL existente resolve | `[09:07] Larissa/Diego` |
| Trigger do MySQL | **Descartado** | Não notifica processo externo; polling de 2s atende | `[09:09] Diego` |
| Envio síncrono no `changeStatus` | **Descartado** | Cliente lento travaria mudanças de status; não há rollback possível por falha de cliente | `[09:03]–[09:04] Bruno/Larissa` |
| Arquivamento de linhas entregues após 30 dias | **Fora do escopo desta feature** | Pode ser tratado depois | `[09:08] Diego` |

---

## 8. Questões em aberto e limitações conhecidas

- **Rate limiting:** não decidido; observar e implementar se virar problema (`[09:39]`).
- **E-mail de fallback:** adiado para a próxima fase (`[09:37]`).
- **Escala para múltiplos workers:** ordering por `order_id` só vale com single-worker; particionamento ou lock pessimista ficam para o futuro (`[09:12]–[09:13]`).
- **`customer_id` no body ou no path:** Larissa deixou em aberto ("passado no body ou no path"), sem escolha final (`[09:32]`).
- **Rota exata da rotação de secret:** o requisito existe, mas o endpoint não foi definido (`[09:21]`).
- **Arquivamento da outbox:** mencionado como necessário no futuro, sem decisão de implementação (`[09:08]`).
- **Retenção do histórico de entregas:** o requisito fala nas últimas 100 entregas, sem definir política de retenção (`[09:34]`).

---

## 9. Prazos e estimativa

- Atlas quer a entrega até **fim de novembro** (`[09:45] Marcos`).
- Estimativa da Larissa: **3 sprints**, incluindo revisão da Sofia (`[09:46]–[09:47] Larissa`).
- Sofia pede **pelo menos 2 dias úteis** para revisar HMAC e geração de secret antes do deploy (`[09:46] Sofia`).

---

## 10. Cronologia enxuta

| Horário | Assunto |
| --- | --- |
| `[09:00]` | Contexto: três clientes B2B, polling atual, risco de churn |
| `[09:02]` | Latência aceitável < 10s; apenas outbound |
| `[09:03]–[09:07]` | Síncrono descartado; outbox no MySQL escolhido; Redis descartado |
| `[09:09]–[09:11]` | Polling de 2s; worker em processo separado; `src/worker.ts` |
| `[09:12]–[09:13]` | Ordering por `order_id`; limitação conhecida |
| `[09:15]–[09:18]` | Retry 5x com backoff; DLQ separada; replay manual |
| `[09:19]–[09:23]` | HMAC-SHA256; secret por endpoint; rotação 24h; HTTPS obrigatório; limite 64KB |
| `[09:24]–[09:26]` | At-least-once; `X-Event-Id`; exactly-once descartado |
| `[09:27]–[09:30]` | Estrutura do módulo; erros `WEBHOOK_`; reuso de AppError/Pino/middleware |
| `[09:31]–[09:37]` | Requisitos funcionais: CRUD, filtro, deliveries, replay ADMIN |
| `[09:37]–[09:40]` | E-mail, rate limiting e dashboard fora de escopo |
| `[09:40]–[09:42]` | Integração no `changeStatus`; `publishWebhookEvent`; timeout 10s |
| `[09:43]–[09:45]` | Payload, headers, prazo |
| `[09:48]` | Resumo final confirmado pelos participantes |
| `[09:51]–[09:52]` | UUID na outbox; payload renderizado como snapshot |

---

## 11. Cuidados ao documentar (armadilhas)

- **Não** transformar e-mail, rate limiting ou dashboard em requisito: foram adiados ou descartados.
- **Não** afirmar que `customer_id` vem do JWT: a decisão final é body/path (`[09:32]`).
- **Não** prometer ordering global nem exactly-once.
- **Não** propor envio síncrono, Redis Streams ou trigger MySQL: todos foram descartados.
- **Não** incluir itens do pedido no payload: a decisão foi manter enxuto.
- **Não** dizer que a secret é escolhida pelo cliente: ela é gerada pela plataforma e devolvida na criação.
- **Não** esquecer que o replay de DLQ exige `ADMIN` e auditoria.
- O histórico de entregas é das **últimas 100**, não de todas.
