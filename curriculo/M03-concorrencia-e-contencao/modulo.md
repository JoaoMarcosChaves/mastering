---
id: M03
titulo: "Concorrência e contenção"
trilho: nucleo
depende-de: [M02]
checkpoint-de-entrada: M02
status: esqueleto
---

# M03 · Concorrência e contenção

**Pergunta do módulo:** como garanto uma regra de negócio quando muita gente age ao mesmo tempo?

## Objetivo do módulo

Ao concluir, você identifica a regra que precisa valer sempre, escolhe o mecanismo mais simples que a protege (restrição do banco, nível de isolamento, lock ou idempotência) e prova com um teste concorrente que a regra se mantém.

## Competências

| ID | Competência |
|---|---|
| M03.C1 | Identificar a regra que precisa valer sempre e as anomalias de concorrência que a quebram. |
| M03.C2 | Escolher e justificar o mecanismo de proteção mais simples que funciona. |
| M03.C3 | Provar com teste concorrente que a regra se mantém. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M03.U1 | Condições de corrida | Achar a corrida em "verificar se o alias existe e depois inserir" e explicar por que ela acontece. | C1 | L03 |
| M03.U2 | Transações sem mito | Explicar o que cada letra do ACID garante e por que o C depende da aplicação. | C1 | L03 |
| M03.U3 | Isolamento e anomalias | Reproduzir uma atualização perdida e um write skew e dizer qual nível de isolamento evita cada um. | C1, C2 | L03 |
| M03.U4 | Mecanismos de proteção | Escolher entre UNIQUE, SELECT FOR UPDATE, versão otimista, SERIALIZABLE e lock distribuído com prazo. | C2 | L03, L06 |
| M03.U5 | Idempotência | Implementar a criação idempotente de link com chave de idempotência. | C2 | L07 |
| M03.U6 | Contenção sob pico | Garantir "não vender além da capacidade" com 1.000 requisições simultâneas. | C3 | L03 |

## No Projeto prático

- **Parte do checkpoint:** `M02`.
- **Lab:** `INSERT … ON CONFLICT`, chave de idempotência e teste com requisições simultâneas.
- **Lab isolado:** reserva com capacidade limitada, cuja regra é não vender além da capacidade.
- **Checkpoint de saída:** `M03`.

## Módulo concluído

1. **Artefato:** testes concorrentes versionados, no encurtador e no Lab isolado.
2. **Medição:** o teste falha sem a proteção e passa com ela, de forma reproduzível.
3. **Alternativas:** lock pessimista × otimista (ou SERIALIZABLE × restrição do banco), com os limites de cada um.

## Cenário de transferência

Ticketmaster (Hello Interview) e Shopify Inventory Reservations (In the Wild).

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Patterns › Dealing with Contention (pago: usar o resumo em In a Hurry › Patterns); Question Breakdowns › Ticketmaster; In the Wild › Shopify Inventory Reservations. Pagos, só como título: Online Auction, Flash Sale.
- **Texto de base e pistas (Primer, índice):** quase não trata transações. A frase de Consistency patterns que associa consistência forte a bancos relacionais vira afirmação a testar no lab.
- **Cenários e números (SDI1, secundária):** não cobre transações nem isolamento.
- **Lente (DDIA, não conta como prova):** cap. 7, via [transacoes](../../knowledge/03-sistemas-distribuidos/transacoes.md).
- **Referências primárias da lista:** nenhuma trata o tema diretamente.
- **Candidatas a Referência primária (o aprendiz decide):** Berenson et al., "A Critique of ANSI SQL Isolation Levels" (1995); análises do Jepsen sobre bancos relacionais.
- **Documentação oficial:** PostgreSQL (Transaction Isolation, Explicit Locking).
- **Pendência para a passada 2:** tema sem primária na lista atual.
