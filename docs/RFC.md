# RFC-001 — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| Autor | Larissa (Tech Lead) — documentação consolidada a partir da reunião técnica de definição da feature |
| Status | Em revisão |
| Data de publicação | 2026-08-16 |
| Revisores | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |
| Documentos relacionados | `docs/PRD.md`, `docs/FDD.md`, `docs/adrs/ADR-001` a `ADR-007` |

## TL;DR

Vamos substituir o polling que clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) fazem hoje em `GET /orders` por webhooks outbound: sempre que o status de um pedido mudar, o cliente é notificado via HTTP em menos de 10 segundos. A entrega é assíncrona, desacoplada da transação de negócio pelo padrão Outbox sobre o MySQL já existente, processada por um worker dedicado em polling de 2s, com retry exponencial e DLQ para falhas persistentes, autenticada por HMAC-SHA256 com secret por endpoint, e com garantia at-least-once deduplicável via `X-Event-Id`. O módulo novo (`src/modules/webhooks`) segue estritamente os padrões já estabelecidos na base de código.

## Contexto e problema

Três clientes B2B pediram formalmente notificação em tempo real de mudança de status de pedido; hoje eles fazem polling periódico em `GET /orders`, o que é lento e caro para eles, e a Atlas sinalizou risco de churn se isso não for entregue até o fim do trimestre. "Tempo real", nas palavras do cliente, significa qualquer latência abaixo de 10 segundos — não é um requisito de streaming nem de latência sub-segundo.

O OMS hoje não tem nenhum mecanismo de notificação externa, eventos, filas ou webhooks. A mudança de status de pedido é feita hoje inteiramente dentro de `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), em uma única transação Prisma que atualiza `orders`, insere em `order_status_history` e ajusta estoque. Qualquer solução de notificação precisa se acoplar a esse ponto sem comprometer a confiabilidade dessa transação nem sua performance para outros pedidos.

## Proposta técnica

A proposta tem três blocos, cada um com o detalhamento de implementação especificado no FDD e a decisão registrada em ADR dedicado:

1. **Publicação de eventos via padrão Outbox** ([ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md)). A mesma transação que muda o status do pedido insere um evento em uma nova tabela `webhook_outbox`, sobre o MySQL já em produção — sem infraestrutura nova. Se a transação principal faz commit, o evento existe; se sofre rollback, o evento nunca existiu. Não há inconsistência possível entre estado do pedido e eventos publicados.

2. **Entrega assíncrona por um worker dedicado** ([ADR-002](adrs/ADR-002-worker-dedicado-com-polling.md)). Um processo Node separado da API (`src/worker.ts`), com seu próprio `PrismaClient`, faz polling da outbox a cada 2 segundos, monta e envia a requisição HTTP ao endpoint do cliente. Falhas de entrega seguem uma política de retry com backoff exponencial (5 tentativas, 1m/5m/30m/2h/12h) até serem movidas para uma Dead Letter Queue reprocessável manualmente por um administrador ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)).

3. **Segurança e confiabilidade de entrega**. Cada requisição é assinada com HMAC-SHA256 usando uma secret exclusiva daquele endpoint de webhook, com suporte a rotação e grace period de 24h ([ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)). A garantia de entrega é at-least-once: o cliente pode eventualmente receber o mesmo evento duas vezes e deve deduplicar usando o header `X-Event-Id` ([ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).

O módulo novo é `src/modules/webhooks`, seguindo a mesma convenção `controller/service/repository/routes/schemas` dos módulos existentes, reaproveitando integralmente a hierarquia de erros (`AppError`), o error middleware central, o logger Pino e o middleware `requireRole` já presentes na base de código, sem exigir alteração em nenhum desses componentes compartilhados ([ADR-006](adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md)). Uma limitação conhecida e aceita é o ordering: eventos são garantidamente entregues em ordem apenas dentro de um mesmo pedido, e apenas enquanto houver um único worker rodando ([ADR-007](adrs/ADR-007-ordering-por-order-id-em-single-worker.md)).

Fluxos, contratos HTTP, formato de payload, matriz de erros e estratégias de observabilidade estão detalhados no FDD — este RFC não repete esse nível de detalhe.

## Alternativas consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| **Disparo síncrono da notificação dentro de `OrderService.changeStatus`** | Acoplaria a disponibilidade/latência de um sistema de terceiro à transação crítica de mudança de status; um cliente lento ou fora do ar travaria mudanças de status de outros pedidos, e não há como fazer rollback de uma chamada HTTP já enviada. |
| **Fila dedicada (ex.: Redis Streams) em vez de Outbox sobre o MySQL** | Exigiria subir e operar infraestrutura nova para um time pequeno ("overengineering"), além de introduzir um segundo ponto de falha para garantir atomicidade com a transação SQL principal — o que o padrão outbox resolve nativamente por estar no mesmo banco. |
| **Retry indefinido, ou limitado a 3 tentativas** | Retry indefinido deixaria eventos pendurados para sempre caso o cliente tivesse desaparecido; 3 tentativas se mostrou curto demais frente a um caso real de indisponibilidade planejada de ~2h de um cliente, que esgotaria o retry antes dele voltar. Cinco tentativas com backoff até 12h cobre essa janela sem manter retry perpétuo. |
| **Garantia de entrega exactly-once** | Exigiria coordenação transacional entre OMS e sistema do cliente, aumentando muito a complexidade para um ganho marginal. At-least-once com deduplicação por `X-Event-Id` é o padrão adotado por players de mercado (Stripe, GitHub) e resolve o problema real dos clientes. |
| **Secret única/global para toda a plataforma** | Um vazamento comprometeria a autenticidade de webhooks de todos os clientes simultaneamente. Secret por endpoint isola o impacto de um vazamento a um único cliente/integração. |

## Questões em aberto

1. **Rate limiting de envio ao cliente.** Se um cliente tiver dezenas de pedidos mudando de status em um curto intervalo, hoje nada limita o volume de chamadas simultâneas disparadas contra o endpoint dele. Ficou decidido observar em produção e decidir depois se vira um problema real — não é definido nesta proposta se/como implementar.
2. **Notificação proativa de webhook com falha recorrente.** A ideia de avisar o cliente (ex.: por e-mail) quando o webhook dele falha repetidamente foi levantada mas explicitamente adiada para uma fase futura, condicionada a medir o impacto da primeira versão da feature.
3. **Arquivamento/retenção de eventos entregues na outbox.** Foi mencionado que linhas com status "entregue" deveriam ser arquivadas após ~30 dias, mas nenhuma estratégia de arquivamento foi desenhada — fica em aberto quando e como implementar isso antes que a tabela cresça sem limite.
4. **Escalonamento para múltiplos workers.** A garantia de ordering por `order_id` (ADR-007) depende da premissa de um único worker. Se o volume de eventos exigir mais de um worker no futuro, será necessário resolver particionamento por `order_id` ou locking pessimista — nenhuma das duas abordagens foi aprofundada ou decidida nesta reunião.

## Impacto e riscos

- **Cronograma:** estimativa inicial de três sprints, incluindo a revisão de segurança da Sofia no fim do ciclo. A Atlas Comercial espera a entrega até o fim do trimestre; atraso tem risco comercial explícito (sinalização de churn).
- **Revisão de segurança obrigatória antes do deploy.** Pelo menos dois dias úteis reservados para revisão específica de HMAC e geração/armazenamento de secret — é uma dependência de cronograma, não apenas uma boa prática.
- **Nova carga sobre o MySQL de produção.** A tabela `webhook_outbox` (e a DLQ) adicionam volume de escrita/leitura sobre o banco transacional que hoje serve toda a aplicação; mitigado por índices dedicados, mas o impacto real só se confirma em produção.
- **Dependência operacional de um novo processo em produção.** O worker é um novo artefato de deploy, com seu próprio ciclo de vida, monitoramento e necessidade de garantir que exatamente uma instância esteja rodando (nada nesta proposta impede acidentalmente subir duas).
- **Responsabilidade transferida ao cliente integrador.** Deduplicação (via `X-Event-Id`) e validação de assinatura (HMAC) dependem de implementação correta do lado do cliente; falhas de integração do lado deles podem gerar chamados de suporte.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-padrao-outbox-no-mysql.md)
- [ADR-002 — Worker dedicado, processo separado, polling de 2s](adrs/ADR-002-worker-dedicado-com-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — Garantia at-least-once com X-Event-Id](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)
- [ADR-006 — Módulo webhooks reaproveitando padrões existentes](adrs/ADR-006-modulo-webhooks-reaproveitando-padroes-existentes.md)
- [ADR-007 — Ordering por order_id em single-worker](adrs/ADR-007-ordering-por-order-id-em-single-worker.md)
