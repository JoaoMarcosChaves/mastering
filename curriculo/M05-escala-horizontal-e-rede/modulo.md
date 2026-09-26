---
id: M05
titulo: "Escala horizontal e rede"
trilho: nucleo
depende-de: [M04]
checkpoint-de-entrada: M04
status: esqueleto
---

# M05 · Escala horizontal e rede

**Pergunta do módulo:** como atendo mais tráfego com mais máquinas, e o que muda na rede?

## Objetivo do módulo

Ao concluir, você explica o caminho de uma requisição, escolhe o protocolo de comunicação adequado, distribui o tráfego entre instâncias sem estado atrás de um balanceador e demonstra, com números, que o sistema continua respondendo quando uma instância cai.

## Competências

| ID | Competência |
|---|---|
| M05.C1 | Explicar o caminho de uma requisição e escolher o protocolo de comunicação adequado. |
| M05.C2 | Escalar horizontalmente com serviços sem estado atrás de um balanceador. |
| M05.C3 | Calcular e demonstrar disponibilidade com redundância. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M05.U1 | O caminho de uma requisição | Explicar DNS, TCP, TLS e HTTP e quanto cada etapa soma à latência. | C1 | L01 |
| M05.U2 | Protocolos de comunicação | Escolher entre REST, RPC e GraphQL, e entre HTTP e WebSocket, para um caso dado. | C1 | L10 |
| M05.U3 | Escala vertical e horizontal | Tornar a API sem estado e decidir onde fica a sessão. | C2 | L06 |
| M05.U4 | Balanceadores e gateways | Escolher entre balanceamento de camada 4 e 7, algoritmo e verificação de saúde; distinguir proxy reverso de API gateway. | C2 | L06, L09 |
| M05.U5 | Disponibilidade em números | Calcular a disponibilidade de componentes em série e em paralelo e traduzir noves em minutos de parada. | C3 | L11 |
| M05.U6 | Queda de uma instância | Derrubar uma instância sob carga e explicar o que o usuário viu. | C2, C3 | L06 |

## No Projeto prático

- **Parte do checkpoint:** `M04`.
- **Lab:** duas ou mais instâncias sem estado atrás de um balanceador; derrubar uma durante o teste de carga.
- **Checkpoint de saída:** `M05`.

## Módulo concluído

1. **Artefato:** Docker Compose versionado com duas ou mais instâncias e o balanceador.
2. **Medição:** teste de carga mostrando o efeito da queda de uma instância.
3. **Alternativas:** balanceamento de camada 4 × 7 (ou escala vertical × horizontal), com os limites de cada um.

## Cenário de transferência

Bitly com 100 milhões de usuários ativos por dia.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Core Concepts › Networking Essentials; Key Technologies › API Gateway; Load Balancer (resumo em In a Hurry › Key Technologies); Question Breakdowns › Bitly (aprofundamento de escala).
- **Texto de base e pistas (Primer, índice):** Domain name system; Load balancer; Reverse proxy; Application layer; Communication; Availability in numbers; solução `scaling_aws`.
- **Cenários e números (SDI1, secundária):** caps. 1–2, via [escalando-uma-aplicacao-web](../../knowledge/05-componentes/escalando-uma-aplicacao-web.md) e [estimativas-de-capacidade](../../knowledge/01-fundamentos/estimativas-de-capacidade.md).
- **Lente (DDIA, não conta como prova):** caps. 1 e 8, via [escalabilidade-e-desempenho](../../knowledge/01-fundamentos/escalabilidade-e-desempenho.md) e [falhas-em-sistemas-distribuidos](../../knowledge/03-sistemas-distribuidos/falhas-em-sistemas-distribuidos.md).
- **Referências primárias da lista:** AWS Well-Architected (confiabilidade, eficiência de desempenho e otimização de custo).
- **Documentação oficial:** NGINX ou HAProxy; MDN (HTTP); Docker Compose.
