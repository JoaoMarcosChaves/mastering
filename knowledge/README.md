# Base de conhecimento de system design

Conhecimento de system design extraído da leitura integral de dois livros e reorganizado **por tema**, com aplicação ao mundo real, conexão com o Projeto prático da trilha (encurtador de links) e referência exata à origem de cada ideia.

É a base de conteúdo a partir da qual o Núcleo validado da Trilha de System Design é produzido (ver `.red/CONTEXT.md`). **Estes arquivos ainda não são o Núcleo validado:** cada um declara seu status de validação no cabeçalho (ver "Status de validação" abaixo).

## Política de fontes e direitos autorais

- **Os livros ficam só na máquina local**, em `knowledge-resources/`, pasta fora do git. **Nunca versione os PDFs nem o texto extraído deles.**
- **Nada aqui reproduz o texto dos livros.** O conteúdo é **síntese própria**, reorganizada por tema, com exemplos e aplicações próprias. Não há trechos copiados, figuras nem tabelas reproduzidas. Fatos, números de exemplo e nomes técnicos são citados com a origem.
- **Toda afirmação relevante aponta a origem** na seção "Referências" do arquivo, no formato `[CHAVE, capítulo, "seção", páginas]`. As chaves estão em [fontes.md](fontes.md).
- **Menções a ferramentas, produtos e preços atuais** são indicativas. Antes de virar exercício, confirme na **documentação oficial** (regra da trilha: fatos de ferramenta são validados pela documentação oficial).
- **Opiniões declaradas** de um autor ficam marcadas como **[opinião do autor]**.

## Status de validação

A regra da trilha para um conteúdo entrar no **Núcleo validado** é: conceitos e trade-offs precisam convergir em **≥2 Referências primárias independentes além do Primer**. Referências secundárias (Alex Xu/ByteByteGo, Hello Interview) não contam para esse mínimo. Divergências viram trade-off explícito.

Hoje:

| Fonte | Papel | Conta para a convergência? |
|---|---|---|
| DDIA1 (Kleppmann, 1ª ed.) | Primária | Sim (1 de 2 necessárias) |
| SDI1 (Alex Xu, vol. 1) | Secundária | Não: estrutura cenários e aponta divergências |

Por isso cada arquivo traz `status-validacao: ... aguarda convergência com ≥2 primárias`. O próximo passo do processo é confrontar cada tema com as demais Referências primárias da lista de validação (Primer, Fowler, Newman, Google SRE, AWS Well-Architected, artigos do Kleppmann).

## Estrutura de cada arquivo

Cabeçalho com `fontes` e `status-validacao`, seguido de:

1. **Resumo** em 2–3 linhas.
2. **Por que importa.**
3. **Conceitos-chave.**
4. **Trade-offs e decisões.**
5. **Divergências entre as fontes**, quando houver.
6. **Aplicação no mundo real.**
7. **No Projeto prático**: como o tema aparece no encurtador de links.
8. **Armadilhas comuns.**
9. **Perguntas de verificação**, semente para Pílulas e revisões espaçadas.
10. **Referências.**

## Índice

### 01 · Fundamentos

- [Confiabilidade](01-fundamentos/confiabilidade.md): falhas de componente × do sistema, hardware, software, erro humano.
- [Escalabilidade e desempenho](01-fundamentos/escalabilidade-e-desempenho.md): parâmetros de carga, percentis, latência de cauda, vertical × horizontal.
- [Manutenibilidade](01-fundamentos/manutenibilidade.md): operabilidade, simplicidade, evolutividade.
- [Estimativas de capacidade](01-fundamentos/estimativas-de-capacidade.md): QPS, armazenamento, noves de disponibilidade, ordens de grandeza.
- [Método de design](01-fundamentos/metodo-de-design.md): as 4 etapas, requisitos mensuráveis, do roteiro de entrevista ao projeto real.

### 02 · Dados

- [Modelos de dados e consulta](02-dados/modelos-de-dados-e-consulta.md): relacional, documento, grafo, declarativo × imperativo.
- [Armazenamento e índices](02-dados/armazenamento-e-indices.md): log, hash, LSM × B-tree, índices secundários, OLTP × OLAP, colunar.
- [Codificação e evolução de esquema](02-dados/codificacao-e-evolucao-de-esquema.md): JSON, Protobuf, Avro, compatibilidade, REST/RPC, brokers.

### 03 · Sistemas distribuídos

- [Replicação](03-sistemas-distribuidos/replicacao.md): líder único, multilíder, sem líder, atraso de replicação, quóruns.
- [Particionamento e hashing consistente](03-sistemas-distribuidos/particionamento.md): faixa × hash, índices secundários, rebalanceamento, roteamento.
- [Transações e isolamento](03-sistemas-distribuidos/transacoes.md): ACID, anomalias, snapshot, write skew, 2PL, SSI.
- [Falhas em sistemas distribuídos](03-sistemas-distribuidos/falhas-em-sistemas-distribuidos.md): redes, relógios, pausas, quóruns, fencing, modelos de sistema.
- [Consistência e consenso](03-sistemas-distribuidos/consistencia-e-consenso.md): linearizabilidade, CAP, causalidade, Lamport, 2PC, Raft/Paxos, ZooKeeper.

### 04 · Dados derivados

- [Processamento em lote](04-dados-derivados/processamento-em-lote.md): filosofia Unix, MapReduce, joins, saídas de lote, motores de dataflow.
- [Processamento de fluxo e filas](04-dados-derivados/processamento-de-fluxo.md): brokers × logs, CDC, event sourcing, janelas, joins, exatamente uma vez.
- [Integração de dados e correção](04-dados-derivados/integracao-de-dados-e-correcao.md): dados derivados × transações distribuídas, ponta a ponta, pontualidade × integridade, ética.

### 05 · Componentes

- [Escalando uma aplicação web](05-componentes/escalando-uma-aplicacao-web.md): mapa de componentes de um servidor a milhões de usuários.
- [Cache](05-componentes/cache.md): camadas, padrões, invalidação, problemas operacionais.
- [CDN](05-componentes/cdn.md): funcionamento, TTL, invalidação, custo.

### 06 · Estudos de caso

- [Encurtador de URLs](06-estudos-de-caso/encurtador-de-urls.md) ★ **Projeto prático da trilha**, com sugestão de evolução por módulos.
- [Rate limiter](06-estudos-de-caso/rate-limiter.md)
- [Key-value store distribuído](06-estudos-de-caso/key-value-store.md)
- [Gerador de IDs únicos](06-estudos-de-caso/gerador-de-ids-unicos.md)
- [Web crawler](06-estudos-de-caso/web-crawler.md)
- [Sistema de notificações](06-estudos-de-caso/sistema-de-notificacoes.md)
- [Feed de notícias](06-estudos-de-caso/feed-de-noticias.md)
- [Chat](06-estudos-de-caso/chat.md)
- [Autocomplete de busca](06-estudos-de-caso/autocomplete.md)
- [Plataforma de vídeo](06-estudos-de-caso/streaming-de-video.md)
- [Armazenamento e sincronização de arquivos](06-estudos-de-caso/armazenamento-e-sincronizacao-de-arquivos.md)

## Principais divergências entre as fontes

As divergências são conteúdo da trilha: aparecem como trade-offs a serem decididos, não escondidas.

| Tema | SDI1 | DDIA1 | Onde |
|---|---|---|---|
| CAP | "2 de 3"; classifica sistemas em CP/AP/CA | Formulação enganosa; evitar rótulos; "consistente ou disponível quando particionado"; o custo diário é latência | [consistência e consenso](03-sistemas-distribuidos/consistencia-e-consenso.md), [key-value store](06-estudos-de-caso/key-value-store.md) |
| Quóruns | w + r > n garante consistência forte | Não garante linearizabilidade (casos de borda) | [key-value store](06-estudos-de-caso/key-value-store.md), [replicação](03-sistemas-distribuidos/replicacao.md) |
| Hashing consistente | Técnica padrão, mitiga chaves quentes | Termo confuso; não resolve uma chave única quente; avaliar a estratégia concreta de rebalanceamento | [particionamento](03-sistemas-distribuidos/particionamento.md), [feed](06-estudos-de-caso/feed-de-noticias.md) |
| Cache | "Faça cache o máximo possível"; invalidar na escrita dá consistência forte | Cache é dado derivado; gravação dupla tem corrida; introduzir com medição | [cache](05-componentes/cache.md), [arquivos](06-estudos-de-caso/armazenamento-e-sincronizacao-de-arquivos.md) |
| ACID e consistência | Relacional = consistência forte "fácil" | Isolamento fraco por padrão; C do ACID é da aplicação; réplicas assíncronas | [transações](03-sistemas-distribuidos/transacoes.md) |

## Leituras complementares

O SDI1 (cap. 16) lista arquiteturas reais publicadas por empresas (Facebook, Amazon Dynamo, Netflix, Google, Twitter, Uber, Dropbox, WhatsApp, entre outras) e blogs de engenharia, inclusive o System Design Primer. Muitos links do livro são encurtados e podem estar desatualizados. Localize as versões atuais antes de usar.
