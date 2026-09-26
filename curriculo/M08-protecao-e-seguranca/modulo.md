---
id: M08
titulo: "Proteção: limites, abuso e segurança"
trilho: nucleo
depende-de: [M07]
checkpoint-de-entrada: M07
status: esqueleto
---

# M08 · Proteção: limites, abuso e segurança

**Pergunta do módulo:** como impeço que o serviço seja abusado ou vire ferramenta de phishing?

## Objetivo do módulo

Ao concluir, você limita a taxa de uso com um algoritmo escolhido de forma justificada e que funciona com várias instâncias, aplica os controles básicos de segurança de uma API e decide que dados de clique coletar com critério de privacidade.

## Competências

| ID | Competência |
|---|---|
| M08.C1 | Escolher e implementar limitação de taxa que funcione com várias instâncias. |
| M08.C2 | Aplicar os controles básicos de segurança de uma API. |
| M08.C3 | Decidir que dados coletar e por quanto tempo, com critério de privacidade. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M08.U1 | Ameaças do encurtador | Listar os abusos possíveis (phishing, enumeração de códigos, scraping, spam) e o controle para cada um. | C2 | L03 |
| M08.U2 | Algoritmos de limitação | Comparar token bucket, leaky bucket, janela fixa e janela deslizante. | C1 | L09 |
| M08.U3 | Limitação distribuída | Implementar o limite com estado no Redis, sem condição de corrida, e decidir onde aplicá-lo. | C1 | L03, L06 |
| M08.U4 | Controles básicos | Validar entradas, autenticar a API e guardar segredos, usando o OWASP Top 10 como checklist. | C2 | L03 |
| M08.U5 | Criptografia e menor privilégio | Proteger dados em trânsito e em repouso e dar a cada componente só o acesso de que precisa. | C2 | L06 |
| M08.U6 | Privacidade nos cliques | Decidir que dados de clique coletar, por quanto tempo e com que base. | C3 | L12 |

## No Projeto prático

- **Parte do checkpoint:** `M07`.
- **Lab:** rate limiter com token bucket no Redis, validação de URL e lista de bloqueio.
- **Checkpoint de saída:** `M08`.

## Módulo concluído

1. **Artefato:** código versionado do limite de taxa e da validação, mais a política de retenção dos cliques escrita.
2. **Medição:** teste de carga mostrando o limite funcionando com duas instâncias.
3. **Alternativas:** dois algoritmos de limitação, com os limites de cada um.

## Cenário de transferência

Comparar o Distributed Rate Limiter do Hello Interview com o cap. 4 do SDI1.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Key Technologies › API Gateway; Question Breakdowns › Distributed Rate Limiter.
- **Texto de base e pistas (Primer, índice):** Security (seção mínima, que o próprio autor pede para atualizar) e seus links para OWASP e API security checklist.
- **Cenários e números (SDI1, secundária):** cap. 4, via [rate-limiter](../../knowledge/06-estudos-de-caso/rate-limiter.md).
- **Lente (DDIA, não conta como prova):** caps. 7 e 12, via [transacoes](../../knowledge/03-sistemas-distribuidos/transacoes.md) e [integracao-de-dados-e-correcao](../../knowledge/04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Referências primárias da lista:** AWS Well-Architected (pilar de segurança).
- **Candidatas a Referência primária (o Responsável pelo Currículo decide):** OWASP Top 10 e OWASP API Security Top 10.
- **Documentação oficial:** Redis (operações atômicas e scripts); Node.js (TLS e criptografia).
