---
id: M02
titulo: "Dados: modelagem, armazenamento e índices"
trilho: nucleo
depende-de: [M01]
checkpoint-de-entrada: M01
status: esqueleto
---

# M02 · Dados: modelagem, armazenamento e índices

**Pergunta do módulo:** como escolho o modelo de dados e os índices a partir de como os dados são lidos e escritos?

## Objetivo do módulo

Ao concluir, você escolhe o modelo de dados a partir dos padrões de acesso, cria índices justificados e mede o efeito deles, e evolui um esquema sem quebrar a versão anterior da aplicação.

## Competências

| ID | Competência |
|---|---|
| M02.C1 | Escolher o modelo de dados a partir dos padrões de acesso. |
| M02.C2 | Criar e justificar índices, medindo o efeito. |
| M02.C3 | Evoluir esquema e formato de dados sem quebrar a versão anterior. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M02.U1 | Padrões de acesso primeiro | Listar como o encurtador lê e escreve dados e separar o caminho transacional do analítico. | C1 | L02 |
| M02.U2 | Modelos de dados | Escolher entre relacional, documento, chave-valor, colunar larga e grafo para links e para cliques. | C1 | L02 |
| M02.U3 | Normalizar ou desnormalizar | Decidir onde duplicar dados no encurtador e dizer o custo de manter as cópias. | C1 | L02, L04 |
| M02.U4 | Como o banco guarda e acha dados | Explicar log, hash, B-tree e LSM, e por que um índice acelera leituras e encarece escritas. | C2 | L02 |
| M02.U5 | Índices na prática | Criar um índice para a consulta de cliques por link e medir antes e depois com EXPLAIN ANALYZE. | C2 | L02, L11 |
| M02.U6 | Evolução de esquema e codificação | Fazer uma migração compatível com a versão anterior e comparar JSON com formatos binários com esquema. | C3 | L10 |

## No Projeto prático

- **Parte do checkpoint:** `M01`.
- **Lab:** esquema de links e de cliques; índices medidos com EXPLAIN ANALYZE; migração que adiciona expiração de link sem quebrar a versão anterior.
- **Checkpoint de saída:** `M02`.

## Módulo concluído

1. **Artefato:** migrações versionadas do esquema.
2. **Medição:** EXPLAIN ANALYZE reproduzível antes e depois de cada índice.
3. **Alternativas:** cliques numa tabela relacional × num armazenamento colunar ou de séries temporais, com os limites de cada um.

## Cenário de transferência

O curador compõe um cenário: nenhum problema gratuito do Hello Interview é centrado em modelagem. Sugestão: modelar os dados de um sistema de reservas.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Core Concepts › Data Modeling; Database Indexing (pago: usar o resumo em In a Hurry › Core Concepts); Key Technologies › DynamoDB, Cassandra; PostgreSQL (pago).
- **Texto de base e pistas (Primer, índice):** Database › Relational database management system, Denormalization, SQL tuning; NoSQL (key-value, document, wide column, graph); SQL or NoSQL.
- **Cenários e números (SDI1, secundária):** pouco além do modelo de dados do cap. 8, via [encurtador-de-urls](../../knowledge/06-estudos-de-caso/encurtador-de-urls.md).
- **Lente (DDIA, não conta como prova):** caps. 2–4, via [modelos-de-dados-e-consulta](../../knowledge/02-dados/modelos-de-dados-e-consulta.md), [armazenamento-e-indices](../../knowledge/02-dados/armazenamento-e-indices.md) e [codificacao-e-evolucao-de-esquema](../../knowledge/02-dados/codificacao-e-evolucao-de-esquema.md).
- **Referências primárias da lista:** Fowler, artigos sobre evolução de bancos de dados e persistência poliglota (confirmar quais).
- **Candidatas a Referência primária (o Responsável pelo Currículo decide):** Markus Winand, *Use The Index, Luke* (índices em SQL).
- **Documentação oficial:** PostgreSQL (índices, EXPLAIN); Protocol Buffers.
- **Pendência para a passada 2:** a lista atual quase não tem primárias para modelos e índices.
