---
id: M10
titulo: "Falhas e resiliência"
trilho: nucleo
depende-de: [M09]
checkpoint-de-entrada: M09
status: esqueleto
---

# M10 · Falhas e resiliência

**Pergunta do módulo:** o que acontece quando algo quebra, e como o sistema se recupera?

## Objetivo do módulo

Ao concluir, você provoca falhas de rede, de processo e de banco de forma controlada, aplica os padrões de resiliência adequados, executa um failover e uma restauração com metas conhecidas e escreve um postmortem sem culpados.

## Competências

| ID | Competência |
|---|---|
| M10.C1 | Nomear o modelo de falha e aplicar padrões de resiliência. |
| M10.C2 | Executar failover e restauração com metas de perda de dados e de tempo conhecidas. |
| M10.C3 | Conduzir a análise de um incidente sem buscar culpados. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M10.U1 | Falhas parciais | Explicar redes não confiáveis, relógios e pausas, e o que significa um nó "não responder". | C1 | L06, L08 |
| M10.U2 | Padrões de resiliência | Aplicar timeout, retentativa com espera crescente e aleatória, disjuntor e degradação graciosa. | C1 | L06 |
| M10.U3 | Failover e eleição de líder | Executar um failover e explicar split brain e fencing. | C2 | L06 |
| M10.U4 | Backup e restauração | Definir RPO e RTO, restaurar um backup e cronometrar. | C2 | L06 |
| M10.U5 | Injeção de falhas | Provocar latência e corte de rede com Toxiproxy a partir de uma hipótese. | C1 | L06 |
| M10.U6 | Incidentes e postmortems | Escrever um postmortem sem culpados de uma falha provocada no lab. | C3 | L06 |

## No Projeto prático

- **Parte do checkpoint:** `M09`.
- **Lab:** Toxiproxy (latência e corte de rede), failover do primário e restauração de backup, observados pelos painéis do M09.
- **Checkpoint de saída:** `M10`, o fim do núcleo.

## Módulo concluído

1. **Artefato:** experimento de falha versionado, com hipótese e resultado, mais o postmortem.
2. **Teste de falha:** failover demonstrado e restauração cronometrada.
3. **Alternativas:** retentativa × disjuntor (ou failover automático × manual), com os limites de cada um.

## Cenário de transferência

Web Crawler: como ele continua trabalhando quando páginas, DNS e workers falham.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Question Breakdowns › Web Crawler (confirmar os aprofundamentos de tolerância a falhas). O tema aparece espalhado nos aprofundamentos de vários problemas.
- **Texto de base e pistas (Primer, índice):** Availability patterns (fail-over ativo-passivo e ativo-ativo); disponibilidade em série × paralelo.
- **Cenários e números (SDI1, secundária):** caps. 6 e 9, via [key-value-store](../../knowledge/06-estudos-de-caso/key-value-store.md) e [web-crawler](../../knowledge/06-estudos-de-caso/web-crawler.md).
- **Lente (DDIA, não conta como prova):** caps. 5, 8 e 9, via [falhas-em-sistemas-distribuidos](../../knowledge/03-sistemas-distribuidos/falhas-em-sistemas-distribuidos.md), [consistencia-e-consenso](../../knowledge/03-sistemas-distribuidos/consistencia-e-consenso.md) e [replicacao](../../knowledge/03-sistemas-distribuidos/replicacao.md).
- **Referências primárias da lista:** Google SRE (cultura de postmortem; falhas em cascata); AWS Well-Architected (pilar de confiabilidade).
- **Candidatas a Referência primária (o Responsável pelo Currículo decide):** Amazon Builders' Library, "Timeouts, retries, and backoff with jitter".
- **Documentação oficial:** Toxiproxy; PostgreSQL (backup e restauração).
