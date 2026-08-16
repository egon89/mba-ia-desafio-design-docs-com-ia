# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

## Contexto e motivação técnica

O RFC (`docs/RFC.md`) e os ADRs (`docs/adrs/`) definem a arquitetura: padrão Outbox sobre o MySQL existente, worker dedicado em polling, retry com backoff e DLQ, autenticação HMAC-SHA256 e garantia at-least-once. Este documento detalha **como implementar** essa arquitetura sobre a base de código atual do OMS — modelos de dados novos, fluxos passo a passo, contratos HTTP, matriz de erros e integração concreta com os arquivos existentes, em nível suficiente para um desenvolvedor começar a codar sem decisões de design pendentes.

## Objetivos técnicos

1. Publicar um evento de webhook, de forma transacional, sempre que `OrderService.changeStatus` mudar o status de um pedido para um status que algum webhook do customer esteja escutando.
2. Entregar esse evento ao endpoint do cliente em até 10 segundos no cenário normal (sem falhas de rede), respeitando o intervalo de polling de 2s do worker (ADR-002).
3. Garantir que nenhuma mudança de status fique sem evento correspondente publicado, e que nenhum evento seja publicado sem uma mudança de status real (ADR-001).
4. Reaproveitar 100% da infraestrutura transversal já existente (erros, logger, validação, autenticação) sem exigir mudanças nesses componentes.
5. Expor APIs de gestão de configuração de webhook e de consulta de histórico de entregas, suficientes para o cliente operar sua integração sem suporte manual da equipe.

## Escopo e exclusões

**Dentro do escopo:**
- Modelagem das tabelas `webhook_endpoint`, `webhook_outbox`, `webhook_delivery` e `webhook_dead_letter`.
- Extensão de `OrderService.changeStatus` para publicar eventos na outbox, dentro da mesma transação.
- Worker (`src/worker.ts` + `src/modules/webhooks/webhook.worker.ts`) com polling, retry, backoff e movimentação para DLQ.
- Endpoints de CRUD de configuração de webhook, rotação de secret, consulta de histórico de entregas e replay de DLQ.
- Assinatura HMAC-SHA256, validação de URL HTTPS, limite de payload de 64KB.

**Fora do escopo (ver PRD, seção "Fora de escopo", e RFC, seção "Questões em aberto"):**
- Rate limiting de envio ao cliente.
- Notificação proativa (e-mail) de webhook com falha recorrente.
- Dashboard visual para o cliente.
- Arquivamento automático de linhas entregues na outbox.
- Suporte a múltiplos workers em paralelo (ordering global).

## Modelagem de dados (novas tabelas)

> Documentação apenas — a alteração de `prisma/schema.prisma` não faz parte deste desafio, que é puramente documental.

**`webhook_endpoint`** — configuração de um endpoint de cliente
| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | UUID (PK) | Padrão do projeto ([09:51] Larissa) |
| `customer_id` | UUID (FK → `customers.id`) | |
| `url` | string | HTTPS obrigatório |
| `events` | JSON (array de `OrderStatus`) | status que o endpoint quer receber |
| `secret` | string (criptografado em repouso) | gerada na criação |
| `previous_secret` | string, nullable | usado durante o grace period de rotação |
| `previous_secret_expires_at` | datetime, nullable | fim do grace period de 24h |
| `active` | boolean, default true | |
| `created_at` / `updated_at` | datetime | |

**`webhook_outbox`** — fila de trabalho do worker
| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` (`event_id`) | UUID (PK) | vai no header `X-Event-Id` |
| `webhook_endpoint_id` | UUID (FK) | destino da entrega |
| `event_type` | string | ex.: `order.status_changed` |
| `payload` | JSON | snapshot renderizado na inserção ([09:51]-[09:52] Larissa/Diego/Bruno) |
| `status` | enum (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`) | índice |
| `attempt_count` | int, default 0 | |
| `next_attempt_at` | datetime | usado pelo backoff |
| `created_at` | datetime | índice, usado para ordenação FIFO por `order_id` |

**`webhook_delivery`** — histórico de tentativas (fonte do endpoint de deliveries)
| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | UUID (PK) | |
| `webhook_endpoint_id` | UUID (FK) | |
| `event_id` | UUID (FK → `webhook_outbox.id`) | |
| `attempt_number` | int | 1 a 5 |
| `outcome` | enum (`SUCCESS`, `FAILURE`) | |
| `http_status_code` | int, nullable | nulo em timeout |
| `response_time_ms` | int | |
| `response_snippet` | string, nullable | primeiros bytes da resposta, truncado |
| `created_at` | datetime | |

**`webhook_dead_letter`** — eventos que esgotaram o retry
| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | UUID (PK) | |
| `webhook_endpoint_id` | UUID (FK) | |
| `event_id` | UUID | evento original |
| `payload` | JSON | cópia do payload |
| `failure_reason` | string | último erro registrado |
| `attempts_made` | int | |
| `created_at` | datetime | |

## Fluxos detalhados

### 1. Criação do evento na outbox

1. `OrderController.changeStatus` chama `OrderService.changeStatus(id, input, userId)` normalmente (`src/modules/orders/order.controller.ts:38-46`).
2. Dentro da transação já existente (`src/modules/orders/order.service.ts:131-178`), após o `update` do status e o `create` do `order_status_history`, o service chama uma nova função `publishWebhookEvent(tx, order, from, to)` do módulo `webhooks`.
3. `publishWebhookEvent`:
   a. Busca, via `tx`, os `webhook_endpoint` ativos do `customerId` do pedido cujo campo `events` contenha `to` (o novo status).
   b. Se nenhum endpoint estiver escutando esse status, não insere nada (filtro na inserção, [09:33]-[09:34] Bruno/Diego) — retorna sem efeito colateral.
   c. Para cada endpoint encontrado, monta o payload (snapshot do pedido no momento da transição) e insere uma linha em `webhook_outbox` com `status = PENDING`, `attempt_count = 0`, `next_attempt_at = now()`.
4. Se a inserção na outbox falhar (ex.: erro de banco), a exceção propaga e toda a transação de `changeStatus` sofre rollback — a mudança de status nunca é persistida sem o evento correspondente ([09:40]-[09:41] Bruno/Diego).
5. A transação faz commit: pedido, histórico e evento(s) de outbox tornam-se visíveis atomicamente.

### 2. Processamento pelo worker

1. `src/worker.ts` inicializa seu próprio `PrismaClient` (independente do da API) e entra em loop.
2. A cada 2 segundos, `webhook.worker.ts` executa uma consulta: `SELECT ... FROM webhook_outbox WHERE status = 'PENDING' AND next_attempt_at <= now() ORDER BY created_at ASC LIMIT <batch_size>`.
3. Para cada linha do lote, marca `status = 'PROCESSING'` (evita reprocessamento concorrente caso o batch size cresça no futuro) e:
   a. Busca `webhook_endpoint` associado (url, secret ativa).
   b. Calcula a assinatura HMAC-SHA256 do payload com a secret.
   c. Envia `POST` ao `url` do endpoint com headers (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json`) e timeout de 10s ([09:42] Diego).
   d. Registra o resultado (sucesso ou falha) em `webhook_delivery`.
4. Em caso de sucesso (2xx): marca `webhook_outbox.status = 'DELIVERED'`.
5. Em caso de falha (não-2xx, timeout, erro de rede): segue para o fluxo de retry (abaixo).

### 3. Retry com backoff exponencial

1. Ao falhar uma tentativa, o worker incrementa `attempt_count` e calcula `next_attempt_at = now() + backoff[attempt_count]`, onde `backoff = [1m, 5m, 30m, 2h, 12h]` ([09:15]-[09:17] Diego/Larissa).
2. `webhook_outbox.status` volta para `'PENDING'`, então a linha reentra no lote de leitura do worker quando `next_attempt_at` for atingido.
3. Se `attempt_count` já atingiu 5 e a tentativa falhou novamente, o evento **não** volta para `PENDING` — segue para o fluxo de DLQ.

### 4. Dead Letter Queue (DLQ)

1. Ao esgotar as 5 tentativas, o worker insere uma linha em `webhook_dead_letter` com o payload, o motivo da última falha e `attempts_made = 5`, e marca a linha original em `webhook_outbox` como `'FAILED'` (mantida para auditoria, não deletada).
2. Um administrador (role `ADMIN`) chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`.
3. O endpoint cria uma **nova** linha em `webhook_outbox` (mesmo `event_id` original, `attempt_count = 0`, `status = 'PENDING'`, `next_attempt_at = now()`), reentrando no fluxo normal do worker, e registra log de auditoria (`userId` do administrador, `event_id`, timestamp) via logger Pino ([09:36] Sofia).

## Contratos públicos

Prefixo base: `/api/v1`. Todos os endpoints (exceto o de replay administrativo) exigem `authenticate` (JWT); nenhuma role específica é exigida para o CRUD de configuração, conforme decidido na reunião ([09:36]-[09:37] Sofia).

### 1. `POST /webhooks` — cadastrar webhook

Cria um endpoint de webhook para um customer. A secret é gerada pelo sistema e devolvida **somente nesta resposta** (não é recuperável depois).

Request:
```json
{
  "customerId": "5f2c9a10-1b3e-4b4a-9b1e-2f6c7d8a9b10",
  "url": "https://integrations.atlascomercial.com/hooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:
```json
{
  "id": "8a1e2f3c-4d5b-4e6a-9c7d-1a2b3c4d5e6f",
  "customerId": "5f2c9a10-1b3e-4b4a-9b1e-2f6c7d8a9b10",
  "url": "https://integrations.atlascomercial.com/hooks/oms",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "secret": "whsec_8f3a1c...redacted-in-logs",
  "active": true,
  "createdAt": "2026-08-16T14:00:00.000Z"
}
```

Erros possíveis: `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_INVALID_EVENT_LIST`, `404 WEBHOOK_CUSTOMER_NOT_FOUND`, `401 UNAUTHORIZED`.

### 2. `GET /webhooks` — listar webhooks

Query params: `customerId` (obrigatório), `page`, `pageSize` — mesmo padrão de paginação de `GET /orders` (`src/modules/orders/order.schemas.ts`).

Response `200 OK`:
```json
{
  "data": [
    {
      "id": "8a1e2f3c-4d5b-4e6a-9c7d-1a2b3c4d5e6f",
      "customerId": "5f2c9a10-1b3e-4b4a-9b1e-2f6c7d8a9b10",
      "url": "https://integrations.atlascomercial.com/hooks/oms",
      "events": ["PAID", "SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-08-16T14:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

A `secret` nunca é retornada em listagens ou consultas — apenas na criação e na rotação.

### 3. `PATCH /webhooks/:id` — editar webhook

Request (campos parciais):
```json
{ "events": ["SHIPPED", "DELIVERED"], "active": true }
```

Response `200 OK`: mesmo formato do item de `GET /webhooks/:id`. Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_INVALID_EVENT_LIST`.

### 4. `DELETE /webhooks/:id` — remover webhook

Response `204 No Content`. Erros: `404 WEBHOOK_NOT_FOUND`.

### 5. `POST /webhooks/:id/rotate-secret` — rotacionar secret

Gera uma nova secret; a anterior permanece válida por 24h ([09:21]-[09:22] Sofia).

Response `200 OK`:
```json
{
  "id": "8a1e2f3c-4d5b-4e6a-9c7d-1a2b3c4d5e6f",
  "secret": "whsec_new7b2c...redacted-in-logs",
  "previousSecretExpiresAt": "2026-08-17T14:00:00.000Z"
}
```

Erros: `404 WEBHOOK_NOT_FOUND`.

### 6. `GET /webhooks/:id/deliveries` — histórico de entregas

Query params: `page`, `pageSize` (mais recentes primeiro, últimas 100 por padrão — [09:34] Marcos).

Response `200 OK`:
```json
{
  "data": [
    {
      "eventId": "3c1d2e3f-4a5b-4c6d-8e9f-0a1b2c3d4e5f",
      "eventType": "order.status_changed",
      "attemptNumber": 2,
      "outcome": "SUCCESS",
      "httpStatusCode": 200,
      "responseTimeMs": 184,
      "createdAt": "2026-08-16T14:05:12.000Z"
    },
    {
      "eventId": "3c1d2e3f-4a5b-4c6d-8e9f-0a1b2c3d4e5f",
      "eventType": "order.status_changed",
      "attemptNumber": 1,
      "outcome": "FAILURE",
      "httpStatusCode": null,
      "responseTimeMs": 10000,
      "createdAt": "2026-08-16T14:00:12.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 2, "totalPages": 1 }
}
```

Erros: `404 WEBHOOK_NOT_FOUND`.

### 7. `POST /admin/webhooks/dead-letter/:id/replay` — reprocessar evento em DLQ

Exige `authenticate` + `requireRole('ADMIN')` ([09:35]-[09:36] Larissa/Sofia).

Response `202 Accepted`:
```json
{ "eventId": "3c1d2e3f-4a5b-4c6d-8e9f-0a1b2c3d4e5f", "status": "PENDING", "requeuedAt": "2026-08-16T15:00:00.000Z" }
```

Erros: `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`, `403 FORBIDDEN` (role diferente de ADMIN), `401 UNAUTHORIZED`.

### 8. Payload de evento enviado ao cliente (não é um endpoint do OMS, é o corpo do `POST` que o worker faz para o cliente)

Headers: `Content-Type: application/json`, `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` ([09:44]-[09:45] Diego/Sofia).

```json
{
  "event_id": "3c1d2e3f-4a5b-4c6d-8e9f-0a1b2c3d4e5f",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-16T14:00:12.000Z",
  "order_id": "9e8d7c6b-5a4f-4e3d-9c2b-1a0f8e7d6c5b",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "5f2c9a10-1b3e-4b4a-9b1e-2f6c7d8a9b10",
  "total_cents": 15990
}
```

Sem `items` — payload enxuto ([09:43] Diego); cliente consulta `GET /orders/:id` se precisar de detalhe.

## Matriz de erros

| Código | Status HTTP | Quando ocorre | Classe base sugerida |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `id` de webhook inexistente em `GET/PATCH/DELETE/:id`, `rotate-secret`, `deliveries` | `NotFoundError` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` informado não existe | `NotFoundError` |
| `WEBHOOK_INVALID_URL` | 400 | URL ausente, malformada ou não-HTTPS | `ValidationError` |
| `WEBHOOK_INVALID_EVENT_LIST` | 400 | lista `events` vazia ou com status inválido (fora do enum `OrderStatus`) | `ValidationError` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | payload do evento excede 64KB no momento da inserção na outbox | `UnprocessableEntityError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `id` de entrada de DLQ inexistente no replay | `NotFoundError` |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | tentativa de replay de uma entrada de DLQ já reenfileirada | `ConflictError` |
| `WEBHOOK_SECRET_ROTATION_IN_PROGRESS` | 409 | nova rotação solicitada antes do grace period da rotação anterior expirar | `ConflictError` |
| `WEBHOOK_INACTIVE` | 409 | tentativa de operar sobre um webhook desativado (ex.: replay para endpoint inativo) | `ConflictError` |

Todas seguem o mesmo formato de resposta do restante da API (`{ error: { code, message, details? } }`), produzido pelo `error.middleware.ts` já existente — nenhuma classe nova precisa de tratamento especial nesse middleware, desde que estenda `AppError`.

## Estratégias de resiliência

- **Timeout de entrega:** 10 segundos por tentativa HTTP ([09:42] Diego/Sofia). Implementado via `AbortController`/timeout do cliente HTTP usado pelo worker.
- **Retry com backoff exponencial:** 5 tentativas, `1m → 5m → 30m → 2h → 12h` (ADR-003). Calculado a partir de `next_attempt_at` na própria linha da outbox — não depende de estado em memória do worker, então sobrevive a restart do processo.
- **DLQ como fallback terminal:** eventos que esgotam o retry não são descartados, ficam disponíveis para reprocessamento manual via endpoint admin (ADR-003).
- **Rollback atômico na origem:** falha ao inserir na outbox aborta a transação de `changeStatus` inteira (ADR-001) — o sistema nunca fica em estado "status mudou, evento não existe".
- **Circuit breaker por endpoint:** **não implementado nesta fase**. Cada tentativa é isolada por evento; um endpoint permanentemente fora do ar apenas acumula eventos em retry/DLQ, sem interromper o processamento de outros endpoints (o worker processa a fila global, não por endpoint). Fica registrado como possível melhoria futura, fora do escopo atual (mesma linha de "observar e decidir depois" do rate limiting — ver RFC, "Questões em aberto").
- **Idempotência do lado do cliente:** garantida pela combinação `X-Event-Id` único e estável por evento em todas as tentativas/reentregas (ADR-005).

## Observabilidade

- **Métricas** (a expor via o mesmo processo de coleta usado no restante da aplicação, quando existir; nesta fase, mínimo via logs estruturados agregáveis):
  - Contador de eventos publicados na outbox, por `event_type` e `to_status`.
  - Contador de entregas por resultado (`SUCCESS`/`FAILURE`) e por tentativa (`attempt_number`).
  - Histograma de `responseTimeMs` das entregas.
  - Gauge do tamanho do backlog da outbox (`COUNT(*) WHERE status = 'PENDING'`), para alertar acúmulo anormal.
  - Contador de eventos movidos para DLQ.
- **Logs:** todo o módulo usa a instância `logger` (Pino) já existente (`src/shared/logger/index.ts`). Cada tentativa de entrega gera um log estruturado (`webhook_delivery_attempt`) com `eventId`, `webhookEndpointId`, `attemptNumber`, `outcome`, `httpStatusCode`, `durationMs`. O replay de DLQ gera um log de auditoria (`webhook_dlq_replay`) com `userId` do administrador. Segredos (`secret`, `X-Signature`) nunca são logados — a lista de redação do logger (`redactPaths` em `src/shared/logger/index.ts`) deve ser estendida com `*.secret` e `*.signature`.
- **Tracing:** a aplicação não usa hoje nenhuma biblioteca de tracing distribuído (não há OpenTelemetry no `package.json`). Nesta fase, a correlação é feita via `event_id` (presente em todo log relacionado ao ciclo de vida do evento, do insert na outbox até a entrega ou DLQ) e via `X-Request-Id` para as chamadas HTTP recebidas pela API (`src/middlewares/request-logger.middleware.ts`). Adotar tracing distribuído fica registrado como melhoria futura, fora do escopo desta feature.

## Dependências e compatibilidade

- MySQL (mesma instância/`DATABASE_URL` da API), via Prisma — nenhuma dependência de banco nova.
- Nenhuma biblioteca nova estritamente necessária para HMAC (módulo `crypto` nativo do Node cobre HMAC-SHA256) nem para o cliente HTTP do worker (pode reaproveitar `fetch` nativo do Node 20, já que o projeto exige `node >= 20` em `package.json`).
- Compatível com o schema de autenticação/autorização atual (`AuthUser` com `role: 'ADMIN' | 'OPERATOR'`, `src/middlewares/auth.middleware.ts`) — nenhuma mudança no formato do JWT é necessária.
- Worker é um novo artefato de deploy/processo — requer atualização de scripts de infraestrutura/orquestração (fora do código da aplicação) para garantir exatamente uma instância rodando.

## Critérios de aceite técnicos

- Uma mudança de status de pedido para um status observado por ao menos um webhook ativo resulta em exatamente uma linha nova em `webhook_outbox` por endpoint interessado, inserida na mesma transação de `changeStatus`.
- Se a inserção na outbox falhar, a mudança de status inteira sofre rollback (nenhuma divergência possível entre `orders.status` e eventos publicados).
- Uma entrega bem-sucedida (2xx) marca o evento como `DELIVERED` e não gera nova tentativa.
- Uma entrega falha (timeout ou não-2xx) incrementa `attempt_count` e agenda `next_attempt_at` conforme a progressão `1m/5m/30m/2h/12h`.
- Ao atingir 5 tentativas falhas, o evento aparece em `webhook_dead_letter` e não é mais reprocessado automaticamente.
- O replay de DLQ só é aceito para usuários com `role = ADMIN` e gera log de auditoria.
- Toda requisição de entrega ao cliente inclui os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, e a assinatura é validável recalculando HMAC-SHA256 do corpo com a secret ativa (ou a secret anterior, se dentro do grace period de 24h).
- Cadastro de webhook com URL não-HTTPS é rejeitado com `WEBHOOK_INVALID_URL` antes de qualquer persistência.
- Payload de evento acima de 64KB não é inserido na outbox; a operação de origem falha com `WEBHOOK_PAYLOAD_TOO_LARGE`.

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Backlog da outbox cresce além da capacidade de leitura do worker (pico de mudanças de status) | Média | Alto (latência de entrega sobe além dos 10s esperados) | Monitorar gauge de backlog (ver Observabilidade); processar em lote configurável; escalar batch size antes de considerar múltiplos workers (ADR-007 documenta o trade-off de ordering ao escalar) |
| `PrismaClient` duplicado (API + worker) esgota o pool de conexões do MySQL | Baixa | Alto (falhas em toda a aplicação, não só em webhooks) | Dimensionar `connection_limit` do Prisma para cada processo considerando o total; validar em ambiente de staging antes do deploy |
| Secret vazada em log ou repositório de código do cliente | Média (já ocorreu antes, [09:22] Diego) | Alto (integridade dos webhooks daquele cliente comprometida) | Rotação com grace period de 24h (ADR-004) já mitigada por desenho; reforçar redação de logs (`redactPaths`) no próprio OMS |
| Endpoint de cliente permanentemente indisponível gera acúmulo constante em DLQ sem ninguém notar | Média | Médio (cliente para de receber notificações sem que a equipe perceba) | Métrica de contagem de DLQ por webhook e revisão periódica; alerta proativo fica registrado como melhoria futura (ver RFC, "Questões em aberto") |

## Integração com o sistema existente

1. **`src/modules/orders/order.service.ts`** — método `changeStatus` (linhas 126-179) é o ponto de integração central: após o `update` do status e o `create` em `order_status_history`, e ainda dentro da mesma transação Prisma (`this.prisma.$transaction`), passa a chamar `publishWebhookEvent(tx, order, from, to)` do novo módulo `webhooks`. Nenhuma outra parte do método muda; a função recebe o `tx` (Prisma `TransactionClient`) já aberto, sem exigir refatoração da assinatura pública do `OrderService` nem injeção de um repository de webhooks inteiro nele ([09:41] Bruno/Diego).

2. **`src/shared/errors/http-errors.ts`** (e `src/shared/errors/app-error.ts`) — todas as novas classes de erro do módulo webhooks (`WebhookNotFoundError`, `WebhookInvalidUrlError`, `WebhookPayloadTooLargeError`, etc.) estendem `AppError` ou uma das subclasses HTTP já existentes (`NotFoundError`, `ValidationError`, `ConflictError`, `UnprocessableEntityError`), do mesmo jeito que `InsufficientStockError extends UnprocessableEntityError` e `InvalidStatusTransitionError extends ConflictError` hoje. Os códigos seguem o prefixo `WEBHOOK_*` (ADR-006).

3. **`src/middlewares/error.middleware.ts`** — não recebe nenhuma alteração. Por já serializar qualquer instância de `AppError` para `{ error: { code, message, details? } }` de forma genérica (linhas 14-24), absorve as novas classes do item 2 automaticamente, exatamente como absorve os erros de `orders` hoje.

4. **`src/middlewares/auth.middleware.ts`** — a função `requireRole('ADMIN')`, já usada em `src/modules/users/user.routes.ts` para restringir `GET /users/:id`, é reaproveitada sem modificação para proteger `POST /admin/webhooks/dead-letter/:id/replay` (ADR-003, [09:36] Larissa). Os demais endpoints do módulo usam apenas `authenticate`, igual a `src/modules/orders/order.routes.ts`.

5. **`src/shared/logger/index.ts`** — a instância `logger` (Pino) exportada por esse módulo é importada tanto pelos handlers HTTP do módulo `webhooks` quanto pelo processo `src/worker.ts`, para os logs de tentativa de entrega e de auditoria de replay descritos na seção Observabilidade. A lista `redactPaths` definida ali precisa ganhar `*.secret` e `*.signature` para não expor segredos em log.

6. **`src/config/database.ts`** — o padrão `createPrismaClient()`/`prisma` singleton usado por `src/server.ts` é replicado (não reaproveitado por importação direta) em `src/worker.ts`: o worker cria sua própria instância de `PrismaClient` apontando para o mesmo `DATABASE_URL` (`src/config/env.ts`), pois `PrismaClient` é vinculado ao processo Node em que é instanciado ([09:29]-[09:30] Diego/Bruno).

7. **`src/middlewares/validate.middleware.ts`** — os schemas Zod do módulo `webhooks` (`webhook.schemas.ts`) usam o mesmo middleware genérico `validate({ body, query, params })` já usado por `order.routes.ts`, `user.routes.ts` etc., incluindo a regra de URL HTTPS (`z.string().url().refine(...)`) e o limite de 64KB do payload.

8. **`src/routes/index.ts`** e **`src/app.ts`** — o novo `WebhookController` é registrado em `buildControllers` (`src/app.ts`) e seu router (`buildWebhookRouter`) é montado em `buildApiRouter` (`src/routes/index.ts`), seguindo exatamente o padrão de composição manual já usado para `orders`, `products`, `customers` e `users`.
