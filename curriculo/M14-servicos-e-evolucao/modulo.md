---
id: M14
titulo: "Serviços, fluxos longos e evolução"
trilho: aprofundamento
depende-de: [M10]
checkpoint-de-entrada: M10
status: esqueleto
---

# M14 · Serviços, fluxos longos e evolução

**Pergunta do módulo:** quando dividir em serviços, e como coordenar uma operação que atravessa vários deles?

## Objetivo do módulo

Ao concluir, você decide com evidência entre um monolito modular e serviços separados, traça fronteiras, coordena operações entre serviços sem transação distribuída e evolui contratos sem quebrar quem os consome.

## Competências

| ID | Competência |
|---|---|
| M14.C1 | Decidir entre monolito e serviços com critérios explícitos. |
| M14.C2 | Coordenar operações que atravessam serviços sem transação distribuída. |
| M14.C3 | Evoluir contratos entre serviços mantendo compatibilidade. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M14.U1 | Monolito primeiro? | Avaliar os pré-requisitos de microserviços e o custo operacional antes de dividir. | C1 | L06 |
| M14.U2 | Fronteiras de serviço | Traçar fronteiras por capacidade de negócio e decidir de qual serviço é cada dado. | C1 | L04 |
| M14.U3 | Os limites da transação distribuída | Explicar o two-phase commit e por que ele é evitado entre serviços. | C2 | L06 |
| M14.U4 | Sagas, outbox e CDC | Coordenar uma operação em várias etapas com saga e publicar eventos com outbox ou CDC. | C2 | L04, L07 |
| M14.U5 | Contratos e evolução | Versionar uma API e um formato de mensagem sem quebrar os consumidores. | C3 | L10 |
| M14.U6 | Gateway e descoberta de serviços | Explicar API gateway e descoberta de serviços e quando cada um é necessário. | C1 | L06 |

## No Projeto prático

- **Parte do checkpoint:** `M10`.
- **Projeto prático:** decidir, com medição e um ADR, se o analytics vira um serviço próprio; publicar os cliques por CDC do banco. Decidir não separar, com evidência, também conclui o lab.

## Módulo concluído

1. **Artefato:** ADR com as medições que sustentam a decisão.
2. **Teste de falha:** se o analytics foi separado, derrubar um serviço no meio de uma operação e mostrar que outbox ou CDC não perde eventos. Se não foi, apresentar a evidência que sustentou a decisão.
3. **Alternativas:** monolito modular × serviço separado, com os limites de cada um.

## Cenário de transferência

O curador compõe um cenário: pagamento em várias etapas (reserva, cobrança, confirmação e estorno).

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Key Technologies › Temporal, API Gateway; Patterns › Multi-step Processes (pago: usar o resumo em In a Hurry › Patterns); Advanced Topics › Change Data Capture (pago). Pagos, só como título: Payment System, Job Scheduler.
- **Texto de base e pistas (Primer, índice):** Application layer (microservices, service discovery); RPC and REST calls comparison.
- **Cenários e números (SDI1, secundária):** não trata do tema.
- **Lente (DDIA, não conta como prova):** caps. 4, 9 e 12, via [codificacao-e-evolucao-de-esquema](../../knowledge/02-dados/codificacao-e-evolucao-de-esquema.md), [consistencia-e-consenso](../../knowledge/03-sistemas-distribuidos/consistencia-e-consenso.md) e [integracao-de-dados-e-correcao](../../knowledge/04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Referências primárias da lista:** Fowler, "MonolithFirst" e "MicroservicePrerequisites"; Newman, artigos públicos sobre microserviços (o livro *Building Microservices* fica fora do alcance do curador).
- **Candidatas a Referência primária (o Responsável pelo Currículo decide):** Tilkov, "Don't start with a monolith" (contraponto); Garcia-Molina e Salem, "Sagas" (1987); Richardson, padrões saga e outbox em microservices.io.
- **Documentação oficial:** Temporal; Debezium.
