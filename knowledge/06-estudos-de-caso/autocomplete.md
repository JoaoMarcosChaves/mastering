---
titulo: "Estudo de caso: autocomplete de busca (top-k por prefixo)"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) + princípios do DDIA1 (Lente); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Estudo de caso: autocomplete de busca (top-k por prefixo)

> Enquanto o usuário digita, mostrar as **k consultas mais populares** que começam com o prefixo, em **menos de ~100 ms**. A solução separa dois caminhos: um **caminho de leitura** muito rápido (uma trie com top-k pré-calculado em cada nó, em cache) e um **caminho de escrita** em lote (logs → agregação → reconstrução periódica da trie). É um exemplo claro de "deslocar trabalho da leitura para a escrita".

## Por que importa

Autocomplete, sugestões e "mais buscados" aparecem em busca, e-commerce e aplicativos. O caso ilustra bem **pré-computação**, **dados derivados reconstruídos em lote** e **particionamento com distribuição desigual** (há muito mais palavras começando com "c" do que com "x").

## Escopo do exercício (SDI1)

- Correspondência só pelo **início** da consulta.
- 5 sugestões, por **popularidade histórica**.
- Sem correção ortográfica.
- Inglês, minúsculas.
- 10 milhões de DAU.
- **Requisitos:** resposta rápida (o SDI1 cita um sistema de autocomplete de rede social com meta de ~100 ms), relevância, ordenação, escala e alta disponibilidade.

### Estimativas

- 10 buscas por usuário por dia, ~20 caracteres por consulta, **uma requisição por caractere digitado**: ≈ **24.000 QPS** (pico ≈ 48.000).
- ~20% de consultas novas por dia ≈ **0,4 GB/dia** de dados novos.

## Desenho

**Alto nível (ingênuo):**
- serviço de coleta atualiza uma tabela de frequências em tempo real;
- serviço de consulta faz `WHERE query LIKE 'prefixo%' ORDER BY frequency DESC LIMIT 5`.

Funciona com poucos dados e vira gargalo em escala.

**Trie (árvore de prefixos):**
- raiz = string vazia; cada nó representa um prefixo; guarda a frequência nas consultas completas;
- algoritmo básico: achar o nó do prefixo (O(p)), percorrer a subárvore (O(c)), ordenar (O(c log c)). Lento no pior caso;
- **otimizações:**
  1. **limitar o tamanho do prefixo** (ex.: 50), o que torna a busca do nó O(1) na prática;
  2. **guardar o top-k em cada nó**, o que torna a obtenção do top-k O(1). **Troca espaço por tempo** (vale a pena, porque latência é crítica).

**Serviço de coleta (caminho de escrita):**
- atualizar a trie a cada consulta é inviável (bilhões por dia) e desnecessário (o top muda devagar);
- **fluxo:** logs de analytics (append-only, sem índice) → **agregadores** (intervalo conforme a necessidade: curto para tendências, semanal para o resto) → dados agregados (consulta, semana, frequência) → **workers** reconstroem a trie periodicamente → **Trie DB** (persistência) → **Trie Cache** (em memória, com snapshot semanal).
- **Trie DB:** a trie serializada inteira em um banco de documentos, ou **key-value** com um par por prefixo → dados do nó.

**Serviço de consulta (caminho de leitura):**
- balanceador → servidores de API → Trie Cache (em caso de *miss*, repõe a partir do DB);
- **otimizações:**
  - requisições AJAX;
  - **cache no navegador** (o SDI1 mostra um mecanismo de busca usando `Cache-Control: private, max-age=3600`);
  - **amostragem** dos logs (1 a cada N consultas).

**Operações na trie:**
- **criar:** pelos workers, a partir dos dados agregados;
- **atualizar:** reconstruir semanalmente e trocar a antiga (preferível), ou atualizar o nó **e todos os ancestrais**, que guardam o top-k (lento);
- **apagar:** uma **camada de filtro** na frente do cache remove sugestões ofensivas ou perigosas na hora, e a remoção física no banco é assíncrona para a próxima reconstrução.

**Escalar o armazenamento:**
- shard pela primeira letra (até 26), depois pela segunda. Problema: **distribuição desigual**;
- solução: um **gerenciador de mapa de shards** baseado na distribuição histórica (ex.: "s" sozinho, "u–z" juntos).

**Extras:**
- vários idiomas (Unicode nos nós);
- tries por país (servidas pela CDN);
- **tendências em tempo real** (processamento de fluxo, mais peso para consultas recentes, sharding para reduzir o conjunto de trabalho).

## Conexões com os fundamentos (DDIA1)

- **Caminho de escrita × leitura:** o top-k pré-computado por nó é o DDIA1 cap. 12 na prática. Mais trabalho na escrita (reconstruir a trie) para leitura O(1). O DDIA1 usa o próprio exemplo de busca: sem índice, grep; com todos os resultados pré-computados, escrita infinita; o meio-termo é pré-computar o comum.
- **Dado derivado reconstruído em lote:** a trie é derivada de logs imutáveis. Reconstruir por inteiro e trocar de forma atômica (como os bancos somente leitura gerados por jobs em lote no DDIA1 cap. 10) facilita reverter se a nova versão estiver ruim. Ver [processamento em lote](../04-dados-derivados/processamento-em-lote.md).
- **Tendências = fluxo:** para frescor, agregação em janelas por **tempo do evento** e atualização incremental. Ver [processamento de fluxo](../04-dados-derivados/processamento-de-fluxo.md).
- **Particionamento por faixa com limites adaptados à distribuição:** é o que o DDIA1 descreve para o particionamento por faixa de chave (limites escolhidos conforme os dados, como os volumes de uma enciclopédia). Ver [particionamento](../03-sistemas-distribuidos/particionamento.md).
- **Índices de busca e busca difusa:** para correção ortográfica, o DDIA1 cita o Lucene com autômatos de Levenshtein. Ver [armazenamento e índices](../02-dados/armazenamento-e-indices.md).

## Aplicação no mundo real

- **Soluções prontas:** motores de busca (Elasticsearch/OpenSearch com *completion suggester* ou *search-as-you-type*; Meilisearch; Typesense). Confirme os recursos na documentação oficial.
- **Debounce no cliente:** não envie uma requisição por tecla. Espere uma pausa curta, o que reduz muito o QPS real em relação à estimativa ingênua.
- **Moderação** de sugestões é requisito de produto e reputação, não detalhe.
- **Privacidade:** logs de busca são sensíveis. Amostragem, anonimização e retenção limitada.

## No Projeto prático (encurtador de links)

- **Autocomplete de aliases:** ao criar um alias personalizado, sugerir disponíveis ou mostrar os aliases mais usados do próprio usuário por prefixo. Na escala do projeto, um índice B-tree com `LIKE 'prefixo%'` (ou índice de trigramas) basta. **Exercício:** medir a latência com 10 mil × 10 milhões de aliases e decidir, com números, se vale uma estrutura pré-computada.

## Armadilhas comuns

- Consultar o banco com `LIKE` e `ORDER BY` a cada tecla em escala.
- Atualizar a trie em tempo real a cada busca.
- Sharding pela primeira letra sem considerar a distribuição.
- Sem camada de filtro para conteúdo impróprio.
- Uma requisição por tecla sem debounce.

## Perguntas de verificação

1. Por que guardar o top-k em cada nó torna a consulta O(1)? Qual o custo?
2. Por que reconstruir a trie em lote em vez de atualizar em tempo real?
3. Como lidar com sugestões ofensivas sem esperar a próxima reconstrução?
4. Por que sharding pela primeira letra gera desbalanceamento, e como mitigar?
5. Como você adaptaria o desenho para tendências em tempo real?

## Referências

- [SDI1, cap. 13, "Design a Search Autocomplete System"]: escopo, estimativas, desenho ingênuo, trie, otimizações, serviço de coleta, Trie DB e cache, serviço de consulta, cache no navegador, amostragem, operações, sharding, extras.
- [DDIA1, cap. 12, "Materialized views and caching", pp. 510–511].
- [DDIA1, cap. 10, "Key-value stores as batch process output", pp. 412–413].
- [DDIA1, cap. 6, "Partitioning by Key Range", pp. 202–203].
