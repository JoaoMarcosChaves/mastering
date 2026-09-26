---
id: M06
titulo: "Replicação e consistência"
trilho: nucleo
depende-de: [M05]
checkpoint-de-entrada: M05
status: esqueleto
---

# M06 · Replicação e consistência

**Pergunta do módulo:** que garantia de consistência a pessoa enxerga quando os dados têm cópias?

## Objetivo do módulo

Ao concluir, você escolhe um esquema de replicação, mede o atraso entre primário e réplica, corrige as anomalias que o usuário percebe e descreve o que o sistema faz durante uma partição real com garantias precisas, sem rótulos CP/AP.

## Competências

| ID | Competência |
|---|---|
| M06.C1 | Escolher e justificar um esquema de replicação. |
| M06.C2 | Identificar e resolver anomalias causadas pelo atraso de replicação. |
| M06.C3 | Descrever o comportamento do sistema durante uma partição com garantias precisas. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M06.U1 | Por que replicar | Explicar os ganhos da replicação e escolher entre replicação síncrona e assíncrona. | C1 | L05, L06 |
| M06.U2 | Líder único, multilíder e sem líder | Comparar os três esquemas e escolher um para um caso dado. | C1 | L05 |
| M06.U3 | Atraso de replicação | Reproduzir "ler as próprias escritas" e leituras não monotônicas, e corrigir as duas. | C2 | L05 |
| M06.U4 | Quóruns | Explicar o que w + r > n garante e o que não garante. | C3 | L05, L06 |
| M06.U5 | Modelos de consistência | Distinguir consistência forte (linearizável), causal e eventual, e separar isso do C do ACID. | C3 | L05 |
| M06.U6 | CAP em contexto | Descrever o que o sistema faz durante uma partição real, e o custo de latência fora dela. | C3 | L05, L06 |

## No Projeto prático

- **Parte do checkpoint:** `M05`.
- **Lab:** réplica de leitura do PostgreSQL; medir o atraso de replicação; corrigir "ler as próprias escritas" no fluxo criar e testar o link.
- **Checkpoint de saída:** `M06`.

## Módulo concluído

1. **Artefato:** Compose versionado com primário e réplica.
2. **Medição:** atraso de replicação medido, mais a anomalia reproduzida e corrigida.
3. **Alternativas:** ler do primário logo após uma escrita × rotear por sessão, com os limites de cada um.

## Cenário de transferência

Revisitar o key-value store do SDI1 (cap. 6) com a Lente: o que o quórum garante de fato?

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Core Concepts › CAP Theorem (ler com a Lente); Key Technologies › DynamoDB, Cassandra; ZooKeeper (pago); In the Wild › Meta ZGateway / ZippyDB (confirmar o tema).
- **Texto de base e pistas (Primer, índice):** Availability vs consistency (apresenta CAP como "2 de 3": ponto a corrigir); Consistency patterns; Replication (master-slave e master-master).
- **Cenários e números (SDI1, secundária):** cap. 6, via [key-value-store](../../knowledge/06-estudos-de-caso/key-value-store.md). Divergências já registradas: quórum e rótulos CP/AP.
- **Lente (DDIA, não conta como prova):** caps. 5 e 9 e o artigo "Please stop calling databases CP or AP", via [replicacao](../../knowledge/03-sistemas-distribuidos/replicacao.md) e [consistencia-e-consenso](../../knowledge/03-sistemas-distribuidos/consistencia-e-consenso.md).
- **Referências primárias da lista:** nenhuma trata o tema diretamente.
- **Candidatas a Referência primária (o aprendiz decide):** Brewer, "CAP Twelve Years Later" (2012); Gilbert e Lynch, prova do teorema CAP (2002); DeCandia et al., "Dynamo" (2007); Abadi, PACELC (2012).
- **Documentação oficial:** PostgreSQL (replicação por streaming).
- **Pendência para a passada 2:** tema sem primária na lista atual.
