# ADR-005: Garantia de entrega at-least-once com deduplicação via X-Event-Id

## Status

Aceito — 2026 (reunião técnica de definição da feature, ver `TRANSCRICAO.md`)

## Contexto

Combinado o padrão outbox (ADR-001) com retry e reentrega manual via DLQ (ADR-003), é possível — e esperado em cenários de falha de rede, timeout do lado do cliente após o processamento, ou replay manual — que o mesmo evento lógico seja entregue mais de uma vez ao cliente. Era necessário decidir o nível de garantia de entrega a oferecer (at-least-once vs. exactly-once) e, dado esse nível, como o cliente consegue identificar duplicatas ([09:24] Diego).

## Decisão

Adotar garantia de entrega **at-least-once**: o sistema garante que todo evento gerado será entregue ao menos uma vez, mas o cliente pode eventualmente receber o mesmo evento mais de uma vez ([09:24] Diego). Para viabilizar deduplicação do lado do cliente, cada evento carrega um **`event_id` (UUID) único, gerado no momento em que o evento entra na outbox**, enviado no header **`X-Event-Id`** em toda tentativa de entrega daquele evento (inclusive reentregas) ([09:25] Diego). O cliente é responsável por usar esse identificador para deduplicar do lado dele.

## Alternativas Consideradas

1. **Garantia exactly-once.** Descartada: exigiria coordenação transacional entre o OMS e o sistema do cliente (ex.: acknowledgment transacional, ou um protocolo de handshake com deduplicação centralizada no servidor), aumentando substancialmente a complexidade de implementação e operação para um ganho marginal, já que o padrão de mercado resolve o mesmo problema de forma mais simples ([09:25] Diego: "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos."; ver também Stripe e GitHub como referências de mercado que adotam a mesma abordagem).

## Consequências

**Positivas**
- Modelo simples de implementar e operar: nenhuma coordenação distribuída é necessária entre o worker e os endpoints dos clientes.
- Alinhado a um padrão de mercado amplamente compreendido (mesma abordagem adotada por Stripe e GitHub), o que reduz a curva de aprendizado do lado do cliente integrador.
- Combinado ao retry do ADR-003, favorece a garantia mais importante para o caso de uso ("o cliente sempre saberá que o pedido mudou"), mesmo em cenários de falha parcial.

**Negativas**
- A responsabilidade de deduplicação é transferida para o cliente ([09:25] Sofia: "Isso joga responsabilidade pro cliente."). Integrações mal implementadas do lado do cliente podem processar o mesmo evento (ex.: mesma mudança de status) mais de uma vez, exigindo comunicação clara no portal do desenvolvedor sobre a obrigatoriedade de tratar `X-Event-Id` ([09:26] Marcos).
- Não há, nesta fase, nenhuma deduplicação do lado do servidor (o OMS não impede duas tentativas de entrega do mesmo evento gerarem duas requisições HTTP); o único mecanismo de correção é o identificador exposto ao cliente.
