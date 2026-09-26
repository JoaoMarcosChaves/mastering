---
id: M09
titulo: "Observabilidade e SLO"
trilho: nucleo
depende-de: [M08]
checkpoint-de-entrada: M08
status: esqueleto
---

# M09 · Observabilidade e SLO

**Pergunta do módulo:** como provo, com números, que o sistema está atendendo bem quem usa?

## Objetivo do módulo

Ao concluir, você instrumenta a aplicação com métricas, logs e traces, define SLIs e SLOs para o redirecionamento, usa o orçamento de erros para decidir prioridades e configura alertas que disparam pelo sintoma que o usuário sente.

## Competências

| ID | Competência |
|---|---|
| M09.C1 | Instrumentar a aplicação e investigar um problema com métricas, logs e traces. |
| M09.C2 | Definir SLIs e SLOs e usar o orçamento de erros para decidir prioridades. |
| M09.C3 | Criar alertas acionáveis a partir dos SLOs. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M09.U1 | Métricas, logs e traces | Distinguir os três sinais e escolher as métricas RED e USE do encurtador. | C1 | L11 |
| M09.U2 | Instrumentação | Instrumentar a API com OpenTelemetry e ver o trace de um redirecionamento de ponta a ponta. | C1 | L11 |
| M09.U3 | SLI, SLO e SLA | Definir SLIs e SLOs de disponibilidade e de latência do redirecionamento, com percentil e janela. | C2 | L01, L11 |
| M09.U4 | Orçamento de erros | Usar o orçamento de erros para decidir entre lançar funcionalidade e investir em confiabilidade. | C2 | L11 |
| M09.U5 | Alertas | Criar alertas por sintoma e por taxa de consumo do orçamento, evitando excesso de alertas. | C3 | L11 |
| M09.U6 | Investigar com dados | Partir de um alerta, achar o trace lento e explicar a causa. | C1 | L06, L08 |

## No Projeto prático

- **Parte do checkpoint:** `M08`.
- **Lab:** OpenTelemetry + Prometheus + Grafana; SLO de disponibilidade e de latência do redirecionamento; alertas.
- **Checkpoint de saída:** `M09`.

## Módulo concluído

1. **Artefato:** configuração versionada (Compose e provisionamento do Grafana), mais o SLO escrito.
2. **Teste de falha:** uma falha provocada dispara o alerta certo.
3. **Alternativas:** alerta por limiar × alerta por taxa de consumo do orçamento, com os limites de cada um.

## Cenário de transferência

Definir SLOs para o Ad Click Aggregator do M07.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Advanced Topics › Time Series Databases. Pago, só como título: Metrics Monitoring.
- **Texto de base e pistas (Primer, índice):** não trata do tema.
- **Cenários e números (SDI1, secundária):** poucas linhas no cap. 1, via [escalando-uma-aplicacao-web](../../knowledge/05-componentes/escalando-uma-aplicacao-web.md).
- **Lente (DDIA, não conta como prova):** caps. 1 e 8, via [escalabilidade-e-desempenho](../../knowledge/01-fundamentos/escalabilidade-e-desempenho.md) e [falhas-em-sistemas-distribuidos](../../knowledge/03-sistemas-distribuidos/falhas-em-sistemas-distribuidos.md).
- **Referências primárias da lista:** Google SRE (capítulos de SLOs e de monitoramento de sistemas distribuídos); The Site Reliability Workbook (alertas baseados em SLO).
- **Documentação oficial:** OpenTelemetry; Prometheus; Grafana.
