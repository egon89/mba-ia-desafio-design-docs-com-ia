# ADR-007: Ordering de eventos garantida apenas por `order_id`, sob premissa de single-worker

## Status

Aceito — 2026 (reunião técnica de definição da feature, ver `TRANSCRICAO.md`)

## Contexto

Um mesmo pedido pode transicionar por vários status em sequência rápida (ex.: `PAID` → `PROCESSING` → `SHIPPED`). Larissa levantou a pergunta de se o cliente recebe esses eventos necessariamente na ordem em que ocorreram ([09:12] Larissa). A resposta depende diretamente do desenho do worker (ADR-002): com um único worker processando a outbox em ordem de `created_at`, a ordem de entrega para um mesmo pedido é preservada; mas isso deixa de valer automaticamente se o sistema evoluir para múltiplos workers processando em paralelo ([09:12] Diego).

## Decisão

Assumir e documentar como **garantia explícita e limitada**: a ordem de entrega dos eventos é preservada **apenas por `order_id`**, e **apenas enquanto houver um único worker** em operação ([09:12]-[09:13] Diego/Larissa). Não há, nesta fase, garantia de ordering global entre eventos de pedidos diferentes (o que também não é um requisito real dos clientes B2B — eles só precisam saber quando *cada pedido deles* mudou, não a ordem relativa entre pedidos distintos) ([09:14] Marcos). Essa limitação é registrada como conhecida, não como algo a resolver nesta entrega ([09:13] Larissa: "Documentamos como limitação conhecida. Não é garantia de ordering global, só por order_id e enquanto for single-worker.").

## Alternativas Consideradas

1. **Particionamento de eventos por `order_id`** (cada partição processada por um worker dedicado, preservando ordem por pedido mesmo com múltiplos workers). Diego apontou como caminho possível para quando o sistema precisar escalar horizontalmente, mas explicitamente **não decidido agora** — fica como direção futura, não como parte desta feature ([09:13] Diego: "Aí dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro, não agora.").
2. **Lock pessimista** sobre eventos do mesmo pedido para permitir múltiplos workers sem quebrar ordering. Mencionada na mesma resposta de Diego como alternativa ao particionamento, também deixada para uma fase futura, sem aprofundamento de trade-offs nesta reunião.

## Consequências

**Positivas**
- Escopo da entrega atual não precisa resolver o problema, mais complexo, de ordering distribuída — reduz complexidade de implementação e de testes para a primeira versão da feature.
- A garantia oferecida (ordem preservada por `order_id`) já atende ao requisito real levantado pelos clientes B2B, evitando trabalho não solicitado ([09:14] Marcos).

**Negativas**
- É uma limitação arquitetural que se torna um bloqueio direto assim que o volume de eventos justificar múltiplos workers: escalar horizontalmente o worker (ADR-002) sem antes resolver particionamento por `order_id` ou locking quebra silenciosamente a garantia de ordering hoje documentada, correndo o risco de ser esquecida se não for revisitada explicitamente antes de qualquer mudança de escala.
- A garantia depende de um pressuposto operacional (exatamente um worker rodando) que não é reforçado por nenhum mecanismo técnico nesta decisão — não há, por exemplo, lock de execução único (singleton) impedindo alguém de subir acidentalmente uma segunda instância do worker.
