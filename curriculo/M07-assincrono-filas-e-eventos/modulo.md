---
id: M07
titulo: "Assíncrono: filas e eventos"
trilho: nucleo
depende-de: [M06]
checkpoint-de-entrada: M06
status: esqueleto
---

# M07 · Assíncrono: filas e eventos

**Pergunta do módulo:** o que tiro do caminho da requisição, e como garanto que nada se perde nem se duplica?

## Objetivo do módulo

Ao concluir, você decide o que processar de forma assíncrona, escolhe entre fila de tarefas e log de eventos, implementa consumidores idempotentes com retentativa e fila de mensagens mortas, e agrega eventos pelo tempo em que aconteceram.

## Competências

| ID | Competência |
|---|---|
| M07.C1 | Decidir o que sai do caminho síncrono e escolher entre fila de tarefas e log de eventos. |
| M07.C2 | Garantir processamento sem perda nem duplicação visível. |
| M07.C3 | Agregar fluxos de eventos com janelas e tempo do evento. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M07.U1 | Síncrono ou assíncrono | Decidir o que tirar do caminho da requisição e explicar back pressure. | C1 | L07 |
| M07.U2 | Fila de tarefas e log de eventos | Comparar RabbitMQ e Kafka em ordenação, retenção e reprocessamento. | C1 | L07 |
| M07.U3 | Garantias de entrega e idempotência | Explicar "no máximo uma vez", "pelo menos uma vez" e "exatamente uma vez" como efeito, e tornar um consumidor idempotente. | C2 | L07 |
| M07.U4 | Falhas no consumo | Configurar retentativa com espera crescente, fila de mensagens mortas e tratamento de mensagem inválida. | C2 | L06, L07 |
| M07.U5 | Lote, fluxo e janelas | Agregar cliques por janela de tempo do evento e tratar eventos atrasados. | C3 | L08 |
| M07.U6 | Tarefas longas | Aceitar um trabalho demorado, responder na hora e entregar o resultado depois. | C1 | L06 |

## No Projeto prático

- **Parte do checkpoint:** `M06`.
- **Lab:** analytics por fila (RabbitMQ): o redirecionamento publica o clique, um worker idempotente agrega por tempo do evento e as falhas vão para a fila de mensagens mortas.
- **Checkpoint de saída:** `M07`.

## Módulo concluído

1. **Artefato:** pipeline versionado (produtor, fila, worker e fila de mensagens mortas).
2. **Teste de falha:** derrubar o worker no meio do processamento e provar que nenhum clique se perde nem se duplica.
3. **Alternativas:** RabbitMQ × Kafka (ou gravação síncrona × fila), com os limites de cada um.

## Cenário de transferência

Ad Click Aggregator: o analytics do encurtador em escala muito maior.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Key Technologies › Kafka, Temporal; Flink (pago); Patterns › Managing Long-Running Tasks e Multi-step Processes (pagos: usar o resumo em In a Hurry › Patterns); Question Breakdowns › Ad Click Aggregator, LeetCode; In the Wild › Slack Job Queue. Pagos, só como título: Notification System, News Aggregator.
- **Texto de base e pistas (Primer, índice):** Asynchronism (message queues, task queues, back pressure); soluções `sales_rank` e `mint`.
- **Cenários e números (SDI1, secundária):** caps. 9–10, via [sistema-de-notificacoes](../../knowledge/06-estudos-de-caso/sistema-de-notificacoes.md) e [web-crawler](../../knowledge/06-estudos-de-caso/web-crawler.md).
- **Lente (DDIA, não conta como prova):** caps. 10–12, via [processamento-em-lote](../../knowledge/04-dados-derivados/processamento-em-lote.md), [processamento-de-fluxo](../../knowledge/04-dados-derivados/processamento-de-fluxo.md) e [integracao-de-dados-e-correcao](../../knowledge/04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Referências primárias da lista:** Fowler, "What do you mean by 'Event-Driven'?" (confirmar).
- **Candidatas a Referência primária (o aprendiz decide):** Akidau et al., "The Dataflow Model" (VLDB 2015); Akidau, "Streaming 101" e "Streaming 102".
- **Documentação oficial:** RabbitMQ (confiabilidade e confirmações); Kafka.
