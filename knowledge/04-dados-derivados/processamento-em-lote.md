---
titulo: Processamento em lote (batch)
fontes: [DDIA1]
status-validacao: base DDIA1 (Lente); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Processamento em lote (batch)

> Um job em lote lê uma **entrada limitada e imutável**, processa tudo e produz uma **saída derivada**, sem efeitos colaterais. O desempenho se mede por **vazão**, não por latência. A filosofia vem do Unix (ferramentas pequenas, composição, entrada intocada) e escala para clusters com MapReduce e seus sucessores (Spark, Flink). A saída mais valiosa quase nunca é um relatório: são **índices, bancos somente leitura e modelos**.

## Por que importa

Muitos problemas não precisam de resposta em milissegundos: agregações diárias, recomputar métricas, reconstruir um índice de busca, treinar um modelo, corrigir dados após um bug. Tratar isso como lote dá **reprocessabilidade**: se o código estava errado, corrija e rode de novo. Um banco que foi alterado por código com bug não tem essa propriedade.

## Conceitos-chave

### Três tipos de sistema

| Tipo | Entrada | Métrica principal |
|---|---|---|
| **Serviço (online)** | Requisição de um usuário esperando | Tempo de resposta, disponibilidade |
| **Lote (offline)** | Conjunto de dados **limitado** | Vazão |
| **Fluxo (quase em tempo real)** | Eventos **ilimitados**, logo após acontecerem | Latência baixa sobre dados contínuos (ver [processamento de fluxo](processamento-de-fluxo.md)) |

### A filosofia Unix aplicada a dados

- **Exemplo:** um pipeline `cat | awk | sort | uniq -c | sort -rn | head` encontra as páginas mais acessadas em gigabytes de log em segundos.
- **Ordenar × agregar em memória:**
  - uma tabela hash em memória funciona se o *conjunto de trabalho* (chaves distintas) couber na RAM;
  - ordenar com derramamento em disco (merge sort, como nas SSTables) escala para além da memória. O `sort` do GNU faz isso e ainda paraleliza.
- **Princípios:**
  - cada programa faz uma coisa bem;
  - a saída de um é entrada do próximo;
  - prototipar cedo e jogar fora o que ficar ruim;
  - automatizar com ferramentas.
- **Interface uniforme:** arquivo e *file descriptor* como sequência de bytes, com texto separado por linhas por convenção. URLs e HTTP são a interface uniforme da web.
- **Separar lógica de ligação:** o programa lê `stdin` e escreve `stdout`, e quem monta o pipeline decide de onde vem e para onde vai (acoplamento fraco).
- **Transparência:** entrada imutável (dá para rodar de novo quantas vezes quiser), inspecionar qualquer etapa e salvar estados intermediários.
- **Limite:** uma máquina só.

### MapReduce e sistemas de arquivos distribuídos

- **HDFS e similares** (GFS; object storage como S3 é análogo): *shared-nothing*. Um NameNode sabe onde está cada bloco, e os blocos são replicados ou protegidos por *erasure coding*. Escala para dezenas de milhares de máquinas.
- **Fluxo de um job:**
  1. ler e quebrar em registros;
  2. **mapper** extrai chave e valor de cada registro, sem guardar estado entre registros;
  3. ordenar por chave (implícito);
  4. **reducer** percorre os valores de cada chave.
- **Distribuição:**
  - um map por bloco de entrada, rodando **perto dos dados** (localidade);
  - saída particionada por hash da chave para os reducers;
  - o **shuffle** particiona, ordena e copia para os reducers;
  - o reducer mescla, mantendo a ordem.
- **Workflows:** jobs encadeados por diretório de saída → entrada, orquestrados por agendadores como Oozie, Luigi e **Airflow**. A saída só vale se o job terminar com sucesso, e saídas parciais são descartadas.

### Joins e agrupamento em lote

- **Em lote, "join" significa resolver todas as associações** de um conjunto de dados (ex.: todos os eventos de todos os usuários), não buscar um registro.
- **Nunca consulte um banco remoto registro a registro** dentro do job: é lento e sobrecarrega o banco, e o resultado fica **não determinístico** porque o banco muda. Traga uma cópia (ETL ou snapshot) para o mesmo armazenamento.
- **Sort-merge join (no reduce):** mappers dos dois lados emitem a chave do join, e o shuffle junta tudo da mesma chave no mesmo reducer. Com *ordenação secundária*, o registro de perfil chega antes dos eventos. É "mandar mensagem" para um endereço, que é a chave. Separa a comunicação de rede da lógica e trata falhas de forma transparente.
- **GROUP BY e sessionização:** mesma mecânica. Exemplo: juntar os eventos de uma sessão para testes A/B.
- **Chaves quentes (skew):**
  - amostrar para detectar as quentes e espalhar os registros delas por vários reducers, replicando o outro lado do join;
  - ou agregar em **dois estágios** (parcial aleatório, depois combinação).
- **Joins no lado do map** (sem shuffle), quando dá para assumir algo sobre a entrada:
  - **broadcast hash join:** o lado pequeno cabe em memória e é carregado inteiro em cada mapper;
  - **hash join particionado:** os dois lados estão particionados igualmente;
  - **merge join no map:** os dois lados já estão particionados **e** ordenados.
  
  A escolha depende de metadados de particionamento (ex.: Hive metastore).

### O que um job em lote produz

- **Índices de busca:** a aplicação original do MapReduce no Google. Índice particionado por documento, reconstruído inteiro (simples de raciocinar) ou de forma incremental.
- **Bancos chave-valor somente leitura** (recomendações, classificadores, "pessoas que você talvez conheça"). **Não grave** registro a registro em um banco de produção de dentro do job: é lento, derruba o banco e expõe resultados parciais. **Gere os arquivos do banco no próprio job** e carregue em massa, com troca atômica para a versão nova e volta fácil para a anterior.
- **Filosofia das saídas:**
  - entradas imutáveis e saída substituída por inteiro dão **tolerância a erro humano**: com bug, reverta o código e rode de novo, ou volte à saída anterior;
  - retry automático de tarefas é seguro;
  - a mesma entrada serve a vários jobs (inclusive jobs de monitoramento que comparam execuções);
  - lógica separada de ligação.
- **Formatos estruturados** (Avro, Parquet) evitam o parsing de texto do Unix.

### Hadoop × bancos MPP

- **Diversidade de armazenamento:** arquivos aceitam qualquer formato. É o "**data lake**": despejar dados brutos primeiro e modelar depois (*schema-on-read*, "dado bruto é melhor"). O fardo de interpretar passa ao consumidor, o que acelera a coleta.
- **Diversidade de processamento:** SQL (Hive) e também código arbitrário (ML, busca, imagens), tudo no mesmo cluster e sobre os mesmos arquivos.
- **Projetado para falhas frequentes:** o MapReduce refaz *tarefas*, não o job inteiro, e grava muito em disco. O motivo real foi o ambiente do Google, onde jobs de baixa prioridade são **preemptados** (~5% de chance por hora de tarefa, muito mais que falha de hardware). Sem preempção frequente, esse projeto faz menos sentido.

### Além do MapReduce

- **Problemas da materialização completa do estado intermediário:** esperar a etapa anterior inteira (tarefas atrasadas seguram tudo), mappers redundantes e replicação desnecessária de dados temporários.
- **Motores de dataflow** (Spark, Flink, Tez) tratam o workflow inteiro como um job:
  - operadores flexíveis;
  - ordenam só quando precisa;
  - estado intermediário em memória ou disco local;
  - começam assim que a entrada fica pronta;
  - reutilizam processos.
  
  Em geral são muito mais rápidos.
- **Tolerância a falhas sem materializar:** recomputar a partir da linhagem (RDD no Spark) ou de checkpoints (Flink). Exige **operadores determinísticos**. Cuidado com iteração em hash sem ordem, aleatoriedade sem semente, relógio e fontes externas.
- **Grafos em lote (Pregel/BSP):** vértices trocam mensagens em rodadas, com estado em memória entre iterações e checkpoints. Particionar grafos é difícil. **Se o grafo cabe em uma máquina, o processamento local costuma ganhar do distribuído.**
- **APIs de alto nível e declaratividade:** Hive, Spark SQL/DataFrames e Flink trazem otimizadores de custo (escolhem o algoritmo de join), leitura colunar e execução vetorizada, sem perder a capacidade de rodar código arbitrário. **Lote e MPP estão convergindo.**

## Trade-offs e decisões

- **Lote × fluxo:** simplicidade, reprocessabilidade e vazão contra latência.
- **Materializar × recomputar:** durabilidade e recuperação fácil contra velocidade.
- **Join no reduce × no map:** generalidade contra velocidade, que exige conhecer a disposição física dos dados.
- **Cluster distribuído × uma máquina:** para muitos volumes reais, uma máquina com boas ferramentas basta e é muito mais simples.

## Aplicação no mundo real

- **Escala pequena:** scripts com ferramentas Unix, DuckDB ou Polars sobre arquivos Parquet e CSV resolvem análises de gigabytes em uma máquina. Só vá para Spark quando o volume exigir.
- **Orquestração:** Airflow, Dagster e Prefect agendam DAGs de jobs com dependências, retries e histórico. Um cron com scripts idempotentes resolve casos simples.
- **Data lake e lakehouse:** arquivos Parquet em object storage (S3, GCS), com formatos de tabela (ex.: Apache Iceberg, Delta Lake) que acrescentam transações e evolução de esquema. Confirme as capacidades na documentação oficial.
- **Jobs idempotentes e reexecutáveis:** escreva em um local novo por execução (ex.: partição por data) e troque um "ponteiro" no fim. Nunca faça `UPDATE` incremental irreversível.
- **Carga em massa em vez de linha a linha:** `COPY` no PostgreSQL, bulk load em bancos analíticos.

## No Projeto prático (encurtador de links)

- **Agregação diária de cliques:** um job lê os eventos brutos do dia (entrada imutável), calcula cliques por link, país e referência, e grava uma **tabela de agregados nova** para aquele dia. Um bug na agregação se corrige rodando o dia de novo.
- **Join de enriquecimento:** eventos de clique × tabela de links (dono, data de criação). Como a tabela de links cabe em memória, é um broadcast hash join em um script simples.
- **Experimento sugerido:** gerar 10 milhões de cliques sintéticos e comparar (a) agregação com `GROUP BY` no PostgreSQL de produção durante carga de redirecionamentos e (b) job em lote sobre um export em Parquet com DuckDB. Medir o impacto no p99 do redirecionamento.

## Armadilhas comuns

- Chamar um banco ou uma API remota por registro dentro de um job em lote.
- Jobs que alteram dados no lugar e não podem ser reexecutados.
- Ignorar chaves quentes (um reducer segura o job inteiro).
- Operadores não determinísticos em frameworks que recomputam.
- Montar um cluster distribuído para dados que cabem em um laptop.

## Perguntas de verificação

1. Por que um job em lote não deve consultar um banco de produção para cada registro?
2. Descreva as etapas de um sort-merge join no MapReduce.
3. Quando um broadcast hash join é possível?
4. Por que gerar os arquivos de um banco somente leitura dentro do job é melhor do que escrever direto no banco?
5. O que "tolerância a erro humano" significa em processamento em lote?
6. Por que motores de dataflow precisam de operadores determinísticos?

## Referências

- [DDIA1, cap. 10, introdução, pp. 389–391]: serviços × lote × fluxo.
- [DDIA1, cap. 10, "Batch Processing with Unix Tools", pp. 391–397]: análise de log, ordenar × agregar em memória, filosofia Unix.
- [DDIA1, cap. 10, "MapReduce and Distributed Filesystems", pp. 397–419]: HDFS, execução de jobs, workflows, joins no reduce e no map, skew, saídas de lote, comparação com MPP, preempção.
- [DDIA1, cap. 10, "Beyond MapReduce", pp. 419–430]: materialização, motores de dataflow, tolerância a falhas por recomputação, Pregel, APIs de alto nível e declaratividade.
