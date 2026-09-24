---
titulo: Processamento de fluxo, filas e logs de eventos
fontes: [DDIA1, SDI1]
status-validacao: base primária única (DDIA1) + secundária (SDI1); aguarda convergência com ≥2 primárias
atualizado: 2026-09-24
---

# Processamento de fluxo, filas e logs de eventos

> Dados reais nunca "terminam": usuários geram eventos todo dia. Processamento de fluxo é o lote aplicado continuamente a **entradas ilimitadas**. O transporte é feito por **brokers de mensagens**, que podem ser de dois estilos: fila tradicional (AMQP/JMS) ou **log particionado** (Kafka). A ideia mais poderosa do capítulo é que **escritas em um banco também são um fluxo** (CDC e event sourcing). Isso resolve a sincronização de caches, índices e warehouses sem gravação dupla.

## Por que importa

Filas aparecem em quase todo sistema real: envio de e-mails, processamento de pagamentos, notificações, analytics, integração entre serviços. Escolher o tipo de broker, a semântica de entrega e a estratégia de idempotência define se o sistema perde mensagens, duplica efeitos ou trava quando um consumidor fica lento.

## Conceitos-chave

### Eventos e transporte

- **Evento:** registro pequeno, autocontido e **imutável** de algo que aconteceu, com timestamp. É produzido uma vez e consumido por vários consumidores, agrupados em **tópicos**.
- **Polling** em banco fica caro com baixa latência. O ideal é **notificar** os consumidores.
- **Duas perguntas para classificar qualquer sistema de mensagens:**
  1. **E se o produtor for mais rápido que o consumidor?** Descartar, **enfileirar** (e aí: e se a fila não couber na memória?) ou aplicar **backpressure**, que bloqueia o produtor, como fazem os pipes Unix e o TCP.
  2. **E se um nó cair?** Mensagens se perdem? Durabilidade exige disco e/ou replicação, e isso custa. Perder métricas periódicas pode ser aceitável. Perder eventos que você **conta** não é.

### Mensagens diretas × brokers

- **Diretas (sem broker):** UDP multicast (feeds financeiros), ZeroMQ, StatsD via UDP (métricas aproximadas) e **webhooks** (callback HTTP). Assumem produtor e consumidor online. Um consumidor offline perde mensagens.
- **Broker (fila de mensagens):** servidor que centraliza as mensagens, tolera clientes entrando e saindo e cuida da durabilidade. O consumo é **assíncrono**: o produtor só espera o broker confirmar o recebimento.

### Broker tradicional (AMQP/JMS: RabbitMQ, ActiveMQ, SQS, Pub/Sub)

- **Diferenças para um banco:** apaga a mensagem após a entrega; assume filas curtas (fica mais lento quando acumula); assinatura por padrão de tópico em vez de índices; **notifica mudanças** em vez de responder consultas pontuais.
- **Vários consumidores:**
  - **balanceamento de carga:** cada mensagem vai para um consumidor, útil para trabalho caro e paralelizável;
  - **fan-out:** cada mensagem vai para todos;
  - dá para combinar os dois (grupos).
- **Ack e reentrega:** o consumidor confirma quando termina. Sem ack (queda ou timeout), o broker reentrega. O ack pode se perder depois do processamento, então **o processamento pode acontecer duas vezes**.
- **Balanceamento + reentrega = mensagens fora de ordem.** Se a ordem importa, use uma fila por consumidor.

### Log particionado (Kafka, Kinesis)

- **Como funciona:** o produtor **acrescenta** ao log, e o consumidor **lê sequencialmente** e espera novas mensagens (como `tail -f`). O log é particionado para escalar. Cada mensagem tem um **offset** crescente dentro da partição: **ordem total dentro da partição, nenhuma entre partições**. Grava tudo em disco e mesmo assim alcança milhões de mensagens por segundo, com replicação.
- **Fan-out é trivial:** ler não apaga.
- **Balanceamento é por partição inteira:** o número de consumidores paralelos é limitado ao número de partições. Uma mensagem lenta atrasa as seguintes da mesma partição (*head-of-line*).
- **Offsets do consumidor:** basta registrar a posição, sem ack por mensagem. É exatamente como o LSN na replicação líder–seguidor. Se o consumidor cai depois de processar e antes de gravar o offset, **reprocessa**.
- **Espaço em disco:** segmentos antigos são apagados, o que forma um buffer circular grande. O livro estima ~11 horas de escrita a toda velocidade em um disco de 6 TB; na prática, dias ou semanas. **A vazão não cai com a retenção**, ao contrário de brokers que derramam para o disco.
- **Consumidor lento:** só ele é afetado. Dá para monitorar o *lag* e corrigir antes de perder mensagens. Também dá para ler o log de produção para testes sem atrapalhar ninguém.
- **Replay:** consumir não é destrutivo, então é possível voltar o offset e reprocessar (ex.: ontem inteiro, com código corrigido) gravando em outro lugar. Traz a filosofia do lote para o fluxo.

| Quando usar | Fila tradicional | Log particionado |
|---|---|---|
| Mensagens caras e independentes, paralelismo por mensagem | ✓ | |
| Alta vazão, mensagens rápidas, **ordem importa** | | ✓ |
| Reler histórico, vários consumidores independentes | | ✓ |
| Fila de tarefas estilo RPC assíncrono | ✓ | |

### Bancos de dados e fluxos

- **O log de replicação já é um fluxo** de eventos de escrita, e a replicação de máquina de estados é a mesma ideia.
- **Manter sistemas em sincronia** (banco, cache, índice de busca, warehouse):
  - **dumps periódicos** são lentos;
  - **gravação dupla** (a aplicação escreve no banco e depois no índice) é **perigosa**: com escritas concorrentes, cada sistema pode ver uma ordem diferente e ficar **permanentemente inconsistente**, sem erro nenhum; e uma das escritas pode falhar (problema de commit atômico).
- **Change Data Capture (CDC):** extrair do log do banco todas as mudanças, **na ordem**, e aplicá-las nos sistemas derivados. O banco vira o **líder** e o resto vira **seguidor**.
  - Implementações: triggers (frágeis e caras) ou leitura do log de replicação (Debezium para MySQL e PostgreSQL, Maxwell, GoldenGate).
  - É assíncrono, então valem todas as anomalias de atraso de replicação.
  - **Snapshot inicial** associado a uma posição do log.
  - **Compactação de log:** mantém só o último valor por chave (tombstone = exclusão), o que permite reconstruir um sistema derivado lendo o tópico compactado desde o offset 0 (Kafka suporta).
- **Event sourcing** (vem do DDD): a aplicação grava **eventos de domínio imutáveis** ("aluno cancelou a matrícula"), e não mudanças de estado ("linha apagada da tabela X").
  - Facilita evoluir a aplicação, depurar ("por que isso aconteceu?") e se proteger de bugs.
  - O estado atual é **derivado** do log de forma determinística, com snapshots como otimização. A compactação por chave não funciona igual ao CDC, porque eventos posteriores não substituem os anteriores.
- **Comando × evento:** o comando ainda pode falhar e precisa ser **validado de forma síncrona** (ex.: transação que valida e publica). O evento é **fato** imutável, e o consumidor não pode rejeitá-lo. Alternativa: reserva provisória, depois confirmação assíncrona.

### Estado, fluxos e imutabilidade

- **Estado mutável e log imutável são duas faces da mesma moeda.** O estado é a integral dos eventos no tempo, e o fluxo de mudanças é a derivada. Resumo da citação de Pat Helland no livro: "a verdade é o log; o banco é um cache de um subconjunto do log".
- **Vantagens da imutabilidade:**
  - **auditoria** (contabilidade corrige com lançamentos compensatórios, nunca apagando);
  - recuperação de bugs;
  - mais informação (ex.: item adicionado e depois removido do carrinho é sinal para analytics).
- **Várias visões a partir do mesmo log (CQRS):** separar a forma de escrita da forma de leitura. É razoável **desnormalizar** visões de leitura, porque o log as mantém coerentes. Uma visão nova para uma funcionalidade nova roda ao lado da antiga, sem migração arriscada. A timeline do Twitter é um exemplo.
- **Concorrência:**
  - **ruim:** consumidores são assíncronos, então a leitura das próprias escritas não é automática;
  - **boa:** um evento autocontido é **uma escrita só** (atômica), e um consumidor de thread única por partição dispensa controle de concorrência.
- **Limites da imutabilidade:** cargas com muitas atualizações geram históricos enormes. E há **obrigações de apagar** (privacidade, dados vazados), o que exige reescrever o histórico (*excision*). Apagar de verdade é difícil: cópias vivem em disco, SSD e backups.

### Processar fluxos

- **Três destinos:** gravar em armazenamento (banco, cache, índice), empurrar para humanos (alertas, dashboards) ou gerar outros fluxos (pipelines de operadores).
- **Diferença para o lote:** o fluxo nunca termina. Não dá para ordenar tudo, e a tolerância a falhas não pode ser "recomeçar do zero".
- **Usos:**
  - **CEP** (processamento de eventos complexos): consultas guardadas, eventos passando por elas (detecção de padrões e fraude);
  - **analytics de fluxo:** taxas, médias móveis, comparação com períodos anteriores. Usa janelas e algoritmos probabilísticos (HyperLogLog, Bloom, percentis aproximados), que são **otimizações**: o fluxo não é inerentemente inexato;
  - **manter views materializadas:** a janela vai até o "início do tempo";
  - **busca em fluxos:** consultas armazenadas, documentos passando (ex.: alertas de novos imóveis; o *percolator* do Elasticsearch).

### Raciocinar sobre tempo

- **Tempo do evento × tempo de processamento.** Usar o relógio do processador cria artefatos: após um redeploy, o backlog parece um pico falso. Eventos chegam fora de ordem (a analogia do livro é a ordem de lançamento dos filmes de Star Wars × a ordem da história).
- **Quando uma janela está completa?** Nunca se sabe. **Retardatários:** ignorá-los (e medir quantos) ou publicar uma **correção**. Marcadores "não haverá mais eventos antes de t" (*watermarks*) ajudam, mas precisam ser rastreados por produtor.
- **Qual relógio?** Dispositivos offline enviam eventos horas depois, e o relógio do dispositivo não é confiável. Solução: registrar **três timestamps** (ocorrido pelo dispositivo, enviado pelo dispositivo, recebido pelo servidor) e estimar o desvio.
- **Tipos de janela:**
  - **tumbling:** fixa, sem sobreposição;
  - **hopping:** fixa, com sobreposição;
  - **sliding:** eventos a até X um do outro;
  - **sessão:** agrupa por inatividade, sem duração fixa.

### Joins em fluxo

| Tipo | Exemplo | Estado mantido |
|---|---|---|
| **Fluxo–fluxo** (janela) | Busca × clique na mesma sessão em até 1 h, para taxa de clique | Eventos recentes indexados pela chave |
| **Fluxo–tabela** (enriquecimento) | Evento de atividade + perfil do usuário | Cópia local da tabela, atualizada via CDC (não consulte o banco remoto por evento) |
| **Tabela–tabela** (view materializada) | Timeline = tweets ⋈ seguidores | As duas tabelas, atualizadas por seus changelogs |

- **Dependência de tempo:** com qual versão do estado se faz o join (ex.: a alíquota de imposto da data da venda)? Sem ordem entre fluxos, o join fica **não determinístico**. Solução de data warehouse: **dimensão de mudança lenta (SCD)**, com ID por versão do registro, ao custo de não poder compactar.

### Tolerância a falhas e "exatamente uma vez"

- **Exactly-once** significaria que o efeito visível é o de processar cada registro uma vez, mesmo com reprocessamentos. O nome mais honesto é "**effectively-once**".
- **Microbatching** (Spark Streaming, ~1 s) e **checkpoints** (Flink, com barreiras no fluxo) dão exatamente uma vez **dentro do framework**. Efeitos externos (banco, e-mail, outro broker) se repetem em caso de retry.
- **Commit atômico restrito:** transações internas ao framework que cobrem estado, mensagens de saída e offsets. Evita XA heterogêneo. É o que o Kafka acrescentou depois com transações; confirme na documentação.
- **Idempotência:** fazer uma operação N vezes tem o mesmo efeito que uma. Definir um valor é idempotente; incrementar não é. **Torne idempotente com metadados**: grave o offset da mensagem junto com o valor e ignore reaplicações. Exige replay na mesma ordem, processamento determinístico, um único escritor e fencing no failover.
- **Reconstruir estado após falha:**
  - estado remoto replicado (lento por mensagem);
  - estado local com snapshots (Flink);
  - changelog em tópico compactado (Kafka Streams, Samza);
  - ou simplesmente **rejogar** a janela curta.

## Trade-offs e decisões

- **Fila × log:** paralelismo por mensagem e simplicidade contra ordem, replay e vários consumidores.
- **Descartar × enfileirar × backpressure** quando o consumidor não acompanha.
- **Gravação dupla × CDC/outbox:** simplicidade aparente contra consistência real.
- **Tempo de evento × tempo de processamento:** correção contra simplicidade.
- **Transações × idempotência** para "exatamente uma vez".

## Aplicação no mundo real

- **Filas de tarefas:** RabbitMQ, SQS (com *visibility timeout* e **DLQ**, a fila de mensagens mortas, para mensagens que falham repetidamente) e BullMQ sobre Redis no ecossistema Node.js.
- **Logs de eventos:** Kafka, Redpanda, Kinesis, Pub/Sub. Monitore o **lag do consumidor** como métrica principal.
- **CDC:** Debezium lendo o WAL do PostgreSQL (replicação lógica) e publicando em Kafka.
- **Padrão outbox:** a transação grava a entidade e a linha na tabela `outbox`, e um relay (ou CDC) publica. Elimina a gravação dupla entre banco e broker.
- **Consumidor idempotente:** tabela de IDs de mensagens processadas (com `UNIQUE`) atualizada na mesma transação do efeito.
- **Frameworks de fluxo:** Flink, Kafka Streams, Spark Structured Streaming. Para volumes pequenos, um consumidor simples com agregação em janelas no banco resolve.
- *Nomes, capacidades e semânticas desses produtos mudam com as versões. Confirme na documentação oficial antes de montar exercícios.*
- *O SDI1 (caps. 1 e 10) trata a fila de mensagens como componente de desacoplamento e absorção de picos, com workers escaláveis de forma independente e retries: a mesma ideia, em nível introdutório.*

## No Projeto prático (encurtador de links)

- **Cliques como fluxo:** o redirecionamento publica o evento `clique` (código, timestamp do evento, referência, user-agent) e responde **sem esperar** o processamento. O worker agrega em janelas (ex.: tumbling de 1 minuto) e grava contadores.
- **Fila ou log?** Começar com RabbitMQ (como o bootstrap sugere) ensina ack, reentrega e DLQ. Depois comparar com um log (Redpanda ou Kafka) para replay e vários consumidores (agregador + detecção de abuso).
- **Idempotência:** grave o ID do evento processado para que a reentrega não conte o clique duas vezes.
- **Tempo:** agregue por **tempo do evento**, não pela hora em que o worker processou, e teste derrubando o worker por 2 minutos (o backlog não pode virar um "pico").
- **Outbox na criação de link:** o evento "link criado" é publicado de forma confiável para consumidores (ex.: verificação de URL maliciosa).

## Armadilhas comuns

- Gravação dupla em banco + cache/índice/fila.
- Consumidor não idempotente com entrega "pelo menos uma vez".
- Agregar por tempo de processamento.
- Fila crescendo sem limite nem alerta de lag.
- Esperar ordem global em um tópico com várias partições.
- Achar que "exactly-once" do framework cobre efeitos externos.

## Perguntas de verificação

1. Quais são as duas perguntas para classificar um sistema de mensagens?
2. Por que balanceamento de carga + reentrega reordena mensagens?
3. Compare fila tradicional e log particionado quanto a ordem, paralelismo e replay.
4. Explique a condição de corrida da gravação dupla e como o CDC a evita.
5. Diferencie comando e evento no event sourcing.
6. Por que agregar por tempo de processamento gera picos falsos?
7. Como tornar idempotente o incremento de um contador disparado por mensagens?

## Referências

- [DDIA1, cap. 11, "Transmitting Event Streams", pp. 440–451]: sistemas de mensagens, brokers × bancos, vários consumidores, ack e reentrega, logs particionados, offsets, espaço em disco, replay.
- [DDIA1, cap. 11, "Databases and Streams", pp. 451–464]: gravação dupla, CDC, snapshot, compactação de log, event sourcing, comandos × eventos, imutabilidade, CQRS, limites da imutabilidade.
- [DDIA1, cap. 11, "Processing Streams", pp. 464–480]: usos, CEP, analytics, views materializadas, tempo de evento × processamento, janelas, joins, tolerância a falhas, microbatching, checkpoints, idempotência.
- [SDI1, cap. 1, seção "Message queue"]: fila como desacoplamento entre produtores e workers.
