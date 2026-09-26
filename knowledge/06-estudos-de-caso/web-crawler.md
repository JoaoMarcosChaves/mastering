---
titulo: "Estudo de caso: web crawler"
fontes: [SDI1, DDIA1]
status-validacao: base secundária (SDI1) + princípios do DDIA1 (Lente); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Estudo de caso: web crawler

> Um crawler parte de URLs iniciais, baixa as páginas, extrai links e repete. O algoritmo é simples. O difícil é fazer isso com **escala** (bilhões de páginas), **robustez** (a web é cheia de armadilhas), **educação** (não derrubar sites) e **extensibilidade** (novos tipos de conteúdo). É um excelente exemplo de pipeline assíncrono com filas, deduplicação e priorização.

## Por que importa

Serve para indexação de busca, arquivamento, mineração de dados e monitoramento de direitos autorais. Os padrões (fila com prioridade e limites por destino, deduplicação por hash, filtros de Bloom, retry, workers sem estado) aparecem em qualquer sistema que consome recursos externos em volume: integrações, scrapers, verificadores de links.

## Escopo do exercício (SDI1)

- **Premissas:** indexação de busca; 1 bilhão de páginas por mês; só HTML; considerar páginas novas e editadas; guardar por 5 anos; ignorar duplicatas.
- **Qualidades:** escalabilidade (paralelismo), robustez (HTML ruim, servidores que não respondem, links maliciosos), educação (não fazer requisições demais ao mesmo site) e extensibilidade.
- **Estimativas:**
  - ~400 páginas/s (pico ~800);
  - ~500 KB por página (premissa do livro), dando ~500 TB por mês e ~30 PB em 5 anos.

## Componentes e fluxo

1. **URLs iniciais (seeds):** divididas por localidade ou por tema.
2. **Fronteira de URLs:** o que falta baixar (fila).
3. **Downloader HTML** + **resolvedor DNS**.
4. **Parser de conteúdo:** valida o HTML, em componente separado para não atrasar o download.
5. **"Conteúdo já visto?":** o SDI1 cita ~29% da web como conteúdo duplicado. Compara **hash** da página, não caractere por caractere.
6. **Armazenamento de conteúdo:** a maior parte em disco, o conteúdo popular em memória.
7. **Extrator de links:** converte caminhos relativos em absolutos.
8. **Filtro de URLs:** tipos, extensões, erros, lista de bloqueio.
9. **"URL já vista?":** **filtro de Bloom** ou tabela hash, para evitar repetição e laços.
10. **Armazenamento de URLs visitadas.**

## Aprofundamentos

- **BFS × DFS:** a web é um grafo dirigido. DFS vai fundo demais. BFS (fila FIFO) é o padrão, mas a BFS ingênua **martela o mesmo host** (links internos) e **ignora prioridade**.
- **Fronteira de URLs, em duas camadas:**
  - **filas de frente (prioridade):** um *prioritizador* calcula a importância (PageRank, tráfego, frequência de atualização) e o seletor escolhe filas com viés para as mais prioritárias;
  - **filas de trás (educação):** um roteador garante **uma fila por host**. Cada worker consome uma fila, baixa uma página por vez e espera entre downloads.
- **Atualização (freshness):** recrawl guiado pelo histórico de mudanças da página e pela importância.
- **Armazenamento da fronteira:** centenas de milhões de URLs. Híbrido: a maior parte em disco, com buffers em memória para enfileirar e desenfileirar.
- **Downloader:**
  - respeitar o **robots.txt** (com cache);
  - crawl distribuído (espaço de URLs particionado entre servidores e threads);
  - **cache de DNS** (resolução de 10–200 ms que bloqueia threads);
  - localidade geográfica;
  - **timeouts curtos**.
- **Robustez:**
  - hashing consistente entre downloaders;
  - **salvar o estado do crawl** para retomar;
  - tratamento de exceções sem derrubar o sistema;
  - validação de dados.
- **Extensibilidade:** módulos plugáveis (ex.: downloader de PNG, monitor de direitos autorais).
- **Conteúdo problemático:**
  - duplicatas (hash);
  - **armadilhas de aranha** (diretórios infinitos: limite de tamanho de URL, detecção manual, filtros);
  - ruído (anúncios, spam).
- **Extras:**
  - **renderização dinâmica** para páginas geradas por JavaScript;
  - antispam;
  - replicação e sharding;
  - workers **sem estado** para escalar horizontalmente;
  - analytics.

## Conexões com os fundamentos (DDIA1)

- **Pipeline de lote e fluxo:** fronteira → download → parse → dedup → extração → fronteira é um **dataflow**. A construção do índice de busca em si é o caso de uso original do MapReduce. Ver [processamento em lote](../04-dados-derivados/processamento-em-lote.md).
- **Filtros de Bloom:** a mesma estrutura das LSM-trees para evitar leituras inúteis, com falsos positivos possíveis (algumas URLs novas seriam puladas por engano, o que é aceitável aqui).
- **Chaves quentes e educação:** particionar a fronteira **por host** é uma escolha de chave de partição que evita pontos quentes nos sites de destino.
- **Idempotência e retomada:** reprocessar uma URL após uma falha não pode duplicar conteúdo. Deduplicar por hash e por URL torna o reprocessamento seguro.
- **Timeouts e falhas parciais:** servidores lentos são a regra. Use timeout curto, não bloqueie a thread e siga em frente. Ver [falhas em sistemas distribuídos](../03-sistemas-distribuidos/falhas-em-sistemas-distribuidos.md).

## Aplicação no mundo real

- **Ética e legalidade:** respeite o `robots.txt`, os termos de uso, os limites de taxa e as leis de proteção de dados e de direitos autorais. Identifique seu crawler (User-Agent com contato).
- **Ferramentas:** frameworks de crawling (ex.: Scrapy; bibliotecas de navegador headless como Playwright para páginas com JavaScript). Para pequena escala, uma fila (Redis ou banco) + workers resolve.
- **Deduplicação aproximada:** SimHash e MinHash detectam quase-duplicatas (páginas com pequenas diferenças), o que um hash exato não faz.

## No Projeto prático (encurtador de links)

- **Verificador de destinos:** um worker que visita as URLs de destino dos links criados, para validar se existem, detectar redirecionamentos maliciosos e gerar pré-visualização. É um mini-crawler com fila, timeout curto, respeito a limites **por host** e cache de DNS.
- **Exercício:** implementar uma fila por host com atraso entre requisições e medir a vazão com 1 host × 50 hosts.

## Armadilhas comuns

- BFS ingênua que derruba um único site.
- Ignorar o robots.txt.
- DNS sem cache virando gargalo.
- Não salvar o estado e ter que recomeçar do zero após uma falha.
- Sem limite de profundidade ou de tamanho de URL (armadilhas de aranha).

## Perguntas de verificação

1. Por que a fronteira precisa de filas por host?
2. Como se combinam prioridade e educação na fronteira de URLs?
3. Por que usar hash para detectar conteúdo duplicado? Quando ele falha (quase-duplicatas)?
4. Qual o papel do filtro de Bloom no "URL já vista?" e qual o custo de um falso positivo?

## Referências

- [SDI1, cap. 9, "Design a Web Crawler"]: usos, escopo, estimativas, componentes, fluxo, BFS × DFS, fronteira (prioridade, educação, atualização, armazenamento), downloader (robots.txt, desempenho), robustez, extensibilidade, conteúdo problemático.
- [DDIA1, cap. 10, "Building search indexes", pp. 411–412]: construção de índices em lote.
- [DDIA1, cap. 3, "Performance optimizations" (filtros de Bloom), p. 79].
