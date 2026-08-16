# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

## Status

Aceito — 2026 (reunião técnica de definição da feature, ver `TRANSCRICAO.md`)

## Contexto

O Order Management System precisa notificar clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) sempre que o status de um pedido muda, substituindo o polling atual que eles fazem em `GET /orders` ([09:00] Marcos). A mudança de status já roda dentro de uma transação Prisma única em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que atualiza `orders`, insere em `order_status_history` e ajusta `stockQuantity` de produtos.

Disparar a notificação de forma síncrona dentro dessa transação foi descartado logo no início da reunião: um HTTP call para um endpoint de cliente lento ou fora do ar travaria a mudança de status de outros pedidos, e não há como fazer rollback de uma chamada HTTP já enviada ([09:04] Bruno: "a transação de mudança de status hoje já é pesada [...] Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos"; [09:04] Bruno: "se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá.").

Era preciso desacoplar "registrar que um evento aconteceu" de "entregar esse evento pela rede", sem introduzir inconsistência entre o estado do pedido e os eventos publicados, e sem exigir infraestrutura nova além do MySQL já em produção.

## Decisão

Adotar o **padrão Outbox** sobre o MySQL existente ([09:06] Diego). Ao mudar o status de um pedido, a mesma transação SQL que atualiza `orders` e `order_status_history` também insere uma linha em uma nova tabela `webhook_outbox`, contendo o evento já pronto para envio. Um worker separado (ver ADR-002) lê essa tabela e realiza as chamadas HTTP de forma assíncrona.

Isso garante atomicidade "grátis": se a transação principal faz commit, o evento foi registrado; se sofre rollback, o evento nunca existiu ([09:06] Diego: "Garante que se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível."). A tabela é indexada por status de processamento (pendente/processando/falhou/entregue) e `created_at`, para que o worker leia lotes pequenos de pendentes de forma eficiente ([09:08] Diego). Linhas com status "entregue" são candidatas a arquivamento após ~30 dias, mas esse arquivamento fica fora do escopo desta feature ([09:08] Diego).

A integração concreta com `OrderService.changeStatus` é detalhada no FDD e depende de uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` (Prisma `TransactionClient`) já aberto pela transação de `changeStatus`, em vez de injetar um repository de webhooks inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).

## Alternativas Consideradas

1. **Disparo síncrono dentro do `changeStatus`.** Descartada: acopla a latência/disponibilidade de um sistema de terceiros à transação crítica de negócio (mudança de status de pedido), sem possibilidade de rollback de um HTTP call já efetuado ([09:04] Bruno).
2. **Fila dedicada (ex.: Redis Streams).** Cogitada por Larissa como alternativa ao outbox ([09:07] Larissa), mas descartada por exigir subir e operar infraestrutura nova para um time pequeno — "overengineering" para o problema em questão ([09:07] Diego: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve."). Além disso, uma fila externa introduziria um segundo ponto de falha para garantir atomicidade com a transação SQL principal, que o padrão outbox resolve nativamente por estar no mesmo banco.

## Consequências

**Positivas**
- Consistência forte entre estado do pedido e eventos publicados, sem transação distribuída nem two-phase commit.
- Nenhuma infraestrutura nova: reaproveita o MySQL e o `PrismaClient` já usados pelo resto da aplicação.
- Falha de rede ou indisponibilidade do cliente de destino nunca compromete a transação de negócio.

**Negativas**
- Introduz um novo componente de leitura (worker) que precisa de sua própria estratégia de retirada de eventos, retry e limpeza — complexidade adicional é movida para fora da transação, mas não desaparece (ver ADR-002 e ADR-003).
- Acoplamento de throughput ao MySQL: em volume muito alto de mudanças de status, a tabela de outbox se torna mais uma fonte de carga de escrita/leitura sobre o banco transacional principal; mitigado pelos índices em status e `created_at` ([09:08] Diego), mas não eliminado.
- Exige disciplina de schema (arquivamento de linhas entregues) para a tabela não crescer indefinidamente — item explicitamente deixado fora do escopo desta feature ([09:08] Diego), portanto uma dívida técnica conhecida desde já.
