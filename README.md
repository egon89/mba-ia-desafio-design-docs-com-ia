# Sistema de Webhooks de Notificação de Pedidos — Design Docs

Este repositório contém o pacote de design docs produzido para a feature de **Sistema de Webhooks de Notificação de Pedidos** do Order Management System (OMS), a partir da transcrição de uma reunião técnica (`TRANSCRICAO.md`) e da exploração do código existente (`src/`). O enunciado original do desafio que motivou este trabalho (contexto do bootcamp de MBA em Engenharia de Software com IA) pode ser consultado no histórico do repositório base; este README documenta o **processo** usado para produzir a entrega.

## Sobre o desafio

O desafio parte de uma decisão técnica já fechada em reunião — presente apenas como transcrição literal, `[hh:mm] Nome: fala` — e pede a transformação dessa conversa (mais o código-fonte do OMS) em um pacote de documentação de engenharia pronto para orientar a implementação: PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade. A IA é a ferramenta principal de produção do conteúdo, mas a responsabilidade de decidir o que entra em cada documento, filtrar o que foi descartado ou adiado na reunião, e garantir que nada seja inventado é de quem conduz o processo.

A restrição mais importante do desafio é a rastreabilidade: toda informação registrada nos documentos precisa ser rastreável à transcrição ou ao código-fonte. Isso muda o critério de qualidade do trabalho — não é "o documento parece bom?", é "toda frase deste documento eu consigo apontar de onde veio?".

## Ferramentas de IA utilizadas

- **Claude Code** (Anthropic, modelo Sonnet 5) — ferramenta principal e única usada nesta entrega. Atuou com acesso direto ao repositório (leitura de arquivos de código e da transcrição, escrita dos documentos de saída) em um processo conversacional guiado por diretivas específicas para cada etapa do pacote (exploração de código, extração estruturada da transcrição, geração de ADRs, RFC, FDD, PRD e Tracker), com revisão humana entre cada etapa.

## Workflow adotado

O processo seguiu a ordem sugerida pelo desafio, com uma adaptação: o **Tracker foi atualizado em paralelo a cada documento**, não deixado inteiramente para o fim, para reduzir o risco de perder a origem de algum item ao longo do caminho.

1. **Exploração de código.** Antes de tocar na transcrição, mapeei a base de código do OMS (módulos, máquina de estados de pedido, transação de `changeStatus`, hierarquia de erros, error middleware, logger Pino, autenticação/`requireRole`) e registrei os achados em um arquivo de contexto interno (não faz parte do pacote entregável — fica fora de `docs/`).
2. **Extração estruturada da transcrição.** Em vez de pedir um resumo genérico, pedi três listas separadas e citáveis: decisões fechadas, requisitos funcionais explícitos, e itens descartados/adiados — cada item com timestamp e falante. Esse arquivo intermediário também ficou fora de `docs/`, mas foi a matéria-prima direta de todos os documentos seguintes.
3. **ADRs primeiro.** Sete decisões arquiteturais, cobrindo as seis decisões principais listadas no desafio mais a limitação de ordering discutida na reunião.
4. **RFC**, referenciando os ADRs já escritos, com alternativas descartadas e questões em aberto extraídas do passo 2. **Tracker atualizado em paralelo** com os itens dos ADRs e do RFC.
5. **FDD**, o documento mais longo e técnico: fluxos, contratos HTTP, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção obrigatória de integração com o código existente, citando caminhos de arquivo reais. **Tracker atualizado** com os novos itens.
6. **PRD**, escrito por último entre os documentos grandes — na prática, uma consolidação em linguagem de produto do que já estava decidido em RFC/FDD/ADRs. **Tracker atualizado** mais uma vez.
7. **README do processo** (este arquivo), escrito ao final.

## Prompts customizados

Dois exemplos de diretivas usadas para conduzir etapas específicas do processo (adaptadas do fluxo recomendado no curso, com o contexto específico desta feature):

**Extração estruturada da transcrição** (evita pedir um resumo genérico, que tende a misturar decisão com especulação):

```
Leia TRANSCRICAO.md e produza três listas separadas, cada item citando
timestamp e falante no formato [hh:mm] Nome:

1. Decisões fechadas pela equipe (candidatas a ADR).
2. Requisitos funcionais explicitamente pedidos (candidatos a PRD).
3. Itens descartados ou adiados durante a reunião (candidatos a
   "Fora de escopo" no PRD e "Questões em aberto" no RFC).

Não inclua nada que não tenha uma citação direta. Se um item foi
mencionado mas não decidido, ele vai na lista 3, não na lista 1.
```

**Geração de ADRs com cobertura obrigatória e referência ao código** (evita ADRs genéricos desconectados da base real):

```
Produza entre 5 e 8 ADRs em docs/adrs/, formato
ADR-NNN-titulo-em-kebab-case.md, cobrindo pelo menos 5 destas 6
decisões: outbox no MySQL, retry com backoff e DLQ, HMAC-SHA256 com
secret por endpoint, garantia at-least-once com X-Event-Id, worker em
processo separado com polling, reuso de padrões existentes do projeto.

Cada ADR: Status, Contexto, Decisão, Alternativas Consideradas (pelo
menos 1 alternativa real discutida), Consequências (positivas e
negativas, com trade-off explícito). Toda afirmação sobre a reunião
precisa citar [hh:mm] Nome. Pelo menos 1 ADR precisa referenciar
arquivos ou módulos reais do código (ex: order.service.ts, AppError,
requireRole) — verifique que esses arquivos existem antes de citá-los.
```

## Iterações e ajustes

O processo não foi linear na primeira passada; os ajustes mais relevantes:

1. **Correção de citação incorreta no PRD.** Ao escrever a tabela de riscos do PRD, citei `[09:18] Marcos` como origem da ameaça de churn da Atlas Comercial. Ao revisar a transcrição para popular o Tracker, percebi que essa fala está em `[09:00] Marcos`, não `[09:18]` (que corresponde a outro trecho da reunião, sobre o endpoint de replay de DLQ). Foi corrigido tanto no `PRD.md` quanto na citação usada no Tracker antes de prosseguir — exatamente o tipo de erro que o Tracker existe para capturar.
2. **Disciplina para não forçar rastreabilidade onde não existe.** Na seção de riscos do FDD, dois riscos (crescimento de backlog na outbox, cliente implementando mal a deduplicação) são análise técnica própria, não uma afirmação literal da transcrição ou do código. Optei por **não** criar linhas de Tracker para esses riscos específicos em vez de inventar uma citação aproximada — mantendo o Tracker como evidência apenas do que é de fato rastreável, e não como checklist decorativo.
3. **Documentos de apoio fora de `docs/`.** O arquivo de contexto de código e a extração estruturada da transcrição (passos 1 e 2 do workflow) foram deliberadamente mantidos fora da pasta `docs/`, que o desafio reserva para os cinco documentos formais do pacote — evita que material de rascunho seja confundido com entregável.
4. **RFC mantido conciso de propósito.** A primeira versão mental do RFC tendia a puxar detalhe de implementação (nomes de tabela, campos, sequência exata de queries) para dentro da proposta técnica. Esse nível de detalhe foi deslocado para o FDD, mantendo o RFC como documento de decisão de arquitetura (2-4 páginas), não de implementação — a mesma distinção de altura que o enunciado do desafio pede explicitamente entre os dois documentos.

## Como navegar a entrega

Ordem sugerida de leitura, da mais alta para a mais baixa altura:

1. [`docs/PRD.md`](./docs/PRD.md) — por quê e o quê: problema, público, objetivos, escopo.
2. [`docs/RFC.md`](./docs/RFC.md) — proposta técnica, alternativas descartadas, questões em aberto.
3. [`docs/adrs/`](./docs/adrs/) — as sete decisões arquiteturais individuais (ADR-001 a ADR-007), cada uma com contexto, alternativas e consequências.
4. [`docs/FDD.md`](./docs/FDD.md) — como construir: fluxos, contratos HTTP, matriz de erros, integração com o código existente.
5. [`docs/TRACKER.md`](./docs/TRACKER.md) — para verificar, item a item, de onde veio qualquer afirmação dos documentos acima.

---

## Critérios de Aceite

A entrega é avaliada contra os critérios abaixo. Todos são obrigatórios.

### PRD (`docs/PRD.md`)

- ☑ Arquivo existe e está em Markdown
- ☑ Contém todas as seções obrigatórias listadas no requisito 1
- ☑ Identifica no mínimo 8 requisitos funcionais discutidos na reunião
- ☑ Inclui pelo menos 1 objetivo com métrica e meta quantitativa
- ☑ Seção "Fora de escopo" lista pelo menos 2 itens explicitamente descartados ou adiados na reunião
- ☑ Seção "Riscos" inclui pelo menos 2 riscos com probabilidade, impacto e mitigação

### RFC (`docs/RFC.md`)

- ☑ Arquivo existe e está em Markdown
- ☑ Contém todas as seções obrigatórias listadas no requisito 2
- ☑ Seção "Alternativas consideradas" lista pelo menos 2 alternativas descartadas na reunião, cada uma com o trade-off que motivou o descarte
- ☑ Seção "Questões em aberto" lista pelo menos 2 pontos adiados ou não decididos na reunião
- ☑ Referencia, com link, pelo menos 2 ADRs do pacote

### FDD (`docs/FDD.md`)

- ☑ Arquivo existe e está em Markdown
- ☑ Contém todas as seções obrigatórias listadas no requisito 3
- ☑ Seção "Contratos públicos" inclui pelo menos 4 endpoints HTTP com payload de exemplo (request e response) e status codes
- ☑ Matriz de erros usa códigos com prefixo `WEBHOOK_`
- ☑ Seção "Integração com o sistema existente" referencia pelo menos 4 caminhos de arquivo reais do código base
- ☑ Seção "Observabilidade" cita métricas, logs e tracing

### ADRs (`docs/adrs/ADR-NNN-*.md`)

- ☑ Pasta `docs/adrs/` contém entre 5 e 8 arquivos no formato `ADR-NNN-titulo-em-kebab-case.md`
- ☑ Cada ADR contém as seções Status, Contexto, Decisão, Alternativas Consideradas, Consequências
- ☑ O conjunto cobre pelo menos 5 das 6 decisões principais listadas no requisito 4
- ☑ Pelo menos 1 ADR referencia explicitamente arquivos, módulos ou classes do código base

### Tracker (`docs/TRACKER.md`)

- ☑ Arquivo existe e segue o formato de tabela definido no requisito 5
- ☑ Pelo menos 80% dos itens identificáveis dos documentos têm linha correspondente
- ☑ Pelo menos 70% das linhas têm Fonte = `TRANSCRICAO` com timestamp válido no formato `[hh:mm] Nome`
- ☑ Pelo menos 5 linhas têm Fonte = `CODIGO` com caminho de arquivo real

### README (`README.md`)

- ☑ Contém todas as seções obrigatórias listadas no requisito 6
- ☑ Lista pelo menos 1 ferramenta de IA utilizada
- ☑ Mostra pelo menos 2 prompts customizados em blocos de código
- ☑ Descreve pelo menos 2 iterações ou ajustes concretos feitos durante a produção

### Consistência geral

- ☑ Nenhum requisito, decisão ou restrição registrada nos documentos contradiz a transcrição ou o código
- ☑ Nenhum arquivo de código mencionado nos documentos é inexistente no repositório

---

## Revisão Final

Checklist acima verificado item a item de forma programática (não apenas por leitura), na entrega já pronta — não é uma autoavaliação genérica, é o resultado de comandos de verificação rodados contra os arquivos reais.

**Como foi verificado:**
- Existência e contagem de arquivos: `ls`/`wc` em `docs/`, `docs/adrs/`.
- Contagens de conteúdo (requisitos funcionais, alternativas, questões em aberto, endpoints, códigos de erro, seções de ADR): `grep -c` sobre os próprios documentos.
- Rastreabilidade: todo caminho `src/...` citado em `docs/*.md`, `docs/adrs/*.md` e neste README foi extraído por regex e checado contra o filesystem do repositório (`[ -f "$caminho" ]`).
- Tracker: recontagem de linhas totais, linhas com Fonte `TRANSCRICAO` (e validação de que 100% delas seguem o formato `[hh:mm] Nome`) e linhas com Fonte `CODIGO`, para confirmar os percentuais mínimos exigidos.

**Resultado da verificação:**

| Item | Resultado |
| --- | --- |
| PRD — requisitos funcionais | 12 (mínimo 8) |
| PRD — objetivo quantitativo | P95 < 10s; prazo de 3 sprints |
| PRD — fora de escopo | 5 itens (mínimo 2) |
| PRD — riscos com probabilidade/impacto/mitigação | 4 (mínimo 2) |
| RFC — alternativas descartadas | 5 (mínimo 2) |
| RFC — questões em aberto | 4 (mínimo 2) |
| RFC — ADRs referenciados | 7 (mínimo 2) |
| FDD — endpoints com payload de exemplo | 7 (mínimo 4) |
| FDD — códigos `WEBHOOK_*` | 9 |
| FDD — caminhos reais na seção de integração | 13 (mínimo 4), todos confirmados no filesystem |
| ADRs — arquivos no padrão exigido | 7 (entre 5 e 8) |
| ADRs — seções obrigatórias completas | 7/7 arquivos, verificado individualmente |
| ADRs — cobertura das decisões principais | 6/6 (mínimo 5) |
| Tracker — linhas totais | 118, distribuídas entre os 5 documentos (38 ADRs, 15 RFC, 31 FDD, 33 PRD, aproximado) |
| Tracker — % Fonte TRANSCRICAO | 80,5% (mínimo 70%), 100% com timestamp `[hh:mm]` válido |
| Tracker — linhas Fonte CODIGO | 22 (mínimo 5), todos os caminhos existentes |
| README — prompts customizados | 2 blocos de código |
| README — iterações descritas | 4 (mínimo 2) |

**Nota sobre consistência de arquivos:** os únicos caminhos citados nos documentos que **não** existem hoje no repositório são `src/worker.ts` e `src/modules/webhooks/*` — e isso é esperado: são os artefatos **novos** que a própria feature propõe criar, sempre apresentados nos documentos como "novo entry point" ou "novo módulo", nunca como código já existente. Todos os demais caminhos citados (`src/modules/orders/order.service.ts`, `src/shared/errors/*`, `src/middlewares/*`, `src/shared/logger/index.ts`, `src/config/database.ts`, `src/routes/index.ts`, `src/app.ts`, `src/server.ts`, `package.json`, entre outros) foram confirmados como reais no código-base.

Durante essa revisão, foi identificado e corrigido um erro de citação no `docs/PRD.md` (timestamp `[09:18]` trocado por `[09:00]` na origem da ameaça de churn da Atlas Comercial) — já registrado na seção "Iterações e ajustes" acima.
