---
titulo: Replicação
fontes: [DDIA1, SDI1]
status-validacao: base primária única (DDIA1) + secundária (SDI1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Replicação

> Replicar é manter cópias dos mesmos dados em várias máquinas. Serve para ficar perto do usuário (latência), continuar funcionando quando algo cai (disponibilidade) e distribuir leituras (vazão). Copiar dados estáticos é trivial. Toda a dificuldade está em **propagar mudanças**, e há três famílias de solução: **líder único**, **multilíder** e **sem líder**.

## Por que importa

Quase todo sistema com usuários reais roda com pelo menos uma réplica: para backup quente, para leitura ou para failover. Cada escolha (síncrono ou assíncrono, ler do líder ou das réplicas, failover automático ou manual) muda o que o usuário pode ver e o que pode se perder.

## Conceitos-chave

### Replicação com líder único (líder–seguidor)

- Um nó é o **líder** e recebe todas as escritas. Os **seguidores** recebem um log de mudanças e aplicam na mesma ordem. Leituras podem ir a qualquer réplica.
- É usada em PostgreSQL, MySQL, SQL Server, MongoDB, e também em brokers como Kafka e filas replicadas do RabbitMQ.

**Síncrono × assíncrono**
- **Síncrono:** o líder espera a confirmação do seguidor. Garante a cópia, mas se o seguidor travar, as escritas travam.
- **Semissíncrono:** um seguidor síncrono e os demais assíncronos. Garante os dados em pelo menos dois nós.
- **Totalmente assíncrono:** é comum. Se o líder morrer, **escritas já confirmadas ao cliente podem se perder**. Em troca, o líder segue escrevendo mesmo com seguidores atrasados.

**Novo seguidor sem parar o sistema**
1. Tirar um snapshot consistente do líder, associado a uma posição exata no log (LSN no PostgreSQL, coordenadas do binlog no MySQL).
2. Copiar o snapshot para o novo nó.
3. Aplicar as mudanças posteriores até alcançar o líder (*caught up*).

**Falha de seguidor:** ele sabe a última transação que aplicou e pede ao líder o que perdeu.

**Falha do líder (failover)**
1. Detectar a falha, geralmente por timeout.
2. Escolher o novo líder, de preferência o seguidor mais atualizado. É um problema de consenso.
3. Reconfigurar clientes e demais nós, e garantir que o líder antigo, ao voltar, aceite ser seguidor.

**O que pode dar errado no failover**
- Escritas não replicadas do líder antigo são descartadas.
- Descartar dados é pior quando **outros sistemas dependem deles**. Caso real citado: um seguidor desatualizado promovido a líder reutilizou chaves primárias autoincrementais que também existiam em um cache externo, e dados privados foram expostos a usuários errados.
- **Split brain:** dois nós se acham líderes. Mecanismos de *fencing* (STONITH) derrubam um deles, mas, mal projetados, derrubam os dois.
- **Timeout:** longo demais atrasa a recuperação; curto demais gera failovers desnecessários sob carga, piorando o problema. Por isso algumas equipes preferem failover manual.

**Como o log de replicação é implementado**
- **Por comando (statement):** reenviar o SQL. Quebra com funções não determinísticas (`NOW()`, `RAND()`), autoincremento, dependência de ordem e efeitos colaterais.
- **Envio do WAL:** replica bytes do armazenamento. É acoplado à versão do motor, então atualizar a versão sem downtime fica difícil.
- **Lógico (por linha):** descreve inserções, atualizações e exclusões por linha. É desacoplado do motor e fácil de consumir externamente. É a base do **CDC** (captura de mudanças).
- **Por trigger:** a aplicação registra mudanças em uma tabela. É flexível (replicar só parte, entre bancos diferentes), mas tem mais custo e mais bugs.

### Atraso de replicação (replication lag)

- Escalar leitura com muitos seguidores só é viável com replicação **assíncrona**. Então leituras de seguidores podem ser **antigas**: é a *consistência eventual*.
- "Eventual" não tem limite definido. Normalmente são frações de segundo; sob carga ou problema de rede, minutos.
- **Três anomalias e as garantias que as evitam:**

| Anomalia | Garantia | Como obter |
|---|---|---|
| O usuário escreve e, ao recarregar, não vê o que escreveu | **Ler as próprias escritas** (*read-after-write*) | Ler do líder o que o usuário pode ter editado; ler do líder por um tempo após a escrita; o cliente guarda a posição/timestamp da última escrita e só lê de réplica que já a alcançou. Entre dispositivos, esses metadados precisam ser centralizados. |
| O usuário vê um dado e, na próxima leitura, ele "desaparece" (o tempo volta) | **Leituras monotônicas** | Cada usuário sempre lê da mesma réplica (ex.: hash do ID do usuário). |
| A resposta aparece antes da pergunta (quebra de causalidade) | **Prefixo consistente** | Escritas causalmente ligadas vão para a mesma partição, ou algoritmos rastreiam dependências causais. |

- **Regra de projeto:** pergunte o que acontece se o atraso chegar a minutos. Se a resposta for ruim, garanta algo mais forte. Fingir que a replicação é síncrona quando é assíncrona é receita de problema.

### Replicação multilíder

- Mais de um nó aceita escritas, e cada líder também é seguidor dos outros.
- **Casos de uso:**
  - **vários datacenters**, com um líder por datacenter. Menor latência de escrita, tolera a queda de um datacenter e problemas no link entre eles;
  - **clientes offline**, onde cada dispositivo é um "datacenter" (ex.: calendário que sincroniza);
  - **edição colaborativa**.
- **O grande problema são os conflitos de escrita.** Duas edições concorrentes do mesmo registro em líderes diferentes só são detectadas depois, de forma assíncrona.
- **Estratégias:**
  - **evitar conflito:** rotear todas as escritas de um registro para o mesmo líder, como um datacenter "casa" por usuário. Quebra quando é preciso mudar o líder;
  - **convergir:** todas as réplicas precisam chegar ao mesmo valor final:
    - **last write wins (LWW)**, que perde dados;
    - prioridade por réplica, que também perde;
    - mesclar valores;
    - guardar o conflito para resolver depois (na aplicação ou com o usuário);
  - **lógica customizada:** na escrita (rápida, em segundo plano) ou na leitura (devolve as versões conflitantes). A resolução costuma ser por registro, não por transação;
  - **resolução automática:** **CRDTs** (estruturas de dados que se mesclam sozinhas), estruturas persistentes mescláveis (fusão de três vias, como no Git) e **transformação operacional** (base do Google Docs).
- **Nem todo conflito é óbvio.** Duas reservas da mesma sala no mesmo horário, feitas em líderes diferentes, são um conflito mesmo que cada líder tenha checado a disponibilidade.
- **Topologias:** circular, estrela ou todos-para-todos. Circular e estrela têm ponto único de falha. Todos-para-todos pode entregar mensagens fora de ordem (uma atualização chega antes da inserção). Timestamps não bastam para ordenar; é preciso **vetores de versão**.
- É frequentemente considerada **território perigoso**: autoincremento, triggers e constraints interagem mal. Teste as garantias que o seu sistema diz oferecer.

### Replicação sem líder (estilo Dynamo)

- Qualquer réplica aceita escrita. O cliente (ou um coordenador) envia para várias réplicas em paralelo, e **não há failover**. Exemplos: Riak, Cassandra, Voldemort. O DynamoDB da AWS, apesar do nome, usava outra arquitetura (panorama de 2017).
- **Réplica que volta:** é atualizada por **read repair** (o cliente detecta o valor antigo ao ler de várias réplicas e o corrige) e por um processo de **anti-entropia** em segundo plano. Sem anti-entropia, valores raramente lidos ficam menos duráveis.
- **Quóruns:** com *n* réplicas, a escrita precisa de *w* confirmações e a leitura consulta *r* nós. Se **w + r > n**, pelo menos um nó lido tem o valor mais recente.
  - Configuração típica: n ímpar (3 ou 5) e w = r = maioria.
  - n=3, w=2, r=2 tolera 1 nó fora; n=5, w=3, r=3 tolera 2.
  - w = n e r = 1 acelera leituras, mas qualquer nó fora bloqueia as escritas.
- **Limites dos quóruns:** mesmo com w + r > n, é possível ler dado antigo:
  - com quórum frouxo (*sloppy*);
  - com escritas concorrentes resolvidas por LWW;
  - com escrita e leitura simultâneas;
  - com escrita que falhou em parte e não foi desfeita;
  - com restauração a partir de réplica antiga;
  - por azar de timing.
  
  Também **não** garante ler as próprias escritas, leituras monotônicas nem prefixo consistente. É um ajuste de *probabilidade*, não uma garantia absoluta.
- **Monitorar a defasagem:** com líder, o atraso é a diferença de posição no log. Sem líder, é mais difícil. Ainda assim, "eventual" precisa ser quantificado.
- **Quórum frouxo e hinted handoff:** durante uma partição de rede, aceita escritas em nós fora do conjunto "casa" e as devolve depois. Aumenta a disponibilidade de escrita, mas deixa de ser quórum de verdade: é só garantia de durabilidade.

### Escritas concorrentes

- **Concorrência não é "ao mesmo tempo".** Duas operações são concorrentes se **nenhuma sabe da outra**. A relação que importa é *happens-before*: B depende de A, conhece A ou se baseia em A.
- **LWW** converge, mas descarta escritas confirmadas, inclusive não concorrentes, por causa de desvio de relógio. Só é seguro se cada chave for escrita uma vez e depois ficar imutável (ex.: chave = UUID).
- **Números de versão por chave:** o cliente lê, recebe a versão, mescla os valores e escreve informando a versão lida. O servidor sobrescreve o que é anterior e mantém os **irmãos** (*siblings*) concorrentes.
- **Mesclar irmãos** é tarefa da aplicação. Para um carrinho, a união funciona para adições, mas remoções exigem **tombstones**, senão itens removidos reaparecem. CRDTs automatizam isso.
- **Vetores de versão:** um contador por réplica e por chave. Permitem distinguir sobrescrita de escrita concorrente e ler de uma réplica e escrever em outra sem perder dados.

## Trade-offs e decisões

| Abordagem | Quando usar | Custo |
|---|---|---|
| Líder único, assíncrono | Maioria dos sistemas; escalar leitura | Leitura antiga em seguidores; perda de escritas no failover |
| Líder único, semissíncrono | Durabilidade mais forte com disponibilidade razoável | Latência de escrita maior |
| Multilíder | Vários datacenters; clientes offline; colaboração | Conflitos; configuração traiçoeira |
| Sem líder (quóruns) | Alta disponibilidade, baixa latência, tolera leitura antiga | Consistência fraca; conflitos; difícil de monitorar |

- **Failover automático × manual:** rapidez contra risco de failovers desnecessários e split brain.
- **Ler de réplicas:** escala leitura, mas exige tratar as anomalias de atraso onde elas importam.

## Aplicação no mundo real

- **Réplicas de leitura gerenciadas** (ex.: réplicas do Amazon RDS, Cloud SQL, ou replicação por streaming do PostgreSQL). Monitore a métrica de atraso de replicação e alerte quando passar de um limite.
- **Ler as próprias escritas na prática:** a camada de acesso a dados envia ao primário as leituras logo após uma escrita do mesmo usuário, ou todas as leituras de "meus dados".
- **Failover gerenciado:** Patroni, Multi-AZ do RDS e similares automatizam o failover. Ainda assim, teste o failover periodicamente e saiba quanto dado pode se perder (RPO) e quanto tempo leva (RTO).
- **CDC com replicação lógica** (ex.: Debezium lendo o WAL do PostgreSQL) alimenta caches, índices de busca e data warehouses sem gravação dupla.
- **CRDTs em produtos:** editores colaborativos e apps local-first usam bibliotecas CRDT (ex.: Yjs, Automerge). Confirme as capacidades na documentação oficial.
- *O SDI1 (cap. 1) introduz a mesma ideia de forma simplificada: um banco principal para escritas e réplicas para leituras, com promoção de réplica quando o principal cai.*

## No Projeto prático (encurtador de links)

- **Réplica de leitura para redirecionamentos:** a leitura é muito mais frequente que a escrita, então é o caso clássico de escalar leitura.
- **Anomalia provável:** o usuário cria um link e o testa imediatamente. Se o redirecionamento ler de uma réplica atrasada, recebe 404. Isso exige ler as próprias escritas (ex.: ler do primário por alguns segundos após a criação, ou fazer fallback ao primário quando der 404).
- **Experimento sugerido:** criar um atraso artificial na réplica e medir quantos redirecionamentos de links recém-criados falham antes e depois da mitigação.
- **Failover:** derrubar o primário em ambiente local e observar quanto tempo o sistema fica sem aceitar criação de links.

## Armadilhas comuns

- Presumir que a réplica está sempre em dia.
- Failover automático com timeout agressivo, que gera failovers sob carga.
- Chaves autoincrementais compartilhadas com sistemas externos durante o failover.
- Usar LWW em dados que não podem se perder.
- Acreditar que quóruns garantem consistência forte.
- Multilíder "porque parece mais disponível", sem estratégia de conflito.

## Perguntas de verificação

1. Por que replicação totalmente síncrona para todos os seguidores é impraticável?
2. Descreva três coisas que podem dar errado em um failover automático.
3. Diferencie ler as próprias escritas, leituras monotônicas e prefixo consistente, com um exemplo de cada.
4. Com n=5, que valores de w e r toleram 2 nós indisponíveis e ainda satisfazem w + r > n?
5. Por que "concorrente" não significa "ao mesmo tempo"?
6. Em que situação LWW é aceitável?

## Referências

- [DDIA1, cap. 5, "Leaders and Followers", pp. 152–161]: síncrono × assíncrono, novos seguidores, falhas de nó, failover, implementação de logs de replicação.
- [DDIA1, cap. 5, "Problems with Replication Lag", pp. 161–167]: consistência eventual, ler as próprias escritas, leituras monotônicas, prefixo consistente.
- [DDIA1, cap. 5, "Multi-Leader Replication", pp. 168–177]: casos de uso, conflitos, convergência, CRDTs, topologias.
- [DDIA1, cap. 5, "Leaderless Replication", pp. 177–191]: read repair, anti-entropia, quóruns e seus limites, quórum frouxo, detecção de escritas concorrentes, LWW, happens-before, vetores de versão.
- [SDI1, cap. 1, "Scale from Zero to Millions of Users", seção sobre replicação de banco]: visão introdutória de primário e réplicas.
