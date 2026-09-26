---
titulo: Armazenamento e índices
fontes: [DDIA1]
status-validacao: base DDIA1 (Lente); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Armazenamento e índices

> Um banco precisa guardar dados e achá-los de novo. A forma como o motor de armazenamento faz isso define o que é rápido e o que é caro. Há duas escolas para cargas transacionais: **log-estruturada** (LSM-tree), que só acrescenta ao fim, e **atualização no lugar** (B-tree). Cargas analíticas pedem outra coisa: **armazenamento colunar**.

## Por que importa

Você dificilmente vai escrever um motor de armazenamento. Mas vai escolher um, criar índices e ajustar parâmetros. Entender o que acontece por baixo permite prever o efeito de cada escolha na leitura, na escrita, no disco e na latência de cauda.

## Conceitos-chave

### Log e índice

- **Log** é uma sequência de registros em que só se acrescenta ao fim (append-only). Acrescentar é a escrita mais barata que existe.
- **Índice** é uma estrutura derivada dos dados primários que funciona como uma placa de sinalização para achar registros. É o **trade-off central do armazenamento**: índices bem escolhidos aceleram leituras, e todo índice deixa as escritas mais lentas. Por isso o banco não indexa tudo: quem conhece as consultas escolhe os índices.

### Índice hash sobre log (estilo Bitcask)

- Um mapa em memória liga cada chave à posição (offset) do valor no arquivo de log. A leitura custa um acesso ao disco.
- Funciona bem com **muitas atualizações e poucas chaves distintas**: todas as chaves precisam caber na RAM.
- **Segmentos e compactação:** o log é dividido em segmentos imutáveis. Em segundo plano, a compactação descarta versões antigas de cada chave e junta segmentos.
- **Detalhes de implementação:**
  - formato binário com tamanho prefixado;
  - *tombstone* (marcador de exclusão) para apagar;
  - snapshot do mapa em disco para reiniciar rápido;
  - checksum para detectar registros escritos pela metade;
  - um único escritor.
- **Limites:** as chaves precisam caber em memória, e consultas por intervalo são ineficientes.

### SSTables e LSM-trees

- **SSTable** (*Sorted String Table*) é um segmento ordenado por chave. Juntar segmentos vira um merge ordenado eficiente, mesmo maior que a memória. O índice em memória pode ser **esparso** (uma chave a cada poucos KB), e blocos podem ser comprimidos.
- **Como se mantém a ordem:**
  1. as escritas vão para uma árvore balanceada em memória (a *memtable*);
  2. quando ela passa de alguns MB, é gravada como SSTable;
  3. a leitura consulta memtable → segmento mais novo → mais antigos;
  4. um **log de escrita separado** (WAL) recupera a memtable após queda.
- **LSM-tree** (*Log-Structured Merge-Tree*) é o nome da família: LevelDB, RocksDB, Cassandra, HBase. O dicionário de termos do Lucene (Elasticsearch/Solr) usa a mesma ideia.
- **Otimizações:**
  - **filtros de Bloom** evitam varrer todos os segmentos para chaves inexistentes;
  - a **estratégia de compactação** pode ser *size-tiered* (segmentos menores e novos se fundem em maiores) ou *leveled* (faixas de chaves em níveis; mais incremental, usa menos disco).
- **Forças:** escrita sequencial, alta vazão de escrita, consultas por intervalo e funcionamento com dados muito maiores que a memória.

### B-trees

- É o índice mais usado: padrão em quase todo banco relacional desde os anos 1970.
- Divide os dados em **páginas de tamanho fixo** (tradicionalmente 4 KB), organizadas em árvore. Cada página cobre uma faixa de chaves e aponta para páginas filhas. A página folha contém os valores ou referências a eles.
- **Fator de ramificação** de centenas de filhos por página. Com isso, 3 ou 4 níveis bastam para a maioria dos bancos: uma árvore de 4 níveis com páginas de 4 KB e fator 500 guarda até ~256 TB.
- **Escrita** sobrescreve a página no lugar. Se a página enche, ela se divide em duas e o pai é atualizado. A árvore se mantém balanceada, com profundidade O(log n).
- **Confiabilidade:**
  - sobrescrever várias páginas não é atômico, por isso existe um **write-ahead log (WAL)**, em que toda modificação é registrada antes de ser aplicada;
  - a concorrência é protegida por *latches* (travas leves).
- **Otimizações:**
  - *copy-on-write*: a página nova vai para outro lugar, como no LMDB, o que também ajuda no isolamento;
  - chaves abreviadas nas páginas internas (B+ tree);
  - folhas dispostas em sequência no disco e ligadas às irmãs para varredura.

### LSM × B-tree

| Aspecto | LSM-tree | B-tree |
|---|---|---|
| Escrita | Geralmente mais rápida; sequencial | Escreve pelo menos duas vezes (WAL + página), página inteira por vez |
| Leitura | Pode consultar vários segmentos | Geralmente mais rápida; a chave está em um só lugar |
| Amplificação de escrita | Existe (compactações repetidas), depende da configuração | Existe (WAL, página inteira, divisões) |
| Espaço em disco | Menor: comprime melhor, sem fragmentação | Deixa espaço livre nas páginas (fragmentação) |
| Previsibilidade | Compactação pode afetar os **percentis altos** de latência | Mais previsível |
| Risco operacional | Compactação que não acompanha a escrita faz o disco encher e a leitura piorar; precisa de monitoramento explícito | Maduro e bem compreendido |
| Transações | Várias cópias da chave em segmentos diferentes | Travas de faixa de chave presas à árvore; bom para isolamento forte |

- **Regra de ouro do livro:** benchmarks genéricos são inconclusivos. Teste com a **sua** carga.
- **Amplificação de escrita** é quando uma escrita lógica gera várias escritas físicas ao longo do tempo. Importa em SSD (desgaste) e quando o gargalo é a banda de disco.

### Outros índices

- **Primário × secundário.** O índice secundário tem chaves não únicas. Ou o valor vira uma lista de IDs, ou o ID é anexado à chave.
- **Onde fica a linha:**
  - **heap file**: o índice aponta para um arquivo de linhas sem ordem, sem duplicar dados;
  - **índice clusterizado**: a linha fica dentro do índice (ex.: chave primária no InnoDB);
  - **índice de cobertura** (*included columns*): algumas colunas ficam no índice, e a consulta se resolve só com ele.
  
  Duplicar dados acelera a leitura, mas custa espaço e escrita.
- **Índice concatenado (multicoluna):** ordena por (coluna A, coluna B). Serve para "A" e para "A+B", nunca para "só B". É a lógica da lista telefônica.
- **Índices multidimensionais:** consultas por faixa em duas ou mais dimensões ao mesmo tempo (latitude e longitude; data e temperatura) pedem **R-trees** (ex.: PostGIS) ou curvas de preenchimento de espaço.
- **Busca textual e difusa:** sinônimos, variações gramaticais, proximidade e distância de edição. O Lucene usa um autômato sobre as chaves (parecido com uma *trie*) que permite buscar palavras a até N edições.

### Tudo em memória

- A vantagem de bancos em memória **não** vem de não ler do disco (o cache do sistema operacional já evita isso). Vem de **não precisar codificar estruturas para o formato de disco**.
- **Durabilidade em memória:** log em disco, snapshots, replicação ou hardware especial. O Redis grava em disco de forma assíncrona, o que dá durabilidade fraca (panorama de 2017). O Memcached é só cache.
- Também permitem estruturas difíceis de ter em disco (filas de prioridade, conjuntos).
- **Anti-caching:** despeja registros pouco usados para o disco, com granularidade melhor que a da memória virtual do sistema operacional.

### OLTP × OLAP

| Propriedade | OLTP (transacional) | OLAP (analítico) |
|---|---|---|
| Leitura típica | Poucos registros, por chave | Agregação sobre milhões de registros |
| Escrita típica | Aleatória, baixa latência, entrada do usuário | Carga em massa (ETL) ou fluxo de eventos |
| Usuário | Cliente final, via aplicação | Analista interno |
| O que os dados representam | Estado atual | Histórico de eventos |
| Gargalo | Tempo de busca no disco | Banda de disco |

- **Data warehouse:** banco separado, somente leitura, alimentado por **ETL** (extrair, transformar, carregar) a partir dos sistemas OLTP. Assim a análise não derruba a operação.
- **Esquema estrela:** uma **tabela de fatos** (um evento por linha, muito larga e enorme) cercada de **tabelas de dimensão** (quem, o quê, onde, quando, como, por quê). O **floco de neve** normaliza mais as dimensões. A estrela é mais simples para analistas.

### Armazenamento colunar

- Consultas analíticas leem poucas colunas de tabelas com centenas de colunas. Guardar **cada coluna separada** permite ler só o necessário. A linha é reconstruída pela posição (o k-ésimo item de cada coluna).
- **Compressão:** colunas repetitivas comprimem muito. **Bitmaps** por valor distinto, com *run-length encoding*, tornam filtros `IN`/`AND`/`OR` operações bit a bit.
- **Processamento vetorizado:** blocos comprimidos cabem no cache L1 da CPU e são processados em laços apertados, com instruções SIMD.
- **Ordenação:** a ordem vale para a linha inteira, não para colunas isoladas. A primeira chave de ordenação acelera filtros por faixa (ex.: data) e comprime melhor. Guardar **cópias em ordens diferentes** (ideia do C-Store/Vertica) aproveita a redundância da replicação.
- **Escrita colunar:** não dá para atualizar no lugar. Usa-se a estratégia LSM: escritas vão para a memória e depois são fundidas em arquivos novos.
- **Atenção:** "famílias de colunas" no Cassandra e no HBase **não** são armazenamento colunar. Elas guardam a linha junta.
- **Agregados materializados:** a *view materializada* é uma cópia desnormalizada do resultado de uma consulta, que precisa ser atualizada quando os dados mudam. O **cubo OLAP** é uma grade de agregados por dimensões: muito rápido para o que foi pré-calculado e inflexível para o resto. Mantenha os dados brutos.

## Trade-offs e decisões

- **Mais índices × escrita mais lenta:** indexe pelas consultas reais, não por hipótese.
- **LSM × B-tree:** carga de escrita alta e dados muito maiores que a memória favorecem LSM. Latência de leitura previsível e transações fortes favorecem B-tree. Na dúvida, meça.
- **Índices clusterizados ou de cobertura × custo de escrita e espaço.**
- **OLTP e OLAP no mesmo banco:** simples no começo, arriscado quando a análise compete com a operação.

## Aplicação no mundo real

- **PostgreSQL e MySQL/InnoDB** usam B-tree como índice padrão. **RocksDB** (LSM) está por trás de muitos sistemas (ex.: bancos distribuídos e filas persistentes). **Cassandra** e **ScyllaDB** são LSM. Confirme o motor de cada produto na documentação oficial, porque isso muda com as versões.
- **Diagnóstico de consultas:** `EXPLAIN (ANALYZE)` no PostgreSQL mostra se a consulta usou índice, varredura sequencial ou índice de cobertura (*index-only scan*).
- **Analytics sem data warehouse:** para volumes pequenos, uma réplica de leitura ou um motor colunar embutido (ex.: DuckDB lendo Parquet) evita misturar análise com o banco transacional.
- **Parquet** é o formato colunar aberto mais comum para dados analíticos em arquivos.
- **Monitorar compactação** em motores LSM: segmentos pendentes, uso de disco, stalls de escrita.

## No Projeto prático (encurtador de links)

- **Redirecionamento:** busca por chave primária (código curto) em índice B-tree. Esse é o caminho quente, então meça o p99.
- **Criação:** índice de unicidade no código curto. Avalie o custo de índices adicionais (ex.: por dono) sobre a vazão de escrita.
- **Analytics de cliques:** tabela de eventos que cresce sem parar. Consultas agregadas por link e período competem com o redirecionamento. É o gancho para separar OLTP de OLAP (réplica, tabela agregada ou armazenamento colunar) quando a medição mostrar interferência.
- **Experimento sugerido:** medir inserções por segundo com 0, 1 e 3 índices secundários na tabela de cliques.

## Armadilhas comuns

- Criar índice para toda coluna "por garantia".
- Índice concatenado na ordem errada para a consulta.
- Rodar consultas analíticas pesadas no banco de produção.
- Achar que "família de colunas" é armazenamento colunar.
- Ignorar a latência de cauda causada pela compactação em motores LSM.

## Perguntas de verificação

1. Por que todo índice deixa a escrita mais lenta? Como decidir quais criar?
2. Explique o caminho de uma escrita e de uma leitura em uma LSM-tree.
3. Por que B-trees precisam de WAL?
4. Em que carga você escolheria LSM, e em qual B-tree? Como validaria a escolha?
5. Por que o armazenamento colunar acelera consultas analíticas e dificulta escritas?
6. Um índice em (sobrenome, nome) ajuda a buscar só por nome? Por quê?

## Referências

- [DDIA1, cap. 3, "Data Structures That Power Your Database", pp. 70–90]: log, índices hash, SSTables, LSM-trees, B-trees, comparação, índices secundários, clusterizados e de cobertura, multicoluna, busca difusa, bancos em memória.
- [DDIA1, cap. 3, "Transaction Processing or Analytics?", pp. 90–95]: OLTP × OLAP, data warehouse, ETL, esquemas estrela e floco de neve.
- [DDIA1, cap. 3, "Column-Oriented Storage", pp. 95–103]: armazenamento colunar, compressão com bitmaps, processamento vetorizado, ordenação, escrita colunar, views materializadas e cubos.
