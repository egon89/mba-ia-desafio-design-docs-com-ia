# PRD — Product Requirements Document: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O OMS hoje não tem nenhum mecanismo de notificação externa: clientes que precisam saber quando um pedido muda de status são obrigados a fazer polling periódico em `GET /orders`. Esta feature introduz **webhooks outbound**: sempre que o status de um pedido muda, o sistema notifica automaticamente, via HTTP, os endpoints cadastrados pelo cliente interessados naquele status, com garantias de entrega, segurança e auditabilidade equivalentes ao padrão adotado por plataformas como Stripe e GitHub. A decisão técnica foi fechada em reunião entre Tech Lead, PM, engenharia e segurança (`TRANSCRICAO.md`); este PRD, o RFC (`docs/RFC.md`), o FDD (`docs/FDD.md`) e os ADRs (`docs/adrs/`) consolidam essa decisão em documentação acionável.

## Problema e motivação

Três clientes B2B — **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** — pediram formalmente notificação em tempo real de mudança de status de pedido. Hoje eles fazem polling manual em `GET /orders`, o que torna a integração deles lenta e cara. A Atlas Comercial sinalizou risco concreto de migrar para um concorrente caso a feature não seja entregue até o fim do trimestre. Para esses clientes, "tempo real" significa qualquer latência de notificação abaixo de 10 segundos — o requisito não é streaming nem latência sub-segundo, é eliminar a necessidade de polling manual.

## Público-alvo e cenários de uso

**Público-alvo:**
- Sistemas de integração de clientes B2B (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo), consumindo a API do OMS via credenciais de usuário já existentes.
- Administradores internos (role `ADMIN`) responsáveis por operar falhas de entrega (replay de DLQ).
- Equipe de engenharia que vai construir e manter o módulo `webhooks`.

**Cenários de uso:**
1. Um integrador do cliente cadastra um webhook informando URL e os status de pedido que quer acompanhar, e recebe a secret gerada pelo sistema.
2. Um pedido do cliente muda de `PAID` para `PROCESSING`; o cliente recebe uma notificação HTTP assinada em poucos segundos, sem precisar consultar a API.
3. O cliente perde uma notificação (endpoint fora do ar); o sistema tenta novamente automaticamente, e o cliente pode consultar o histórico de entregas para investigar.
4. O cliente suspeita que sua secret vazou; ele rotaciona a secret pela API sem perder eventos durante a transição.
5. Um evento falha em todas as tentativas automáticas; um administrador do OMS reprocessa manualmente esse evento específico.

## Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| Eliminar a necessidade de polling manual, entregando notificações dentro da janela definida pelo cliente como "tempo real" | Latência de entrega (do commit da mudança de status até a chamada HTTP ao cliente), medida em produção | P95 < 10 segundos ([09:02] Marcos) |
| Cumprir o compromisso comercial assumido com a Atlas Comercial | Prazo de entrega da feature em produção | Até o fim do trimestre, dentro da estimativa de 3 sprints incluindo revisão de segurança ([09:45]-[09:47] Larissa/Marcos) |
| Garantir confiabilidade de entrega mesmo com indisponibilidade temporária do cliente | Janela de retry coberta antes de mover para DLQ | 5 tentativas, cobrindo até ~15 horas de indisponibilidade do cliente ([09:15]-[09:17] Diego) |
| Garantir consistência entre estado do pedido e eventos publicados | Divergência entre `orders.status` e eventos na outbox | Zero — toda mudança de status elegível gera evento na mesma transação, sem exceção ([09:40]-[09:41] Bruno/Diego) |

## Escopo

### Incluso
- Cadastro, edição, remoção e listagem de webhooks por customer (RF1, RF3, RF4, RF5).
- Filtro de eventos por status de pedido, aplicado na inserção do evento (RF6).
- Consulta de histórico de entregas por webhook (RF7).
- Endpoint administrativo de replay manual de eventos em DLQ, restrito a role `ADMIN` (RF8).
- Rotação de secret com grace period de 24h (RF11).
- Entrega assinada (HMAC-SHA256), com headers de idempotência e identificação (RF12).
- Worker dedicado, retry com backoff exponencial e DLQ.

### Fora de escopo
- **Notificação por e-mail em caso de falha recorrente de entrega.** Levantado por Marcos como possível melhoria, explicitamente descartado desta fase pela Larissa: "Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto." ([09:37]-[09:38] Marcos/Larissa).
- **Dashboard visual para o cliente acompanhar webhooks.** Descartado desta fase — a entrega é só por API; painel visual seria projeto separado do time de frontend ([09:39]-[09:40] Marcos/Larissa).
- **Rate limiting de envio ao cliente.** Levantado por Diego como preocupação real (rajada de eventos podendo sobrecarregar o endpoint do cliente), mas deixado como ponto em aberto — "observar e decidir depois", não faz parte desta entrega ([09:38]-[09:39] Diego/Larissa).
- **Garantia de ordering global entre pedidos diferentes.** Ordering só é garantida por `order_id`, e apenas enquanto houver um único worker; escalar para múltiplos workers é problema futuro, não resolvido nesta feature ([09:12]-[09:13] Diego/Larissa; ver ADR-007).
- **Arquivamento automático de eventos entregues na outbox.** Mencionado como necessário eventualmente (30 dias), mas fora do escopo desta entrega ([09:08] Diego).

## Requisitos funcionais

1. **RF1** — O cliente deve poder cadastrar um webhook via `POST`, informando `url` e a lista de status de pedido desejados; a `secret` é gerada pelo sistema e devolvida na resposta de criação ([09:31] Marcos).
2. **RF2** — O cadastro é feito por um endpoint autenticado com o JWT do sistema (usuário operador/admin do OMS), não com credencial própria do cliente; o `customer_id` é informado explicitamente, não inferido do token ([09:32] Bruno/Marcos/Larissa).
3. **RF3** — O cliente deve poder editar (`PATCH`) a configuração de um webhook existente ([09:33] Bruno).
4. **RF4** — O cliente deve poder remover (`DELETE`) um webhook ([09:33] Bruno).
5. **RF5** — O cliente deve poder listar (`GET`) os webhooks cadastrados de um customer ([09:33] Bruno).
6. **RF6** — Por webhook, o cliente escolhe quais status de pedido quer receber; eventos de status não escutados não geram notificação ([09:33]-[09:34] Marcos/Bruno).
7. **RF7** — O cliente deve poder consultar o histórico de entregas de um webhook (sucesso/falha, payload, resposta, tempo de resposta) via `GET /webhooks/:id/deliveries` ([09:34]-[09:35] Marcos/Larissa).
8. **RF8** — Um administrador (role `ADMIN`) deve poder reprocessar manualmente um evento em DLQ via `POST /admin/webhooks/dead-letter/:id/replay`, com log de auditoria de quem executou ([09:18], [09:34]-[09:36] Diego/Sofia/Larissa).
9. **RF9** — Toda mudança de status de pedido elegível deve ser notificada em até 10 segundos no cenário sem falhas ([09:02] Marcos).
10. **RF10** — O sistema deve garantir que um evento só existe se, e somente se, a transação de mudança de status correspondente foi commitada (sem divergência possível) ([09:40]-[09:41] Bruno/Diego).
11. **RF11** — O cliente deve poder rotacionar a secret do seu webhook via API, com a secret anterior permanecendo válida por 24h em paralelo ([09:21]-[09:22] Sofia).
12. **RF12** — Cada entrega deve ser assinada (HMAC-SHA256) e incluir headers de identificação (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`) para o cliente validar autenticidade e deduplicar ([09:19]-[09:20], [09:25], [09:44]-[09:45] Sofia/Diego).

## Requisitos não funcionais

- **Latência:** P95 de entrega abaixo de 10 segundos, com worker em polling de 2 segundos como piso de latência mínima aceitável ([09:02] Marcos, [09:10] Larissa).
- **Segurança em trânsito:** URL de webhook deve ser obrigatoriamente HTTPS; cadastro com HTTP é recusado na validação ([09:23] Sofia).
- **Segurança de payload:** entregas assinadas com HMAC-SHA256, secret exclusiva por endpoint, rotacionável com grace period de 24h ([09:19]-[09:22] Sofia).
- **Limite de payload:** eventos maiores que 64KB são rejeitados na origem, não truncados ([09:23]-[09:24] Sofia/Diego).
- **Confiabilidade de entrega:** garantia at-least-once, com deduplicação de responsabilidade do cliente via `X-Event-Id` ([09:24]-[09:25] Diego).
- **Resiliência a indisponibilidade do cliente:** retry com backoff exponencial (5 tentativas, até ~15h de janela) antes de mover para DLQ ([09:15]-[09:17] Diego).
- **Isolamento operacional:** processamento de entrega roda em processo separado da API, não pode ser afetado por deploys/restarts da API ([09:11] Diego).
- **Consistência transacional na origem:** inserção do evento ocorre na mesma transação SQL da mudança de status ([09:06] Diego, [09:40]-[09:41] Bruno/Diego).

## Decisões e trade-offs principais

Documentadas em detalhe nos ADRs correspondentes:

- Padrão Outbox sobre o MySQL existente, em vez de fila dedicada nova — [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md).
- Worker em processo separado, com polling de 2s, em vez de mecanismo reativo (indisponível no MySQL) — [ADR-002](adrs/ADR-002-worker-dedicado-com-polling.md).
- Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada, em vez de retry indefinido ou tentativas mais agressivas — [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md).
- HMAC-SHA256 com secret por endpoint (não global), com rotação e grace period — [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md).
- Garantia at-least-once com dedup por `X-Event-Id`, em vez de exactly-once — [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md).
- Módulo `webhooks` reaproveitando integralmente os padrões de erro, logging, autenticação e validação já existentes no OMS — [ADR-006](adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md).
- Ordering de eventos garantida apenas por `order_id`, apenas em single-worker — limitação conhecida e aceita — [ADR-007](adrs/ADR-007-ordering-por-order-id-em-single-worker.md).

## Dependências

- Banco de dados MySQL já em produção (via Prisma) — nenhuma infraestrutura nova de mensageria.
- Novo processo de deploy para o worker (`src/worker.ts`), exigindo ajuste de orquestração/infraestrutura para garantir exatamente uma instância ativa.
- Revisão de segurança dedicada da Sofia (mínimo 2 dias úteis) antes do deploy em produção, com foco em HMAC e geração/armazenamento de secret ([09:46]-[09:47] Sofia/Larissa).
- Documentação no portal do desenvolvedor, a cargo do Marcos, explicando integração, formato de payload e deduplicação por `X-Event-Id` ([09:26], [09:40] Marcos).
- Confirmação de prazo com a Atlas Comercial, a cargo do Marcos, condicionada à estimativa de 3 sprints ([09:47] Marcos).

## Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Atraso na entrega além do prazo comunicado à Atlas Comercial (fim do trimestre) | Média | Alto — risco explícito de churn do cliente ([09:00] Marcos) | Escopo desta fase deliberadamente reduzido (e-mail de alerta, dashboard, rate limiting adiados) para viabilizar a estimativa de 3 sprints; acompanhamento de progresso sprint a sprint |
| Revisão de segurança da Sofia identifica problema tarde no ciclo, atrasando o deploy | Baixa-Média | Alto — HMAC e geração de secret são caminho crítico de segurança | Reservar os 2 dias úteis de revisão já no planejamento das sprints, não no fim; revisar design de secret (ADR-004) com a Sofia antes da implementação começar |
| Cliente integrador implementa incorretamente a deduplicação por `X-Event-Id` ou a validação HMAC, gerando processamento duplicado ou falha de autenticação do lado dele | Média | Médio — impacto concentrado no cliente, mas gera suporte adicional para a equipe | Documentação clara e exemplos no portal do desenvolvedor (responsabilidade do Marcos); manter payload e headers estáveis e bem especificados no FDD |
| Backlog da tabela `webhook_outbox` cresce além da capacidade de leitura do worker em picos de mudança de status | Média | Alto — latência de entrega ultrapassa a meta de 10s (P95) | Monitorar volume de eventos pendentes (ver FDD, seção Observabilidade); reavaliar tamanho de lote do worker antes de considerar múltiplos workers, que reabriria a limitação de ordering (ADR-007) |

## Critérios de aceitação

- Um cliente cadastrado consegue, via API, criar, listar, editar e remover webhooks, e recebe a secret apenas no momento da criação.
- Uma mudança de status de pedido elegível (customer com webhook ativo escutando aquele status) resulta em uma notificação HTTP assinada, entregue dentro da meta de latência definida.
- Nenhuma mudança de status ocorre sem o evento correspondente ser publicado, e vice-versa (garantia transacional).
- Falhas de entrega são reentregues automaticamente conforme a política de retry, e eventos que esgotam o retry ficam disponíveis em DLQ para replay manual por um `ADMIN`.
- Rotação de secret não interrompe a entrega: a secret anterior continua sendo aceita durante as 24h de grace period.
- O histórico de entregas de um webhook está disponível via API, incluindo tentativas falhas.
- Cadastro de webhook com URL não-HTTPS é rejeitado antes de qualquer persistência.

## Estratégia de testes e validação

- **Testes unitários** para a extensão de `OrderService.changeStatus` (`publishWebhookEvent`), cobrindo: evento inserido quando há webhook interessado, nenhum evento inserido quando não há, e rollback completo da transação quando a inserção do evento falha.
- **Testes de integração** (Vitest + Supertest, seguindo o padrão de `tests/orders.test.ts`) para todos os endpoints do módulo `webhooks`: CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ (incluindo caso de acesso negado para role diferente de `ADMIN`).
- **Testes do worker** com cliente HTTP mockado, simulando sucesso, timeout, falha 5xx e falha após as 5 tentativas — validando a progressão exata de `next_attempt_at` e a criação da entrada em DLQ.
- **Testes de contrato de segurança**: verificação de que a assinatura HMAC-SHA256 enviada é validável recalculando com a secret correta, e que a secret anterior ainda é aceita dentro do grace period de 24h.
- **Revisão de segurança manual** pela Sofia antes do deploy em produção, com foco em geração/armazenamento/rotação de secret ([09:46]-[09:47]).
- **Validação de carga leve** do worker sob rajada de mudanças de status simuladas, para observar o comportamento do backlog da outbox antes do lançamento para os três clientes-piloto (Atlas, MaxDistribuição, Nova Cargo).
