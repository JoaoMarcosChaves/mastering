---
id: M13
titulo: "Busca e dados especializados"
trilho: aprofundamento
depende-de: [M10]
checkpoint-de-entrada: M10
status: esqueleto
---

# M13 · Busca e dados especializados

**Pergunta do módulo:** quando o banco principal não basta, com busca textual, geografia, séries temporais ou ranking, o que muda?

## Objetivo do módulo

Ao concluir, você reconhece quando um caso pede um armazenamento especializado, mantém esse índice em sincronia com a fonte da verdade e usa estruturas aproximadas quando a exatidão custa caro demais.

## Competências

| ID | Competência |
|---|---|
| M13.C1 | Reconhecer quando um armazenamento especializado compensa e o custo de mantê-lo. |
| M13.C2 | Manter um índice derivado em sincronia com a fonte da verdade. |
| M13.C3 | Usar estruturas aproximadas quando a exatidão custa caro. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M13.U1 | Busca textual | Explicar o índice invertido e decidir quando o `LIKE` do banco deixa de bastar. | C1 | L02 |
| M13.U2 | Busca por proximidade | Comparar geohash, quadtree e índices geoespaciais para "o que está perto de mim". | C1 | L02 |
| M13.U3 | Séries temporais e armazenamento colunar | Escolher o armazenamento dos cliques para consultas analíticas por período. | C1 | L02 |
| M13.U4 | Índices derivados em sincronia | Manter um índice de busca atualizado com CDC ou gravação dupla consciente, e reindexar. | C2 | L04 |
| M13.U5 | Top K e estruturas aproximadas | Implementar top K e comparar contagem exata com count-min sketch, HyperLogLog e filtro de Bloom. | C3 | L01 |

## No Projeto prático

- **Parte do checkpoint:** `M10`.
- **Projeto prático:** ranking dos links mais clicados (top K).
- **Lab isolado:** busca por proximidade.

## Módulo concluído

1. **Artefato:** ranking e Lab isolado versionados.
2. **Medição:** exatidão, memória e latência da contagem exata comparadas com as da aproximada.
3. **Alternativas:** contagem exata × aproximada, com os limites de cada uma.

## Cenário de transferência

FB Post Search, Uber ou Top K.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Key Technologies › Elasticsearch; Advanced Topics › Proximity Search, Time Series Databases; Vector Databases e Data Structures for Big Data (pagos); Question Breakdowns › FB Post Search, Uber, Tinder, Top K, Local Delivery (Gopuff); In the Wild › Spotify Data Lake. Pago, só como título: Yelp.
- **Texto de base e pistas (Primer, índice):** soluções `query_cache`, `sales_rank` e a busca da solução `twitter`.
- **Cenários e números (SDI1, secundária):** cap. 13, via [autocomplete](../../knowledge/06-estudos-de-caso/autocomplete.md).
- **Lente (DDIA, não conta como prova):** caps. 3 e 10–12, via [armazenamento-e-indices](../../knowledge/02-dados/armazenamento-e-indices.md), [processamento-em-lote](../../knowledge/04-dados-derivados/processamento-em-lote.md) e [integracao-de-dados-e-correcao](../../knowledge/04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Referências primárias da lista:** nenhuma trata o tema diretamente.
- **Candidatas a Referência primária (o aprendiz decide):** Cormode e Muthukrishnan, count-min sketch (2005); Flajolet et al., HyperLogLog (2007).
- **Documentação oficial:** Elasticsearch ou OpenSearch; PostGIS; Redis (sorted sets e HyperLogLog).
- **Pendência para a passada 2:** tema sem primária na lista atual.
