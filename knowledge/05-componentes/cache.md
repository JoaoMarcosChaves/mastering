---
titulo: Cache
fontes: [SDI1, DDIA1]
status-validacao: síntese de secundária (SDI1, vários capítulos) com o DDIA1 (Lente); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Cache

> Cache é uma cópia de dados (ou de resultados caros) mantida em um lugar mais rápido para atender leituras repetidas. Ele aparece em todas as camadas: navegador, CDN, aplicação, cache distribuído em memória, cache do banco e do sistema operacional. Para o DDIA1, todo cache é **dado derivado**: ele **desloca trabalho do caminho de leitura para o de escrita** e traz o problema de **mantê-lo coerente** com a fonte.

## Por que importa

Cache costuma ser a otimização de maior retorno para cargas com muita leitura, e também uma fonte clássica de bugs: dado antigo exibido, avalanche quando expira, inconsistência após escrita, ponto único de falha. Usá-lo bem exige saber **quando**, **onde** e **como invalidar**.

## Conceitos-chave

### Onde há cache

| Camada | Exemplo | Nota |
|---|---|---|
| Cliente/navegador | `Cache-Control: private, max-age=3600` em sugestões de busca (SDI1 cap. 13) | Zero rede; difícil de invalidar |
| CDN / borda | Estáticos e respostas públicas (ver [CDN](cdn.md)) | Perto do usuário; custo por transferência |
| Aplicação local | Mapa em memória do processo | Rápido; cada instância tem uma cópia diferente |
| Cache distribuído | Redis, Memcached | Compartilhado entre instâncias; mais um sistema para operar |
| DNS | Cache de resolução (SDI1 cap. 9: crawler) | Resolução de 10–200 ms vira gargalo sem cache |
| Banco / SO | Buffer pool, *page cache* | O DDIA1 lembra que um banco em disco com RAM suficiente já serve da memória |

### Padrões de leitura e escrita

- **Cache-aside:** a aplicação lê do cache; em caso de *miss*, lê do banco e grava no cache. É o padrão descrito no SDI1 cap. 1, que o chama de *read-through*. Na terminologia de mercado, *read-through* costuma designar o caso em que o **próprio cache** busca na fonte. Os nomes variam.
- **Write-through:** escreve no cache e na fonte de forma síncrona.
- **Write-behind / write-back:** escreve no cache e persiste de forma assíncrona. Rápido, mas com risco de perda.
- **Pré-computado (materializado):** o cache é preenchido pelo **caminho de escrita** (ex.: o feed de notícias com fan-out, a trie do autocomplete). Leitura O(1), escrita cara.

### Decisões do SDI1 (cap. 1)

- **Quando usar:** dado muito lido e pouco alterado. O cache é volátil: dados importantes ficam no armazenamento persistente.
- **Expiração (TTL):** nem curta demais (recarrega o banco o tempo todo), nem longa demais (dado antigo).
- **Consistência:** cache e banco não estão na mesma transação. Entre regiões, é ainda mais difícil. O SDI1 cita o artigo sobre o Memcache em escala no Facebook.
- **Falhas:** o cache vira **ponto único de falha**. Use vários nós (e datacenters) e memória sobrando.
- **Despejo:** **LRU** (mais comum), LFU, FIFO.

### O que guardar

- **IDs × objetos:** o feed de notícias do SDI1 guarda **só IDs** no cache de feed e **hidrata** com os caches de objetos. Economiza memória e evita duplicar objetos que mudam.
- **Camadas por tipo de dado** (SDI1 cap. 11): feed, conteúdo (com cache quente para os populares), grafo social, ações e contadores.
- **Top-k pré-computado** (SDI1 cap. 13): troca espaço por tempo quando a latência é crítica.

## O que o DDIA1 acrescenta

- **Cache é dado derivado.** Mantê-lo por **gravação dupla** (escrever no banco e depois invalidar ou atualizar o cache) está sujeito a **condições de corrida**: duas escritas concorrentes podem deixar o cache com o valor antigo **para sempre** (até o TTL), sem erro nenhum. Alternativas mais robustas:
  - atualizar ou invalidar a partir do **log de mudanças** (CDC), na mesma ordem das escritas;
  - **versionar** as chaves (a chave inclui a versão, e o dado antigo nunca é sobrescrito, só abandonado);
  - aceitar consistência eventual com **TTL curto** onde for aceitável.
  
  Ver [processamento de fluxo](../04-dados-derivados/processamento-de-fluxo.md).
- **Fronteira escrita × leitura:** caches, índices e views materializadas **deslocam** trabalho. A pergunta de projeto é quanto pré-computar para quais consultas (DDIA1 cap. 12).
- **Bancos em memória:** a vantagem não é "não ler do disco", e sim evitar codificar estruturas para o formato de disco. O Redis grava em disco de forma assíncrona (durabilidade fraca, panorama de 2017). O Memcached é só cache (DDIA1 cap. 3).
- **Ler as próprias escritas:** um cache (como uma réplica) pode mostrar ao usuário o dado de antes da escrita que ele acabou de fazer. Trate os fluxos em que isso importa.
- **Medir antes:** o DDIA1 insiste em parâmetros de carga e percentis. O cache se justifica por medição (hit ratio esperado, p99 de leitura, carga no banco), não por reflexo.

## Divergências e nuances

- **"Faça cache o máximo que puder" (SDI1, resumo do cap. 1)** × "cache é um problema de sincronização de dados derivados" (DDIA1). **Posição da trilha:** cache é a primeira ferramenta para leitura intensa, **introduzida com medição** e **com estratégia explícita de invalidação**.
- **"Invalidar o cache na escrita garante consistência forte" (SDI1 cap. 15):** não garante (corrida da gravação dupla, réplicas assíncronas). Ver [armazenamento e sincronização de arquivos](../06-estudos-de-caso/armazenamento-e-sincronizacao-de-arquivos.md).

## Problemas operacionais clássicos (conhecimento de mercado)

- **Avalanche de cache (*stampede*):** uma chave popular expira e milhares de requisições vão ao banco ao mesmo tempo. Mitigações: *request coalescing* (uma única requisição recompõe, as outras esperam), expiração com jitter, renovação antecipada (*stale-while-revalidate*).
- **Chave quente:** um item muito acessado sobrecarrega um único nó do cache distribuído. Mitigações: cache local em cada instância para os top-N, réplicas da chave.
- **Penetração de cache:** consultas a chaves inexistentes sempre vão ao banco. Mitigações: cachear o "não existe" com TTL curto, filtro de Bloom.
- **Aquecimento:** um cache frio após deploy ou reinício derruba o banco. Faça pré-aquecimento ou reinício gradual.

## Aplicação no mundo real

- **Redis** (estruturas de dados, TTL, operações atômicas, pub/sub) e **Memcached** (chave-valor simples, multithread). Confirme as garantias de persistência e replicação de cada modo na documentação oficial.
- **Métricas:** hit ratio, latência do cache, taxa de despejo, uso de memória, conexões.
- **HTTP:** `Cache-Control`, `ETag` e `Last-Modified` para cache no cliente e em proxies.

## No Projeto prático (encurtador de links)

- **Cache-aside no redirecionamento** (código → URL) em Redis, com TTL. Links imutáveis tornam a invalidação quase trivial. Com exclusão por abuso, é preciso invalidar a partir do evento de exclusão (outbox ou CDC).
- **Experimentos:**
  - hit ratio e p99 antes e depois;
  - **stampede:** expirar a chave de um link viral sob carga e medir o pico no banco, com e sem *request coalescing*;
  - derrubar o Redis (*fail-open*: ler do banco) e observar a degradação.

## Armadilhas comuns

- Cache sem TTL e sem estratégia de invalidação.
- Gravação dupla achando que isso dá consistência.
- Guardar dados que não podem se perder só no cache.
- Ignorar stampede e chaves quentes.
- Cache distribuído como novo ponto único de falha.

## Perguntas de verificação

1. Descreva cache-aside, write-through e write-behind, com uma vantagem e um risco de cada.
2. Por que invalidar o cache depois de escrever no banco pode deixar o valor antigo no cache?
3. O que é uma avalanche de cache e como mitigá-la?
4. Por que guardar IDs em vez de objetos no cache de feed?
5. Em que sentido um cache "desloca a fronteira" entre escrita e leitura?

## Referências

- [SDI1, cap. 1, seções "Cache" e "Considerations for using cache"].
- [SDI1, cap. 9 (cache de DNS), cap. 11 (arquitetura de cache do feed), cap. 13 (cache no navegador e top-k por nó), cap. 15 (cache de metadados e consistência)].
- [DDIA1, cap. 3, "Keeping everything in memory", pp. 88–90].
- [DDIA1, cap. 11, "Keeping Systems in Sync", pp. 452–454].
- [DDIA1, cap. 12, "Materialized views and caching", pp. 510–511].
