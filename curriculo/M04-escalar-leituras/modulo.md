---
id: M04
titulo: "Escalar leituras: cache e CDN"
trilho: nucleo
depende-de: [M03]
checkpoint-de-entrada: M03
status: esqueleto
---

# M04 · Escalar leituras: cache e CDN

**Pergunta do módulo:** quando um cache compensa, e que inconsistência ele traz junto?

## Objetivo do módulo

Ao concluir, você decide se e onde colocar cache com base em medição, escolhe a estratégia de atualização sabendo quanta desatualização ela aceita, trata os problemas operacionais de cache e decide o que servir por CDN.

## Competências

| ID | Competência |
|---|---|
| M04.C1 | Decidir se e onde cachear, com medição antes e depois. |
| M04.C2 | Escolher a estratégia de atualização e de invalidação sabendo a inconsistência que ela aceita. |
| M04.C3 | Tratar problemas operacionais de cache: avalanche, chave quente e falta de memória. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M04.U1 | Onde cachear | Listar as camadas de cache possíveis no redirecionamento e escolher uma com justificativa. | C1 | L01, L04 |
| M04.U2 | Cache como dado derivado | Definir por quanto tempo um link pode ficar desatualizado no cache e o que isso significa para o produto. | C2 | L04, L05 |
| M04.U3 | Estratégias de atualização | Escolher entre cache-aside, read-through, write-through, write-behind e refresh-ahead e explicar a corrida da gravação dupla. | C2 | L04 |
| M04.U4 | Remoção e dimensionamento | Medir a taxa de acerto e ajustar tamanho, política de remoção e TTL. | C1, C3 | L11 |
| M04.U5 | Problemas operacionais | Reproduzir uma avalanche de cache e uma chave quente, e mitigar as duas. | C3 | L06, L09 |
| M04.U6 | CDN | Decidir o que do encurtador vai para a CDN, sabendo que um 301 guardado em cache esconde cliques do analytics. | C1 | L04 |

## No Projeto prático

- **Parte do checkpoint:** `M03`.
- **Lab:** Redis com cache-aside no redirecionamento, TTL, taxa de acerto e p99 medidos antes e depois.
- **Checkpoint de saída:** `M04`.

## Módulo concluído

1. **Artefato:** código e configuração do Redis versionados.
2. **Medição:** p99 e taxa de acerto antes e depois do cache, mais uma avalanche reproduzida e mitigada.
3. **Alternativas:** cache-aside × write-through (ou cache × réplica de leitura), com os limites de cada um.

## Cenário de transferência

Ticketmaster na visualização de eventos durante um pico de acesso (outra face do problema do M03).

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Core Concepts › Caching; Key Technologies › Redis; CDN (resumo em In a Hurry › Key Technologies); Patterns › Scaling Reads (pago: usar o resumo em In a Hurry › Patterns); Question Breakdowns › Bitly (aprofundamento de redirecionamento rápido). Pago, só como título: Distributed Cache.
- **Texto de base e pistas (Primer, índice):** Cache (todas as subseções); Content delivery network; solução `query_cache`.
- **Cenários e números (SDI1, secundária):** cap. 1, via [cache](../../knowledge/05-componentes/cache.md), [cdn](../../knowledge/05-componentes/cdn.md) e [escalando-uma-aplicacao-web](../../knowledge/05-componentes/escalando-uma-aplicacao-web.md). Divergência já registrada: "faça cache o máximo possível".
- **Lente (DDIA, não conta como prova):** caps. 5 e 11–12, via [replicacao](../../knowledge/03-sistemas-distribuidos/replicacao.md) e [integracao-de-dados-e-correcao](../../knowledge/04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Referências primárias da lista:** AWS Well-Architected (pilar de eficiência de desempenho).
- **Candidatas a Referência primária (o aprendiz decide):** Nishtala et al., "Scaling Memcache at Facebook" (NSDI 2013).
- **Documentação oficial:** Redis (expiração e políticas de remoção); CDN escolhida (TTL, invalidação, custo).
