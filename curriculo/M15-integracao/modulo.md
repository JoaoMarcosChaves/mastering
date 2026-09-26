---
id: M15
titulo: "Integração: cenários mistos e entrevista"
trilho: integracao
depende-de: [M11, M12, M13, M14]
checkpoint-de-entrada: M10
status: esqueleto
---

# M15 · Integração: cenários mistos e entrevista

**Pergunta do módulo:** consigo decidir uma arquitetura nova, sozinho e com tempo contado?

## Objetivo do módulo

Ao concluir, você conduz o desenho de um sistema que nunca viu, do requisito aos aprofundamentos, em 45 a 60 minutos. Justifica cada escolha com o que mediu nos módulos anteriores e revisa as decisões do encurtador à luz das medições acumuladas.

## Competências

| ID | Competência |
|---|---|
| M15.C1 | Conduzir, com tempo contado, o desenho completo de um problema novo. |
| M15.C2 | Justificar decisões com evidência medida e dizer quando não sabe. |
| M15.C3 | Revisar decisões anteriores diante de evidência nova. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M15.U1 | Roteiro com tempo contado | Conduzir o roteiro de desenho em 45 a 60 minutos e saber o que se espera em cada nível. | C1 | L01 |
| M15.U2 | Cenários mistos | Resolver problemas que exigem escolher entre técnicas de vários módulos. | C1 | L01–L12 |
| M15.U3 | Arquiteturas reais | Ler um post de engenharia com olhar crítico: requisitos, decisão, evidência e o que ficou de fora. | C2 | L01–L12 |
| M15.U4 | Revisão das decisões do encurtador | Revisar os ADRs do encurtador à luz das medições acumuladas. | C3 | L11 |
| M15.U5 | Comunicar trade-offs | Explicar uma decisão e o que ela custa para quem não é técnico. | C2 | — |

## No Projeto prático

- **Parte do checkpoint:** o estado do encurtador ao fim dos aprofundamentos.
- **Lab:** revisão dos ADRs do encurtador, confirmando, ajustando ou revertendo cada decisão com as medições.

## Módulo concluído

1. **Artefato:** ADRs do encurtador revisados.
2. **Desempenho:** três cenários novos defendidos com tempo contado, com nível ≥3 na rubrica.
3. **Alternativas:** em cada cenário, duas alternativas e os limites de cada uma.

## Cenário de transferência

Repetir cenários 2 a 4 semanas depois. É a evidência do Nível 4, que conta como Conquista, não como requisito.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** In a Hurry › Delivery Framework e as expectativas por nível; os 16 Question Breakdowns gratuitos, misturados; In the Wild (os 6 casos).
- **Texto de base e pistas (Primer, índice):** Additional system design interview questions; Real world architectures; Company architectures; Company engineering blogs.
- **Cenários e números (SDI1, secundária):** caps. 3 e 16 e os casos ainda não usados, via [metodo-de-design](../../knowledge/01-fundamentos/metodo-de-design.md) e [`knowledge/06-estudos-de-caso/`](../../knowledge/06-estudos-de-caso/).
- **Lente (DDIA, não conta como prova):** cap. 12 e todas as perguntas L01–L12, como checklist de revisão.
- **Referências primárias da lista:** os posts de engenharia originais por trás dos casos In the Wild, promovidos pelo Responsável pelo Currículo caso a caso.
