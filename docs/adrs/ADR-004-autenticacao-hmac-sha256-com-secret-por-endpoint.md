# ADR-004: Autenticação de entregas via HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito — 2026 (reunião técnica de definição da feature, ver `TRANSCRICAO.md`)

## Contexto

Os eventos de webhook carregam dados de pedidos de clientes e saem da infraestrutura do OMS em direção a sistemas de terceiros pela internet aberta. O cliente que recebe a chamada precisa de uma forma de verificar que a requisição realmente veio do OMS e que o payload não foi adulterado no caminho ([09:19] Sofia). Também era necessário decidir o escopo da(s) credencial(is) usada(s) para essa verificação — uma secret única para toda a plataforma versus uma por cliente/endpoint — considerando o histórico de que já houve vazamento de secret em log de aplicação de um cliente ([09:22] Diego).

## Decisão

Assinar o corpo de cada requisição de entrega com **HMAC-SHA256**, enviando a assinatura no header `X-Signature`, para que o cliente valide autenticidade e integridade do payload do lado dele ([09:20] Sofia: "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso.").

Cada **endpoint de webhook cadastrado tem sua própria secret**, gerada pelo sistema no momento do cadastro (não existe secret global da plataforma) — a tabela de configuração de webhook armazena `url`, `secret`, `customer_id` e estado ativo ([09:21] Sofia/Bruno). A secret é **rotacionável via API**: ao rotacionar, a secret antiga permanece válida em paralelo por um **grace period de 24 horas**, dando tempo do cliente migrar seus sistemas antes que a secret anterior deixe de funcionar ([09:21]-[09:22] Sofia).

Complementarmente (registrado aqui por serem requisitos de segurança da mesma decisão, embora de nível de implementação): a URL cadastrada deve ser obrigatoriamente HTTPS, sendo recusada na validação de schema caso seja HTTP ([09:23] Sofia); e o payload de cada evento tem um limite de 64KB, sendo rejeitado (não truncado) caso o exceda ([09:23]-[09:24] Sofia/Diego).

## Alternativas Consideradas

1. **Secret única/global para a plataforma inteira.** Descartada explicitamente por Sofia: um vazamento de secret comprometeria a autenticidade de webhooks para todos os clientes simultaneamente, em vez de isolar o impacto a um único endpoint comprometido ([09:21] Sofia: "Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo.").
2. **Rotação de secret sem grace period** (troca imediata, a antiga para de funcionar na hora). Não foi essa a proposta final: Sofia propôs diretamente a rotação com paralelismo de 24h, motivada por um incidente real de vazamento em log de aplicação de cliente que exigiria troca coordenada ([09:22] Diego). Uma troca instantânea forçaria uma janela de indisponibilidade de entrega para o cliente até ele atualizar a secret do lado dele, o que o grace period evita.

## Consequências

**Positivas**
- Blast radius de um vazamento de secret limitado a um único endpoint/cliente, não à base inteira.
- Cliente tem uma forma padrão de mercado (HMAC-SHA256, mesmo padrão usado por Stripe e GitHub) para validar autenticidade, reduzindo fricção de integração.
- Rotação com grace period permite reação a incidentes de segurança (secret vazada) sem downtime forçado de entrega.

**Negativas**
- Manter duas secrets válidas simultaneamente durante 24h após rotação aumenta a superfície de ataque temporariamente (a secret antiga, potencialmente comprometida, continua aceita por até 24h).
- Geração, armazenamento seguro e rotação de secret por endpoint adiciona complexidade operacional e de schema (nova tabela de configuração com campo sensível a proteger) em comparação com uma única secret compartilhada.
- Responsabilidade de validar a assinatura e implementar rotação do lado do cliente é delegada a ele — exige documentação clara no portal do desenvolvedor ([09:26] Marcos) e pode gerar suporte adicional para clientes com implementações incorretas.
