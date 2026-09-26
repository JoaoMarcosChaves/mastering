---
id: M11
titulo: "Escalar escritas: particionamento"
trilho: aprofundamento
depende-de: [M10]
checkpoint-de-entrada: M10
status: esqueleto
---

# M11 · Escalar escritas: particionamento

**Pergunta do módulo:** como divido dados e escritas entre máquinas sem criar pontos quentes?

## Objetivo do módulo

Ao concluir, você decide quando particionar compensa, escolhe a chave e a estratégia de particionamento, detecta e mitiga chaves quentes e planeja o rebalanceamento sem parar o sistema.

## Competências

| ID | Competência |
|---|---|
| M11.C1 | Escolher chave e estratégia de particionamento a partir dos padrões de acesso. |
| M11.C2 | Detectar e mitigar chaves quentes. |
| M11.C3 | Planejar rebalanceamento e consultas que atravessam partições. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M11.U1 | Quando particionar | Dizer quando particionar compensa e quando réplicas, federação ou uma máquina maior bastam. | C1 | L09 |
| M11.U2 | Faixa ou hash | Escolher a chave e a estratégia de particionamento para um caso dado. | C1 | L02, L09 |
| M11.U3 | Hashing consistente | Explicar hashing consistente e nós virtuais, e o que eles não resolvem. | C3 | L09 |
| M11.U4 | Chaves quentes | Detectar uma chave quente e mitigar com divisão da chave, cache ou fan-out ajustado. | C2 | L09 |
| M11.U5 | Índices e consultas entre partições | Comparar índice secundário local e global e o custo de consultas espalhadas. | C3 | L02, L09 |
| M11.U6 | Rebalanceamento e roteamento | Planejar o rebalanceamento sem parar o sistema e decidir quem roteia as requisições. | C3 | L06 |

## No Projeto prático

- **Parte do checkpoint:** `M10`.
- **Projeto prático (opcional):** particionar pelo hash do código curto, só se a medição justificar.
- **Lab isolado:** distribuição de carga entre partições e uma chave quente provocada e mitigada.

## Módulo concluído

1. **Artefato:** Lab isolado versionado.
2. **Medição:** distribuição de carga por partição, com uma chave quente antes e depois da mitigação.
3. **Alternativas:** particionamento por faixa × por hash, com os limites de cada um.

## Cenário de transferência

FB News Feed (fan-out e usuários com muitos seguidores) e Discord Message Storage (In the Wild).

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Core Concepts › Sharding, Consistent Hashing; Key Technologies › Cassandra, DynamoDB; Patterns › Scaling Writes (pago: usar o resumo em In a Hurry › Patterns); Question Breakdowns › FB News Feed; In the Wild › Discord Message Storage.
- **Texto de base e pistas (Primer, índice):** Sharding, Federation, Denormalization; soluções `twitter` e `social_graph`. Hashing consistente aparece como "em desenvolvimento".
- **Cenários e números (SDI1, secundária):** caps. 5, 6, 7 e 11, via [particionamento](../../knowledge/03-sistemas-distribuidos/particionamento.md), [key-value-store](../../knowledge/06-estudos-de-caso/key-value-store.md), [gerador-de-ids-unicos](../../knowledge/06-estudos-de-caso/gerador-de-ids-unicos.md) e [feed-de-noticias](../../knowledge/06-estudos-de-caso/feed-de-noticias.md). Divergência já registrada sobre hashing consistente e chaves quentes.
- **Lente (DDIA, não conta como prova):** cap. 6, via [particionamento](../../knowledge/03-sistemas-distribuidos/particionamento.md).
- **Referências primárias da lista:** nenhuma trata o tema diretamente.
- **Candidatas a Referência primária (o aprendiz decide):** Karger et al., hashing consistente (1997); DeCandia et al., "Dynamo" (2007).
- **Documentação oficial:** Cassandra e DynamoDB (chave de partição).
- **Pendência para a passada 2:** tema sem primária na lista atual.
