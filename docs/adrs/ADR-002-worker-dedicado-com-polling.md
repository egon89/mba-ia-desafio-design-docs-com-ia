# ADR-002: Worker dedicado, em processo separado, com polling de 2 segundos

## Status

Aceito — 2026 (reunião técnica de definição da feature, ver `TRANSCRICAO.md`)

## Contexto

Com o padrão outbox definido (ADR-001), era necessário decidir quem lê a tabela `webhook_outbox` e realiza as entregas HTTP, e com qual mecanismo de ativação. A API HTTP hoje tem um único entry point, `src/server.ts`, que sobe o Express via `buildApp` e não tem nenhum processo em background.

Dois pontos guiaram a decisão: (1) o processo que entrega webhooks não pode compartilhar o ciclo de vida da API — se a API reinicia (deploy, crash, scaling), o processamento de eventos pendentes não pode parar junto ([09:11] Diego: "o worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker."); e (2) o MySQL não oferece um mecanismo nativo de notificação de mudança (ao contrário do `LISTEN`/`NOTIFY` do PostgreSQL), então a leitura de novos eventos não pode ser reativa por natureza ([09:09] Diego).

## Decisão

Criar um novo entry point `src/worker.ts` (paralelo a `src/server.ts`), executável via um novo script (ex.: `npm run worker`), rodando como **processo Node separado da API**, conectado ao mesmo `DATABASE_URL` mas com sua **própria instância de `PrismaClient`** — `PrismaClient` é por processo, então não pode ser compartilhado entre API e worker mesmo apontando para o mesmo banco ([09:29]-[09:30] Diego/Bruno).

O worker opera em **polling a cada 2 segundos**: a cada ciclo, busca os eventos pendentes mais antigos na `webhook_outbox` em lote pequeno, processa (assina, envia, trata resultado) e marca o status ([09:09] Diego). Um intervalo de 2 segundos garante, no pior caso, uma latência de entrega dominada pelo polling muito abaixo do requisito de "tempo real" definido pelos clientes B2B como inferior a 10 segundos ([09:02] Marcos; [09:10] Larissa: "Worker em polling, 2s. A latência mínima vai ser 2 segundos no pior caso. Aceitamos.").

Uma consequência aceita explicitamente é a limitação de ordering: como (nesta fase) roda um único worker, os eventos são processados na ordem de `created_at` da outbox, o que preserva a ordem de eventos de um mesmo pedido, mas não oferece garantia de ordering global entre pedidos diferentes caso o modelo evolua para múltiplos workers no futuro (ver ADR-007).

## Alternativas Consideradas

1. **Trigger de banco de dados para notificar o worker.** Descartada: o MySQL suporta triggers, mas eles só executam SQL dentro do próprio banco — não conseguem notificar um processo externo. Implementar isso exigiria um mecanismo improvisado (escrever em arquivo, chamar um endpoint HTTP a partir do trigger), considerado "esquisito" e desnecessariamente complexo frente ao requisito real de latência ([09:09] Diego).
2. **Worker embutido no mesmo processo da API** (ex.: um `setInterval` dentro de `src/server.ts`). Não foi essa a opção discutida em detalhe, mas foi implicitamente descartada pelo mesmo argumento que motivou o processo separado: acoplar o ciclo de vida do worker ao da API faria qualquer reinício, deploy ou crash da API interromper o processamento de eventos pendentes ([09:11] Diego).

## Consequências

**Positivas**
- Ciclo de vida do worker independente da API: deploys, restarts e scaling da API não impactam o processamento de eventos em andamento.
- Simplicidade operacional: sem infraestrutura de mensageria nova, apenas um processo Node adicional consumindo o mesmo banco.
- Latência previsível e simples de explicar/operar (pior caso = intervalo de polling).

**Negativas**
- Latência mínima de entrega é limitada pelo intervalo de polling (2s), não é evento-reativo; para requisitos de latência sub-segundo no futuro, este desenho precisaria mudar.
- Cada ciclo de polling gera uma consulta ao MySQL mesmo quando não há eventos pendentes — custo de leitura constante, mitigado pelos índices de status/`created_at` definidos no ADR-001, mas presente.
- Escalar para múltiplos workers (necessário para lidar com maior volume) não é imediato: exige resolver particionamento ou locking para não reprocessar o mesmo evento duas vezes e para não quebrar a ordenação por pedido (ver ADR-007) — deixado como problema futuro, não resolvido nesta decisão ([09:13] Diego).
