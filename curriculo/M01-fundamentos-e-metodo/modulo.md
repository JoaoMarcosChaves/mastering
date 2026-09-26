---
id: M01
titulo: "Fundamentos e método"
trilho: nucleo
depende-de: [triagem]
checkpoint-de-entrada: nenhum
status: esqueleto
---

# M01 · Fundamentos e método

**Pergunta do módulo:** como transformo uma ideia em requisitos mensuráveis e numa primeira versão que funciona?

## Objetivo do módulo

Ao concluir, você parte de uma ideia de produto, escreve requisitos funcionais e não funcionais com números, estima carga e armazenamento, desenha e implementa a primeira versão do encurtador e descreve o desempenho dela em percentis medidos.

## Competências

| ID | Competência |
|---|---|
| M01.C1 | Transformar uma ideia em requisitos mensuráveis e num desenho inicial justificado. |
| M01.C2 | Estimar carga, armazenamento e banda, e conferir as próprias contas. |
| M01.C3 | Medir uma API sob carga e descrever o desempenho em percentis. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M01.U1 | O que é decidir uma arquitetura | Explicar confiabilidade, escalabilidade e manutenibilidade e dar um exemplo de decisão que melhora uma e piora outra. | C1 | L01, L06 |
| M01.U2 | Requisitos mensuráveis | Escrever os requisitos do encurtador com números para volume, latência (p99) e disponibilidade. | C1 | L01, L11 |
| M01.U3 | Roteiro de desenho | Aplicar ao encurtador o roteiro requisitos → entidades → API → fluxo → desenho geral → aprofundamentos. | C1 | L02 |
| M01.U4 | Estimativas de guardanapo | Estimar QPS, armazenamento e banda do encurtador, conferir a conta e achar o erro de 10× do exemplo do SDI1. | C2 | L01 |
| M01.U5 | Carga, latência e percentis | Explicar por que o p99 importa mais que a média e ler um relatório do k6. | C3 | L01, L11 |
| M01.U6 | A primeira versão do encurtador | Implementar criação e redirecionamento com código base62 e restrição UNIQUE, escolher entre 301 e 302 e medir p50, p95 e p99. | C1, C3 | L02, L11 |

## No Projeto prático

- **Parte de:** repositório vazio (é o primeiro módulo).
- **Lab:** API em TypeScript + PostgreSQL; código curto único (UNIQUE); sequência convertida para base62; script do k6 medindo p50, p95 e p99 de criação e de redirecionamento.
- **Checkpoint de saída:** `M01`.

## Módulo concluído

1. **Artefato:** repositório com criação e redirecionamento funcionando, documento de requisitos com números e script do k6 versionado.
2. **Medição:** relatório do k6 reproduzível com p50, p95 e p99.
3. **Alternativas:** código por hash com tratamento de colisão × base62 de uma sequência, com os limites de cada um (tamanho, previsibilidade, colisão).

## Cenário de transferência

Comparar três soluções publicadas para o encurtador (Hello Interview, Primer e SDI1) e apontar onde cada uma acerta, simplifica ou erra.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** In a Hurry › Delivery Framework; Core Concepts › API Design; Numbers to Know (pago: usar o resumo em In a Hurry › Core Concepts); Question Breakdowns › Bitly.
- **Texto de base e pistas (Primer, índice):** How to approach a system design interview question; Performance vs scalability; Latency vs throughput; Back-of-the-envelope calculations e Appendix (latências só como ordem de grandeza); System design topics: start here; solução `pastebin` (inclui o caso de uso de analytics).
- **Cenários e números (SDI1, secundária):** caps. 1, 2, 3, 7 e 8, via [metodo-de-design](../../knowledge/01-fundamentos/metodo-de-design.md), [estimativas-de-capacidade](../../knowledge/01-fundamentos/estimativas-de-capacidade.md), [gerador-de-ids-unicos](../../knowledge/06-estudos-de-caso/gerador-de-ids-unicos.md) e [encurtador-de-urls](../../knowledge/06-estudos-de-caso/encurtador-de-urls.md).
- **Lente (DDIA, não conta como prova):** cap. 1, via [confiabilidade](../../knowledge/01-fundamentos/confiabilidade.md), [escalabilidade-e-desempenho](../../knowledge/01-fundamentos/escalabilidade-e-desempenho.md) e [manutenibilidade](../../knowledge/01-fundamentos/manutenibilidade.md).
- **Referências primárias da lista:** AWS Well-Architected (visão geral dos pilares); Google SRE (capítulo de SLOs, para percentis e metas).
- **Documentação oficial:** k6; PostgreSQL (sequências e restrições); Node.js.
