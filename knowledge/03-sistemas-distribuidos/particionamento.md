---
titulo: Particionamento (sharding) e hashing consistente
fontes: [DDIA1, SDI1]
status-validacao: base primária única (DDIA1) + secundária (SDI1); contém divergência registrada entre as fontes
atualizado: 2026-09-24
---

# Particionamento (sharding) e hashing consistente

> Quando os dados ou a vazão não cabem em uma máquina, dividimos o conjunto em **partições** (shards): cada registro pertence a exatamente uma, e cada nó cuida de algumas. O objetivo é dividir dados e carga **por igual**. O inimigo é o **ponto quente** (*hot spot*). Quase sempre o particionamento vem combinado com replicação: cada partição tem cópias em vários nós.

## Por que importa

O particionamento é o que permite escalar escrita e volume de dados além de uma máquina. A escolha da chave de partição é uma das decisões mais difíceis de reverter em um sistema distribuído: define quais consultas são baratas, onde surgem pontos quentes e como o cluster cresce.

## Conceitos-chave

### Vocabulário

- **Nomes para a mesma ideia:** partição, *shard* (MongoDB, Elasticsearch), *region* (HBase), *tablet* (Bigtable), *vnode* (Cassandra, Riak), *vBucket* (Couchbase).
- **Não confundir** com *partição de rede*, que é uma falha de comunicação entre nós.
- **Distribuição desigual** (*skew*) torna o particionamento inútil. No extremo, 1 de 10 nós recebe toda a carga.

### Particionar dados chave-valor

**Por faixa de chave** (como volumes de enciclopédia)
- Cada partição cobre um intervalo contínuo de chaves, com limites ajustados à distribuição real dos dados.
- **Forças:** chaves ordenadas dentro da partição, então consultas por faixa são eficientes (ex.: todas as leituras de um mês).
- **Fraqueza:** padrões de acesso sequenciais criam pontos quentes. Chave = timestamp faz todas as escritas de "hoje" irem para a mesma partição. A mitigação é prefixar com outra dimensão (ex.: sensor + timestamp), ao custo de uma consulta por sensor.
- Exemplos: Bigtable, HBase, MongoDB (modo por faixa).

**Por hash da chave**
- Uma função hash espalha chaves parecidas de forma uniforme, e cada partição cobre uma faixa de valores de hash.
- A função não precisa ser criptográfica, mas precisa ser **estável entre processos**. O `hashCode` de algumas linguagens muda entre processos e não serve.
- **Custo:** perde a ordenação, então consultas por faixa viram consultas a todas as partições.
- **Meio-termo (chave composta, estilo Cassandra):** a primeira parte da chave é hasheada e define a partição; as demais ordenam os dados dentro dela. Com `(user_id, timestamp)`, busca-se eficientemente as postagens de um usuário em um intervalo de tempo.

**Pontos quentes que o hash não resolve**
- Se todo mundo acessa **a mesma chave** (ex.: a conta de uma celebridade), o hash não ajuda: a chave cai sempre na mesma partição.
- **Mitigação na aplicação:** adicionar um sufixo aleatório (ex.: 00–99) à chave quente para espalhar as escritas por 100 chaves. As leituras passam a juntar as 100, e é preciso registrar quais chaves foram divididas.

### Particionamento e índices secundários

| Tipo | Como funciona | Escrita | Leitura |
|---|---|---|---|
| **Por documento (índice local)** | Cada partição indexa só os próprios documentos | Toca uma partição | **Scatter/gather:** consulta todas as partições, sujeita a amplificação de cauda |
| **Por termo (índice global)** | O índice é particionado pelo valor indexado (ex.: `cor:vermelho`), por termo ou por hash do termo | Um documento pode tocar várias partições do índice; costuma ser **assíncrono** | Vai direto à partição do termo |

- MongoDB, Cassandra, Elasticsearch e outros usam índices locais. Índices secundários globais (ex.: os do DynamoDB) costumam ser atualizados de forma assíncrona (panorama de 2017).
- Implementar índice secundário "à mão" em um banco chave-valor exige cuidado com condições de corrida e escritas parciais.

### Rebalanceamento

- **Quando é necessário:** a vazão cresce (mais CPU), os dados crescem (mais disco e RAM) ou um nó cai. Mover partições entre nós é rebalancear.
- **Requisitos:** a carga fica justa no fim; o sistema continua aceitando leitura e escrita durante o processo; move-se o mínimo de dados.
- **O que não fazer: `hash(chave) mod N`.** Mudar N remapeia quase todas as chaves. Em um cache, isso vira uma tempestade de *cache misses*.

**Estratégias**

| Estratégia | Como funciona | Observações |
|---|---|---|
| **Número fixo de partições** | Criar muito mais partições que nós (ex.: 1.000 para 10 nós); um nó novo "rouba" partições inteiras dos outros | Simples; o número inicial vira o teto de nós; partições crescem com os dados. Usado por Riak, Elasticsearch, Couchbase. |
| **Particionamento dinâmico** | A partição se divide ao passar de um tamanho (ex.: 10 GB no HBase) e se funde ao encolher | Adapta-se ao volume; um banco vazio começa com uma partição só (mitigação: *pre-splitting*) |
| **Proporcional aos nós** | Número fixo de partições por nó; um nó novo divide partições aleatórias e fica com metade de cada | Tamanho de partição estável; exige hash. É o que mais se aproxima da definição original de hashing consistente. Usado por Cassandra (256 por nó, panorama de 2017). |

- **Automático × manual:** o rebalanceamento automático é conveniente, mas imprevisível. Combinado com detecção automática de falhas, pode causar **falha em cascata**: um nó lento é dado como morto, a carga é redistribuída e todos ficam mais lentos. Ter um humano no processo evita surpresas.

### Hashing consistente (anel de hash)

- **Ideia:** mapear servidores e chaves no mesmo espaço de hash, visto como um **anel**. Cada chave pertence ao primeiro servidor encontrado no sentido horário.
- Ao **adicionar** um servidor, só as chaves entre ele e o anterior mudam de dono. Ao **remover**, as chaves dele vão para o próximo. Em média, apenas k/n chaves são remapeadas, contra quase todas no `mod N`.
- **Dois problemas do anel básico:** partições de tamanhos muito diferentes, e distribuição desigual de chaves.
- **Nós virtuais:** cada servidor aparece em várias posições do anel. Com centenas de nós virtuais por servidor, a distribuição fica equilibrada: o SDI1 cita desvio padrão de ~5–10% da média com 100–200 nós virtuais. O custo é guardar mais metadados. Nós virtuais também permitem dar mais carga a máquinas mais fortes.
- **Usos citados:** particionamento do Dynamo e do Cassandra, CDNs (Akamai), balanceadores (Maglev), e o Discord.

### Roteamento de requisições

- É um caso de **descoberta de serviço**. Três arquiteturas:
  1. o cliente fala com qualquer nó, que encaminha se não for o dono;
  2. uma **camada de roteamento** ciente das partições;
  3. o **cliente** conhece o mapa de partições.
- **Todos precisam concordar sobre o mapa** partição → nó. Soluções:
  - serviço de coordenação (ex.: **ZooKeeper**, usado por HBase e Kafka na época; o Kafka migrou depois para o KRaft, confira na documentação atual);
  - servidores de configuração próprios (MongoDB);
  - **gossip** entre nós (Cassandra, Riak).
- **Endereços IP** de nós mudam pouco, e DNS costuma bastar.
- **Consultas paralelas (MPP):** data warehouses dividem consultas complexas em estágios executados em paralelo em várias partições.

## Divergências entre as fontes

- **Hashing consistente é uma boa técnica para bancos?**
  - O **DDIA1** diz que o termo é confuso ("consistente" aqui não tem relação com consistência de réplicas nem com ACID). Diz também que a versão com limites aleatórios "não funciona muito bem para bancos" e que a documentação de muitos produtos usa o termo de forma imprecisa. A recomendação é chamar de *particionamento por hash* e escolher uma estratégia de rebalanceamento explícita.
  - O **SDI1** apresenta o anel com nós virtuais como técnica padrão e amplamente usada (Dynamo, Cassandra).
  - **Leitura conciliada:** as duas fontes descrevem o mesmo mecanismo (vnodes ≈ "partições proporcionais aos nós"). A divergência está na ênfase: o SDI1 foca em **caches e redistribuição mínima**; o DDIA1 alerta para os **limites em bancos com estado** e para a imprecisão do termo. Para um cache distribuído, o anel é ótimo. Para um banco, avalie a estratégia de rebalanceamento concreta que o produto usa.
- **Hashing mitiga pontos quentes?** O SDI1 diz que ajuda a espalhar dados de chaves populares. O DDIA1 lembra que o hash **não resolve** uma chave única muito quente: é preciso dividir a chave na aplicação. As duas coisas são verdade em escalas diferentes: muitas chaves populares × uma chave extremamente popular.

## Trade-offs e decisões

- **Faixa × hash:** consultas por faixa baratas contra distribuição uniforme.
- **Índice local × global:** escrita simples com leitura cara (scatter/gather) contra leitura direta com escrita complexa e índice eventualmente consistente.
- **Número fixo × dinâmico × proporcional aos nós:** simplicidade contra adaptação ao volume.
- **Rebalanceamento automático × manual:** menos trabalho operacional contra risco de cascata.

## Aplicação no mundo real

- **Adie o particionamento enquanto puder.** Réplicas de leitura, cache e máquinas maiores resolvem muita coisa antes de exigir sharding. Quando precisar, prefira bancos com particionamento nativo (ex.: Citus para PostgreSQL, Vitess para MySQL, CockroachDB, Cassandra, DynamoDB) a sharding manual na aplicação. Confirme as capacidades na documentação oficial.
- **Escolha a chave de partição pelas consultas mais frequentes:** ela deve distribuir a escrita e manter juntas as leituras que andam juntas (ex.: `tenant_id` em SaaS multi-inquilino).
- **Detecte chaves quentes:** métricas por partição e por chave (top-N) antes que um cliente grande derrube um shard.
- **Caches distribuídos** (ex.: clientes de Memcached e Redis Cluster) usam hashing consistente ou *hash slots* fixos para que adicionar um nó não invalide o cache inteiro.

## No Projeto prático (encurtador de links)

- **Chave natural:** o código curto. Particionar pelo **hash do código** distribui bem redirecionamentos e criações.
- **Consultas que sofrem:** "todos os links do usuário X" vira scatter/gather se o particionamento for pelo código. Opções: um índice global por dono, ou aceitar o scatter/gather para essa consulta rara.
- **Ponto quente:** um link viral concentra os redirecionamentos em uma partição. Resposta natural: cache e CDN na frente, não reparticionar.
- **Cliques:** particionar eventos por (código, dia) facilita agregações por link e período.
- **Experimento sugerido:** simular a troca de 3 para 4 nós de cache com `mod N` e com anel de hash, e medir a porcentagem de chaves remapeadas (a teoria prevê ~75% × ~25%).

## Armadilhas comuns

- Usar `hash mod N` para distribuir chaves entre nós que mudam.
- Timestamp como início da chave de partição em dados de série temporal.
- Esquecer que índices secundários precisam ser particionados também.
- Rebalanceamento automático sem limite de banda.
- Particionar cedo demais.

## Perguntas de verificação

1. Por que chave = timestamp cria ponto quente no particionamento por faixa? Como mitigar?
2. O que se perde ao particionar por hash? Como a chave composta recupera parte disso?
3. Compare índice secundário local e global em custo de escrita e de leitura.
4. Por que `mod N` é ruim para rebalancear? Quantas chaves se movem em média com hashing consistente?
5. Para que servem os nós virtuais?
6. Como uma camada de roteamento descobre que uma partição mudou de nó?

## Referências

- [DDIA1, cap. 6, "Partitioning and Replication", pp. 200–201].
- [DDIA1, cap. 6, "Partitioning of Key-Value Data", pp. 201–205]: faixa de chave, hash, crítica ao termo "consistent hashing", chaves compostas, cargas enviesadas.
- [DDIA1, cap. 6, "Partitioning and Secondary Indexes", pp. 206–209]: índices por documento e por termo.
- [DDIA1, cap. 6, "Rebalancing Partitions", pp. 209–214]: `mod N`, número fixo, dinâmico, proporcional aos nós, automático × manual.
- [DDIA1, cap. 6, "Request Routing", pp. 214–216]: descoberta de serviço, ZooKeeper, gossip, consultas paralelas.
- [SDI1, cap. 5, "Design Consistent Hashing"]: problema do rehash, anel de hash, adição e remoção de servidores, nós virtuais, chaves afetadas, usos reais.
