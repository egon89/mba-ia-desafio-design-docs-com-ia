# Tracker de Rastreabilidade

Mapeia cada item registrado nos documentos do pacote à sua origem na transcrição (`TRANSCRICAO.md`) ou no código-fonte (`src/`, `prisma/`).

**Estado deste arquivo:** cobre, por enquanto, os documentos já produzidos — `docs/adrs/ADR-001` a `ADR-007`, `docs/RFC.md` e `docs/FDD.md`. Será estendido com os itens de `docs/PRD.md` assim que for produzido.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001-DEC | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Decisão | Padrão Outbox sobre o MySQL existente para publicar eventos de webhook | TRANSCRICAO | [09:06] Diego |
| ADR-001-CTX-01 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Restrição | Disparo síncrono travaria a transação de `changeStatus` para outros pedidos | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Trade-off | Disparo síncrono descartado — sem rollback possível de HTTP call já enviado | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Trade-off | Redis Streams descartado — overengineering para o tamanho do time | TRANSCRICAO | [09:07] Diego |
| ADR-001-REQ-01 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Requisito | Outbox indexada por status e `created_at`; arquivamento após 30 dias fora de escopo | TRANSCRICAO | [09:08] Diego |
| ADR-001-COD-01 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Referência de código | Transação única de `changeStatus` que a outbox precisa se integrar | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-001-COD-02 | `docs/adrs/ADR-001-padrao-outbox-no-mysql.md` | Referência de código | Função proposta `publishWebhookEvent(tx, ...)` recebe `tx` da transação de `changeStatus` | TRANSCRICAO | [09:41] Bruno |
| ADR-002-DEC | `docs/adrs/ADR-002-worker-dedicado-com-polling.md` | Decisão | Worker em processo separado (`src/worker.ts`), polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| ADR-002-CTX-01 | `docs/adrs/ADR-002-worker-dedicado-com-polling.md` | Restrição | Worker não pode compartilhar ciclo de vida com a API (perde eventos em restart) | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-worker-dedicado-com-polling.md` | Trade-off | Trigger de banco descartado — MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| ADR-002-REQ-01 | `docs/adrs/ADR-002-worker-dedicado-com-polling.md` | Requisito Não Funcional | Polling de 2s atende requisito de latência abaixo de 10s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-COD-01 | `docs/adrs/ADR-002-worker-dedicado-com-polling.md` | Referência de código | Worker precisa de instância própria de `PrismaClient` (por processo) | CODIGO | `src/config/database.ts` |
| ADR-002-COD-02 | `docs/adrs/ADR-002-worker-dedicado-com-polling.md` | Referência de código | Novo entry point paralelo ao existente | CODIGO | `src/server.ts` |
| ADR-003-DEC | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Decisão | Retry com backoff exponencial, 5 tentativas (1m/5m/30m/2h/12h), DLQ em tabela separada | TRANSCRICAO | [09:15]-[09:18] Diego |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Trade-off | Retry indefinido descartado — evento ficaria pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Trade-off | 3 tentativas descartadas — caso real de cliente com 2h de indisponibilidade | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Trade-off | Flag "failed" na outbox descartada em favor de tabela `webhook_dead_letter` separada | TRANSCRICAO | [09:18] Diego |
| ADR-003-REQ-01 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Requisito Funcional | Reprocessamento manual de DLQ via `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | [09:18] Diego |
| ADR-003-REQ-02 | `docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md` | Requisito Não Funcional | Timeout de 10s por tentativa de entrega HTTP | TRANSCRICAO | [09:42] Diego |
| ADR-004-DEC | `docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, header `X-Signature`, secret por endpoint | TRANSCRICAO | [09:19]-[09:20] Sofia |
| ADR-004-ALT-01 | `docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Trade-off | Secret global descartada — vazamento comprometeria todos os clientes | TRANSCRICAO | [09:21] Sofia |
| ADR-004-REQ-01 | `docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Requisito Funcional | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21]-[09:22] Sofia |
| ADR-004-REQ-02 | `docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Requisito Não Funcional | URL do webhook obrigatoriamente HTTPS | TRANSCRICAO | [09:23] Sofia |
| ADR-004-REQ-03 | `docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md` | Requisito Não Funcional | Limite de 64KB de payload, erro se exceder | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego |
| ADR-005-DEC | `docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md` | Decisão | Garantia at-least-once com deduplicação via header `X-Event-Id` | TRANSCRICAO | [09:24]-[09:25] Diego |
| ADR-005-ALT-01 | `docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md` | Trade-off | Exactly-once descartada — exigiria coordenação complexa entre as partes | TRANSCRICAO | [09:25] Diego |
| ADR-006-DEC | `docs/adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md` | Decisão | Módulo `src/modules/webhooks` segue estrutura e padrões já existentes | TRANSCRICAO | [09:27]-[09:30] Bruno/Diego/Larissa |
| ADR-006-REQ-01 | `docs/adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md` | Requisito | Códigos de erro do módulo prefixados `WEBHOOK_*` | TRANSCRICAO | [09:28]-[09:29] Bruno/Larissa |
| ADR-006-COD-01 | `docs/adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md` | Referência de código | Padrão modular `controller/service/repository/routes/schemas` a seguir | CODIGO | `src/modules/orders/` |
| ADR-006-COD-02 | `docs/adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md` | Referência de código | Hierarquia de erros `AppError`/subclasses a ser reutilizada | CODIGO | `src/shared/errors/app-error.ts` |
| ADR-006-COD-03 | `docs/adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md` | Referência de código | Error middleware central, sem alteração necessária | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-COD-04 | `docs/adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md` | Referência de código | Logger Pino compartilhado, reaproveitado pelo worker | CODIGO | `src/shared/logger/index.ts` |
| ADR-006-COD-05 | `docs/adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md` | Referência de código | `requireRole` reaproveitado para o endpoint de replay de DLQ | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-COD-06 | `docs/adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md` | Referência de código | Middleware `validate` (Zod) reaproveitado para schemas do módulo | CODIGO | `src/middlewares/validate.middleware.ts` |
| ADR-007-DEC | `docs/adrs/ADR-007-ordering-por-order-id-em-single-worker.md` | Decisão | Ordering garantida apenas por `order_id`, apenas em single-worker | TRANSCRICAO | [09:12]-[09:13] Diego/Larissa |
| ADR-007-ALT-01 | `docs/adrs/ADR-007-ordering-por-order-id-em-single-worker.md` | Trade-off | Particionamento por `order_id` para múltiplos workers — adiado | TRANSCRICAO | [09:13] Diego |
| ADR-007-ALT-02 | `docs/adrs/ADR-007-ordering-por-order-id-em-single-worker.md` | Trade-off | Lock pessimista para múltiplos workers — adiado | TRANSCRICAO | [09:13] Diego |
| ADR-007-CTX-01 | `docs/adrs/ADR-007-ordering-por-order-id-em-single-worker.md` | Restrição | Clientes não pedem garantia de ordering global entre pedidos distintos | TRANSCRICAO | [09:14] Marcos |
| RFC-CTX-01 | `docs/RFC.md` | Restrição | Clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pedem notificação em tempo real; polling atual é lento e caro | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | `docs/RFC.md` | Requisito Não Funcional | "Tempo real" definido pelo cliente como latência abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | `docs/RFC.md` | Restrição | Webhook é outbound only — cliente não envia eventos para o OMS | TRANSCRICAO | [09:02] Marcos |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Visão geral da proposta: outbox + worker + HMAC + at-least-once | TRANSCRICAO | [09:48] Larissa (resumo da reunião) |
| RFC-ALT-01 | `docs/RFC.md` | Trade-off | Disparo síncrono descartado | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Trade-off | Redis Streams descartado | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | `docs/RFC.md` | Trade-off | Retry indefinido/3 tentativas descartados | TRANSCRICAO | [09:15]-[09:16] Diego |
| RFC-ALT-04 | `docs/RFC.md` | Trade-off | Exactly-once descartada | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | `docs/RFC.md` | Trade-off | Secret global descartada | TRANSCRICAO | [09:21] Sofia |
| RFC-OPEN-01 | `docs/RFC.md` | Restrição | Rate limiting de envio ao cliente — não decidido, observar depois | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| RFC-OPEN-02 | `docs/RFC.md` | Restrição | Notificação de webhook com falha recorrente (ex.: email) — adiado para fase futura | TRANSCRICAO | [09:37]-[09:38] Marcos/Larissa |
| RFC-OPEN-03 | `docs/RFC.md` | Restrição | Arquivamento/retenção de eventos entregues na outbox — não desenhado | TRANSCRICAO | [09:08] Diego |
| RFC-OPEN-04 | `docs/RFC.md` | Restrição | Escalonamento para múltiplos workers — não decidido | TRANSCRICAO | [09:13] Diego |
| RFC-RISK-01 | `docs/RFC.md` | Risco | Prazo de 3 sprints; Atlas espera entrega até fim do trimestre, risco de churn | TRANSCRICAO | [09:45]-[09:47] Marcos/Larissa |
| RFC-RISK-02 | `docs/RFC.md` | Risco | Dependência de revisão de segurança da Sofia (mín. 2 dias úteis) antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-DADOS-01 | `docs/FDD.md` | Restrição | Novas tabelas (`webhook_endpoint`, `webhook_outbox`, etc.) usam UUID como chave primária, padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-02 | `docs/FDD.md` | Decisão | Payload do evento é snapshot renderizado no momento da inserção na outbox, não recalculado no envio | TRANSCRICAO | [09:51]-[09:52] Larissa/Diego/Bruno |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | Inserção na outbox ocorre dentro da mesma transação de `changeStatus`; falha na inserção reverte a mudança de status | TRANSCRICAO | [09:40]-[09:41] Bruno/Diego |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | Worker processa a outbox em lotes via polling de 2 segundos | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Retry recalcula `next_attempt_at` conforme a progressão de backoff a cada falha | TRANSCRICAO | [09:15]-[09:17] Diego |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | Evento esgotado vai para `webhook_dead_letter`; reprocessamento só via replay admin | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | `docs/FDD.md` | Requisito Funcional | `POST /webhooks` — cadastro de webhook, secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | `docs/FDD.md` | Requisito Funcional | `GET /webhooks` — listagem dos webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Requisito Funcional | `PATCH /webhooks/:id` — edição de configuração de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Requisito Funcional | `DELETE /webhooks/:id` — remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Requisito Funcional | `POST /webhooks/:id/rotate-secret` — rotação de secret com grace period de 24h | TRANSCRICAO | [09:21]-[09:22] Sofia |
| FDD-CONTRATO-06 | `docs/FDD.md` | Requisito Funcional | `GET /webhooks/:id/deliveries` — histórico de entregas (payload, sucesso/falha, tempo de resposta) | TRANSCRICAO | [09:34]-[09:35] Marcos/Larissa |
| FDD-CONTRATO-07 | `docs/FDD.md` | Requisito Funcional | `POST /admin/webhooks/dead-letter/:id/replay` — replay manual de DLQ, role ADMIN | TRANSCRICAO | [09:18], [09:35] Diego |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Payload do evento e headers de entrega (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`) | TRANSCRICAO | [09:43]-[09:45] Diego/Sofia |
| FDD-ERRO-01 | `docs/FDD.md` | Restrição | Matriz de erros do módulo usa prefixo `WEBHOOK_*`, seguindo o padrão de códigos existente | TRANSCRICAO | [09:28]-[09:29] Bruno/Larissa |
| FDD-RESIL-01 | `docs/FDD.md` | Requisito Não Funcional | Timeout de 10s por tentativa de entrega HTTP | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | `docs/FDD.md` | Requisito Não Funcional | Progressão de backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-RESIL-03 | `docs/FDD.md` | Decisão | DLQ como fallback terminal, sem descarte de evento | TRANSCRICAO | [09:18] Diego |
| FDD-OBS-01 | `docs/FDD.md` | Referência de código | Logger Pino reaproveitado pelo módulo webhooks e pelo worker | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-02 | `docs/FDD.md` | Referência de código | Lista de redação de logs (`redactPaths`) precisa incluir `*.secret`/`*.signature` | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-03 | `docs/FDD.md` | Referência de código | Correlação de requisições HTTP via `X-Request-Id` já gerado pelo request logger | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-DEP-01 | `docs/FDD.md` | Dependência | Node >= 20 (fetch nativo disponível para o worker) | CODIGO | `package.json` |
| FDD-DEP-02 | `docs/FDD.md` | Dependência | Compatibilidade com roles `ADMIN`/`OPERATOR` do JWT existente | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-01 | `docs/FDD.md` | Referência de código | `changeStatus` estendido para publicar evento na outbox dentro da mesma transação | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Referência de código | Novas classes de erro estendem `AppError`/subclasses HTTP existentes | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-03 | `docs/FDD.md` | Referência de código | Error middleware central não precisa de alteração para tratar erros `WEBHOOK_*` | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-04 | `docs/FDD.md` | Referência de código | `requireRole('ADMIN')` reaproveitado no endpoint de replay de DLQ | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Referência de código | Instância `logger` reaproveitada pelo worker fora do ciclo de request HTTP | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-06 | `docs/FDD.md` | Referência de código | Worker cria `PrismaClient` próprio seguindo o padrão de `createPrismaClient()` | CODIGO | `src/config/database.ts` |
| FDD-INT-07 | `docs/FDD.md` | Referência de código | Schemas Zod do módulo usam o middleware `validate` genérico existente | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-08 | `docs/FDD.md` | Referência de código | Registro do novo módulo em `buildControllers`/`buildApiRouter`, mesmo padrão dos módulos existentes | CODIGO | `src/routes/index.ts` |
