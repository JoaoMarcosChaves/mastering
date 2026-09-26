---
id: M12
titulo: "Tempo real e arquivos grandes"
trilho: aprofundamento
depende-de: [M10]
checkpoint-de-entrada: M10
status: esqueleto
---

# M12 · Tempo real e arquivos grandes

**Pergunta do módulo:** como envio atualizações ao cliente e movo arquivos grandes sem sobrecarregar o servidor?

## Objetivo do módulo

Ao concluir, você escolhe e escala o mecanismo de atualização em tempo real, move arquivos grandes direto para o object storage e resolve conflitos quando vários clientes editam o mesmo dado.

## Competências

| ID | Competência |
|---|---|
| M12.C1 | Escolher e escalar o mecanismo de atualização em tempo real. |
| M12.C2 | Projetar o envio e a entrega de arquivos grandes. |
| M12.C3 | Resolver conflitos entre clientes que editam o mesmo dado. |

## Unidades

| ID | Unidade | Objetivo | Competências | Lente |
|---|---|---|---|---|
| M12.U1 | Polling, SSE e WebSocket | Escolher entre polling, long polling, SSE e WebSocket para um caso dado. | C1 | L01 |
| M12.U2 | Escalar conexões persistentes | Distribuir conexões entre servidores e entregar uma mensagem a quem está conectado em outro servidor. | C1 | L06, L09 |
| M12.U3 | Arquivos grandes | Enviar arquivos direto para o object storage com URL pré-assinada e envio em partes. | C2 | L04 |
| M12.U4 | Entrega de mídia | Explicar CDN, transcodificação e streaming adaptativo, e o custo de cada um. | C2 | L01 |
| M12.U5 | Sincronização e conflitos | Escolher entre "a última escrita vence", versões e CRDT para clientes que editam offline. | C3 | L05, L08 |

## No Projeto prático

- **Parte do checkpoint:** `M10`.
- **Projeto prático:** painel de cliques em tempo real (SSE ou WebSocket).
- **Lab isolado:** envio direto para object storage com URL pré-assinada.

## Módulo concluído

1. **Artefato:** painel e Lab isolado versionados.
2. **Medição:** teste de carga do painel com muitas conexões simultâneas.
3. **Alternativas:** SSE × WebSocket, com os limites de cada um.

## Cenário de transferência

WhatsApp, FB Live Comments, Dropbox ou YouTube.

## Fontes de partida

- **Estrutura (Hello Interview, secundária):** Patterns › Real-time Updates e Handling Large Blobs (pagos: usar o resumo em In a Hurry › Patterns); Question Breakdowns › WhatsApp, FB Live Comments, Dropbox, YouTube; In the Wild › Figma Multiplayer. Pagos, só como título: Google Docs, Online Chess.
- **Texto de base e pistas (Primer, índice):** Communication (TCP × UDP); Content delivery network.
- **Cenários e números (SDI1, secundária):** caps. 12, 14 e 15, via [chat](../../knowledge/06-estudos-de-caso/chat.md), [streaming-de-video](../../knowledge/06-estudos-de-caso/streaming-de-video.md) e [armazenamento-e-sincronizacao-de-arquivos](../../knowledge/06-estudos-de-caso/armazenamento-e-sincronizacao-de-arquivos.md).
- **Lente (DDIA, não conta como prova):** caps. 5 e 12, via [replicacao](../../knowledge/03-sistemas-distribuidos/replicacao.md) e [integracao-de-dados-e-correcao](../../knowledge/04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Referências primárias da lista:** nenhuma trata o tema diretamente.
- **Candidatas a Referência primária (o Responsável pelo Currículo decide):** post original do Figma sobre a tecnologia multiplayer; Shapiro et al., CRDTs (2011).
- **Documentação oficial:** MDN (SSE e WebSocket); RFC 6455; S3 ou GCS (URLs pré-assinadas e envio em partes).
- **Pendência para a passada 2:** tema sem primária na lista atual.
