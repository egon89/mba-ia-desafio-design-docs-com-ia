# ADR-003: Retry com backoff exponencial (5 tentativas) e Dead Letter Queue em tabela separada

## Status

Aceito — 2026 (reunião técnica de definição da feature, ver `TRANSCRICAO.md`)

## Contexto

Endpoints de clientes B2B podem estar temporariamente indisponíveis (manutenção planejada, instabilidade, etc.) — a equipe já tinha um caso concreto de cliente com ~2 horas de indisponibilidade em manutenção planejada ([09:16] Diego). Era necessário decidir: quantas vezes tentar reentregar um evento que falhou, com qual espaçamento entre tentativas, e o que fazer quando as tentativas se esgotam sem sucesso.

## Decisão

Adotar **retry com backoff exponencial, limitado a 5 tentativas**, com a progressão de intervalos **1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas** entre tentativas sucessivas — uma janela total de aproximadamente 15 horas entre a primeira falha e a última tentativa ([09:15]-[09:17] Diego/Larissa). Esgotadas as 5 tentativas, o evento é movido para uma **Dead Letter Queue (DLQ) em tabela separada**, `webhook_dead_letter`, contendo o payload, o motivo da falha e o timestamp ([09:18] Diego).

A reentrega de um evento em DLQ não é automática: existe um **endpoint administrativo** `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento como pendente na `webhook_outbox` ([09:18] Diego, [09:35] Diego). Esse endpoint exige role `ADMIN` (ver ADR-004/middleware `requireRole`) e deve registrar log de auditoria de quem executou o replay ([09:36] Sofia).

O timeout de cada tentativa de entrega HTTP é de 10 segundos: se o cliente não responder nesse prazo, a tentativa é tratada como falha e segue para o próximo passo do backoff ([09:42] Diego/Sofia).

## Alternativas Consideradas

1. **Retry indefinido com backoff.** Descartada: deixaria eventos pendurados indefinidamente caso o cliente de destino tivesse desaparecido de vez, sem um ponto claro de "falha permanente" para intervenção humana ([09:15] Diego).
2. **3 tentativas, mais agressivo.** Descartada: no caso real de indisponibilidade planejada de ~2 horas de um cliente, 3 tentativas concentradas em ~30 minutos esgotariam o retry antes do cliente voltar, gerando falso negativo de entrega ([09:16] Diego: "3 é pouco. [...] Já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada.").
3. **Marcar falha permanente diretamente na tabela de outbox** (flag `failed`), sem tabela de DLQ separada. Descartada: mistura o estado operacional "linha em processamento normal" com "linha que precisa de intervenção humana", dificultando a leitura da outbox principal; uma tabela dedicada mantém a outbox limpa e serve como evidência para debug e reprocessamento ([09:18] Diego: "Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento.").

## Consequências

**Positivas**
- Cobertura de janelas realistas de indisponibilidade de cliente (até ~15h) sem intervenção manual.
- Separação clara entre "evento em fluxo normal" (outbox) e "evento que precisa de decisão humana" (DLQ), com histórico de motivo de falha preservado para investigação.
- Reprocessamento controlado e auditável (endpoint admin, role obrigatória, log de quem agiu), evitando reenvio acidental em massa.

**Negativas**
- Um evento com falha real de 5 tentativas fica até ~15 horas sem chegar ao cliente antes de cair em DLQ — aceitável para o caso de uso atual ([09:17] Marcos: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável."), mas é uma janela longa se o requisito de negócio mudar.
- DLQ exige processo operacional: sem alguém monitorando e agindo sobre entradas na DLQ, eventos falhos ficam parados indefinidamente (não há reprocessamento automático nem alerta proativo nesta fase — ver RFC, "Questões em aberto").
- Mais uma tabela para gerenciar (schema, índices, eventual retenção/expurgo), fora do escopo definido para arquivamento de outbox já registrado no ADR-001.
