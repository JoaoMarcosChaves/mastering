---
titulo: Consistência, ordenação e consenso (linearizabilidade, CAP, 2PC, Raft)
fontes: [DDIA1, SDI1]
status-validacao: base DDIA1 (Lente) + secundária (SDI1); contém divergência registrada sobre CAP
atualizado: 2026-09-24
---

# Consistência, ordenação e consenso

> Consistência eventual é uma garantia fraca: "um dia converge". Garantias mais fortes facilitam a vida da aplicação, mas custam latência e disponibilidade. Este tema liga três ideias: **linearizabilidade** (parecer que existe uma única cópia), **ordenação e causalidade** (o que aconteceu antes do quê) e **consenso** (todos os nós concordarem de forma irrevogável). O resultado central é que vários problemas práticos (trava, unicidade, eleição de líder, commit distribuído) **são equivalentes a consenso**.

## Por que importa

Toda vez que um sistema precisa de "só um": um líder, um dono de trava, um único dono de um nome de usuário, um único comprador do último ingresso. Nesses casos, ou existe um nó que decide sozinho (ponto único de falha), ou existe consenso. Entender isso evita inventar protocolos quebrados e ajuda a escolher entre consistência forte e disponibilidade.

## Conceitos-chave

### Linearizabilidade

- **Definição:** o sistema se comporta como se houvesse **uma única cópia** dos dados, e cada operação acontece atomicamente em algum instante entre o início e o fim da requisição. É uma **garantia de recência**: assim que alguém lê o valor novo, ninguém depois pode ler o antigo. Também é chamada de consistência atômica, forte ou imediata.
- **Exemplo:** Alice vê o placar final. Bob recarrega *depois* de ouvir Alice e vê "jogo em andamento", porque leu de uma réplica atrasada. Isso **viola** a linearizabilidade.
- **Linearizabilidade × serializabilidade:**
  - **serializabilidade:** isolamento de *transações* com vários objetos; a ordem serial pode diferir da ordem real;
  - **linearizabilidade:** recência de leituras e escritas em *um objeto*; não evita write skew sozinha;
  - as duas juntas formam a **serializabilidade estrita**. 2PL e execução serial costumam ser linearizáveis. **SSI não é**, porque lê de snapshot.
- **Onde ela é necessária:**
  - **travas e eleição de líder**, porque todos precisam concordar sobre o dono. ZooKeeper e etcd existem para isso;
  - **restrições de unicidade rígidas** (nome de usuário, caminho de arquivo, "não vender além do estoque", "não reservar o mesmo assento"). Algumas regras de negócio toleram violação com compensação, como *overbooking*;
  - **dependências entre canais:** um exemplo é um servidor web que grava a foto no armazenamento e avisa o redimensionador por fila. Se o armazenamento não for linearizável, a fila pode chegar antes da réplica e o redimensionador processa a versão antiga ou não encontra nada.
- **Quem consegue ser linearizável:**

| Replicação | Linearizável? |
|---|---|
| Líder único (lendo do líder ou de seguidor síncrono) | Potencialmente, mas há o risco do "líder iludido" e da perda de escritas no failover assíncrono |
| Algoritmos de consenso (ZooKeeper, etcd) | Sim |
| Multilíder | Não |
| Sem líder (quóruns) | Provavelmente não: LWW e quórum frouxo quebram; mesmo quórum estrito tem corridas. Dá para forçar com read repair síncrono, mas não há CAS linearizável sem consenso |

### O custo: CAP, desmontado

- **O trade-off real:** durante uma falha de rede, réplicas desconectadas precisam escolher entre (a) **ser linearizáveis**, recusando ou esperando (indisponíveis), ou (b) **ficar disponíveis**, respondendo de forma não linearizável.
- **Por que "escolha 2 de 3" engana:** partições de rede não são opcionais, elas acontecem. A formulação melhor é "**consistente ou disponível quando particionado**". Com a rede funcionando, dá para ter os dois.
- **CAP formal tem escopo estreito:** um modelo de consistência (linearizabilidade), um tipo de falha (partição de rede) e uma definição peculiar de disponibilidade. Não trata latência, nós mortos nem outros trade-offs. Foi importante historicamente (abriu caminho para o NoSQL), mas **tem pouco valor prático para projetar sistemas**. O DDIA recomenda **evitar os rótulos CP e AP**.
- **O custo que importa todo dia é a latência.** Até a RAM de uma CPU multinúcleo não é linearizável, por desempenho. Está provado que o tempo de resposta linearizável é, no mínimo, proporcional à **incerteza do atraso da rede**. Muitos bancos abrem mão da linearizabilidade **por desempenho**, não por tolerância a falhas.

### Ordenação e causalidade

- **Causalidade impõe ordem:** causa antes do efeito, pergunta antes da resposta, criação antes da atualização. Um sistema **causalmente consistente** respeita isso. O isolamento por snapshot, por exemplo, é consistente com a causalidade.
- **Ordem total × parcial:** a linearizabilidade dá **ordem total** (uma linha do tempo única). A causalidade dá **ordem parcial**: operações concorrentes são incomparáveis, como os ramos do histórico do Git.
- **Linearizabilidade implica causalidade**, mas a **consistência causal** é o modelo mais forte que **não fica lento com atrasos de rede** e **continua disponível em falhas de rede** (o CAP não se aplica). Muitos sistemas que "precisam" de linearizabilidade na verdade só precisam de causalidade.
- **Números de sequência:**
  - com líder único, o contador do log de replicação dá ordem total e causal;
  - **geradores sem coordenação** (pares/ímpares por nó, timestamps físicos, blocos pré-alocados) são rápidos e únicos, mas **não respeitam a causalidade**.
- **Timestamps de Lamport:** um par (contador, ID do nó). Todo nó e todo cliente propagam o **maior contador visto** e avançam o próprio. O resultado é uma ordem total consistente com a causalidade. Ao contrário dos vetores de versão, **não distinguem** concorrente de dependente, mas são compactos.
- **Ordem por timestamp não basta** para decidir *na hora* (ex.: unicidade de nome de usuário): é preciso saber que **ninguém mais vai inserir algo antes** na ordem. Isso leva ao broadcast de ordem total.

### Broadcast de ordem total

- **Propriedades:** entrega confiável (se um nó recebe, todos recebem) e **na mesma ordem** para todos. A ordem é fixada na entrega: não se insere nada retroativamente.
- **Equivale a um log compartilhado:** é a base da **replicação de máquina de estados**, de transações serializáveis determinísticas e de fencing tokens (o `zxid` do ZooKeeper).
- **Dá para construir um no outro:**
  - **CAS linearizável a partir do log:** anexar "quero o nome X", ler o log até ver a própria mensagem e vencer se a sua for a primeira para X. Leituras linearizáveis exigem sequenciar leituras pelo log, esperar a posição mais recente (`sync()` no ZooKeeper) ou ler de réplica síncrona;
  - **log a partir de um registrador linearizável** com incremento atômico, que dá números **sem lacunas** (diferente de Lamport).
- **Resultado profundo:** CAS linearizável, incremento atômico, broadcast de ordem total e **consenso** são **equivalentes**.

### Transações distribuídas e 2PC

- **Commit atômico:** todos os nós confirmam ou todos abortam. Um commit é **irrevogável**. Desfazer depois exige uma transação de **compensação** separada, que é problema da aplicação.
- **Two-phase commit (2PC):**
  1. o coordenador envia *prepare*;
  2. cada participante que vota "sim" **promete** que conseguirá confirmar (já gravou tudo em disco e checou restrições), abrindo mão do direito de abortar;
  3. o coordenador grava a decisão no seu log (**ponto de commit**);
  4. o coordenador envia *commit* ou *abort* e **repete para sempre** até todos confirmarem.
  
  São dois pontos sem volta: o "sim" do participante e a decisão do coordenador.
- **Se o coordenador cai depois do "sim",** os participantes ficam **em dúvida**. Não podem abortar nem confirmar sozinhos e **seguram as travas** até o coordenador voltar. Se o log do coordenador se perde, alguém precisa resolver à mão. As "decisões heurísticas" existem, mas quebram a atomicidade.
- **3PC** exige atraso limitado e um detector de falhas perfeito, então não é prático.
- **Na prática:**
  - **transações internas** de um banco distribuído (mesmo software em todos os nós) funcionam razoavelmente bem;
  - **transações heterogêneas (XA)** entre bancos e brokers diferentes permitem processamento "exatamente uma vez" (ack da mensagem + escrita no banco atômicos), mas: o coordenador vira um banco crítico e costuma não ser replicado; servidores de aplicação deixam de ser sem estado; é o denominador comum (sem detecção de deadlock entre sistemas, sem SSI); e o sistema **amplifica falhas**, porque qualquer participante fora aborta tudo.
  - Há relatos de transações distribuídas no MySQL >10× mais lentas que as locais (panorama de 2017).
- **2PC não é 2PL.** Ver [transações](transacoes.md).

### Consenso tolerante a falhas

- **Propriedades:** acordo uniforme (ninguém decide diferente), integridade (ninguém decide duas vezes), validade (o valor decidido foi proposto por alguém) e **terminação** (quem não caiu decide). As três primeiras são de segurança; a terminação é de vivacidade. **O 2PC não termina** se o coordenador morrer.
- **Precisa de maioria viva** para terminar: 3 nós toleram 1 falha, 5 toleram 2. A segurança se mantém mesmo quando a maioria cai: o sistema para, mas não decide errado.
- **FLP:** consenso é impossível no modelo assíncrono sem relógio. Com timeouts ou aleatoriedade, é solucionável na prática.
- **Algoritmos:** Viewstamped Replication, **Paxos** (Multi-Paxos), **Raft** (etcd) e **Zab** (ZooKeeper). Implementam broadcast de ordem total diretamente. **Não implemente o seu.**
- **O truque da época:** o líder é único *por época* (termo no Raft, ballot no Paxos). Toda decisão exige votos de um quórum, e os quóruns de eleição e de proposta **se sobrepõem**, então um líder antigo descobre que foi deposto. Diferenças para o 2PC: o líder é eleito, basta maioria e existe recuperação definida.
- **Limites:**
  - votar é **replicação síncrona**, o que custa desempenho;
  - exige maioria estrita;
  - membros fixos (membros dinâmicos são menos maduros);
  - depende de timeouts, e em redes instáveis há eleições demais;
  - o Raft tem casos patológicos com um único link ruim.

### Serviços de coordenação (ZooKeeper, etcd, Consul)

- **O que são:** "bancos" pequenos, em memória, replicados por consenso, para dados que **mudam devagar** (quem é líder da partição 7). **Não servem** para estado da aplicação que muda milhares de vezes por segundo.
- **O que oferecem:**
  - operações atômicas linearizáveis (travas e leases);
  - **ordem total** (fencing tokens);
  - **detecção de falhas** por sessão e heartbeat, com nós efêmeros que somem quando a sessão expira;
  - **notificações de mudança** (*watches*).
- **Usos:** eleição de líder, atribuição de partições a nós, **descoberta de serviço** (que em geral não precisa de consenso: o DNS é não linearizável e está tudo bem) e **serviço de membros** (concordar sobre quem está vivo).
- **"Terceirizar" o consenso:** 3 ou 5 nós fazem as votações para milhares de clientes.
- **Um banco com líder único** oferece linearizabilidade sem consenso a cada escrita, mas precisa de consenso para **trocar de líder**. As opções são esperar o líder voltar, failover manual (consenso "por ato divino") ou eleição automática com algoritmo comprovado.

## Divergências entre as fontes

- **Como falar de CAP:**
  - o **SDI1** (cap. 6, projeto de key-value store) usa a classificação **CP / AP / CA** para explicar escolhas de projeto (ex.: um banco que bloqueia escritas em partição × um que aceita e reconcilia depois);
  - o **DDIA1** considera esses rótulos falhos e recomenda evitá-los: "CA" não existe na prática, porque partições acontecem, e "disponibilidade" no CAP formal tem definição peculiar;
  - **Posição da trilha:** usar a pergunta concreta "**durante uma falha de rede, esta operação deve recusar (consistência) ou responder com dado possivelmente antigo (disponibilidade)?**", **por operação**, e medir latência no cotidiano. Os rótulos CP/AP servem como vocabulário de entrevista, não como classificação de sistemas. Isso está alinhado com a diretriz do bootstrap: "CAP em contexto de particionamento real, sem reduzir consistência a uma escolha binária permanente".

## Trade-offs e decisões

- **Linearizável × causal × eventual:** facilidade de raciocínio contra latência e disponibilidade.
- **2PC × alternativas** (log de eventos, sagas com compensação, outbox): atomicidade forte entre sistemas contra acoplamento e amplificação de falhas. Ver [integração de dados e correção](../04-dados-derivados/integracao-de-dados-e-correcao.md).
- **Consenso próprio × serviço de coordenação:** quase sempre use o serviço.

## Aplicação no mundo real

- **Eleição de líder e travas:** etcd (base do Kubernetes), ZooKeeper, Consul. Em Kubernetes, *leases* do próprio cluster fazem eleição de líder para controladores. Sempre com fencing no recurso protegido.
- **Unicidade em um banco com líder único:** o `UNIQUE` do PostgreSQL já é linearizável naquele nó, sem precisar de consenso adicional.
- **Bancos "NewSQL"** (ex.: CockroachDB, Spanner, TiDB, YugabyteDB) usam Raft ou Paxos por faixa de dados para ter transações distribuídas serializáveis. Confirme os detalhes na documentação oficial.
- **Evite XA** entre banco e broker. Prefira o padrão **outbox** (grava o evento na mesma transação do banco, um relay publica) e consumidores **idempotentes**.
- **Testes de consistência:** a ferramenta Jepsen e seus relatórios públicos mostram como bancos reais violam as garantias anunciadas. É leitura recomendada para desconfiar de marketing.

## No Projeto prático (encurtador de links)

- **Unicidade do código curto:** com um PostgreSQL primário, o `UNIQUE` resolve, porque o líder único é o "ditador". Se um dia os códigos forem gerados em vários nós sem banco central, o problema vira consenso ou geração sem colisão (IDs com prefixo de nó, ver [gerador de IDs únicos](../06-estudos-de-caso/gerador-de-ids-unicos.md)).
- **Dependência entre canais:** criar o link (banco) e publicar "link criado" (fila) pode ter a mensagem processada antes da réplica ter o link. Use o padrão outbox e leitura do primário no consumidor.
- **Exercício de raciocínio:** para cada operação (criar, redirecionar, contar clique, listar links), decidir **consistência ou disponibilidade durante uma falha de rede**, e justificar.

## Armadilhas comuns

- Dizer que um sistema "é CP" ou "é AP" como se fosse uma propriedade global.
- Achar que quórum w + r > n dá linearizabilidade.
- Implementar eleição de líder ou trava com timeouts caseiros.
- Usar ZooKeeper ou etcd como banco de dados da aplicação.
- Transação distribuída XA como solução padrão de integração.
- Confundir linearizabilidade com serializabilidade.

## Perguntas de verificação

1. Dê um exemplo de violação de linearizabilidade com dois canais de comunicação.
2. Por que o DDIA considera "escolha 2 de 3" uma formulação enganosa do CAP?
3. Qual é a diferença entre ordem total e ordem parcial? Qual delas a causalidade define?
4. Por que timestamps de Lamport não bastam para garantir unicidade de nome de usuário em tempo real?
5. O que acontece com um participante do 2PC que votou "sim" e perdeu contato com o coordenador?
6. Como os algoritmos de consenso evitam dois líderes decidindo ao mesmo tempo?
7. Cite quatro problemas equivalentes a consenso.

## Referências

- [DDIA1, cap. 9, "Consistency Guarantees", pp. 322–324].
- [DDIA1, cap. 9, "Linearizability", pp. 324–338]: definição, × serializabilidade, usos, implementações, quóruns, custo, CAP, atrasos de rede.
- [DDIA1, cap. 9, "Ordering Guarantees", pp. 339–352]: causalidade, ordem total × parcial, números de sequência, Lamport, broadcast de ordem total.
- [DDIA1, cap. 9, "Distributed Transactions and Consensus", pp. 352–375]: FLP, 2PC, 3PC, XA, travas em dúvida, consenso tolerante a falhas, épocas e quóruns, ZooKeeper e etcd, descoberta de serviço, serviços de membros.
- [SDI1, cap. 6, "Design a Key-Value Store", seção sobre o teorema CAP]: classificação CP/AP/CA, usada como contraponto na divergência acima.
