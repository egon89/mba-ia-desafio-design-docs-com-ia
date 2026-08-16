# ADR-006: Módulo `webhooks` seguindo a estrutura e os padrões já existentes no código

## Status

Aceito — 2026 (reunião técnica de definição da feature, ver `TRANSCRICAO.md`)

## Contexto

O OMS já tem uma convenção estabelecida de organização de código: cada domínio de negócio vive em `src/modules/<dominio>` com os arquivos `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts` (confirmado nos módulos existentes `auth`, `users`, `customers`, `products`, `orders`). Há também padrões transversais consolidados: hierarquia de erros baseada em `AppError` (`src/shared/errors/app-error.ts`) com subclasses HTTP (`src/shared/errors/http-errors.ts`) e códigos de erro em `SCREAMING_SNAKE_CASE` (ex.: `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`); um middleware de erro central único (`src/middlewares/error.middleware.ts`) que já sabe serializar qualquer `AppError`, `ZodError` e erros conhecidos do Prisma; logger estruturado Pino compartilhado (`src/shared/logger/index.ts`); autenticação JWT e controle de acesso por role via `authenticate`/`requireRole` (`src/middlewares/auth.middleware.ts`); e validação de entrada via Zod (`src/middlewares/validate.middleware.ts`). A pergunta da reunião foi se o novo módulo de webhooks deveria seguir esses padrões ou justificar um desenho próprio ([09:27] Bruno).

## Decisão

O módulo de webhooks segue **exatamente** a convenção existente, com o menor número de exceções possível:

- Novo módulo `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, registrado em `src/routes/index.ts` (`buildApiRouter`) e `src/app.ts` (`buildControllers`), do mesmo jeito que os módulos atuais ([09:27]-[09:28] Bruno/Diego).
- A lógica de processamento do worker fica em um arquivo dentro do próprio módulo (ex.: `src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`), enquanto o *entry point* do processo separado é `src/worker.ts`, paralelo a `src/server.ts` (ver ADR-002) ([09:28] Bruno/Diego).
- Novas classes de erro **estendem `AppError`** (via `BadRequestError`, `NotFoundError`, `ConflictError`, etc., de `src/shared/errors/http-errors.ts`, seguindo o mesmo padrão de `InsufficientStockError`/`InvalidStatusTransitionError`), com **códigos prefixados `WEBHOOK_*`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28]-[09:29] Bruno/Larissa).
- O middleware de erro central (`src/middlewares/error.middleware.ts`) **não precisa ser alterado**: por já tratar qualquer subclasse de `AppError` de forma genérica, absorve os novos erros de webhook automaticamente ([09:29] Bruno: "O middleware de erro centralizado já trata AppError, Zod e Prisma. Vai pegar nossos erros sem precisar mudar nada.").
- O logger é a instância Pino já existente (`src/shared/logger/index.ts`), reaproveitada tanto pelos handlers HTTP do módulo quanto pelo processo do worker — nenhuma dependência de logging nova é introduzida ([09:29] Bruno).
- Autorização reaproveita `requireRole` (`src/middlewares/auth.middleware.ts`) tal como já usado em `user.routes.ts` (`requireRole('ADMIN')`), aplicado ao endpoint de replay de DLQ (ver ADR-003) ([09:36] Larissa/Sofia).
- Validação de entrada usa o mesmo middleware `validate({ body, query, params })` com schemas Zod, incluindo as regras específicas de webhook (URL HTTPS obrigatória, limite de 64KB de payload) ([09:23] Sofia).
- O worker abre sua própria instância de `PrismaClient`, já que `PrismaClient` é por processo, mas aponta para o mesmo `DATABASE_URL` usado pela API (`src/config/database.ts`) ([09:29]-[09:30] Diego/Bruno).

## Alternativas Consideradas

1. **Desenhar uma estrutura própria para o módulo de webhooks** (por exemplo, uma camada de eventos genérica desacoplada do padrão `controller/service/repository/routes/schemas`, ou uma hierarquia de erros paralela específica para o domínio de eventos). Foi implicitamente descartada pelo consenso do time: a proposta de seguir o padrão existente foi apresentada por Bruno e aceita sem contraponto por Diego e Larissa ([09:27]-[09:30]), já que introduzir uma segunda convenção estrutural na mesma base de código aumentaria o custo cognitivo para quem já conhece o padrão dos demais módulos, sem benefício correspondente — o padrão atual já comportava bem os requisitos discutidos (erros tipados, validação, autorização por role, logging estruturado).

## Consequências

**Positivas**
- Zero curva de aprendizado adicional para quem já trabalha nos módulos existentes (`orders`, `products`, etc.) — o código de webhooks "parece" com o resto do projeto.
- Reuso direto e sem adaptação de infraestrutura crítica já testada em produção: error middleware, logger, autenticação/autorização, validação — reduz superfície de código novo e, consequentemente, de bugs novos.
- Rastreabilidade de erros consistente: qualquer erro de webhook aparece nos logs e nas respostas HTTP no mesmo formato que qualquer outro erro do sistema (`{ error: { code, message, details? } }`), sem necessidade de documentação separada de formato de erro.

**Negativas**
- Amarra o módulo de webhooks às limitações do padrão atual (por exemplo, composição manual de dependências em `buildControllers`/`buildApiRouter`, sem um container de DI) — qualquer futura evolução da arquitetura base do projeto arrasta o módulo de webhooks junto.
- O worker, por ser um processo sem requisições HTTP recebidas, não passa pelo `request-logger.middleware.ts` (que depende do ciclo de request/response do Express); logging de contexto de execução do worker (ex.: qual evento está sendo processado) precisa ser construído manualmente reaproveitando apenas a instância `logger`, não o middleware inteiro.
- Duas instâncias de `PrismaClient` ativas simultaneamente (API e worker) sobre o mesmo banco aumentam o número total de conexões abertas no MySQL, exigindo atenção ao dimensionamento do pool de conexões conforme a feature entra em produção.
